---
tipo: vision
area: General
estado: vision del sistema destino (.NET 10) + entorno del sistema origen
stack_destino: .NET 10 / ASP.NET Core / EF Core 10 / Blazor
---

# Vision del sistema y entorno

> Vision general del nuevo **ECOREX - Sistema de Tareas**, la reconstruccion
> sobre **.NET 10** del gestor de tareas/actividades que hoy corre en WebForms
> (`GestionMovil`). Este documento describe el sistema DESTINO — que se va a
> construir y por que — y conserva al final el entorno del sistema ORIGEN como
> referencia de migracion. El aspecto y la navegacion definitivos estan en
> [[00 - Prototipo Final ECOREX]].

## 1. Vision general de la plataforma

ECOREX es una plataforma SaaS multi-tenant de **gestion operativa del trabajo**:
tareas, proyectos, flujos de proceso (BPMN), formularios dinamicos y reglas de
negocio, todo gobernado por empresa (tenant). El activo que administra no es una
agenda ni un catalogo de ventas, sino **el trabajo de la organizacion**: quien
hace que, bajo que proceso, con que datos y siguiendo que reglas. Una tarea que
se pierde o un flujo que se traba es productividad perdida.

El sistema no es una simple lista de pendientes. Es un **motor de procesos de
negocio** donde cada actividad (categoria `TIPO_TAR`) puede tener asociado un
flujo BPMN que la guia estado por estado, un formulario dinamico que captura sus
datos, y reglas que automatizan decisiones. Alrededor de ese nucleo viven los
tableros Kanban, los proyectos, el organigrama de dependencias, el CRM, el
inventario y la analitica. Cada empresa (tenant) opera aislada con sus propios
usuarios, modulos habilitados, actividades, flujos, formularios y parametros,
mientras el operador de la plataforma gobierna planes, limites y salud global.

El diferenciador de ECOREX frente a un gestor de tareas generico (Asana, Trello,
Monday) es que **el proceso, el formulario y la regla son configurables sin
codigo**: un administrador modela un flujo en BPMN, arma su formulario de captura
y define reglas, y el sistema los ejecuta. Esta capacidad viene de 15 años de
motores probados en el ecosistema Bitcode/ECOREX que se portan a .NET 10 (ver 3.2).

Desde el punto de vista operativo, ECOREX concentra:

- Creacion, asignacion y seguimiento de tareas y actividades.
- Tableros Kanban por proyecto con columnas configurables.
- Proyectos con equipos, etiquetas, fechas y timeline.
- Flujos de proceso BPMN que dirigen las actividades.
- Formularios dinamicos (patron EAV) para capturar datos de negocio.
- Motor de reglas para automatizar decisiones y acciones.
- Organigrama de dependencias (areas, equipos, responsables).
- Registro de modulos con permisos y variables por tenant.
- Anuncios, notificaciones en tiempo real y analitica.

La plataforma se disena desde el inicio para concurrencia (varios usuarios
trabajando a la vez), aislamiento estricto entre empresas, trazabilidad completa
y escalabilidad horizontal.

## 2. Analisis del prototipo

El prototipo final ([[00 - Prototipo Final ECOREX]], `ECOREX - Prototipo Final.html`)
fija el aspecto y la interaccion definitivos. Es una SPA React de referencia
visual — no codigo de produccion — que evidencia con claridad la orientacion del
producto:

- **Layout de doble panel**: un rail de iconos permanente (extrema izquierda) y
  un sidebar contextual `SKY SYSTEM / Plan Empresa - ECOREX` con la navegacion
  agrupada en **PRINCIPAL** (Inicio, Anuncios, Gestor de tareas, Configuracion) y
  **MODULOS** (los procesos del tenant, cada uno con su codigo `000XXX`).
- **Inicio / Resumen**: dashboard contextual ("Buenos dias, Guadalupe") con
  resumen de pendientes y tarjetas KPI (Tareas activas, Proyectos en curso,
  Flujos ejecutandose, Alertas).
- **Gestor de tareas / Tableros**: lista de tableros con filtros (Usuario,
  Etiqueta, Categoria, Subcategoria, Fecha) y tarjetas de tablero con su pipeline.
- **Proyecto - detalle + Kanban**: cabecera (Estado, Asignados, Fecha limite,
  Etiquetas) + tablero `Por hacer / En progreso / En revision / Completado`.
- **Dependencias**: organigrama de la empresa (modulo `000850`).
- **Modulos web**: registro de modulos con permisos y variables por tenant
  (modulo `000109`).

El estilo es claro, moderno, estilo Linear/Height: tarjetas KPI con tile de icono
en color suave, breadcrumbs, boton `Compartir`, buscador global (Cmd+K), modo
oscuro. La oportunidad arquitectonica es convertir esta capa visual estatica en
un producto empresarial modular con backend .NET 10, persistencia transaccional,
aislamiento multi-tenant real y motores de proceso/formularios/reglas.

### 2.1 Decision de stack

El producto se construye 100% sobre stack Microsoft con **.NET 10 / ASP.NET Core
/ Blazor**. El unico artefacto JavaScript del repositorio es el prototipo visual
de referencia; no condiciona el stack. Esta decision mantiene coherencia
tecnologica, comparte DTOs, validaciones y contratos entre frontend y backend en
una sola solucion, y usa SignalR para tiempo real (tableros y notificaciones
vivas sin recargar).

## 3. Arquitectura general recomendada

La solucion se construye como **monolito modular con Clean Architecture**,
preparada para evolucionar a microservicios cuando el volumen lo justifique.

```txt
Frontend Blazor (App del tenant + Consola PlatformAdmin)
        |
API / Gateway (ASP.NET Core 10)
        |
-----------------------------------------
| Auth & Tenant Resolver                |
| Task Service (TAREAS)                 |  <- nucleo: tareas, tableros, estados
| Project Service (PROYECTOS)           |
| Workflow Engine (FLUJOS BPMN)         |  <- motor portado AdmWorkflow
| Dynamic Forms Engine (FORMULARIOS)    |  <- motor portado EAV
| Rules Engine (REGLAS)                 |  <- motor portado cl_gestion_reglas
| Org Service (DEPENDENCIAS)            |
| Module Registry (MODULOS WEB)         |  <- permisos y variables por tenant
| Notification Service (SignalR)        |
| Analytics Service                     |
-----------------------------------------
        |
Event Bus (RabbitMQ / MassTransit)
        |
Workers Asincronos (recordatorios, escalamientos, jobs de flujo)
        |
DAL dual (PostgreSQL o SQL Server) + Redis + Object Storage
```

El **Workflow Engine** es el componente mas delicado: traduce la definicion BPMN
+ el estado actual de la tarea + las reglas de cada nodo en "cual es el siguiente
estado", garantizando que un flujo nunca entre en ciclo infinito (limite de
iteraciones heredado del origen) ni deje tareas huerfanas. Redis se usa para
cache de definiciones de flujo y locks de avance.

### 3.1 DAL dual PostgreSQL / SQL Server

El motor de acceso a datos es una **abstraccion `IEcorexDbContext`** con dos
backends: `PostgresDbContext` (Npgsql) y `SqlServerDbContext`
(Microsoft.Data.SqlClient). La eleccion se hace por configuracion
(`appsettings.json` -> `Database:Provider`), sin cambios de codigo. Esto permite
alojar cada tenant en el motor que su contrato exija: PostgreSQL para el SaaS
gestionado, SQL Server para clientes enterprise con infraestructura Windows
existente (como el `db3dev` legacy). El patron es identico al del vault
CUBOT.nails; ver su nota "Motor SQL DAL Dual - PostgreSQL y SQL Server".

### 3.2 Modulos base portados de Bitcode/ECOREX

ECOREX .NET 10 NO se construye desde cero: hereda motores transversales con 15
años de produccion, adaptados a .NET 10:

1. **Motor BPMN** — `AdmWorkflow` -> `WorkflowEngine`. Ver [[00 - Visión Flujos]].
2. **Constructor de Formularios** — `crtCargaEncuestaII` -> `DynamicFormRenderer`. Ver [[00 - Visión Formularios]].
3. **Motor de Reglas** — `cl_gestion_reglas` -> `RulesEngine`. Ver [[Reglas - Quien invoca realmente (cierre)]].
4. **PermissionsManager** — permisos por menu -> policies .NET. Ver [[Puntos ciegos y motores transversales]].
5. **DAL** — `MotherData.AdmDatos` -> `IEcorexDbContext` dual. Ver [[00 - Visión MotherData]].

## 4. Modulo de gestion multi-tenant (real)

Cada empresa es un tenant aislado con sus propios usuarios, modulos, actividades,
flujos, formularios, reglas y parametros. **A diferencia del origen** (que aislaba
por una columna `SUCURSAL` y disciplina del desarrollador), el destino hace del
aislamiento una invariante del sistema:

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    project_id UUID,
    activity_type_id UUID,
    workflow_instance_id UUID,
    title VARCHAR(300) NOT NULL,
    status VARCHAR(40) NOT NULL,
    assigned_to UUID,
    due_date DATE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

Toda entidad implementa `ITenantScoped` y el filtro global es obligatorio:

```csharp
modelBuilder.Entity<TaskItem>()
    .HasQueryFilter(x => x.TenantId == _tenantProvider.CurrentTenantId);
```

Sobre el filtro EF Core se aplica **Row-Level Security** en la BD (Postgres
`CREATE POLICY`, SQL Server `SECURITY POLICY`) como defensa en profundidad: aunque
un reporte o job salte EF Core, la BD niega el cruce. El detalle de este modelo y
la correccion de los 9 errores heredados esta en
[[Gestion de Empresas - Admin multi-tenant]]. JWT con `tenant_id`, middleware de
resolucion de tenant, claims y auditoria completan la defensa. **Nunca debe haber
fuga de datos entre empresas.**

## 5. Modulo nucleo: Tareas y Tableros

La tarea es la entidad transaccional central: titulo, descripcion, actividad
(`TIPO_TAR`), proyecto, responsable, fechas, etiquetas, estado y — opcionalmente —
una instancia de flujo BPMN y un formulario asociado. Los **tableros** (Kanban)
agrupan tareas por proyecto con columnas configurables (`Por hacer / En progreso /
En revision / Completado`) y filtros por usuario, etiqueta, categoria,
subcategoria y fecha.

El ciclo de vida basico de una tarea:

```txt
Creada -> En progreso -> En revision -> Completada
   |                          |
   +-> Bloqueada              +-> Devuelta (reabre)
```

Cuando la actividad tiene flujo asociado, el estado NO es libre: lo dicta el
Workflow Engine (seccion 7). El detalle de la UI de creacion y detalle vive en
[[ctrTareasII - Spec para reconstruir en Claude Design]] y
[[ctrVertareasII - Spec para reconstruir en Claude Design]].

## 6. Modulo de Proyectos

Un proyecto agrupa tareas bajo un objetivo comun con equipo asignado (avatares),
etiquetas, fecha limite y vistas (Tablero, Timeline, Calendario). En el prototipo,
`PRY-0042 Comercial - Requerimiento Infraestructura` ilustra un proyecto con su
pipeline de aprobacion. Los proyectos alimentan los KPIs del dashboard
("Proyectos en curso") y son el contenedor natural de los tableros.

## 7. Modulo de Flujos de Proceso (BPMN)

El corazon diferenciador. Cada actividad puede tener un **flujo BPMN 2.0** que la
dirige estado por estado. El Workflow Engine (`AdmWorkflow` portado) resuelve las
transiciones combinando la definicion del flujo, el estado actual y las reglas de
cada nodo:

```txt
SiguienteEstado(tarea) =
      Nodo actual del flujo
    + Reglas del nodo (verbos Ensamblado)
    + Condiciones de las transiciones
    -> Nuevo estado (o fin del flujo)
```

Reglas criticas del motor:

- Deteccion de ciclos: maximo de iteraciones para evitar bucles infinitos.
- Cada nodo puede ejecutar reglas (integracion con el Motor de Reglas).
- El XML BPMN es estandar OMG (portable a bpmn.io, Camunda, Activiti) — validado.
- Toda transicion emite eventos (`task.state.changed`, `workflow.completed`) para
  notificaciones, metricas y automatizacion.

El editor visual usa **bpmn-js** embebido en Blazor. Detalle en
[[00 - Visión Flujos]], [[AdmWorkflow - Motor de ejecucion]] y
[[Parametrizacion por nodo - panel Propiedades]].

## 8. Modulo Constructor de Formularios

Motor de formularios dinamicos con patron **EAV**: un administrador diseña un
formulario (preguntas, tipos de control, validaciones) sin codigo, y el motor lo
renderiza y captura respuestas. En el origen las respuestas viven en `FORX_DATA`;
en el destino migran a columnas `jsonb` (Postgres) / `nvarchar(max)` (SQL Server)
gestionadas por el DAL dual. Los formularios se asocian a actividades y flujos
(union `FORX_DATA_FLUJO`), de modo que un paso del proceso captura datos
estructurados. Detalle en [[00 - Visión Formularios]] y
[[Constructor - Patron EAV y motor visual]].

## 9. Modulo Motor de Reglas

Motor de reglas de negocio configurable (`cl_gestion_reglas` portado, modulo
`000802`). Una regla asocia una condicion a una accion (verbo). Ejecuta acciones
sobre tareas, formularios y flujos: validar, calcular, notificar, disparar otro
flujo. Se invoca desde los nodos de un flujo o de forma directa. El origen tiene
3 modos declarados (mDATA / Execute / Ensamblado) pero solo **Ensamblado** esta
implementado; el destino los formaliza con un registro de verbos tipado. Detalle
en [[Reglas - Quien invoca realmente (cierre)]] y
[[Reglas - Catalogo real y verbos Ensamblado]].

## 10. Modulo de Dependencias (Organigrama)

Estructura organizacional del tenant (modulo `000850`): areas, equipos y
responsables en un arbol (`ECOREX S.A.S > Gerencia General > Comercial /
Operaciones / Tecnologia`) con conteo de usuarios por nodo. Alimenta la
asignacion de tareas y los permisos por area. En el prototipo se ve como
organigrama con KPIs (Dependencias, Usuarios, Areas) y panel de detalle.

## 11. Modulo de Modulos web (Registro)

Registro de todos los modulos del sistema (modulo `000109`): cada modulo tiene un
codigo `000XXX`, permisos gestionables y **variables por tenant**. Es la pieza que
hace configurable que ve y que puede hacer cada empresa. En el destino, los
permisos por menu del origen evolucionan a **policies .NET** consumidas por el
PermissionsManager. Se conecta con [[Gestion de Empresas - Admin multi-tenant]]
(habilitacion de modulos por tenant).

## 12. Backend recomendado

Stack empresarial destino:

- **.NET 10 / ASP.NET Core 10** (.NET 9 solo como puente temporal documentado si 10 no esta disponible).
- **Entity Framework Core 10** + DAL dual PostgreSQL / SQL Server (filtros globales por tenant, jsonb/nvarchar para campos dinamicos).
- **Blazor** (Server interactivo) para app del tenant y consola PlatformAdmin.
- **SignalR** para tableros y notificaciones en tiempo real.
- **Redis** para cache (definiciones de flujo, permisos), locks y rate limiting.
- **RabbitMQ + MassTransit** para eventos de negocio y workers.
- **MediatR, FluentValidation, Serilog, OpenTelemetry, Hangfire/BackgroundService**.
- **Docker** + despliegue hibrido (VPS / Railway) preparado para Kubernetes.

Arquitectura Clean por proyecto: `Domain, Application, Infrastructure,
Infrastructure.Postgres, Infrastructure.SqlServer, Api, Web, Web.Platform,
Workers, Shared`. Patrones: CQRS, Repository, Unit of Work, Domain Events, DTOs
desacoplados, Tenant Resolution Middleware, Correlation IDs.

## 13. Persistencia y base de datos

DAL dual como base transaccional. El sistema gestiona: tenants, usuarios, roles,
dependencias, modulos, actividades, tareas, proyectos, tableros, definiciones e
instancias de flujo, formularios y respuestas, reglas, parametros, auditoria y
metricas. Indices clave orientados al tenant:

```sql
CREATE INDEX idx_task_tenant_status ON tasks(tenant_id, status, due_date);
CREATE INDEX idx_task_project ON tasks(tenant_id, project_id);
CREATE INDEX idx_wf_instance ON workflow_instances(tenant_id, entity_id);
```

Redis para: definiciones de flujo cacheadas, permisos por usuario, locks de avance
de flujo, presencia y colas rapidas. Detalle del acceso a datos en
[[Manejo de Datos - Alias, parametros, UDFs, consecutivos]] y [[00 - Visión MotherData]].

## 14. Seguridad empresarial

La plataforma maneja datos de negocio de multiples empresas, credenciales de
integraciones (Slack, WhatsApp/Evolution, SendGrid, Azure, AWS) y permisos por
tenant. Debe existir: JWT con `tenant_id` + refresh tokens, MFA para PlatformAdmin,
cifrado de secretos (DataProtection / Key Vault), HTTPS obligatorio, auditoria
inmutable (`AdminAuditLog`), proteccion CSRF/XSS, rate limiting y validacion de
webhooks. **El error #1 heredado que se corrige es el SQL concatenado** (inyeccion
sistemica en el origen): en el destino todo pasa por EF Core parametrizado. La
segregacion multi-tenant (seccion 4) es la defensa principal contra exposicion
cruzada. Detalle en [[Gestion de Empresas - Admin multi-tenant]].

## 15. DevOps y contenedores

Toda la solucion se ejecuta dockerizada para portabilidad. Compose inicial con
api, postgres (o sqlserver), redis y rabbitmq. Estrategia hibrida progresiva: VPS
para servicios persistentes + Railway para frontend/API. CI corre tests en AMBOS
motores (Postgres y SQL Server) por cada PR. El origen se despliega hoy por Web
Deploy a IIS/Contabo — ver [[Deploy IIS y perfiles de publicacion]] para el estado
actual y la ruta al pipeline destino.

## 16. Observabilidad y monitoreo

Logs centralizados (Serilog), metricas, trazabilidad distribuida (OpenTelemetry),
health checks de la BD (ambos motores), Redis, RabbitMQ y servicios internos.
Metricas de dominio:

```txt
tasks_created_total
tasks_completed_total
workflow_instances_active
workflow_stuck_rate          (flujos trabados / total)
form_submissions_total
rule_executions_total
```

El ratio de flujos trabados es un KPI operativo clave: mide cuanto del trabajo
esta bloqueado en un proceso mal modelado.

## 17. Riesgos tecnicos detectados

- **Fuga entre tenants**: un query que salte el filtro global. Mitigacion: filtro
  EF obligatorio + RLS en BD + tests cross-tenant que DEBEN fallar.
- **Flujos en ciclo infinito**: BPMN mal modelado. Mitigacion: limite de
  iteraciones + deteccion de ciclos (heredado del origen).
- **Migracion del EAV**: `FORX_DATA` a jsonb sin perdida. Mitigacion: ETL validado
  campo por campo + tests de round-trip.
- **Operaciones multi-tabla sin transaccion** (error heredado). Mitigacion: todo
  comando multi-entidad en transaccion.
- **Reglas con SQL directo** (modo Execute inseguro). Mitigacion: sandbox +
  whitelist + prohibir concatenacion.
- **Doble edicion concurrente** de una tarea/flujo. Mitigacion: concurrencia
  optimista (rowversion / xmin).
- **Zonas horarias**: fechas de tareas mal calculadas. Mitigacion: almacenar TZ
  del tenant y normalizar a UTC.

## 18. Entorno del sistema ORIGEN (referencia de migracion)

> Lo siguiente describe el sistema legacy que se esta migrando. Se conserva como
> referencia para el ETL y la comparacion origen -> destino.

Sistema **ECOREX v1.0** (c) 2025 sobre plataforma `GestionMovil` (Bitcode),
ASP.NET WebForms + VB.NET (.NET 4.8.1), Master `MasterFormsII`. Multi-tenant por
columna `SUCURSAL`. Tenant demo: **SKY SYSTEM**.

### Acceso de desarrollo (origen)

| Campo | Valor |
|---|---|
| URL local | `http://localhost:47640/GestionMovil/Formularios/Modulos/Login/LoginBitcode.aspx` |
| URL produccion | `https://app.bitcode.com.co/` |
| Usuario demo | `1048064705` (interno `GUADALUPE.LA`) |
| Clave | temporal `12345` (pide cambio al login) |
| Post-login | `/GestionMovil/Formularios/modulos/Dashboard/das_module.aspx` |

### Base de datos `db3dev` (origen)

- **Servidor**: `sql.bitcode.com.co,44566` (SQL Server 2022 Developer)
- **Catalogo**: `db3dev` — la cadena completa vive en scratchpad de sesion, NUNCA en el vault
- **Volumetria**: 946 tablas. Familias: `TAR_*` (tareas + 5 `TAR_WORKFLOW_*`),
  `DOC_*`/`DOCU_*` (documentos/documental), `ENCUESTAS_*` + `FORX_DATA*` (formularios),
  `GEN_*` (usuarios, roles, dependencias, parametros, tokens), `ADM_*` (modulos, controles),
  `CONTROL_REGLAS*` (motor de reglas), `SUCURSAL*` (tenants).

### Shell y branding (origen)

- Master `MasterFormsII.Master`, topbar "SKY SYSTEM", footer `ECOREX v1.0 (c) 2025`.
- Frontend Bootstrap 4 + Metronic/KT (`Frontend4/`), FontAwesome 4.7, Toastr, tema Light/Dark.
- Componentes del shell: `sideNavBar` (menu por roles), `navBarTop`, `topBar` (por pagina).
- Secciones de menu: OPERACIONES, NEGOCIO, AUTOMATIZACION (Flujos, Formularios,
  Analitica), Agentes IA, SISTEMA (General, Desarrollo: Reglas/SQL/Controles/Reportes...).

El aspecto DESTINO que reemplaza a este shell legacy esta en
[[00 - Prototipo Final ECOREX]].

## Relacion con otras notas

- [[00 - Prototipo Final ECOREX]] — aspecto y navegacion definitivos
- [[Gestion de Empresas - Admin multi-tenant]] — multi-tenant real y correccion de errores
- [[Puntos ciegos y motores transversales]] — motores transversales del origen
- [[HOJA DE RUTA DESARROLLO]] — fases de la migracion
- [[INVENTARIO GENERAL]] — todos los modulos del sistema

---

*Vision del sistema destino. Actualizar a medida que se implementen los servicios
y se cierre el ETL de migracion.*
