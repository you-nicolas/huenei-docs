# Keycloak Migration Guide: DOCKERPROD to DOCKERPROD01

This document outlines the strategy and steps for migrating the Keycloak service to a modernized, CI/CD-driven infrastructure.

## 1. Objective

Migrate Keycloak and its PostgreSQL database from the legacy CentOS server (`192.169.0.101`) to the new server (`192.169.0.102`), implementing a centralized CI/CD pipeline using Azure DevOps.

## 2. Infrastructure Architecture

- **Source Server (EOL):** 192.169.0.101 (Manual Docker Compose)
- **Target Server (Prod):** 192.169.0.102 (DOCKERPROD01)
- **Centralized Agent Server:** 192.168.1.107 (AZAGENT01)
- **Deployment Method:** Azure Pipelines using Docker Contexts via SSH.
- **Build Strategy:** **Containerized Builds**. Build tools (Maven, Node, etc.) are executed within ephemeral Docker containers to ensure environment isolation and consistency.

## 3. Configuration Files

### `compose.yaml`

The deployment uses the modern Compose Specification.

- **Naming:** `compose.yaml` (replaces `docker-compose.yml`).
- **Standardization:** Named volumes for DB data, host-mounted volumes for certificates.
- **Security:** Secrets are managed via environment variables.

### `azure-pipelines.yml`

- **Agent Pool:** Uses the self-hosted pool.
- **Workflow:**
    1. Injects Variable Group secrets.
    2. Switches Docker context to `prod-server`.
    3. Executes `docker compose up -d`.

## 4. Migration Steps

### Step 1: Centralized Agent Setup (AZAGENT01 - 192.168.1.107)

#### 1.1 Modern Docker Installation (Latest Engine)

On **AZAGENT01**, execute these commands to install the latest Docker Engine (not the distro version):

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install the latest Docker packages:
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Post-install: Allow devopsuser to run docker without sudo
sudo usermod -aG docker devopsuser
# NOTE: User must log out and back in for this to take effect.
```

#### 1.2 SSH Trust & Remote Context Configuration

On **AZAGENT01**, establish a secure, passwordless connection to **DOCKERPROD01**:

1. **Generate Ed25519 Key Pair:**

   ```bash
   # Create a modern secure key without a passphrase for automation
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -q
   ```

2. **Distribute Public Key to DOCKERPROD01:**

   ```bash
   # Copy the key to the target server (requires devops password once)
   ssh-copy-id -i ~/.ssh/id_ed25519.pub devops@192.169.0.102
   ```

3. **Verify Passwordless Access:**

   ```bash
   # This should return the hostname of the target server without asking for a password
   ssh devops@192.169.0.102 "hostname"
   ```

4. **Initialize Remote Docker Context:**

   ```bash
   # Create a context that uses the SSH tunnel
   docker context create prod-server --docker "host=ssh://devops@192.169.0.102"

   # Test the context (should list containers on DOCKERPROD01)
   docker --context prod-server ps
   ```

#### 1.3 Azure DevOps Agent Installation (AZAGENT01)

We will use **Service Principal (SP)** authentication for better security and to avoid expiring Personal Access Tokens (PAT).

**Prerequisites:**

- A Service Principal created in Microsoft Entra ID.
- The Service Principal must be added as a **User** in Azure DevOps (Organization Settings > Users) with **Basic** access.
- The Service Principal must be added to the **Agent Pool** (Project Settings > Agent pools > Default > Security) with the **Administrator** role.

**Installation Steps on AZAGENT01:**

1. **Create the Agent Directory:**

   ```bash
   mkdir ~/azagent && cd ~/azagent
   ```

2. **Download the Agent Package:**

   ```bash
   curl -fkSL https://vstsagentpackage.azureedge.net/agent/3.236.1/vsts-agent-linux-x64-3.236.1.tar.gz -o agent.tar.gz
   tar zxvf agent.tar.gz
   ```

3. **Configure the Agent using Service Principal:**

   ```bash
   ./config.sh
   ```

   *Responses to prompts:*
   - **Server URL:** `https://dev.azure.com/{your-organization}`
   - **Authentication type:** Enter `SP`
   - **Application (Client) ID:** (Enter your SP Client ID)
   - **Tenant ID:** (Enter your Entra ID Tenant ID)
   - **Client Secret:** (Enter your SP Client Secret)
   - **Agent Pool:** `Default`
   - **Agent Name:** `AZAGENT01`
   - **Work Folder:** `_work`

4. **Install and Start as a Service:**

   ```bash
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```

5. **Verify Status:**

   ```bash
   sudo ./svc.sh status
   ```

### Step 2: Target Directory & Certificate Setup (DOCKERPROD01)

Before migrating data, prepare the environment on **DOCKERPROD01**:

1. **Create Work Directory:**

   ```bash
   sudo mkdir -p /home/devops/keycloak/certs
   sudo chown -R devops:devops /home/devops/keycloak
   ```

2. **Transfer SSL Certificates from Source:**
   From the **Source Server (192.169.0.101)**, push the certs directly to the new Prod server:

   ```bash
   sudo scp -r /root/keycloak/certs/* devops@192.169.0.102:/home/devops/keycloak/certs/
   ```

3. **Validate Cert Permissions:**
   On **DOCKERPROD01**, ensure the certs are readable:

   ```bash
   ls -lh /home/devops/keycloak/certs
   # Should see server.crt.pem and server.key.pem
   ```

### Step 3: CI/CD Pipeline Execution (Azure DevOps)

1. **Repository Setup:** Push `compose.yaml` and `azure-pipelines.yml` to the `main` branch.
2. **Variable Group:** Ensure `keycloak-prod` is configured with `POSTGRES_PASSWORD` and `DB_PASSWORD`.
3. **Agent Pool:** Ensure **AZAGENT01** is online in the `Default` pool.
4. **Trigger:** Run the pipeline manually. This will orchestrate the deployment from AZAGENT01 to DOCKERPROD01.

### Step 4: Database Migration (Logical Dump/Restore)

1. **Identify the Correct Source Container (192.169.0.101):**
   Legacy servers may run multiple PostgreSQL instances. Verify which one Keycloak is using.

   ```bash
   # List containers and look for keycloak-postgres
   docker ps | grep postgres

   # Inspect for credentials (e.g., keycloak-postgres-1)
   docker inspect keycloak-postgres-1 --format '{{range .Config.Env}}{{println .}}{{end}}'
   ```

2. **Generate Dump on Source:**
   Use the identified user (e.g., `keycloak`) and database name (e.g., `keycloak`).

   ```bash
   # Create a consistent logical backup
   docker exec keycloak-postgres-1 pg_dump -U keycloak keycloak > ~/keycloak_migration.sql
   ```

3. **Transfer Dump to Target (192.169.0.102):**

   ```bash
   scp ~/keycloak_migration.sql devops@192.169.0.102:/home/devops/keycloak/
   ```

4. **Restore on Target (DOCKERPROD01):**

   ```bash
   # 1. Clean existing schema
   docker exec -i keycloak-postgres psql -U keycloak -d keycloak -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"

   # 2. Import data
   cat /home/devops/keycloak/keycloak_migration.sql | docker exec -i keycloak-postgres psql -U keycloak -d keycloak

   # 3. Restart Keycloak
   docker restart keycloak-app
   ```

### Step 5: Production Cutover (Cloudflare Tunnel)

1. **Update Ingress Rules:**
   On the Cloudflare tunnel server (`cloudflarepadua`), update the service IP for `keycloak-prod.huenei.net`.

   ```bash
   sudo vi /etc/cloudflared/config.yml
   ```

   *Change:*

   ```yaml
   - hostname: keycloak-prod.huenei.net
     service: http://192.169.0.102:8085  # Updated IP
   ```

2. **Reload Tunnel Service:**

   ```bash
   sudo systemctl restart cloudflared
   ```

3. **Validation:**
   Access `https://keycloak-prod.huenei.net/auth/admin` and verify login with existing production credentials.

## 5. Troubleshooting Notes

- **Proxy Forwarding Issues:** When using Cloudflare, `PROXY_ADDRESS_FORWARDING=true` is **mandatory** in the Keycloak environment variables. Without it, redirects will fail to `localhost` or `http` instead of `https`.
- **FATAL: role "keycloak" does not exist**: Check if the superuser was renamed during initial setup (e.g., to `admin` or `postgres`). Use `docker inspect` to verify `POSTGRES_USER`.
- **Multiple DB Instances**: Always verify the correct container using `docker ps`. Dumping from the wrong PostgreSQL instance will result in missing tables or wrong schema.
- **Port Conflicts**: Ensure port `8085` (HTTP) and `8443` (HTTPS) are open in the host firewall (`ufw` or `firewalld`).

## 6. Migration Work Notes (Runbook)

| Date | Task | Status | Notes |
| :--- | :--- | :--- | :--- |
| 2026-06-02 | Infrastructure Provisioned | Done | AZAGENT01 (192.168.1.107) and DOCKERPROD01 (192.169.0.102) ready. |
| 2026-06-02 | Migration Plan Documented | Done | Runbook and modern Compose/Pipeline files generated. |
| 2026-06-05 | AZAGENT01 Docker Install | Done | Using latest Docker Engine repository. |
| 2026-06-05 | SSH Trust Established | Done | Passwordless ED25519 access verified. |
| 2026-06-05 | Azure DevOps Agent Config | Done | Registered via Service Principal as AZAGENT01. |
| 2026-06-16 | Initial Keycloak Deploy | Done | Remote deploy via Docker Context to DOCKERPROD01. |
| 2026-06-16 | Database Migration | Done | Logical dump/restore successful from .101 to .102. |
| 2026-06-16 | Production Cutover | Done | Cloudflare tunnel updated to point to 192.169.0.102. |

## 7. Verification Checklist

- [ ] Containers are running on target (`docker --context prod-server ps`).
- [ ] SSL Certificates correctly mounted (`docker exec keycloak ls /opt/keycloak/conf`).
- [ ] Keycloak Admin Console accessible via HTTPS (192.169.0.102:8443).
- [ ] Database restored and users can authenticate.
- [ ] Logs show no connection errors (`docker --context prod-server compose logs -f`).

## 8. Appendix: Service Principal Setup for Azure DevOps

Following the official Microsoft documentation, follow these steps to prepare your Service Principal for agent registration.

### Part A: Create the Service Principal (Azure Portal)

1. Go to **Microsoft Entra ID** (Azure AD) > **App registrations** > **New registration**.
2. **Name:** `AZAGENT-Keycloak-Migration` (or preferred name).
3. **Supported account types:** Accounts in this organizational directory only.
4. Click **Register**.
5. **Note the following values:**
   - Application (client) ID
   - Directory (tenant) ID
6. Go to **Certificates & secrets** > **Client secrets** > **New client secret**.
7. Copy the **Value** of the secret immediately (it will be hidden later).

### Part B: Add Service Principal to Azure DevOps Organization

*Reference: [Microsoft Docs - Add SP to Org](https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/service-principal-managed-identity?view=azure-devops)*

1. Go to **Organization Settings** > **Users**.
2. Click **Add users**.
3. In the **Users** field, search for the **Application ID** or **Name** of the Service Principal you created.
4. **Access level:** Set to **Basic**.
5. **Add to projects:** Select the project where your Keycloak migration repo resides.
6. Click **Add**.

### Part C: Grant Agent Pool Permissions

*Reference: [Microsoft Docs - SP Agent Registration](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/service-principal-agent-registration?view=azure-devops)*

1. Go to **Project Settings** > **Agent pools**.
2. Select the pool you intend to use (e.g., `Default`).
3. Click the **Security** tab.
4. Click **Add**.
5. Search for your Service Principal name.
6. **Role:** Set to **Administrator**.
7. Click **Add**.

---
