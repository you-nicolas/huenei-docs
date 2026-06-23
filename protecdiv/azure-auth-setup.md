# Azure App Registration Configuration Guide (ProtectDiv)

This guide provides step-by-step instructions to configure the Azure App Registrations for the **FormLogic** (ProtectDiv) project. 

We have two main components:
1.  **Frontend**: Azure Static Web App (React)
2.  **Backend**: Azure Container App (.NET API)

Please perform these steps for each environment: **Dev**, **QA**, and **UAT**.

---

## 1. Information to Collect (Overview)

First, we need the basic identifiers from the **Overview** blade of the App Registration.

1.  Search for **Microsoft Entra ID** (formerly Azure AD) in the Azure Portal.
2.  Select **App registrations** from the left menu.
3.  Click on the relevant app (e.g., `FormLogic-Dev`).
4.  Copy the following values from the **Overview** pane:
    *   **Application (client) ID**: (e.g., `00000000-0000-0000-0000-000000000000`)
    *   **Directory (tenant) ID**: (e.g., `00000000-0000-0000-0000-000000000000`)

---

## 2. Expose an API (Backend Configuration)

To allow the Frontend to securely call the Backend, we must define the Backend as an "API" and create a permission scope.

1.  In the left menu, click **Expose an API**.
2.  **Application ID URI**:
    *   Click **Set** next to "Application ID URI".
    *   Accept the default `api://<client-id>` and click **Save**.
    *   *Note: This URI is the **Audience** required by the backend developer.*
3.  **Scopes defined by this API**:
    *   Click **+ Add a scope**.
    *   **Scope name**: `access_as_user`
    *   **Who can consent?**: `Admins and users`
    *   **Admin consent display name**: `Access FormLogic API as user`
    *   **Admin consent description**: `Allows the application to access the FormLogic API on behalf of the signed-in user.`
    *   **State**: `Enabled`
    *   Click **Add scope**.

---

## 3. Authentication & Redirect URIs

### A. For the Frontend (React / Static Web App)
The Frontend **MUST** have a Redirect URI configured so users can be sent back to the app after logging in.

1.  In the left menu, click **Authentication**.
2.  Click **+ Add a platform** -> **Single-page application (SPA)**.
3.  **Redirect URIs**: 
    *   For **Dev**: Add `http://localhost:5173` (for local dev) and your dev URL (e.g., `https://dev.protectdiv.com`).
    *   For **QA/UAT**: Add the corresponding Static Web App URL.
4.  Click **Configure**.

### B. For the Backend (Container App)
*   **Case 1: In-code Auth (Recommended)**: If the developers are validating tokens in the .NET code, **NO Redirect URI is needed** for the backend registration.
*   **Case 2: Easy Auth (Platform-level)**: If we use the "built-in" Azure Container App authentication, you must add a **Web** platform (not SPA) with the following Redirect URI:
    *   `https://<your-container-app-url>/.auth/login/aad/callback`

---

## 4. Summary Table for Development Team

Once configured, please fill out this table and return it to the DevOps/Backend team:

| Environment | Client ID | Tenant ID | Audience (App ID URI) | Scope URI |
| :--- | :--- | :--- | :--- | :--- |
| **Dev** | | | `api://<client-id>` | `api://<client-id>/access_as_user` |
| **QA** | | | `api://<client-id>` | `api://<client-id>/access_as_user` |
| **UAT** | | | `api://<client-id>` | `api://<client-id>/access_as_user` |

---

## Summary of Technical Requirements

| Requirement | Value / Location | Why? |
| :--- | :--- | :--- |
| **Client ID** | Overview | Identifies the application. |
| **Tenant ID** | Overview | Identifies the Azure Directory. |
| **Audience** | Expose an API -> App ID URI | Tells the backend: "This token was made for you." |
| **Scopes** | Expose an API -> Scopes | Tells the frontend: "You have permission to call this API." |
| **Redirect URI** | Authentication -> SPA | Tells Azure: "Send the user back here after login." |
