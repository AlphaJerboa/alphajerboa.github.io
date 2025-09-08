---
layout: post
title: "Storing Let's Encrypt Certificates in OpenBao vault and deploying them with Ansible"
date: 2024-10-09 16:59:00 +0000
categories: [security, automation]
tags: [letsencrypt, ansible, openbao]
author: alphajerboa
excerpt: "Certificate management with Openbao and Ansible"
---


## Introduction

Managing SSL/TLS certificates across your infrastructure becomes seamless when you combine OpenBao's secure storage capabilities with Ansible's automation power.


## Setting Up OpenBao for Certificate Storage

### Storing DNS Registrar API Keys

First, we'll store the DNS registrar API key in OpenBao, which will be used for the DNS challenge during Let's Encrypt certificate generation:

```bash
# Store the DNS registrar API key in OpenBao
bao kv put -mount kv cloud/dnsregistrar api_key="your-dns-api-key"
```

### Retrieving API Keys for Certificate Generation

When generating Let's Encrypt certificates using DNS challenge, retrieve the API key from OpenBao:

```bash
# Retrieve the DNS registrar API key
API_KEY=$(bao kv get -format json -mount kv cloud/dnsregistrar | jq -r '.data.data.api_key')
```

This approach ensures that sensitive API credentials are never hardcoded in your automation scripts or stored in plain text files.

### Storing Certificates in OpenBao

Once the Let's Encrypt certificate is generated using the DNS challenge, store both the certificate and private key in OpenBao encoded in base64:

```bash
# Store certificate and private key in OpenBao (base64 encoded)
bao kv put -mount kv services/$DOMAIN \
    public_key="$(cat $CRT_FILE | base64 -w0)" \
    private_key="$(cat $KEY_FILE | base64 -w0)"
```

Where:
- `$DOMAIN` is the domain name for the certificate
- `$CRT_FILE` is the path to the certificate file
- `$KEY_FILE` is the path to the private key file

The base64 encoding ensures the certificate data is safely stored as text in OpenBao's key-value store, preventing any formatting issues that might occur with multiline PEM data.

## Renewing certificates stored in Openbao

Use a script to check existing certificate and renew them if needed. [Here](https://github.com/AlphaJerboa/sysadmin_scripts/blob/main/renew-letsencrypt-certificate-dns-challenge.sh) is an example to adapt (Use GANDI DNS challenge with the [lego](https://github.com/go-acme/lego) tool)

## Configuring Ansible for Certificate Deployment

### Variable Definition

Define the OpenBao paths and secrets to retrieve in your Ansible variables:

```yaml
# Ansible variables for OpenBao integration
vault_kv_path: "kv/data"

vault_secrets:
  - fact_name: "svc_cert_pub"
    vault_path: "{{ vault_kv_path }}/services/{{ service_fqdn.split(',')[0] }}"
    value_name: "public_key"
  - fact_name: "svc_cert_key"
    vault_path: "{{ vault_kv_path }}/services/{{ service_fqdn.split(',')[0] }}"
    value_name: "private_key"
```

This configuration:
- Uses `vault_kv_path` to define the base path for the KV v2 mount (`kv/data`)
- Extracts the primary domain from `service_fqdn` using `split(',')[0]` to handle multi-domain certificates
- Maps OpenBao secret values to Ansible facts (`svc_cert_pub` and `svc_cert_key`)
- Dynamically constructs the vault path based on the service domain

### Ansible Tasks for Certificate Retrieval

The following Ansible tasks authenticate with OpenBao and retrieve the stored certificates:

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

This task sequence:
1. **Authentication**: Authenticates with OpenBao using userpass authentication method
2. **Debug output**: Displays the authentication request details and received token
3. **Secret retrieval**: Loops through the `vault_secrets` list to retrieve each certificate component
4. **Fact setting**: Creates Ansible facts (`svc_cert_pub` and `svc_cert_key`) containing the base64-encoded certificate data

### Certificate Deployment Tasks

Once the certificate data is retrieved and stored in Ansible facts, deploy them to the target servers:

```yaml
- name: Create public certificate
  ansible.builtin.copy:
    content: '{{ svc_cert_pub | b64decode }}'
    dest: '{{ web_cert_folder }}/{{ service_fqdn }}.crt'
    mode: 0640
  when: svc_cert_pub

- name: Create private certificate
  ansible.builtin.copy:
    content: '{{ svc_cert_key | b64decode }}'
    dest: '{{ web_cert_folder }}/{{ service_fqdn }}.key'
    mode: 0640
  when: svc_cert_key
```

These deployment tasks:
- **Decode base64 data**: Use the `b64decode` filter to convert the stored base64 certificate data back to PEM format
- **Create certificate files**: Write the decoded certificates to the specified directory using the service FQDN as the filename
- **Set secure permissions**: Apply `0640` permissions to ensure certificates are readable by the web server but not world-readable
- **Conditional execution**: Only create files when the corresponding certificate data exists in the facts

