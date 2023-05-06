## Prerequirements

On each VM call:
```console
$ sudo su -m
```

## VM1

### Install Node-Exporter

```console
$ sudo wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
$ sudo tar xvfz node_exporter-1.5.0.linux-amd64.tar.gz
$ sudo mv node_exporter-1.5.0.linux-amd64/node_exporter /usr/local/bin/
$ sudo useradd -rs /bin/false node_exporter
$ sudo vim /etc/systemd/system/node_exporter.service
```

Configure systemd service to run Node Exporter.
```conf
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Reload systemd info and start Node Exporter service.
```console
$ sudo systemctl daemon-reload
$ sudo systemctl start node_exporter
$ sudo systemctl enable node_exporter
```

### Alert Manager

#### Install

```console
$ sudo wget https://github.com/prometheus/alertmanager/releases/download/v0.24.0/alertmanager-0.24.0.linux-amd64.tar.gz
$ sudo tar xvf alertmanager-0.24.0.linux-amd64.tar.gz
$ sudo mkdir /etc/alertmanager /var/lib/prometheus/alertmanager
$ sudo (cd alertmanager-0.24.0.linux-amd64 && cp alertmanager amtool /usr/local/bin/ && cp alertmanager.yml /etc/alertmanager)
```

#### Configuration

Register at https://t.me/MiddlemanBot - this bot will handle your alerts.

Create AlertManager configuration file.
```console
$ sudo vim /etc/alertmanager/alertmanager.yml
```

Place this into created file. Add your token instead of placeholder.
```yaml
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 1m
  repeat_interval: 3m
  receiver: 'telepush'
receivers:
  - name: 'telepush'
    webhook_configs:
      - url: 'https://telepush.dev/api/inlets/alertmanager/<your token from bot>'
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

Create user to run AlertManager from.
```console
$ sudo useradd --no-create-home --shell /bin/false alertmanager
$ sudo chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/prometheus/alertmanager
$ sudo chown alertmanager:alertmanager /usr/local/bin/{alertmanager,amtool}
```

#### Service

```console
$ sudo vim /etc/systemd/system/alertmanager.service
```

Place this as AlertManager config.
```conf
[Unit]
Description=Alertmanager Service
After=network.target

[Service]
EnvironmentFile=-/etc/default/alertmanager
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
--config.file=/etc/alertmanager/alertmanager.yml \
--storage.path=/var/lib/prometheus/alertmanager \
--cluster.advertise-address="127.0.0.1:9093"\
$ALERTMANAGER_OPTS
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Reload systemd info and start Prometheus service.
```console
$ sudo systemctl daemon-reload
$ sudo systemctl start alertmanager
$ sudo systemctl enable alertmanager
```


### Prometheus
#### Download

Download the source using curl, untar it, and rename the extracted folder to prometheus-files.
```console
$ sudo wget https://github.com/prometheus/prometheus/releases/download/v2.43.0-rc.1%2Bstringlabels/prometheus-2.43.0-rc.1%2Bstringlabels.linux-amd64.tar.gz
$ sudo tar -xvf prometheus-2.22.0.linux-amd64.tar.gz
$ sudo mv prometheus-2.22.0.linux-amd64 prometheus-files
```

#### Installation

Create a Prometheus user, required directories, and make Prometheus the user as the owner of those directories.

```console
$ sudo useradd --no-create-home --shell /bin/false prometheus
$ sudo mkdir -p /etc/prometheus
$ sudo mkdir -p /var/lib/prometheus
$ sudo chown prometheus:prometheus /etc/prometheus
$ sudo chown prometheus:prometheus /var/lib/prometheus
```

Copy prometheus and promtool binary from prometheus-files folder to /usr/local/bin and change the ownership to prometheus user.
```console
$ sudo cp prometheus-files/prometheus /usr/local/bin/
$ sudo cp prometheus-files/promtool /usr/local/bin/
$ sudo chown prometheus:prometheus /usr/local/bin/prometheus
$ sudo chown prometheus:prometheus /usr/local/bin/promtool
```

Move the consoles and console_libraries directories from prometheus-files to /etc/prometheus folder and change the ownership to prometheus user.
```console
sudo cp -r prometheus-files/consoles /etc/prometheus
sudo cp -r prometheus-files/console_libraries /etc/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

#### Configuration

Create a Prometheus configuration file.

```console
$ sudo vim /etc/prometheus/prometheus.yml
```

Put this as Prometheus config. Replace placeholders with correct IPs.
```yaml
gglobal:
  scrape_interval: 10s

scrape_configs:
- job_name: 'prometheus'
  static_configs:
    - targets: ['<first host IP>:9090']

- job_name: node
  scrape_interval: 5s
  static_configs:
    - targets: ['<first host IP>:9100', '<second host IP>:9100']

- job_name: 'telegraf'
  scrape_interval: 5s
  static_configs:
    - targets: ['<second host IP>:9273']

rule_files:
  - "alerts.yml"

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - "localhost:9093"
```

#### Alerts

Create a Prometheus alerts configuration file.

```console
$ vim /etc/prometheus/alerts.yml
```

```yaml
groups:
  - name: Critical alers
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute.'
          summary: Instance {{ $labels.instance }} downestart=on-failure
      - alert: Nginx systemd down
        expr: node_systemd_unit_state{name="nginx.service",state="active"} == 0
        for: 1s
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} is down."
```

#### Service

Create a prometheus service file.

```console
$ sudo vim /etc/systemd/system/prometheus.service
```

```conf
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --storage.tsdb.retention.time=20d \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

Reload systemd info and start Prometheus service.
```console
$ sudo systemctl daemon-reload
$ sudo systemctl start prometheus
$ sudo systemctl enable prometheus
```

## VM2

### Node Exporter

Do everything from VM1 but place this systemd config instead of previos one.
```conf
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --collector.systemd \
  --collector.systemd.unit-whitelist="nginx.service"

[Install]
WantedBy=multi-user.target
```

### Nginx

#### Stats

Add separated server for stub status only:
```console
$ sudo vim /etc/nginx/conf.d/stub_status_nginx.conf
```

And place this contents to created file.
```
server {
        listen 81 default_server;
        listen [::]:81 default_server;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                try_files $uri $uri/ =404;
        }

        location /nginx_status {
                stub_status on;
                allow 127.0.0.1;
                allow ::1;
                deny all;
        }
}
```

Then apply new changes:
```console
$ sudo systemctl restart nginx
```

#### Logs

Change log files mod to use them later in Telegraf:
```console
$ (cd /var/log/nginx/ && sudo chmod 755 access.log error.log)
```

### Logrotate

```console
$ sudo apt-get install logrotate
```

Make sure your `/etc/logrotate.d/nginx` to look like:
```
/var/log/nginx/*.log {
        daily
        missingok
        rotate 14
        compress
        delaycompress
        notifempty
        create 0640 www-data adm
        sharedscripts
        prerotate
                if [ -d /etc/logrotate.d/httpd-prerotate ]; then \
                        run-parts /etc/logrotate.d/httpd-prerotate; \
                fi \
        endscript
        postrotate
                invoke-rc.d nginx rotate >/dev/null 2>&1
        endscript
}
```

Logrotate will automatically run on your server, once installed. You don’t need to start the service on boot or set up cron jobs for it.

### Telegraf

```console
$ sudo apt-get update && sudo apt-get install telegraf
$ sudo vim /etc/telegraf/telegraf.conf
```

You need to add next parts to your Telegraf config

#### Nginx Status

Make `[[inputs.nginx]]` part to be like:
```conf
[[inputs.nginx]]
  urls = ["http://localhost:81/nginx_status"]
  response_timeout = "5s"
```

#### Nginx logs

The same, just make it to be like that one:
```conf
[[inputs.tail]]
        name_override = "nginxlog"
        files = ["/var/log/nginx/access.log"]
        from_beginning = true
        pipe = false
        data_format = "grok"
        grok_patterns = ["%{COMBINED_LOG_FORMAT}"]
```

#### Misc

Also edit `[[inputs.cpu]] to be like:
```conf
[[inputs.cpu]]
   percpu = false
   totalcpu = true
   collect_cpu_time = true
   report_active = true
```

Finally, make shure your `[[outputs.prometheus_client]]` looks like the one here.

__**Important**__ - port here and in Prometheus config must match.
```conf
[[outputs.prometheus_client]]
    listen = ":9273"
    metric_version = 2
```

### Install Grafana

Install Grafanf service.
```console
$ sudo apt-get install -y adduser libfontconfig1
$ sudo wget https://dl.grafana.com/oss/release/grafana_9.4.3_amd64.deb
$ sudo dpkg -i grafana_9.4.3_amd64.deb
$ sudo /bin/systemctl daemon-reload
$ sudo /bin/systemctl enable grafana-server
$ sudo /bin/systemctl start grafana-server
```

Configure Grafana to serve behind nginx proxy.
```console
$ sudo vim /etc/grafana/grafana.ini
```

Add these lines after `[server]`:
```
root_url = %(protocol)s://%(domain)s:%(http_port)s/grafana/
serve_from_sub_path = true
```

Restart server to apply new settings
```
$ sudo systemctl restart grafana-server
```

### Nginx

#### Configure

Edit nginx config from lab 2 to serve grafana.

```console
$ sudo vim /etc/nginx/sites-enabled/default
```

Add this one outside of server config.
```
map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
}

upstream grafana {
        server localhost:3000;
}
```

And this inside your server on `:80`

```
        location /grafana/ {
                rewrite  ^/grafana/(.*)  /$1 break;
                proxy_set_header Host $http_host;
                proxy_pass http://grafana;
        }

        # Proxy Grafana Live WebSocket connections.
        location /grafana/api/live/ {
                rewrite  ^/grafana/(.*)  /$1 break;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
                proxy_set_header Host $http_host;
                proxy_pass http://grafana;
        }
```

Restart nginx to apply changes.
```console
$ sudo systemctl restart nginx
```

### Configure Grafana

* Add Prometheus Datasource with address `http://<first host IP>:9090`
* Import New Dashboard with ID `1860` (or copy from `node-exporter-full_rev30.json`)
* Import New Dashboard with ID `3662` (or copy from `nginx_rev2.json`)
* Import New Dashboard with ID `14900` (or copy from `prometheus-2-0-overview_rev2.json`)
