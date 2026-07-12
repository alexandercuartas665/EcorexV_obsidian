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
  si `IniciaModulo`). Se ve en el tablero del concepto y en el TABLERO de cada asignado (alcance
  "mis pendientes"); el paso se atiende DENTRO del detalle de la tarea (seccion Flujo). NO hay
  bandeja/pagina "Mis pasos" aparte (ADR-0038).
- **(B) Simple**: subcategoria con `WorkflowDefinitionId == null`. Entra por un tablero, el modal
  pide Empresa/Area -> Tipo(categoria) -> Actividad(subcategoria) -> Encargado. Crea TaskItem sin
  instancia de flujo; vive en el tablero.

### 5.1 Historia de usuario detallada (el SISTEMA tambien es actor) -- ADR-0038

> **Contexto visual** (prototipo `ECOREX.dc.html`, seccion "FLUJO DEL PROCESO" ~3478): al abrir una
> tarea-proceso desde el tablero, el modal muestra la **RUTA del flujo** -- un grafo de nodos con el
> paso ACTUAL resaltado, los nodos CERRADOS en verde, QUIEN atiende cada uno (por cargo), badge
> Manual/Automatico, y las compuertas de decision. **Esa ruta ES el resultado vivo de la ejecucion**
> de la tarea. Actores humanos: Solicitante, Jefe de Compras, Comprador. Actor no humano: **ECOREX
> (el sistema)**, que enciende, rutea, avanza y cierra.

**Lo que ve el usuario al abrir la tarjeta de la tarea (la RUTA):**

```
FLUJO DE COMPRAS                         [ A = Automatico   M = Manual ]

   [*] Solicitud de compra       (M - Solicitante)    CERRADO  (nota)
        |
        v
   [>] Aprobacion jefe compras   (M - Jefe Compras)   <=== PASO ACTUAL
        |
        v
      < ?Aprobada? >   (compuerta / decision)
       /            \
     Si              No
      |               |
      v               v
   [ ] Generar OC     [x] Rechazada
       (Comprador)        (Fin)
        |
        v
   [*] Fin (Compra OK)
```

Leyenda: `[*]` inicio/fin  ·  `[>]` paso ACTUAL (resaltado)  ·  `[ ]` pendiente  ·  `[x]` fin
alterno  ·  `< ? >` compuerta  ·  `CERRADO` nodo terminado (verde)  ·  `(nota)` anotacion del
equipo. Tocar un nodo salta a ese paso; su menu (`...`) permite ANOTAR o "Dar por terminado".

**La historia, paso a paso (actor -> accion), leyendo la ruta:**

1. *Solicitante* crea la actividad de compra desde su concepto (Mis Procesos).
2. **Sistema**: en UNA transaccion crea la tarea, arranca la instancia de flujo y ENCIENDE el primer
   nodo real ("Aprobacion jefe de compras") como PASO ACTUAL, ruteado al cargo Aprobador. En la
   ruta, "Solicitud" queda CERRADO y "Aprobacion" resaltado.
3. **Sistema**: resuelve los candidatos del cargo por el organigrama (`INodeAssigneeResolver`) y hace
   que la tarea aparezca en **"mis pendientes" del TABLERO** de cada candidato (Jefe de Compras);
   ademas notifica.
4. *Jefe de Compras* ve la tarjeta en su tablero, la abre y en la RUTA ve su paso resaltado. Lo toma,
   deja una anotacion si hace falta y en la compuerta "?Aprobada?" elige la ruta: Si o No.
5. **Sistema**: registra la decision, AUTO-RESUELVE la compuerta (ADR-0037) por la ruta elegida y
   MUEVE el paso actual: si "Si" -> "Generar OC" (cargo Comprador); si "No" -> "Rechazada" (Fin). El
   nodo atendido queda CERRADO (verde) en la ruta.
6. *Comprador* (si se aprobo) ve la tarea en SU tablero, la abre, atiende "Generar OC" y emite la
   orden de compra.
7. **Sistema**: cierra el ultimo nodo y marca el caso como terminado (tarea Done / instancia
   Completed) sin estancarse; la ruta queda toda en verde hasta "Fin (Compra OK)".

**Donde vive cada cosa (descubrir vs. ejecutar):**

```
   DESCUBRIR                      EJECUTAR
   (el TABLERO)                   (DENTRO de la tarea)

   +--------------------+  abrir  +----------------------------------+
   | Tablero            | ------> | Modal de la tarea                |
   | "mis pendientes"   |         |  +-- RUTA DEL FLUJO (paso actual) |
   |  * tarea A (mi paso)|        |  +-- anotar / cerrar nodo / ruta  |
   |  * tarea B (mi paso)|        |  +-- registro, subtareas, adjuntos|
   +--------------------+         +----------------------------------+
```

**Invariante:** descubrir = el tablero ("mis pendientes", que incluye los pasos ruteados por cargo);
ejecutar = la tarea y su RUTA de flujo. No existe la bandeja "Mis pasos" (ADR-0038).

### 5.2 De donde salen los procesos: el menu "Mis Procesos" (flag `IsProcessGroup`)

El menu "Mis Procesos" NO es una lista fija: se **activa con una marca de seleccion** -- el flag
**`IsProcessGroup`** sobre una **Seccion** del menu (se marca en el editor de menu / ConfiguracionMenu;
la UI le pinta un badge "procesos", `NavMenu.razor` ~102). Al marcarlo, la seccion se vuelve DINAMICA
(Ola 4): en vez de items estaticos, despliega el arbol **Categoria -> Subcategorias TIPO PROCESO**,
filtrando SOLO las subcategorias (conceptos) con `WorkflowDefinitionId != null` y no archivadas
(`NavMenu` ~383-391). Es decir, un **concepto-actividad se vuelve un "proceso" (caso especial)
precisamente por tener un flujo agregado**; los conceptos SIN flujo (actividad simple) no aparecen aqui.

```
Mis Procesos                 <- Seccion con IsProcessGroup = true  (badge "procesos")
  Compras
    Solicitud de compra [proc] -> crear-actividad?sub=<id>  (al crear, DISPARA el flujo)
  Comercial
    Cotizacion          [proc] -> crear-actividad?sub=<id>
```

- Cada hoja lleva el badge `proc` ("Actividad tipo proceso") y enlaza a `crear-actividad?sub=<id>`:
  el modal llega con categoria/subcategoria fijadas y, al guardar, arranca la `WorkflowInstance` (§5.A).
- Si el tenant no tiene ningun concepto con flujo, la seccion muestra: "Sin actividades tipo proceso.
  Vincula un flujo a una subcategoria en Conceptos."
- Resumen de la cadena: **marca `IsProcessGroup` (menu) -> lista los conceptos con `WorkflowDefinitionId`
  (Conceptos) -> crear desde ahi dispara el flujo -> se ejecuta DENTRO de la tarea (RUTA, §5.1)**.

## 6. Reglas transversales (del proyecto, obligatorias)

- Multi-tenant real (`HasQueryFilter` + RLS); imposible ver/crear cross-tenant.
- DAL dual (PG/SQL Server) via `IEcorexDbContext`; SQL parametrizado.
- Transaccion para crear-tarea + instancia-flujo + formulario + notificacion (todo o nada).
- Soft-delete + auditoria; concurrencia optimista (`Version`).
- Tiempo real por SignalR (tableros/conteos). El runtime de flujos se atiende en el detalle de la
  tarea; los pasos pendientes se descubren en el tablero ("mis pendientes"), sin bandeja aparte (ADR-0038).
- Permisos como policies (`[Authorize(Policy=...)]`), no chequeos ad-hoc.
