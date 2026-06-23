# Gananor SAST Pipeline Documentation

This document provides a comprehensive overview of the standardized Static Application Security Testing (SAST) pipeline implemented for Gananor projects.

## 1. Overview

The SAST pipeline is a CI/CD process designed to automatically analyze the source code of our applications for security vulnerabilities, code quality issues, and bugs before they reach production. It also tracks test coverage to ensure our code is well-tested.

The pipeline integrates several key technologies:
- **Jenkins:** The orchestrator that runs the pipeline.
- **Docker:** Provides a consistent, isolated environment for building the application and running the analysis tools.
- **SonarQube:** The core static analysis engine that scans code for security vulnerabilities (SAST), bugs, and code smells. It also visualizes test coverage reports.
- **OWASP Dependency-Check:** A tool that scans third-party libraries for known vulnerabilities (Software Composition Analysis - SCA).

## 2. How It Works

The pipeline is designed to be reusable and maintainable by separating orchestration from execution.

### Jenkins & Shared Library

- **`Jenkinsfile`:** Each project has a minimal `Jenkinsfile`. Its sole purpose is to call the shared pipeline and provide project-specific parameters.
- **Shared Library (`pipelineSAST.groovy`):** The core logic for the entire SAST process is centralized in a Jenkins Shared Library. This ensures that all projects follow the exact same process and that updates to the pipeline logic can be made in a single place.

### Docker & Multi-Stage Builds

The `Dockerfile-SAST` in each project is the heart of the execution. It uses a multi-stage build to create a clean and efficient analysis environment:

1.  **`deps` Stage:** Installs the project's dependencies (e.g., `npm ci`). This stage is cached and only re-runs if the `package-lock.json` changes.
2.  **`build` Stage:** Copies the source code and runs linting and tests. Crucially, this is where the **test coverage report** (`coverage/lcov.info`) is generated. If tests fail, a marker file is created so Jenkins can mark the build as `UNSTABLE`.
3.  **`scan` Stage:** This stage uses a Java environment (`eclipse-temurin`), installs the SonarScanner CLI, copies the built application code and coverage report from the `build` stage, and executes the SonarQube scan.
4.  **Final Stage:** This stage uses the official `owasp/dependency-check` image, copies the source code, and runs a vulnerability scan on all third-party libraries.

## 3. Onboarding a New Project

To add a new project to this standardized SAST pipeline, follow these steps:

1.  **Prerequisites in `package.json`:**
    - Ensure the project has a test script that generates a coverage report in `lcov` format. For Node.js projects, this is typically `vitest run --coverage` or `jest --coverage`.
    - The script should be named `"coverage"`. For example: `"coverage": "dotenv -e .env.test -- vitest run --coverage"`.

2.  **Create `DevOps/Dockerfile-SAST`:**
    - Add a `Dockerfile-SAST` to the project's `DevOps` directory. Use the one from `gananor-services` as a template, adjusting the Node.js version and test commands as needed.

3.  **Create `.dockerignore`:**
    - Add a `.dockerignore` file to the project root to exclude unnecessary files from the build context, which speeds up the Docker build.

4.  **Create the `Jenkinsfile`:**
    - Add a `Jenkinsfile` to the project's corresponding folder within the `Jenkins-Pipelines` repository (on the `integration` branch).
    - This file calls the shared library and passes the required parameters:
      ```groovy
      @Library('Gananor-Shared-Library@main') _

      pipelineSAST {
          GIT_BRANCH = "dev" // The branch to scan
          GIT_REPO = "https://dev.azure.com/HueneiITS/Gananor/_git/your-repo-name"
          APP_NAME = "your-app-name-sast"
          DOCKERFILE_PATH = "DevOps/Dockerfile-SAST"
          PROJECT_KEY = "Your_SonarQube_Project_Key"
          CONFIG_FILE = "YourProject.env.test" // The name of the Managed File in Jenkins
          TARGET_LOCATION = ".env.test" // The name it should have in the workspace
      }
      ```

5.  **Configure Jenkins Managed File:**
    - If your project's tests require environment variables, create a "Managed file" in Jenkins containing the contents of your `.env.test` file.
    - The name of this file must match the `CONFIG_FILE` parameter in your `Jenkinsfile`.

## 4. Key Configuration Files

- **`Jenkinsfile`:** Defines the parameters for a specific project's SAST run. Stored in the shared library repository.
- **`DevOps/Dockerfile-SAST`:** Defines the multi-stage Docker build for testing and scanning the application. Stored in the application's repository.
- **`.dockerignore`:** Optimizes the Docker build by excluding files. Stored in the application's repository.
- **`gananor-shared-library/vars/pipelineSAST.groovy`:** The master pipeline script. Stored in the shared library repository.

## 5. Troubleshooting

- **0% Code Coverage in SonarQube:**
    - **Cause:** The tests are not running or the coverage report is not being found.
    - **Solution:**
        1. Check the pipeline logs. Look for errors during the `npm run coverage` step.
        2. Ensure the `.env.test` file is being correctly provided by the `configFileProvider` if your tests need it.
        3. Verify that the `sonar.javascript.lcov.reportPaths` property in your `Dockerfile-SAST` correctly points to the `lcov.info` file (usually `coverage/lcov.info`).

- **`sh: dotenv: not found` Error:**
    - **Cause:** The `dotenv-cli` package was not installed correctly during the `npm ci` step. This is almost always due to the `package-lock.json` file being out of sync with `package.json`.
    - **Solution:**
        1. In your local development environment, delete `node_modules` and `package-lock.json`.
        2. Run `npm install` to generate a fresh, correct `package-lock.json`.
        3. Commit the updated `package-lock.json` file to your repository.

- **Dependency-Check Warnings:**
    - **Cause:** The scanner may have trouble identifying project dependencies if the `node_modules` directory is not correctly copied into the final scanner stage.
    - **Solution:** Review the `COPY` commands in the `Dockerfile-SAST` to ensure the application source, including `node_modules`, is available in the final `owasp/dependency-check` stage.
