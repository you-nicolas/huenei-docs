# Production Migration Plan: CentOS to Ubuntu 24.04 (DOCKERPROD01)

## 1. Project Overview
- **Objective:** Migrate all production services from an EOL CentOS server to a new Ubuntu 24.04 server.
- **Source Server:** `DOCKERPROD` (CentOS EOL)
- **Target Server:** `DOCKERPROD01` (Ubuntu 24.04, 192.169.0.102)
- **Status:** Initial Server Setup (Docker + Log Rotation) **COMPLETED**.

## 2. Infrastructure Modernization
- **Edge Routing:** Replace Nginx Proxy with Cloudflare Tunnels (direct routing per container).
- **Observability:** Implement a Prometheus + Grafana + Loki (PLG) stack for metrics and logs.
- **CI/CD:** 
    - Maintain existing Jenkins pipelines (modifying target server/credentials).
    - Migrate Keycloak to Azure DevOps pipelines (parallel task).
- **Log Management:** Global Docker log rotation enabled (10MB x 3 files) to prevent disk exhaustion.

## 3. Data Persistence & Discovery
### 3.1 Critical Bind Mounts (To be rsync'd)
| Service | Source Path (CentOS) | Target Path (Ubuntu) |
| :--- | :--- | :--- |
| SGD Docs | `/opt/sgd/docs` | `/home/devops/sgd-backend_data/docs` |
| SGD Certs | `/opt/sgd/certs` | `/home/devops/sgd-frontend_data/certs` |
| Objetivos SEO | `/home/root/seguimiento-objetivos-seo-prod-back_data` | `/home/devops/seguimiento-objetivos-seo-prod-back_data` |
| Postgres DB | `/home/sonar/postgres/data` | `/home/devops/postgres_data` |
| Keycloak Certs| `/root/keycloak/certs` | `/home/devops/keycloak_data/certs` |

### 3.2 Jenkins Deployment Convention
Most apps follow the pattern:
- **Mount:** `/home/${USER_NAME}/${APP_NAME}_data:/app/data`
- **Action:** Ensure target directories exist on the new server before deployment.

## 4. Phased Implementation Plan

*Note: Detailed CLI commands for each phase are available in the [MIGRATION_RUNBOOK.md](./MIGRATION_RUNBOOK.md).*

### Phase 1: Preparation (Done)
- [x] Install Docker Engine & Docker Compose on `DOCKERPROD01`.
- [x] Configure non-root Docker access.
- [x] Configure global log rotation in `/etc/docker/daemon.json`.

### Phase 2: Core Infrastructure Setup
- [ ] Deploy Prometheus, Grafana, and Loki via `compose.yaml`.
- [ ] Configure Cloudflare Tunnel entries (Infrastructure Team coordination).
- [ ] Migrate Keycloak (Azure DevOps Pipeline + Postgres DB).

### Phase 3: Database & Persistence Migration
- [ ] **Postgres:** Perform `pg_dumpall` on CentOS and restore on Ubuntu.
- [x] **Bind Mounts:** `rsync` data from CentOS to Ubuntu using the new naming convention.

### Phase 4: Application Migration (One-by-One)
- [ ] Update Jenkins `SERVER_CREDENTIALS` and `TARGET_SERVER`.
- [ ] Modify Jenkins logic to use Cloudflare hostnames.
- [ ] Deploy and verify: `huenei-avatar`, `navioper`, `ai-recruiting`, etc.

### Phase 5: Verification & Cleanup
- [ ] Validate observability metrics and log collection.
- [ ] Verify SSL/TLS via Cloudflare.
- [ ] Shut down CentOS server after 7 days of stability.

## 5. Verification Checklist
- [ ] Connectivity via Cloudflare.
- [ ] Data integrity in Postgres.
- [ ] Logging visibility in Grafana/Loki.
- [ ] Jenkins pipeline success to new server.
