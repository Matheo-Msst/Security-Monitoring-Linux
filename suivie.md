## Installation du docker's

``` powershell
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```


## modification docker:
``` powershell 
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

## Installation de prometheus 

```powershell
mkdir /etc/prometheus
nano prometheus.yml
```

## Script 

```powershell
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
```

# Run prometheus
```powershell
docker run -d --name=prometheus -e TZ=IST -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml -p 9090:9090 
```

#ecrire ce qu'on a modifier sur le site

# Run grafana
```powershell
docker run -d - name=grafana -p 3000:3000 grafana/grafana
```
# Pour ouvrire sur le site web 

http://192.168.56.108:3000

# Install Node Exporter

```powershell
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.2.linux-amd64.tar.gz

tar xvf node_exporter-1.8.2.linux-amd64.tar.gz

cd node_exporter-1.8.2.linux-amd64.tar.gz

./node_exporter
``` 

On peut acceder au Node Exporter sur le web :

http://192.168.56.108:9100/metrics

## Modification du script prometheus

cd /etc/prometheus
nano prometheus.yml

- job_name: 'node-exporter'
    static_configs:
    - targets: ['<node-ip>:9100']

# Redemarre prometheus et on vérifie qu'il capte bien. 

Status > Targets

### Setup les alertes

```powershell
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.28.0-rc.0.linux-amd64.tar.gz

tar -xvf alertmanager-0.28.0-rc.0.linux-amd64.tar.

cd alertmanager-0.28.0-rc.0.linux-amd64

./alertmanager

```

### Configuration d'AlertManager en tant que service sur une machine Linux :

modification du fichier 

```powershell
sudo mkdir /var/lib/alertmanager
sudo mv alertmanager-0.27.0.linux-amd64/* /var/lib/alertmanager
```

```powershell
cd /var/lib/alertmanager
```

modification des droits 

```powershell
sudo chown -R prometheus:prometheus /var/lib/alertmanager
sudo chown -R prometheus:prometheus /var/lib/alertmanager/*
sudo chmod -R 775 /var/lib/alertmanager
sudo chmod -R 775 /var/lib/alertmanager/*
```

sudo vi /etc/systemd/system/alertmanager.service

```
# Add the below content to the file
[Unit]
Description=Prometheus Alertmanager
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target
[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/var/lib/alertmanager/alertmanager --storage.path="/var/lib/alertmanager/data" --config.file="/var/lib/alertmanager/alertmanager.yml"
SyslogIdentifier=prometheus_alert_manager
Restart=always
[Install]
WantedBy=multi-user.target
```

``` 
systemctl daemon-reload
systemctl stop alertmanager 
systemctl start alertmanager 
systemctl enable alertmanager 