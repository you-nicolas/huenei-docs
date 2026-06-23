# Huenei Vaultwarden DevSecOps

This repository manages the deployment and configuration of **Vaultwarden** for Huenei IT Services. It provides a secure, self-hosted, SSO-integrated secret management solution designed for MSP (Managed Service Provider) multi-project isolation.

## 🚀 Quick Links
- [**Vaultwarden Demo & Features Guide**](VAULTWARDEN_DEMO_GUIDE.md) - **Start Here for Team Demos**
- [**Phase 2: Production Plan**](PRODUCTION_PLAN.md) - CI/CD & Hardening Roadmap
- [Project Documentation](PROJECT_DOCS.md) - Detailed architecture and setup instructions.
- [DevSecOps Guidelines (GEMINI.md)](GEMINI.md) - Project standards and critical configuration.

## 🏗️ Architecture Overview
- **Deployment:** Docker Compose on `docker-florida-03`.
- **Ingress:** Cloudflare Tunnels via `vw.huenei.net`.
- **Identity:** Entra ID (Azure AD) via OIDC SSO.
- **Isolation:** Multi-tenant support using Organizations and Collections.

## 🛠️ Getting Started
### 1. Requirements
- Docker & Docker Compose.
- Access to Huenei Azure DevOps (for CI/CD).
- Entra ID App Registration credentials.

### 2. Manual Setup (POC)
1. Configure `.env` with SSO credentials.
2. Run `docker compose up -d`.
3. Access the web vault at `https://vw.huenei.net`.

## 📜 Documentation Index
- `compose.yaml`: Infrastructure as Code.
- `PROJECT_DOCS.md`: Comprehensive technical documentation.
- `VAULTWARDEN_DEMO_GUIDE.md`: Guide for training the team.
- `GEMINI.md`: Development and operational mandates.
