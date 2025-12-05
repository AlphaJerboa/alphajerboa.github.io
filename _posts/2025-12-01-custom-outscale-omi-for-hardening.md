---
layout: post
title: "Building a Hardened Debian 12 OMI on OUTSCALE with Custom Partitioning"
date: 2025-12-01 06:25:00 +0000
categories: [security, automation, cloud]
tags: [packer, hardening, outscale]
author: alphajerboa
license: "CC BY 4.0"
license_url: "https://creativecommons.org/licenses/by/4.0/"
excerpt: "Creating a custom Debian 12 OUTSCALE Machine Image (OMI) with a security-focused partition scheme using HashiCorp Packer"
---

## Introduction

This article demonstrates how to create a custom Debian 12 OUTSCALE Machine Image (OMI) with a security-focused partition scheme using HashiCorp Packer.

## Custom Partitioning for Hardening

Standard cloud images typically use a single root partition for simplicity. However, this approach exposes several security risks:

- **No filesystem isolation**: Critical directories like `/var`, `/tmp`, and `/home` share the same partition, allowing resource exhaustion attacks
- **Unrestricted execution**: Without `noexec` flags, attackers can execute malicious binaries from temporary directories
- **Device file exposure**: The `nodev` option prevents creation of device files that could be exploited
- **Setuid risks**: The `nosuid` flag prevents privilege escalation through setuid/setgid executables

This articles helps creating a dedicated `/var` partition and applies security mount options across multiple mount points.

## Architecture Overview

Our hardened Debian 12 OMI includes:

- **Root partition** (`/dev/sda1`): 20GB for OS and core system files
- **Dedicated /var partition** (`/dev/xvdb`): 20GB separate volume for variable data
- **Bind mounts with security flags**: Additional protection for `/var/log`, `/var/tmp`, `/srv`, `/home`, and `/tmp`

## Prerequisites

Before beginning, ensure you have:

- OUTSCALE account with API credentials
- Packer installed (version supporting OUTSCALE plugin ≥ 1.0.0)
- SSH key pair registered in OUTSCALE (referenced as "admin public key")
- SSH agent running with your private key loaded, or private key file available
- Appropriate IAM permissions to create VMs, volumes, and OMIs

## Packer Configuration Breakdown

### Variables and Authentication

```hcl
variable "osc_access_key" {
  type        = string
  description = "Outscale access key"
  default     = env("OUTSCALE_ACCESSKEYID")
}

variable "osc_secret_key" {
  type        = string
  description = "Outscale secret key"
  default     = env("OUTSCALE_SECRETKEYID")
}

variable "region" {
  type        = string
  description = "Outscale region"
  default     = "cloudgouv-eu-west-1"
}
```

The configuration uses environment variables for credentials, following security best practices by avoiding hardcoded secrets. Set these before running Packer:

```bash
export OUTSCALE_ACCESSKEYID="your_access_key"
export OUTSCALE_SECRETKEYID="your_secret_key"
```

### Source Builder Configuration

```hcl
source "outscale-bsu" "debian12-custom" {
  region               = var.region
  vm_type             = "tinav6.c1r1p3"
  source_omi          = "ami-938d49ae"  # Debian 12 base
  omi_name            = "Debian 12 - {{timestamp}}"
  omi_description     = "Debian 12 with separate /var partition on 20GB BSU volume"
  
  communicator        = "ssh"
  ssh_username        = "outscale"
  ssh_interface       = "public_ip"
  ssh_keypair_name    = "admin public key"
  ssh_agent_auth      = "true"
```

**Key points:**

- `vm_type`: Uses a small instance type suitable for image building
- `source_omi`: Official Debian 12 base image from OUTSCALE
- `ssh_agent_auth`: Leverages SSH agent for key management
- Dynamic naming with timestamp prevents OMI name conflicts

### Block Device Mappings

```hcl
  launch_block_device_mappings {
    device_name           = "/dev/sda1"
    volume_size           = 20
    volume_type           = "standard"
    delete_on_vm_deletion = true
  }

  launch_block_device_mappings {
    device_name           = "/dev/xvdb"
    volume_size           = 20
    volume_type           = "standard"
    delete_on_vm_deletion = true
  }
```

Two volumes are attached during the build:

1. **Root volume** (`/dev/sda1`): Contains the base OS
2. **Secondary volume** (`/dev/xvdb`): Will become the dedicated `/var` partition

## Build Process and Hardening Steps

### Step 1: Wait for Cloud-Init

```hcl
provisioner "shell" {
  inline = [
    "echo 'Waiting for cloud-init to complete...'",
    "cloud-init status --wait",
    "echo 'Cloud-init completed'"
  ]
}
```

This ensures all cloud initialization tasks complete before modifications begin, preventing race conditions.

### Step 2: Partition Setup and Data Migration

```hcl
provisioner "shell" {
  inline = [
    "echo 'Installing python3 env'",
    "sudo DEBIAN_FRONTEND='noninteractive' apt-get update",
    "sudo DEBIAN_FRONTEND='noninteractive' apt-get install -y python3",
    "sudo /usr/sbin/mkfs.ext4 -F /dev/xvdb || true",
    "sudo mkdir -p /mnt/var",
    "sudo mount /dev/xvdb /mnt/var",
    "sudo cp -a /var/* /mnt/var/",
    "sudo umount /mnt/var",
```

**Process explanation:**

1. **Python3 installation**: Required for potential Ansible provisioning or automation
2. **Filesystem creation**: Formats `/dev/xvdb` with ext4 filesystem
3. **Data preservation**: Copies existing `/var` contents to new partition using `cp -a` to preserve attributes
4. **Safe unmount**: Unmounts temporary mount point before permanent configuration

### Step 3: Configure Permanent Mounts with Security Options

```hcl
    "echo '/dev/xvdb /var ext4 defaults,nofail,nosuid,nodev 0 2' | sudo tee -a /etc/fstab",
    "sudo mount -a",
```

The `/var` partition is added to `/etc/fstab` with:

- `defaults`: Standard mount options (rw, suid, dev, exec, auto, nouser, async)
- `nofail`: System boots even if this volume is unavailable
- `nosuid`: Prevents setuid/setgid bit execution
- `nodev`: Prevents device file interpretation
- `0 2`: No dump, fsck after root partition

### Step 4: Bind Mounts with Hardening Flags

```hcl
    "echo '/var/log /var/log none bind,noexec,nosuid,nodev 0 0' | sudo tee -a /etc/fstab",
    "echo '/var/tmp /var/tmp none bind,noexec,nosuid,nodev 0 0' | sudo tee -a /etc/fstab",
    "echo '/srv /srv none bind,noexec,nosuid,nodev 0 0' | sudo tee -a /etc/fstab",
    "echo '/home /home none bind,nosuid,nodev 0 0' | sudo tee -a /etc/fstab",
    "echo '/tmp /tmp none bind,noexec,nosuid,nodev 0 0' | sudo tee -a /etc/fstab",
  ]
}
```

Bind mounts apply additional restrictions without separate partitions:

| Mount Point | Security Flags | Purpose |
|-------------|----------------|---------|
| `/var/log` | noexec, nosuid, nodev | Prevents execution from log directories |
| `/var/tmp` | noexec, nosuid, nodev | Hardens temporary storage used by applications |
| `/srv` | noexec, nosuid, nodev | Protects service data directories |
| `/home` | nosuid, nodev | Allows user scripts but prevents privilege escalation |
| `/tmp` | noexec, nosuid, nodev | Critical hardening for temporary files |

**Important note**: These bind mounts remount directories on themselves with stricter options, effectively applying security flags without physically separating partitions.

## Building the OMI

### Step 1: Prepare Your Environment

```bash
# Set credentials
export OUTSCALE_ACCESSKEYID="your_access_key"
export OUTSCALE_SECRETKEYID="your_secret_key"

# Ensure SSH agent has your key
ssh-add /path/to/your/private/key

# Verify SSH agent
ssh-add -l
```

### Step 2: Validate Configuration

```bash
packer validate debian12-hardened.pkr.hcl
```

### Step 3: Build the OMI

```bash
packer build debian12-hardened.pkr.hcl
```


## Verification and Testing

After launching an instance from your new OMI, verify the hardening:

### Check Mount Points

```bash
mount | grep -E '(var|tmp|home|srv)'
```

Expected output:

```
/dev/xvdb on /var type ext4 (rw,nosuid,nodev,relatime)
/var/log on /var/log type none (rw,noexec,nosuid,nodev,relatime)
/var/tmp on /var/tmp type none (rw,noexec,nosuid,nodev,relatime)
/srv on /srv type none (rw,noexec,nosuid,nodev,relatime)
/home on /home type none (rw,nosuid,nodev,relatime)
/tmp on /tmp type none (rw,noexec,nosuid,nodev,relatime)
```

### Verify Security Flags

```bash
# Test noexec on /tmp
echo '#!/bin/bash
echo "test"' > /tmp/test.sh
chmod +x /tmp/test.sh
/tmp/test.sh
# Should fail with: Permission denied
```

## Security Benefits Achieved

This configuration provides multiple layers of defense:

1. **Resource Isolation**: `/var` on dedicated partition prevents log flooding from affecting root filesystem
2. **Execution Prevention**: `noexec` on `/tmp`, `/var/tmp`, and `/var/log` blocks common malware deployment vectors
3. **Privilege Escalation Mitigation**: `nosuid` prevents setuid binary exploitation
4. **Device File Protection**: `nodev` prevents device node attacks
5. **Compliance Alignment**: Meets CIS Benchmark recommendations for filesystem hardening


## Additional Resources

### OUTSCALE Documentation
- [OUTSCALE Packer Plugin Repository](https://github.com/outscale/packer-plugin-outscale) - Official plugin source code and documentation
- [OUTSCALE Packer User Guide](https://docs.outscale.com/en/userguide/Packer.html) - Official documentation for using Packer with OUTSCALE
- [OUTSCALE OMI Packer Examples](https://github.com/outscale/omi-packer/blob/master/linux.pkr.hcl) - Reference Packer configurations for Linux OMIs

### Security Standards
- [CIS Debian Linux 12 Benchmark](https://www.cisecurity.org/benchmark/debian_linux) - Industry security baseline
- [NIST Guide to General Server Security](https://csrc.nist.gov/publications) - Federal security guidelines

### Technical References
- [Linux Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html) - Official FHS documentation
- [HashiCorp Packer Documentation](https://developer.hashicorp.com/packer/docs) - Core Packer documentation

---

