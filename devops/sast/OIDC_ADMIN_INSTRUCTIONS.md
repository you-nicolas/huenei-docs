# Instructions for Azure Admin: Setting Up OIDC for GitHub Actions

Hi,

Please follow these steps to create a secure, passwordless OpenID Connect (OIDC) trust relationship between our Azure subscription and our GitHub Actions CI/CD pipeline. This is the modern, recommended approach for Azure/GitHub integration.

The process involves creating a single identity in Azure AD and configuring it to trust deployment requests coming from both our `test` and `uat` branches of the GitHub repository.

---

### Information I Need Back

After you have completed the steps, I will need the following three values to configure the pipeline:

1.  **Application (client) ID**
2.  **Directory (tenant) ID**
3.  **Azure Subscription ID**

---

## Execution Steps

### Part 1: Commands for Execution (Copy/Paste)

Please execute the entire script block below. It will prompt you for the necessary values as it proceeds.

```bash
#!/bin/bash

# --- Configuration ---
# Please set the following variables before running the rest of the script.
# This is the name for the identity that will be created in Azure AD.
DISPLAY_NAME="GitHubActions-GameStrong"

# This is the target Azure Subscription ID.
SUBSCRIPTION_ID="425f7188-bb6e-4e52-97d6-60f23188f6b1"

# This is the target Resource Group where the pipeline will deploy resources.
# Both test and uat resources are located here.
RESOURCE_GROUP="gamestrong-rg"

# --- Script ---
echo "Step 1: Creating the Application Registration in Azure AD..."
APP_ID=$(az ad app create --display-name "$DISPLAY_NAME" --query appId -o tsv)

if [ -z "$APP_ID" ]; then
    echo "Error: Failed to create Application Registration. Aborting."
    exit 1
fi
echo "Successfully created App Registration. App (Client) ID is: $APP_ID"
echo "------------------------------------------------------------------"


echo "Step 2: Creating the Service Principal..."
SP_OBJECT_ID=$(az ad sp create --id "$APP_ID" --query id -o tsv)

if [ -z "$SP_OBJECT_ID" ]; then
    echo "Error: Failed to create Service Principal. Aborting."
    exit 1
fi
echo "Successfully created Service Principal."
echo "------------------------------------------------------------------"


echo "Step 3: Assigning 'Contributor' role to the Service Principal..."
az role assignment create \
  --role "Contributor" \
  --subscription "$SUBSCRIPTION_ID" \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

echo "Successfully assigned 'Contributor' role to the shared resource group."
echo "------------------------------------------------------------------"


echo "Step 4: Creating Federated Credential for the 'test' branch..."
az ad app federated-credential create --id "$APP_ID" --params "{
    \"name\": \"gamestrong-appservices-test-branch\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:Gamestrong/AppServices:ref:refs/heads/test\",
    \"description\": \"Trust the test branch of the AppServices repo\",
    \"audiences\": [
        \"api://AzureADTokenExchange\"
    ]
}"

echo "Successfully created Federated Credential for 'test' branch."
echo "------------------------------------------------------------------"


echo "Step 5: Creating Federated Credential for the 'uat' branch..."
az ad app federated-credential create --id "$APP_ID" --params "{
    \"name\": \"gamestrong-appservices-uat-branch\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:Gamestrong/AppServices:ref:refs/heads/uat\",
    \"description\": \"Trust the uat branch of the AppServices repo\",
    \"audiences\": [
        \"api://AzureADTokenExchange\"
    ]
}"

echo "Successfully created Federated Credential for 'uat' branch."
echo "------------------------------------------------------------------"


echo "Setup Complete."
echo "Please provide the developer with the App (Client) ID: $APP_ID"
echo "------------------------------------------------------------------"

```

---

### Part 2: Command Reference (with Explanations)

This section provides a commented version of the script above for your reference.

```bash
#!/bin/bash

# --- Configuration ---
# Please set the following variables before running the rest of the script.

# This is the name for the identity that will be created in Azure AD.
DISPLAY_NAME="GitHubActions-GameStrong"

# This is the target Azure Subscription ID.
SUBSCRIPTION_ID="425f7188-bb6e-4e52-97d6-60f23188f6b1"

# This is the target Resource Group where the pipeline will deploy resources.
# Both test and uat resources are located here.
RESOURCE_GROUP="gamestrong-rg"

# --- Script ---

# Step 1: Create the Application Registration.
# This is the core identity of our pipeline in Azure Active Directory.
# We capture the output 'appId' (Client ID) for use in subsequent steps.
echo "Step 1: Creating the Application Registration in Azure AD..."
APP_ID=$(az ad app create --display-name "$DISPLAY_NAME" --query appId -o tsv)

if [ -z "$APP_ID" ]; then
    echo "Error: Failed to create Application Registration. Aborting."
    exit 1
fi
echo "Successfully created App Registration. App (Client) ID is: $APP_ID"
echo "------------------------------------------------------------------"


# Step 2: Create the Service Principal.
# This creates the "instance" of the application in the tenant, which is the entity
# that can be assigned roles and permissions.
echo "Step 2: Creating the Service Principal..."
SP_OBJECT_ID=$(az ad sp create --id "$APP_ID" --query id -o tsv)

if [ -z "$SP_OBJECT_ID" ]; then
    echo "Error: Failed to create Service Principal. Aborting."
    exit 1
fi
echo "Successfully created Service Principal."
echo "------------------------------------------------------------------"


# Step 3: Assign the 'Contributor' role.
# This grants the Service Principal the necessary permissions to create and manage
# resources, but limits its scope to only the specified resource group.
# Since both test and uat resources are in 'gamestrong-rg', this single assignment covers both.
echo "Step 3: Assigning 'Contributor' role to the Service Principal..."
az role assignment create \
  --role "Contributor" \
  --subscription "$SUBSCRIPTION_ID" \
  --assignee-object-id "$SP_OBJECT_ID" \
  --assignee-principal-type ServicePrincipal \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

echo "Successfully assigned 'Contributor' role to the shared resource group."
echo "------------------------------------------------------------------"


# Step 4: Create the Federated Credential for the 'test' branch.
# This is the key OIDC step. It establishes a trust relationship, telling Azure AD
# to accept tokens from GitHub's token issuer for the specified repository and the 'test' branch.
echo "Step 4: Creating Federated Credential for the 'test' branch..."
az ad app federated-credential create --id "$APP_ID" --params "{
    \"name\": \"gamestrong-appservices-test-branch\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:Gamestrong/AppServices:ref:refs/heads/test\",
    \"description\": \"Trust the test branch of the AppServices repo\",
    \"audiences\": [
        \"api://AzureADTokenExchange\"
    ]
}"

echo "Successfully created Federated Credential for 'test' branch."
echo "------------------------------------------------------------------"


# Step 5: Create the Federated Credential for the 'uat' branch.
# This creates a second trust relationship for the 'uat' branch, allowing it to also deploy.
echo "Step 5: Creating Federated Credential for the 'uat' branch..."
az ad app federated-credential create --id "$APP_ID" --params "{
    \"name\": \"gamestrong-appservices-uat-branch\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:Gamestrong/AppServices:ref:refs/heads/uat\",
    \"description\": \"Trust the uat branch of the AppServices repo\",
    \"audiences\": [
        \"api://AzureADTokenExchange\"
    ]
}"

echo "Successfully created Federated Credential for 'uat' branch."
echo "------------------------------------------------------------------"


echo "Setup Complete."
echo "Please provide the developer with the App (Client) ID: $APP_ID"
echo "------------------------------------------------------------------"
```