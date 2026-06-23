# Introduction

Este repositorio sirve para el servicio de Keycloak en el servidor de Producción Huenei.

- Hostname: dockerprod01
- IP: 192.169.0.102

En el servidor viejo de prod, el despliegue se hacía con la ejecución manual de 'docker compose' en el servidor.
La idea es guardar el compose.yml en este repo y crear el pipeline de ci/cd para adoptar un flujo moderno y menos vulnerable a errores humanos.

## DevSecOps Guardrails

Este proyecto utiliza `pre-commit` para garantizar la calidad del código y la seguridad (escaneo de secretos).

### Instalación (usando uv)

1. Instala `pre-commit` como herramienta global:

   ```bash
   uv tool install pre-commit
   ```

2. Instala los hooks de git:

   ```bash
   pre-commit install
   ```

### Ejecución Manual

Para ejecutar las validaciones manualmente en todos los archivos:

```bash
pre-commit run --all-files
```
