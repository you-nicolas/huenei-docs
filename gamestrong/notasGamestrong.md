- [GameStrong](#gamestrong)
  - [Propuesta técnica](#propuesta-técnica)
  - [Entorno Test Azure Huenei](#entorno-test-azure-huenei)
    - [Flujo Fase 1](#flujo-fase-1)
  - [Despliegue de Fase 1](#despliegue-de-fase-1)
    - [Intro](#intro)
    - [Part I: Manual Infrastructure Deployment](#part-i-manual-infrastructure-deployment)
      - [Prerequisites](#prerequisites)
      - [Step 1: Setup and Configuration](#step-1-setup-and-configuration)
      - [Step 2: Provision Core Infrastructure](#step-2-provision-core-infrastructure)
        - [2.1 Create Resource Group](#21-create-resource-group)
        - [2.2 Create and Configure MySQL Database](#22-create-and-configure-mysql-database)
        - [2.3 Create Container Registry and Key Vault](#23-create-container-registry-and-key-vault)
          - [2.4 Securely Store the Database Connection String](#24-securely-store-the-database-connection-string)
      - [Step 3: Build and Deploy Application](#step-3-build-and-deploy-application)
      - [Step 4: Create and Configure the Container App](#step-4-create-and-configure-the-container-app)
        - [4.1 Create the Container App Environment and Resource](#41-create-the-container-app-environment-and-resource)
        - [4.2 Grant Permissions to the App's Identity](#42-grant-permissions-to-the-apps-identity)
        - [4.3 Configure Ingress, Registry, and Health Probes](#43-configure-ingress-registry-and-health-probes)
        - [4.4 Link Secret and Deploy Final Revision](#44-link-secret-and-deploy-final-revision)
    - [Part II: Automated Deployment with Azure Pipelines (CI/CD)](#part-ii-automated-deployment-with-azure-pipelines-cicd)
      - [Step 1: Pipeline Prerequisites](#step-1-pipeline-prerequisites)
        - [A. Create the Azure DevOps Service Connection](#a-create-the-azure-devops-service-connection)
        - [B. Create a Pipeline Variable Group](#b-create-a-pipeline-variable-group)
      - [Step 2: The Pipeline YAML (azure-pipelines.yml)](#step-2-the-pipeline-yaml-azure-pipelinesyml)
    - [Part III: Ongoing Operations \& Troubleshooting](#part-iii-ongoing-operations--troubleshooting)
      - [Deploying Application Updates](#deploying-application-updates)
      - [Troubleshooting Guide](#troubleshooting-guide)
  - [gamestrong-web](#gamestrong-web)
    - [HUE-176 Provisioning the GameStrong Web Test Environment](#hue-176-provisioning-the-gamestrong-web-test-environment)
    - [1. Introduction](#1-introduction)
    - [2. Prerequisites](#2-prerequisites)
    - [3. Step-by-Step Provisioning Guide](#3-step-by-step-provisioning-guide)
      - [Step 1: Set Environment Variables](#step-1-set-environment-variables)
      - [Step 2: Create the Static Web App Resource](#step-2-create-the-static-web-app-resource)
      - [Step 2.1: Retrieve the Deployment Token](#step-21-retrieve-the-deployment-token)
      - [Step 3: Create the Azure DevOps Pipeline](#step-3-create-the-azure-devops-pipeline)
        - [Part A: Create the `azure-pipelines.yml` file](#part-a-create-the-azure-pipelinesyml-file)
        - [Part B: Create the Pipeline and Secret Variable in Azure DevOps](#part-b-create-the-pipeline-and-secret-variable-in-azure-devops)
      - [Step 4: Configure Application Environment Variables](#step-4-configure-application-environment-variables)
      - [Step 5: Configure Backend CORS](#step-5-configure-backend-cors)
    - [4. Verification](#4-verification)
  - [SAST, Code Quality, and Tooling Summary HUE-175](#sast-code-quality-and-tooling-summary-hue-175)
    - [1. Core Concepts: SonarQube, SAST, and Test Coverage](#1-core-concepts-sonarqube-sast-and-test-coverage)
    - [2. The Role of ESLint \& Prettier](#2-the-role-of-eslint--prettier)
    - [3. Project-Specific Implementations](#3-project-specific-implementations)
      - [`gamestrong-web`](#gamestrong-web-1)
      - [`gamestrong-app`](#gamestrong-app)
      - [`gamestrong-services`](#gamestrong-services)
  - [HUE-202 Troubleshooting `VITE_API_BASE_URL` in `gamestrong-web`](#hue-202-troubleshooting-vite_api_base_url-in-gamestrong-web)
    - [1. Initial Problem](#1-initial-problem)
    - [2. Hypothesis 1: Variable Not Provided at Build Time](#2-hypothesis-1-variable-not-provided-at-build-time)
    - [3. Hypothesis 2: `.env` File Precedence](#3-hypothesis-2-env-file-precedence)
    - [4. Hypothesis 3: Variable Not Reaching the Script](#4-hypothesis-3-variable-not-reaching-the-script)
    - [5. Final Root Cause and Solution](#5-final-root-cause-and-solution)
  - [Entorno cliente URLs](#entorno-cliente-urls)
    - [TEST](#test)
    - [UAT](#uat)
    - [PROD](#prod)

# GameStrong

## Propuesta técnica

Proyecto nuevo, Adrian Barreiro, Martin Santamaria

![propuesta diagrama](./notas/GameStrongDiagrama.png)

## Entorno Test Azure Huenei

Trabajar dentro de la subscripción DEVOPS, crear el resource group GameStrong

### Flujo Fase 1

![fase 1](./notas/GameStrongDiagramaFase1.png)

## Despliegue de Fase 1

### Intro

This document provides a comprehensive, step-by-step guide for deploying the containerized Node.js application to Azure. It details the setup for the internal test environment and serves as the primary blueprint for replicating this infrastructure on the client's Azure subscription for the production environment.

The architecture consists of a Node.js application running on Azure Container Apps, with images stored in Azure Container Registry (ACR), credentials managed by Azure Key Vault, and data persisted in Azure Database for MySQL.

### Part I: Manual Infrastructure Deployment

This section details the manual, one-time setup of the Azure infrastructure. These steps can be automated later using IaC.

#### Prerequisites

- Azure CLI (logged in to correct account and subscription)
- Docker
- App source code

#### Step 1: Setup and Configuration

Define environment variables for consistency.

~~~bash
# --- Core Infrastructure ---
export RESOURCE_GROUP="gamestrong-rg"
export LOCATION="eastus"

# --- Application Naming ---
export ACR_NAME="crgamestrong"
export KEY_VAULT_NAME="kv-gamestrong"
export CONTAINER_APP_ENV="caenv-gamestrong-test"
export CONTAINER_APP_NAME="ca-gamestrong-api-test"
export IMAGE_NAME_BASE="gamestrong-api-test"

# --- Database Credentials ---
export MYSQL_SERVER_NAME="mysql-gamestrong-test"
export MYSQL_LOCATION="canadacentral"
export MYSQL_ADMIN_USER="adminuser"
export MYSQL_ADMIN_PASSWORD="Huenei$3030"
export MYSQL_DB_NAME="gamestrong"
~~~

#### Step 2: Provision Core Infrastructure

##### 2.1 Create Resource Group

~~~bash
az group create --name $RESOURCE_GROUP --location $LOCATION
~~~

##### 2.2 Create and Configure MySQL Database

~~~bash
# Create the MySQL Flexible Server
az mysql flexible-server create \
  --name $MYSQL_SERVER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $MYSQL_LOCATION \
  --admin-user $MYSQL_ADMIN_USER \
  --admin-password $MYSQL_ADMIN_PASSWORD \
  --tier Burstable \
  --sku-name Standard_B1s \
  --storage-size 32

# Create the specific database within the server
az mysql flexible-server db create \
  --resource-group $RESOURCE_GROUP \
  --server-name $MYSQL_SERVER_NAME \
  --database-name $MYSQL_DB_NAME

# Create a firewall rule to allow connections from Azure services
az mysql flexible-server firewall-rule create \
    --resource-group $RESOURCE_GROUP \
    --name $MYSQL_SERVER_NAME \
    --rule-name "AllowAllWindowsAzureIps" \
    --start-ip-address "0.0.0.0" \
    --end-ip-address "255.255.255.255"
~~~

**Production Consideration**: The Standard_B1s (Burstable) tier is cost-effective for testing. For the client's production environment, evaluate performance needs and consider a General Purpose tier (Standard_D2ds_v4 or higher) for consistent performance. For enhanced security, the production database should also disable public access and be accessed via a Private Endpoint.

##### 2.3 Create Container Registry and Key Vault

~~~bash
# Create Azure Container Registry (ACR)
az acr create --name $ACR_NAME --resource-group $RESOURCE_GROUP --sku Basic

# Create Azure Key Vault
az keyvault create --name $KEY_VAULT_NAME --resource-group $RESOURCE_GROUP --location $LOCATION
~~~

**Production Consideration**: The Basic ACR SKU is sufficient for testing. The client's environment may require a Standard or Premium SKU to support features like private endpoints, geo-replication, or higher performance.

###### 2.4 Securely Store the Database Connection String

Store the connection string in a standard URI format, which is highly compatible with Node.js libraries.

~~~bash
# Construct the standard connection URI
export HOST_NAME="$MYSQL_SERVER_NAME.mysql.database.azure.com"
export CONN_STRING="mysql://$MYSQL_ADMIN_USER:$MYSQL_ADMIN_PASSWORD@$HOST_NAME:3306/$MYSQL_DB_NAME"

# Store the URI in Key Vault
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "MYSQL-CONNECTION-STRING" \
  --value "$CONN_STRING"
~~~

#### Step 3: Build and Deploy Application

Containerize the application and deploy it to Azure. This step should be run from the root of the application's source code.

~~~bash
# Define a version tag for your image
# This will later be replaced with pipeline build tag"
export IMAGE_TAG="v1.0.0"
export FULL_IMAGE_NAME="$ACR_NAME.azurecr.io/$IMAGE_NAME_BASE:$IMAGE_TAG"

# Log in, build, and push the image
az acr login --name $ACR_NAME
docker build -t $FULL_IMAGE_NAME .
docker push $FULL_IMAGE_NAME
~~~

#### Step 4: Create and Configure the Container App

This final set of steps creates the app, connects all the resources, and deploys the first revision.

##### 4.1 Create the Container App Environment and Resource

~~~bash
az containerapp env create \
  --name $CONTAINER_APP_ENV \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

az containerapp create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINER_APP_ENV \
  --system-assigned
~~~

##### 4.2 Grant Permissions to the App's Identity

~~~bash
# Get the app's managed identity ID
export IDENTITY_ID=$(az containerapp identity show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --query principalId -o tsv)
export ACR_ID=$(az acr show --name $ACR_NAME --resource-group $RESOURCE_GROUP --query id -o tsv)

# Grant Key Vault and ACR permissions
az keyvault set-policy --name $KEY_VAULT_NAME --resource-group $RESOURCE_GROUP --object-id $IDENTITY_ID --secret-permissions get
az role assignment create --assignee $IDENTITY_ID --role AcrPull --scope $ACR_ID
~~~

##### 4.3 Configure Ingress, Registry, and Health Probes

~~~bash
# 1. Set the registry to use the managed identity for pulls
az containerapp registry set \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --server "$ACR_NAME.azurecr.io" \
  --identity system

# 2. Enable external ingress, targeting the port your app listens on (5055)
az containerapp ingress enable \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --type external \
  --target-port 5055

# 3. Configure a custom health probe for stability
az containerapp update \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --liveness-probe-path "/healthz" `# Assumes a /healthz endpoint exists` \
  --liveness-probe-timeout 30
~~~

##### 4.4 Link Secret and Deploy Final Revision

~~~bash
# Get the secret's URI from Key Vault
export KV_SECRET_URI=$(az keyvault secret show --vault-name $KEY_VAULT_NAME --name "MYSQL-CONNECTION-STRING" --query id -o tsv)

# Link the Key Vault secret to the container app
az containerapp secret set \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --secrets "db-conn-string-from-kv=$KV_SECRET_URI"

# Update the app with the image and environment variable to create the first running revision
az containerapp update \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --image $FULL_IMAGE_NAME \
  --set-env-vars "DATABASE_URL=secretref:db-conn-string-from-kv"
~~~

### Part II: Automated Deployment with Azure Pipelines (CI/CD)

This section details setting up an Azure Pipeline to automate the build and deploy process.

#### Step 1: Pipeline Prerequisites

##### A. Create the Azure DevOps Service Connection

This provides a secure identity for your pipeline to authenticate with Azure.

1. In Azure DevOps, go to Project Settings -> Service connections.

2. Click New service connection, select Azure Resource Manager, then Service principal (automatic).

3. Choose your Subscription, name the connection (e.g., Azure-Gamestrong-Connection), and grant access to all pipelines.

4. Once created, find the Service Principal ID from the connection details and grant it AcrPush permissions on your registry.

##### B. Create a Pipeline Variable Group

Securely store configuration values like database credentials.

1. In Azure DevOps, go to Pipelines -> Library.

2. Click + Variable group, name it gamestrong-test-vars.

3. Add the following variables. Click the padlock icon for each password to make it a secret.

    - DB_HOST: mysql-gamestrong-test.mysql.database.azure.com

    - DB_ROOT_USER: adminuser

    - DB_ROOT_PASSWORD: (Your MySQL password)

#### Step 2: The Pipeline YAML (azure-pipelines.yml)

Place this file in the root of your source code repository.

~~~YAML
# azure-pipelines.yml
trigger:
- test

pool:
  vmImage: "ubuntu-22.04"

# Link the Variable Group to this pipeline
variables:
- group: gamestrong-test-vars

# Define static variables
- name: acrName
  value: "crgamestrong"
- name: containerAppName
  value: "ca-gamestrong-api-test"
- name: resourceGroup
  value: "gamestrong-rg"
- name: imageName
  value: "crgamestrong.azurecr.io/gamestrong-api-test:$(Build.BuildId)"

steps:
- task: AzureContainerApps@1
  displayName: Build and deploy Container App
  inputs:
    azureSubscription: "Azure-Gamestrong-Connection"
    appSourcePath: "$(System.DefaultWorkingDirectory)"
    acrName: "$(acrName)"
    # NOTE: acrUsername and acrPassword are removed for security.
    # The task uses the service connection's identity to push to ACR.
    dockerfilePath: "Dockerfile"
    imageToBuild: "$(imageName)"
    containerAppName: "$(containerAppName)"
    resourceGroup: "$(resourceGroup)"

- script: |
    sudo apt-get update && sudo apt-get install -y mysql-client
    mysql -h $(DB_HOST) -u $(DB_ROOT_USER) -p$(DB_ROOT_PASSWORD) < ./database/init.sql
  displayName: "Initialize MySQL Schema"
  workingDirectory: "$(System.DefaultWorkingDirectory)"
~~~

**Production Consideration**: For the client's production deployment, create a new pipeline triggering from a main or release branch. The Initialize MySQL Schema step should be reviewed; relying on application-led migrations is often more robust than running SQL scripts from a CI/CD agent.

### Part III: Ongoing Operations & Troubleshooting

#### Deploying Application Updates

Pushing code changes to the test branch will automatically trigger the pipeline to build and deploy a new revision.

#### Troubleshooting Guide

If the application fails, follow this checklist:

1. Check the System Logs First: This tells you why Azure thinks the container is unhealthy.

~~~bash
az containerapp logs show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --type "system"
~~~

2. Check the Application (Console) Logs: This shows the error message from your Node.js code itself.

~~~bash
az containerapp logs show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --type "console"
~~~

3. Common Issues & Solutions:

- "Probe of StartUp failed": The application is crashing on startup. Check the Console Logs for a specific error (e.g., database connection refused, port mismatch).

- "Probe of Liveness failed": The app started but is now unhealthy. Configure a dedicated /healthz endpoint and update the liveness probe (see Step 4.3).

- Image Pull Errors: Verify the app's identity has the AcrPull role on the Container Registry.

- Cannot GET / or 404: This is an application-level issue. The infrastructure is working. Test a valid API endpoint with curl.

## gamestrong-web

Front dev: Yasser Barzotto

### HUE-176 Provisioning the GameStrong Web Test Environment

Solución: Azure Static Web Apps

Otras opciones: Azure blob storage + Azure CDN

Azure App Service / Azure Container Apps

Pros:

Integración con Azure DevOps, tarea de deploy predefinida

Free tier suficiente para ambientes de prueba

Manejo automático de certificados ssl, CDN y dominios custom

Cons:

Menor configuración granular de cache CDN

### 1. Introduction

  This document provides a step-by-step guide for provisioning a continuous integration and
  deployment (CI/CD) environment for the GameStrong Web application on Microsoft Azure.

  The solution uses Azure Static Web Apps for hosting, integrated with Azure DevOps Pipelines
  for automated builds and deployments. All steps are performed using the Azure Command-Line
  Interface (CLI).

### 2. Prerequisites

- Azure CLI: Installed on your local machine.
- Azure DevOps Extension: The CLI extension for Azure DevOps is required. Install it by
     running az extension add --name azure-devops.
- Authentication: You must be logged into your Azure account via the CLI (az login).
- Permissions: You need sufficient permissions in:
  - Azure: To create resource groups and static web apps (e.g., Contributor role).
  - Azure DevOps: To create pipelines and service connections in the GameStrong project.

### 3. Step-by-Step Provisioning Guide

#### Step 1: Set Environment Variables

  Open a bash/zsh shell and execute the following commands to set up variables for this
  session. This will reduce repetition and errors.

~~~bash
# --- Azure Resource Configuration ---
RESOURCE_GROUP="gamestrong-rg"
SWA_NAME="gamestrong-web-test"
LOCATION="eastus2"
# --- Backend Configuration ---
BACKEND_URL="https://ca-gamestrong-api-test.happymeadow-d4eaf473.eastus.azurecontainerapps.io"
~~~

#### Step 2: Create the Static Web App Resource

 This command provisions the Static Web App in Azure

~~~bash
az staticwebapp create \
   --name "$SWA_NAME" \
   --resource-group "$RESOURCE_GROUP" \
   --location "$LOCATION" \
   --source "$REPO_URL" \
   --branch test \
   --login-with-ado
~~~

Note: check --login-with-ado option. Maybe should use another auth option.

#### Step 2.1: Retrieve the Deployment Token

  The pipeline needs a secret key (a deployment token) to authorize
  deployments to the new Static Web App.

  Run the following command to get the token and copy the output value. You
  will need it in the next step.

~~~bash
az staticwebapp secrets list \
  --name "$SWA_NAME" \
  --resource-group gamestrong-rg \
  --query "properties.apiKey" \
  --output tsv
~~~

#### Step 3: Create the Azure DevOps Pipeline

  This process involves two parts: adding the YAML file to your repository
  and creating the pipeline in the Azure DevOps UI.

##### Part A: Create the `azure-pipelines.yml` file

~~~yml
# azure-pipelines.yml

trigger:
- test

pool:
  vmImage: 'ubuntu-22.04'

stages:
- stage: build_and_deploy
  displayName: 'Build and Deploy'
  jobs:
  - job: build_and_deploy_job
    displayName: 'Build and Deploy Job'
    steps:
    - checkout: self
      submodules: true

    - task: NodeTool@0
      displayName: 'Install Node.js'
      inputs:
        versionSpec: '22.x'

    - script: |
        npm ci
      displayName: 'Install Dependencies'

    - script: |
        npm run lint
      displayName: 'Run Linter'

    - script: |
        npm run test
      displayName: 'Run Unit Tests'

    - script: |
        npm run build
      displayName: 'Build Application'

    - task: AzureStaticWebApp@0
      displayName: 'Deploy to Azure Static Web Apps'
      inputs:
        app_location: '.'
        output_location: 'dist'
        azure_static_web_apps_api_token: $(deployment_token)
~~~

##### Part B: Create the Pipeline and Secret Variable in Azure DevOps

   1. Navigate to your GameStrong project in Azure DevOps.
   2. Go to Pipelines and click "New pipeline".
   3. Select "Azure Repos Git" and choose your gamestrong-web repository.
   4. Select "Existing Azure Pipelines YAML file".
   5. Choose the test branch and the /azure-pipelines.yml path, then click
      "Continue".
   6. Click the "Variables" button in the top right.
   7. Click "New variable".
       - Name: deployment_token
       - Value: Paste the deployment token you copied in Step 3.
       - Check the box for "Keep this value secret".
       - Click "OK" and then "Save".
   8. Finally, click "Run" to save and trigger your new pipeline.

#### Step 4: Configure Application Environment Variables

  Set the runtime environment variables required by the React application. These are securely
  stored in the Static Web App's configuration.

~~~bash
az staticwebapp appsettings set \
  --name "$SWA_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --setting-names \
  VITE_APP_ENV="test" \
  VITE_API_BASE_URL="$BACKEND_URL" \
  VITE_ENABLE_MSW="false" \
  VITE_ANALYTICS_ID="" \
  VITE_ROBOTS_NOINDEX="true" \
  VITE_ROBOTS_DIRECTIVE="noindex,nofollow"
~~~

#### Step 5: Configure Backend CORS

  To allow the frontend application to communicate with your backend API, you must configure a
  Cross-Origin Resource Sharing (CORS) rule on the backend Azure Container App.

  First, retrieve the default URL of your new Static Web App:

~~~bash
SWA_URL=$(az staticwebapp show --name "$SWA_NAME" --resource-group "$RESOURCE_GROUP"
     --query "defaultHostname" -o tsv)
echo "Static Web App URL: https://$SWA_URL"
~~~

  Next, use this URL to update the backend's CORS policy:

~~~bash
az containerapp update \
  --name "ca-gamestrong-api-test" \
  --resource-group "$RESOURCE_GROUP" \
  --set-ingress-cors-policy allowed-origins="[\"https://$SWA_URL\"]"
~~~

### 4. Verification

  Once you have committed the final azure-pipelines.yml file, the pipeline will run in Azure
  DevOps. After it completes successfully, the test environment will be live and accessible at
  the URL retrieved in Step 5.

  Any future commits pushed to the test branch will automatically trigger this pipeline and
  deploy the new version.

## SAST, Code Quality, and Tooling Summary HUE-175

This document summarizes the relationship between static analysis (SAST), testing, and various code quality tools as used in the `gamestrong` project.

### 1. Core Concepts: SonarQube, SAST, and Test Coverage

The key is to understand the difference between what SonarQube does natively and what it imports from other tools.

- **Static Analysis (SAST, Linting, Formatting):**
  - This involves analyzing code **without executing it**.
  - **SonarQube's primary function is SAST**. It scans the source code to find security vulnerabilities, bugs, and maintainability issues ("code smells").
  - Linting and formatting are also forms of static analysis.

- **Dynamic Analysis (Testing & Coverage):**
  - This involves **executing the code** to check its behavior.
  - **SonarQube does NOT run your tests**. Your CI pipeline runs them using a tool like Vitest or Jest.
  - The most important output of the test run for SonarQube is the **coverage report** (e.g., `lcov.info`). SonarQube **imports** this report to visualize how much of the code is covered by tests.

### 2. The Role of ESLint & Prettier

Even though SonarQube performs its own analysis, ESLint and Prettier are crucial for different reasons and at different stages:

1. **Immediate Developer Feedback:** They integrate directly into code editors (like VS Code) to provide real-time feedback, catching errors and style issues as code is written. This is the "inner loop" of development.

2. **Automated Commit-Time Gatekeeping:** Using **Husky** and **lint-staged**, the project automatically formats and fixes code with Prettier and ESLint every time a developer makes a `git commit`. This ensures no poorly formatted or lint-erroring code enters the repository.

3. **CI/CD "Fail-Fast" Mechanism:** The `RUN npm run lint` command in the `Dockerfile-SAST` acts as a quick, early check in the pipeline. If linting errors are present, the build fails immediately, saving time and resources by not waiting for the longer SonarQube scan to finish.

### 3. Project-Specific Implementations

This section details the final state and troubleshooting journey for each project's SAST pipeline.

#### `gamestrong-web`

- **Summary:** Node.js 22, React, Vitest.
- **Process:** A project-specific `DevOps/Dockerfile-SAST` was created. A `.dockerignore` file was added to optimize the build context.
- **Troubleshooting Journey:**
  - **Initial 0% Coverage:** The first runs reported 0% coverage in SonarQube. Debugging revealed a two-part problem:
        1. The `lcov.info` report was being generated in the `build` stage but was not being copied to the later `scan` stage.
        2. The root cause was an incorrect `COPY . .` command that created a nested `/app/app` directory structure in the `scan` stage, preventing SonarQube from matching the report paths to the source files.
  - **Solution:** The `COPY` command in the `scan` stage was corrected to `COPY --from=build /app/. .`, which correctly copies the contents of the build stage (including the `coverage` directory) into a flat structure.
  - **Noisy Coverage Report:** The report initially included system directories from the Docker container (e.g., `/opt`, `/usr`).
  - **Solution:** The `sonar-scanner` command was updated with `-Dsonar.coverage.exclusions` to ignore these paths, resulting in a clean and accurate final report.

#### `gamestrong-app`

- **Summary:** Node.js 20, React Native, Jest.
- **Process:** A project-specific `DevOps/Dockerfile-SAST` and a `.dockerignore` file were created.
- **Proactive Configuration:** Based on learnings from the other projects, we proactively identified that the Jest configuration in `package.json` was missing the `collectCoverageFrom` property. The development team was asked to add this to ensure an accurate coverage report is generated from the start.
- **Troubleshooting Journey:**
  - **Dependency-Check Warnings:** The initial pipeline runs showed many warnings from the OWASP Dependency-Check scanner, indicating it could not find the `node_modules` directory.
  - **Root Cause:** Similar to other issues, the `node_modules` directory was not being correctly copied into the final `owasp/dependency-check` stage of the Docker build.
  - **Current Status (On Hold):** Several attempts to fix this by explicitly copying the `node_modules` directory did not resolve the issue. This indicates a more subtle problem with the filesystem inside the `owasp/dependency-check` image. This task is currently on hold per user request.

#### `gamestrong-services`

- **Summary:** Node.js 20.15.1, Express, Vitest.
- **Process:** A project-specific `DevOps/Dockerfile-SAST` was created.
- **Prerequisites:** The development team was asked to add `eslint`/`prettier` configurations and to update their `vitest.config.js` to include the `lcov` reporter and an `include` property. This was crucial for generating the correct reports.
- **Troubleshooting Journey:**
  - **Initial Test Failures:** The pipeline failed because the test suite could not run in the clean CI environment. The root cause was missing environment variables (e.g., `SHARED_SECRET`) required by the tests.
  - **Solution:** The user configured a Jenkins Managed File to securely provide a `.env.test` file to the build environment, allowing the tests to pass.
  - **0% Coverage:** After fixing the tests, SonarQube still reported 0% coverage. The root cause was discovered to be the `vitest.config.js` file being incorrectly listed in the `.dockerignore` file. This prevented the configuration from being copied into the container, meaning the `lcov` reporter was never activated.
  - **Solution:** The `vitest.config.js` line was removed from `.dockerignore`. This allowed the correct configuration to be used, the `lcov.info` file to be generated, and SonarQube to report the correct coverage.

## HUE-202 Troubleshooting `VITE_API_BASE_URL` in `gamestrong-web`

This document summarizes the troubleshooting steps taken to resolve an issue where the deployed Azure Static Web App was not correctly connecting to the backend API.

### 1. Initial Problem

The deployed frontend application was making API calls to its own domain instead of the configured backend URL. This was confirmed by inspecting the browser's network tab, which showed requests going to a relative path (e.g., `/api/...`).

### 2. Hypothesis 1: Variable Not Provided at Build Time

- **Theory:** Vite applications are statically built. For the backend URL to be included in the final JavaScript, the `VITE_API_BASE_URL` variable must be available during the `npm run build` command.
- **Action:** The `azure-pipelines.yml` file was modified to inject the backend URL (stored in a pipeline variable like `$(BACKEND_URL_DEV)`) into the build step using an `env:` block.
- **Result:** The issue persisted. This indicated the problem was more complex than simply providing the variable.

### 3. Hypothesis 2: `.env` File Precedence

- **Theory:** Vite's environment variable loading rules state that variables in `.env` files (e.g., `.env.production`) take precedence over system-level environment variables set by the pipeline. An empty variable in one of these files could be overriding the pipeline's value.
- **Action:** The user confirmed that only a `.env.example` file existed in the repository, which Vite ignores by default. This eliminated the precedence theory.

### 4. Hypothesis 3: Variable Not Reaching the Script

- **Theory:** The variable might be defined in the pipeline but not correctly passed from the `env:` block to the actual `npm` script environment.
- **Action:** The `build` script in `package.json` was modified to include a guard clause: `if [ -z "$VITE_API_BASE_URL" ]; then exit 1; fi`. This would fail the build if the variable was empty.
- **Result:** The build succeeded, proving the variable was present and not empty within the script's environment. This was confirmed by adding an `echo` command, which printed the correct backend URL to the pipeline log.

### 5. Final Root Cause and Solution

After further investigation, it was discovered that the `AzureStaticWebApp@0` task in the pipeline was performing its own automatic build using a tool called Oryx. This second build process did not have access to the environment variables defined in the manual `script` task, and it was overwriting the correctly built artifacts.

**The final solution was to:**

1. **Remove the manual `npm run build` step** from the pipeline to eliminate the redundant build.
2. **Pass the environment variable directly to the `AzureStaticWebApp@0` task** using its own `env:` block, as specified in the official Microsoft documentation.

This ensures that the Oryx build process receives the `VITE_API_BASE_URL` variable at the correct time, bakes it into the final JavaScript files, and deploys the correctly configured application.

**Corrected Pipeline Snippet:**

```yaml
- task: AzureStaticWebApp@0
  displayName: 'Deploy to Dev SWA'
  inputs:
    app_location: '.'
    output_location: 'dist'
    azure_static_web_apps_api_token: $(deployment_token_dev)
  env:
    VITE_API_BASE_URL: $(BACKEND_URL_DEV)
```

## Entorno cliente URLs

Gamestrong entorno cliente

### TEST

Web: <https://zealous-moss-0a445b50f.1.azurestaticapps.net>
api: <https://ca-gamestrong-gs-api-test.ambitiousgrass-d089940c.eastus.azurecontainerapps.io>
db: mysql-gamestrong-gs-test.mysql.database.azure.com

mysql -h mysql-gamestrong-gs-test.mysql.database.azure.com -P 3306 -u adminuser -p
password: <stored in Key Vault — not committed>

### UAT

web: <https://nice-tree-0a757950f.1.azurestaticapps.net>
api: <https://ca-gamestrong-gs-api-uat.ambitiousgrass-d089940c.eastus.azurecontainerapps.io>
db: mysql-gamestrong-gs-uat.mysql.database.azure.com

mysql -h mysql-gamestrong-gs-uat.mysql.database.azure.com -P 3306 -u adminuser -p
password: <stored in Key Vault — not committed>

### PROD

web: <https://gamestrong.ai>
api: <https://ca-gamestrong-gs-api-prod.agreeablesmoke-c37277b2.canadacentral.azurecontainerapps.io>
db: mysql-gamestrong-gs-prod.mysql.database.azure.com

mysql -h mysql-gamestrong-gs-prod.mysql.database.azure.com -P 3306 -u adminuser -p
password: <stored in Key Vault — not committed>
