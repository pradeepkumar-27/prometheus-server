# Prometheus Server installation on RHEL8

Prometheus is an open-source time series monitoring and alerting toolkit originally developed at SoundCloud. It has very active development and community and has seen wide adoption by many organizations and companies.

Below are the steps to install Prometheus monitoring tool on RHEL 8.

Step 1 : Add system user and group for Prometheus
```
sudo groupadd --system prometheus

sudo useradd -s /sbin/nologin --system -g prometheus prometheus
```
Note that this user doesn’t have /bin/bash shell, that’s why we used -s /sbin/nologin.

Step 2 : Create data directory for Prometheus
Once the system user and group has been created, proceed to create a directory that will be used to store Prometheus data. This includes the metrics collected from the agents being monitored.
```
sudo mkdir /var/lib/prometheus
```
You can choose to use a different path, e.g separate partition.

Step 3 : Create configuration directories for Prometheus
Prometheus primary configuration files directory is /etc/prometheus/. It will have some sub-directories.
```
mkdir /etc/prometheus/rules /etc/prometheus/rules.d \
  /etc/prometheus/files_sd
 ```

Step 4 : Download Prometheus on CentOS 8 / RHEL 8
LTS version 2.37.8 is used in this installation

You can use curl or wget to download from the command line.
```
wget https://github.com/prometheus/prometheus/releases/download/v2.37.8/prometheus-2.37.8.linux-amd64.tar.gz

tar xvf prometheus-*.tar.gz

cd prometheus-*/

cp prometheus promtool /usr/local/bin/
```
Also copy consoles and console_libraries to /etc/prometheus directory:
```
cp -r prometheus.yml consoles/ console_libraries/ /etc/prometheus/ 
```

Step 5 : Create a Prometheus configuration file.
Prometheus configuration file will be located under /etc/prometheus/prometheus.yml. Create simple configurations using content:
```
# Global config
global: 
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.  
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.  
  scrape_timeout: 15s  # scrape_timeout is set to the global default (10s).

# A scrape configuration containing exactly one endpoint to scrape:# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
```
Make changes to the file to fit your initial setting and save the file.

Step 6 : Create systemd Service unit
To be able to manage Prometheus service with systemd, you need to explicitly define this unit file.

Create a file
```
sudo vi /etc/systemd/system/prometheus.service 
```
Add the following contents to it.
```
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
```

Set correct directory permissions.
```
sudo chown -R prometheus:prometheus /etc/prometheus

sudo chmod -R 775 /etc/prometheus/

sudo chown -R prometheus:prometheus /var/lib/prometheus/
```

Start Prometheus service.
```
sudo systemctl daemon-reload

sudo systemctl start prometheus
```
Enable the service to start at system boot:
```
sudo systemctl enable prometheus
```
Check status using systemctl status prometheus command:
```
systemctl status prometheus.service 
```

Step 7 : Configure firewalld
I’ll allow access to Prometheus management interface port 9090 from my trusted network using Firewalld rich rules.
```
sudo firewall-cmd --add-port=9090/tcp --permanent

sudo firewall-cmd --reload
```
Open Prometheus Server IP/Hostname and port 9090

## Enable Basic Server Authentication
- Create /etc/prometheus/web.yaml file
```
basic_auth_users:
        <username>: '<bcrypt_encrypted_password>'
```
where <bcrypt_scrypeted_password> is the pasword encrypted with bcrypt tool

- Edit /etc/systemd/system/prometheus.yaml and add "--web.config.file=/etc/prometheus/web.yml"
```
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --web.config.file=/etc/prometheus/web.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
```

- Restart prometheus service

---

## Promtheus Linux Node Exporter

Set up Prometheus Node Exporter on Linux

- Download Node Exporter: Visit the official Prometheus GitHub repository (https://github.com/prometheus/node_exporter/releases) and download the latest version of Node Exporter for Linux. You can choose the appropriate binary based on your system architecture (e.g., 64-bit or ARM).
```
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.6.0/node_exporter-1.6.0.linux-amd64.tar.gz
```

- Extract the Archive
```
tar -xvzf node_exporter-1.6.0.linux-amd64.tar.gz
```
This will create a new directory with the extracted files.

- Move the Files: Move the extracted files to the desired installation location. For example, you can move them to /usr/local/bin/node_exporter:
```
sudo mv node_exporter-1.6.0.linux-amd64/node_exporter /usr/local/bin/
```

- Create a Systemd Service: Create a systemd service unit file to manage the Node Exporter service. Use a text editor to create a new file:

Create a service /etc/systemd/system/prometheus-node-exporter.service
Add the following content to the file:
```
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/bin/bash -c '/usr/local/bin/node_exporter'

[Install]
WantedBy=multi-user.target
```
Save the file and exit the text editor.

- Reload Systemd and Start Node Exporter: After creating the service unit file, reload the systemd configuration and start the Node Exporter service:
```
sudo systemctl daemon-reload
sudo systemctl start prometheus-node-exporter
```
- Enable Autostart on Boot: To ensure that Node Exporter starts automatically on system boot, enable the service:
```
sudo systemctl enable prometheus-node-exporter
```
- Allow firewall  for port 9100
```
sudo firewall-cmd --add-port=9090/tcp --permanent

sudo firewall-cmd --reload
```

- Verify the Installation: Check if the Node Exporter service is running and accessible. You can access the metrics exposed by Node Exporter by visiting http://localhost:9100/metrics in a web browser or using tools like curl:
```
curl http://localhost:9100/metrics
```
If the service is running correctly, you should see a list of metrics in the response.

- Configure promtheus job for node exeporter in prometheus config file /etc/prometheus/prometheus.yml
```
# Global config
global: 
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.  
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.  
  scrape_timeout: 15s  # scrape_timeout is set to the global default (10s).

# A scrape configuration containing exactly one endpoint to scrape:# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
    basic_auth:
      username: prometheus
      password: prometheus
      
  - job_name: 'prometheus.devops.org-node-exporter'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9100']
```

That's it! You have successfully set up Prometheus Node Exporter on your Linux system. The Node Exporter will now collect various system metrics that can be scraped by Prometheus for monitoring and analysis.


## Ngninx Proxy with TLS Encryption for Prometheus

Refer [nginx-server](https://github.com/pradeepkumar-27/nginx-server) doc for Nginx server installation.

- Generate TLS/SSL certificate with Openssl for the server
```
openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 \
    -subj "/C=IN/ST=Tami Nadu/L=Ooty \
    /O=DevOps Org/CN=prometheus.devops.org" \
    -keyout /etc/nginx/cert.key -out /etc/nginx/cert.crt
```

- Create prometheus.conf in /etc/nginx/conf.d directory
```
server {
    listen 80;
    listen [::]:80;
    server_name  prometheus.devops.org;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name prometheus.devops.org;

    ssl_certificate /etc/nginx/cert.crt;
    ssl_certificate_key /etc/nginx/cert.key;

    location / {
        proxy_pass           http://127.0.0.1:9090;
    }
}

```

- Configure /etc/nginx/nginx.conf
```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;
}

```

- Restart nginx
```
systemctl restart nginx
```

## Alert Manager
- Dowload alert manager package from prometheus downloads
```
curl -LO https://github.com/prometheus/alertmanager/releases/download/v0.25.0/alertmanager-0.25.0.linux-amd64.tar.gz
```

- Extract file
```
tar -xvzf alertmanager-0.25.0.linux-amd64.tar.gz
```

- Move the Files: Move the extracted files to the desired installation location
```
cp alertmanager /usr/local/bin

cp amtool /usr/local/bin
```

- Create and configure /etc/systemd/system/prometheus-alert-manager.service file
```
[Unit]
Description=Prometheus Alert Manager
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/bin/bash -c '/usr/local/bin/alertmanager \
  --config.file=/etc/prometheus/alertmanager.yml \
  --storage.path=/var/lib/alertmager'
Restart=always

[Install]
WantedBy=multi-user.target
```

- Create and configure the /etc/prometheus/alertsmanager.yaml file
```
route:
  receiver: admin

receivers:
  - name: admin
    email_configs:
      - to: 'devops@gmail.com'
        from: 'prometheusalertmanager@gmail.com'
        smarthost: smtp.gmail.com:587
        auth_username: 'devops@gmail.com'
        auth_identity: 'devops@gmail.com'
        auth_password: 'xxxxxxxxxxxxxxxx'
```

- Set alertmanager service permissions to user
```
chown prometheus:prometheus /usr/local/bin/alertmanager
```

- Start and enable alertmanager
```
systemctl start prometheus-alert-manager

systemctl enable prometheus-alert-manager
```

alermanager should be enabled on localhost port 9093

- Configure prometheus.yaml to use alertmanager
```
rule_files:
  - "rules/alerts.yaml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - localhost:9093
```

### Sample alerts.yaml with Node Exporter alerts
```
groups:
  - name: prometheus:node_exporter
    rules:
      # rules
      - record: job:node_cpu_seconds:avg_usage_percentage
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
      - record: job:node_cpu_seconds:usage_by_mode
        expr: sum by (mode) (irate(node_cpu_seconds_total[5m])) * 100
      - record: job:node_memory:avg_usage_percentage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
      - record: job:node_filesystem:disk_usage_percentage
        expr: 100 - (node_filesystem_avail_bytes{mountpoint="/"} * 100 / node_filesystem_size_bytes{mountpoint="/"})
      - record: job:node_network:traffic_in_mb_per_sec
        expr: (irate(node_network_receive_bytes_total[5m]) + irate(node_network_transmit_bytes_total[5m]))/1024/1024
      # alerts  
      - alert: prometheus:node_exporter:down
        expr: up{job="prometheus.devops.org:node-exporter"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "prometheus.devops.org node exporter down"
      - alert: prometheus:node_exporter:cpu:avg_cpu_percentage
        expr: job:node_cpu_seconds:avg_usage_percentage > 50
        labels:
          severity: warning
        annotations:
          summary: "prometheus.devops.org server cpu usage above threshold"
          description: "prometheus.devops.org server cpu usage above threshold {{ $value }}"
      - alert: prometheus:node_exporter:memory:avg_usage_percentage
        expr: job:node_memory:avg_usage_percentage > 60
        labels:
          severity: warning
        annotations:
          summary: "prometheus.devops.org server memory usage above threshold"
          description: "prometheus.devops.org server memory usage above threshold {{ $value }}"

      - alert: prometheus:node_exporter:disk:usage_percentage
        expr: job:node_filesystem:disk_usage_percentage > 40
        labels:
          severity: low
```