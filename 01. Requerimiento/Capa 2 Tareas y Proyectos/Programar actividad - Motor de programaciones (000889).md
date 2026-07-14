---
tipo: spec-modulo
capa: Capa 2 - Tareas y Proyectos
modulo_codigo: "000889"
modulo_nombre: Programar actividad
grupo_menu: Mis Procesos / TAREAS
tabla_principal_origen: PROG_ACTIVIDADES (+ PROG_ACTIVIDADES_R)
consecutivo_origen: "PAC"
motor_destino: ScheduledJobEngine (worker) + ScheduledJobService
stack_destino: .NET 10 / ASP.NET Core / EF Core 10 / Blazor / worker (hosted service) / SignalR + canales
estado: spec destino .NET 10 + ingenieria inversa del origen (stub incompleto) como referencia ETL
prototipo: 01. Requerimiento/Prototipo/ECOREX.dc.html (pantalla isProgramar, aprox. lineas 2532-2616)
---

# Programar actividad - Motor de programaciones (000889)

> Spec del sistema DESTINO para el modulo **Programar actividad** de ECOREX sobre .NET 10.
> Describe QUE se construye (un **programador/scheduler** que dispara **notificaciones** o
> **crea actividades** por reglas de recurrencia y las entrega por canales) y conserva al
> final el analisis del ORIGEN (`adm_programador.aspx` + `PROG_ACTIVIDADES*`), que **nunca
> llego a completarse**, como plano del ETL. El aspecto es el del
> [[Visión y entorno|Prototipo Final ECOREX]] (pantalla `isProgramar`); enlaza con
> [[00 - INDICE y estado actual vs objetivo|Modulo de Tareas]] (alta de actividad) y con los
> Conceptos de actividad.

## D1. Que es en el destino

El modulo permite **programar en el tiempo** dos cosas, sin intervencion manual:

1. **Notificaciones / alertas** recurrentes (ej. "Recordatorio pago proveedores", mensual dia 5).
2. **Creacion automatica de actividades** (ej. "Visita tecnica preventiva", bimestral), que
   entran al sistema de Tareas **consumiendo un concepto** (categoria / subcategoria), igual
   que si un humano las creara.

Cada programacion lleva **uno o varios conjuntos de reglas de recurrencia** y se entrega por
**uno o varios canales** (Correo, WhatsApp, Slack, in-app). Es el equivalente a un "cron de
negocio" gobernado, multi-tenant y con bitacora de ejecuciones.

## D2. Los dos tipos (que se programa)

El modal pregunta primero **"Que deseas programar?"** con dos tipos (campo `TIPO_ACCION` en el
origen):

| Tipo | Que hace al dispararse | Datos extra |
|---|---|---|
| **Notificacion** | Envia un mensaje/alerta por los canales elegidos | (solo nombre + reglas + canales) |
| **Actividad** | **Crea una tarea** consumiendo el concepto elegido (arranca su flujo/formulario/tablero y asigna por cargo) | Categoria + Sub-categoria del concepto |

Para el tipo Actividad, la creacion **reutiliza el camino de alta del modulo de Tareas** (el
puente Concepto->Tarea ya construido): no se duplica logica, el scheduler solo "dispara" el
alta con la categoria/subcategoria configuradas. Ver
[[01 - Arquitectura, decisiones y consumo|Modulo de Tareas: como el alta consume el concepto]].

## D3. Modelo de datos destino

El origen usa cabecera + reglas 1:N (`PROG_ACTIVIDADES` / `PROG_ACTIVIDADES_R`). El destino lo
formaliza tenant-scoped y agrega lo que faltaba (canales explicitos y **bitacora de ejecucion**):

```txt
scheduled_job            (cabecera: tenant_id, code, name, type {Notification|Activity},
                          priority, status {Active|Paused}, created_by, created_at)
   -> responsibles       (RESPONSABLES del origen: lista de usuarios destinatarios/encargados)
   -> area / entidad_id  (AREA del origen -> Entidad Sede/Area del tenant)
   -> categoria_id / subcategoria_id  (solo type=Activity: el concepto que se disparara)

scheduled_job_rule       (1:N recurrencia: frequency {Once|Daily|Weekly|Monthly|SpecificDate},
                          interval_num + interval_unit, weekdays, month_ordinal + month_weekday
                          (o day_of_month), repeat_intraday {every_n_hours, from, to} o at_time,
                          valid_from, valid_to (null = sin fin), description)

scheduled_job_channel    (N canales: Email | WhatsApp | Slack | InApp)

scheduled_job_run        (BITACORA DE EJECUCION -> lo que el origen dejo como placeholder:
                          job_id, fired_at, result {Ok|Error|Skipped}, detail, created_entity_ref)
```

**Decision clave:** `scheduled_job_run` es la pieza que el origen nunca integro (ver O4). Es
lo que habilita los KPIs "ejecutados hoy" / "errores" y la idempotencia del worker.

## D4. Motor de recurrencia (la regla)

Una programacion tiene **N reglas** (el prototipo lo llama "CONJUNTOS DE REGLAS DE EJECUCION");
cada regla describe cuando dispara:

- **Frecuencia**: Una vez | Diaria | Semanal | Mensual | Fecha especifica.
- **Intervalo**: "cada N {dias/semanas/meses}" (interval_num + interval_unit).
- **Semanal**: seleccion de dias (L M M J V S D).
- **Mensual**: "el [primer/segundo/.../ultimo] [dia de semana]" o dia del mes.
- **Repeticion intradia** (opcional): "cada N horas, de HH:MM a HH:MM" o "a las HH:MM".
- **Vigencia**: desde / hasta (sin fin si vacio).

El origen tenia un modelo mas pobre (`ReglaProgramacion`: IntervaloNum, IntervaloTipo=DIAS,
DiasEjecucion, HoraInicio); el destino lo enriquece a las 5 frecuencias del prototipo. El
calculo de la **proxima ejecucion** debe ser **timezone-aware por tenant** (el origen ya
importaba NodaTime en `TaskProgrammer`, senal de esa intencion).

## D5. El runner (worker) destino

Un **background worker** (hosted service / job scheduler tipo Hangfire o Quartz) que, por cada
tenant y de forma continua:

1. Calcula que reglas deben dispararse ahora (proxima ejecucion vencida), timezone-aware.
2. Ejecuta el job en el **contexto del tenant** (ambient TenantId) dentro de una transaccion:
   - **Notificacion** -> Notification Service (SignalR + Correo/WhatsApp/Slack por los canales).
   - **Actividad** -> crea la `TaskItem` consumiendo el concepto (mismo path del modulo Tareas).
3. Escribe `scheduled_job_run` (Ok/Error/Skipped) para bitacora, KPIs e **idempotencia** (no
   disparar dos veces la misma ventana; reintento controlado; dead-letter en error).

Reglas del worker: idempotente (una ventana = un disparo), reintentable, aislado por tenant, y
observable (la bitacora es la fuente de "ejecutados hoy" y "errores").

## D6. Reutilizacion (no se construye de cero)

| Necesidad | Se reutiliza |
|---|---|
| Crear la actividad al disparar | Alta de Tareas que consume el concepto (puente Concepto->Tarea) |
| Enviar por canales | Notification Service + integraciones (SendGrid, WhatsApp HSM, Slack) |
| Numeracion de la programacion | Consecutivos por tenant (`TenantSequence`/`ISequenceService`; consecutivo `PAC` en el origen) |
| Categoria/Subcategoria (tipo Actividad) | Conceptos de actividad (`ActividadCategoria`/`ActividadSubcategoria`) |
| Area / Empresa | Entidad (Sede/Area) de Configuracion de la entidad |
| Responsables | Usuarios del tenant |

## D7. UI (prototipo)

Pantalla `isProgramar` del [[Visión y entorno|Prototipo Final ECOREX]]
(`ECOREX.dc.html`, aprox. lineas 2532-2616):

- **Lista**: header "MODULO 000889 - TAREAS / Programar actividad", boton **Nueva
  programacion**, tabla con columnas **TIPO | NOMBRE | REGLA | CANALES | ESTADO** + editar.
- **Modal "Nueva programacion"**:
  - "Que deseas programar?" (tabs Notificacion / Actividad).
  - Nombre.
  - Si es Actividad: selects **Categoria** + **Sub-categoria** (del concepto).
  - "Conjuntos de reglas de ejecucion" (agregar N reglas; cada una con frecuencia, intervalo,
    dias/ordinal, repeticion intradia y vigencia -> D4).
  - "Canales de transmision" (chips: Correo, WhatsApp, Slack, ...).
  - Guardar programacion.
- Ejemplos del prototipo: *Recordatorio pago proveedores* (Notificacion, Mensual dia 5, Correo);
  *Visita tecnica preventiva* (Actividad, Operaciones/Visita tecnica, Bimestral, WhatsApp+Correo);
  *Alerta cierre de mes* (Notificacion, Fecha especifica 30/07, Slack).

## D8. Multi-tenant y correcciones vs origen

- **Multi-tenant real**: `TenantId` + `HasQueryFilter` + RLS (el origen filtra por `SUCURSAL`
  concatenado en SQL). El worker dispara en el contexto del tenant dueno del job.
- **Cero SQL concatenado**: EF Core parametrizado (el origen arma todo el SQL por concatenacion
  de `SUCURSAL`/`REG` -> SQLi masiva; ver O5).
- **Bitacora de ejecucion**: `scheduled_job_run` (el origen la dejo como placeholder, O4).
- **Idempotencia y reintento** en el worker (el origen no tenia runner terminado).

---

# ORIGEN (referencia de migracion) - adm_programador (incompleto)

> Analisis del modulo legacy que se estaba construyendo y **nunca se completo**. Se conserva
> como reglas de negocio a preservar y plano del ETL. Rutas reales:
> `Bootstrap/Formularios/Modulos/Utilidades/adm_programador.aspx` y los controles/clases del
> modulo Tareas.

## O1. Ubicacion y estado (era un stub)

`adm_programador.aspx` (carpeta Utilidades) es una **cascara**: su `Page_Load` esta copiado de
otro modulo ("TIPO DE PROVEEDORES", Modulo `000340`, `Home/travel/Tipos Proveedores`,
`manejo_tabla = TRA_TIPOSPRO`), y los handlers `topBar_Save/Delete/Clear` estan **vacios**. La
funcionalidad real vive en los controles embebidos:

- `Tareas/controles/ctrListaProgramaciones.ascx` -> lista ("Gestiona tareas recurrentes, backups
  y recordatorios de soporte").
- `Tareas/controles/ctrProgramador.ascx` -> el formulario de alta (evento `ProgramacionGuardada`).
- `Tareas/clases/cl_programacionActividades.vb` -> acceso a datos (CRUD + consultas).
- `Funciones/UtilidadesVarias/TaskProgrammer.vb` (+ `Servicios/Bitcode/sweb_taskProgramer.asmx`)
  -> intento de runner (importa **NodaTime**), sin dispatch de canales terminado.

## O2. Tablas y consecutivo (plano del ETL)

| Tabla origen | Rol | Migra a |
|---|---|---|
| `PROG_ACTIVIDADES` | Cabecera: `REG, NOMBRE, TIPO_ACCION, PRIORIDAD, RESPONSABLES, AREA, CATEGORIA, SUBCATEGORIA, ESTADO, FECHA_CREACION, USUARIO_CREACION, SUCURSAL` | `scheduled_job` |
| `PROG_ACTIVIDADES_R` | Reglas de periodicidad 1:N: `ID, ID_ACTIVIDAD, HORA_INICIO, INTERVALO_NUM, INTERVALO_TIPO, ...` | `scheduled_job_rule` |

Consecutivo: **`PAC`** (Programacion de ACtividades), via `Funciones.tipdoc.Consecutivo`.
Multi-tenant: columna **`SUCURSAL`**. Estados observados: `ACTIVO` (hay `ContarActivos`).

Modelo de regla (`ReglaProgramacion`, serializable en ViewState): `IntervaloNum`,
`IntervaloTipo` (default `DIAS`), `DiasEjecucion` (`"Lun,Mar,Mie,Jue,Vie;Sab;Dom"`),
`HoraInicio` (`08:00`), `Descripcion`. Mas pobre que el prototipo destino (ver D4).

## O3. Componentes del origen

| Componente | Rol | Sucesor destino |
|---|---|---|
| `adm_programador.aspx` | Cascara/host (stub) | pagina Blazor del modulo |
| `ctrProgramador.ascx` | Form de alta (reglas + responsables) | modal "Nueva programacion" |
| `ctrListaProgramaciones.ascx` | Lista de programaciones | bandeja/lista |
| `cl_programacionActividades.vb` | DAL (CRUD + `CargarLista/Detalle/Reglas`, `ContarActivos`) | `ScheduledJobService` |
| `TaskProgrammer.vb` + `sweb_taskProgramer.asmx` | Intento de runner (NodaTime) | `ScheduledJobEngine` (worker) |

## O4. Lo que quedo incompleto (por eso "nunca se completo")

- **Sin bitacora de ejecucion**: `ContarEjecutadosHoy()` y `ContarErrores()` son **placeholders
  que retornan 0** ("hasta integrar log de ejecuciones/errores"). No hay `scheduled_job_run`.
- **Runner sin cerrar**: `TaskProgrammer`/`sweb_taskProgramer.asmx` no tienen el **dispatch de
  canales** (Correo/WhatsApp/Slack) ni el bucle de ejecucion terminado.
- **Host stub**: la `.aspx` nunca se cablo de verdad (Page_Load y handlers heredados/vacios).

## O5. Deudas de seguridad del origen (NO heredar)

- **SQLi masiva**: todo el DAL (`cl_programacionActividades`) arma SQL por **concatenacion** de
  `SUCURSAL`, `REG`, `ID_ACTIVIDAD` -> el destino usa EF Core parametrizado.
- **Multi-tenant fragil** por `SUCURSAL` en string -> `TenantId` + RLS.

## Notas relacionadas

- [[00 - INDICE y estado actual vs objetivo]] - modulo de Tareas (alta que consume el concepto)
- [[01 - Arquitectura, decisiones y consumo]] - el puente Concepto->Tarea que reutiliza el tipo Actividad
- [[AdmWorkflow - Motor de flujo de la tarea]] - flujo que puede arrancar la actividad creada
- [[Puntos ciegos y motores transversales]] - Notification Service, SendGrid, canales
- [[Visión y entorno|Prototipo Final ECOREX]] - aspecto (pantalla isProgramar)

---

# Estado de construccion (worktree tareasprogramadas)

## Ola P1 - HECHO (2026-07-14): dominio + CRUD + UI milimetrica

Persistencia + pantalla `isProgramar` fiel al prototipo. El disparo real (worker + bitacora) es P2.

**Entidades (tenant-scoped, `Ecorex.Domain/Entities`):** `ScheduledJob` (cabecera, consecutivo PAC,
concurrencia optimista Version), `ScheduledJobRule` (recurrencia 1:N), `ScheduledJobChannel` (N canales),
`ScheduledJobRun` (bitacora, se llena en P2). Enums (texto): `ScheduledJobType {Notification|Activity}`,
`ScheduledJobStatus {Active|Paused}`, `ScheduledJobPriority {Low|Normal|High}`, `ScheduledJobFrequency
{Once|Daily|Weekly|Monthly}`, `ScheduledJobChannelType {Email|WhatsApp|Slack|Sms}`, `ScheduledJobRunResult
{Ok|Error|Skipped}`.

**Servicio:** `Ecorex.Application/Scheduling/ScheduledJobService` (EF parametrizado; NO el SQL concatenado
del legacy). List/Get/Save(crear-actualizar con reglas+canales; PAC via ISequenceService)/ToggleStatus/
Delete + catalogo de Conceptos (000270) para los selects Categoria/Sub-categoria.

**UI:** pagina `/programar-actividad` (`ProgramarActividad.razor` + `.razor.css`). Nodo de menu 000889
apuntado a la pagina real (antes stub `modulo/programar-actividad`). Visible para Owner/Admin; el filtro
por permisos lo oculta a roles sin acceso (por diseno, ADR-0033).

### Esquema para PROD (migracion DUAL `AddScheduledJobs`, aplicada SOLO a la BD local `ecorex_forms`)

> La sesion principal debe aplicar esta migracion a prod al integrar la rama (build-from-git migra al
> arrancar). El worktree es el DUENO UNICO de la migracion; no regenerar. Reutiliza NADA existente: son
> 4 tablas nuevas, sin FK a otras tablas de negocio (category_id/subcategory_id/area_entity_id son
> referencias sueltas en P1; se endurecen a FK en P3).

- **`scheduled_jobs`**: id, tenant_id, code(varchar20 UNIQUE por tenant), name(varchar200),
  type/status/priority (varchar40 texto), area_entity_id/category_id/subcategory_id (uuid null),
  **assignee_tenant_user_id (uuid null)** = encargado OPCIONAL,
  version(bigint, concurrencia), + auditoria. Indices: (tenant_id, code) UNIQUE, (tenant_id, status).
  > `assignee_tenant_user_id` llega en la migracion DUAL **aditiva** `AddScheduledJobAssignee` (2a migracion
  > de P1). Si tu prod ya aplico `AddScheduledJobs`, esta se aplica encima sin tocar datos.
- **`scheduled_job_rules`**: id, tenant_id, job_id(FK scheduled_jobs, CASCADE), sort_order,
  frequency(varchar40), interval_num(int def 1), weekdays(varchar40 csv), month_ordinal/month_weekday
  (varchar20), day_of_month(int null), at_time(varchar8 "HH:mm"), repeat_intraday(bool),
  repeat_every_hours(int null), repeat_from/repeat_to(varchar8), valid_from/valid_to(date null),
  description(varchar300), next_run_at(timestamptz null; lo llena P2), + auditoria. Indices:
  (job_id, sort_order), (next_run_at).
- **`scheduled_job_channels`**: id, tenant_id, job_id(FK CASCADE), channel(varchar40), + auditoria.
  Indice (job_id, channel) UNIQUE.
- **`scheduled_job_runs`**: id, tenant_id, job_id(FK RESTRICT - la bitacora sobrevive), rule_id(uuid null),
  fired_at(timestamptz), result(varchar40), detail(varchar600), created_entity_ref(varchar100), + auditoria.
  Indice (tenant_id, job_id, fired_at). SIN uso en P1 (la escribe el worker en P2).

### REGLA DE DOMINIO (verificada en codigo, 2026-07-14): tarea == actividad == TaskItem

"Tarea" y "actividad" son LA MISMA entidad: **`TaskItem`**, exactamente lo que produce el **wizard de 4
pasos** (`TaskWizard.razor`). El wizard solo RECOLECTA datos y llama a
`ITaskItemService.CreateAsync(CreateTaskItemRequest{...}, actor, actorName)`.

El campo clave es **`SubcategoriaId`** (el concepto, 000270): dentro de `CreateAsync` dispara el puente
Concepto->Tarea = resuelve **TituloAuto** (tokens @cliente), coloca en el **tablero del concepto**
(`TaskBoardId`), arranca el **flujo** (IniciaModulo, IWorkflowEngine) y notifica a los **destinatarios del
concepto**. **NO auto-asigna por cargo**: `AssigneeTenantUserId` viene del request; si va null, la tarea
nace **Pending / sin asignar**.

**Consecuencia para P3:** el tipo Activity del scheduler NO duplica nada: llama a ese mismo `CreateAsync`
con el `SubcategoriaId` de la programacion, pasando `AssigneeTenantUserId = job.AssigneeTenantUserId`
(encargado opcional, ver abajo) y, si el concepto no define TituloAuto, `Title = job.Name` (fallback).

### Encargado OPCIONAL de la programacion (decision del usuario, 2026-07-14)

Se agrego `scheduled_jobs.assignee_tenant_user_id` (nullable) + select "Encargado (opcional)" en el modal,
con el MISMO origen que el wizard (`ITenantUserService` + `TaskUi.UserLabel`). Semantica por tipo:
- **Activity**: se pasa tal cual a `CreateTaskItemRequest.AssigneeTenantUserId`. Vacio -> la actividad nace
  sin asignar (Pendiente) en el tablero del concepto (comportamiento por defecto de TaskItemService).
- **Notification**: es el destinatario in-app del aviso (lo consumira P2).

Verificado E2E: PAC-000003 (Actividad, Operaciones/Visita tecnica) persiste el encargado
`operator@sky-system.local`; editar lo recarga; las programaciones previas quedan "sin asignar".

### Verificado E2E (Chrome, ecorex_forms)
Crear Notificacion (Semanal Lun/Mie, Correo+WhatsApp) -> PAC-000001; crear Actividad (Operaciones/Visita
tecnica, Mensual primer Lunes) -> PAC-000002; editar recarga los datos; enlace de menu visible para Owner.

### Cierre de P1 (2026-07-14): pausar/activar + tests duales + bug de edicion

- **Pausar/activar**: el chip de ESTADO de la fila es un boton que alterna Activa/Pausada
  (`ToggleStatusAsync`). El worker de P2 SOLO debe disparar las **Activas**.
- **Nodo de menu en PROD**: 000889 se agrego a la lista `expected` de `ReconcileMenuNodesAsync`, asi que
  los tenants YA sembrados (prod) se auto-corrigen al arrancar (stub `modulo/...` -> `programar-actividad`,
  State=Ready). No hace falta SQL manual en el deploy.
- **BUG REAL cazado por los tests** (invisible en la prueba manual): al editar, el servicio hacia
  `RemoveRange(hijos)` y ademas vaciaba las **navs** del padre. Vaciar la nav de una relacion con CASCADA
  marca los hijos como huerfanos -> EF emite un SEGUNDO DELETE sobre filas ya borradas -> "affected 0 rows"
  -> `DbUpdateConcurrencyException` espuria. **Ninguna edicion se podia guardar.** Arreglado: el reemplazo
  total opera sobre los DbSet, nunca sobre las navs.
- **Tests de integracion DUALES** (PG + SQL Server), `ScheduledJobsTests.cs`, **10/10 verde**: consecutivo
  PAC sin duplicados, validaciones, reemplazo total al editar (sin huerfanos, el codigo no cambia), tipo
  Activity conserva concepto + encargado opcional, toggle de estado, y el **BLOQUEANTE de aislamiento
  cross-tenant** (B no lista/lee/edita/borra/pausa lo de A; el consecutivo de B arranca en su PAC-000001).

**P1 CERRADA.**

## Ola P2 - HECHA (2026-07-14): motor de recurrencia + worker + bitacora + KPIs

Las programaciones ya DISPARAN solas. Cierra las 3 piezas que el origen nunca completo (O4): el bucle de
ejecucion, la bitacora y los contadores (ContarEjecutadosHoy/ContarErrores devolvian 0 fijo).

- **Motor de recurrencia** (`ScheduledJobRecurrence`, PURO): proxima ejecucion calculada en la **zona del
  tenant** (regla 9) y devuelta en UTC -> "los lunes a las 08:00" = 08:00 EN EL TENANT. Cubre las 4
  frecuencias del prototipo, intervalos "cada N", dias marcados, ordinal mensual (Primer..Ultimo x
  Lunes..Domingo o "dia"), repeticion intradia y vigencia. Con tope de busqueda (no se cuelga).
- **Worker** (`ScheduledJobWorker`): hosted service **dentro de Ecorex.SuperAdmin**, NO en Ecorex.Workers.
  > OJO (hallazgo): el compose de prod (`deploy/docker-prod`) solo levanta el servicio `ecorex-app` (=
  > SuperAdmin). Un worker en `Ecorex.Workers` **nunca correria en produccion**.
- **Dispatcher**: barrido cross-tenant (unico `IgnoreQueryFilters`, devuelve solo ids de tenant) +
  ejecucion acotada con `AmbientTenantContext.Begin`. Dispara solo las **Activas**; escribe
  `scheduled_job_run` con la **VENTANA** como `fired_at` (no el "ahora") y avanza `NextRunAt` desde ella.
  Notification -> notificacion in-app al encargado. Activity -> **Skipped hasta P3**.
- **Idempotencia**: indice UNICO `(tenant_id, job_id, rule_id, fired_at)` -> una ventana = un disparo,
  aunque corran dos instancias del worker.
- **Auto-reparacion**: reglas sin `NextRunAt` quedarian MUERTAS; el barrido las visita y las reprograma.

### Esquema P2 para PROD (migracion DUAL `AddSchedulerEngine`, ADITIVA)
- `tenants.time_zone_id` (varchar60, nullable, IANA; fallback `America/Bogota`).
- Indice UNICO `(tenant_id, job_id, rule_id, fired_at)` en `scheduled_job_runs`.

### Verificado
34 tests verde: 12 unitarios del motor + 22 de integracion DUAL (dispara, ignora Pausadas, idempotente,
Activity=Skipped, barrido de plataforma, auto-reparacion, aislamiento cross-tenant). **En vivo**: se forzo
una ventana vencida -> el worker disparo solo, dejo bitacora Ok, entrego la notificacion in-app y avanzo
`next_run_at` a las 08:00 de Bogota. UI: KPIs "1 ejecutados hoy / 0 errores / 3 activas" y proximas
correctas (semanal Lun -> 20/07; mensual primer Lunes -> 03/08; Lun+Mie -> 15/07).

**P2 CERRADA.**

## Ola P3 - HECHA (2026-07-14): el tipo Actividad crea la TAREA real

Aplica la regla de dominio: **tarea == actividad == TaskItem**, la MISMA que produce el wizard de 4 pasos.
El motor **NO duplica nada**: llama al MISMO `ITaskItemService.CreateAsync` con el `SubcategoriaId` de la
programacion, que es quien dispara el puente Concepto->Tarea dentro del servicio (titulo automatico,
tablero del concepto, arranque del flujo y aviso a los destinatarios del concepto).

Al dispararse una programacion de tipo Activity, el dispatcher:
- **Titulo**: manda el `TituloAuto` del concepto (tokens tipo @cliente); si el concepto no lo define, cae
  al **nombre de la programacion** (sin esto `TaskItemService` rechazaria la tarea sin titulo).
- **Encargado** opcional -> `AssigneeTenantUserId`. Vacio = la actividad nace **Pendiente / sin asignar**
  (comportamiento por defecto de TaskItemService: NO auto-asigna por cargo).
- **Actor**: quien configuro la programacion (`job.CreatedBy`), con actorName "Motor de programaciones
  (PAC-xxxxxx)" para dejar rastro en la auditoria.
- **Trazabilidad**: el NUMERO de la tarea creada queda en `scheduled_job_runs.created_entity_ref`.
- Sin concepto configurado -> la ejecucion queda como **Error** en la bitacora (no revienta el motor).

### Verificado
26/26 tests DUAL (3 nuevos: crea la tarea con concepto+encargado y registra su numero; TituloAuto tiene
precedencia; sin concepto = Error). **En vivo**: se forzo una ventana vencida de PAC-000003 (Actividad,
concepto Operaciones/Visita tecnica, encargado operator@) -> el worker disparo solo y nacio la tarea REAL
**T00215** (Active, con su concepto y encargado); la bitacora la registra y la UI muestra "2 ejecutados hoy".

**P3 CERRADA.** Sin cambios de esquema.

## Olas siguientes (pendientes)
- **P2**: motor de recurrencia (proxima ejecucion timezone-aware) + worker (hosted service) que dispara
  Notificacion por canales + escribe `scheduled_job_runs` + KPIs (ejecutados hoy / errores), idempotente.
- **P3**: tipo Actividad crea la `TaskItem` consumiendo el concepto (puente Concepto->Tarea); endurecer
  FK category/subcategory/area.
- **P4**: canales reales (Notification Service: in-app SignalR + Correo; WhatsApp/Slack) + reintento/dead-letter.
