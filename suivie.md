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

http://192.168.56.109:3000

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

sudo nano /etc/systemd/system/alertmanager.service

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
systemctl start alertmanager 
systemctl enable alertmanager 
systemctl status alertmanager 
```
On peut accéder au site web avec 

"http://192.168.56.108:9093"

## Configurer les alertes

```
sudo mkdir /etc/prometheus/rules
chown -R prometheus:prometheus /etc/prometheus
sudo nano /etc/prometheus/rules/alert.yaml
```
Modification du fichier alert.yaml
```bash
- alert: PrometheusTargetMissing
    expr: up == 0
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: Prometheus target missing (instance {{ $labels.instance }})
      description: "A Prometheus target has disappeared. An exporter might be crashed.\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```

# Modification du fichier 

sudo nano /etc/prometheus/prometheus.yml

```bash
# Location of the alert rules.
rule_files:
    - "rules/alert.yaml"
```
```bash
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: 'application-server'
    static_configs:
      - targets: ['3.90.108.255:9100']
```

```
sudo systemctl restart prometheus
```

# MAJ alertmanager.yml

```
sudo nano /var/lib/alertmanager/alertmanager.yml
```

```
global:
  slack_api_url: 'https://hooks.slack.com/services/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'
route:
  # The below receiver is a Default receiver. If the alert doesn't match any of the receivers in routes section, It will send the alert to the default receiver.
  receiver: 'slack-notifications'
  # To send alerts to different receivers based on different conditions, We can use the "routes" section.
  # routes:
  #   - match:
  #       severity: critical
  #     receiver: 'slack-notifications'
  #   - match:
  #       severity: warning
  #     receiver: 'slack-notifications'
  
receivers:
  - name: 'slack-notifications'
    slack_configs:
      - send_resolved: true
        channel: '#channel_name'
        icon_emjoi: ':warning:'
```
```
sudo systemctl restart alertmanager
```


## Configuration alerte envoyer par mail 

```powershell
 mkdir /etc/prometheus/rules
 chown -R prometheus:prometheus /etc/prometheus
 nano /etc/prometheus/prometheus.yml
```
### modification du fichier /prometheus/prometheus.yml

```powershell
global:                                                       scrape_interval:     15s # Set the scrape interval to eve>  evaluation_interval: 15s # Evaluate rules every 15 second>  # scrape_timeout is set to the global default (10s).                                                                    # Attach these labels to any time series or alerts when c>  # external systems (federation, remote storage, Alertmana>  external_labels:
      monitor: 'example'

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets: ['localhost:9093']

# Load rules once and periodically evaluate them according >rule_files:
  - /etc/prometheus/rules/alerts.yml
# A scrape configuration containing exactly one endpoint to># Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to an>  - job_name: 'prometheus'
    scrape_interval: 5s
    scrape_timeout: 5s
    scrape_timeout: 5s
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node_exporter
    # If prometheus-node-exporter is installed, grab stats >    # machine by default.
    static_configs:
      - targets: ['localhost:9100']    
```

### création / modification du fichier alerts.yml

```
nano /etc/prometheus/rules/alerts.yml
```


```powershell
 GNU nano 7.2  /etc/prometheus/rules/alerts.yml            groups:
  - name: mail
    rules:
      - alert: Trop_de_load
        expr: node_load1 >= 0.6
        for: 10s
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} - trop de load"
          description: "Le serveur a trop de load "

```

``` powershell
systemctl restart prometheus
```

## La partie notifications 

### On télécharge alertmanager

```
 wget https://github.com/prometheus/alertmanager/releases/download/v0.28.0-rc.0/alertmanager-0.28.0-rc.0.linux-amd64.tar.gz
```
### On extrait les documents

```
tar -xvzf /etc/alertmanager-0.28.0-rc.0.linux-amd64.tar.gz -C /etc/
```

### modification du fichier alertmanager.yml

```
nano /etc/alertmanager/alertmanager.yml
```

```powershell

global:              #génériques
  resolve_timeout: 30s
  smtp_require_tls: false

route:                #règles de déclenchement
  group_by: ['instance', 'severity']
  group_wait: 10s       #temps d'attente avant no>  group_interval: 1m    #délai par rapport aux al>  repeat_interval: 1h    #attente avant répétition  receiver: 'email-me'
  routes:
  - match:
      alertname: Trop_de_load

receivers:
  - name: 'email-me'
    email_configs:
    - to: "linuxmonitoring2@gmail.com"
      from: "linuxmonitoring2@gmail.com"
      smarthorst: "smtp.gmail.com:465"
      auth_username: "linuxmonitoring2@gmail.com"
      auth_identity: "linuxmonitoring2@gmail.com"
      auth_password: "monitoring_test17"
```


### modification des mails pour accepter les modification venant d'application




