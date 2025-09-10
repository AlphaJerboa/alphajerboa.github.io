---
layout: post
title: "Monitoring Proxmox Mail Gateway Queue with Prometheus"
date: 2025-03-19 18:52:00 +0000
categories: [monitoring, automation]
tags: [proxmox mail gateway, prometheus, grafana]
author: alphajerboa
excerpt: "Tutorial to monitoring Proxmox Mail gateway queue with Prometheus"
---


## Overview

The monitoring solution consists of three components:
- A shell script that extracts Postfix queue metrics
- Prometheus Node Exporter for metric collection
- A cron job for automated metric generation

## Implementation

### Queue Monitoring Script

The following script generates Prometheus-compatible metrics for all Postfix queue instances:

```bash
{%raw%}
#!/bin/sh
# Based on https://github.com/anarcat/puppet-mtail/blob/master/files/postfix-queues-sizes
test -x /usr/sbin/postmulti || exit 0

queues="active bounce corrupt deferred flush hold incoming maildrop"
instances=$(/usr/sbin/postmulti -l | awk '{print $1}')

cat << EOF
# HELP postfix_queue_length Postfix mail queue.
# TYPE postfix_queue_length gauge
EOF

for instance in ${instances}; do
    if [ "x${instance}" = "x-" ]; then
        instance=postfix
    fi
    spool_dir=/var/spool/${instance}
    for queue in ${queues}; do
        test -d ${spool_dir}/${queue} || continue
        printf 'postfix_queue_length{postfix_instance="%s",queue="%s"} ' $instance $queue
        find "${spool_dir}/${queue}" -type f -print | wc -l
    done
done
exit 0
{%endraw%}
```


### Queue Types Monitored

- **active**: Messages currently being delivered
- **bounce**: Bounce notifications awaiting delivery
- **corrupt**: Messages with formatting issues
- **deferred**: Messages scheduled for retry
- **flush**: Messages awaiting flush operation
- **hold**: Administrative hold queue
- **incoming**: Newly received messages
- **maildrop**: Messages from local submission

### Add a cron job

The cron configuration ensures continuous monitoring:

```bash
*/5 * * * * root /root/tx-monitor-queues.sh > /var/lib/prometheus/node-exporter/postfix.prom
```

This executes the script every five minutes, writing metrics to the Node Exporter textfile directory.


## Prometheus Configuration

Add the PMG target to your Prometheus configuration:

```yaml
scrape_configs:
  - job_name: 'pmg-mail-gateway'
    static_configs:
      - targets: ['pmg-server:9100']
    scrape_interval: 30s
```

## Alerting Rules

Implement proactive monitoring with these Prometheus alerting rules:

```yaml
{%raw%}
groups:
  - name: postfix_queues
    rules:
      - alert: PostfixQueueHigh
        expr: postfix_queue_length > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Postfix queue length on {{ $labels.instance }}"
          description: "Queue {{ $labels.queue }} has {{ $value }} messages"
      
      - alert: PostfixDeferredQueueCritical
        expr: postfix_queue_length{queue="deferred"} > 500
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical deferred queue length"
          description: "Deferred queue has {{ $value }} messages, indicating delivery issues"
{%endraw%}
```


## Grafana Visualization

Create comprehensive dashboards using these PromQL queries:

```promql
# Total queue length across all instances
sum(postfix_queue_length) by (queue)

# Queue growth rate
rate(postfix_queue_length[5m])

# Instance-specific queue distribution
postfix_queue_length{postfix_instance="postfix"}
```

