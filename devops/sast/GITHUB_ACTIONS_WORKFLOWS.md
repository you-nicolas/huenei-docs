# GitHub Actions Workflows for GameStrong Project

This document contains the GitHub Actions workflow YAML files for the `gamestrong-services` (backend) and `gamestrong-web` (frontend) applications. These workflows are configured for OpenID Connect (OIDC) authentication with Azure and support both `test` and `uat` environments.

---

## 1. `gamestrong-services` (Backend) Workflow

This workflow builds the Docker image for the backend application, pushes it to Azure Container Registry (ACR), and deploys it to Azure Container Apps.

**File Location:**
Place this content in the client's `Gamestrong/AppServices` repository at: `.github/workflows/deploy.yml`

```yaml
name: Build and Deploy GameStrong Services

on:
  push:
    branches:
      - test
      - uat
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-22.04
    environment: ${{ github.ref_name }}
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v3

      - name: Set Environment Variables
        id: vars
        run: |
          if [ "${{ github.ref_name }}" == "uat" ]; then
            echo "ACR_NAME=crgamestronggs" >> $GITHUB_OUTPUT
            echo "CONTAINER_APP_NAME=gamestrong-gs-api-uat" >> $GITHUB_OUTPUT
            echo "RESOURCE_GROUP=gamestrong-rg" >> $GITHUB_OUTPUT
          else
            echo "ACR_NAME=crgamestronggs" >> $GITHUB_OUTPUT
            echo "CONTAINER_APP_NAME=ca-gamestrong-gs-api-test" >> $GITHUB_OUTPUT
            echo "RESOURCE_GROUP=gamestrong-rg" >> $GITHUB_OUTPUT
          fi
          echo "IMAGE_NAME=gamestrong-api" >> $GITHUB_OUTPUT

      - name: Log in to Azure
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Log in to Azure Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ steps.vars.outputs.ACR_NAME }}.azurecr.io

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.vars.outputs.ACR_NAME }}.azurecr.io/${{ steps.vars.outputs.IMAGE_NAME }}:${{ github.sha }}

      - name: Deploy to Azure Container App
        uses: azure/container-apps-deploy-action@v1
        with:
          imageToDeploy: ${{ steps.vars.outputs.ACR_NAME }}.azurecr.io/${{ steps.vars.outputs.IMAGE_NAME }}:${{ github.sha }}
          containerAppName: ${{ steps.vars.outputs.CONTAINER_APP_NAME }}
          resourceGroup: ${{ steps.vars.outputs.RESOURCE_GROUP }}
```

---

## 2. `gamestrong-web` (Frontend) Workflow

This workflow builds the React application and deploys it to Azure Static Web Apps. It dynamically retrieves the backend API URL and the Static Web App deployment token.

**File Location:**
Place this content in the client's `Gamestrong/Web` repository at: `.github/workflows/deploy.yml`

```yaml
name: Build and Deploy GameStrong Web

on:
  push:
    branches:
      - test
      - uat
  workflow_dispatch:

jobs:
  build_and_deploy_job:
    runs-on: ubuntu-22.04
    environment: ${{ github.ref_name }}
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v3

      - name: Set Environment Variables
        id: vars
        run: |
          if [ "${{ github.ref_name }}" == "uat" ]; then
            echo "SWA_NAME=gamestrong-gs-web-uat" >> $GITHUB_OUTPUT
            echo "BACKEND_CONTAINER_APP_NAME=gamestrong-gs-api-uat" >> $GITHUB_OUTPUT
            echo "RESOURCE_GROUP=gamestrong-rg" >> $GITHUB_OUTPUT
          else
            echo "SWA_NAME=gamestrong-gs-web-test" >> $GITHUB_OUTPUT
            echo "BACKEND_CONTAINER_APP_NAME=ca-gamestrong-gs-api-test" >> $GITHUB_OUTPUT
            echo "RESOURCE_GROUP=gamestrong-rg" >> $GITHUB_OUTPUT
          fi

      - name: Log in to Azure
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get Backend API URL
        id: get_backend_url
        run: |
          BACKEND_FQDN=$(az containerapp show \
            --name ${{ steps.vars.outputs.BACKEND_CONTAINER_APP_NAME }} \
            --resource-group ${{ steps.vars.outputs.RESOURCE_GROUP }} \
            --query "properties.configuration.ingress.fqdn" -o tsv)
          echo "BACKEND_API_BASE_URL=https://$BACKEND_FQDN" >> $GITHUB_OUTPUT

      - name: Get Static Web App Deployment Token
        id: get_swa_token
        run: |
          SWA_TOKEN=$(az staticwebapp secrets list \
            --name ${{ steps.vars.outputs.SWA_NAME }} \
            --resource-group ${{ steps.vars.outputs.RESOURCE_GROUP }} \
            --query "properties.apiKey" -o tsv)
          echo "SWA_DEPLOYMENT_TOKEN=$SWA_TOKEN" >> $GITHUB_OUTPUT

      - name: Deploy to Azure Static Web Apps
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ steps.get_swa_token.outputs.SWA_DEPLOYMENT_TOKEN }}
          repo_token: ${{ secrets.GITHUB_TOKEN }}
          action: "upload"
          app_location: "."
          output_location: "dist"
        env:
          VITE_API_BASE_URL: ${{ steps.get_backend_url.outputs.BACKEND_API_BASE_URL }}
```
