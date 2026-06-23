# Vaultwarden Team Demo & Feature Guide

This document provides a structured walkthrough for demonstrating the Vaultwarden POC to the Huenei IT Services team. It covers the core DevSecOps value proposition, setup instructions, and advanced features.

---

## 🚀 Part 1: The Demo Breakdown

### 1. The "Why": Solving Secret Sprawl
*   **Problem:** Insecure storage (Excel, .txt), lack of audit trails, and risk of credential leakage.
*   **Solution:** A centralized, Zero-Knowledge encrypted vault integrated with our existing identity provider.

### 2. Identity & Access (Entra ID SSO)
*   **Action:** Demonstrate login via **"Enterprise Single Sign-On"**.
*   **Key Value:** Automated offboarding (deactivate Entra ID -> deactivate Vault access) and no new passwords for the team to memorize.

### 3. MSP Multi-Project Isolation
Show how we maintain client privacy using the **Organizations** and **Collections** model:
*   **Organizations:** One per Client (e.g., `Client-A`, `Client-B`).
*   **Collections:** Internal folders within organizations (e.g., `[PROD] Database`, `[STAGING] API Keys`).

### 4. Developer Workflow: Browser Extension
*   **Setup:** Click Gear Icon -> Server URL: `https://vw.huenei.net` -> Login.
*   **Action:** Show "Auto-fill" for a web console (AWS, Azure, or Jenkins).
*   **Troubleshooting Tip:** If syncing fails, ensure the server is configured with `IP_HEADER="X-Forwarded-For"` (required for Cloudflare Tunnels) to prevent rate-limiting.

---

## 🛠️ Part 2: Key Features Deep-Dive

### 1. Bitwarden Send (Secure Sharing)
*   **What:** Share text or files (up to 500MB) via an ephemeral link.
*   **Control:** Set expiration dates, password protection, and view limits.
*   **Use Case:** Sharing credentials with contractors or temporary vendors.

### 2. Emergency Access
*   **What:** Designate a trusted contact (e.g., Security Lead) who can request vault access.
*   **The "Wait Period":** If the user doesn't deny the request within X days, access is granted.
*   **Use Case:** Business continuity for critical accounts.

### 3. Vault Health Reports
*   **Exposed Passwords:** Cross-references vault data with known data breaches.
*   **Reused/Weak Passwords:** Identifies security risks across the entire organization.
*   **Unsecured Websites:** Flags credentials being sent over `http://`.

### 4. Item Archiving
*   **What:** Hide items from search and auto-fill without deleting them.
*   **Use Case:** Preserving legacy server credentials for audit trails without cluttering the active vault.

### 5. API Key & CLI Support
*   **What:** Generate a Client ID/Secret for programmatic access.
*   **Use Case:** Integrating Vaultwarden secrets into local developer scripts or CI/CD pipelines.

### 6. Admin Panel (`/admin`)
*   **What:** Infrastructure management interface for the Huenei Ops team.
*   **Tools:** User invitation management, database diagnostics, and global configuration overrides.
*   **Security Note:** Access is controlled via a hashed `ADMIN_TOKEN`. In our setup, this is currently **disabled** as a security best practice.
*   **Tech Tip:** If enabling, dollar signs in the Argon2id hash must be escaped as `$$` in the `compose.yaml` to avoid Docker interpolation errors.

### 7. Native Push Notifications
*   **What:** Real-time synchronization across all devices (Mobile, Desktop, Browser).
*   **Benefit:** Updates to secrets are reflected instantly without manual refreshing.

---

## 📋 Part 3: Access Details for Demo
*   **Server URL:** `https://vw.huenei.net`
*   **Admin Access:** Refer to `PROJECT_DOCS.md` for the Master Password.
*   **Extension Config:** Set Server Type to "Self-hosted" before login.
