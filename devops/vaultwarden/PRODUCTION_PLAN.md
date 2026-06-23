# Vaultwarden Phase 2: CI/CD & Production Hardening

Now that the POC is verified, this plan outlines the transition to a fully managed DevSecOps deployment.

## 🏁 Phase 1 Recap (Completed)
- [x] Docker Compose deployment on `docker-florida-03`.
- [x] Cloudflare Tunnel integration (`vw.huenei.net`).
- [x] Entra ID SSO (OIDC) integration.
- [x] Admin Token RCA & Fix (Escaping `$$`).
- [x] Master Password & Browser Extension documentation.

## 🛠️ Phase 2: CI/CD Implementation (Next Steps)

### 1. Azure DevOps Pipeline Setup
- **Source:** Move this repository to a Huenei Azure DevOps Git Repo.
- **Variables:** Map the following to **Library Secret Groups**:
    - `SSO_CLIENT_SECRET`
    - `ADMIN_TOKEN` (The escaped `$$` string)
- **Agent:** Deploy using the **Self-Hosted On-Prem Agent** already present on the Huenei network.

### 2. Production Hardening (Apply to `compose.yaml`)
Update the environment section to enforce our security standards:
```yaml
environment:
  DOMAIN: "https://vw.huenei.net"
  SIGNUPS_ALLOWED: "false"
  INVITATIONS_ALLOWED: "true"
  SSO_ENABLED: "true"
  # SSO_ONLY: "true" # Enable after the 4th user is confirmed
```

### 3. Backup Strategy
- **Target:** `./vw-data/db.sqlite3` and `config.json`.
- **Action:** Implement a cron-job on the host VM to rsync the encrypted `vw-data` folder to a secure network share or Azure Blob Storage.

### 4. Team Onboarding Workflow
1. **Admin:** Log in to `/admin` using the Master Password.
2. **Invite:** Add the 3 remaining DevSecOps team emails.
3. **Verify:** Each user logs in once via SSO to initialize their vault.
4. **Lockdown:** Set `SSO_ONLY: "true"` in the pipeline and redeploy.

---

## 📅 Timeline
- **Week 1:** Team Demo & Feedback.
- **Week 2:** Azure DevOps Pipeline migration & Backup testing.
- **Week 3:** Full production cut-over.
