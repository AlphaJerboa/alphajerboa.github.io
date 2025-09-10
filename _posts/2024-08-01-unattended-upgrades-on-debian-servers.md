---
layout: post
title: "Enabling Unattended Upgrades on Debian Servers"
date: 2024-08-01 17:55:00 +0000
categories: [security, automation]
tags: [ansible, prometheus, grafana]
author: alphajerboa
excerpt: "Turn unattended upgrades on Debian systems"
---


Keeping Debian servers updated with security patches is critical for maintaining system security. This guide covers setting up automated security updates using `unattended-upgrades`, automating deployment with Ansible, and monitoring pending updates with Prometheus and Grafana.


## Ansible Automation

Automate the deployment across multiple servers using this playbook:

### Main Task (`tasks/main.yml`)

```yaml
{%raw%}
---
- name: Install unattended-upgrades packages
  ansible.builtin.apt:
    name:
    - unattended-upgrades
    state: present
  become: true

- name: Create required directory
  ansible.builtin.file:
    path: "/etc/systemd/system/apt-daily-upgrade.timer.d/"
    state: directory
  become: true

- name: Copy unattended configuration files
  copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    mode: 0640
    group: root
  with_items:
    - src: "files/50unattended-upgrades"
      dest: "/etc/apt/apt.conf.d/50unattended-upgrades"
    - src: "files/apt-daily-upgrade.timer.override.conf"
      dest: "/etc/systemd/system/apt-daily-upgrade.timer.d/override.conf"
  become: true
  tags: install
  notify:
    - Reload systemd
{%endraw%}
```

### Handler (`handlers/main.yml`)

```yaml
---
- name: Reload systemd
  ansible.builtin.shell:
    cmd: systemctl daemon-reload
  become: true
```

### Required Files

Create the required files:
```
─── files/
    ├── 50unattended-upgrades
    └── apt-daily-upgrade.timer.override.conf
```

The file files/apt-daily-upgrade.timer.override.conf can be used to override default schedule
```
[Timer]
OnCalendar=
OnCalendar=Mon *-*-* 06:00:00
```

Finetune the file 50unattended-upgrades, specially the section Unattended-Upgrade::Package-Blacklist in a production environment, here is an extract:
```
[...]
Unattended-Upgrade::Package-Blacklist {

    // Prevent docker containers restart
    "docker-ce";
    "docker-ce-cli";
    "containerd.io";
    "docker-compose-plugin";
    "docker-buildx-plugin";
    "docker.io";

    // Prevent critical packages autoupgrade to avoid system instability, keep it manual
    "systemd";
    "dbus";
    "udev";
    "openssh-server";

    // Prevent wazuh agent version mismach with server
    "wazuh-agent";

};
[...]
```

## Monitoring with Prometheus

### Prometheus Query

Monitor servers with pending updates:

```promql
apt_upgrades_pending > 0
```

### Prometheus Alerts

Add these alert rules to your Prometheus configuration to get notified about pending updates and reboot requirements:

```yaml
{%raw%}
groups:
  - name: debian_updates
    rules:
      - alert: PendingSecurityUpdates
        expr: apt_upgrades_pending > 0
        for: 24h
        labels:
          severity: warning
        annotations:
          summary: "Server {{ $labels.instance }} has pending updates"
          description: "{{ $labels.instance }} has {{ $value }} pending security updates for more than 24 hours"

      - alert: RebootRequired
        expr: node_reboot_required > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Server {{ $labels.instance }} requires reboot"
          description: "{{ $labels.instance }} requires a reboot to complete security updates"

      - alert: HighPendingUpdates
        expr: apt_upgrades_pending > 20
        for: 1h
        labels:
          severity: critical
        annotations:
          summary: "Server {{ $labels.instance }} has many pending updates"
          description: "{{ $labels.instance }} has {{ $value }} pending updates, immediate attention required"
{%endraw%}
```

### Grafana Dashboard

Import this dashboard configuration for a clean view of pending updates:

```json
{%raw%}
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {
          "type": "grafana",
          "uid": "-- Grafana --"
        },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "panels": [
    {
      "datasource": {
        "type": "prometheus",
        "uid": "YOUR_PROMETHEUS_UID"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "custom": {
            "align": "auto",
            "cellOptions": {
              "type": "auto"
            },
            "filterable": true,
            "inspect": false
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              },
              {
                "color": "#EAB839",
                "value": 10
              },
              {
                "color": "red",
                "value": 20
              }
            ]
          }
        },
        "overrides": [
          {
            "matcher": {
              "id": "byName",
              "options": "instance"
            },
            "properties": [
              {
                "id": "displayName",
                "value": "Server"
              }
            ]
          },
          {
            "matcher": {
              "id": "byName",
              "options": "Value"
            },
            "properties": [
              {
                "id": "displayName",
                "value": "Pending Updates"
              }
            ]
          }
        ]
      },
      "gridPos": {
        "h": 12,
        "w": 24,
        "x": 0,
        "y": 0
      },
      "id": 1,
      "options": {
        "cellHeight": "sm",
        "footer": {
          "countRows": true,
          "enablePagination": true,
          "fields": "",
          "reducer": ["sum"],
          "show": true
        },
        "showHeader": true,
        "sortBy": [
          {
            "desc": true,
            "displayName": "Pending Updates"
          }
        ]
      },
      "pluginVersion": "11.6.1",
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "YOUR_PROMETHEUS_UID"
          },
          "editorMode": "code",
          "exemplar": false,
          "expr": "apt_upgrades_pending > 0",
          "format": "table",
          "instant": true,
          "legendFormat": "__auto",
          "range": false,
          "refId": "A"
        }
      ],
      "title": "🔄 Servers with Pending Updates",
      "type": "table"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "YOUR_PROMETHEUS_UID"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "vis": false
            }
          },
          "mappings": [],
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 12
      },
      "id": 2,
      "options": {
        "displayLabels": [],
        "legend": {
          "displayMode": "table",
          "placement": "right",
          "values": ["value"]
        },
        "pieType": "pie",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "tooltip": {
          "mode": "single"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "YOUR_PROMETHEUS_UID"
          },
          "editorMode": "code",
          "expr": "sum by (instance) (apt_upgrades_pending)",
          "legendFormat": "{{instance}}",
          "range": true,
          "refId": "A"
        }
      ],
      "title": "📊 Update Distribution by Server",
      "type": "piechart"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "YOUR_PROMETHEUS_UID"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green"
              },
              {
                "color": "#EAB839",
                "value": 5
              },
              {
                "color": "red",
                "value": 15
              }
            ]
          },
          "unit": "short"
        }
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 12
      },
      "id": 3,
      "options": {
        "colorMode": "background",
        "graphMode": "area",
        "justifyMode": "center",
        "orientation": "horizontal",
        "reduceOptions": {
          "calcs": ["lastNotNull"],
          "fields": "",
          "values": false
        },
        "textMode": "value_and_name",
        "wideLayout": true
      },
      "pluginVersion": "11.6.1",
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "YOUR_PROMETHEUS_UID"
          },
          "editorMode": "code",
          "expr": "sum(apt_upgrades_pending)",
          "legendFormat": "Total Pending Updates",
          "range": true,
          "refId": "A"
        }
      ],
      "title": "📈 Total Updates Across All Servers",
      "type": "stat"
    }
  ],
  "schemaVersion": 41,
  "tags": ["debian", "updates", "security"],
  "time": {
    "from": "now-1h",
    "to": "now"
  },
  "timezone": "Europe/Paris",
  "title": "🔒 Debian Security Updates Dashboard",
  "uid": "debian_updates_dashboard",
  "version": 1
}
{%endraw%}
```


## Troubleshooting

Check service status:

```bash
sudo systemctl status unattended-upgrades
sudo systemctl list-timers apt-daily-upgrade.timer
```

View recent upgrade logs:

```bash
sudo tail -f /var/log/unattended-upgrades/unattended-upgrades.log
```

