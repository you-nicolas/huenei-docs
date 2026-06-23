# GameStrong Client Environment Replication Guide

## 1. Introduction

This document provides a complete guide for provisioning the `gamestrong-services` (backend) and `gamestrong-web` (frontend) applications into `test` and `UAT` environments on a new Azure subscription.

The architecture consists of:
*   **Backend**: A containerized Node.js application on **Azure Container Apps**, using **Azure Database for MySQL**, **Azure Container Registry (ACR)**, and **Azure Key Vault**.
*   **Frontend**: A Vite-based React application on **Azure Static Web Apps**.
*   **CI/CD**: Automated builds and deployments using **GitHub Actions**.

The guide is designed to be reusable. By changing a single environment variable, you can provision either the `test` or `UAT` environment.

## 2. Prerequisites

Before you begin, ensure you have the following:
*   **Azure CLI**: You are logged into the client's Azure tenant (`az login`).
*   **Required Permissions**: Your user account has **`Contributor`** and **`User Access Administrator`** roles on the target Azure subscription or resource group.
*   **GitHub Permissions**: You have Admin rights on the `Gamestrong/AppServices` and `Gamestrong/Web` GitHub repositories to configure secrets and workflows.
*   **Tools**: `git` is installed locally or in your chosen shell environment.

## 3. Part 1: One-Time Subscription Setup

These commands only need to be run once per subscription.

### 3.1. Registering Azure Resource Providers

Run this script to ensure all necessary services are enabled for the subscription.

```bash
echo "Registering necessary Azure resource providers..."

# For Azure Database for MySQL
az provider register --namespace Microsoft.DBforMySQL

# For Azure Container Registry (ACR)
az provider register --namespace Microsoft.ContainerRegistry

# For Azure Key Vault
az provider register --namespace Microsoft.KeyVault

# For Azure Container Apps
az provider register --namespace Microsoft.App

# For Container Apps' underlying Log Analytics Workspace
az provider register --namespace Microsoft.OperationalInsights

# For Azure Static Web Apps
az provider register --namespace Microsoft.Web

echo "Registration commands sent. It may take several minutes for all providers to be fully registered."
```

### 3.2. Create Service Principal for GitHub Actions

This creates a dedicated identity for your CI/CD pipeline to authenticate with Azure.

```bash
# Create the Service Principal, scoped to the subscription
az ad sp create-for-rbac \
  --name "GitHubActions-GameStrong" \
  --role "Contributor" \
  --scopes "/subscriptions/$(az account show --query id -o tsv)" \
  --sdk-auth
```

**CRITICAL**: The command above will output a JSON object. **Copy the entire JSON block.** You will paste this into a GitHub secret named `AZURE_CREDENTIALS` in both repositories.

## 4. Part 2: Provisioning an Environment

This section details how to create all the necessary Azure resources.

### 4.1. Set Environment Variables

Copy the block below into your shell. **This is the only section you need to change to switch between `test` and `UAT`.**

```bash
# ------------------------------------------------------------------
# SET THE TARGET ENVIRONMENT HERE: 'test' or 'uat'
# ------------------------------------------------------------------
export ENVIRONMENT="test"

# --- Derived Resource Names (do not change) ---
# Test and UAT environments share the same resource group
export RESOURCE_GROUP="gamestrong-rg"
export LOCATION="eastus"
export ACR_NAME="crgamestronggs${ENVIRONMENT}"
export KEY_VAULT_NAME="kv-gamestrong-gs-${ENVIRONMENT}"
export MYSQL_SERVER_NAME="mysql-gamestrong-gs-${ENVIRONMENT}"
export CONTAINER_APP_ENV="caenv-gamestrong-gs-${ENVIRONMENT}"
export CONTAINER_APP_NAME="ca-gamestrong-gs-api-${ENVIRONMENT}"
export SWA_NAME="gamestrong-gs-web-${ENVIRONMENT}"

# --- Database Credentials (USE A STRONG, UNIQUE PASSWORD) ---
export MYSQL_ADMIN_USER="adminuser"
export MYSQL_ADMIN_PASSWORD="<REPLACE_WITH_A_STRONG_PASSWORD>"
export MYSQL_DB_NAME="gamestrong"
```

### 4.2. Provision Backend Infrastructure (`gamestrong-services`)

Run these commands to create the backend resources.

```bash
# Create Resource Group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create MySQL Flexible Server (consider a higher SKU for UAT/Prod)
az mysql flexible-server create \
  --name $MYSQL_SERVER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --admin-user $MYSQL_ADMIN_USER \
  --admin-password $MYSQL_ADMIN_PASSWORD \
  --tier Burstable \
  --sku-name Standard_B1s \
  --storage-size 32

# Create the specific database
az mysql flexible-server db create \
  --resource-group $RESOURCE_GROUP \
  --server-name $MYSQL_SERVER_NAME \
  --database-name $MYSQL_DB_NAME

# Create Azure Container Registry (ACR) and Key Vault
az acr create --name $ACR_NAME --resource-group $RESOURCE_GROUP --sku Basic
az keyvault create --name $KEY_VAULT_NAME --resource-group $RESOURCE_GROUP --location $LOCATION --enable-rbac-authorization

# Construct and store the database connection string in Key Vault
export HOST_NAME="$MYSQL_SERVER_NAME.mysql.database.azure.com"
export CONN_STRING="mysql://$MYSQL_ADMIN_USER:$MYSQL_ADMIN_PASSWORD@$HOST_NAME:3306/$MYSQL_DB_NAME"
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "MYSQL-CONNECTION-STRING" \
  --value "$CONN_STRING"

# Create the Container App Environment and the App itself
az containerapp env create \
  --name $CONTAINER_APP_ENV \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION
az containerapp create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINER_APP_ENV \
  --system-assigned
```

### 4.3. Provision Frontend Infrastructure (`gamestrong-web`)

```bash
# Create the Static Web App
az staticwebapp create \
   --name "$SWA_NAME" \
   --resource-group "$RESOURCE_GROUP" \
   --location "$LOCATION"

# Get the Static Web App's URL for the CORS configuration
export SWA_URL=$(az staticwebapp show --name "$SWA_NAME" --resource-group "$RESOURCE_GROUP" --query "defaultHostname" -o tsv)

# Configure Backend CORS to allow requests from the frontend
az containerapp update \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --set-ingress-cors-policy allowed-origins="[\"https://$SWA_URL\"]"

# Configure the frontend app's API URL
export BACKEND_URL=$(az containerapp show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --query "properties.configuration.ingress.fqdn" -o tsv)
az staticwebapp appsettings set \
  --name "$SWA_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --setting-names VITE_API_BASE_URL="https://$BACKEND_URL"
```

### 4.4. Grant Permissions

These commands grant the necessary permissions for the services to work together.

```bash
# Get the Container App's Managed Identity ID
export IDENTITY_ID=$(az containerapp show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --query identity.principalId -o tsv)

# Grant the Container App's identity permission to read secrets from Key Vault
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee-object-id $IDENTITY_ID \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/$KEY_VAULT_NAME"

# Grant the Container App's identity permission to pull images from ACR
az role assignment create \
  --role "AcrPull" \
  --assignee-object-id $IDENTITY_ID \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ContainerRegistry/registries/$ACR_NAME"

# Configure the Container App to use the identity to pull from ACR
az containerapp registry set \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --server "$ACR_NAME.azurecr.io" \
  --identity system

# Configure Container App Ingress
az containerapp ingress enable \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --type external \
  --target-port 5055
```

## 5. Part 3: Application Deployment via CI/CD

### 5.1. Configure GitHub Secrets

In **both** the `Gamestrong/AppServices` and `Gamestrong/Web` repositories, go to `Settings` > `Secrets and variables` > `Actions` and add the following secrets:

*   `AZURE_CREDENTIALS`: Paste the full JSON output from the `az ad sp create-for-rbac` command in step 3.2.

In the `Gamestrong/Web` repository, you also need the deployment token for each environment:

*   `AZURE_STATIC_WEB_APP_API_TOKEN_TEST`:
    *   **Value**: Run `az staticwebapp secrets list --name gamestrong-gs-test-web-swa --query "properties.apiKey" -o tsv` and paste the output.
*   `AZURE_STATIC_WEB_APP_API_TOKEN_UAT`:
    *   **Value**: Run `az staticwebapp secrets list --name gamestrong-gs-uat-web-swa --query "properties.apiKey" -o tsv` and paste the output.

### 5.2. `gamestrong-services` GitHub Workflow

In the `Gamestrong/AppServices` repository, create the file `.github/workflows/deploy.yml` with the following content. This single workflow handles deployment to both `test` and `uat` based on the branch name.

```yaml
name: Build and Deploy GameStrong Services

on:
  push:
    branches:
      - test
      - uat

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name }} # Sets the environment based on the branch name

    steps:
      - name: 'Checkout source code'
        uses: actions/checkout@v3

      - name: 'Set Environment Variables'
        id: vars
        run: |
          if [ "${{ github.ref_name }}" == "uat" ]; then
            echo "ACR_NAME=crgamestronggsuat" >> $GITHUB_OUTPUT
            echo "CONTAINER_APP_NAME=ca-gamestrong-gs-api-uat" >> $GITHUB_OUTPUT
            echo "RESOURCE_GROUP=gamestrong-rg" >> $GITHUB_OUTPUT
          else
            echo "ACR_NAME=crgamestronggstest" >> $GITHUB_OUTPUT
            echo "CONTAINER_APP_NAME=ca-gamestrong-gs-api-test" >> $GITHUB_OUTPUT
            echo "RESOURCE_GROUP=gamestrong-rg" >> $GITHUB_OUTPUT
          fi
          echo "IMAGE_NAME=gamestrong-api" >> $GITHUB_OUTPUT

      - name: 'Log in to Azure'
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: 'Log in to Azure Container Registry'
        uses: azure/docker-login@v1
        with:
          login-server: ${{ steps.vars.outputs.ACR_NAME }}.azurecr.io

      - name: 'Build and push Docker image'
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.vars.outputs.ACR_NAME }}.azurecr.io/${{ steps.vars.outputs.IMAGE_NAME }}:${{ github.sha }}

      - name: 'Deploy to Azure Container App'
        uses: azure/container-apps-deploy-action@v1
        with:
          imageToDeploy: ${{ steps.vars.outputs.ACR_NAME }}.azurecr.io/${{ steps.vars.outputs.IMAGE_NAME }}:${{ github.sha }}
          containerAppName: ${{ steps.vars.outputs.CONTAINER_APP_NAME }}
          resourceGroup: ${{ steps.vars.outputs.RESOURCE_GROUP }}
```

### 5.3. `gamestrong-web` GitHub Workflow

In the `Gamestrong/Web` repository, create `.github/workflows/deploy.yml`.

```yaml
name: Build and Deploy GameStrong Web

on:
  push:
    branches:
      - test
      - uat

jobs:
  build_and_deploy_job:
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name }}
    name: Build and Deploy Job
    steps:
      - uses: actions/checkout@v3

      - name: 'Set Environment Variables'
        id: vars
        run: |
          if [ "${{ github.ref_name }}" == "uat" ]; then
            echo "SWA_TOKEN=${{ secrets.AZURE_STATIC_WEB_APP_API_TOKEN_UAT }}" >> $GITHUB_OUTPUT
          else
            echo "SWA_TOKEN=${{ secrets.AZURE_STATIC_WEB_APP_API_TOKEN_TEST }}" >> $GITHUB_OUTPUT
          fi

      - name: Build and Deploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ steps.vars.outputs.SWA_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: "upload"
          app_location: "/"
          output_location: "dist"
```

## 6. Part 4: Provisioning the UAT Environment

To provision the UAT environment, simply go back to **Section 4.1**, change the environment variable to `export ENVIRONMENT="uat"`, and re-run all the commands in **Part 4**. This will create a parallel set of resources for UAT.

---

## 7. Part 5: Provisioning the Production Environment

This section details the provisioning of the **production** environment. It includes critical upgrades for performance, security, and reliability.

### 7.1. Production Environment Considerations

*   **Resource Isolation**: Production resources will be created in a separate resource group (`gamestrong-rg-prod`) and a dedicated Container App Environment to ensure complete isolation from non-production environments.
*   **Initial SKUs**: For the initial deployment, resources will be provisioned with the same cost-effective SKUs as the `test` environment. These can be scaled up at any time with minimal downtime as performance needs are established.
    *   **MySQL**: `Burstable` tier (`Standard_B1s`).
    *   **ACR**: `Basic` tier.
*   **Enhanced Security**: The MySQL database will have public access disabled and will be accessed securely via a private endpoint from the Container App's virtual network.

### 7.2. Set Production Environment Variables

```bash
# ------------------------------------------------------------------
# SET THE TARGET ENVIRONMENT TO 'prod'
# ------------------------------------------------------------------
export ENVIRONMENT="prod"

# --- Derived Resource Names for Production ---
export RESOURCE_GROUP="gamestrong-rg-${ENVIRONMENT}"
export LOCATION="eastus"
export ACR_NAME="crgamestronggs${ENVIRONMENT}"
export KEY_VAULT_NAME="kv-gamestrong-gs-${ENVIRONMENT}"
export MYSQL_SERVER_NAME="mysql-gamestrong-gs-${ENVIRONMENT}"
export CONTAINER_APP_ENV="caenv-gamestrong-gs-${ENVIRONMENT}"
export CONTAINER_APP_NAME="ca-gamestrong-gs-api-${ENVIRONMENT}"
export SWA_NAME="gamestrong-gs-web-${ENVIRONMENT}"

# --- Database Credentials (USE A STRONG, UNIQUE PASSWORD) ---
export MYSQL_ADMIN_USER="adminuser"
export MYSQL_ADMIN_PASSWORD="<REPLACE_WITH_A_STRONG_PRODUCTION_PASSWORD>"
export MYSQL_DB_NAME="gamestrong"
```

### 7.3. Provision Production Infrastructure

Run these commands to create the production resources.

```bash
# Create Production Resource Group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create Production MySQL Flexible Server (Burstable tier, public access disabled)
# NOTE: This can be scaled up to a GeneralPurpose tier later as needed.
az mysql flexible-server create \
  --name $MYSQL_SERVER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --admin-user $MYSQL_ADMIN_USER \
  --admin-password $MYSQL_ADMIN_PASSWORD \
  --tier Burstable \
  --sku-name Standard_B1s \
  --storage-size 32 \
  --public-access Disabled

# Create the specific database
az mysql flexible-server db create \
  --resource-group $RESOURCE_GROUP \
  --server-name $MYSQL_SERVER_NAME \
  --database-name $MYSQL_DB_NAME

# Create Production ACR (Basic tier) and Key Vault
# NOTE: This can be scaled up to a Standard tier later as needed.
az acr create --name $ACR_NAME --resource-group $RESOURCE_GROUP --sku Basic
az keyvault create --name $KEY_VAULT_NAME --resource-group $RESOURCE_GROUP --location $LOCATION --enable-rbac-authorization

# Construct and store the database connection string in Key Vault
export HOST_NAME="$MYSQL_SERVER_NAME.mysql.database.azure.com"
export CONN_STRING="mysql://$MYSQL_ADMIN_USER:$MYSQL_ADMIN_PASSWORD@$HOST_NAME:3306/$MYSQL_DB_NAME"
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "MYSQL-CONNECTION-STRING" \
  --value "$CONN_STRING"

# Create the Production Container App Environment. This also creates the underlying VNet.
az containerapp env create \
  --name $CONTAINER_APP_ENV \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# --- Configure Private Networking for Database ---

# Get the VNet name created by the Container App Environment
export VNET_NAME=$(az containerapp env show --name $CONTAINER_APP_ENV --resource-group $RESOURCE_GROUP --query "properties.vnetConfiguration.internalSubnetId" -o tsv | cut -d '/' -f 9)

# Create a Private DNS Zone for MySQL
az network private-dns zone create \
  --resource-group $RESOURCE_GROUP \
  --name "privatelink.mysql.database.azure.com"

# Link the VNet to the Private DNS Zone
az network private-dns link vnet create \
  --resource-group $RESOURCE_GROUP \
  --zone-name "privatelink.mysql.database.azure.com" \
  --name "MySQLLink" \
  --virtual-network $VNET_NAME \
  --registration-enabled false

# Get the Subnet ID from the Container App Environment
export SUBNET_ID=$(az containerapp env show --name $CONTAINER_APP_ENV --resource-group $RESOURCE_GROUP --query "properties.vnetConfiguration.internalSubnetId" -o tsv)

# Get the MySQL Server Resource ID
export MYSQL_ID=$(az mysql flexible-server show --name $MYSQL_SERVER_NAME --resource-group $RESOURCE_GROUP --query "id" -o tsv)

# Create the Private Endpoint for the MySQL Server
az network private-endpoint create \
  --name "mysql-private-endpoint" \
  --resource-group $RESOURCE_GROUP \
  --subnet $SUBNET_ID \
  --private-connection-resource-id $MYSQL_ID \
  --group-id "mysqlServer" \
  --connection-name "mysql-connection"

# --- Continue with App Creation ---

# Create the Container App itself
az containerapp create \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINER_APP_ENV \
  --system-assigned

# Create the Production Static Web App
az staticwebapp create \
   --name "$SWA_NAME" \
   --resource-group "$RESOURCE_GROUP" \
   --location "$LOCATION"

# Get the Static Web App's URL for the CORS configuration
export SWA_URL=$(az staticwebapp show --name "$SWA_NAME" --resource-group "$RESOURCE_GROUP" --query "defaultHostname" -o tsv)

az containerapp update \
  --name $CONTAINER_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --set-ingress-cors-policy allowed-origins="[\"https://$SWA_URL\"]"

# Configure the frontend app's API URL
export BACKEND_URL=$(az containerapp show --name $CONTAINER_APP_NAME --resource-group $RESOURCE_GROUP --query "properties.configuration.ingress.fqdn" -o tsv)
az staticwebapp appsettings set \
  --name "$SWA_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --setting-names VITE_API_BASE_URL="https://$BACKEND_URL"
```

### 7.4. Grant Production Permissions

Follow the same steps as in **Section 4.4** to grant the necessary permissions to the production Container App's managed identity.

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
