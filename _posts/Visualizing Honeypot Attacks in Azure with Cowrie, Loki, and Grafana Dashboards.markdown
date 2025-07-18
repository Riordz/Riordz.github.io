---
layout: post
title:  Visualizing Honeypot Attacks in Azure with Cowrie, Loki, and Grafana Dashboards
description: In this post, I walk through the deployment of a secure, low-interaction SSH/Telnet honeypot using Cowrie on an Azure virtual machine. The honeypot is instrumented with Promtail for log forwarding, Loki for log aggregation, and Grafana for real-time visualization and analysis of attacker behavior. The setup emphasizes secure deployment practices in a cloud environment, including outbound traffic restrictions, network isolation, and proper logging hygiene. This solution provides a scalable, cloud-native approach to threat intelligence and intrusion monitoring using open-source observability tools.
date:   2025-07-18 21:01:35 +0300
image:  '/images/post-1-image1.png'
tags:   [work, technology]
---
Modern infrastructure is under constant threat from automated scans, brute-force attempts, and targeted attacks. One effective way to study this behavior and improve your security posture is by deploying a honeypot—a system designed to attract and log unauthorized access attempts.

In this post, I’ll demonstrate how to deploy a Cowrie honeypot on an Azure virtual machine, capture SSH/Telnet activity in real time, and centralize the logs using Promtail, Loki, and Grafana. This setup not only helps visualize attacker behavior but also integrates seamlessly with modern observability pipelines.

We’ll cover:

* Secure deployment of Cowrie on Azure
* Forwarding logs with Promtail to Loki
* Visualizing activity in Grafana dashboards

# Solution Diagram

![post-1-image2.png]({{site.baseurl}}/images/post-1-image2.png)
*Diagram of Architecture

# Installing and Configuring Cowrie


{% highlight bash %}

Install Prequisites

sudo apt update && sudo apt install python3 python3-venv python3-pip git
{% endhighlight bash %}

{% highlight bash %}
Clone Clowrie Repo

git clone https://github.com/cowrie/cowrie.git
{% endhighlight bash %}

{% highlight bash %}
Create Python virtual environment and activate it

cd cowrie && python3 -m venv cowrie-env && source cowrie-env/bin/activate
{% endhighlight bash %}

{% highlight bash %}
Install Requirements

pip install --upgrade pip && pip install -r requirements.txt
{% endhighlight bash %}

{% highlight bash %}
Configure Cowrie

Edit cowrie.cfg in cowrie/etc/ to set logging paths, SSH ports etc.
{% endhighlight bash %}

{% highlight bash %}
Start Cowrie Manually

bin/cowrie start
{% endhighlight bash %}

{% highlight bash %}
Set up systemd service

Create /etc/systemd/system/cowrie.service with exec start pointing to cowrie start script
{% endhighlight bash %}

{% highlight bash %}
Verify logs are generated

tail -f cowrie/var/log/cowrie/cowrie.log
{% endhighlight bash %}



Sic igitur in homine perfectio ista in eo potissimum, quod est optimum, id est in virtute, laudatur. Natura sic ab iis investigata est, ut nulla pars caelo, mari, terra, ut poëtice loquar, praetermissa sit. Eadem nunc mea adversum te oratio est. Mihi quidem Homerus huius modi quiddam vidisse videatur in iis, quae de Sirenum cantibus finxerit. Neque enim disputari sine reprehensione nec cum iracundia aut pertinacia recte disputari potest. An, partus ancillae sitne in fructu habendus, disseretur inter principes civitatis. Put in geometria, prima si dederis, danda sunt omnia. Longum est enim ad omnia respondere.

<p><iframe src="https://www.youtube.com/embed/QyQ85DEVpbc" frameborder="0" allowfullscreen></iframe></p>

Qua ex cognitione facilior facta est investigatio rerum occultissimarum. Negat enim tenuissimo victu, id est contemptissimis escis et potionibus, minorem voluptatem percipi quam rebus exquisitissimis ad epulandum. Non enim iam stirpis bonum quaeret, sed animalis. Qui autem esse poteris, nisi te amor ipse ceperit? Sic igitur in homine perfectio ista in eo potissimum, quod est optimum, id est in virtute, laudatur disputari sine.

Sin tantum modo ad indicia veteris memoriae cognoscenda, curiosorum. Haec et tu ita posuisti, et verba vestra sunt. Idemne potest esse dies saepius, qui semel fuit? Ampulla enim sit necne sit, quis non iure optimo irrideatur, si laboret? Ego vero volo in virtute vim esse quam maximam; Serpere anguiculos, nare anaticulas, evolare merulas, cornibus uti videmus boves, nepas aculeis. Archytam? Qua ex cognitione facilior facta est investiga.