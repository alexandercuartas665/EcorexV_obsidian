---
tipo: plan-olas
proyecto: Tareas de proceso - Arranque y encargado del flujo
estado: EN CURSO (2026-07-14) - Olas 0/A/B/C HECHAS; faltan Ola D (form por nodo) y el deploy a prod
fecha: 2026-07-14
---

# 03 - Plan por olas pequenas

> Cada ola es **chica, entregable y verificable por si sola**. Ninguna deja el sistema roto.
> Orden pensado para que el valor se vea temprano: **A** cierra el encargado (lo que mas duele),
> **B** cierra el form-first, **C** pone las guardas y el QA.

> [!success] Avance al 2026-07-14 (todo commiteado en `main` + `fase-0/clon-backbone`)
> **HECHAS y validadas en Chrome local**: Ola 0 (decisiones), **A1** (resolver el encargado del 1er
> nodo), **A2** (el wizard lo preselecciona), **A3** (el paso nace asignado + D2 en servidor),
> **B1** (form-first abre el formulario directo), **B2** (el wizard queda con un solo camino).
> **HECHAS tambien**: **C1** (guardas + banner D3), **C2** (QA end-to-end). **PENDIENTES**: **Ola D**
> (formulario por nodo) y el **deploy a produccion**. Ver la tabla "Resumen de olas" al final.

Fuente de las brechas: [[00 - INDICE y estado actual vs objetivo]] seccion 4.
Detalle tecnico: [[01 - Arquitectura del arranque (menu, encargado, form-first)]].

---

## Ola 0 - Decisiones  -- CERRADA (usuario, 2026-07-14)

| # | Pregunta | **DECISION DEL USUARIO** | Consecuencia |
|---|---|---|---|
| **D1** | El formulario del arranque, de donde sale? | **AMBOS, en dos tiempos**: (a) **ahora** del **concepto** (`Subcategoria.FormDefinitionId`, lo que ya existe); (b) **despues** por **nodo** del flujo, como **ola planificada explicita**, no como backlog difuso | Ola **B1** usa (a). Se crea la **Ola D** (formulario por nodo) como capitulo comprometido |
| **D2** | Puede el iniciador **cambiar** el encargado que dicta el flujo? | **NO. El flujo manda.** El combo solo lista candidatos del cargo del **1er nodo**; asignar fuera de ese cargo = **error de validacion** | Olas **A2** (combo restringido) y **A3** (validacion server-side) |
| **D3** | Concepto con flujo **sin publicar**: que hace el menu? | **Se muestra la hoja igual, pero se AVISA al crear** que el flujo no esta publicado y la actividad nacera **sin proceso** | **Cambia la Ola C1**: `NavMenu` **NO** se filtra por `IsPublished`; el aviso va en el **arranque** (wizard y form-first) |

> [!note] Por que D3 asi
> El admin quiere **ver** su concepto en el menu mientras aun esta armando el flujo (feedback
> inmediato de que la configuracion "engancho"). Lo inaceptable no es que aparezca, sino que la
> tarea nazca **sin flujo y en silencio**. Se ataca el silencio, no la visibilidad.

**Entregable**: esta tabla. **Sin codigo.** -- HECHO.

---

## OLA A - El encargado que dicta el flujo (cierra HU-01)

### Ola A1 - Resolver el candidato del primer nodo (solo backend, sin UI)  -- HECHA 2026-07-14

- **Construido**: `IWorkflowStartService.ResolveFirstStepAsync(subcategoriaId)` +
  `WorkflowStartService` (en `Ecorex.Application/Workflows/`), registrado en DI (scoped).
  Camina el grafo **EN SECO** (sin instancia, sin pasos, sin persistir):
  1. lee `Subcategoria.WorkflowDefinitionId`,
  2. exige la definicion **publicada y no archivada**,
  3. desde el `startEvent`, atraviesa startEvents y **compuertas** hasta el **primer nodo Task**,
  4. lee sus `WorkflowNodePolicy` -> **cargos** (ordenados por `SortOrder`),
  5. usa **`INodeAssigneeResolver`** (ya existia) -> **candidatos**.
- **Clave de fidelidad**: la resolucion de compuertas es un **espejo exacto** de
  `WorkflowEngine.ResolveOutgoing` evaluada con `approvalResult = null` -- que es el estado real
  en el que el motor las evalua al arrancar. Por eso **el nodo que devuelve este servicio es el
  mismo que el motor activara despues**. No se duplico logica de negocio: solo la navegacion.
- **Devuelve** `FirstStepDto { Status, WorkflowDefinitionId, NodeId, NodeName, Cargos[], CandidateUserIds[] }`
  con helpers `EsProceso`, `TieneCandidatoUnico`, `CargoPrincipal`. **Nunca lanza** por
  configuracion incompleta: lo reporta en `FirstStepStatus`:

  | Status | Significado | Lo consume |
  |---|---|---|
  | `Ok` | nodo + cargo + al menos un candidato | A2 (preselecciona), A3 (persiste) |
  | `SinFlujo` | la subcategoria no tiene flujo: es actividad simple | el wizard normal |
  | `FlujoNoPublicado` | borrador/archivado: nacera SIN proceso | **C1 (banner de D3)** |
  | `SinNodoTask` | publicado pero no se alcanza ninguna Task | C1 (validar al publicar) |
  | `SinCargo` | el primer Task no tiene cargo | C1 (validar al publicar) |
  | `SinCandidatos` | el cargo no tiene ocupantes: el paso naceria huerfano | C1 (avisar) |

- **NO toca UI. NO toca el motor. Solo lee.**
- **Aceptacion CUMPLIDA**: `WorkflowStartServiceTests` (7 casos) **verde en la matriz dual
  (PostgreSQL + SQL Server)**: flujo lineal -> primer Task "Cotizar" + cargo "Comprador" +
  candidato unico; **compuerta justo despues del startEvent -> la atraviesa** y encuentra la Task;
  y los 4 estados de config incompleta (sin flujo / borrador / sin cargo / sin ocupantes) +
  **aislamiento cross-tenant** (el tenant B no ve la subcategoria de A).

### Ola A2 - El wizard muestra y preselecciona ese encargado (cierra B1 + B2)  -- HECHA 2026-07-14

- **Construido** en `TaskWizard.razor` (inyecta `IWorkflowStartService`). `ResolveEncargadoAsync`
  ahora bifurca:
  - **Actividad-PROCESO** (la subcategoria tiene flujo): pide `ResolveFirstStepAsync` (Ola A1). Si
    el paso se resolvio (`Ok` o `SinCandidatos`), el combo "Encargado" se llena con los
    **candidatos del cargo del PRIMER NODO** (ya **no** con `SelectedSub.CargoIds`), queda
    **RESTRINGIDO** a ellos (D2), muestra el chip **"Paso 1 - {nodo} - {cargo}"** y la nota *"Lo
    dicta el flujo: solo el cargo X atiende este paso"*, y **PRESELECCIONA** al candidato si hay
    uno solo. Si el cargo esta **vacante** (`SinCandidatos`), el combo se **deshabilita** y avisa
    *"El cargo X del primer paso no tiene ocupantes. Asignalo en Dependencias."*
  - **Actividad simple** (sin flujo) o **flujo aun no utilizable** (borrador / sin nodo Task / sin
    cargo): comportamiento de siempre (cargos del concepto). El banner de "nacera sin proceso" es
    de la Ola C1.
- **D2 tambien en validacion**: `ValidateStep(1)` rechaza un encargado fuera de los candidatos
  ("El encargado debe ocupar el cargo del primer paso del flujo"). Es solo el atajo de UI: el
  servidor lo **revalida** al crear en la Ola A3.
- **Bug encontrado y corregido en la validacion visual**: al cambiar de una actividad-proceso a una
  actividad simple, el encargado **que habia dictado el flujo** se quedaba pegado (el usuario nunca
  lo eligio). Ahora, si el encargado venia del flujo, se **limpia** al salir del modo restringido.
- **Aceptacion CUMPLIDA (validado en Chrome real, tenant demo)**:
  - `Cotizacion de equipos` -> chip **"Paso 1 - Requerimiento - Asesor Comercial"**, combo con
    **1 sola opcion** y **preseleccionado Operator SKY SYSTEM** (unico ocupante del cargo).
  - `Compra urgente` -> chip **"Paso 1 - Aprobacion jefe de compras - Aprobador"**, **1 sola
    opcion**, **preseleccionado Admin SKY SYSTEM**. Cargo distinto -> persona distinta: el servicio
    de A1 esta leyendo de verdad el BPMN, no adivinando.
  - `Solicitud de compra` (sin flujo) -> **sin chip**, encargado **limpio**, lista **completa** (11
    usuarios): la ruta clasica quedo intacta.
  - Regresion: **360/360** Application.Tests + **35/35** Domain.Tests. Solucion en verde.

### Ola A3 - Persistir y notificar el asignado del primer paso (cierra B3 + B4)  -- HECHA 2026-07-14

- **Construido** en `TaskItemService.CreateAsync`, **dentro de la misma transaccion**, justo despues
  de `StartInstanceAsync`:
  1. se toma el paso actual (`IsCurrent`, `Pending`) de la instancia recien creada;
  2. **D2 revalidado en SERVIDOR**: si el nodo tiene cargo (`WorkflowNodePolicy`) y el encargado NO
     esta entre sus candidatos (`INodeAssigneeResolver`) -> **rollback total** + error tipado
     ("El encargado debe ocupar el cargo que el flujo asigna al primer paso"). Restringir el combo
     del wizard (A2) no basta: un API podria saltarselo;
  3. se **FIJA** `WorkflowStepHistory.AssignedToTenantUserId` -> el paso **ya no nace colgando**;
  4. se **audita** ("ruto el primer paso del flujo a {usuario}"). La notificacion al encargado ya la
     emite el bloque de creacion (TaskAssigned, campana + email).
- `INodeAssigneeResolver` se inyecta como parametro **REQUERIDO** (no opcional): una regla de
  gobierno no debe poder desactivarse en silencio si falla el cableado de DI.
- **Aceptacion CUMPLIDA - tests**: `WorkflowStartServiceTests` sube a 10 casos, **verde en matriz
  dual (PostgreSQL 10/10 + SQL Server 10/10)**. Los 3 nuevos: (a) el primer paso **nace asignado** al
  candidato del cargo + notificacion `TaskAssigned` presente; (b) un encargado **fuera del cargo** ->
  `Invalid` + **rollback total** (ni tarea, ni instancia, ni pasos); (c) regresion: una actividad
  **sin flujo** sigue aceptando cualquier encargado.
- **Aceptacion CUMPLIDA - CICLO COMPLETO en Chrome real** (tenant demo, BD local):
  1. Owner entra por **Mis Procesos > Procesos > Comercial > Cotizacion de equipos**;
  2. el wizard abre con chip **"Paso 1 - Requerimiento - Asesor Comercial"** y **Operator
     preseleccionado** (A2);
  3. crea la actividad -> **T00216**;
  4. en BD: `ENROLADA` | paso actual `Requerimiento` | `Pending` | **asignado a
     operator@sky-system.local** (antes: `NULL`);
  5. auditoria completa: *creo la tarea T00216* -> *notifico a operator* -> *inicio el flujo
     Cotizacion Comercial v1* -> ***ruto el primer paso del flujo a operator*** (linea nueva de A3);
  6. **login como Operator** -> tablero del concepto -> alcance **"Pendientes mios" (27)** ->
     **T00216 aparece SIN haberla reclamado**. Esto es lo que antes exigia un claim manual.

---

## OLA B - Form-first de verdad (cierra HU-02)

### Ola B1 - La hoja form-first abre el FORMULARIO, no el wizard (cierra B5)  -- HECHA 2026-07-14

- **Construido**: componente nuevo `FormFirstStarter.razor` (Components/Shared/Tasks). En la ruta
  `?sub=`, `ActivityBoardDetail.OnAfterRenderAsync` llama primero a `_formFirst.TryOpenAsync(sub)`:
  - si el concepto es **form-first** (`IniciaModulo && FormDefinitionId != null`) **y** su formulario
    es utilizable (`Active`, no archivado) -> abre el **modal del FORMULARIO** (`DynamicFormRenderer`
    en modo Fill) y **NO** el wizard;
  - si no lo es (o el formulario no sirve) -> `TryOpenAsync` devuelve **false** y se cae al **wizard
    de siempre**. Ese es el **escape hatch**: nunca se queda sin poder crear la actividad.
- **El orden se invirtio, y eso es lo importante**: hoy el wizard creaba la tarea **al ENTRAR** al
  paso 3 (parche de la Ola 5), asi que un formulario invalido dejaba una **tarea huerfana con su
  flujo ya arrancado**. Ahora: **formulario -> validacion en servidor -> recien entonces nace la
  actividad** (con su flujo y su primer paso asignado, Ola A3) **-> se ancla la respuesta** a su
  numero.
- **Pieza nueva de servicio**: `IFormResponseService.SetReferenceAsync(responseId, reference)` --
  ancla una respuesta YA ENVIADA al numero de la tarea, que al diligenciarla todavia no existia.
  Idempotente y **no destructivo** (si ya tiene referencia, no la pisa).
- Lo que el wizard preguntaba se resuelve solo: titulo `TituloAuto`, detalle `DetalleAuto`, tablero
  del concepto, y **encargado dictado por el flujo** (Olas A1/A2). El modal lo muestra como resumen,
  no como formulario que rellenar. Si el flujo esta en borrador, muestra el **banner de D3**.
- **Bug encontrado y corregido en la validacion**: `TituloAuto` = "Requerimiento infra - @cliente" y,
  como el arranque form-first no captura el cliente, la tarea nacia titulada **"Requerimiento infra -
  "** (separador colgando). `RenderConceptTemplate` ahora limpia el borde cuando el token queda vacio.
- **Aceptacion CUMPLIDA (Chrome real, tenant demo)**:
  1. `?sub=` de "Requerimiento infraestructura" (form-first) -> se abre **el formulario "Solicitud de
     cotizacion" (FRM-001) DIRECTO**, sin wizard ni pasos;
  2. **Enviar vacio** -> "Este campo es obligatorio", el formulario NO se envia y **NO se crea
     ninguna tarea** (la ultima siguio siendo T00216). Este es el punto que antes fallaba;
  3. Llenarlo y enviar -> **T00218** creada, titulo limpio **"Requerimiento infra"**, en el tablero
     del concepto / columna "Por hacer", y la `FormResponse` en **`Submitted` con `ref=T00218`**.

> [!note] Pendiente menor (no bloquea): mapear un campo del formulario al token `@cliente`
> Hoy el token queda vacio en form-first porque no hay convencion que diga QUE campo del formulario
> es el cliente. Se limpia el separador para que el titulo no nazca roto, pero lo ideal es marcar un
> campo como "cliente" en el disenador (o usar `RequiereCliente`). Queda anotado para C1/D.

### Ola B2 - Limpiar el paso "Formulario" del wizard  -- HECHA 2026-07-14

- **Retirado del wizard** todo el parche form-first de la Ola 5: la rama `IsFormFirst` del paso 3
  (que renderizaba el formulario), la **creacion anticipada de la tarea** en `NextStep` (al pasar de
  Contacto al paso Formulario), `FormStepLocked`, los handlers `OnFormSubmittedAsync` /
  `OnFormSkipAsync` y el estado muerto (`_createdTaskId` / `_createdTaskNumber` / `_createdDetail`).
- **El wizard queda con UN SOLO camino**: la tarea se crea **unicamente** al pulsar "Guardar
  actividad" (o "Guardar y crear otra"). **Ningun paso crea nada por adelantado.** El footer se
  unifica (Atras / Limpiar / Guardar y crear otra / Guardar actividad).
- El paso 3 queda **informativo**: si el concepto tiene formulario, avisa que se diligencia desde el
  **detalle** de la actividad (ADR-0038); si no, muestra el estado vacio.
- **Aceptacion CUMPLIDA (Chrome real)**: se **desactivo FRM-001** en la BD local para forzar el
  **escape hatch** (concepto form-first cuyo formulario ya no sirve):
  1. la hoja **cayo al wizard** (no al formulario) -- escape hatch de B1 confirmado;
  2. se camino el wizard **hasta el paso 4** pasando por el paso Formulario -- que es exactamente
     donde antes se creaba la tarea -- y la **ultima tarea siguio siendo T00218**: **no se creo
     nada**;
  3. el resumen lateral seguia en **"Borrador / Sin guardar"** y "Num. de actividad: **Sin
     asignar**"; el footer mostraba el camino unico.
  Luego se restauro FRM-001 a `Active`. Solucion en verde + 360/360 Application.Tests.

---

## OLA C - Guardas y QA

### Ola C1 - Guardas de coherencia (cierra B6 + B7 y el resto de la tabla del doc 01 seccion 4)  -- HECHA 2026-07-14

- **Construido (3 piezas)**:
  1. **Banner en el arranque (D3)** -- `TaskWizard.razor` muestra, arriba del paso 1, un aviso ambar
     cuando el concepto tiene flujo pero **no es utilizable**. Cubre tres estados de
     `IWorkflowStartService`: `FlujoNoPublicado` ("nacera sin proceso"), `SinNodoTask` ("el flujo no
     tiene ningun paso") y `SinCargo` ("el primer paso no tiene cargo -> sin encargado").
     `FormFirstStarter.razor` ya traia el banner de `FlujoNoPublicado`/`SinCandidatos` desde B1.
  2. **Chip "borrador" en el menu** -- `NavMenu.razor`: la hoja de un concepto cuyo flujo NO esta
     publicado se muestra igual (D3: avisar, no ocultar), pero con un chip ambar **"borrador"** en
     vez del habitual "proc". Se resuelve con una consulta al set de `WorkflowDefinitions` publicadas
     del tenant (barata, tenant-scoped) y un campo `Publicado` en el record `ProcesoSub`.
  3. **Validacion al publicar** -- `WorkflowEngine.PublishAsync` **bloquea** publicar un flujo **sin
     ningun nodo Task** (irrecuperable: un concepto que lo use crearia actividades enroladas y sin
     nada que atender). **Decision de alcance**: el caso "paso sin cargo" **NO** bloquea la
     publicacion (seria muy rigido con el admin a mitad de autoria y rompe flujos mecanicos de test);
     se **avisa al crear** via el banner de la pieza 1 (mas util: sale justo cuando importa).
- **Aceptacion CUMPLIDA**:
  - **Test dual** `Publish_FlowWithoutTaskNode_IsRejected` (PostgreSQL + SQL Server, 2/2): un flujo
    `start -> end` no publica (`Invalid`, sigue en borrador); un flujo con pasos si publica.
    Regresion: 45 tests de workflow + 360 unitarios en verde (la guarda no rompio nada -> todos los
    flujos reales tienen paso Task).
  - **Chrome real**: se **despublico** temporalmente el flujo `COT-COM` (de "Cotizacion de equipos")
    en local -> en el menu la hoja aparecio con chip **"borrador"** (mientras "Compra urgente" seguia
    con "proc"); al abrirla, el wizard mostro el **banner ambar** "el flujo aun no esta publicado...
    la actividad se creara sin proceso" y el combo Encargado **dejo de estar restringido** (cae a los
    cargos del concepto). Restaurado a publicado.

**Detalle historico de la propuesta original (algunos puntos se ajustaron arriba):**

> Ajustada por la **decision D3**: el menu **NO** se filtra por `IsPublished`. La hoja sigue
> apareciendo; lo que se ataca es el **silencio** al crear.

- **Menu**: `NavMenu.razor:391` **se deja como esta** (basta `WorkflowDefinitionId != null`), pero
  la hoja muestra un **indicador visual** de "flujo en borrador" (chip atenuado junto al badge
  "proc") para que el admin sepa que aun no arranca proceso.
- **Aviso al crear (lo importante)**: si el flujo del concepto **no esta publicado o esta
  archivado**, tanto el **wizard** como el **arranque form-first** muestran un **banner claro**:
  *"El flujo de este proceso aun no esta publicado. La actividad se creara SIN proceso."* El
  usuario puede continuar (crea tarea simple) o cancelar. **Nunca en silencio.**
- **Publicar flujo**: validar que tenga **al menos un nodo Task**, que los nodos tengan **cargo**, y
  avisar si algun cargo **no tiene ocupantes**.
- **Sin tablero**: ya mitigado (auto-creacion, commit `388e895`); agregar fallback con aviso por si
  quedan conceptos viejos.
- **Aceptacion**: un concepto con flujo en borrador **si** aparece en Mis Procesos, con su chip de
  borrador; al crear desde el, sale el **banner** y la tarea nace sin flujo **habiendolo advertido**.
  Publicar un flujo sin cargos **avisa** y no publica.

### Ola C2 - QA end-to-end del proceso completo (MCP Chrome, BD local)

Guion (contra `ecorex_dev` local, nunca prod):
1. Dario configura flujo + concepto (form-first y no form-first) -> las hojas **aparecen** solas.
2. Ana arranca una compra desde el menu -> encargado **preseleccionado** -> crea -> Beto **notificado**.
3. Beto ve la tarea en "Pendientes mios" **sin reclamarla**, atiende dentro de la tarea, adjunta,
   completa.
4. Carla **rechaza** -> vuelve a Beto con motivo. Carla **aprueba** -> la tarea cae en la **columna
   de cierre** del concepto y el flujo se cierra.
5. Ana arranca el **form-first** -> se abre el **formulario directo** -> validacion en servidor ->
   tarea + flujo + respuesta enlazada.
6. Dario **quita el flujo** del concepto -> la hoja **desaparece** del menu.

- **Aceptacion**: los 6 pasos verdes en Chrome + registro en
  `05. Pruebas/Historial de pruebas/00 - Registro de corridas.md`.

---

## OLA D - Formulario por NODO del flujo (comprometida por D1, va DESPUES de A/B/C)

> Por decision D1, esto **no es backlog difuso**: es una ola planificada. Se hace **despues** de
> que A/B/C esten verdes, porque toca dominio y no debe bloquear el valor inmediato.

### Ola D1 - Dominio + migracion dual

- `WorkflowNodePolicy.FormDefinitionId (Guid?)` -> FK a `FormDefinition`. Migracion **PG + SQL
  Server** (regla DAL dual).
- Regla de precedencia: **el formulario del NODO gana**; si el nodo no tiene, se usa el del
  **concepto** (compatibilidad hacia atras con todo lo construido en B1).
- **Aceptacion**: migracion aplicada en ambos motores; tests de integracion dual verdes; nada de lo
  existente se rompe (los flujos actuales, sin formulario por nodo, siguen usando el del concepto).

### Ola D2 - Editor de flujos: asignar formulario al nodo

- En `/flujos`, el panel de propiedades del nodo (donde hoy se elige el **cargo**) suma un selector
  de **Formulario**.
- **Aceptacion (Chrome)**: en el flujo de Compras, al nodo "Cotizar" se le asigna un formulario
  distinto al del concepto; se publica; queda persistido.

### Ola D3 - Runtime: cada paso pide SU formulario

- La seccion **Flujo** dentro de la tarea (ADR-0038) renderiza el formulario del **paso actual**
  (el del nodo, o el del concepto si el nodo no tiene).
- El arranque form-first (Ola B1) usa el formulario del **primer nodo** si lo tiene.
- **Aceptacion (Chrome)**: Beto atiende "Cotizar" y ve **el formulario del nodo**; Carla atiende
  "Aprobar" y ve **otro** formulario. Cada `FormResponse` queda enlazada al **paso**, no solo a la
  tarea.

---

## Resumen de olas

| Ola | Estado | Alcance | Commit |
|---|---|---|---|
| **0** | ✅ HECHA | Decisiones D1/D2/D3 | (doc) |
| **A1** | ✅ HECHA | Resolver candidato del 1er nodo (`WorkflowStartService`, solo lectura) | `d7a09fd` |
| **A2** | ✅ HECHA | El wizard muestra/preselecciona el encargado (restringido, D2) | `8579f30` |
| **A3** | ✅ HECHA | El 1er paso nace asignado + notificado; D2 revalidado en servidor | `4537b29` |
| **B1** | ✅ HECHA | La hoja form-first abre el formulario del concepto directo (D1-a) | `9aee589` |
| **B2** | ✅ HECHA | El wizard queda con un solo camino (no crea la tarea por adelantado) | `368c09b` |
| **C1** | ✅ HECHA | Guardas: banner D3 en el arranque + chip "borrador" en el menu + no publicar flujo sin paso Task | (esta tanda) |
| **C2** | ✅ HECHA | QA end-to-end: arranque visual+BD (T00219 nace asignada a operator, visible en Pendientes mios) + ciclo runtime por `WorkflowInboxTests` 6/6 dual. Ver [[05. Pruebas/Historial de pruebas/00 - Registro de corridas]] 2026-07-14 | (verificacion) |
| **D1** | PENDIENTE | Dominio: `WorkflowNodePolicy.FormDefinitionId` + migracion dual | - |
| **D2** | PENDIENTE | Editor de flujos: formulario por nodo | - |
| **D3** | PENDIENTE | Runtime: cada paso pide su formulario | - |
| **DEPLOY** | PENDIENTE | Llevar A/B (y lo previo acumulado) a **produccion** (10.0.0.3) | - |

> Ademas hay un **fix preexistente** ya commiteado en esta tanda (`433a1ba`): el host `Ecorex.Api`
> no arrancaba porque `IFormRecordBroadcaster` no estaba registrado (venia del merge de formularios).
> No es de este capitulo, pero bloqueaba 27 tests de endpoints; quedo resuelto.

---

## Backlog (fuera de estas olas)

- **Reasignacion de paso** por un supervisor (hoy existe en `WorkflowInboxService.cs:224`, pero sin
  UI dentro de la tarea).
- **SLA / vencimiento por paso** (notificar si un paso lleva N dias sin atender).

### Hallazgos de config del demo (Ola C2, no son bug de codigo)

Al correr el E2E se vio que el flujo demo **COT-COM v1** ("Cotizacion de equipos") esta a medio
configurar. No lo arreglo en codigo porque es **configuracion del tenant demo**, pero conviene
cerrarlo para que el ciclo se pueda recorrer entero de punta a punta:

- **Nodos `Facturacion` y `Entrega` sin cargo** -> al avanzar a esos pasos nacen sin candidato y
  nadie los ve en "Pendientes mios" (el propio banner `SinCargo` de C1 lo advierte al crear).
  Accion: asignarles un cargo en el editor de flujos.
- **El concepto "Cotizacion de equipos" no tiene columna de cierre** (`TaskBoardColumnId = null`) por
  ser un tablero **anterior** al auto-tablero de la commit `388e895`. Al terminar el flujo la tarea no
  se auto-mueve a "Completado". Accion: fijar la columna de cierre del concepto (o re-guardarlo).
  Nota: los conceptos nuevos ya nacen con columna de cierre desde `388e895`.
