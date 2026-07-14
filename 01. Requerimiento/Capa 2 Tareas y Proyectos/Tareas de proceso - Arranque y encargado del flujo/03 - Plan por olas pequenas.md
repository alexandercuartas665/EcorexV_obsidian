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

## Ola 0 - Decisiones (bloqueante, no se codea)

Tres preguntas al usuario. Sin esto, las olas B y C quedan a ciegas.

| # | Pregunta | Opciones | Recomendacion |
|---|---|---|---|
| **D1** | El formulario del arranque, de donde sale? | **(a)** del **concepto** (`Subcategoria.FormDefinitionId`) - lo que ya existe. **(b)** **por nodo** del flujo (`WorkflowNodePolicy.FormDefinitionId`) - dominio + migracion + editor | **(a) para v1**; (b) al backlog |
| **D2** | Puede el iniciador **cambiar** el encargado que dicta el flujo? | **(a)** NO: solo candidatos del cargo del 1er nodo (combo restringido). **(b)** SI, pero avisando que se sale del flujo | **(a)**: el flujo manda |
| **D3** | Concepto con flujo **sin publicar**: que hace el menu? | **(a)** **no** muestra la hoja. **(b)** la muestra pero avisa al crear | **(a)**: si no esta listo, no se ofrece |

**Entregable**: esta tabla respondida en este mismo doc. **Sin codigo.**

---

## OLA A - El encargado que dicta el flujo (cierra HU-01)

### Ola A1 - Resolver el candidato del primer nodo (solo backend, sin UI)

- **Que**: metodo nuevo en `Ecorex.Application` (p.ej. `IWorkflowStartService.ResolveFirstStepAsync(subcategoriaId)`)
  que, dada una subcategoria con flujo **publicado**:
  1. carga la `WorkflowDefinition` publicada,
  2. recorre desde el `startEvent` saltando gateways (misma logica de auto-resolucion, ADR-0037),
  3. encuentra el **primer nodo Task**,
  4. lee su `WorkflowNodePolicy` -> **cargo**,
  5. usa **`INodeAssigneeResolver`** (ya existe) -> **candidatos**.
- **Devuelve**: `{ NodeId, NodeName, CargoId, CargoNombre, Candidatos[] }`.
- **NO toca UI. NO toca el motor.** Solo lee.
- **Aceptacion**: test unitario/integracion con el flujo de Compras del demo -> devuelve nodo
  "Cotizar", cargo "Comprador", candidato Beto. Caso borde: flujo sin nodo Task -> devuelve null
  sin reventar; cargo sin ocupantes -> lista vacia.

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

- **Menu**: la hoja solo aparece si el flujo esta **publicado** (segun D3) -> `NavMenu.razor:391`
  pasa a exigir `IsPublished && !IsArchived`.
- **Publicar flujo**: validar que tenga **al menos un nodo Task**, que los nodos tengan **cargo**, y
  avisar si algun cargo **no tiene ocupantes**.
- **Sin tablero**: ya mitigado (auto-creacion, commit `388e895`); agregar fallback con aviso por si
  quedan conceptos viejos.
- **Aceptacion**: un concepto con flujo en borrador **no** aparece en Mis Procesos; publicar un
  flujo sin cargos **avisa** y no publica.

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

## Resumen de olas

| Ola | Alcance | Toca | Cierra |
|---|---|---|---|
| **0** | Decisiones D1/D2/D3 | nada (doc) | - |
| **A1** | Resolver candidato del 1er nodo | Application (servicio nuevo, solo lectura) | - |
| **A2** | Wizard muestra/preselecciona el encargado | TaskWizard.razor | B1, B2 |
| **A3** | Persistir + notificar asignado del 1er paso | TaskItemService, WorkflowEngine | B3, B4 |
| **B1** | Hoja form-first abre el formulario directo | Actividades / ActivityBoardDetail / servicio de alta | B5 |
| **B2** | Limpiar el paso Formulario del wizard | TaskWizard.razor | deuda de la Ola 5 anterior |
| **C1** | Guardas (flujo publicado, validacion al publicar, tablero) | NavMenu, WorkflowService | B6, B7 |
| **C2** | QA end-to-end en Chrome | - | verificacion |

---

## Backlog explicito (fuera de estas olas)

- **Formulario por NODO del flujo** (decision D1 opcion (b)): `WorkflowNodePolicy.FormDefinitionId`,
  migracion dual, editor de flujos, y el runtime pidiendo el formulario del paso. Es el camino
  "BPMN de verdad", pero es un capitulo aparte.
- **Reasignacion de paso** por un supervisor (hoy existe en `WorkflowInboxService.cs:224`, pero sin
  UI dentro de la tarea).
- **SLA / vencimiento por paso** (notificar si un paso lleva N dias sin atender).
