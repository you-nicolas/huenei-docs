# ProtectDiv: Project DevOps Work Notes

**Date**: March 13, 2026
**Project**: ProtectDiv (formlogic-api & formlogic-web)
**Objective**: Establish a professional DevOps lifecycle using Azure and GitHub Actions.

---

## 1. Executive Summary

We have successfully transitioned the ProtectDiv project from manual deployments to a fully automated **Trunk-Based Development (TBD)** pipeline. The infrastructure is now defined as code (IaC), and we have established a secure, secret-less connection between GitHub and Azure using OIDC.

---

## 2. Infrastructure as Code (Bicep)

* **Location**: `/infra` directory.
* **Strategy**: Modular Bicep templates that support environment parity.
* **Provisioned Resources**:
  * **Registry**: Centralized Azure Container Registry (ACR) `formlogicdevacr` used by all environments.
  * **Compute**: Azure Container Apps (Backend) and Azure Static Web Apps (Frontend).
  * **Database**: PostgreSQL Flexible Server (v16) with automated firewall rules for Azure services.
  * **Identity**: User-Assigned Managed Identity (`GitHub-Deployer-Identity`) for OIDC authentication.

---

## 3. CI/CD Pipeline (GitHub Actions)

We implemented a "Build Once, Deploy Many" promotion flow across two separate repositories.

### 3.1. Backend (`formlogic-api`)

* **Dockerfile**: Created a multi-stage .NET 9 production build.
* **OIDC Integration**: Implemented password-less login using Managed Identity.
* **Runner Hydration**: Fixed a critical issue where the deployment runner lacked project metadata for EF Core migrations by adding an explicit `dotnet restore` and `dotnet build` step.
* **Database Migrations**: Integrated `dotnet-ef` into the pipeline to automate schema updates.
* **Environment Sync**: Resolved 404/500 errors by correcting the `targetPort` (8080) and ensuring environment variables (`ASPNETCORE_ENVIRONMENT=Development`) and `ConnectionStrings` were correctly injected using comma-separated syntax.

### 3.2. Frontend (`formlogic-web`)

* **Build System**: Configured `pnpm` for building React 19 / Vite assets.
* **Variable Injection**: Integrated `VITE_API_BASE_URL` and feature flags (`VITE_USE_STUBS`, `VITE_MOCK_SESSION`) into the build-time environment.
* **SWA Deployment**: Automated deployment to Azure Static Web Apps using environment-scoped deployment tokens.

---

## 4. Key Technical Resolutions

* **Azure B2B Permissions**: Identified guest user restrictions in the client tenant. Pivoted from App Registrations to Managed Identities to unblock the OIDC setup.
* **Naming Convention Mismatch**: Identified and resolved a conflict where EF Core migrations used snake_case tables while the application code expected PascalCase.
* **Network/Port Sync**: Aligned the .NET 9 default container port (8080) with the Azure Container App ingress.

---

## 5. Current Environment Status

| Environment | Status | Configuration |
| :--- | :--- | :--- |
| **Development** | 🟢 Functional | Deployed, connected to DB, Swagger active. |
| **QA** | 🟡 Provisioned | Infrastructure created. Awaiting App Registration IDs from Lester to finalize GitHub variables. |
| **UAT / Prod** | ⚪ Planned | Documented in `prod-readiness-proposal.md`. |

---

## 6. Next Steps

1. **Finalize QA**: Add QA App Registration variables to GitHub and verify Alejandro's manual approval flow.
2. **Hardening**: Remove `continue-on-error: true` once the developers provide consistent migration files.
3. **Quality Gates**: Implement 60% code coverage check for Dev and 80% for Prod.
4. **Security**: Integrate container vulnerability scanning.
5. **Landing Zone**: Begin implementation of the Production VNET and Key Vault integration.

---
