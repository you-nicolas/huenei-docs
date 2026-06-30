- [Seguimiento Objetivos SEO](#seguimiento-objetivos-seo)
  - [BBDD](#bbdd)
  - [Bind mount](#bind-mount)
  - [Ambientes](#ambientes)
    - [Integracion](#integracion)
      - [front](#front)
      - [back](#back)
      - [db](#db)
    - [Prod](#prod)
      - [front](#front-1)
      - [back](#back-1)
      - [db](#db-1)
  - [dotenv - Variables de entorno según rama](#dotenv---variables-de-entorno-según-rama)
  - [HUE-232 Logging Implementation Plan](#hue-232-logging-implementation-plan)
    - [Objective](#objective)
    - [Current Status \& Problem](#current-status--problem)
    - [Proposed Solution: Local Persistent Logging (`syslog`)](#proposed-solution-local-persistent-logging-syslog)
      - [Strategy Explained: `syslog`, `rsyslog`, and `logrotate`](#strategy-explained-syslog-rsyslog-and-logrotate)
        - [Understanding the `logrotate` Configuration](#understanding-the-logrotate-configuration)
    - [Implementation Steps](#implementation-steps)
      - [Part 1: Modify Jenkins Shared Library (`pipelineDeployToServerBindMount.groovy`)](#part-1-modify-jenkins-shared-library-pipelinedeploytoserverbindmountgroovy)
      - [Part 2: Modify Application Jenkinsfiles (`Jenkinsfile-Prod`)](#part-2-modify-application-jenkinsfiles-jenkinsfile-prod)
      - [Part 3: Configure Host Machine (`192.169.0.101`)](#part-3-configure-host-machine-1921690101)
    - [Verification](#verification)
- [NaviJet](#navijet)
  - [Datos lado cliente](#datos-lado-cliente)
- [NaviTango](#navitango)
    - [INT](#int)
    - [QA](#qa)
      - [HUE-127](#hue-127)
    - [Prod](#prod-1)
  - [HUE-134](#hue-134)
- [NaviPdfProcess](#navipdfprocess)
  - [HUE-167 Dockerfile Optimization for NaviPdfProcess Service](#hue-167-dockerfile-optimization-for-navipdfprocess-service)
- [GameStrong](#gamestrong)
  - [Propuesta técnica](#propuesta-técnica)
  - [Entorno Test Azure Huenei](#entorno-test-azure-huenei)
    - [Flujo Fase 1](#flujo-fase-1)
- [SwissJust](#swissjust)
  - [HUE-157 Configuración HTTPS](#hue-157-configuración-https)
    - [**Step 1: Configure S3 Bucket for Static Website Hosting**](#step-1-configure-s3-bucket-for-static-website-hosting)
    - [**Step 2: Provision a Public SSL/TLS Certificate (ACM)**](#step-2-provision-a-public-ssltls-certificate-acm)
    - [**Step 3: Create and Configure the CloudFront Distribution**](#step-3-create-and-configure-the-cloudfront-distribution)
    - [**Step 4: Configure DNS Records in Azure**](#step-4-configure-dns-records-in-azure)
    - [**Step 5: Agregar registros a DNS interno Huenei**](#step-5-agregar-registros-a-dns-interno-huenei)
  - [HUE-173 Blank page \& 404 error](#hue-173-blank-page--404-error)
    - [Pasos](#pasos)
  - [HUE-174 CORS](#hue-174-cors)
    - [Detalle ticket](#detalle-ticket)
  - [HUE-162](#hue-162)
    - [**Ticket Summary: Automating CloudFront Cache Invalidation in CI/CD Pipeline**](#ticket-summary-automating-cloudfront-cache-invalidation-in-cicd-pipeline)
  - [HUE-185](#hue-185)
    - [**Ticket Summary: Continuing HUE-162 automating rest of services**](#ticket-summary-continuing-hue-162-automating-rest-of-services)
- [Sitio web Huenei](#sitio-web-huenei)


# Seguimiento Objetivos SEO

Proyecto interno
monorepo
Lugel

[Repo](https://dev.azure.com/HueneiITS/Seguimiento%20Objetivos%20SEO)

## BBDD

mysql 8.0
tiene los docker-compose.yml dentro de los servidores a donde deploya
/home/$USER/docker-compose/seguimiento-objetivos-seo

## Bind mount

El backend tiene un directorio '/app/data' con archivos planos para cache. Este tiene que ser persistente. Se implementó un bind mount volume. Se creó una variación del shared library pipelineDeployToServer.groovy: [link](https://dev.azure.com/HueneiITS/_git/Jenkins-Pipelines?path=%2FPipelines%2Fvars%2FpipelineDeployToServerBindMount.groovy&version=GBmain)

El pipeline de deploy ejecuta el docker run con la opción: 
~~~bash 
-v /home/${USER_NAME}/${APP_NAME}_data:/app/data
~~~

Crea y monta un volumen en el directorio home del usuario con el nombre de la app y lo vincula con '/app/data' dentro del contenedor.

Como la app escribe archivos dentro de este mismo directorio, se debe agregar los permisos al usuario node desde el host.

~~~bash
# modificar la ruta según necesidad
$ sudo chown -R 1000:1000 /home/root/seguimiento-objetivos-seo-prod-back_data/
# verificar dentro del contenedor
$ docker exec -it seguimiento-objetivos-seo-int-back sh
/app $ ls -ld data/
drwxr-xr-x    2 node     node             6 Jun 19 17:28 data/
~~~


## Ambientes

### Integracion

192.168.137.211 - integración Padua

#### front

seguimiento-objetivos-seo-int-front
puerto: 8060
[url](http://192.168.137.211:8060/)
[pipeline en Jenkins](http://192.168.1.155:8190/job/HUENEI-SEGUIMIENTO-OBJETIVOS-SEO/job/INTEGRACION/job/FRONT/job/OBJETIVOS-SEO-FE/)

#### back

seguimiento-objetivos-seo-int-back
puerto: 8061
[pipeline en Jenkins](http://192.168.1.155:8190/job/HUENEI-SEGUIMIENTO-OBJETIVOS-SEO/job/INTEGRACION/job/BACK/job/OBJETIVOS-SEO-BE/)

#### db

seguimiento-objetivos-seo-int-mysql
puerto: 8062

      MYSQL_USER: seguimientouser
      MYSQL_PASSWORD: Huenei$3030

### Prod

192.169.0.101 - Prod


front 192.169.0.101:8060
back 192.169.0.101:8061
db 192.169.0.101:8062


#### front

#### back

#### db

seguimiento-objetivos-seo-prod-mysql
puerto: 8062

      MYSQL_USER: seguimientouser
      MYSQL_PASSWORD: Huenei$3030



## dotenv - Variables de entorno según rama

El backend debe tomar variables de entorno según la rama que despliega. Por el momento tenemos int y prod. Se modificó el 'src/index.ts' y el Dockerfile. Se usó el módulo 'dotenv'

index.ts:
~~~ts
import dotenv from "dotenv";

// Cargar el .env genérico para variables compartidas
dotenv.config({ path: `.env` });

// Cargar el .env segun entorno definido en el Dockerfile
// Fallback a 'development'
const nodeEnv = process.env.NODE_ENV || 'development';

// Cargar .env especifico del entorno (e.g., .env.int, .env.prod)
dotenv.config({ path: `.env.${nodeEnv}` });

.
.
.

~~~

Dockerfile con comando ENV:
~~~Dockerfile
.
.
.

ENV NODE_ENV=production

# Start the app
CMD ["node", "dist/index.js"]
~~~

Evaluar si el frontend también necesita esta configuración

Se configuró las variables de entorno del front. Con Next.js solo hay que aclararar el entorno antes del 'npm run build'.

~~~Dockerfile
# Set NODE_ENV to production for build stage
ENV NODE_ENV=production
# Copy the environment file for production
ARG ENV_FILE=.env.production
COPY frontend/${ENV_FILE} .env
# Copy the full frontend directory
COPY frontend .
~~~

[Link a docu Next.js](https://nextjs.org/docs/pages/guides/environment-variables)


## HUE-232 Logging Implementation Plan

### Objective

Implement a persistent logging strategy for the backend and frontend applications to prevent log loss during container redeployments, utilizing the host machine's `syslog` daemon.

### Current Status & Problem

Currently, Docker containers are using the default `json-file` logging driver. This means logs are tied to the container's lifecycle and are lost whenever a container is stopped, removed, or redeployed. This makes historical log analysis impossible.

### Proposed Solution: Local Persistent Logging (`syslog`)

We will configure Docker to send container logs to the host machine's `syslog` daemon. This decouples logs from the container lifecycle, ensuring persistence. We will also configure `rsyslog` and `logrotate` on the host to manage these logs effectively.

#### Strategy Explained: `syslog`, `rsyslog`, and `logrotate`

This logging strategy relies on three standard Linux components working together to provide a robust and persistent logging system.

*   **`syslog`**: This is not a program, but a standard protocol for forwarding log messages in an IP network. When we set Docker's log driver to `syslog`, we are telling the Docker daemon to send container logs as `syslog` messages to the host operating system, instead of managing them itself.

*   **`rsyslog`**: This is the powerful log processing service running on the host machine that receives these `syslog` messages. We configure `rsyslog` with rules to filter the incoming logs. In our case, it looks for the unique `tag` associated with each container (`seo-prod-back` or `seo-prod-front`) and directs those logs into separate files (`/var/log/docker/seo-prod-back.log` and `/var/log/docker/seo-prod-front.log`). This separates our application logs from other system logs.

*   **`logrotate`**: This is a utility designed to manage the size and lifecycle of log files. As applications generate logs, the files can grow indefinitely. `logrotate` runs on a schedule (e.g., daily) to archive the current log file and create a new empty one. It also handles compressing old logs and deleting the oldest ones to prevent them from filling up the disk.

##### Understanding the `logrotate` Configuration

The configuration for `logrotate` tells it how to manage our specific Docker log files.

```
/var/log/docker/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 syslog adm
}
```

*   `daily`: Rotate logs once a day.
*   `rotate 7`: Keep the 7 most recent log files. Older files are deleted.
*   `compress`: Compress rotated log files (e.g., with `gzip`) to save space.
*   `delaycompress`: Postpones compression of the most recently rotated log file. The file from yesterday remains uncompressed, but older ones are compressed. This ensures that any process that might still have the old file open can finish writing to it.
*   `missingok`: If a log file doesn't exist, don't report an error.
*   `notifempty`: Don't rotate a log file if it's empty.
*   `create 0640 syslog adm`: When creating a new log file, set its permissions to `0640` (read/write for owner, read for group), set the owner to the `syslog` user, and the group to the `adm` group.

This combination ensures that logs are captured, organized, and maintained efficiently without manual intervention.

### Implementation Steps

The implementation involves three main parts:

1.  Modifying the Jenkins shared library (`pipelineDeployToServerBindMount.groovy`).
2.  Modifying the application-specific Jenkinsfiles (`Jenkinsfile-Prod`).
3.  Configuring the host machine (`rsyslog` and `logrotate`).

---

#### Part 1: Modify Jenkins Shared Library (`pipelineDeployToServerBindMount.groovy`)

The `pipelineDeployToServerBindMount` shared library function needs to be updated to accept and apply logging configuration to the `docker run` command it executes.

**File to Modify:** `pipelineDeployToServerBindMount.groovy` (in your Jenkins shared library repository)

**1. Add Logging Parameters to the `environment` block:**

Add these lines inside the `environment` block to make the new parameters available:

```groovy
// Inside the environment block
environment {
    // ... existing variables ...
    DOCKER_LOG_DRIVER = "${pipelineParams.DOCKER_LOG_DRIVER}" // OPCIONAL
    DOCKER_LOG_OPTS = "${pipelineParams.DOCKER_LOG_OPTS}"     // OPCIONAL
}
```

**2. Build the Logging Flags String:**

Add a `script` block within the `Deploy` stage to dynamically construct the `docker run` logging flags based on the provided parameters. This should be placed *before* the `sshagent` step.

```groovy
// Inside the Deploy stage
stage('Deploy') {
    steps {
        script {
            def loggingFlags = ""
            if (env.DOCKER_LOG_DRIVER && env.DOCKER_LOG_DRIVER != "null") { // Check for null string from pipelineParams
                loggingFlags += "--log-driver ${env.DOCKER_LOG_DRIVER}"
                if (env.DOCKER_LOG_OPTS && env.DOCKER_LOG_OPTS != "null") { // Check for null string
                    // Split by comma to support multiple options
                    def opts = env.DOCKER_LOG_OPTS.split(',')
                    opts.each { opt ->
                        loggingFlags += " --log-opt ${opt.trim()}"
                    }
                }
            }
            // Make the variable available to the next step
            env.LOGGING_FLAGS = loggingFlags
        }
        sshagent(credentials: ["${SERVER_CREDENTIALS}"], ignoreMissing: true) {
            // ... rest of the sshagent block
        }
    }
}
```

**3. Update the `docker run` Command:**

Modify the `docker run` command within the `sshagent` block to include the newly constructed `LOGGING_FLAGS`.

**Original `docker run` command:**
```groovy
sh 'ssh -o "StrictHostKeyChecking no" ${USER_NAME}@${TARGET_SERVER} sudo docker run -p ${APP_PORT}:${DOCKER_PORT} -v /home/${USER_NAME}/${APP_NAME}_data:/app/data -d --name ${APP_NAME} --restart unless-stopped $DOCKER_REGISTRY/${APP_NAME}:latest'
```

**Modified `docker run` command:**
```groovy
sh 'ssh -o "StrictHostKeyChecking no" ${USER_NAME}@${TARGET_SERVER} sudo docker run -p ${APP_PORT}:${DOCKER_PORT} -v /home/${USER_NAME}/${APP_NAME}_data:/app/data ${env.LOGGING_FLAGS} -d --name ${APP_NAME} --restart unless-stopped $DOCKER_REGISTRY/${APP_NAME}:latest'
```

---

#### Part 2: Modify Application Jenkinsfiles (`Jenkinsfile-Prod`)

Once the shared library is updated, you can configure your application-specific Jenkinsfiles to use the new logging parameters.

**File to Modify:** `backend/DevOps/Jenkinsfile-Prod` (and the equivalent for the frontend)

**Action:** Add `DOCKER_LOG_DRIVER` and `DOCKER_LOG_OPTS` to the `pipelineDeployToServerBindMount` call.

```groovy
@Library('Shared-Library@main') _

pipelineDeployToServerBindMount {
    SERVER_CREDENTIALS = "Produccion-Huenei"
    USER_NAME = "root"
    TARGET_SERVER = "192.168.0.101"
    DOCKERFILE_PATH = "backend/DevOps/Dockerfile-Prod"
    APP_NAME = "seguimiento-objetivos-seo-prod-back"
    APP_PORT = "8061"
    DOCKER_PORT = "4000"
    // --- Add these two lines for logging ---
    DOCKER_LOG_DRIVER = "syslog"
    DOCKER_LOG_OPTS = "tag=seo-prod-back" // Unique tag for backend logs
}
```

**For the Frontend Jenkinsfile:**
Apply the same change, but use a unique tag:
```groovy
    DOCKER_LOG_DRIVER = "syslog"
    DOCKER_LOG_OPTS = "tag=seo-prod-front" // Unique tag for frontend logs
```

---

#### Part 3: Configure Host Machine (`192.169.0.101`)

The host machine needs to be configured to properly receive and manage the logs sent by Docker via `syslog`.

**1. Configure `rsyslog` to Separate Docker Logs:**

This will direct logs from our applications to dedicated files.

*   **SSH into the host machine:**
    ```bash
    ssh root@192.169.0.101
    ```
*   **Create a new rsyslog configuration file:**
    ```bash
    sudo nano /etc/rsyslog.d/60-docker-apps.conf
    ```
*   **Add the following content:**

    ```
    # /etc/rsyslog.d/60-docker-apps.conf

    # Create a template to format logs without the syslog header
    $template DockerLogs, "%msg%\n"

    # Filter messages for the backend app and write to its own file
    if $programname == 'seo-prod-back' then {
        /var/log/docker/seo-prod-back.log;DockerLogs
        & stop
    }

    # Filter messages for the frontend app and write to its own file
    if $programname == 'seo-prod-front' then {
        /var/log/docker/seo-prod-front.log;DockerLogs
        & stop
    }
    ```
*   **Restart the `rsyslog` service:**
    ```bash
    sudo systemctl restart rsyslog
    ```

**2. Configure `logrotate` for the New Log Files:**

This will manage the size and retention of the new log files.

*   **Create a new logrotate configuration file:**
    ```bash
    sudo nano /etc/logrotate.d/docker-apps
    ```
*   **Add the following content:**

    ```
    /var/log/docker/*.log {
        daily
        rotate 7
        compress
        delaycompress
        missingok
        notifempty
        create 0640 syslog adm
    }
    ```

---

### Verification

1.  After applying all changes, run your Jenkins pipelines to deploy both the backend and frontend applications.
2.  SSH into the host machine (`192.169.0.101`).
3.  Check for the new log files:
    ```bash
    ls -l /var/log/docker/
    ```
    You should see `seo-prod-back.log` and `seo-prod-front.log`.
4.  Monitor the logs in real-time to confirm messages are being written:
    ```bash
    tail -f /var/log/docker/seo-prod-back.log
    ```
    And similarly for the frontend log.

This plan ensures that your application logs are persistently stored and properly managed on the host machine.



***

# NaviJet

## Datos lado cliente
**Azure DevOps**: http://isauy-devops.isa-agents.com.ar/Navijet - 192.168.10.81

Usuario: huenei@isa-agents.com.ar           
Password: Pu7xDS184-

PAT: [REDACTED]

**NaviOper Servers**:

PROD:
Usuario: huenei - azureuser
Password: Cls86jCLs__?395j
10.32.0.5 Build
10.32.0.21 Front
10.32.0.37 Back
10.32.0.53 Data

QA: 
Usuario: huenei - azureuser
Password: Pls85jskf_?3289JNE*
10.32.0.20 Front
10.32.0.36 Back
10.32.0.52 Data

**pgAdmin**
http://10.32.0.52/pgadmin4
huenei@isa-agents.com.ar
Pu7xDS184-
http://10.32.0.53/pgadmin4
huenei@isa-agents.com.ar
Pu7xDS184-
BD: NAVIOPER_QA y NAVIOPER	postgres	cA25T=ef(%XkUw_j
BD: NAVIOPER_QA y NAVIOPER	backup_user	57uMUabDSMR=r{Gb

**Azure Container Registry**:
Repositorio: AcrNaviOper
Password: [REDACTED]


# NaviTango

Es solo Back. DB postgres.

Paso importante para establecer conexión a nuevos servicios a la base de datos:
Editar el archivo 'pg_hba.conf' en el servidor de DB correspondiente al entorno.
~~~bash
/etc/postgresql/<version>/main/pg_hba.conf
~~~
Agregar una línea debajo de todo similar a esto:
~~~bash
hostssl NAVITANGO_INT   navitango_user  10.32.0.32/28           scram-sha-256
~~~
Cargar la config y reiniciar postgres.
~~~bash
sudo systemctl reload postgresql
sudo systemctl restart postgresql
~~~

### INT

Deploy en el server de QA.

http://10.32.0.36:8087

### QA

http://10.32.0.36:8088

#### HUE-127

Deploy a QA estuvo con errores de conexión a base de datos.
Logs:
~~~bash
***************************

APPLICATION FAILED TO START

***************************



Description:



Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.



Reason: Failed to determine suitable jdbc url
~~~

Se realizaron pruebas para verificar la conexión a la db desde el host.

~~~bash
# ping desde el deploy host al db host
huenei@NaviOper-Back-QA:~$ ping -c 3 10.32.0.52

PING 10.32.0.52 (10.32.0.52) 56(84) bytes of data.

64 bytes from 10.32.0.52: icmp_seq=1 ttl=64 time=0.803 ms

64 bytes from 10.32.0.52: icmp_seq=2 ttl=64 time=1.00 ms

64 bytes from 10.32.0.52: icmp_seq=3 ttl=64 time=0.983 ms
--- 10.32.0.52 ping statistics ---

3 packets transmitted, 3 received, 0% packet loss, time 2026ms

rtt min/avg/max/mdev = 0.803/0.929/1.002/0.089 ms
~~~
~~~bash
# conexión a la db con las mismas credenciales

huenei@NaviOper-Back-QA:~$ psql -h 10.32.0.52 -p 5432 -U navitango_user -d NAVITANGO_QA

Password for user navitango_user:

psql (14.18 (Ubuntu 14.18-0ubuntu0.22.04.1), server 16.0 (Ubuntu 16.0-1.pgdg22.04+1))

WARNING: psql major version 14, server major version 16.

         Some psql features might not work.

SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)

Type "help" for help.



NAVITANGO_QA=> exit
~~~
curl a ip y puerto de db
~~~bash
huenei@NaviOper-Back-QA:~$ curl 10.32.0.52:5432

curl: (52) Empty reply from server
~~~

No se pudo hacer las pruebas desde el contenedor ya que esta en un restart loop. Se pudo verificar que sí hay conexión desde el servidor de QA. Se implementó una linea de debugging en el pipeline para verificar el contenido del application.properties.
azure-pipelines.yml
~~~bash
# Luego de la lina de Set Config Files
      - script: |
            cat ./src/main/resources/application.properties
        displayName: Verify application.properties content  
~~~

Logs del pipeline:
~~~bash
# database properties
spring.datasource.url=jdbc:postgresql://10.32.0.52:5432/NAVITANGO_QA
spring.datasource.username=navitango_user
spring.datasource.password=navitango3030
spring.jpa.properties.hibernate.jdbc.lob.non_contextual_creation=true
.
.
.
~~~

Estaba copiando bien los datos para el .properties.

Con ayuda de Gemini, se dedujo que el error está en los perfiles de Spring.

The Remaining Suspect: Spring Profiles

Given all the above, the most likely culprit is still how Spring Boot is consuming the application.properties within the Docker container, specifically related to the active profiles.

Recall your Dockerfile's ENTRYPOINT:
~~~Dockerfile
ENTRYPOINT [ "sh", "-c", "java -Dspring.profiles.active=docker -Djava.security.egd=file:/dev/./urandom -jar /usr/local/tomcat/webapps/app.jar" ]
~~~
This line explicitly tells Spring Boot to activate the docker profile.

Here's the problem:

Your application.properties file (which contains your database URL for either INT or QA) does not define properties under the docker profile. It defines them as generic, default properties.

When spring.profiles.active=docker is set, Spring Boot will primarily look for application-docker.properties and then fall back to application.properties for non-profile-specific settings.

However, if application.properties contains properties that are also expected to be overridden or only available in a specific profile (like database URL), and that profile isn't active, it can lead to issues like "Failed to determine suitable jdbc url." This happens if the general application.properties is overridden or ignored in favor of an expected profile-specific file that doesn't exist or is empty for critical properties.

Se cambió el pipeline para que plasme los datos a 'application-docker.properties' en vez de 'application.properties'

~~~yml
      - script: |
            mv $(application.secureFilePath) ./src/main/resources/application-docker.properties
~~~
### Prod

http://10.32.0.37:8089

## HUE-134
tags: navi, sonarqube, sast

Modificación a la opción de sonar.exclusions en la ejecución del sonar-scanner.
Investigando la opción de sonar.sources para que tome solo el directorio app y descartar las exclusions innecesarias.

~~~bash
-Dsonar.exclusions=**/node_modules/**,**/coverage/**,**/*.test.ts,**/*.test.tsx,**/*.spec.ts,**/*.spec.tsx,**/public/**,**/DevOps/**,**/Tests/** \
~~~

Se dejó la opción de sonar.exclusions.
Se investigó modificar el COPY en el Dockerfile en la etapa de scan e indicar la ruta en sonar.sources

~~~Dockerfile
FROM openjdk:17-jdk-slim AS scan
WORKDIR /app
COPY --from=build . .
~~~
~~~Dockerfile
FROM openjdk:17-jdk-slim AS scan
WORKDIR /app
COPY --from=build /app/app ./app
~~~
Sin embargo, después de hablar con Jorge, esto implicaría también un cambio en la estructura de directorios para el dependency-check. Por eso se optó a mantener el sonar.exclusions

# NaviPdfProcess

## HUE-167 Dockerfile Optimization for NaviPdfProcess Service
1. Overview
This document outlines the optimization techniques applied to the Dockerfile for the NaviPdfProcess microservice. The goal was to reduce the final image size, decrease build times in the CI/CD pipeline, and improve the overall security and efficiency of the container, based on modern containerization best practices.

2. The Naive Approach (Before Optimization)
A straightforward, unoptimized Dockerfile would typically involve a single stage, resulting in a very large and inefficient image.

Example of an unoptimized Dockerfile:
~~~Dockerfile
# Unoptimized single-stage build
FROM maven:3.9.6-eclipse-temurin-21

WORKDIR /app
COPY . .

# Build the application
RUN mvn clean package -DskipTests

# Install runtime dependencies
RUN apt-get update && apt-get install -y libreoffice-nogui

EXPOSE 8086
CMD ["java", "-jar", "target/ms-navi-tango-0.0.1-SNAPSHOT.jar"]
~~~
Drawbacks of this approach:

Massive Image Size: The final image contains the full Maven build environment, the JDK, all source code, and intermediate build artifacts, leading to an image size of over 1.5 GB.

Slow Builds: Every code change would require re-downloading all Maven dependencies.

Large Attack Surface: Including build tools like Maven and the full JDK in a production image introduces unnecessary security vulnerabilities.

3. The Optimized Approach (Final Version)
The final Dockerfile leverages several key techniques to produce a lean, fast, and more secure image.

Optimized Dockerfile:
~~~Dockerfile
# =========================================================================
# BUILD STAGE
# =========================================================================
# Enable BuildKit features like cache mounts.
# syntax=docker/dockerfile:1.4

# Use a Maven image with JDK 21 for the build environment.
FROM maven:3.9.6-eclipse-temurin-21 AS build

# Set the working directory inside the container.
WORKDIR /app

# Copy the entire project context.
COPY . .

# Use a BuildKit cache mount to persist the Maven repository on the host.
# This makes subsequent builds much faster by avoiding re-downloading dependencies.
# Then, build the application and skip running tests.
RUN --mount=type=cache,target=/root/.m2 mvn clean package -DskipTests


# =========================================================================
# RUNTIME STAGE
# =========================================================================
# Use a minimal JRE 21 image for the runtime environment.
FROM eclipse-temurin:21-jre-jammy

# Install LibreOffice for document processing.
# Cleaning up in the same RUN command reduces the final image size.
RUN apt-get update && \
    apt-get install -y --no-install-recommends libreoffice-nogui && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Set the working directory for the application.
WORKDIR /app

# Copy the compiled JAR from the build stage.
COPY --from=build /app/target/*.jar ./app.jar

# Expose the application port.
EXPOSE 8086

# Set the command to run the application when the container starts.
ENTRYPOINT [ "java", "-Dspring.profiles.active=docker", "-Djava.security.egd=file:/dev./urandom", "-jar", "app.jar" ]
~~~
4. Key Optimization Techniques Explained
a. Multi-Stage Builds
The AS build syntax defines a separate "build stage." This allows us to use a full JDK and Maven environment to compile the code, and then copy only the resulting .jar file into a clean, minimal runtime image. The build environment and all its tools are completely discarded, drastically reducing the final image size.

b. Minimal Base Image (eclipse-temurin:21-jre-jammy)
Instead of using a full JDK image for the final stage, we use a jre-slim (or in this case, jre-jammy) image. This image contains only the Java Runtime Environment (JRE) needed to run the application, not the compiler and other development tools, further reducing size and attack surface.

c. Efficient Layering
The command to install LibreOffice (apt-get install ...) is combined with the cleanup command (apt-get clean ...) in a single RUN instruction. This ensures that the downloaded package lists and cache are removed within the same layer they were created in, preventing them from bloating the final image.

d. BuildKit Cache Mounts (--mount=type=cache)
This is the most significant optimization for CI/CD build speed. Instead of relying on Docker's layer cache (which breaks if pom.xml changes), we mount a persistent cache directory from the build agent's host machine. Maven uses this cache to store downloaded dependencies, so it only ever downloads new or updated ones, just like on a local developer machine. This makes subsequent builds incredibly fast.

5. Sources

[A Step-by-Step Guide to Docker Image Optimisation: Reduce Size by Over 95%](https://freedium.cfd/https://blog.prateekjain.dev/a-step-by-step-guide-to-docker-image-optimisation-reduce-size-by-over-95-d90bcab3819d)

[Maven build in docker taking too much time?](https://medium.com/@bhaktadip/maven-build-in-docker-taking-too-much-time-47ebcdc7de53)
# GameStrong

## Propuesta técnica

Proyecto nuevo, Adrian Barreiro, Martin Santamaria

![propuesta diagrama](./notas/GameStrongDiagrama.png)

## Entorno Test Azure Huenei

Trabajar dentro de la subscripción DEVOPS, crear el resource group GameStrong

### Flujo Fase 1
![fase 1](./notas/GameStrongDiagramaFase1.png)

Checklist

<input checked="" type="checkbox"> Recursos creados con nombres claros y consistentes
<input checked="" type="checkbox"> Pipeline configurado para build/push/deploy
<input type="checkbox"> Container App con Managed Identity
<input checked="" type="checkbox"> Key Vault con secretos y permisos
<input checked="" type="checkbox"> Variables de entorno referenciando Key Vault
<input checked="" type="checkbox"> Prueba punta a punta


# SwissJust

## HUE-157 Configuración HTTPS

This guide details the end-to-end process for hosting a static website on **Amazon S3**, securing it with HTTPS using **AWS Certificate Manager (ACM)** and **Amazon CloudFront**, and pointing a custom domain to it using an **Azure DNS Zone**.

---

### **Step 1: Configure S3 Bucket for Static Website Hosting**

First, we set up the S3 bucket to serve the website content.

1.  **Create the S3 Bucket**
    * Navigate to the **Amazon S3** console.
    * Create a new bucket (e.g., `your-domain-com`). The name must be globally unique.
    * During setup, uncheck **Block all public access** and acknowledge that the bucket will be made public.

2.  **Enable Static Website Hosting**
    * Select the newly created bucket and go to the **Properties** tab.
    * Scroll down to **Static website hosting** and click **Edit**.
    * Select **Enable**, choose **Host a static website**, and enter your index document (e.g., `index.html`).
    * Save the changes and **copy the Bucket website endpoint URL**. It will look like this: `http://your-bucket-name.s3-website.your-region.amazonaws.com`.

3.  **Set Bucket Policy for Public Access**
    * Go to the **Permissions** tab and click **Edit** under **Bucket policy**.
    * Paste the following JSON, replacing `your-bucket-name` with your actual bucket name. This allows public read access to your website files.
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "PublicReadGetObject",
          "Effect": "Allow",
          "Principal": "*",
          "Action": "s3:GetObject",
          "Resource": "arn:aws:s3:::your-bucket-name/*"
        }
      ]
    }
    ```

4.  **Upload Website Files**
    * Go to the **Objects** tab and upload your `index.html` and other static files (CSS, JS, images, etc.).

---

### **Step 2: Provision a Public SSL/TLS Certificate (ACM)**

Next, we create a free SSL certificate to enable HTTPS.

1.  **Request a Certificate**
    * Navigate to **AWS Certificate Manager (ACM)**.
    * **Crucially**, ensure you are in the **US East (N. Virginia) `us-east-1`** region in the top-right console corner. This is mandatory for CloudFront.
    * Click **Request a certificate** and choose **Request a public certificate**.

2.  **Add Domain Names**
    * Enter your fully qualified domain names. It is best practice to add both the root domain and the `www` subdomain.
        * `your-domain.com`
        * `www.your-domain.com`

3.  **Validate Domain Ownership**
    * Select **DNS validation** as the validation method and request the certificate.
    * ACM will provide **CNAME Name** and **CNAME Value** pairs needed to prove you own the domain. Keep these values ready for the Azure DNS step.

---

### **Step 3: Create and Configure the CloudFront Distribution**

CloudFront acts as the secure, fast entry point (CDN) to your website.

1.  **Create a Distribution**
    * Navigate to the **Amazon CloudFront** console and click **Create a distribution**.

2.  **Configure the Origin**
    * For the **Origin domain**, **paste the S3 bucket website endpoint URL** you copied in Step 1.
    * **Important**: Do **not** select the S3 bucket from the dropdown menu. This is the most common point of failure and leads to 504 Gateway Timeout errors.

3.  **Set Viewer Protocol Policy**
    * Under the **Viewer** section, set the **Viewer protocol policy** to **Redirect HTTP to HTTPS**.

4.  **Configure Settings (CNAMEs, Certificate, and Root Object)**
    * **Alternate domain names (CNAMEs)**: Add your custom domain names (e.g., `www.your-domain.com` and `your-domain.com`).
    * **Custom SSL certificate**: Select the certificate you created in ACM from the dropdown list.
    * **Default root object**: Enter `index.html`.

5.  **Deploy and Get Domain Name**
    * Click **Create distribution**. Deployment can take 5-15 minutes.
    * Once the status is "Enabled", copy the **Distribution domain name** (e.g., `d1234abcd.cloudfront.net`).

---

### **Step 4: Configure DNS Records in Azure**

Finally, we point the custom domain to the CloudFront distribution.

1.  **Log into the Azure Portal** and navigate to your **DNS Zone**.

2.  **Add the ACM Validation Record**
    * Click **+ Record set**.
    * Create a **CNAME** record using the **Name** and **Value** provided by ACM in Step 2. This proves domain ownership to AWS and is only needed until the certificate status becomes "Issued".

3.  **Add the Main CNAME Record for Live Traffic**
    * Click **+ Record set** again.
    * **Name**: Enter the subdomain part (e.g., `www`).
    * **Type**: Select **CNAME**.
    * **Alias**: Paste the **Distribution domain name** you copied from CloudFront (e.g., `d1234abcd.cloudfront.net`).
    * **TTL**: A value of `1 Hour` is standard.
    * Click **OK** to save the record.

After these steps, your setup is complete. Once DNS has fully propagated, your website will be fully functional and accessible globally at `https://www.your-domain.com`.

### **Step 5: Agregar registros a DNS interno Huenei**

Puede que en las oficinas no tome el cambio inmediatamente. Consultar con Lucas Llarull si se puede actualizar la cache del servidor DNS.

## HUE-173 Blank page & 404 error

La pantalla en blanco y error 404 not found se solucionó configurando el manejo de errores de CloudFront en AWS.
En la consola aws:

### Pasos

1. Ir a CloudFront y elegir la distribución. En este caso para autoincorporacion-test d380tziq2ql3ms.cloudfront.net

2. Error Pages: Create custom error response

3. Configuración:

    a. HTTP error code: Select 404: Not Found.

    b. Customize error response: Select Yes.

    c. Response page path: Enter /index.html

    d. HTTP Response code: Select 200: OK.

4. Esperar 5 minutos para que se propague el cambio.

5. Si es urgente:

    a. Invalidations tab

    b. Create invalidation

    c. Object Paths, poner /*

    d. Click Create invalidation


## HUE-174 CORS

### Detalle ticket

En autoincorporacion test:

Se intentó solucionar agregando CORS policy al bucket de s3, no funcionó.

Se analizó el error log.

Error log de CORS en el navegador:

Access to XMLHttpRequest at 'https://incorporarme-test.swissjust.com/api//consultants/public-profile/' from origin 'https://swissjust-auto-incorporacion-test.huenei.com.ar ' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.

Se pidió a Sergio Ruiz si el equipo de back (o quien maneje incorporarme-test.swissjust.com) puede agregar el header Access-Control-Allow-Origin.

En las otras 3 URLs no se encontró CORS error. Por lo menos en el landing page.

## HUE-162

### **Ticket Summary: Automating CloudFront Cache Invalidation in CI/CD Pipeline**

**1. Problem Description:**

Following successful deployments to the S3 bucket via the Jenkins pipeline, the updated changes were not visible on the public HTTPS URL. The CloudFront distribution continued to serve an old, cached version of the website content. This required a manual cache invalidation to be performed in the AWS console after every deployment, which was inefficient and error-prone.

**2. Solution Implemented:**

The CloudFront cache invalidation process was fully automated by integrating it into the existing Jenkinsfile. A new step was added to the pipeline's `post` section that triggers automatically only after a deployment is successful. This step uses the AWS CLI to create a cache invalidation for all objects (`/*`) in the CloudFront distribution, ensuring that new changes are reflected for all users immediately after a release.

**3. Implementation Steps:**

The automation was achieved with the following steps:

1.  **Verify IAM Permissions:** Confirmed that the IAM user (`DevOps`) associated with the Jenkins credentials (`SwissJust-Front`) had `AdministratorAccess`, which includes the necessary `cloudfront:CreateInvalidation` permission. No permission changes were required.

2.  **Add Environment Variable:** The CloudFront Distribution ID (`E1TVNUA1UHK3FO`) was added as an environment variable (`CLOUDFRONT_DISTRIBUTION_ID`) at the top of the Jenkinsfile for easy management.

3.  **Update the Jenkinsfile:** The pipeline script was modified to include a `post { success { ... } }` block. This ensures the invalidation command runs only when the build and deploy stages are completed without errors.

4.  **Add Invalidation Command:** The AWS CLI command to create the invalidation was added within the `success` block. The final, working code snippet added to the `post` section is as follows:

    ```groovy
    post {
        success {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'SwissJust-Front', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                sh 'echo "Invalidating CloudFront cache"'
                sh "aws cloudfront create-invalidation --distribution-id ${env.CLOUDFRONT_DISTRIBUTION_ID} --paths '/*'"
            }
        }
        always {
            // ... existing script and cleanWs steps
        }
    }
    ```
## HUE-185

### **Ticket Summary: Continuing HUE-162 automating rest of services**

List of services and Distribution ID:

1. swissjust-oficina-virtual-test.huenei.com.ar		
E2JPBOZ14CWIYP	

2. swissjust-oficina-virtual-pedidos.huenei.com.ar		PEDIDOS AND TEST ARE THE SAME?
E2NXAFD4EB0QQ4

3. swissjust-oficina-virtual-pre.huenei.com.ar		
E1I4U9M5S8C8P8

4. swissjust-pedidos-test.huenei.com.ar			
E1168W1NN5ENPE

5. swissjust-oficina-virtual-int.huenei.com.ar		
E1SBRYRZ1SLGDV

# Sitio web Huenei

Datos del sitio web Huenei comercial.

Diseño -> Dani
Host -> Lucas
Code -> Tomás (externo)

credenciales:
Administrator
3)^Usi@pdNFZXhghYMQzva3R