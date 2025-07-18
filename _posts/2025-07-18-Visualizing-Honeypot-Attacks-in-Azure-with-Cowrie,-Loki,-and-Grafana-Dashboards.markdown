---
layout: post
title: Visualizing Honeypot Attacks in Azure with Cowrie, Loki, and Grafana Dashboards
description: A step-by-step guide to deploying a secure Cowrie SSH/Telnet honeypot on Azure, forwarding logs with Promtail and Loki, and visualizing attacker activity in Grafana.
date: 2025-07-18 21:01:35 +0300
image: '/images/post-1-image1.png'
tags: [incident-response]
---

# Introduction

Modern infrastructure is constantly targeted by automated scans, brute-force attacks, and sophisticated adversaries. Deploying a honeypot—a system designed to attract and record unauthorized access attempts—is an effective way to study attacker behavior and improve your security posture.

In this post, I will walk you through deploying a **Cowrie** honeypot on an Azure VM, capturing SSH and Telnet activity in real time, and centralizing logs using **Promtail**, **Loki**, and **Grafana**. This powerful observability stack enables you to monitor, analyze, and visualize attacker actions effortlessly.

We will cover:

- Securely deploying Cowrie on Azure  
- Forwarding logs from Cowrie with Promtail to Loki  
- Building insightful Grafana dashboards to visualize attack patterns  

# Architecture Diagram

![post-1-image2.png]({{site.baseurl}}/images/post-1-image2.png)  
*High-level architecture overview*

# Azure NSG Rules

| Name                | Details                      |
|---------------------|------------------------------|
| Allow_SSH_2222      | TCP port 2222, Source: Any   |
| Allow_Telnet_23     | TCP port 23, Source: Any     |
| Allow_Loki_3100     | TCP port 3100, Source: Any   |
| Allow_Grafana_3000  | TCP port 3000, Source: Any   |

# Install Dependencies

Update your system and install required packages:  
{% highlight bash %}
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-venv python3-pip git curl wget unzip software-properties-common
{% endhighlight %}

# Installing and Configuring Cowrie

Clone the Cowrie repository and set up a Python virtual environment:  
{% highlight bash %}
cd /opt
sudo git clone https://github.com/cowrie/cowrie.git
cd cowrie
sudo python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
{% endhighlight %}

Copy sample configuration files and create necessary directories:  
{% highlight bash %}
cp etc/cowrie.cfg.dist etc/cowrie.cfg
cp etc/userdb.example etc/userdb.txt
mkdir -p var/log/cowrie var/lib/cowrie
{% endhighlight %}

Edit the Cowrie configuration to change SSH port to 2222:  
{% highlight bash %}
nano etc/cowrie.cfg
{% endhighlight %}

Modify the following section in *cowrie.cfg*:  
{% highlight bash %}
[ssh]
listen_endpoints = tcp:2222
{% endhighlight %}

Start the Cowrie honeypot:  
{% highlight bash %}
bin/cowrie start
{% endhighlight %}

# Install Loki

Download and install Loki:  
{% highlight bash %}
wget https://github.com/grafana/loki/releases/download/v3.5.2/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
chmod +x loki-linux-amd64
sudo mv loki-linux-amd64 /usr/local/bin/loki
{% endhighlight %}

Create Loki config file `/etc/loki-config.yaml`:  
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

Create Loki storage directories and set ownership:  
{% highlight bash %}
sudo mkdir -p /var/loki/index /var/loki/cache /var/loki/chunks
sudo chown -R $USER:$USER /var/loki
{% endhighlight %}

Start Loki:  
{% highlight bash %}
sudo loki -config.file=/etc/loki-config.yaml
{% endhighlight %}

# Install Promtail

Download and install Promtail:  
{% highlight bash %}
wget https://github.com/grafana/loki/releases/download/v3.5.2/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
chmod +x promtail-linux-amd64
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
{% endhighlight %}

Create Promtail config `/etc/promtail-config.yaml`:  
{% highlight yaml %}
server:
  http_listen_port: 9080
  http_grpc_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
  - job_name: cowrie
    static_configs:
      - targets:
          - localhost
        labels:
          job: cowrie
          __path__: /opt/cowrie/var/log/cowrie/cowrie.json
{% endhighlight %}

Run Promtail:  
{% highlight bash %}
sudo promtail -config.file=/etc/promtail-config.yaml
{% endhighlight %}

# Install Grafana

Add the Grafana repository and install Grafana:  
{% highlight bash %}
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt update
sudo apt install grafana -y
{% endhighlight %}

Enable and start Grafana service:  
{% highlight bash %}
sudo systemctl enable --now grafana-server
{% endhighlight %}

Access Grafana UI at: http://<your-vm-ip>:3000

**Default login credentials:**  
Username: `admin`  
Password: `admin`

# Configure Loki Datasource in Grafana

In the Grafana UI:  

1. Go to **Configuration → Data Sources**  
2. Click **Add data source**  
3. Select **Loki**  
4. Set URL to `http://<your-vm-ip>:3100`  
5. Click **Save & Test** (should confirm datasource is working)

![3.png]({{site.baseurl}}/images/3.png)

# Example Loki Queries for Grafana Panels

| Panel Name           | Panel Type   | Purpose                          | Example Loki Query                                           |
|----------------------|--------------|---------------------------------|-------------------------------------------------------------|
| Total Login Attempts  | Stat         | Count total login attempts       | `count_over_time({filename=~".*cowrie.log"} |= "login attempt" [1h])` |
| Successful Logins     | Stat         | Count successful logins          | `{filename=~".*cowrie.log"} |= "login attempt [success]"`   |
| Top Usernames        | Table        | List usernames by login attempts | `sum by (user) (count_over_time({filename=~".*cowrie.log"} | pattern "<_> login attempt [<result>] for user <user> from <_>" [1h]))` |
| Active Sessions       | Time series  | Sessions open/closed over time   | `count_over_time({filename=~".*cowrie.log"} |= "New connection" [5m])` and `count_over_time({filename=~".*cowrie.log"} |= "Session Closed" [5m])` |
| Command Executions    | Table/Logs   | Commands executed by attackers   | `{filename=~".*cowrie.log"} |= "CMD"`                       |
| Failed Logins         | Stat/Table   | Count or list of failed logins   | `{filename=~".*cowrie.log"} |= "login attempt [failed]"`   |

# Final Result

![4.png]({{site.baseurl}}/images/4.png)
