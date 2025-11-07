---
layout: post
title: "Integrating Ansible Vault with OpenBao"
date: 2024-10-03 07:59:00 +0000
categories: [security, automation]
tags: [ansible, openbao]
author: alphajerboa
license: "CC BY 4.0"
license_url: "https://creativecommons.org/licenses/by/4.0/"
excerpt: "How to store Ansible vault password in OpenBAo"
---

# Integrating Ansible Vault with OpenBao

## Overview

This guide demonstrates how to store Ansible Vault password in OpenBao and retrieve them dynamically during playbook execution, eliminating the need to store sensitive vault passwords in plaintext files.

## Prerequisites

- OpenBao server running and accessible
- OpenBao CLI (`bao`) installed
- Valid OpenBao authentication token
- Ansible installed
- KV secrets engine enabled in OpenBao

## Architecture

The solution uses a shell script as a vault password file that queries OpenBao at runtime to retrieve the Ansible Vault passphrase. This approach provides:

- Centralized secret management
- Audit logging through OpenBao
- Token-based authentication
- No plaintext passwords on disk

## Implementation

### 1. Configure OpenBao Policy

Create a policy file `ansible-vault-reader.hcl` with read permissions for the Ansible vault password:

```hcl
path "kv/data/iac/ansible" {
  capabilities = ["read"]
}
```

Apply the policy to OpenBao:

```bash
bao policy write ansible-vault-reader ansible-vault-reader.hcl
```

Attach this policy to your user or token:

```bash
# For userpass authentication
bao write auth/userpass/users/your-username policies="ansible-vault-reader"

# Or create a token with this policy
bao token create -policy="ansible-vault-reader"
```

### 2. Store the Vault Password in OpenBao

First, store your Ansible Vault password in OpenBao's KV secrets engine:

```bash
bao kv put -mount=kv iac/ansible vault_passphrase="your-secure-vault-password"
```

### 3. Create the Password Retrieval Script

Create `bao-password.sh` with the following content:

```bash
#!/usr/bin/env bash
set -o pipefail

bao() {
  VAULT_ADDR="https://vault.company.com" /usr/bin/bao $*
}

! bao token lookup >/dev/null 2>&1 && echo "No token to reach vault, login to your openbao instance before running this script" && exit 1

bao kv get --format=json -mount=kv iac/ansible | jq -r '.data.data.vault_passphrase'
```

Make the script executable:

```bash
chmod +x bao-password.sh
```

### 4. Script Breakdown

- **`set -o pipefail`**: Ensures the script fails if any command in a pipeline fails
- **`bao()` function**: Wraps the OpenBao CLI with the correct VAULT_ADDR
- **Token validation**: Checks if a valid OpenBao token exists before proceeding
- **Secret retrieval**: Fetches the secret in JSON format and extracts the vault_passphrase field using `jq`

### 5. Run Ansible Playbooks

Execute your playbooks using the script as the vault password file:

```bash
ansible-playbook -i inventory/ --vault-password-file=./bao-password.sh main.yml
```

## Authentication

Before running playbooks, ensure you're authenticated to OpenBao:

```bash
bao login -method=userpass username=your-username
# or
bao login -method=token token=your-token
```

The script validates the token before attempting to retrieve secrets, providing clear error messages if authentication is missing.

## Security Considerations

- **Token Storage**: OpenBao tokens are typically stored in `~/.bao-token` with restricted permissions
- **Script Permissions**: Ensure the password script has appropriate file permissions (700 or 750)
- **Token Lifecycle**: Use token renewal mechanisms for long-running automation
- **Audit Logging**: All secret access is logged by OpenBao for compliance and security audits
- **Network Security**: Use HTTPS for all OpenBao communications

## Advantages

1. **No Plaintext Passwords**: Vault passwords never stored on disk in plaintext
2. **Centralized Management**: Single source of truth for vault passwords
3. **Access Control**: Leverage OpenBao's RBAC for fine-grained permissions
4. **Rotation**: Easy password rotation without updating multiple files
5. **Audit Trail**: Complete audit log of when vault passwords were accessed

## Troubleshooting

**Error: "No token to reach vault"**
- Run `bao login` to authenticate
- Verify your OpenBao token hasn't expired

**Error: Permission denied**
- Ensure your token has read permissions on the `kv/iac/ansible` path
- Check OpenBao policies assigned to your token

**Error: Connection refused**
- Verify VAULT_ADDR is correct
- Ensure network connectivity to OpenBao server
- Check firewall rules

## Conclusion

This integration provides a secure, auditable method for managing Ansible Vault passwords through OpenBao, following infrastructure-as-code and secret management best practices.
