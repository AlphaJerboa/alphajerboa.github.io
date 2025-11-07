---
layout: post
title: "Systemd Timers vs Cron: Better Monitoring for Scheduled Tasks"
date: 2025-04-04 17:39:00 +0000
categories: [automation, monitoring, observability]
tags: [prometheus, grafana]
author: alphajerboa
license: "CC BY 4.0"
license_url: "https://creativecommons.org/licenses/by/4.0/"
excerpt: "Migrate your cron jobs as systemd timers to monitor their status"
---


## The Monitoring Problem with Cron

Traditional cron jobs present significant observability challenges. Cron provides no native success/failure status, execution duration metrics, or structured logging. Monitoring cron requires parsing syslog entries, wrapping jobs in custom scripts to emit metrics, or deploying separate monitoring agents. This approach is fragile and increases operational complexity.

## Systemd Timers: Built-in Observability

Systemd timers provide first-class integration with systemd's journal and service management, enabling direct Prometheus monitoring without additional tooling.

### Key Advantages

**Execution State Tracking**
Each timer triggers a systemd service unit whose state (active, failed, inactive) is queryable via systemd APIs. The node_exporter's systemd collector automatically exposes these states as Prometheus metrics without configuration.

**Automatic Failure Detection**
Failed service executions are tracked by systemd. The `node_systemd_unit_state` metric immediately reflects failures, enabling alerting on job success rates or consecutive failures through simple PromQL queries.

**Precise Timing Metrics**
Systemd records exact execution timestamps and durations in the journal. These are exposed via `node_systemd_timer_last_trigger_seconds` and related metrics, allowing monitoring of job latency, schedule drift, and SLA compliance.

**Resource Usage Visibility**
Systemd's cgroup integration tracks CPU, memory, and I/O per service execution. These metrics flow naturally into Prometheus via node_exporter, providing resource accountability without custom instrumentation.

**Structured Logging**
Journal integration provides structured metadata (unit name, exit code, timestamps) that's queryable and correlatable with metrics. This eliminates log parsing fragility inherent to cron monitoring.

## Implementation Pattern

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Database backup timer

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Database backup job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

### Activation Steps

```bash
# Reload systemd to recognize new units
systemctl daemon-reload

# Enable the timer to start on boot
systemctl enable backup.timer

# Start the timer immediately
systemctl start backup.timer

# Verify timer is active and check next execution
systemctl list-timers
```

The `list-timers` output shows all active timers with their next and last trigger times, making schedule verification immediate and transparent.

### Monitoring Queries

Enable node_exporter's systemd collector and query:
```promql
node_systemd_unit_state{name="backup.service",state="failed"}
```

## Operational Benefits

- **Zero instrumentation overhead**: No wrapper scripts or metric emission code required
- **Unified tooling**: Use `systemctl` and `journalctl` for both interactive debugging and monitoring
- **Dependency management**: Systemd units support explicit ordering and dependencies between jobs
- **Calendar expressions**: More flexible scheduling than cron syntax with built-in validation
- **Transparent scheduling**: `systemctl list-timers` provides instant visibility into all scheduled tasks

## Conclusion

Systemd timers transform scheduled task monitoring from an afterthought requiring custom tooling into a native capability. For Prometheus-based infrastructure, this integration eliminates monitoring gaps while reducing operational complexity. The transition cost is minimal—converting cron entries to timer units—while the observability gains are substantial.
