---
layout: post
title: "Implementing Prometheus Security Compliance Monitoring for CrowdStrike Falcon Agents"
date: 2025-09-04 17:00:00 +0000
categories: [security, monitoring, prometheus]
tags: [prometheus, crowdstrike, falcon, compliance, alerting]
author: alphajerboa
excerpt: "Compliance monitoring using Prometheus."
---

## Introduction

This post demonstrates how to configure Prometheus to automatically detect servers missing CrowdStrike Falcon endpoint protection agents.

## Implementation

### PromQL Query

The core compliance check uses the following PromQL expression:

```promql
group by (instance) (
  up{job="servers"}==1 
  unless on(instance) 
  node_systemd_unit_state{name="falcon-sensor.service", state="active"}
)
```

**Query breakdown:**
- `up{job="servers"}==1`: Identifies all actively monitored servers
- `node_systemd_unit_state{name="falcon-sensor.service", state="active"}`: Selects instances where Falcon sensor service is active
- `unless on(instance)`: Excludes instances that match the condition
- `group by (instance)`: Organizes results by host instance

### Alerting Rule Configuration

Add this rule to your Prometheus configuration:

```yaml
{%raw%}
groups:
  - name: security_compliance
    rules:
      - alert: CrowdStrikeFalconAgentDown
        expr: |
          group by (instance) (
            up{job="servers"}==1 
            unless on(instance) 
            node_systemd_unit_state{name="falcon-sensor.service", state="active"}
          )
        for: 5m
        labels:
          severity: critical
          compliance: security
          service: crowdstrike-falcon
        annotations:
          summary: "CrowdStrike Falcon agent not running on {{ $labels.instance }}"
          description: |
            The CrowdStrike Falcon endpoint protection agent is not active on 
            {{ $labels.instance }}. This requires immediate attention.
          remediation: "Verify falcon-sensor.service status and restart if necessary"
{%endraw%}
```


### Advanced Patterns

**Multi-environment support:**
```promql
{%raw%}
group by (instance, environment) (
  up{job="servers", environment=~"production|staging"}==1 
  unless on(instance) 
  node_systemd_unit_state{name="falcon-sensor.service", state="active"}
)
{%endraw%}
```

**Grace period for new systems:**
```promql
{%raw%}
group by (instance) (
  up{job="servers"}==1 
  unless on(instance) 
  node_systemd_unit_state{name="falcon-sensor.service", state="active"}
) 
and on(instance) 
(time() - node_boot_time_seconds) > 3600
{%endraw%}
```

**Handle exceptions:**
```promql
{%raw%}
group by (instance) (
  up{job="servers", falcon_required!="false"}==1 
  unless on(instance) 
  node_systemd_unit_state{name="falcon-sensor.service", state="active"}
)
{%endraw%}
```

### Compliance Metrics

Monitor overall compliance status:

```promql
{%raw%}
# Total managed servers
count(up{job="servers"}==1)

# Servers with active Falcon agents
count(node_systemd_unit_state{name="falcon-sensor.service", state="active"})

# Compliance percentage
(count(node_systemd_unit_state{name="falcon-sensor.service", state="active"}) / count(up{job="servers"}==1)) * 100
{%endraw%}
```
