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

## Run prometheus
```powershell
docker run -d --name=prometheus -e TZ=IST -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml -p 9090:9090 
```

## Run grafana
```powershell
docker run -d - name=grafana -p 3000:3000 grafana/grafana
```


## Pour ouvrire sur le site web 

http://192.168.56.109:3000

## Install Node Exporter

```powershell
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.2.linux-amd64.tar.gz

tar xvf node_exporter-1.8.2.linux-amd64.tar.gz

cd node_exporter-1.8.2.linux-amd64.tar.gz

./node_exporter
``` 

### On peut acceder au Node Exporter sur le web :

http://192.168.56.108:9100/metrics

## Modification du script prometheus

```powershell
cd /etc/prometheus
nano prometheus.yml
```
```powershell
- job_name: 'node-exporter'
    static_configs:
    - targets: ['<node-ip>:9100']
```

# Redemarre prometheus et on vérifie qu'il capte bien. 

Status > Targets


## Modification du fichier 

```powershell
sudo nano /etc/prometheus/prometheus.yml
```

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

```powershell
sudo systemctl restart prometheus
```

# Configuration alerte envoyer par mail 

```powershell
 nano /etc/prometheus/prometheus.yml
```
### modification du fichier /prometheus/prometheus.yml

```powershell
global:                                                       
scrape_interval:     15s # Set the scrape interval to eve>  
evaluation_interval: 15s # Evaluate rules every 15 second>  
# Attach these labels to any time series or alerts when c>  
# external systems (federation, remote storage, Alertmana>  
external_labels:
  monitor: 'example'

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets: 
      - 'localhost:9093'

# Load rules once and periodically evaluate them according >
rule_files:
  - /etc/prometheus/rules/alerts.yml
# A scrape configuration containing exactly one endpoint to># Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    scrape_interval: 5s
    scrape_timeout: 5s
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node_exporter
    # If prometheus-node-exporter is installed, grab stats about the local
    # machine by default.
    static_configs:
      - targets: ['localhost:9100']  
```

### création / modification du fichier alerts.yml

``` powershell
 mkdir /etc/prometheus/rules

nano /etc/prometheus/rules/alerts.yml
```

```powershell
 GNU nano 7.2  /etc/prometheus/rules/alerts.yml    
groups:
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

```powershell
tar -xvzf alertmanager-0.28.0-rc.0.linux-amd64.tar.gz -C /etc/
mv alertmanager-0.28.0-rc.0.linux-amd64/* /etc/alertmanager/
```

### Modification des droits 

```powershell
sudo chown -R prometheus:prometheus /var/lib/alertmanager
sudo chown -R prometheus:prometheus /var/lib/alertmanager/*
sudo chmod -R 775 /var/lib/alertmanager
sudo chmod -R 775 /var/lib/alertmanager/*
```

```powershell
 nano /etc/systemd/system/alertmanager.service
```

``` powershell
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

``` powershell
systemctl daemon-reload
systemctl start alertmanager 
systemctl enable alertmanager 
systemctl status alertmanager 
```
### On peut accéder au site web avec 

"http://192.168.56.108:9093"

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
  group_wait: 10s
  group_interval: 1m    
  repeat_interval: 1h    #attente avant répétition  
  receiver: 'email-me'
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
      auth_password: "dcqj cskq amtx zmih "
```
> #### L'```auth_password``` est obtenu par google grace aux mot de passe pour application : On a crée pour l'application ```Alertmanager``` un mot de passe et on l'utilise pour l'envoie de mail.

### modification des mails pour accepter les modification venant d'application

# Création de scripts de surveillance
- Surveiller les ressources systèmes 
- Surveillance réseaux

## - Surveiller les ressources systèmes

 Création d'un dossier ou stocker les données

```powershell
sudo touch /var/log/network_monitor.log /var/log/system_monitor.log
sudo chmod 666 /var/log/network_monitor.log /var/log/system_monitor.log
```

### création du fichier 

```powershell
sudo nano  monitor_systemes /usr/local/bin/
```

scrip :
```bash
#!/bin/bash

# Chemin du fichier log
LOG_FILE="/var/log/system_monitor.log"

# Fonction pour surveiller l'utilisation du CPU
monitor_cpu() {
  CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2 + $4}')
  echo "$(date): CPU Usage: $CPU_USAGE%" >> $LOG_FILE
}

# Fonction pour surveiller l'utilisation de la mémoire
monitor_memory() {
  MEM_USAGE=$(free -m | awk 'NR==2{printf "%.2f", $3*100/$2 }')
  echo "$(date): Memory Usage: $MEM_USAGE%" >> $LOG_FILE
}

# Fonction pour surveiller l'utilisation de l'espace disque
monitor_disk() {
  DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}')
  echo "$(date): Disk Usage: $DISK_USAGE" >> $LOG_FILE
}

# Appel des fonctions
monitor_cpu
monitor_memory
monitor_disk
```

```powershell
sudo chmod +x /usr/local/bin/monitor_systemes
```

```powershell
crontab -e
```
on choisi 1 pour nano

script :
```
*/5 * * * * /usr/local/bin/monitor_systemes
```

## Surveiller le reseaux

```powershell
/usr/local/bin$ sudo nano monitor_reseaux
```

script : 
```bash
#!/bin/bash

# Chemin du fichier log
LOG_FILE="/var/log/network_monitor.log"

# Fonction pour vérifier la latence
monitor_latency() {
  LATENCY=$(ping -c 1 google.com | grep 'time=' | awk -F'time=' '{print $2}' | awk '{print $1}')
  if [ -z "$LATENCY" ]; then
    echo "$(date): Latency: Unreachable" >> $LOG_FILE
  else
    echo "$(date): Latency: $LATENCY ms" >> $LOG_FILE
  fi
}

# Fonction pour surveiller l'utilisation du réseau
monitor_bandwidth() {
  RX_PREV=$(cat /sys/class/net/enp0s8/statistics/rx_bytes)
  TX_PREV=$(cat /sys/class/net/enp0s8/statistics/tx_bytes)
  sleep 1
  RX_NEXT=$(cat /sys/class/net/enp0s8/statistics/rx_bytes)
  TX_NEXT=$(cat /sys/class/net/enp0s8/statistics/tx_bytes)

  RX_RATE=$((($RX_NEXT - $RX_PREV) / 1024))
  TX_RATE=$((($TX_NEXT - $TX_PREV) / 1024))

  echo "$(date): RX Rate: ${RX_RATE} KB/s, TX Rate: ${TX_RATE} KB/s" >> $LOG_FILE
}

# Appel des fonctions
monitor_latency
monitor_bandwidth
```

```powershell 
sudo chmod +x /usr/local/bin/monitor_reseaux
```

On viens rajouter le nouveau script dans le cron
```
crontab -e
```

```powershell
*/5 * * * * /usr/local/bin/monitor_reseaux
```

Pour lire les scrips 

```powershell
 tail -f /var/log/system_monitor.log
 tail -f /var/log/network_monitor.log
```
