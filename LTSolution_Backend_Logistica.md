# Arquitectura Backend .NET para LogisticaDB (Completa)

## 1. Objetivo y Alcance
- Propósito: construir un backend .NET robusto para operar sobre `LogisticaDB`, soportando gestión de flota, viajes, cotizaciones, facturación/pagos, RRHH, documentos y alertas.
- Alcance: API HTTP, acceso a datos SQL Server, jobs en background, seguridad, cache, observabilidad, y preparación para escalar por módulos.

## 2. Arquitectura Propuesta
- Estilo: Modular Monolith + Clean Architecture + Vertical Slices.
- Capas:
  - `Domain`: entidades, reglas y conceptos del negocio (sin dependencias externas).
  - `Application`: casos de uso (CQRS), validaciones, DTOs, mapeos, puertos/repo interfaces.
  - `Infrastructure`: EF Core (Database-First), Dapper, proveedores externos (Redis, Hangfire, Serilog).
  - `API`: ASP.NET Core Minimal APIs por vertical slice (feature-first), seguridad, filtros y OpenAPI.
- Vertical Slices (features por contexto):
  - Comercial: `Clientes`, `Cotizaciones`, `CotizacionDetalles`.
  - Operación: `Viajes`, `DetallesCargaViaje`, `ParadasDescanso`, `GuiasRemision`.
  - Flota: `Flota`, `MantenimientoReparaciones`, `DocumentosVehiculo`, `ProveedoresTalleres`.
  - Facturación: `Facturas`, `FacturaDetalles`, `Pagos`.
  - RRHH: `Empleados`, `Usuarios`, `DocumentosEmpleado`, `AsistenciasPermisos`.
  - Catálogos/Geo: `TiposMaestros`, `Ubigeo`, `Configuraciones`.
- Principios clave:
  - Aislar la lógica por slice para mantener bajo acoplamiento.
  - Mantener DTOs y validaciones en `Application`; nunca exponer entidades EF directamente desde la API.
  - Usar EF Core para escritura transaccional y Dapper para lecturas/reportes intensivos.

## 3. Tecnologías y Librerías
- Runtime/API
  - `.NET 8` + `ASP.NET Core` (Minimal APIs) + `Swashbuckle.AspNetCore` (OpenAPI/Swagger).
- Datos
  - `Microsoft.EntityFrameworkCore.SqlServer (8.x)` (Database-First) + `dotnet-ef`.
  - `Dapper (2.x)` para consultas de alto rendimiento.
- Validación y mapeo
  - `FluentValidation (11.x)` para DTOs.
  - `AutoMapper (12.x)` opcional; recomendación: mapeo manual por slice para claridad y rendimiento.
- CQRS/mediación (opcional)
  - `MediatR (12.x)` si se desea handlers explícitos por comando/consulta.
- Seguridad
  - `ASP.NET Core Identity` + JWT con `Microsoft.AspNetCore.Authentication.JwtBearer`.
  - `BCrypt/Argon2` opcional si se personaliza el hashing; Identity usa PBKDF2 por defecto.
- Observabilidad
  - `Serilog.AspNetCore (7.x)`, `Serilog.Sinks.File (5.x)`, `Serilog.Sinks.Seq (5.x)`.
  - `OpenTelemetry.Extensions.Hosting (1.x)` opcional para trazas/métricas.
  - Health Checks: `Microsoft.Extensions.Diagnostics.HealthChecks`; opcional `AspNetCore.Diagnostics.HealthChecks` (SQL Server, Redis).
- Cache
  - `StackExchange.Redis (2.x)` para cache de catálogos (`TiposMaestros`) y resultados frecuentes.
- Jobs en background
  - `Hangfire.Core (1.7.x)` + `Hangfire.AspNetCore (1.7.x)` para recordatorios y notificaciones.
- Versionado API
  - `Microsoft.AspNetCore.Mvc.Versioning (6.x)`.
- Otros
  - `Scrutor (4.x)` para registrar servicios por escaneo de ensamblados (opcional).

## 4. Estructura de Solución (Proyectos)
- `Logistica.Domain`
  - Entidades del dominio, value objects, reglas básicas.
  - Interfaces: `IDateTimeProvider`, `ICurrentUserService`.
- `Logistica.Application`
  - Casos de uso: comandos/consultas por slice.
  - DTOs, validadores (FluentValidation), mapeos.
  - Interfaces de repositorios (puertos) y servicios (catálogo, notificaciones).
- `Logistica.Infrastructure`
  - `LogisticaDbContext` scaffolded desde `LogisticaDB`.
  - Mapeos EF, repositorios concretos, consultas Dapper.
  - Integraciones: Redis, Hangfire, Serilog, Identity, HealthChecks.
- `Logistica.Api`
  - Minimal APIs organizadas por slice (`/features/<slice>`).
  - Configuración JWT/Identity, Swagger, filtros globales (error handling, logging).

## 5. Organización interna por Slice
- Por cada slice (ej. `Viajes`) en `Application` y `Api`:
  - `Commands/`: crear/actualizar/cambiar estado.
  - `Queries/`: listados, detalles, reportes.
  - `Dtos/`: request/response.
  - `Validators/`: reglas de forma y negocio.
  - `Mappings/`: funciones de mapeo entidad↔DTO.
  - `Endpoints/` (en API): definición de rutas Minimal API.

## 6. Acceso a Datos: Database-First + EF Core
- Scaffold inicial (ejecutar en la raíz de la solución):
  - `dotnet tool install --global dotnet-ef`
  - `dotnet ef dbcontext scaffold "Server=localhost;Database=LogisticaDB;Trusted_Connection=True;TrustServerCertificate=True;" Microsoft.EntityFrameworkCore.SqlServer -c LogisticaDbContext -o Logistica.Infrastructure/EF/Models --use-database-names --no-pluralize --schema dbo --data-annotations`
- Prácticas:
  - Usar `AsNoTracking()` para lecturas.
  - Consultas complejas: preferir Dapper (joins grandes, agregaciones).
  - Transacciones: `await using var tx = await db.Database.BeginTransactionAsync();` y confirmar al final del caso de uso.
  - Compiled queries (EF) para rutas calientes.
  - Separación de consultas: `QuerySplittingBehavior.SplitQuery` cuando existan colecciones cargadas.

## 7. Dapper para Reportes
- Uso típico: dashboard de facturación y viajes por periodos.
- Patrón: repositorio de lectura (“ReadRepository”) por slice con SQL parametrizado y DTOs.
- Cuida índice y paginación (`OFFSET ... FETCH ...`).

## 8. Seguridad
- Identidad:
  - `AspNetCore Identity` para gestión de usuarios, roles y contraseñas.
  - JWT Bearer con expiración corta + refresh tokens (tabla `UserTokens` o propia).
- Autorización:
  - Políticas por rol (Admin, Gerente, Comercial, Logistica, Conductor, Mantenimiento).
  - Claims por secciones si se requiere granularidad (p. ej., acceso a flota/operación).
- Almacenamiento:
  - Cadena de conexión y secretos con `User Secrets` (dev) y `Azure Key Vault`/`DotEnv` (prod/staging).
- Endpoints sensibles:
  - Auditar intentos y bloquear por `IntentosLogin`/`FechaBloqueo`.

## 9. Observabilidad y Salud
- Logging: `Serilog` (enrichers: `FromLogContext`, correlación de petición, usuario actual).
- Trazas: OpenTelemetry (opcional) con export a OTLP.
- Health Checks:
  - `/health` (liveness), `/health/ready` (readiness con SQL Server y Redis).
- Alertas: integrar con Seq/ELK.

## 10. Cache (Redis)
- Estrategia:
  - Cache de catálogos `TiposMaestros` por `Seccion`.
  - Cache de `Ubigeo`.
  - TTL y evicción en cambios (invalida en escrituras relevantes).
- Paquetes: `StackExchange.Redis`.

## 11. Jobs en Background (Hangfire)
- Jobs programados:
  - Vencimiento documentos (`DocumentosVehiculo`, `DocumentosEmpleado`).
  - Próximos mantenimientos (`Flota.ProximoMantenimiento`).
  - Facturas vencidas/no pagadas.
- Almacenamiento Hangfire: SQL Server (misma BD o dedicada).
- Dashboard: protegido por rol (Admin/Gerente).

## 12. Versionado y Contratos
- Versionado de APIs: `v1`, `v2` por ruta y/o encabezado.
- OpenAPI/Swagger:
  - Agrupar por slice.
  - Definir esquemas de seguridad (JWT) y ejemplos.
- Compatibilidad:
  - Mantener DTOs estables; cambios breaking solo en nuevas versiones.

## 13. Validación de TiposMaestros por Sección
- Modelo elegido: mantener `TiposMaestros` centralizado.
- Implementación en Aplicación:
  - Servicio `ICatalogoTiposService` con `GetBySeccion(seccion)` (cacheado) y `ValidateTipo(tipoId, seccion)`.
  - Validadores (FluentValidation) que invoquen `ValidateTipo` en DTOs con IDs de tipo.
- Auditorías periódicas (consultas ya documentadas en `LogisticaDB_Documentacion.md`).

## 14. Estructura de Carpetas (sugerida)
- `Logistica.Domain/`
  - `Entities/`, `ValueObjects/`, `Services/`, `Abstractions/`.
- `Logistica.Application/`
  - `Features/<Slice>/Commands|Queries|Dtos|Validators|Mappings/`, `Interfaces/`, `Behaviors/`.
- `Logistica.Infrastructure/`
  - `EF/Models/`, `EF/Configurations/`, `Repositories/`, `Dapper/`, `Services/` (Redis, Catalogo, Email), `Jobs/` (Hangfire), `Logging/`.
- `Logistica.Api/`
  - `Features/<Slice>/Endpoints/`, `Auth/`, `Filters/`, `Swagger/`, `Configuration/`.

## 15. Manejo de Errores y Respuestas
- Filtro global que convierte excepciones en respuestas estándar (`traceId`, `code`, `message`).
- Validación: respuestas 400 con detalles por campo (FluentValidation).
- Dominios: errores de negocio con códigos propios (ej. `FacturaNoPagablePorEstado`).

## 16. Transacciones, Concurrencia y Consistencia
- Transacciones por caso de uso; repositorios coordinados via `DbContext`.
- Concurrencia:
  - Opcional `rowversion` en tablas críticas (Facturas, Viajes, Flota) para evitar overwrites.
- Consistencia eventual:
  - Notificaciones y tareas derivadas vía jobs/mensajería (más adelante MassTransit).

## 17. Rendimiento y Escalabilidad
- Índices: aplicar recomendaciones del documento de BD.
- Dapper: reportes y listados con joins y agregaciones.
- EF Core: `AsNoTracking`, compiled queries, split queries.
- Cache: Redis para catálogos y queries frecuentes.
- Paginación: estándar con `page/size`, `totalCount` y `links`.

## 18. Seguridad Avanzada
- JWT Refresh Tokens con rotación e invalidación.
- Roles/Policies por slice.
- Auditoría de cambios (quién, cuándo) usando `CreadoPor`/`ModificadoPor` y middleware que inyecte el usuario actual.

## 19. DevOps y Despliegue
- Configuración:
  - `appsettings.{Environment}.json`, `User Secrets` (dev), `Key Vault` (prod).
- Docker:
  - Base: `mcr.microsoft.com/dotnet/aspnet:8.0` (runtime), `mcr.microsoft.com/dotnet/sdk:8.0` (build).
  - Multi-stage: `restore` → `build` → `publish` → `final`.
- CI/CD (ejemplo GitHub Actions):
  - `actions/setup-dotnet@v3` → `dotnet restore` → `dotnet build` → `dotnet test` → `dotnet publish` → `docker build/push` → deploy.
- Health/Observabilidad: exponer `/health`, logs a Seq/ELK.

## 20. Tests
- Unit: validadores, mapeos, reglas de negocio en `Application`.
- Integración: Minimal APIs con `WebApplicationFactory`, DB con `Testcontainers` (SQL Server) o un `LocalDB`/Docker.
- Contratos: snapshot testing de OpenAPI.

## 21. Comandos de Setup Inicial
- Crear solución y proyectos:
  ```bash
  dotnet new sln -n Logistica
  dotnet new classlib -n Logistica.Domain
  dotnet new classlib -n Logistica.Application
  dotnet new classlib -n Logistica.Infrastructure
  dotnet new web -n Logistica.Api
  dotnet sln add Logistica.Domain Logistica.Application Logistica.Infrastructure Logistica.Api
  dotnet add Logistica.Application reference Logistica.Domain
  dotnet add Logistica.Infrastructure reference Logistica.Application Logistica.Domain
  dotnet add Logistica.Api reference Logistica.Application Logistica.Infrastructure
  ```
- Paquetes NuGet (ejemplos):
  ```bash
  dotnet add Logistica.Api package Swashbuckle.AspNetCore
  dotnet add Logistica.Api package Microsoft.AspNetCore.Authentication.JwtBearer
  dotnet add Logistica.Api package Microsoft.AspNetCore.Mvc.Versioning
  dotnet add Logistica.Application package FluentValidation
  dotnet add Logistica.Infrastructure package Microsoft.EntityFrameworkCore.SqlServer
  dotnet add Logistica.Infrastructure package Dapper
  dotnet add Logistica.Infrastructure package Serilog.AspNetCore
  dotnet add Logistica.Infrastructure package Serilog.Sinks.Seq
  dotnet add Logistica.Infrastructure package Serilog.Sinks.File
  dotnet add Logistica.Infrastructure package StackExchange.Redis
  dotnet add Logistica.Infrastructure package Hangfire.AspNetCore
  dotnet add Logistica.Infrastructure package Hangfire.SqlServer
  dotnet tool install --global dotnet-ef
  ```
- Scaffold EF (ajusta cadena de conexión):
  ```bash
  dotnet ef dbcontext scaffold "Server=localhost;Database=LogisticaDB;Trusted_Connection=True;TrustServerCertificate=True;" Microsoft.EntityFrameworkCore.SqlServer -c LogisticaDbContext -o Logistica.Infrastructure/EF/Models --use-database-names --no-pluralize --schema dbo --data-annotations
  ```

## 22. Endpoints Iniciales (ejemplos Minimal API)
- `GET /api/v1/vehiculos` (Flota): listado con filtros por estado.
- `POST /api/v1/viajes` (Operación): crea viaje desde cotización.
- `POST /api/v1/pagos` (Facturación): registra pago y actualiza estado.
- `GET /api/v1/documentos/vencimientos` (Alertas): documentos próximos a vencer.

## 23. Roadmap por Hitos
- Hito 1: solución, scaffolding EF, endpoints read-only (Clientes, Flota, Viajes).
- Hito 2: autenticación JWT/Identity, comandos de creación/actualización (Viajes, Facturas, Pagos).
- Hito 3: Hangfire jobs (vencimientos, mantenimientos), cache Redis para catálogos.
- Hito 4: observabilidad avanzada (Seq/OTel), health checks, hardening seguridad.
- Hito 5: reportes con Dapper y vistas operativas; performance tuning.

## 24. Riesgos y Mitigaciones
- Integridad por Sección (TiposMaestros): validar en capa de aplicación y auditar periódicamente.
- Rendimiento: índices y Dapper para consultas pesadas; cache.
- Concurrencia: `rowversion` en tablas críticas (opcional) y transacciones por caso de uso.
- Seguridad: rotación de tokens, bloqueo de cuentas; almacenar secretos seguro.

## 25. Guías de Estilo y Convenios
- Nombres de endpoints: `kebab-case` con `/api/v{version}/...`.
- DTOs: sufijos `Request/Response`.
- Validadores: uno por DTO; mensajes claros.
- Mapeos: funciones puras por slice.
- Errores: estructura estándar `{ traceId, code, message }`.

---
Este documento sirve como guía integral para crear y evolucionar el backend .NET sobre LogisticaDB, priorizando mantenibilidad, rendimiento y seguridad, con capacidad de escalar a microservicios en el futuro si el negocio lo requiere.