## SAST, Code Quality, and Tooling Summary

This document summarizes the relationship between static analysis (SAST), testing, and various code quality tools as used in the `gamestrong` project.

### 1. Core Concepts: SonarQube, SAST, and Test Coverage

The key is to understand the difference between what SonarQube does natively and what it imports from other tools.

* **Static Analysis (SAST, Linting, Formatting):**
  * This involves analyzing code **without executing it**.
  * **SonarQube's primary function is SAST**. It scans the source code to find security vulnerabilities, bugs, and maintainability issues ("code smells").
  * Linting and formatting are also forms of static analysis.

* **Dynamic Analysis (Testing & Coverage):**
  * This involves **executing the code** to check its behavior.
  * **SonarQube does NOT run your tests**. Your CI pipeline runs them using a tool like Vitest or Jest.
  * The most important output of the test run for SonarQube is the **coverage report** (e.g., `lcov.info`). SonarQube **imports** this report to visualize how much of the code is covered by tests.

### 2. The Role of ESLint & Prettier

Even though SonarQube performs its own analysis, ESLint and Prettier are crucial for different reasons and at different stages:

1. **Immediate Developer Feedback:** They integrate directly into code editors (like VS Code) to provide real-time feedback, catching errors and style issues as code is written. This is the "inner loop" of development.

2. **Automated Commit-Time Gatekeeping:** Using **Husky** and **lint-staged**, the project automatically formats and fixes code with Prettier and ESLint every time a developer makes a `git commit`. This ensures no poorly formatted or lint-erroring code enters the repository.

3. **CI/CD "Fail-Fast" Mechanism:** The `RUN npm run lint` command in the `Dockerfile-SAST` acts as a quick, early check in the pipeline. If linting errors are present, the build fails immediately, saving time and resources by not waiting for the longer SonarQube scan to finish.

### 3. Project-Specific Implementations

This section details the final state and troubleshooting journey for each project's SAST pipeline.

#### `gamestrong-web`

* **Summary:** Node.js 22, React, Vitest.
* **Process:** A project-specific `DevOps/Dockerfile-SAST` was created. A `.dockerignore` file was added to optimize the build context.
* **Troubleshooting Journey:**
  * **Initial 0% Coverage:** The first runs reported 0% coverage in SonarQube. Debugging revealed a two-part problem:
        1. The `lcov.info` report was being generated in the `build` stage but was not being copied to the later `scan` stage.
        2. The root cause was an incorrect `COPY . .` command that created a nested `/app/app` directory structure in the `scan` stage, preventing SonarQube from matching the report paths to the source files.
  * **Solution:** The `COPY` command in the `scan` stage was corrected to `COPY --from=build /app/. .`, which correctly copies the contents of the build stage (including the `coverage` directory) into a flat structure.
  * **Noisy Coverage Report:** The report initially included system directories from the Docker container (e.g., `/opt`, `/usr`).
  * **Solution:** The `sonar-scanner` command was updated with `-Dsonar.coverage.exclusions` to ignore these paths, resulting in a clean and accurate final report.

#### `gamestrong-app`

* **Summary:** Node.js 20, React Native, Jest.
* **Process:** A project-specific `DevOps/Dockerfile-SAST` and a `.dockerignore` file were created.
* **Proactive Configuration:** Based on learnings from the other projects, we proactively identified that the Jest configuration in `package.json` was missing the `collectCoverageFrom` property. The development team was asked to add this to ensure an accurate coverage report is generated from the start.
* **Troubleshooting Journey:**
  * **Dependency-Check Warnings:** The initial pipeline runs showed many warnings from the OWASP Dependency-Check scanner, indicating it could not find the `node_modules` directory.
  * **Root Cause:** Similar to other issues, the `node_modules` directory was not being correctly copied into the final `owasp/dependency-check` stage of the Docker build.
  * **Current Status (On Hold):** Several attempts to fix this by explicitly copying the `node_modules` directory did not resolve the issue. This indicates a more subtle problem with the filesystem inside the `owasp/dependency-check` image. This task is currently on hold per user request.

#### `gamestrong-services`

* **Summary:** Node.js 20.15.1, Express, Vitest.
* **Process:** A project-specific `DevOps/Dockerfile-SAST` was created.
* **Prerequisites:** The development team was asked to add `eslint`/`prettier` configurations and to update their `vitest.config.js` to include the `lcov` reporter and an `include` property. This was crucial for generating the correct reports.
* **Troubleshooting Journey:**
  * **Initial Test Failures:** The pipeline failed because the test suite could not run in the clean CI environment. The root cause was missing environment variables (e.g., `SHARED_SECRET`) required by the tests.
  * **Solution:** The user configured a Jenkins Managed File to securely provide a `.env.test` file to the build environment, allowing the tests to pass.
  * **0% Coverage:** After fixing the tests, SonarQube still reported 0% coverage. The root cause was discovered to be the `vitest.config.js` file being incorrectly listed in the `.dockerignore` file. This prevented the configuration from being copied into the container, meaning the `lcov` reporter was never activated.
  * **Solution:** The `vitest.config.js` line was removed from `.dockerignore`. This allowed the correct configuration to be used, the `lcov.info` file to be generated, and SonarQube to report the correct coverage.
