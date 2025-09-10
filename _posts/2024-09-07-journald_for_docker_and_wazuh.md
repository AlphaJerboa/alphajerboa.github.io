---
layout: post
title: "Monitor Docker Container Logs with Wazuh's New Journald Integration"
date: 2024-09-07 12:35:00 +0000
categories: [container, security]
tags: [docker, wazuh]
author: alphajerboa
excerpt: "Centralizing Container Logs: Wazuh Journald + Docker Log Driver Setup"
---

Container log monitoring just got easier with Wazuh's new journald integration.
The latest release enables seamless collection of log messages through journald, and when paired with Docker's journald log driver, you can now centralize all your container logs directly into Wazuh.
This powerful combination opens up new possibilities for creating tailored decoders and custom alert rules specific to your containerized environments.

Let's setup your docker environment.

## Method 1: Global Configuration with Docker Daemon

Configure Docker daemon to use journald as the default logging driver for all containers.

### Step 1: Configure the Docker Daemon

Create or edit the Docker daemon configuration file at `/etc/docker/daemon.json`:

```json
{
  "log-driver": "journald"
}
```

### Step 2: Restart Docker

Apply the changes by restarting the Docker daemon:

```bash
sudo systemctl restart docker
```

**Important**: This change only affects newly created containers. Existing containers will continue using their current logging driver until they're recreated.

## Method 2: Per-Container Configuration

If you prefer more granular control, you can specify the journald driver for individual containers.

### Using Docker CLI

```bash
docker run --log-driver=journald --name my-app nginx:latest
```

#### Common Log Options

```bash
{%raw%}
docker run --log-driver=journald \
  --log-opt tag="{{.Name}}/{{.ID}}" \
  --log-opt labels=environment,version \
  --log-opt env=APP_ENV,DEBUG_LEVEL \
  nginx:latest
{%endraw%}
```

The available options include:

- **tag**: Custom identifier for log entries (supports Go templates)
- **labels**: Include specific container labels in log entries
- **env**: Include specific environment variables in log entries
- **env-regex**: Include environment variables matching a regex pattern
- **labels-regex**: Include labels matching a regex pattern

### Using Docker Compose

For docker-compose deployments, add the logging configuration to your service definition:

```yaml
{%raw%}
version: '3.8'
services:
  web:
    image: nginx:latest
    logging:
      driver: journald
  
  api:
    image: my-api:latest
    logging:
      driver: journald
      options:
        tag: "api-service"
{%endraw%}
```

#### Docker Compose with Advanced Options

```yaml
{%raw%}
version: '3.8'
services:
  app:
    image: my-app:latest
    labels:
      - "environment=production"
      - "service=web"
    environment:
      - APP_ENV=production
      - LOG_LEVEL=info
    logging:
      driver: journald
      options:
        tag: "{{.ImageName}}/{{.Name}}"
        labels: environment,service
        env: APP_ENV,LOG_LEVEL
{%endraw%}
```

## Viewing and Analyzing Your Logs

Once your containers are configured to use journald, you can leverage the powerful journalctl command to view and analyze logs.

### Basic Log Viewing

```bash
# View logs for a specific container
journalctl CONTAINER_NAME=my-app

# View logs for all Docker containers
journalctl -u docker.service

# Follow logs in real-time
journalctl -f CONTAINER_NAME=my-app
```

### Advanced Filtering

```bash
# View logs from the last hour
journalctl CONTAINER_NAME=my-app --since "1 hour ago"

# Filter by log level
journalctl CONTAINER_NAME=my-app PRIORITY=3

# Combine multiple filters
journalctl CONTAINER_NAME=my-app \
  CONTAINER_TAG=production \
  --since "2023-01-01" \
  --until "2023-01-31"
```


## Best Practices and Considerations

### Performance Considerations

While journald is efficient, consider these factors for high-volume logging scenarios:

- **Log level filtering**: Configure your applications to log appropriately for the environment
- **journald configuration**: Tune `/etc/systemd/journald.conf` for your storage and retention needs
- **Disk space**: Monitor disk usage, especially in `/var/log/journal/`


### Log Retention Issues

Configure journald retention in `/etc/systemd/journald.conf`:

```ini
[Journal]
MaxRetentionSec=1month
MaxFileSec=1week
MaxFileSize=100M
```

