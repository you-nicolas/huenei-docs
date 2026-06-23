# Troubleshooting `VITE_API_BASE_URL` in `gamestrong-web`

This document summarizes the troubleshooting steps taken to resolve an issue where the deployed Azure Static Web App was not correctly connecting to the backend API.

## 1. Initial Problem

The deployed frontend application was making API calls to its own domain instead of the configured backend URL. This was confirmed by inspecting the browser's network tab, which showed requests going to a relative path (e.g., `/api/...`).

## 2. Hypothesis 1: Variable Not Provided at Build Time

*   **Theory:** Vite applications are statically built. For the backend URL to be included in the final JavaScript, the `VITE_API_BASE_URL` variable must be available during the `npm run build` command.
*   **Action:** The `azure-pipelines.yml` file was modified to inject the backend URL (stored in a pipeline variable like `$(BACKEND_URL_DEV)`) into the build step using an `env:` block.
*   **Result:** The issue persisted. This indicated the problem was more complex than simply providing the variable.

## 3. Hypothesis 2: `.env` File Precedence

*   **Theory:** Vite's environment variable loading rules state that variables in `.env` files (e.g., `.env.production`) take precedence over system-level environment variables set by the pipeline. An empty variable in one of these files could be overriding the pipeline's value.
*   **Action:** The user confirmed that only a `.env.example` file existed in the repository, which Vite ignores by default. This eliminated the precedence theory.

## 4. Hypothesis 3: Variable Not Reaching the Script

*   **Theory:** The variable might be defined in the pipeline but not correctly passed from the `env:` block to the actual `npm` script environment.
*   **Action:** The `build` script in `package.json` was modified to include a guard clause: `if [ -z "$VITE_API_BASE_URL" ]; then exit 1; fi`. This would fail the build if the variable was empty.
*   **Result:** The build succeeded, proving the variable was present and not empty within the script's environment. This was confirmed by adding an `echo` command, which printed the correct backend URL to the pipeline log.

## 5. Final Root Cause and Solution

After further investigation, it was discovered that the `AzureStaticWebApp@0` task in the pipeline was performing its own automatic build using a tool called Oryx. This second build process did not have access to the environment variables defined in the manual `script` task, and it was overwriting the correctly built artifacts.

**The final solution was to:**

1.  **Remove the manual `npm run build` step** from the pipeline to eliminate the redundant build.
2.  **Pass the environment variable directly to the `AzureStaticWebApp@0` task** using its own `env:` block, as specified in the official Microsoft documentation.

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
