# Migration Runbook: Technical Execution Guide

This document contains the exact commands and technical procedures for the DOCKERPROD to DOCKERPROD01 migration.

## 0. Initial Server Setup (Ubuntu 24.04) - COMPLETED

The following steps were performed on `192.169.0.102` to prepare the environment.

### 0.1 Docker Engine Installation

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 0.2 Post-Installation & Security

```bash
# Enable Docker
sudo systemctl enable --now docker

# Non-root access for 'devops'
sudo usermod -aG docker devops
```

### 0.3 Global Log Rotation

Configured in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

*Applied via `sudo systemctl restart docker`.*

## 1. Database Migration: PostgreSQL

### 1.1 Extract (Old CentOS Server)

```bash
# Generate full dump
docker exec -t postgres pg_dumpall -c -U postgres > full_postgres_backup.sql

# Transfer to new server
scp full_postgres_backup.sql devops@192.169.0.102:/home/devops/
```

### 1.2 Load (New Ubuntu Server)

```bash
# Create data directory
mkdir -p /home/devops/postgres_data

# Start Postgres 14 container
docker run -d \
  --name postgres \
  --restart unless-stopped \
  -e POSTGRES_PASSWORD=Huenei$9090$3030 \
  -v /home/devops/postgres_data:/var/lib/postgresql/data \
  postgres:14

# Wait 10s then restore
cat /home/devops/full_postgres_backup.sql | docker exec -i postgres psql -U postgres
```

## 2. Persistence Migration: Bind Mounts

### 2.1 Target Directory Preparation (New Ubuntu Server)

```bash
mkdir -p /home/devops/sgd-backend_data/docs
mkdir -p /home/devops/sgd-frontend_data/certs
mkdir -p /home/devops/seguimiento-objetivos-seo-prod-back_data
mkdir -p /home/devops/keycloak_data/certs
```

### 2.2 Data Sync (Old CentOS Server)

```bash
# SGD Docs
sudo rsync -aHAXS --progress /opt/sgd/docs/ devops@192.169.0.102:/home/devops/sgd-backend_data/docs/

# SGD Certs
sudo rsync -aHAXS --progress /opt/sgd/certs/ devops@192.169.0.102:/home/devops/sgd-frontend_data/certs/

# Objetivos SEO Data
sudo rsync -aHAXS --progress /home/root/seguimiento-objetivos-seo-prod-back_data/ devops@192.169.0.102:/home/devops/seguimiento-objetivos-seo-prod-back_data/

# Keycloak Certs
sudo rsync -aHAXS --progress /root/keycloak/certs/ devops@192.169.0.102:/home/devops/keycloak_data/certs/
```

### 2.3 Permissions Fix (New Ubuntu Server)

```bash
sudo chown -R devops:devops /home/devops/*_data
```

## 4. Observability Stack (PLG)

Deployment of Prometheus, Loki, and Grafana on the new server.

### 4.1 Directory Setup

```bash
mkdir -p /home/devops/infra/observability/{prometheus,grafana,loki}
chown -R devops:devops /home/devops/infra/observability
```

### 4.2 Compose Blueprint (`/home/devops/infra/observability/compose.yaml`)

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    container_name: loki
    volumes:
      - ./loki:/etc/loki
    ports:
      - "3100:3100"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=Huenei$9090$3030
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

## 5. Work Notes & Observations

- **[2026-06-02]**: Server initialized with Docker 24.x and global log rotation.
- **[2026-06-02]**: User 'devops' added to docker group.
- **Note**: Ensure containers are stopped on the old server before final `rsync` to ensure data consistency.
