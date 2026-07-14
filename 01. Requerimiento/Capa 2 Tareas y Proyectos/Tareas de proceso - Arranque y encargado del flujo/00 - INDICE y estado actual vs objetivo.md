---
tipo: indice-proyecto
proyecto: Tareas de proceso - Arranque y encargado del flujo
modulos_web: /actividades (000636), /conceptos (000270), /flujos (000291), /formularios (000131), /dependencias (000850), /configuracion-menu (000194)
estado: EN ESPECIFICACION (2026-07-14) - auditoria de codigo hecha, olas por ejecutar
fecha: 2026-07-14
autor: documentado por agente IA (auditoria read-only sobre el codigo real) a partir del dictado del usuario
---

# Tareas de proceso - Arranque y encargado del flujo

> Capitulo de construccion. Continua a [[Modulo de Tareas - Creacion y ejecucion/00 - INDICE y estado actual vs objetivo|Modulo de Tareas - Creacion y ejecucion]],
> que dejo construido el **puente concepto <-> tarea** y el menu **Mis Procesos**. Este capitulo
> cierra lo que quedo a medias en el **arranque de una tarea que nace de un proceso**: el
> **encargado que dicta el flujo** y el **arranque form-first real**.

> [!warning] ESTADO 2026-07-14: el enrolamiento SI funciona; el ENCARGADO no
> Al crear una actividad desde el menu Mis Procesos, la tarea **si se enrola** en el flujo
> (instancia + primer paso `Pending`/`IsCurrent`) y sigue la secuencia BPMN. Pero el **encargado
> del paso** no se resuelve desde el flujo: el wizard lo pide a mano (filtrado por los cargos del
> CONCEPTO, no por el cargo del NODO), y el primer paso nace **sin asignado**. Ademas, los
> conceptos marcados `IniciaModulo` (form-first) **igual abren el wizard de 4 pasos** en vez de
> abrir directamente el formulario del proceso.

---

## 1. Por que existe el grupo "Procesos" en el menu

El menu no es una lista fija: una seccion puede llevar el flag **`IsProcessGroup`** y entonces
`NavMenu` **se dibuja desde los datos**. Recorre los conceptos de actividad
(`ActividadCategoria` -> `ActividadSubcategoria`) y publica como hoja **solo** las subcategorias
que tienen un **flujo asociado** (`WorkflowDefinitionId != null && !IsArchived`).

La razon de negocio: **algunas actividades no son tareas sueltas, nacen como un proceso**
(una compra, un requerimiento de infraestructura). Ese grupo es la **puerta de entrada** para
arrancar uno de esos procesos. Es autoadministrable: si se configura un concepto nuevo y se le
asocia un flujo, **aparece solo** en el menu; si se le quita el flujo, **desaparece**.

Referencias de codigo (2026-07-14):

| Pieza | Archivo:linea | Que hace |
|---|---|---|
| Flag de seccion | `MenuNode.IsProcessGroup` | marca la seccion que despliega procesos |
| Armado del grupo | `NavMenu.razor:384-395` | filtra subcategorias con `WorkflowDefinitionId != null && !IsArchived` |
| Hoja del menu | `NavMenu.razor:134` | enlaza a `actividades?sub={subcategoriaId}` |

---

## 2. La cadena esperada (dictado del usuario, 2026-07-14)

> "Al pinchar sobre la opcion final que determina un concepto que tiene un flujo de proceso
> relacionado se abre el tablero que corresponde y abre inmediatamente el modal wizard
> preseleccionando la categoria de actividad y la subcategoria, **tambien el encargado actual, ya
> que ello esta dibujado en el flujo de trabajo**. Si el usuario crea la actividad, esta se
> **enrola con el flujo** y sigue la secuencia de la configuracion del flujo de trabajo."

Y la precision posterior:

> "Unas tareas estan marcadas en conceptos de actividades, lo que quiere decir que **arrancan en
> el formulario** que tenga asociado el flujo del proceso, **y no en el formulario wizard**."

---

## 3. Estado actual vs objetivo (auditoria del codigo real, 2026-07-14)

| # | Paso de la cadena | Hoy | Objetivo |
|---|---|---|---|
| 1 | Hoja del menu abre el **tablero del concepto** (`sub.TaskBoardId`) | OK | OK |
| 2 | Abre **automaticamente** el wizard grande (4 pasos) | OK | OK, *salvo form-first* (ver 4) |
| 3 | Preselecciona **Categoria** y **Subcategoria** | OK | OK |
| 4 | Preselecciona el **Encargado que dicta el flujo** | **NO** | **SI** (candidato del 1er nodo) |
| 5 | Al crear, **enrola** la tarea en el flujo (instancia + 1er paso `Pending`/`IsCurrent`) | OK | OK |
| 6 | El **primer paso** nace con `AssignedToTenantUserId` resuelto y **notificado** | **NO** (nace null) | **SI** |
| 7 | Concepto `IniciaModulo` -> **arranca en el formulario**, no en el wizard | **NO** (form es el paso 3) | **SI** |

Detalle de codigo de cada brecha en [[01 - Arquitectura del arranque (menu, encargado, form-first)]].

---

## 4. Las brechas (por que pasan)

- **B1 - El wizard no mira el flujo para el encargado.** `TaskWizard.OpenAsync` llama
  `ResolveEncargadoAsync` (`TaskWizard.razor:515-524`), que **solo filtra las opciones del combo**
  por `SelectedSub.CargoIds` (los cargos configurados en el CONCEPTO, pestana de permisos), **no**
  por la `WorkflowNodePolicy` del **primer nodo del BPMN**. `INodeAssigneeResolver` **no esta
  inyectado** en el wizard (hoy solo lo consumen `WorkflowInboxService.cs:124,307` y
  `ActivityBoardService.cs:759`).
- **B2 - No hay preseleccion de valor.** `_assigneeId` **nunca se asigna** en `OpenAsync`; solo se
  *limpia* si el elegido no esta en las opciones. El combo arranca en "Selecciona Encargado" y el
  resumen dice "Sin asignar", incluso cuando hay un unico candidato posible.
- **B3 - El primer paso del flujo nace sin asignado.** `WorkflowEngine.AddStep`
  (`WorkflowEngine.cs:639-656`) **no fija** `AssignedToTenantUserId`. La asignacion por cargo se
  resuelve **perezosamente**: el paso queda "atendible" por el cargo y se concreta cuando alguien
  lo **reclama** desde el tablero (`WorkflowInboxService.cs:200`).
- **B4 - Dos nociones de "responsable" que pueden divergir.** `TaskItem.AssigneeTenantUserId`
  (elegido a mano en el wizard) vs `WorkflowStepHistory.AssignedToTenantUserId` (paso actual del
  flujo). Nada los reconcilia en `CreateAsync`.
- **B5 - Form-first no es "first".** `TaskWizard` ya distingue `IsFormFirst`
  (`TaskWizard.razor:385` = `IniciaModulo && FormDefinitionId != null`), pero el wizard **igual
  arranca en el paso 1**; el formulario es el **paso 3**. El usuario debe caminar Informacion ->
  Contacto -> Formulario. Lo dictado es que **la puerta de entrada sea el formulario**.
- **B6 - Flujo sin publicar: enrolamiento silencioso.** `TaskItemService.cs:287-288` solo arranca
  el flujo si `IsPublished && !IsArchived`. Pero `NavMenu.razor:391` publica la hoja con solo
  `WorkflowDefinitionId != null`. Resultado: el concepto **aparece** en Mis Procesos, pero la
  actividad se crea **sin flujo y sin avisar a nadie**.
- **B7 - Concepto sin tablero: el preset se pierde.** `Actividades.razor:60-64`: si la subcategoria
  no tiene `TaskBoardId`, cae al indice de tableros y **el wizard nunca se abre**. Mitigado desde
  2026-07-14 por la auto-creacion de tablero al guardar concepto (commit `388e895`), pero puede
  quedar en conceptos viejos.

---

## 5. Decisiones de diseno -- CERRADAS (usuario, 2026-07-14)

Hallazgo que las motivo: **el formulario del arranque sale del CONCEPTO, no del nodo del flujo.**
Verificado: `FormDefinitionId` existe **unicamente** en `ActividadSubcategoria`
(`Ecorex.Domain/Entities/ActividadSubcategoria.cs:60`); **no hay** formulario por nodo BPMN. Asi que
cuando el dictado dice "el formulario que tenga asociado el flujo del proceso", en el modelo actual
eso es **el formulario del concepto que tambien lleva el flujo**.

| # | Decision | Impacto |
|---|---|---|
| **D1** | **Ambos, en dos tiempos**: el arranque form-first usa **ahora** el formulario del **concepto** (`Subcategoria.FormDefinitionId`); el **formulario por NODO** (`WorkflowNodePolicy.FormDefinitionId`) se compromete como **Ola D**, no como backlog difuso | Ola B1 (ahora) + Olas D1/D2/D3 (despues) |
| **D2** | **El flujo manda**: el iniciador **no puede** elegir un encargado fuera del cargo del primer nodo. El combo se restringe a los candidatos y el servidor **valida** | Olas A2 + A3 |
| **D3** | Un concepto con flujo **sin publicar** **si aparece** en el menu (con chip de "borrador"), pero al crear se muestra un **banner**: la actividad nacera **sin proceso**. Se ataca el **silencio**, no la visibilidad | Ola C1 (cambia respecto de la propuesta inicial) |

Detalle y consecuencias en [[03 - Plan por olas pequenas]] > Ola 0.

---

## 6. Documentos del capitulo

- [[01 - Arquitectura del arranque (menu, encargado, form-first)]] - como funciona hoy la cadena
  (con file:line), el modelo objetivo del encargado y del arranque form-first.
- [[02 - Historias de usuario (arranque de tareas de proceso)]] - 5 historias (el SISTEMA tambien
  es actor) de como se usa el modulo al iniciar tareas de proceso.
- [[03 - Plan por olas pequenas]] - olas chicas, cada una entregable y verificable, con criterio
  de aceptacion.

---

## 7. Reglas del proyecto que aplican

- Multi-tenant real (`HasQueryFilter`), transacciones en toda operacion multi-tabla,
  soft-delete, concurrencia optimista, zona horaria del tenant + UTC. Ver `CLAUDE.md` seccion 5.
- El runtime del flujo vive **DENTRO de la tarea** (ADR-0038): no hay bandeja "Mis pasos"; los
  pasos pendientes se descubren en el **tablero** (alcance "Pendientes mios").
- Archivos nuevos en **ASCII**.
