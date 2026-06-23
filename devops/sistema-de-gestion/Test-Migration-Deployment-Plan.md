# Deployment Plan: `test-migration` Environment

This document outlines the architecture and deployment plan for the new `test-migration` environment for the "Sistema de Gestion" project.

## 1. Final Architecture

The new environment consists of three services deployed as Docker containers on the test server (`192.168.1.142`).

| Service Name                          | Repository                          | Host Port | Domain / Access Point                                  |
| ------------------------------------- | ----------------------------------- | --------- | ------------------------------------------------------ |
| `huenei-sd-test-migration-front`      | `HUE-SD-Sistema-De-Gestion-FRONT`   | `8085`    | `sistemagestion-sd-test-migration.huenei.com.ar`       |
| `huenei-sd-test-migration-back`       | `HUE-SD-Sistema-De-Gestion`         | `8086`    | `http://192.168.1.142:8086`                            |
| `huenei-sd-test-migration-keycloak` | `HUE-SD-Sistema-De-Gestion-BACK`    | `8087`    | `http://192.168.1.142:8087`                            |

### Authentication Flow

The login process follows this sequence:

1. The **Frontend** sends user credentials to the **Node.js Backend**.
2. The **Node.js Backend** calls the **Java Keycloak Adapter**.
3. The **Java Keycloak Adapter** communicates with the main **Keycloak Server** to get a JWT.
4. The JWT is returned to the frontend for use in subsequent API calls.

## 2. Required Development Work (BLOCKING)

Deployment cannot succeed until the following code changes are made by the development team:

1. **Frontend Must Be Modified:**
    * **Problem:** The current plan to do a "1-to-1 replication" of the frontend will fail. The old frontend talks directly to Keycloak, but the new backend expects to handle the login.
    * **Required Action:** The frontend code on the `test-migration` branch **must be changed** to call the new `/api/login` endpoint on the Node.js backend (`http://192.168.1.142:8086/api/login`) instead of trying to communicate directly with Keycloak.

2. **Backend Authentication Middleware Must Be Fixed:**
    * **Problem:** The current `auth.js` middleware in the Node.js backend uses an incorrect validation method for Keycloak tokens (symmetric shared secret vs. asymmetric public key).
    * **Required Action:** The middleware must be replaced with a correct implementation that uses a library like `jose` to validate the token signature against Keycloak's public keys.

## 3. Deployment Artifacts

The following files have been generated in the `replication/` directory. They must be moved to their respective repositories on the `test-migration` branch.

| File                            | Destination Repository                |
| ------------------------------- | ------------------------------------- |
| `Jenkinsfile.frontend`          | `HUE-SD-Sistema-De-Gestion-FRONT`     |
| `Dockerfile.frontend`           | `HUE-SD-Sistema-De-Gestion-FRONT`     |
| `frontend.env.test`             | `HUE-SD-Sistema-De-Gestion-FRONT`     |
| `Jenkinsfile.backend`           | `HUE-SD-Sistema-De-Gestion`           |
| `backend.env`                   | `HUE-SD-Sistema-De-Gestion`           |
| `Jenkinsfile.keycloak-adapter`  | `HUE-SD-Sistema-De-Gestion-BACK`      |

## 4. Deployment Steps

Once the development work is complete:

1. **Move Files:** Copy the artifacts listed above into their target repositories.
2. **Create Jenkins Pipelines:** Create the three new pipeline jobs in Jenkins, configuring each to use the `test-migration` branch of its respective repository.
3. **Run Pipelines:** Trigger the jobs to build and deploy the three services to the test environment.
