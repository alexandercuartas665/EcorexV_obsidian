---
tipo: spec-arquitectura
proyecto: Modulo de Tareas
proposito: Decisiones de arquitectura (unificacion de tipos, empresa/area, menu dinamico) y como el alta de tarea CONSUME el concepto para arrancar flujo, formulario y asignacion por cargo.
---

# 01 - Arquitectura, decisiones y consumo (el puente)

## 1. Decisiones de arquitectura (a confirmar con el usuario)

### D1 - Unificar `ActivityType` vs `ActividadSubcategoria` (LA decision madre)
Hoy conviven dos "tipos":
- `ActivityType` (Category=texto + Name): lo que usa `TaskItem.ActivityTypeId` hoy.
- `ActividadSubcategoria` (Concepto): el modelo rico (flujo, formulario, tablero+columna, cargos,
  terceros, flags). Es el que describe el usuario y el que debe gobernar la creacion.

Opciones:
- **(a) Pivotar a Conceptos (recomendado).** `TaskItem` referencia `ActividadSubcategoriaId`
  (y por navegacion su `CategoriaId`). `ActivityType` se deprecia o se mantiene solo para tareas
  legacy. Migracion EF: agregar `SubcategoriaId` a `TaskItem`, backfill, y apuntar el wizard a
  Conceptos. Es lo mas coherente con el dictado (tablero/flujo/cargo vienen del concepto).
- (b) Puente sin migrar: mapear cada `ActivityType` a una `ActividadSubcategoria`. Mas fragil
  (dos verdades) y arrastra el modelo pobre.

Recomendacion: **(a)**. El resto de este doc asume (a).

### D2 - Fuente de "Empresa / Area"  (DECISION DEL USUARIO: modulo de config de la empresa)
El prototipo pide un selector "Empresa / Area". **Decision del usuario:** el dato de Empresa/Area
debe salir del **modulo de configuracion de la EMPRESA misma** (la configuracion del tenant/empresa,
no del organigrama). Esto se marca como un **PREREQUISITO a resolver ANTES** de tocar el modal (ver
[[03 - Plan por olas y preguntas abiertas]] seccion "Prerequisitos - checklist").

Realidad del codigo hoy (a reconciliar):
- La ruta "Configuracion de entidad" (000615, `/configuracion`) resuelve a `Cuenta.razor`, que es
  config del **tenant** (plan/facturas/API); **NO** define areas.
- Las **areas** hoy existen solo en el organigrama (`OrgUnit` Kind=Area/Classifier=Dependencia,
  modulo Dependencias 000850).
- "Empresa" como entidad = el **Tenant**.

Por tanto el prerequisito es: **definir/exponer las Areas (y, si aplica, sedes/empresas) dentro del
modulo de configuracion de la empresa**, de forma que el selector "Empresa/Area" del modal las
consuma. Opciones de implementacion del prerequisito (a decidir al resolverlo):
- (a) Que el modulo de config de empresa **administre y exponga las areas** (nueva entidad `Area`
  a nivel tenant, o reusando `OrgUnit` Kind=Area presentado desde ese modulo).
- (b) Unificar: "Area" del negocio = `OrgUnit` Kind=Area, pero **administrado/visible desde la
  config de la empresa** (no solo desde Dependencias).

El resto del spec asume que, resuelto el prerequisito, existe una fuente de "Areas del tenant" que
el modal consume; hasta entonces, el paso 1 del wizard queda bloqueado en ese selector.

### D3 - Menu "Mis Procesos": dinamico desde Conceptos
El menu es data-driven (`menu_nodes`), y "Mis Procesos" hoy es estatico. Objetivo: que
**Mis Procesos** liste dinamicamente **categoria -> subcategoria (concepto tipo proceso)** y al
hacer clic cargue el **tablero del concepto + el modal**. Opciones de implementacion:
- **(a) Nodo especial "grupo dinamico" (recomendado).** En el editor de menu se marca UN grupo
  como "Muestra actividades tipo proceso" (bandera en `MenuNode`, `Kind=DynamicProcesos`). Al
  render, `NavMenu` expande ese nodo con las categorias/subcategorias de Conceptos que tengan
  flujo (via `IActividadCatalogoService`), en vez de hijos estaticos. Cumple el pedido del
  usuario ("arreglar el editor de menu para indicar que grupo muestra las actividades proceso").
- (b) Generar/seedear nodos por cada categoria/subcategoria (se desincroniza al cambiar conceptos).

Recomendacion: **(a)** grupo dinamico. "Actividad tipo proceso" = subcategoria con
`WorkflowDefinitionId != null` (no hay flag `esProceso`; se infiere del flujo asignado).

### D4 - Arranque form-first al crear
Si la subcategoria tiene `IniciaModulo=true` (y `FormDefinitionId`), el alta debe abrir el
**formulario** como parte del wizard (paso "Formulario"), reutilizando `DynamicFormRenderer`
(modo Fill) + `FormResponseService`. Para el iniciador, la tarea "arranca" diligenciando datos;
para el resto del equipo es el flujo de aprobaciones. Es viable (motor existe); hay que cablearlo.

## 2. Modelo de datos objetivo (cambios sobre lo existente)

Asumiendo D1(a), D2(a), D3(a):

- `TaskItem`: agregar `SubcategoriaId (Guid?)` FK a `ActividadSubcategoria` (y opcional
  `AreaOrgUnitId (Guid?)` FK a `OrgUnit` para Empresa/Area). Mantener `BoardId/ColumnId`,
  `AssigneeTenantUserId`, `WorkflowInstanceId`, prioridad, estado (ya existen). Migracion EF +
  backfill de las tareas existentes (mapear su `ActivityType` a una subcategoria o dejar null).
- `MenuNode`: agregar soporte para `Kind=DynamicProcesos` (o un flag `ShowsProcessActivities`)
  para el grupo dinamico. Sin romper el editor actual.
- NO se tocan Conceptos, Flujos, Formularios ni Organigrama (ya estan).

## 3. Como el alta de tarea CONSUME el concepto (secuencia)

Al confirmar la creacion de una actividad ligada a una subcategoria, dentro de UNA transaccion
(regla del proyecto: operaciones multi-tabla transaccionales):

```
1. Resolver la subcategoria elegida (con sus FKs y flags) via IActividadCatalogoService.
2. Crear TaskItem:
   - SubcategoriaId, AreaOrgUnitId, AssigneeTenantUserId (encargado), Priority, DueDate,
     Title (o TituloAuto con tokens si aplica), Description (o DetalleAuto),
     BoardId/ColumnId = del concepto (TaskBoardId / primera columna),
     Number = consecutivo por tenant (TenantSequence).
3. Si Subcategoria.WorkflowDefinitionId != null:
   - WorkflowEngine.StartInstance(tenant, taskId, workflowDefinitionId) -> crea WorkflowInstance
     + primer WorkflowStepHistory (IsCurrent, Pending). La asignacion a usuarios la resuelve
     INodeAssigneeResolver/OrgAssigneeTree a partir de la WorkflowNodePolicy (cargo del nodo).
4. Si Subcategoria.IniciaModulo && FormDefinitionId != null:
   - Crear/abrir el FormResponse (draft) del formulario para que el iniciador lo diligencie
     (paso "Formulario" del wizard). Al enviar, FormResponseService completa el paso vinculado.
5. Notificaciones: a los usuarios de Subcategoria.Notificaciones (+ el asignado), via el
   Notification Service (SignalR + email), disparado por evento.
6. Auditoria + commit. Emitir evento SignalR (task.created / board updated) al grupo del tenant.
```

## 4. Asignacion del "Encargado" (cascada del modal)

- Para el selector "Encargado": listar **cargos** (OrgUnit Classifier=Cargo, con sangria por
  Dependencia) primero, y como opcion el **encargado/responsable** de la unidad
  (`OrgUnit.ResponsibleTenantUserId`) o los **Funcionarios** ocupantes. Reusar
  `WorkflowNodePolicyService.ListAssignableUnitsAsync()` + `OrgAssigneeTree` para expandir
  cargo -> personas.
- Deuda tecnica (dictado): hoy no hay flag "encargado principal" por miembro; existe
  `ResponsibleTenantUserId` a nivel de unidad. Si se quiere el indicador por miembro, agregar un
  flag `IsPrincipal` en `OrgUnitMember` (o usar el Responsible existente). Exponerlo en el
  desplegable de encargado. Es un ajuste chico en Dependencias.
- Para actividades-proceso, el "atiende cada paso" NO es el encargado unico: lo resuelve el flujo
  por cargo del nodo (el encargado inicial es quien crea/lidera; el flujo reparte los pasos).

## 5. Los dos caminos, a nivel de datos

- **(A) Proceso**: subcategoria con `WorkflowDefinitionId != null`. Entra por Mis Procesos (D3),
  el modal llega con categoria/subcategoria fijadas, crea TaskItem + StartInstance (+ Formulario
  si `IniciaModulo`). Se ve en el tablero del concepto y en las bandejas de los asignados.
- **(B) Simple**: subcategoria con `WorkflowDefinitionId == null`. Entra por un tablero, el modal
  pide Empresa/Area -> Tipo(categoria) -> Actividad(subcategoria) -> Encargado. Crea TaskItem sin
  instancia de flujo; vive en el tablero.

## 6. Reglas transversales (del proyecto, obligatorias)

- Multi-tenant real (`HasQueryFilter` + RLS); imposible ver/crear cross-tenant.
- DAL dual (PG/SQL Server) via `IEcorexDbContext`; SQL parametrizado.
- Transaccion para crear-tarea + instancia-flujo + formulario + notificacion (todo o nada).
- Soft-delete + auditoria; concurrencia optimista (`Version`).
- Tiempo real por SignalR (tableros/conteos/bandeja).
- Permisos como policies (`[Authorize(Policy=...)]`), no chequeos ad-hoc.
