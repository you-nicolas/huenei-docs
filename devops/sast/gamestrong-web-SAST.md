# SAST Pipeline Implementation for gamestrong-web

## Objective

Implement a SAST (Static Application Security Testing) pipeline using Jenkins, SonarQube, and OWASP Dependency-Check for the `gamestrong-web` repository.

## Approach

A project-specific `Dockerfile-SAST` is used to define the analysis environment. This provides clarity and isolation for the project's build and test process, while leveraging the existing `pipelineSAST.groovy` shared library in Jenkins.

## 1. Create `.dockerignore`

This file is crucial for optimizing the Docker build process. It prevents large or unnecessary local directories, like `node_modules`, from being sent to the Docker daemon, speeding up the build context upload.

```
# See https://help.github.com/articles/ignoring-files/ for more about ignoring files.

# dependencies
node_modules

# testing
coverage

# production
dist
build

# misc
.DS_Store
*.pem

# debug
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# local env files
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# vercel
.vercel

# Docker
Dockerfile
.dockerignore
```

## 2. Create `Dockerfile-SAST`

This file, placed in the `DevOps/` directory, defines the multi-stage process for analyzing the project. It is based on the project's `azure-pipelines.yml` for build steps and the standard SAST procedure for scanning.

```dockerfile
FROM node:22-alpine AS base

# Install dependencies only when needed
FROM base AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install dependencies based on the preferred package manager
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then yarn global add pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi


# Rebuild the source code only when needed
FROM base AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules/
COPY . .

RUN npm run lint
RUN npm run build
# Allow the build process to continue even if tests fail
RUN npm run test:coverage || echo "Tests failed but continuing"

FROM openjdk:17-jdk-slim AS scan
WORKDIR /app
COPY --from=build . .
# Explicitly copy the coverage report from the build stage
COPY --from=build /app/coverage ./coverage

# Se declaran argumentos necesarios
ARG sonarScannerVersion=5.0.1.3006
ARG projectKey
ARG hostUrl
ARG loginToken

# Se instalan herramientas necesarias
RUN apt-get update && apt-get install -y wget unzip

# Se instala Sonar Scanner
RUN wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-${sonarScannerVersion}-linux.zip && \
    unzip sonar-scanner-cli-${sonarScannerVersion}-linux.zip -d /opt && \
    rm sonar-scanner-cli-${sonarScannerVersion}-linux.zip
ENV PATH="${PATH}:/opt/sonar-scanner-${sonarScannerVersion}-linux/bin"

# Se ejecuta Sonar Scanner
RUN sonar-scanner -Dsonar.projectKey=${projectKey} \
                  -Dsonar.sources=. \
                  -Dsonar.host.url=${hostUrl} \
                  -Dsonar.login=${loginToken} \
                  -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info \
                  -Dsonar.coverage.exclusions="**/node_modules/**,opt/**,usr/**" \
                  -Dsonar.exclusions="**/node_modules/**,**/coverage/**,**/*.test.ts,**/*.test.tsx,**/*.spec.ts,**/*.spec.tsx,**/public/**,**/dist/**" \
                  -X

FROM owasp/dependency-check:latest
WORKDIR /src
COPY --from=scan /app /src
```

## 3. Create `Jenkinsfile`

This file, stored in a central repository, triggers the SAST pipeline with the correct parameters for this project.

```groovy
@Library('Shared-Library@main') _

pipelineSAST {
    GIT_BRANCH = "test"
    GIT_REPO = "https://dev.azure.com/HueneiITS/GameStrong/_git/gamestrong-web"
    APP_NAME= "gamestrong-web-sast"
    DOCKERFILE_PATH = "DevOps/Dockerfile-SAST"
    PROJECT_KEY = "GameStrong_Web_FE"
}
```

## Troubleshooting Notes

- **Initial 0% Coverage:** The root cause was that the `coverage` directory, generated in the `build` stage, was not being copied to the `scan` stage. This was resolved by adding an explicit `COPY --from=build /app/coverage ./coverage` command to the `scan` stage in the `Dockerfile-SAST`.
- **Incorrect Directories in SonarQube Report:** The report initially included system directories like `/opt/` and `/usr/`. This was caused by the LCOV report containing absolute paths to Node.js runtime files. The issue was resolved by adding the `sonar.coverage.exclusions` property to the `sonar-scanner` command to ignore these paths during coverage calculation.
