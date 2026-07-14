---
tipo: arquitectura
proyecto: Tareas de proceso - Arranque y encargado del flujo
fecha: 2026-07-14
---

# 01 - Arquitectura del arranque (menu, encargado, form-first)

Este documento describe **como funciona hoy** la cadena (con `file:line` del codigo real,
auditado el 2026-07-14) y **cual es el modelo objetivo**. Es la referencia para las olas de
[[03 - Plan por olas pequenas]].

---

## 1. La cadena de hoy, paso a paso (codigo real)

```
[Menu] Mis Procesos > Procesos > Comercial > Requerimiento infraestructura
   |
   |  NavMenu.razor:384-395  arma el grupo: subcategorias con WorkflowDefinitionId != null
   |  NavMenu.razor:134      la hoja enlaza a  /actividades?sub={subcategoriaId}
   v
[Pagina] Actividades.razor
   |
   |  :52-66   parsea ?sub=  ->  catalogo.GetSubcategoriaAsync(subId)
   |  :60      el TABLERO = sub.TaskBoardId       (si es null -> cae al indice, ver B7)
   |  :25-28   <ActivityBoardDetail BoardId=... PresetSubcategoriaId=_resolvedSubId />
   v
[Tablero] ActivityBoardDetail.razor
   |
   |  :656       [Parameter] Guid? PresetSubcategoriaId
   |  :800       _openWizardPending = PresetSubcategoriaId is not null
   |  :753-761   OnAfterRender (una sola vez) -> _wizard.OpenAsync(presetSubcategoriaId: presetSub)
   v
[Modal] TaskWizard.razor  (wizard de 4 pasos)
   |
   |  :424-428   _categoriaId    = sub.CategoriaId            <- OK preselecciona
   |  :430-433   _subcategoriaId = presetSubcategoriaId       <- OK preselecciona
   |  :434       ResolveEncargadoAsync()
   |  :515-524      -> SOLO filtra las opciones del combo por SelectedSub.CargoIds
   |                   (cargos del CONCEPTO, NO el cargo del nodo BPMN)
   |                -> _assigneeId NUNCA se asigna  ==> "Selecciona Encargado" / "Sin asignar"
   |  :385       IsFormFirst = IniciaModulo && FormDefinitionId != null
   |  :195-211   el formulario es el PASO 3 (DynamicFormRenderer con SelectedSub.FormDefinitionId)
   v
[Crear] TaskItemService.CreateAsync
   |
   |  :285       workflowDefinitionId = subcategoria?.WorkflowDefinitionId ?? activityType?...
   |  :286-298   si existe Y (IsPublished && !IsArchived)  ->  WorkflowEngine.StartInstanceAsync
   |                (en la MISMA transaccion; si falla -> rollback)
   v
[Motor] WorkflowEngine.StartInstanceAsync
   |
   |  :235-244   crea WorkflowInstance (Running)
   |  :246-268   task.WorkflowInstanceId = instancia;  tarea -> Active
   |  :273-274   ActivateNode(startNode) + Advance(...)
   |  :609-613   el startEvent se auto-completa
   |  :614-622   los gateways se auto-resuelven (ADR-0037)
   |  :623-635   el primer nodo Task queda  Status=Pending, IsCurrent=true
   |  :639-656   AddStep(...)  ***NUNCA setea AssignedToTenantUserId***  ==> nace SIN asignado
   v
[Descubrimiento perezoso]  el paso se resuelve por cargo solo cuando alguien lo consulta/reclama
       WorkflowInboxService.cs:124   ResolveCandidatesAsync (candidatos por cargo)
       WorkflowInboxService.cs:200   claim -> step.AssignedToTenantUserId = tenantUserId
       ActivityBoardService.cs:754-759  alcance "Pendientes mios" del tablero
```

**Veredicto**: el **enrolamiento funciona** (pasos 5 y 6 de la tabla del doc 00). Lo que falta es
el **encargado**: ni el wizard lo propone desde el flujo, ni el paso queda asignado al crear.

---

## 2. Modelo objetivo del ENCARGADO

### 2.1 De donde debe salir

El "encargado" de una actividad-proceso **no lo inventa el usuario**: lo **dicta el flujo**. En el
BPMN, el **primer nodo de tipo Task** (despues del startEvent y de los gateways, que se
auto-resuelven) tiene una `WorkflowNodePolicy` con su **cargo**. De ese cargo salen los
**candidatos** (ocupantes del cargo en el organigrama).

```
WorkflowDefinition (publicada)
  startEvent  --(auto)-->  [gateway?]  --(auto)-->  PRIMER NODO Task
                                                      |
                                                      +-- WorkflowNodePolicy -> Cargo (OrgUnit)
                                                                                  |
                                                       INodeAssigneeResolver ------+
                                                                                  v
                                                                     candidatos = ocupantes del cargo
```

**Regla**: el combo "Encargado" del wizard, **cuando la subcategoria tiene flujo**, debe llenarse
con **esos** candidatos (no con `SelectedSub.CargoIds`), mostrar el **cargo del paso** como
etiqueta (ej. "Paso 1 - Comprador") y **preseleccionar** cuando hay un unico candidato.

`INodeAssigneeResolver` **ya existe** y ya resuelve cargo -> candidatos. Hoy solo lo consumen el
inbox y el tablero; **hay que consumirlo tambien desde el arranque**.

### 2.2 Que se persiste al crear

Hoy `TaskItem.AssigneeTenantUserId` (elegido a mano) y `WorkflowStepHistory.AssignedToTenantUserId`
(paso actual) **pueden divergir** (B4 del doc 00). Objetivo:

- Al crear una actividad-proceso, tras `StartInstanceAsync`, **fijar el asignado del paso actual**
  con el encargado elegido, **validando que sea candidato legitimo del cargo del nodo**. Si no lo
  es -> error de validacion (no se permite asignar fuera del cargo que dicta el flujo).
- **Notificar** a ese usuario. Hoy las notificaciones de creacion (`TaskItemService.cs:262-275`)
  solo van al assignee elegido a mano y a los destinatarios del concepto; **el candidato del paso
  no recibe nada**.
- `TaskItem.AssigneeTenantUserId` sigue siendo el **lider/iniciador** de la actividad; el
  **atendedor de cada paso** lo dicta el flujo. Son dos cosas distintas **por diseno**, pero en el
  **primer paso deben coincidir** (es lo que el usuario ve como "el encargado actual").

> [!note] Sigue valiendo lo dicho en el capitulo anterior
> "Para actividades-proceso, el 'atiende cada paso' NO es el encargado unico: lo resuelve el flujo
> por cargo del nodo." Lo que agregamos es que **el arranque no deje el primer paso colgando**.

---

## 3. Modelo objetivo del ARRANQUE FORM-FIRST

### 3.1 Que significa la marca del concepto

En `/conceptos` > editar sub-categoria > **"Gestion del inicio del proceso"** existe el switch
**"Inicia modulo al crear la tarea"** (`Conceptos.razor:336`), que persiste
`ActividadSubcategoria.IniciaModulo` (`ActividadSubcategoria.cs:38`).

Semantica dictada por el usuario: **esa actividad NO arranca por el wizard, arranca por el
FORMULARIO**. El formulario es la puerta de entrada; el usuario diligencia datos y con eso nace la
tarea y arranca el flujo.

### 3.2 Que hay hoy y que falta

Hoy `TaskWizard` **si distingue** el caso (`IsFormFirst`, `TaskWizard.razor:385`) y **si** crea la
tarea al entrar al paso Formulario (Ola 5 del capitulo anterior). **Pero el wizard igual arranca en
el paso 1**: el usuario camina Informacion -> Contacto -> **Formulario**. Eso **no** es form-first.

```
Hoy (form-first a medias)             Objetivo (form-first real)
-------------------------             --------------------------
click hoja del menu                   click hoja del menu
   -> tablero                            -> tablero
   -> wizard PASO 1 Informacion          -> MODAL DE FORMULARIO directo
   -> PASO 2 Contacto                       (DynamicFormRenderer, modo Fill)
   -> PASO 3 Formulario   <-- aqui          titulo/detalle desde TituloAuto/DetalleAuto
      (recien aqui se crea la tarea)        encargado = candidato del 1er nodo del flujo
   -> PASO 4 Documentos                  -> "Enviar" -> crea tarea + arranca flujo + guarda respuesta
```

**De donde salen los datos que el wizard pedia**, ahora que no hay wizard:

| Dato | Origen en form-first |
|---|---|
| Categoria / Subcategoria | del `?sub=` de la hoja del menu (ya se conocen) |
| Titulo | `Subcategoria.TituloAuto` (con tokens, ej. `@cliente`) |
| Descripcion | `Subcategoria.DetalleAuto` |
| Tablero / columna | `Subcategoria.TaskBoardId` + primera columna |
| Encargado | **candidato del primer nodo del flujo** (seccion 2) |
| Cliente / tercero | del propio formulario, si `RequiereCliente` |
| Prioridad / fecha | por defecto del concepto; editables luego en el detalle de la tarea |

**El formulario que se abre es `Subcategoria.FormDefinitionId`** (el del concepto). Verificado:
**no existe** formulario por nodo BPMN (`FormDefinitionId` solo esta en
`ActividadSubcategoria.cs:60`). Ver la decision (a)/(b) en el doc 00, seccion 5.

### 3.3 Escape hatch

Si el concepto es form-first pero **el formulario no esta configurado** (o esta archivado), se debe
**caer al wizard normal con un aviso**, no romper. Igual si el usuario no tiene permiso sobre el
formulario.

---

## 4. Guardas de coherencia (las que hoy no existen)

| Guarda | Hoy | Objetivo |
|---|---|---|
| Concepto con flujo **sin publicar** | la hoja aparece en el menu, la tarea se crea **sin flujo** y **en silencio** (`TaskItemService.cs:287-288` vs `NavMenu.razor:391`) | o **no publicar la hoja**, o **avisar** al crear ("el flujo no esta publicado") |
| Concepto con flujo **sin tablero** | el preset se pierde, aterriza en el indice sin explicacion (`Actividades.razor:60-64`) | auto-crear tablero (ya hecho, commit `388e895`) + fallback con aviso |
| Flujo publicado **sin primer nodo Task** (solo start + end) | la tarea nace enrolada y se cierra sola o queda rara | validar al **publicar** el flujo |
| Cargo del primer nodo **sin ocupantes** | no hay candidatos -> el paso queda huerfano | avisar al crear y ofrecer el responsable de la dependencia |

---

## 5. Reglas transversales (obligatorias del proyecto)

- **Transaccion unica**: crear tarea + arrancar flujo + fijar asignado del paso + persistir la
  respuesta del formulario -> **todo o nada**.
- **Multi-tenant**: nada de filtros manuales; `HasQueryFilter` global.
- **Concurrencia optimista** en tareas y pasos.
- **Auditoria**: la asignacion del paso queda registrada (quien, cuando, por que regla).
- **ADR-0038**: el runtime del flujo se ve **dentro de la tarea**; el descubrimiento de pendientes
  es el **tablero**, no una bandeja.
