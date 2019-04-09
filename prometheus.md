Prometheus is an open source, metrics-based monitoring system.
Time series data is a sequence of data points collected at regular intervals over a period of time (metrics). Most Prometheus components are written in Go. Some are also written in Java, Python, and Ruby. Prometheus is a system to collect and process metrics, not an event logging system (so you can't feed logs!). The three primary components of Prometheus are the Prometheus server, the visualization layer with Grafana and the Alert Management with Prometheus Alert Manager.

Prometheus server monitor particular things (which can be anything from a complete Linux server to a single process to a database service). The things that Prometheus monitors are called Targets. The Prometheus server monitors the targets. Each unit of a target such as current CPU status, memory usage (in case of a Linux server Prometheus Target) or any other specific unit that you would like to monitor is called **a metric**. So Prometheus server collects metrics from targets (over HTTP), stores them locally or remotely and displays them back in the Prometheus server. Metrics collection with Prometheus relies on the _pull model_, meaning that Prometheus is responsible for getting metrics (_scraping_) from the services that it monitors. Scrapes happen regularly; usually you would configure it to happen every 10 to 60 seconds for each target. This is diametrically opposed from other tools like Graphite, which are passively waiting on clients to _push_ their metrics to a known server. A potential challenge with this model is that Prometheus now needs to know where to collect these metrics. In a microservices architecture, services can be short-lived and their number of instances can vary over time, which means that static configuration is not an option. Fortunately, Prometheus server comes with built-in service discovery solutions and makes it pretty seamless to integrate with a number of platforms and tools, including [Kubernetes](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#<kubernetes_sd_config) and [Consul](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#<consul_sd_config). Clients have only one responsibility: make their metrics available for a Prometheus server to scrape. This is done by exposing an HTTP endpoint, usually `/metrics`, which returns the full list of metrics (with label sets) and their values. On the Prometheus server side, each target (statically defined, or dynamically discovered) is scraped at a regular interval (_scrape interval_). Each scrape reads the `/metrics` to get the current state of the client metrics, and persists the values in the Prometheus time-series database and you use a query language called **PromQL** in the Prometheus server to query metrics about the targets. Prometheus provides client-libraries in a number of languages that you can use to provide health-status of your application. But Prometheus is not only about application monitoring, you can use something called **Exporters** to monitor third-party systems (Such as a Linux Server, MySQL daemon etc). An Exporter is a piece of software that gets existing metrics from a third-party system and export them to the metric format that the Prometheus server can understand. The most common one is the [node exporter](https://github.com/prometheus/node_exporter), which can be installed on every machine to read system level metrics (cpu, memory, file system…) and expose them under a `/metrics` endpoint in the way Prometheus can scrape it.

Prometheus also has an AlertManager component, which can fire alerts via Email, Slack, Telegram, Pagerduty or other notification clients. The Alert Rules are defined in a file called alert.rules, via which the Prometheus server reads the alert configurations, then fires alerts at the necessary times via the Alert Manager component.

```
metric name and a set of key-value pairs, also known as labels
==>
<metric name>{<label name>=<label value>, ...} value [ timestamp ]
==>
http_requests_total{method="post",code="200"} 1027 1395066363000
```
Install
```
groupadd --system prometheus
useradd -s /sbin/nologin --system -g prometheus prometheus
mkdir /var/lib/prometheus
mkdir -p /etc/prometheus/
yum -y install wget
cd /tmp
export RELEASE=2.4.2
wget https://github.com/prometheus/prometheus/releases/download/v${RELEASE}/prometheus-${RELEASE}.linux-amd64.tar.gz
tar xvf prometheus-${RELEASE}.linux-amd64.tar.gz
cd prometheus-${RELEASE}.linux-amd64/
cp prometheus promtool /usr/local/bin/
cp prometheus promtool /usr/local/sbin/
cp -r consoles/ console_libraries/ /etc/prometheus/
```

```bash
# vi /etc/prometheus/prometheus.yml

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
```
promtool check config /etc/prometheus/prometheus.yml
chown -R prometheus:prometheus /etc/prometheus/*
chmod -R 775 /etc/prometheus/*
chown -R prometheus:prometheus /var/lib/prometheus/
```
```bash
# vi /etc/systemd/system/prometheus.service 

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
```bash
systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
```
```
http://<prometheus-ip>:9090/graph
http://<prometheus-ip>:9090/metrics
```

```
cd /etc/prometheus/
wget https://github.com/prometheus/node_exporter/releases/download/v0.17.0/node_exporter-0.17.0.linux-amd64.tar.gz
tar -xzvf node_exporter-0.17.0.linux-amd64.tar.gz
mv node_exporter-0.17.0.linux-amd64 node_exporter
```
```bash
# vi /etc/systemd/system/node_exporter.service

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
ExecStart=/etc/prometheus/node_exporter/node_exporter

[Install]
WantedBy=default.target
```
```
systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter
```

```bash
# vim /etc/prometheus/prometheus.yml
Under the 'scrape_config' line, add new job_name node_exporter by copy-pasting the configuration below.

  - job_name: 'node_exporter'  
    static_configs:  
      - targets: ['localhost:9100']
```
```
promtool check config /etc/prometheus/prometheus.yml
systemctl restart prometheus
```
Alerting in Prometheus is separated into two parts: 
1. Alert rules are defined in Prometheus configuration. If any alert condition hits, Prometheus send alert to AlertManager. 
2. AlertManager handles alerts sent by Prometheus server and notifies end user.

AlertManager manages alerts through its pipeline of silencing, inhibition, grouping and sending out notifications

Silencing is to mute alerts for a given time. Alerts are checked to match against active silent alerts, if a match is found then no notifications are sent. 
Inhibition is to suppress notifications for certain alerts if other alerts are already fired.
Grouping group alerts of similar nature into a single notification. This helps prevent firing multiple notifications simultaneously.
```
wget https://raw.githubusercontent.com/anirvanr/scripts/master/Prometheus-Alertmanager.sh
chmod +x Prometheus-Alertmanager.sh && ./Prometheus-Alertmanager.sh
```
Follow the instruction in script and restart alertmanager/Prometheus
Note:- You just have to replace the channel name and api_url of the Slack with your information.

Incoming Webhook Integration Setup :

https://api.slack.com/incoming-webhooks

https://medium.com/cyber-city-never-sleeps/slack-api-how-to-add-incoming-webhook-for-one-channel-only-420040a5f51d

As of now, we have AlertManager running on an instance at port 9093

We need to configure the Prometheus server so it can talk to AlertManager service. 
We are going to set up an alert rule file which defines all rules needed to trigger an alert.

rule_files:
   - "/etc/prometheus/alert.rules"

```bash
 #cat /etc/prometheus/alert.rules
groups:
    - name: alert.rules
      rules:
      - alert: high_cpu_load
        expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{job='node_exporter',mode="idle"}[5m])) * 100) > 75
        for: 10s
        labels:
          severity: critical
        annotations:
          summary: "Server under high load (current value is: {{ $value }})"
 ```
