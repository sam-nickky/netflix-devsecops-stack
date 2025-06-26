# DevOps Monitoring Stack Setup

## Step 1: Launch an Ubuntu (22.04) T2 Large Instance

Launch an AWS T2 Large Instance using Ubuntu 22.04. Create or use a key pair. Enable HTTP and HTTPS in the security group and open all ports (only for learning purposes).

## Step 2: Install Jenkins, Docker, and Trivy. Create a SonarQube Container

### 2A. Install Jenkins

Create `jenkins.sh` and run as root:

```bash
#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```

Open port 8080 in EC2. Access Jenkins via `http://<EC2_PUBLIC_IP>:8080`.

Get initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### 2B. Install Docker

```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

Create SonarQube container:

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

### 2C. Install Trivy

Create `trivy.sh`:

```bash
#!/bin/bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

Run it:

```bash
chmod +x trivy.sh
./trivy.sh
```

## Step 3: Create a TMDB API Key

- Visit [https://www.themoviedb.org](https://www.themoviedb.org)
- Create an account > Settings > API > Create Developer Key

## Step 4: Install Prometheus & Grafana

### Install Prometheus

```bash
sudo useradd --system --no-create-home --shell /bin/false prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
sudo mkdir -p /data /etc/prometheus
cd prometheus-2.47.1.linux-amd64/
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

Create systemd service:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

### Install Node Exporter

```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
```

Create service:

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

Update Prometheus config:

```yaml
- job_name: 'node_export'
  static_configs:
    - targets: ['localhost:9100']
```

Check config and reload:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

### Install Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get -y install grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

- Access: `http://<ip>:3000` (admin/admin)
- Add Prometheus data source (`http://localhost:9090`)
- Import Dashboard ID: `1860`

## Step 5: Monitor Jenkins with Prometheus & Grafana

- Install **Prometheus plugin** in Jenkins from Manage Jenkins > Plugins
- Prometheus endpoint available at `/prometheus`

Update Prometheus config:

```yaml
- job_name: 'jenkins'
  metrics_path: '/prometheus'
  static_configs:
    - targets: ['<jenkins-ip>:8080']
```

Reload Prometheus config:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

- Confirm in `http://<prometheus-ip>:9090/targets`
- Import Grafana dashboard ID: `9964`
