---
layout: post
title: "Automated Root Password Rotation with OpenBao and Ansible"
date: 2025-11-07 07:35:00 +0000
categories: [security, automation]
tags: [ansible, openbao]
author: alphajerboa
license: "CC BY 4.0"
license_url: "https://creativecommons.org/licenses/by/4.0/"
excerpt: "Password management with Openbao and Ansible"
---


# Automated Root Password Rotation with OpenBao and Ansible

## Overview

This workflow automates root password rotation across server fleets using OpenBao for secrets management and Ansible for deployment. The system generates cryptographically secure passwords, stores both plaintext and hashed versions in OpenBao, then deploys OS-appropriate hashes via Ansible.

## Architecture

### Password Generation and Storage

The bash script (https://github.com/AlphaJerboa/sysadmin_scripts/blob/main/openbao_root_password_rotation.sh) performs three key operations:

1. **Password Generation**: Creates 20-character passwords from alphanumeric and special characters (`?`,`#`,$`,`-`,`_`) using `/dev/urandom`
2. **Hash Generation**: Produces both SHA-512 and yescrypt hashes using Python's `crypt` module with 16-character random salts
3. **OpenBao Storage**: Stores secrets in a hierarchical structure:
   - `servers/<hostname>/passwords`: Current and previous plaintext passwords
   - `servers/<hostname>/hashes`: SHA-512 and yescrypt hashes

```bash
# Usage
./openbao_root_password_rotation.sh web-server-01 db-server-02
```

### Hash Type Selection by OS Version

Use Ansible variables to selects the appropriate hash algorithm based on Debian version:

| OS Version | Hash Type | Reason |
|------------|-----------|--------|
| Debian 10  | SHA-512   | Legacy glibc support |
| Debian 11+ | yescrypt  | Modern password hashing standard |

The `default_hash_type` variable in version-specific var files controls this selection.
```bash
$ grep default_hash_type: vars/debian*.yml
vars/debian-10.yml:default_hash_type: sha512
vars/debian-11.yml:default_hash_type: yescrypt
vars/debian-12.yml:default_hash_type: yescrypt
vars/debian-13.yml:default_hash_type: yescrypt
```

### Ansible Deployment Workflow

**Step 1: Authentication**

Ansible authenticates to OpenBao using userpass authentication:

```yaml
POST {{ openbao_url }}/v1/auth/userpass/login/{{ openbao_username }}
```

**Step 2: Secret Retrieval**

The playbook retrieves secrets dynamically based on inventory hostname and OS-specific hash type:

```yaml
vault_path: "{{ vault_kv_path }}/servers/{{ inventory_hostname_short }}/hashes"
value_name: "root_{{ default_hash_type }}"
```

This constructs paths like:
- `kv/servers/web-server-01/hashes` → `root_yescrypt` (Debian 11+)
- `kv/servers/legacy-01/hashes` → `root_sha512` (Debian 10)

Define the root hashes location
$ grep -A3 vault_secrets var/all.yml
```yaml
{%raw%}
vault_secrets:
  - fact_name: "root_hash"
    vault_path: "{{ vault_kv_path }}/servers/{{ inventory_hostname_short }}/hashes"
    value_name: "root_{{ default_hash_type }}"
{%endraw%}
```

Create the playbook to load the vault secrets
$ cat tasks/load_vault.yml
```yaml
{%raw%}
- name: Sent Curl request to vault
  ansible.builtin.uri:
    url: '{{ openbao_url }}/v1/auth/userpass/login/{{ openbao_username }}'
    method: 'POST'
    body: '{"password": "{{ openbao_password }}"}'
    body_format: json
    return_content: yes
    timeout: 60
  retries: 3
  register: login_output
  delegate_to: 127.0.0.1

- name: Display curl output
  ansible.builtin.debug:
    msg:
      - 'Request sent to {{ openbao_url }}/v1/auth/userpass/login/{{ openbao_username }}'
      - 'Token received: {{ login_output.json.auth.client_token }}'

- name: Load secrets
  ansible.builtin.uri:
    url: '{{ openbao_url }}/v1/{{ item.vault_path }}'
    method: 'GET'
    headers:
      X-Vault-Request: true
      X-Vault-Token: '{{ login_output.json.auth.client_token }}'
    body_format: json
    return_content: yes
    timeout: 60
  retries: 3
  register: apicall_output
  delegate_to: 127.0.0.1
  loop: "{{ vault_secrets }}"
  ignore_errors: true

- name: Set fact for any secret
  ansible.builtin.set_fact:
    "{{ item.item.fact_name }}": "{{ item | json_query('json.data.data.' + item.item.value_name ) }}"
  loop: '{{ apicall_output.results }}'
  loop_control:
    label: "{{ item.item.fact_name }}"
{%endraw%}
```


**Step 3: Password Deployment**

The retrieved hash is applied using `chpasswd -e` (encrypted mode):

```bash
echo 'root:$y$j9T$...' | chpasswd -e
```

Create the playbook task to update the password hash
$ cat tasks/update_root_password.yml
```yaml
{%raw%}
  - name: Debug output
    ansible.builtin.debug:
      msg: "{{ root_hash }}"
    when: root_hash is defined

  - name: Update password file
    ansible.builtin.shell:
      cmd:  echo 'root:{{ root_hash }}' | chpasswd -e
    register: shadow_updated
    when: root_hash is defined
{%endraw%}
```

## Security Considerations

1. **Password Complexity**: 20 characters with mixed character classes provides ~119 bits of entropy
2. **Salt Randomization**: Each hash uses a unique 16-character salt
3. **Old Password Retention**: Previous passwords are stored for rollback capability
4. **Encrypted Hashes**: Passwords are never deployed in plaintext; only pre-hashed values are sent to target systems
5. **Credential Delegation**: All OpenBao API calls are delegated to localhost, keeping tokens off managed nodes

## Operational Workflow

1. Security team runs rotation script for target hosts
2. Script generates new passwords and updates OpenBao KV store
3. Infrastructure team deploys changes via Ansible playbook
4. Ansible retrieves OS-appropriate hashes and updates `/etc/shadow`
5. Old passwords remain accessible in OpenBao for emergency access

## Benefits

- **Automated Rotation**: Eliminates manual password management
- **OS Awareness**: Automatically selects compatible hash algorithms
- **Audit Trail**: All password changes tracked in OpenBao
- **Zero-Downtime**: Password rotation occurs without service interruption
- **Separation of Duties**: Password generation and deployment are separate operations
