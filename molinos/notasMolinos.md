# Notas Molinos

Este documento sirve como notas y contexto sobre el proyecto Molinos en Huenei.

## PPVA-587 // MMS-1769

### Automatización de Migraciones de Base de Datos (EF Core)

**Objetivo:** Integrar la ejecución de migraciones de Entity Framework Core en el pipeline de CI/CD de Azure DevOps (Legacy GUI).

**Estrategia:** Se utiliza "Migration Bundles" para generar un ejecutable autocontenido (`migrate.exe`). Esta estrategia permite:

1. **Seguridad:** Manejar el Connection String como variable secreta en el Release.
2. **Portabilidad:** Ejecutar las migraciones sin requerir el SDK de .NET completo en el servidor de destino.
3. **Desacoplamiento:** Separar la creación del paquete de migración (Build) de su ejecución (Release).

**Configuración del Build Pipeline (QA):**

1. **Create Tool Manifest:** `dotnet new tool-manifest --force`
2. **Install dotnet-ef (Local):** `dotnet tool install dotnet-ef --version 7.0.0` (Instalación local para evitar conflictos de versiones en agentes privados).
3. **Copy Workaround Files:** Copiar `nlog.config`, `appsettings.json` y `Repositories/Scripts/**` a `$(Build.ArtifactStagingDirectory)/DB-Migrations`.
   - *Nota:* Estos archivos son necesarios porque el proceso de "discovery" de EF Core intenta inicializar el contexto de la aplicación.
4. **Create Migration Bundle:**

   ```bash
   dotnet tool run dotnet-ef migrations bundle --project Entities/Entities.csproj --startup-project Api/Api.csproj --self-contained -r win-x64 --output $(Build.ArtifactStagingDirectory)/DB-Migrations/migrate.exe
   ```

5. **Publish Artifact:** Publicar la carpeta `DB-Migrations`.

**Configuración del Release Pipeline:**

1. **Variable Secreta:** `Database.ConnectionString` (String desencriptado).
2. **Task (Command Line):** Ejecutar el bundle.
   - **Script:** `$(System.DefaultWorkingDirectory)/_Alias/DB-Migrations/migrate.exe --connection "$(Database.ConnectionString)"`
   - **Working Directory:** `$(System.DefaultWorkingDirectory)/_Alias/DB-Migrations` (Indispensable para que el `.exe` localice los scripts SQL y archivos de config).

**Consideraciones Técnicas y Lecciones Aprendidas:**

- **Error 10060 (Timeout):** Indica bloqueo de firewall. El Agente de Azure DevOps requiere acceso al SQL Server por el puerto 1433.
- **Error 3762504530 (FileLoadException):** Ocurre cuando la versión de `dotnet-ef` es más nueva que el SDK instalado (ej. Herramienta v8+ en SDK v6). Se solucionó usando herramientas locales v7.0.0.
- **Contexto de Diseño:** Se recomendó a los desarrolladores implementar `IDesignTimeDbContextFactory` para eliminar la dependencia de `appsettings.json` y `nlog.config` durante las migraciones.
- **Embedded Resources:** Se recomendó incluir los archivos `.sql` como recursos embebidos en el `.csproj` para evitar errores de "Ruta no encontrada" y eliminar la necesidad de copiar carpetas de scripts manualmente.
