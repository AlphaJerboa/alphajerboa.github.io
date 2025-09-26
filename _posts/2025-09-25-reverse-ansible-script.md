---
layout: post
title: "Reverse Ansible: When the Target Pulls Its Own Configuration"
date: 2025-09-25 15:10:00 +0000
categories: [automation, gitops]
tags: [ansible, git, github, bash]
author: alphajerboa
license: "CC BY 4.0"
license_url: "https://creativecommons.org/licenses/by/4.0/"
excerpt: "GitOps-style workstation management with a single command!"
---

## Reverse Ansible: When the Target Pulls Its Own Configuration

Most Ansible setups follow the traditional push model - a control node connects to target servers and applies configurations. But what if we flip this around?

Here's an interesting pattern where the target server pulls and applies its own Ansible playbook from a remote Git repository:

```bash
{%raw%}
#!/usr/bin/env bash
# Author: alphajerboa
cat << EOF
Welcome to your workstation setup program
This script will install all usual tools on your workstation
This script requires root priviligies (sudo), so please enter your user password when prompted (sudo, become)
EOF

clean_exit(){
    # Remove temporary folder if needed
    [[ "$GIT_REPO_PATH" =~ ^/tmp/tmp* ]] && rm -Rf $GIT_REPO_PATH
    exit ${1:-0}
}

# Variables definition
GIT_REPO_NAME="workstation_setup"
GIT_REPO_BRANCH="main"
GIT_REPO_CREDS="<my_cred>"
GIT_REPO_URL="<my_repo_url"
TAGS=${1}

# Folder check/creation to store local copy of the git repository
GIT_REPO_PATH=$(mktemp -d)  # Using a temporary folder (deleted at the end of the script)
trap clean_exit SIGINT SIGKILL SIGTERM

# Check network connectivity
! nc -w1 -z 1.1.1.1 53 &>/dev/null && echo "No internet connectivity, please check your network configuration (IP+DNS)" && clean_exit 1

# List of packages to install
APT_PACKAGES="ca-certificates \
curl \
software-properties-common \
git"

# Install basic package
sudo apt update && \
sudo apt install -y $APT_PACKAGES

# Install ansible
sudo add-apt-repository --yes --update ppa:ansible/ansible && \
sudo apt install -y ansible

# Check tools installation
for TOOL in git ansible-playbook
do
    ! command -v $TOOL &> /dev/null && echo "$TOOL is missing" && clean_exit 1
done

# Get ansible repository
if [[ -d $GIT_REPO_PATH/$GIT_REPO_NAME ]];
then
    # Folder already exist, check if it is a git repo and if it is the good repo
    cd $GIT_REPO_PATH/$GIT_REPO_NAME
    [[ ! -d .git ]] && echo "The current working directory isn't a git repository" && clean_exit 1
    ! git config -l | grep -q "^remote.origin.url=.*@${GIT_REPO_URL}$" && echo "The current working directory is not a copy of the github repository $GIT_REPO_NAME" && clean_exit 1
    # Reset local copy of the repository
    git fetch origin
    git checkout $GIT_REPO_BRANCH
    git reset --hard origin/$GIT_REPO_BRANCH
else
    # Not local copy of the git repository yet
    mkdir -p $GIT_REPO_PATH
    cd $GIT_REPO_PATH
    git clone --depth=1 --branch=$GIT_REPO_BRANCH https://${GIT_REPO_CREDS}@${GIT_REPO_URL} $GIT_REPO_NAME
    cd $GIT_REPO_NAME
fi

# Run ansible playbook on local host
ansible-playbook \
--connection=local \
--limit 127.0.0.1 \
--tags $TAGS \
--ask-become-pass \
main.yml

clean_exit 0
{%endraw%}
```

Note: The `--connection=local` flag makes Ansible run against localhost

## Why This Pattern Works

This approach turns the traditional Ansible model on its head:

- **No control node needed** - each target manages itself
- **Git as the source of truth** - configurations are versioned and centrally stored
- **Self-bootstrapping** - the script installs its own dependencies (git, ansible)
- **Flexible targeting** - uses tags to run specific parts of the playbook
- **Security-conscious** - supports private repositories with credentials

## Perfect For

- Workstation setup scripts
- Cloud instance bootstrapping
- Self-configuring containers
- GitOps-style infrastructure management

Sometimes it's easier to teach a server to configure itself than to configure it remotely. 


## Real-World Usage Example

Here's how to set this up for Linux workstation configuration:

### 1. Create Your Private Ansible Repository

First, create a private GitHub repository (e.g., `my-workstation-ansible`) with your Ansible playbooks:

```
my-workstation-ansible/
├── main.yml
├── roles/
│   ├── common/
│   ├── development/
│   └── multimedia/
└── group_vars/
    └── all.yml
```

### 2. Fork and Customize the Script

Clone the original script:
```bash
wget https://raw.githubusercontent.com/AlphaJerboa/sysadmin_scripts/refs/heads/main/run_ansible_playbook_locally.sh
```

Update the variables for your setup:
```bash
# Variables definition
GIT_REPO_NAME="my-workstation-ansible"
GIT_REPO_BRANCH="main"
GIT_REPO_CREDS="your_github_token"
GIT_REPO_URL="github.com/yourusername/my-workstation-ansible.git"
TAGS=${1}
```

Then push your customized script to a public repository (e.g., `my-workstation-bootstrap`).

### 3. One-Line Workstation Setup

On any fresh Linux workstation, run:
```bash
wget -qO- https://raw.githubusercontent.com/yourusername/my-workstation-bootstrap/refs/heads/main/run_ansible_playbook_locally.sh | bash
```

Or with specific tags:
```bash
wget -qO- https://raw.githubusercontent.com/yourusername/my-workstation-bootstrap/refs/heads/main/run_ansible_playbook_locally.sh | bash -s development
```

### 4. What Happens

1. Script downloads and installs git + ansible
2. Clones your private Ansible repository using credentials
3. Runs `ansible-playbook --connection=local` against localhost
4. Cleans up temporary files

This gives you GitOps-style workstation management with a single command!
