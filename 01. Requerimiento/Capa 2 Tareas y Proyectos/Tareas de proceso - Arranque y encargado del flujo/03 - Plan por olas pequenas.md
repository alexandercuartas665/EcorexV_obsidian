---
tipo: plan-olas
proyecto: Tareas de proceso - Arranque y encargado del flujo
estado: POR EJECUTAR (2026-07-14)
fecha: 2026-07-14
---

# 03 - Plan por olas pequenas

> Cada ola es **chica, entregable y verificable por si sola**. Ninguna deja el sistema roto.
> Orden pensado para que el valor se vea temprano: **A** cierra el encargado (lo que mas duele),
> **B** cierra el form-first, **C** pone las guardas y el QA.

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

### Ola A2 - El wizard muestra y preselecciona ese encargado (cierra B1 + B2)

- **Que**: `TaskWizard` inyecta el servicio de A1. En `OpenAsync`, **si la subcategoria tiene
  flujo**, el combo "Encargado":
  - se llena con los **candidatos del primer nodo** (no con `SelectedSub.CargoIds`),
  - muestra la **etiqueta del paso** ("Paso 1 - Comprador"),
  - **preselecciona** `_assigneeId` si hay **un unico** candidato,
  - queda **restringido** a esos candidatos (segun D2).
- Si la subcategoria **no** tiene flujo -> comportamiento actual intacto (cargos del concepto).
- **Aceptacion (Chrome)**: clic en Mis Procesos > Compras > Compra -> el modal abre con Encargado
  **ya puesto** en Beto y la etiqueta del cargo. Un concepto sin flujo sigue igual que hoy.

### Ola A3 - Persistir y notificar el asignado del primer paso (cierra B3 + B4)

- **Que**: en `TaskItemService.CreateAsync`, **dentro de la misma transaccion**, despues de
  `StartInstanceAsync`:
  1. tomar el paso actual (`IsCurrent`, `Pending`),
  2. **validar** que el encargado elegido sea candidato del cargo del nodo,
  3. fijar `WorkflowStepHistory.AssignedToTenantUserId`,
  4. **notificar** a ese usuario (campana + email), como ya se hace con el assignee,
  5. auditar.
- **Aceptacion**: crear una compra -> en BD el primer `WorkflowStepHistory` tiene
  `AssignedToTenantUserId = Beto`; Beto **recibe notificacion al nacer la tarea** (no al reclamarla);
  la tarea aparece en su alcance "Pendientes mios" **sin** necesidad de claim.

---

## OLA B - Form-first de verdad (cierra HU-02)

### Ola B1 - La hoja form-first abre el FORMULARIO, no el wizard (cierra B5)

- **Que**: en la ruta `?sub=`, si la subcategoria es **form-first**
  (`IniciaModulo && FormDefinitionId != null`), `ActivityBoardDetail` abre un **modal de
  formulario** (`DynamicFormRenderer` en modo Fill) en vez del `TaskWizard`.
- Los datos que el wizard pedia se resuelven **del concepto** (tabla en el doc 01, seccion 3.2):
  titulo `TituloAuto`, detalle `DetalleAuto`, tablero, y **encargado desde el flujo** (Ola A1).
- **Al "Enviar"** (y solo entonces): validar en servidor -> crear tarea -> arrancar flujo -> asignar
  1er paso -> persistir `FormResponse` enlazada. **Todo en una transaccion.** Si el formulario no
  valida, **no se crea nada** (cambio respecto de hoy, que crea la tarea al entrar al paso 3).
- **Escape hatch**: formulario archivado / sin permiso -> **cae al wizard** con aviso.
- **Aceptacion (Chrome)**: Mis Procesos > Comercial > "Requerimiento infraestructura" -> se abre
  **el formulario directo** (sin wizard); al enviar nace la tarea con su flujo y la respuesta
  enlazada por numero. Dejar un campo obligatorio vacio -> bloquea y **no crea** tarea.

### Ola B2 - Limpiar el paso "Formulario" del wizard

- **Que**: con B1, el paso 3 del wizard ya **no** necesita crear la tarea (eso era el parche de la
  Ola 5 anterior). El wizard queda para el camino **no** form-first: si el concepto tiene formulario
  pero **no** `IniciaModulo`, el formulario sigue siendo un paso **opcional** (diligenciar luego).
- Retirar `FormStepLocked` / la creacion anticipada; simplificar el footer.
- **Aceptacion**: el wizard normal crea la tarea **solo** al pulsar "Crear actividad"; ningun camino
  crea tareas a medias.

---

## OLA C - Guardas y QA

### Ola C1 - Guardas de coherencia (cierra B6 + B7 y el resto de la tabla del doc 01 seccion 4)

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

| Ola | Alcance | Toca | Cierra |
|---|---|---|---|
| **0** | Decisiones D1/D2/D3 -- **CERRADA 2026-07-14** | nada (doc) | - |
| **A1** | Resolver candidato del 1er nodo | Application (servicio nuevo, solo lectura) | - |
| **A2** | Wizard muestra/preselecciona el encargado (**restringido**, D2) | TaskWizard.razor | B1, B2 |
| **A3** | Persistir + notificar asignado del 1er paso (**validado**, D2) | TaskItemService, WorkflowEngine | B3, B4 |
| **B1** | Hoja form-first abre el formulario del **concepto** directo (D1-a) | Actividades / ActivityBoardDetail / servicio de alta | B5 |
| **B2** | Limpiar el paso Formulario del wizard | TaskWizard.razor | deuda de la Ola 5 anterior |
| **C1** | Guardas: **banner** si el flujo no esta publicado (D3) + validacion al publicar + tablero | Wizard, form-first, WorkflowService, NavMenu (chip borrador) | B6, B7 |
| **C2** | QA end-to-end en Chrome | - | verificacion |
| **D1** | Dominio: `WorkflowNodePolicy.FormDefinitionId` + migracion dual | Domain, Infrastructure | D1-b |
| **D2** | Editor de flujos: formulario por nodo | Flujos.razor | D1-b |
| **D3** | Runtime: cada paso pide su formulario | TaskDetailModal (seccion Flujo), form-first | D1-b |

---

## Backlog (fuera de estas olas)

- **Reasignacion de paso** por un supervisor (hoy existe en `WorkflowInboxService.cs:224`, pero sin
  UI dentro de la tarea).
- **SLA / vencimiento por paso** (notificar si un paso lleva N dias sin atender).
