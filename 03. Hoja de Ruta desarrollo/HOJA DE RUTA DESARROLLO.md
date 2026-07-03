# ECOREX Tareas - Hoja de Ruta de Desarrollo

**Plan paso a paso para reconstruir el sistema de tareas/flujos/formularios sobre .NET 10, migrando desde el legacy WebForms sin heredar sus errores.**

Esta hoja de ruta describe el camino recomendado para construir el nuevo **ECOREX - Sistema de Tareas** usando el vault Obsidian como fuente de verdad. A diferencia de un producto nuevo, ECOREX Tareas es una **migracion**: existe un sistema legacy en produccion (ASP.NET WebForms + VB.NET sobre `db3dev` SQL Server) cuyo comportamiento y datos hay que preservar, corrigiendo de paso los 9 errores estructurales heredados. El desarrollo avanza desde la plataforma multi-tenant y la autenticacion hacia el nucleo de tareas/tableros, luego los motores configurables (flujos BPMN, formularios, reglas) y por ultimo el ETL de datos legacy. Construir en otro orden produce pantallas vistosas pero sin aislamiento real ni motor de proceso.

**Convenciones de esta guia:**

- **Vault Obsidian de especificaciones:** `C:\Users\acuartas\OneDrive - Bitcode IT Services S.A.S\Bitcode\13. Proyectos\044. Tareas\OBSIDIAN.tareas`
- **Codigo legacy (ORIGEN):** `C:\Desarrollo\core` (solucion `DoomBitcode`, proyecto `Bootstrap` / namespace `GestionMovil`)
- **Codigo destino (.NET 10):** `C:\DesarrolloIA\ECOREX.tareas` (clon local del repo destino)
- **Repositorio GitHub destino (oficial):** [alexandercuartas665/EcorexV](https://github.com/alexandercuartas665/EcorexV.git)
- **Documento maestro de vision:** [[Visión y entorno]]
- **Prototipo visual definitivo:** [[00 - Prototipo Final ECOREX]]
- **Inventario modular:** [[INVENTARIO GENERAL]]
- **Multi-tenant real:** [[Gestion de Empresas - Admin multi-tenant]]
- **DAL dual:** [[00 - Visión MotherData]]
- **Modelo de datos (plano del ETL):** [[Modelo Entidad-Relacion logico]]
- **BD del sistema actual (fuente del ETL / descubrimiento):** `db3dev` (SQL Server, copia de produccion safe). Tenant a migrar = sucursal `01` = **BITCODE**. Como conectar, guardrails y por que las credenciales NO estan en este repo: [[Conexion a la BD del sistema actual (db3dev)]]

**Stack destino:** .NET 10 / ASP.NET Core 10, Blazor (Server interactivo), EF Core 10, **DAL dual PostgreSQL o SQL Server**, Redis (cache de definiciones de flujo + locks), RabbitMQ + MassTransit, SignalR, Serilog, OpenTelemetry, Docker. Frontend 100% .NET (sin Node/React).

---

## ECOREX Tareas y la columna vertebral compartida Bitcode

Los productos de la familia (CUBOT.travels, CUBOT.nails, ECOREX Tareas...) **comparten la misma columna vertebral SaaS**: identidad y multi-tenancy (`PlatformUser`, `TenantUser`, JWT con `tenant_id`, filtros globales), consola PlatformAdmin (Tenants, Planes, Auditoria, integraciones), y los **5 motores base** (BPMN, Formularios, Reglas, Permisos, SignalR). Lo que cambia por producto es el nucleo operativo.

Para ECOREX Tareas, los 5 motores base NO son solo infraestructura: **son el producto**. El diferenciador frente a Asana/Trello/Monday es que el proceso, el formulario y la regla se configuran sin codigo. Por eso esta migracion invierte mas esfuerzo en el nucleo (Tareas + Flujos + Formularios + Reglas) que un vertical tipico.

**Dos estrategias de arranque (decidir antes de codear):**

1. **Reusar el spine de la familia** (recomendado si esta disponible): clonar el backbone SaaS ya probado (multi-tenant, auth, PlatformAdmin, integraciones Evolution/SendGrid/Slack) y construir encima el nucleo de tareas + los motores. Ahorra meses.
2. **Construir el destino desde cero**: scaffolding Clean Architecture nuevo. Mas control, mas trabajo.

En ambos casos, la fuente de la logica de negocio es el legacy (`C:\Desarrollo\core`) documentado en el vault, y los datos se migran por ETL desde `db3dev`.

---

## 0. Lectura obligatoria antes de escribir codigo

| Orden | Documento | Que extraer |
|-------|-----------|-------------|
| 1 | [[Visión y entorno]] | Arquitectura destino, stack, multi-tenant real, riesgos, seccion ORIGEN. |
| 2 | [[INVENTARIO GENERAL]] | Modulos (codigos 000XXX), capas, dependencias, orden de construccion. |
| 3 | [[Gestion de Empresas - Admin multi-tenant]] | Multi-tenant real, los 9 errores heredados a corregir, PlatformAdmin. |
| 4 | [[00 - Visión MotherData]] | DAL dual `IEcorexDbContext` (PostgreSQL + SQL Server). |
| 5 | [[Modelo Entidad-Relacion logico]] | Mapeo de tablas legacy -> entidades EF Core con `TenantId` (plano del ETL). |
| 6 | [[00 - Visión Flujos]] / [[00 - Visión Formularios]] | Motores WorkflowEngine y DynamicFormRenderer. |
| 7 | [[00 - Prototipo Final ECOREX]] | Aspecto y navegacion definitivos. |

**Prompt sugerido para el agente de desarrollo:**

```txt
Lee el vault Obsidian de ECOREX Tareas: Vision y entorno, INVENTARIO GENERAL,
Gestion de Empresas - Admin multi-tenant, 00 - Vision MotherData y Modelo
Entidad-Relacion logico.

Devuelveme:
1. Resumen de arquitectura destino en 15 puntos.
2. Como garantizarias el aislamiento multi-tenant real (que el legacy no tenia).
3. Como abstraes el DAL para soportar PostgreSQL Y SQL Server sin duplicar logica.
4. Los 9 errores heredados y como los corriges.
5. Orden de construccion y que datos migrar primero por ETL.
6. Decisiones tecnicas que deben quedar en CLAUDE.md.

No implementes codigo todavia. Confirma que entendiste el mapa.
```

---

## Primera tarea funcional: organizar el menu del sistema

> **Esto es lo PRIMERO que se construye una vez haya scaffold, multi-tenancy y autenticacion.** No se pueden armar vistas sueltas sin definir la columna vertebral de navegacion. El menu debe quedar prototipado en Blazor y conectado a las policies antes de implementar el contenido de cada pagina, para que el producto no nazca incoherente.

La base es el layout del [[00 - Prototipo Final ECOREX|prototipo final]]: doble panel (rail de iconos + sidebar `SKY SYSTEM`) con la navegacion en dos bloques.

### Menu objetivo (consola del tenant)

| Seccion | Item | Origen | Comentario |
|---------|------|--------|------------|
| **PRINCIPAL** | Inicio / Resumen | Prototipo | Dashboard "Buenos dias" con KPIs (Tareas activas, Proyectos, Flujos, Alertas). |
| PRINCIPAL | Anuncios | Prototipo | Tablon del workspace (badge de no leidos). Pendiente de completar. |
| PRINCIPAL | Gestor de tareas / Tableros | Prototipo | Kanban por proyecto con filtros (Usuario, Etiqueta, Categoria, Subcategoria, Fecha). |
| PRINCIPAL | Configuracion | Prototipo | Ajustes del tenant. |
| **MODULOS** | Proyectos | Legacy 000042 | Proyectos con equipo, etiquetas, fechas, tablero. |
| MODULOS | Flujos del proceso | Legacy 000291 | Editor BPMN (bpmn-js) + WorkflowEngine. |
| MODULOS | Formularios | Legacy 000131 | Constructor EAV -> DynamicFormRenderer. |
| MODULOS | Actividades (Crear/Administrar/Programar) | Legacy 000038/000636/000889 | Catalogo TIPO_TAR y su parametrizacion. |
| **CATALOGOS / SISTEMA** | Dependencias | Legacy 000850 | Organigrama (areas, equipos, responsables). |
| CATALOGOS / SISTEMA | Modulos web | Legacy 000109 | Registro de modulos con permisos y variables por tenant. |
| CATALOGOS / SISTEMA | Reglas | Legacy 000802 | Motor de reglas -> RulesEngine. |

### Entregables de esta primera tarea

1. `NavMenu.razor` + `MainLayout.razor` con el doble panel del prototipo (rail + sidebar), secciones PRINCIPAL / MODULOS / CATALOGOS.
2. Stubs de pagina por item con titulo y "Pendiente Fase X".
3. Cada item protegido por su policy (los del legacy se derivan de `PermissionsManager` -> policies .NET).
4. Badge en vivo donde aplique (ej. contador de tareas en "Gestor de tareas").
5. Sidebar plegable + modo oscuro (del prototipo).

> Cada modulo posterior llena su stub. Esto evita paginas sueltas sin ubicacion.

---

## 1. Pre-requisitos del entorno

| Herramienta | Version | Uso | Validar |
|-------------|---------|-----|---------|
| Git | latest | Control de versiones | `git --version` |
| .NET SDK | 10.x | Backend, Blazor, workers | `dotnet --version` |
| Docker Desktop | latest | Postgres, SQL Server, Redis, RabbitMQ | `docker --version` |
| PowerShell 7 | latest | Scripts en Windows | `pwsh --version` |
| DBeaver / Azure Data Studio | latest | Inspeccion Postgres Y SQL Server | abrir ambas conexiones |
| Playwright .NET | latest | E2E del frontend Blazor | `Microsoft.Playwright` |

**Decision:** se asume .NET 10. Si no esta disponible, .NET 9 como puente documentado. Frontend sin Node.js (Playwright .NET para E2E).

---

## 2. Setup del repositorio y la solucion .NET 10

Estructura destino Clean Architecture con **DAL dual** (dos proyectos de infraestructura, uno por motor):

```txt
ECOREX.tareas/
├── src/
│   ├── Ecorex.Domain/                    (entidades + enums, sin EF)
│   ├── Ecorex.Application/               (CQRS, servicios, DTOs, validaciones)
│   ├── Ecorex.Infrastructure/            (IEcorexDbContext, repos, integraciones comunes)
│   ├── Ecorex.Infrastructure.Postgres/   (PostgresDbContext + migraciones _pg_)
│   ├── Ecorex.Infrastructure.SqlServer/  (SqlServerDbContext + migraciones _sql_)
│   ├── Ecorex.Web/                        (Blazor: consola del tenant)
│   ├── Ecorex.Web.Platform/              (Blazor: consola PlatformAdmin, separada)
│   ├── Ecorex.Api/                        (Minimal API + JWT + webhooks)
│   ├── Ecorex.Workers/                    (BackgroundServices/Hangfire)
│   └── Ecorex.Shared/                     (contratos comunes)
├── tests/
│   ├── Ecorex.Domain.Tests/
│   ├── Ecorex.Application.Tests/
│   └── Ecorex.Integration.Tests/         (Testcontainers Postgres + SQL Server)
├── etl/                                   (proyecto/consola de migracion desde db3dev)
├── deploy/docker/                         (compose: postgres, sqlserver, redis, rabbitmq)
├── docs/decisiones/                       (ADRs)
├── CLAUDE.md
└── .github/workflows/
```

**Decision clave:** el DAL es una abstraccion `IEcorexDbContext` con dos implementaciones elegidas por `Database:Provider`. NUNCA se escribe SQL crudo fuera de repositorios especificos por proveedor. Ver [[00 - Visión MotherData]].

### 2.1 CLAUDE.md del repo destino

```txt
Crea CLAUDE.md en C:\DesarrolloIA\ECOREX.tareas con:
1. Contexto: ECOREX Tareas es la reconstruccion .NET 10 del gestor de tareas/flujos/
   formularios/reglas legacy (WebForms/GestionMovil). Multi-tenant real.
2. Fuente de verdad: vault Obsidian OBSIDIAN.tareas (INVENTARIO GENERAL, Vision y entorno).
3. Codigo legacy de referencia: C:\Desarrollo\core (solucion DoomBitcode).
4. Stack: .NET 10, ASP.NET Core 10, Blazor, EF Core 10, DAL dual Postgres/SQL Server,
   Redis, RabbitMQ, MassTransit, SignalR, Serilog, OpenTelemetry, Docker.
5. Multi-tenancy REAL: todo dato lleva TenantId; HasQueryFilter global + RLS en BD;
   tests de aislamiento cross-tenant que DEBEN fallar si se rompe.
6. DAL dual: nunca SQL crudo fuera de repos por proveedor; tests corren en AMBOS motores.
7. Los 9 errores heredados estan PROHIBIDOS (SQL concatenado, sin transacciones,
   DELETE sin cascada, secretos en codigo, sin auditoria, aislamiento por disciplina).
8. Motores base: WorkflowEngine, DynamicFormRenderer, RulesEngine, PermissionsManager.
9. Orden: multi-tenant, auth, menu, Tareas/Tableros, motores, Dependencias/Modulos, ETL.
```

### 2.2 Validar el scaffold antes de tocar dominio

`dotnet restore` + `dotnet build` verde. Levantar Postgres Y SQL Server con compose. Aplicar migracion inicial en ambos contextos. Entrar a la consola en `https://localhost:7xxx` y ver el layout del prototipo con el menu (stubs).

---

## 3. Docker local para servicios base

> **Aviso de puertos (critico).** La maquina de desarrollo ya corre varios stacks
> Docker en paralelo (doktrino, cubot, cubot-travels, cubotrm, visal, entre otros).
> Los puertos "estandar" (5432, 6379, 5672...) se ven libres solo porque cada
> proyecto usa su propio offset de host. Para que ECOREX Tareas **no choque con
> ningun hermano ni ahora ni al arrancar todos juntos**, usa un **bloque de puertos
> dedicado** y **nombres de contenedor/volumen con prefijo `ecorex-tareas-`**.

### 3.1 Bloque de puertos dedicado (verificado libre 2026-07-03)

| Servicio | Puerto host | Puerto interno | Contenedor |
|---|---|---|---|
| PostgreSQL 16 | **5442** | 5432 | `ecorex-tareas-postgres` |
| SQL Server 2022 | **1443** | 1433 | `ecorex-tareas-sqlserver` |
| Redis | **6389** | 6379 | `ecorex-tareas-redis` |
| RabbitMQ (AMQP) | **5682** | 5672 | `ecorex-tareas-rabbitmq` |
| RabbitMQ (panel) | **15682** | 15672 | `ecorex-tareas-rabbitmq` |
| Adminer / pgAdmin | **8092** | 8080 | `ecorex-tareas-adminer` |

Definir `name: ecorex-tareas` en el `docker-compose.yml` (project name) para que los
**volumenes queden prefijados** (`ecorex-tareas_postgres-data`, etc.) y no se
mezclen con los de otros proyectos. Los puertos de host se parametrizan en `.env`
(no versionado) para poder reajustarlos sin tocar el compose:

```env
PG_PORT=5442
MSSQL_PORT=1443
REDIS_PORT=6389
RABBITMQ_PORT=5682
RABBITMQ_UI_PORT=15682
ADMINER_PORT=8092
```

### 3.2 Validaciones PREVIAS antes de `docker compose up` (pre-flight)

El arranque NO debe asumir que el entorno esta limpio. Un script de pre-flight
(`deploy/docker/preflight.ps1` / `.sh`) corre ANTES de levantar los contenedores y
**aborta con mensaje claro** si algo falla. Chequea, en orden:

1. **Docker corriendo**: `docker info` responde; si no, abortar ("inicia Docker Desktop").
2. **Cada puerto de host libre**: por cada puerto del `.env`, verificar que nadie
   escucha. Si esta ocupado, abortar indicando el puerto y sugerir el siguiente libre.
3. **Sin colision de nombres**: que no exista ya un contenedor `ecorex-tareas-*`
   (de una corrida previa a medias) ni un volumen con datos de otro proyecto.
4. **Recursos minimos**: espacio en disco y memoria para SQL Server (>= 2 GB RAM asignada a Docker).

```powershell
# preflight.ps1 (resumen)
$ports = @{ PostgreSQL=5442; SqlServer=1443; Redis=6389; RabbitMQ=5682; RabbitUI=15682; Adminer=8092 }
$fail = $false
if (-not (docker info 2>$null)) { Write-Error "Docker no esta corriendo."; exit 1 }
foreach ($k in $ports.Keys) {
  $p = $ports[$k]
  if (Get-NetTCPConnection -LocalPort $p -State Listen -ErrorAction SilentlyContinue) {
    Write-Warning "Puerto $p ($k) OCUPADO. Ajusta el .env a uno libre."; $fail = $true
  }
}
if (docker ps -a --format '{{.Names}}' | Select-String '^ecorex-tareas-') {
  Write-Warning "Ya existen contenedores ecorex-tareas-*. Corre 'docker compose down' antes."; $fail = $true
}
if ($fail) { exit 1 } else { Write-Output "Pre-flight OK. Puedes levantar el stack." }
```

### 3.3 Validaciones antes de crear / migrar las bases de datos

Levantar el contenedor NO es lo mismo que tener la BD lista. Antes de crear el
esquema o aplicar migraciones, el arranque de la app (`Program.cs` / job de bootstrap)
debe validar, de forma **idempotente y no destructiva**:

1. **Salud del motor**: esperar el healthcheck real, no un `sleep`. PostgreSQL con
   `pg_isready`; SQL Server con `SELECT 1` via `sqlcmd`; Redis `PING`; RabbitMQ `ping`.
   Reintentar con backoff; abortar si no responde en N intentos.
2. **Proveedor correcto**: `Database:Provider` (`Postgres`/`SqlServer`) debe coincidir
   con el motor que respondio; si no, abortar (evita migrar el contexto equivocado).
3. **Existencia de la BD/esquema**: si la base ya existe, **NO recrearla ni dropearla**.
   Solo aplicar migraciones pendientes (`GetPendingMigrations()`); si no hay
   pendientes, no hacer nada.
4. **Migracion segura**: aplicar con `Migrate()` dentro de transaccion cuando el motor
   lo permita; nunca `EnsureDeleted`/`EnsureCreated` fuera de tests. Si una migracion
   falla, abortar el arranque (no dejar la BD a medias).
5. **Aislamiento activo antes de sembrar**: verificar que el filtro global de tenant y
   la RLS estan aplicados (test rapido: consulta sin tenant activo devuelve 0 filas).
   Recien ahi correr los seeders.
6. **Seed solo si vacio**: los seeders (planes demo, tenant piloto `SKY SYSTEM`) se
   ejecutan solo si la tabla objetivo esta vacia; nunca duplican.

```csharp
// Program.cs (bootstrap idempotente)
await WaitForHealthyAsync(db);                       // 1. backoff hasta healthy
EnsureProviderMatches(config, db);                    // 2. provider correcto
var pending = await db.Database.GetPendingMigrationsAsync();
if (pending.Any()) await db.Database.MigrateAsync();  // 3-4. solo pendientes, nunca drop
AssertTenantIsolationActive(db);                      // 5. RLS + query filter activos
await SeedIfEmptyAsync(db);                            // 6. seed idempotente
```

Este bloque es la contraparte destino del arranque del legacy, que abria la BD por
`CargaConfig` sin validar estado ni idempotencia. Ver [[00 - Visión MotherData]] y
[[CargaConfig - Decode Config.xml]] para el contraste.

### 3.4 Criterio de salida del setup

`preflight` pasa, `docker compose up -d` levanta los 5 servicios, los healthchecks
quedan verdes, la app arranca aplicando solo migraciones pendientes y el test de
aislamiento cross-tenant corre en **ambos** motores. Recien entonces empieza el
trabajo de dominio.

---

## 4. Fundamentos de dominio y multi-tenancy real (BLOQUEANTE)

El corazon de la migracion: el aislamiento que el legacy NO tenia.

- `BaseEntity`, `ITenantScoped { Guid TenantId }`, entidades globales (`Tenant`, `PlatformUser`, `TenantUser`, `AdminAuditLog`).
- `IEcorexDbContext` con `HasQueryFilter` global aplicado por reflexion a TODA entidad `ITenantScoped`.
- **RLS** en ambos motores (Postgres `CREATE POLICY`, SQL Server `SECURITY POLICY`) como defensa en profundidad.
- Interceptor de auditoria que escribe `AdminAuditLog` en la misma transaccion.

**Test de aislamiento (criterio de aceptacion, NO negociable):** con tenant A activo una consulta no ve datos de B; un intento de leer cross-tenant DEBE fallar. Este test corre en la CI en AMBOS motores. Ver [[Gestion de Empresas - Admin multi-tenant]] seccion 3.

---

## 5. Autenticacion, autorizacion y PlatformAdmin

Login local (cookie en Blazor) + JWT en Api. Claims: `sub`, `tenant_id`, `tenant_role`, `platform_role`. Policies: `PlatformAdmin` (consola separada, MFA obligatorio) y `TenantMember`. Los permisos del legacy (`PermissionsManager` + menu 000109) se derivan a **policies .NET** dinamicas. Refresh tokens con rotacion; secretos en Key Vault. Ver [[Puntos ciegos y motores transversales]] (Login y PermissionsManager).

---

## 6. Nucleo operativo: Tareas, Tableros y Proyectos

La fase que el tenant usa a diario. Se apoya en el prototipo, no copia estructura monolitica.

| Orden | Modulo | Resultado | Link |
|-------|--------|-----------|------|
| 1 | Tareas | `TaskItem` (titulo, actividad, proyecto, responsable, estado, fechas, etiquetas). | [[ctrTareasII - Spec para reconstruir en Claude Design]] |
| 2 | Detalle de tarea | Panel con cabecera + worklog + concurrencia optimista. | [[ctrVertareasII - Spec para reconstruir en Claude Design]] |
| 3 | Tableros | Kanban por proyecto (Por hacer/En progreso/En revision/Completado) + filtros + SignalR. | [[00 - Prototipo Final ECOREX]] |
| 4 | Proyectos | Contenedor con equipo, etiquetas, vistas Tablero/Timeline/Calendario. | [[Tareas y Proyectos - paginas basicas]] |

**Validacion:** dos usuarios ven el mismo tablero actualizarse en vivo (SignalR); mover una tarjeta cambia el estado con auditoria.

---

## 7. Motores base configurables (el diferenciador)

Se portan del legacy conservando el modelo mental (ver capas 3, 4 y las notas de Reglas).

### 7.1 WorkflowEngine (Flujos BPMN)
`AdmWorkflow` -> `WorkflowEngine.AdvanceAsync`. Tablas `workflow_definition` / `workflow_instance` / `workflow_step_history` (Guid v7). Editor bpmn-js en Blazor. Deteccion de ciclos (limite de iteraciones heredado). **Validacion:** un flujo avanza estado por estado y no entra en bucle infinito. Ver [[00 - Visión Flujos]] y [[AdmWorkflow - Motor de ejecucion]].

### 7.2 DynamicFormRenderer (Formularios)
`crtCargaEncuestaII` -> renderer polimorfico Blazor. EAV `FORX_DATA` -> `form_response.data jsonb` / `nvarchar(max)`. Union con flujos via `form_flow_link`. **Validacion:** un formulario con 5+ tipos de control captura y persiste respuestas. Ver [[00 - Visión Formularios]] y [[Constructor - Patron EAV y motor visual]].

### 7.3 RulesEngine (Reglas)
`cl_gestion_reglas` -> `RulesEngine` con registro de verbos tipado (elimina el RCE de `Activator.CreateInstance`). **Validacion:** una regla Ensamblado ejecuta y queda en el historial. Ver [[Reglas - Quien invoca realmente (cierre)]].

---

## 8. Modulos de sistema: Dependencias y Modulos web

- **Dependencias** (000850): organigrama del tenant (areas, equipos, responsables); alimenta asignacion y permisos.
- **Modulos web** (000109): registro de modulos con permisos y variables por tenant; conecta con [[Gestion de Empresas - Admin multi-tenant]] (habilitacion por tenant) y con las policies.

---

## 9. ETL: migracion de datos legacy (pieza critica de esta migracion)

Lo que un producto nuevo no tiene: traer los datos de `db3dev` (SQL Server legacy) al esquema destino, respetando el multi-tenant real.

### 9.1 Estrategia
Proyecto `etl/` (consola .NET) que lee `db3dev` y escribe al destino via `IEcorexDbContext`. Se corre por tenant (`SUCURSAL`), idempotente, con log por fila y verificacion de conteos. La conexion, los guardrails y las tablas ancla estan en [[Conexion a la BD del sistema actual (db3dev)]] (BD real: SQL Server 2022, ~1004 tablas, 105 tenants, 532 modulos; tenant a migrar = `01` = BITCODE).

### 9.2 Mapeo (del [[Modelo Entidad-Relacion logico]])

| Familia legacy | Entidad destino | Nota |
|---|---|---|
| `SUCURSAL` | `Tenant` (Guid v7 + `PublicSlug`) | conservar CODIGO como `LegacyId` |
| `USUARIO.SUCURSAL` | `UserTenant` | permite N tenants por usuario |
| `TAREAS` / `TAR_*` | `TaskItem` + `Project` | `int REG` -> Guid v7 / LegacyId |
| `DOC_PROCESOS*` / `TAR_WORKFLOW_*` | `workflow_definition` / `_instance` | XML BPMN estandar (portable) |
| `ENCUESTAS_MOV*` + `FORX_DATA` | `FormDefinition` + `FormResponse` (jsonb) | consolidar EAV a jsonb |
| `CONTROL_REGLAS*` | `Rule` | verbos tipados |
| (columna `SUCURSAL` en cada tabla) | `TenantId` (Guid) + filtro global | el filtro deja de ser manual |

### 9.3 Validacion del ETL
- Round-trip: un registro migrado se lee identico en el destino.
- Conteos por tabla/tenant coinciden origen vs destino.
- Test de aislamiento sobre datos migrados (un tenant no ve otro).

---

## 10. CI/CD y deployment

`.github/workflows`: build + **matriz de tests dual (PostgreSQL + SQL Server)** + `dotnet format` + SAST. Regla de merge: no entra PR si falla build, el **test de aislamiento cross-tenant**, la matriz dual, auth o formato. Ambientes Dev -> Staging -> Prod (blue/green, rollback <2 min). Ver [[CICD Pipeline]], [[Deploy a Produccion - Docker e hibrido]] y [[Backup y DR por proveedor]].

---

## 11. Checklist antes de cada commit

- [ ] `dotnet build` y `dotnet test` pasan en AMBOS motores (Postgres y SQL Server).
- [ ] El test de aislamiento cross-tenant pasa.
- [ ] No se agregaron secretos al repo (nada de keys en codigo, corrigiendo el error de `CargaConfig.vb`).
- [ ] No hay queries tenant-scoped sin el filtro global.
- [ ] No hay SQL crudo fuera de repositorios por proveedor.
- [ ] Operaciones multi-tabla envueltas en transaccion.
- [ ] Borrados son soft-delete con cascada (nunca DELETE directo).
- [ ] Si toca PlatformAdmin, hay auditoria.
- [ ] Si se cierra un modulo, se actualiza [[INVENTARIO GENERAL]].
- [ ] Decision arquitectonica nueva -> `docs/decisiones` + reflejo en el vault.

---

## 12. Riesgos principales durante el desarrollo

| Riesgo | Senal | Mitigacion |
|--------|-------|------------|
| Fuga entre tenants | Endpoint/query sin filtro | Filtro global EF + RLS + test cross-tenant que debe fallar |
| Divergencia entre motores | Feature que solo corre en uno | Matriz de tests dual obligatoria en CI |
| ETL con perdida de datos | Conteos que no cuadran | Migracion idempotente + round-trip + verificacion de conteos |
| Flujo en ciclo infinito | Tarea que nunca avanza | Limite de iteraciones + deteccion de ciclos (heredado) |
| Migracion EAV -> jsonb | Respuestas mal mapeadas | Tests de round-trip por tipo de control |
| Secretos filtrados | Key en codigo/logs | Key Vault + DataProtection (corrige `CargaConfig.vb:7`) |
| Concurrencia en tarea/flujo | Doble edicion | Concurrencia optimista (rowversion / xmin) |
| Zona horaria | Fechas mal calculadas | TZ del tenant + normalizar a UTC |

---

## 13. Siguiente paso recomendado

1. Decidir la estrategia de arranque (reusar spine de la familia vs scaffold nuevo).
2. Montar el scaffold Clean Architecture con DAL dual y el test de aislamiento vacio (que debe existir desde el dia 1).
3. Implementar la "Primera tarea funcional: el menu del sistema" con el layout del prototipo.
4. Producir la spec detallada del ETL por familia de tablas (a partir del [[Modelo Entidad-Relacion logico]]).

*Documento vivo. Actualizar cuando se complete un hito, cambie una decision arquitectonica o se cree una especificacion de modulo nueva.*
