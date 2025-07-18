---
layout: post
title:  Visualizing Honeypot Attacks in Azure with Cowrie, Loki, and Grafana Dashboards
description: This post shows how to deploy a secure Cowrie SSH/Telnet honeypot on Azure with Promtail, Loki, and Grafana for log monitoring and analysis.
date:   2025-07-18 21:01:35 +0300
image:  '/images/post-1-image1.png'
tags:   [work, technology]
---

# Introduciton
Modern infrastructure is under constant threat from automated scans, brute-force attempts, and targeted attacks. One effective way to study this behavior and improve your security posture is by deploying a honeypot—a system designed to attract and log unauthorized access attempts.

In this post, I’ll demonstrate how to deploy a Cowrie honeypot on an Azure virtual machine, capture SSH/Telnet activity in real time, and centralize the logs using Promtail, Loki, and Grafana. This setup not only helps visualize attacker behavior but also integrates seamlessly with modern observability pipelines.

We’ll cover:

* Secure deployment of Cowrie on Azure
* Forwarding logs with Promtail to Loki
* Visualizing activity in Grafana dashboards

# Solution Diagram

![post-1-image2.png]({{site.baseurl}}/images/post-1-image2.png)
*Diagram of Architecture

# NSG Rules

| Name                 | Indicators                 |
| -------------------- | -------------------------- |
| Allow\_SSH\_2222     | Port 2222 TCP, Source: Any |
| Allow\_Telnet\_23    | Port 23 TCP, Source: Any   |
| Allow\_Loki\_3100    | Port 3100 TCP, Source: Any |
| Allow\_Grafana\_3000 | Port 3000 TCP, Source: Any |


# Install Dependencies

Update system and install prerequisites
{% highlight bash %}
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-venv python3-pip git curl wget unzip software-properties-common
{% endhighlight %}


# Installing and Configuring Cowrie

1. Clone Cowrie repository and create Python virtual environment
{% highlight bash %}
cd /opt
sudo git clone https://github.com/cowrie/cowrie.git
cd cowrie
sudo python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
{% endhighlight %}

2. Copy sample configuration files and create necessary directories
{% highlight bash %}
cp etc/cowrie.cfg.dist etc/cowrie.cfg
cp etc/userdb.example etc/userdb.txt
mkdir -p var/log/cowrie
mkdir -p var/lib/cowrie
{% endhighlight %}

3. Configure Cowrie ports (SSH on 2222)
{% highlight bash %}
nano cowrie.cfg
{% endhighlight %}

Modify in *cowrie.cfg*

{% highlight bash %}
[ssh]
listen_endpoints = tcp:2222
{% endhighlight %}

3. Start Cowrie honeypot
{% highlight bash %}
bin/cowrie start
{% endhighlight %}

# Install Loki

1. Download and install Loki binary
{% highlight bash %}
wget https://github.com/grafana/loki/releases/download/v3.5.2/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
chmod +x loki-linux-amd64
sudo mv loki-linux-amd64 /usr/local/bin/loki
{% endhighlight %}

2. Create Loki configuration file /etc/loki-config.yaml
{% highlight yaml %}
auth_enabled: false
server:
http_listen_port: 3100
http_listen_address: 0.0.0.0

limits_config:
allow_structured_metadata: false

ingester:
lifecycler:
ring:
kvstore:
store: inmemory
replication_factor: 1
chunk_idle_period: 5m
max_chunk_age: 1h
chunk_retain_period: 30s

schema_config:
configs:
- from: 2020-10-24
store: boltdb-shipper
object_store: filesystem
schema: v11
index:
prefix: index_
period: 24h

storage_config:
boltdb_shipper:
active_index_directory: /var/loki/index
cache_location: /var/loki/cache

filesystem:
directory: /var/loki/chunks
{% endhighlight %}

3. Create directories for Loki storage and set permissions
{% highlight bash %}
sudo mkdir -p /var/loki/index /var/loki/cache /var/loki/chunks
sudo chown -R $USER:$USER /var/loki
{% endhighlight %}

4. Run Loki
{% highlight bash %}
sudo loki -config.file=/etc/loki-config.yaml
{% endhighlight %}

# Install Promtail

1. Download and install Promtail binary
{% highlight bash %}
wget https://github.com/grafana/loki/releases/download/v3.5.2/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
chmod +x promtail-linux-amd64
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
{% endhighlight %}

2. Create Promtail configuration file /etc/promtail-config.yaml
{% highlight yaml %}
server:
http_listen_port: 9080
http_grpc_port: 0

positions:
filename: /tmp/positions.yaml

clients:

url: http://localhost:3100/loki/api/v1/push

scrape_configs:

job_name: cowrie
static_configs:

targets:

localhost
labels:
job: cowrie
path: /opt/cowrie/var/log/cowrie/cowrie.json
{% endhighlight %}

3. Run Promtail
{% highlight bash %}
sudo promtail -config.file=/etc/promtail-config.yaml
{% endhighlight %}

# Install Grafana

Add Grafana repository and install Grafana
{% highlight bash %}
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt update
sudo apt install grafana -y
{% endhighlight %}

Enable and start Grafana service
{% highlight bash %}
sudo systemctl enable --now grafana-server
{% endhighlight %}

Open Grafana UI at http://<your-vm-ip>:3000

**Default login: admin / admin**

# Configure Loki Datasource in Grafana

In Grafana UI:

Go to Configuration → Data Sources

Click Add data source

Select Loki

Set URL to http://<your-vm-ip>:3100

Click Save & Test (should say datasource is working)

# Create Grafana Dashboard

Go to Explore tab

Select Loki datasource

Run query {job="cowrie"} or {} to see logs

Create dashboards with panels based on logs and metrics as needed
