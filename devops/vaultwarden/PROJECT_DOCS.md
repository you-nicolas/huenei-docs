# Vaultwarden DevSecOps Secret Management

## Project Overview
This project implements a centralized, self-hosted secret management solution for **Huenei IT Services** using Vaultwarden. It is designed to replace insecure Excel and `.txt` storage with a secure, SSO-integrated platform capable of isolating multiple MSP projects.

## Architecture
- **Application:** Vaultwarden (Rust-based Bitwarden API)
- **Ingress:** Cloudflare Tunnels (`vw.huenei.net` -> Local Port 11001)
- **Identity:** Entra ID (Azure AD) via OIDC
- **Infrastructure:** On-prem VM (`docker-florida-03`)
- **CI/CD:** Azure DevOps with Self-Hosted Agent

## Technical Configuration

### 1. Docker Deployment (`compose.yaml`)
To run the instance manually:
```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      # General Settings
      DOMAIN: "https://vw.huenei.net"
      SIGNUPS_ALLOWED: "true" # Set to false after initial team onboarding

      # Native OIDC SSO Settings
      SSO_ENABLED: "true"
      SSO_AUTHORITY: "${SSO_AUTHORITY}"
      SSO_CLIENT_ID: "${SSO_CLIENT_ID}"
      SSO_CLIENT_SECRET: "${SSO_CLIENT_SECRET}"
      SSO_SCOPES: "openid profile email offline_access"
      SSO_ALLOW_UNKNOWN_EMAIL_VERIFICATION: "true" # Required for Entra ID

      # Security & Permissions
      I_REALLY_WANT_VOLATILE_STORAGE: "false"
    volumes:
      - ./vw-data/:/data/
    ports:
      - 11001:80
```

### 2. Environment Variables (`.env`)
Ensure there are no typos (e.g., `AUTHORITY` vs `AUTHORITHY`).
```bash
SSO_AUTHORITY="https://login.microsoftonline.com/3153101c-34b3-4eb6-ba11-0a16fc57c0db/v2.0"
SSO_CLIENT_ID="d617be8c-8dcd-42c9-8ad9-82b3eb229a12"
SSO_CLIENT_SECRET="<stored in server .env — not committed>"
```

## Entra ID App Registration
- **Name:** `Huenei-Vaultwarden`
- **Redirect URI:** `https://vw.huenei.net/identity/connect/oidc-signin` (Must be exact)
- **Scopes Required:** `openid`, `profile`, `email`, `User.Read`

## Access & Client Setup

### 1. Master Password
- **Admin Master Password:** `Huenei$3030!` (Used for initial setup and emergency access).

### 2. Browser Extension Setup
To use the Bitwarden/Vaultwarden browser extension:
1. Download the extension (Chrome, Firefox, Edge).
2. Open the extension and click the **Settings (Gear icon)** before logging in.
3. Under **Server URL**, select **Self-hosted**.
4. Enter the Server URL: `https://vw.huenei.net`.
5. Save and proceed to login using **Enterprise Single Sign-On**.

## MSP Organizational Structure
To maintain isolation between different projects:
1. **Huenei Internal (Org):** For core team credentials (AWS Root, Jenkins).
2. **Project-X (Org):** Create a dedicated Organization for each client.
3. **Collections (Within Orgs):**
   - `[PROD] App Secrets`
   - `[STAGING] Env Vars`
   - `[SSH] Server Access`

## CI/CD Strategy
- **Source:** Azure DevOps Git Repo.
- **Agent:** Self-hosted on Huenei Infra.
- **Workflow:**
    1. Pipeline pulls the repository.
    2. Secret variables are mapped to shell environment.
    3. `docker compose up -d` is executed to apply changes.

## Security & Governance Model

### 1. Identity-First Security (SSO)
For the 4-user DevSecOps team, we prioritize **Identity Provider (IdP) Security** over local infrastructure complexity.
- **Primary Gatekeeper:** Microsoft Entra ID (MFA, Conditional Access, and Login Alerts are managed at this layer).
- **Zero-Knowledge:** Vaultwarden handles encryption/decryption, but never sees the Master Password or the Entra ID credentials.

### 2. SMTP Omission Rationale
We have explicitly decided **not** to implement a local SMTP server for this deployment.
- **Reasoning:**
    - Entra ID already provides superior "New Sign-in" and "Risky Activity" alerts.
    - With a team of 4, manual user onboarding via the `/admin` panel is more secure than automated email links.
    - Reduces infrastructure "noise" and attack surface (no SMTP credentials stored in the environment).

### 3. Hardening Standards
- `SIGNUPS_ALLOWED=false`: Prevents unauthorized account creation.
- `INVITATIONS_ALLOWED=true`: Only Admins can whitelist new team members.
- `SSO_ONLY=true`: (Post-Onboarding) Forces all authentication through Huenei's Entra ID.

### 4. Browser Extension / Autofill Issues
- **Problem:** Extension fails to sync or autofill stops working despite being logged in.
- **Root Cause:** IP Header Mismatch. Vaultwarden may be rate-limiting the proxy IP if it cannot see the real client IPs.
- **Fix:** Ensure `IP_HEADER="X-Forwarded-For"` is set in `compose.yaml`. Verify this in `/admin/diagnostics`.
- **Action:** After fixing the header, force a manual sync in the extension (Settings -> Sync).

## Maintenance & Operations

### 1. Admin Page & Security
- **Admin Token:** To enable the admin page (`/admin`), use an Argon2id hash.
- **Critical Fix (Escaping):** In `compose.yaml`, you **must** escape the dollar signs in the hash by doubling them (e.g., `$$argon2id$$v=19...`) to prevent Docker Compose from attempting variable interpolation.
- **Status:** The admin page is currently **disabled** for the POC to minimize the attack surface.

### 2. Permissions
- **Permissions:** Fix volume permissions if the container crashes: `sudo chown -R 1000:1000 ./vw-data`.
- **Backups:** Schedule a nightly backup of the `vw-data` directory.
- **Emergency Access:** Maintain one non-SSO admin account with a physical backup of the master password.
