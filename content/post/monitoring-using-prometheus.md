---
title: "Monitoring Using Prometheus"
date: 2017-09-06T18:09:42+05:30
draft: fasle
---

<details>
 <summary>*Setting up monitoring for Machine, Application, and URLs.*</summary>
  Using [prometheus](https://github.com/prometheus/prometheus/), [node_exporter](https://github.com/prometheus/node_exporter), [blackbox_exporter](https://github.com/prometheus/blackbox_exporter), [alertmanager](https://github.com/prometheus/alertmanager) and [grafana](https://github.com/grafana/grafana) for monitoring systems in non-containerized world.
</details>
<!--more-->

##### What to expect from Monitoring System
A Monitoring system gathers data from systems, store them for later viewing and trigger any alerts for unexpected behaviours/errors. As you can see, it requires four major parts:

1. Gathering component for accumulating data from various systems
2. Storage component for storing gathered data for future use
3. Viewing component for getting information from stored data
4. Alerting component for processing the stored data and triggering alerts based on certain conditions


##### Introduction to Prometheus
[Prometheus](https://prometheus.io/) is almost a complete monitoring tool, it has all four major components as briefed earlier. Below are the daemons used by these four components.

1. Gathering component  - `*_exporter`s ,  `prometheus`
2. Storage component    - `prometheus`
3. Viewing component    - `prometheus`'s web UI/API , `graphana`
4. Alerting component   - `prometheus` , `alertmanager`

###### Gathering component in Prometheus (How to gather data/metrics on Prometheus)
Prometheus uses [pull based model](https://prometheus.io/docs/introduction/faq/#why-do-you-pull-rather-than-push?) to gather metrics. What it does is that the main monitoring daemon will fetch metrics from the target systems that we want to monitor. The target system will run an agent that exposes the health metrics as a HTTP endpoint on the system, which would be scraped by `prometheus` daemon on specific time interval.

But it raises one problem. You need to specify every endpoint on `prometheus` daemon that you want to monitor. It can be a very tedious task if those endpoints are dynamically changing because of auto scaling or Kubernetes. Prometheus solves this problem by adding service discovery in `prometheus` daemon and this service discovery can gather endpoints from various cloud infrastructures(aws, azure, gce), Kubernetes and more.

Now lets focus on agents that will run on each monitored system. These are called `exporter` in Prometheus. Here is a [list](https://prometheus.io/docs/instrumenting/exporters/) of exporters available for different applications. Two in-house exporters from Prometheus are worhty of introduction:

- `node_exporter` exposes OS and hardware level metrics like memory, cpu, network, disk utilizations and more. This exporter is perfect for monitoring to find out if a given machine is up or not.
- `blackbox_exporter` exposes DNS, HTTP, HTTPS, DNS, TCP and ICMP level metrics. This exporter is perfect for monitoring to find out if a given site/endpoint is up or not.

###### Storage component in Prometheus
Prometheus has inbuilt timeseries database to store metrics.

###### Viewing component in Prometheus
Prometheus provides inbuilt web UI for visualising stored value.

Still most users prefer to use [Graphana](https://grafana.com/). Grafana is a very powerful and already has community shared dashboards for many exporters. Prometheus provides very powerful query language for fetching data to support the same.

###### Alerting component in Prometheus
You can use [Prometheus's expression language](https://prometheus.io/docs/querying/basics/) to create alert rules for error/unexpected state in `prometheus` daemon to trigger `alertmanager` daemon for further action.  `alertmanager` daemon then picks this data and triggers alert through email, Slack, Telegram, Pagerduty etc. It has many features like clubbing similar alerts, snooze, silent, labelling alerts and many more.

##### Making Monitor Server
Monitoring Server requires below five components, each component can be on the same machine or different ones:

1. `prometheus`
2. `node_exporter`
3. `blackbox_exporter`
4. `grafana`
5. `alertmanager`

###### Download and Install all above components
Download `prometheus` 1.x, `node_exporter`, `blackbox_exporter` and `grafana` from [here](https://prometheus.io/download/).

Now untar and extract binary from each downloaded file to `/usr/local/bin/`.

```sh
tar -xvzf prometheus-*.tar.gz -C /usr/local/bin --wildcards --no-anchored 'prometheus' --strip 1
tar -xvzf alertmanager-*.tar.gz -C /usr/local/bin --wildcards --no-anchored 'alertmanager' --strip 1
tar -xvzf node_exporter-*.tar.gz -C /usr/local/bin --wildcards --no-anchored 'node_exporter' --strip 1
tar -xvzf blackbox_exporter-*.tar.gz -C /usr/local/bin --wildcards --no-anchored 'blackbox_exporter' --strip 1
```

Download `grafana` from [here](https://grafana.com/grafana/download). Install it using package manager (like on Ubuntu):

```sh
sudo dpkg -i /opt/grafana_*.deb
```

- As its dangerous to run any program as root, create a non-root user

```sh
sudo adduser --shell /bin/nologin --no-create-home --disabled-login --gecos "Prometheus Monitoring User" prometheus
```

- Create a directory to put configuration file

```sh
sudo mkdir /etc/prometheus
```

###### Configuring `node_exporter`
This file configures `node_exporter` to make OS level metric stats available on 9100(default) port. `node_exporter` server will be started and managed by systemd. Create the `node_exporter` systemd unit file:

```sh
cat > node_exporter.service <<EOF
[Unit]
Description=Prometheus node exporter
After=network.target auditd.service

[Service]
User=prometheus
ExecStart=/usr/local/bin/node_exporter -collector.diskstats.ignored-devices="^(ram|loop|fd)\d+$" -collectors.enabled="diskstats,filesystem,loadavg,meminfo,stat,textfile,time,netdev" --collectors.enabled="conntrack,diskstats,entropy,filefd,filesystem,hwmon,loadavg,mdadm,meminfo,netdev,netstat,sockstat,stat,textfile,time,uname,vmstat,systemd"
Restart=on-failure

[Install]
WantedBy=default.target
EOF
```

Move it to the systemd system directory:

```sh
sudo mv node_exporter.service /etc/systemd/system/
```

Now enable and start it.

```sh
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter --no-pager
```

###### Configuring `blackbox_exporter`
This blackbox configuration file sets up the simplest HTTP monitoring. It will check returned status for URLs and raise error if it's not 2xx status.

```sh
cat > blackbox.yml <<EOF
modules:
  http_200_module:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: []          # Defaults to 2xx
      method: GET
      no_follow_redirects: false
      fail_if_ssl: false
      fail_if_not_ssl: false
      tls_config:
        insecure_skip_verify: false
      protocol: "tcp"                 # accepts "tcp/tcp4/tcp6", defaults to "tcp"
      preferred_ip_protocol: "ip4"    # used for "tcp", defaults to "ip6"
EOF

sudo mv balckbox.yml /etc/prometheus/
```

`blackbox_exporter` server will be started and managed by systemd. Create the `blackbox_exporter` systemd unit file and move it to the systemd system directory:

```sh
cat > blackbox_exporter.service <<EOF
[Unit]
Description=Prometheus blackbox exporter
After=network.target auditd.service

[Service]
User=prometheus
ExecStart=/usr/local/bin/blackbox_exporter -config.file=/etc/prometheus/blackbox.yml
Restart=on-failure

[Install]
WantedBy=default.target
EOF

sudo mv blackbox_exporter.service /etc/systemd/system/
```

Now enable and start it.

```sh
sudo systemctl daemon-reload
sudo systemctl enable blackbox_exporter
sudo systemctl start blackbox_exporter
sudo systemctl status blackbox_exporter --no-pager
```

###### Configuring `alertmanager`
This alertmanager configuration sends notification to different teams using email and slack based on the alert’s severity level. You can customize this configuration to fit your needs. The `alertname` used in this file like `CPU_Threshold_Exceeded` are setup from `prometheus`.

```
cat > alertmanager.yml <<EOF
global:
  # The smarthost and SMTP sender used for mail notifications.
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'xxxxx'
  # auth for Slack
  slack_api_url: 'https://hooks.slack.com/services/xxx/xxx/xxx'
# The directory from which notification templates are read.
templates:
- '/etc/alertmanager/template/*.tmpl'
# The root route on which each incoming alert enters.
route:
  group_by: ['alertname', 'cluster', 'service']   # The labels by which incoming alerts are grouped together.
  group_wait: 30s                                 # When a new group of alerts is created by an incoming alert, it waits at least this much to send the initial notification.
  group_interval: 5m                              # When the first notification was sent, wait until 'group_interval' to send a batch of new alerts that started firing for that group.
  repeat_interval: 5h                             # If an alert has successfully been sent, wait until 'repeat_interval' to resend them.
  receiver: 'slack_alerts_channel'                # A default receiver
  routes:                                         # This route performs a regular expression match on alert labels to catch alerts that are related to a list of services.
  - match:
      severity: 'warning'
    receiver: 'slack_alerts_channel'
  - match:
      severity: 'critical'
    receiver: 'devTeam'
  - match:
      severity: 'websiteDown'
    receiver: 'email_managers'
inhibit_rules:                                    # Inhibition rules allow to mute a set of alerts given that another alert is firing.
- source_match:
    alertname: 'CPU_Threshold_Exceeded'
  target_match:
    alertname: 'Instance_DOWN'
  equal: ['instance']
- source_match:
  alertname: 'Instance_Low_Memory'
  target_match:
  alertname: 'Instance_DOWN'
  equal: ['instance']
receivers:
- name: 'email_devTeam'
  email_configs:
  - to: 'dev@example.com'
  slack_configs:
  - send_resolved: true
    channel: '#emergency'
    text: "<!here> {{ range .Alerts }}\nInstance: {{ .Labels.instance }}\nsummary: {{ .Annotations.summary }}\ndescription: {{ .Annotations.description }}\n {{ end }}"
- name: 'slack_alerts_channel'
  slack_configs:
  - send_resolved: true
    channel: '#alerts'
    text: "description: {{ .CommonAnnotations.description }}\nsummary: {{ .CommonAnnotations.summary }}{{ range .Alerts }}\nInstance: {{ .Labels.instance }} {{ end }}"
- name: 'email_managers'
  email_configs:
  - to: 'leads@example.com, managers@example.com'

EOF

sudo mv alertmanager.yml /etc/prometheus/
```

alertmanager server will be started and managed by systemd. Create the alertmanager systemd unit file and move it to the systemd system directory:

```
cat > alertmanager.service <<EOF
[Unit]
Description=Prometheus Alert Manager service
After=network.target auditd.service

[Service]
Type=simple
User=prometheus
ExecStart=/usr/local/bin/alertmanager -config.file="/etc/prometheus/alertmanager.yml"
Restart=always

[Install]
WantedBy=default.target

EOF

sudo mv alertmanager.service /etc/systemd/system/
```

Now enable and start it.

```sh
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
sudo systemctl status alertmanager --no-pager
```

Now you can access an alertmanager dashboard on default 9093 port.

###### Configuring `prometheus` 1.x
`prometheus` is the main daemon that will interact with others(`alertmanager`,`blackbox_exporter`,`node_exporter`) to make an intergated monitoring solution. Lets start by configuring alert trigger rules. This file configures pretty generic alert rules, tune it according to your needs.

```
cat > alert.rules <<EOF
ALERT CPU_Threshold_Exceeded
  IF (100 * (1 - avg by(instance)(irate(node_cpu{mode='idle'}[1h])))) > 90
  FOR 1h
  LABELS { severity = "critical" }
  ANNOTATIONS {
    summary = "Instance {{ $labels.instance }} CPU usage is dangerously high",
    description = "This device's CPU usage has exceeded the thresold of 90% with a value of {{ $value }} for 1 hour."
  }

ALERT Instance_Low_Memory
  IF node_memory_MemAvailable < 268435456
  FOR 10m
  LABELS { severity = "critical" }
  ANNOTATIONS {
    summary = "Instance {{$labels.instance}}: memory low",
    description = "{{$labels.instance}} has less than 256M memory available"
  }

ALERT Disk_USAGE_Thresold_Exceeded
  IF node_filesystem_free{fstype!~"rootfs|tmpfs|fuse.lxcfs"} / node_filesystem_size{fstype!~"rootfs|tmpfs|fuse.lxcfs"} < 0.10
  FOR 15m
  LABELS { severity = "warning" }
  ANNOTATIONS {
    summary     = "Instance {{$labels.instance}}: disk {{$labels.instance}} filling up",
    description = "{{$labels.device}} mounted on {{$labels.mountpoint}} on {{$labels.instance}} is 90% filled."
  }

 ALERT Disk_Will_Fill_In_2_Hours
  IF predict_linear(node_filesystem_free{fstype!~"rootfs|tmpfs|fuse.lxcfs", job='aws'}[1h], 2*3600) < 0
  FOR 20m
  LABELS { severity = "warning" }
  ANNOTATIONS {
    summary     = "Instance {{$labels.instance}}: disk {{$labels.device}} filling up",
    description = "{{$labels.device}} mounted on {{$labels.mountpoint}} on {{$labels.instance}} will fill up within 2 hours."
  }

ALERT Site_DOWN
  IF probe_success < 1
  FOR 2m
  LABELS {
    severity="critical"
  }
  ANNOTATIONS {
    summary = "{{$labels.instance}} is down",
   description = "site {{$labels.instance}} is down from 2min."
  }
ALERT Site_DOWN
  IF probe_success < 1
  FOR 4m
  LABELS {
    severity="websiteDown"
  }
  ANNOTATIONS {
    summary = "{{$labels.instance}} is down",
    description = "site {{$labels.instance}} is down/unreachable."
  }

ALERT Instance_DOWN
  IF up == 0
  FOR 5m
  LABELS { severity = "critical" }
  ANNOTATIONS {
    summary = "Instance {{$labels.instance}} is down",
    description = "{{$labels.instance}} has been down for more than 8 minutes."
  }

ALERT Blackbox_Exporter_Down
  IF count(up{job="blackbox_site_monitoring"} == 1) < 1
  FOR 1m
  LABELS {
    severity="warning"
  }
  ANNOTATIONS {
    summary = "blackbox_exporter is not up",
    description = "blackbox_exporter is not up for 1min."
  }

EOF

mv alert.rules /etc/prometheus/
```

Below file is the `prometheus` configuration and it tells `prometheus`, where to scrape the metric data from, when to raise alerts etc. It configures `prometheus` to scrape aws instances having the tag `Monitoring` of value `enabled` and `example.com`, `app.example.com` domains to monitor. Customise it to tailor your needs.

```
cat > prometheus.yml <<EOF
global:
  scrape_interval:     15s  # By default, scrape targets every 15 seconds.
  evaluation_interval: 15s  # By default, scrape targets every 15 seconds.
  external_labels:
      monitor: 'example-prometheus-monitor'

rule_files:         # Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
   - "/etc/prometheus/alert.rules"

scrape_configs:       # A scrape configuration containing exactly one endpoint to scrape:
# The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'  # Here it's Prometheus itself.
    scrape_interval: 10s
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'aws'
    ec2_sd_configs:
    # The AWS Region.
    - region: us-west-2
      # The AWS API keys. If blank, the environment variables `AWS_ACCESS_KEY_ID`
      # and `AWS_SECRET_ACCESS_KEY` are used.
      refresh_interval: 10s
      port: 9100      # The port to scrape metrics from. If using the public IP address, this must instead be specified in the relabeling rule.
    relabel_configs:
    # only scrapes instances with tag "Monitoring" = "enabled", gives very flexible way to monitor and demonitor without touch this config file.
    - source_labels: [__meta_ec2_tag_monitoring]
      regex: 'NaN|Null|Nil'
      action: drop
    - source_labels: [__meta_ec2_tag_monitoring]
      regex: 'enabled'
      action: keep
    - source_labels: [__meta_ec2_tag_monitoring]
      regex: 'disabled'
      action: drop
    - source_labels: [__meta_ec2_tag_Name, __meta_ec2_private_ip]
      target_label: instance
      separator: ':'

  - job_name: blackbox_site_monitoring
    scrape_interval: 60s
    metrics_path: /probe
    params:
      module: [http_200_module]
    static_configs:
      - targets:
        - example.com
        - app.example.com
    relabel_configs:
      - source_labels: [__address__]
        regex: (.*)(:80)?
        target_label: __param_target
        replacement: ${1}
      - source_labels: [__param_target]
        regex: (.*)
        target_label: instance
        replacement: ${1}
      - source_labels: []
        regex: .*
        target_label: __address__
        replacement: 127.0.0.1:9115    # Blackbox exporter URL.

EOF

mv prometheus.yml /etc/prometheus/
```

Create a directory for prometheus to store data.

```sh
sudo mkdir /data/prometheus
chown prometheus. /data/prmetheus
```

prometheus server will be started and managed by systemd. Create the prometheus systemd unit file and move it to the systemd system directory:

```
cat > prometheus.service <<EOF
[Unit]
Description=Prometheus Monitoring service
After=network.target auditd.service

[Service]
Type=simple
WorkingDirectory=/data/prometheus
User=prometheus
ExecStart=/opt/prometheus -config.file="/etc/prometheus/prometheus.yml" -alertmanager.url="http://localhost:9093" -storage.local.retention="1440h"
Restart=always

[Install]
WantedBy=default.target

EOF

mv prometheus.service /etc/systemd/system/
```

Now enable and start it.

```sh
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus --no-pager
```

###### Configuring Grafana
As grafana is already installed and running. You can access grafana on port `3000`. Configure grafana for prometheus using [this guide](http://docs.grafana.org/features/datasources/prometheus/#using-prometheus-in-grafana).
Now you can configure graphs/dashboards as per your need. But for getting started, import below dashboards using [this guide](http://docs.grafana.org/reference/export_import/#importing-a-dashboard):

- https://grafana.com/dashboards/22
- https://grafana.com/dashboards/405

These two dashboards are excellent for visualizing `node_exporter` metrics. You can find out more pre-made dashboards [at here](https://grafana.com/dashboards).
