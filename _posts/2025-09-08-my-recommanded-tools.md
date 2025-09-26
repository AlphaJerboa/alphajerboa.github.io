---
layout: post
title: "My recommended tools for sysadmin"
date: 2025-09-08 17:00:00 +0000
categories: [security, monitoring, virtualization, networking, automation, observability, iam, authentication, sso]
tags: [prometheus, grafana, tailscale, headscale, proxmox, docker, fleetdm, wazuh, lynis, openbao]
author: alphajerboa
license: "CC BY 4.0"
license_url: "https://creativecommons.org/licenses/by/4.0/"
excerpt: "My sysadmin toolbox"
---


## My Sysadmin Toolbox

As a systems administrator, I've spent years evaluating, deploying, and maintaining a wide range of infrastructure components. Over time, a handful of tools have consistently proven their worth—whether it's for virtualization, networking, observability, security, or automation. Below is a curated list of the solutions I rely on daily, along with brief notes on why they work so well (and a couple of caveats).

## [Proxmox VE](https://proxmox.com/en/products/proxmox-virtual-environment/overview) – The All‑In‑One Hypervisor

- **Feature‑rich yet open source** – Offers KVM‑based virtualization, LXC containers, and built‑in Ceph support.
- **Comparable to VMware** – Performance, live migration, and HA are on par with commercial offerings.
- **Integrated backup with PBS** – Proxmox Backup Server (PBS) provides fast, deduplicated VM snapshots.

**Gotchas**

- **Version consistency matters** – Mixing major Proxmox versions within a single cluster can cause subtle incompatibilities. Keep all nodes on the same release line.

## [Docker](https://www.docker.com/) – Production‑Ready Container Runtime

**Docker isn't just for development anymore.**

In production, it solves multiple challenges simultaneously: simplified OS updates, dependency management, application security through sandboxing, easy snapshots, and granular resource control. The operational benefits far outweigh the initial learning curve.

## [Headscale](https://github.com/juanfont/headscale) / [Tailscale](https://tailscale.com/) – WireGuard‑Powered Mesh Networking

Both projects turn WireGuard into a zero‑trust overlay network:

- **Headscale** – Self‑hosted coordination server, perfect for privacy‑focused environments.
- **Tailscale** – Hosted service with quick onboarding, ideal for mixed‑provider fleets.

Together they enable "micro‑firewalling" at the IP layer, provides seamless multi-provider infrastructure connectivity., simplifying remote access and service segmentation.

## [Prometheus](https://github.com/prometheus/prometheus) + [Grafana](https://github.com/grafana/grafana) – Observability Stack

- **Prometheus** scrapes metrics efficiently and stores them in a time‑series database. Prometheus excels in its simplicity and effectiveness.
- **Grafana** visualizes those metrics with beautiful dashboards.

## [FleetDM](https://github.com/fleetdm/fleet) – Scalable Osquery Management

Fleet provides a interface for running **osquery** across thousands of endpoints. You get real‑time visibility into system state, configuration drift, and security posture—all without installing heavy agents.

## [Wazuh](https://wazuh.com/) – Unified SIEM & FIM

Wazuh builds on the OSSEC engine, delivering security features including:

- **File Integrity Monitoring (FIM)** – Detects unauthorized changes in critical files.
- **Log collection & correlation** – Centralized SIEM capabilities with built‑in alerts.

## [Lynis](https://cisofy.com/lynis) - Simple auditing tool

Lynis is an auditing tool that performs in-depth security scans on Unix-based systems.

## [Keycloak](https://www.keycloak.org/) – Enterprise‑Grade SSO

Whether you need SAML, OIDC, 2FA, or traditional username/password authentication, Keycloak provides a unified solution that scales with your organization.

## [OpenBao](https://openbao.org/) – IaC‑Friendly Secrets Management

OpenBao (a community fork of HashiCorp Vault) acts as a **Swiss‑army knife** for authentication, dynamic secrets, and encryption‑as‑a‑service. Its API‑first design fits neatly into CI/CD pipelines, IaC scripts.

## [Ansible](https://www.ansible.com/) – Simple Yet Powerful Automation

Ansible is capable of incredibly advanced automation tasks. The agentless architecture and YAML-based playbooks make it accessible to teams while providing the power needed for complex infrastructure management.

