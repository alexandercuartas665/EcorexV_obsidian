---
tipo: indice-proyecto
proyecto: Modulo de Tareas - Creacion y ejecucion de actividades
modulos_web: /actividades (000636), /crear-actividad (000038), /proyectos (000042), /conceptos (000270), /dependencias (000850), /flujos (000291), /formularios (000131), /configuracion-menu (000194)
estado: especificacion (por construir el "puente")
fecha: 2026-07-11
autor: documentado por agente IA (4 exploradores read-only sobre el codigo real) a partir de la dictado del usuario + prototipo + Capa 2
---

# Modulo de Tareas - Creacion y ejecucion de actividades

> Capitulo de construccion. El objetivo es que crear una actividad quede **milimetricamente
> igual al prototipo** (`01. Requerimiento/Prototipo/ECOREX.dc.html`) y que la actividad
> **consuma el concepto** (categoria/subcategoria) para arrancar su flujo, su formulario y su
> tablero, con la asignacion por cargo. Los motores ya existen; **falta el puente**.

## 1. Hallazgo central (auditoria del codigo real, 2026-07-11)

Los cimientos estan construidos y son solidos; lo que falta es CONECTARLOS al alta de tarea y
al menu:

| Cimiento | Estado real | Rutas |
|----------|-------------|-------|
| Conceptos 2 niveles (Categoria/Subcategoria) con FKs a flujo/formulario/tablero+columna y M:N a cargos/terceros/notifs + flags (`IniciaModulo`, `RequiereCliente`, `CierreManual`, `TituloAuto`, `DetalleAuto`) | **Construido** (config UI completa en Conceptos.razor) | `Ecorex.Domain/Entities/ActividadCategoria.cs`, `ActividadSubcategoria.cs`; `Ecorex.Application/Actividades/ActividadCatalogoService.cs`; `Components/Pages/Conceptos.razor` |
| Runtime de flujos (nodo->cargo, cargo->usuarios, bandeja reclamar/atender/aprobar, motor Advance/Reject) | **Construido y maduro** | `WorkflowEngine.cs`, `WorkflowInboxService.cs`, `WorkflowNodePolicy.cs`, `OrgAssigneeTree.cs`, `INodeAssigneeResolver.cs`; embebido en `TaskDetailModal.razor` + pagina `/mis-pasos` |
| Organigrama (Dependencia->Cargo->Funcionario, responsable de unidad) | **Construido** | `OrgUnit.cs` (`Classifier`, `TenantUserId`, `ResponsibleTenantUserId`), `OrgUnitService.cs`, `Dependencias.razor` |
| Motor de Formularios dinamicos (definir/render/guardar; submit completa paso de flujo) | **Construido y usable** (14+ tipos; multimedia/firma/GPS son placeholders) | `FormDefinition/Response.cs`, `FormResponseService.cs`, `DynamicFormRenderer.razor`, `FormDesigner.razor` |
| Menu lateral (vistas/nodos por tenant, editor, poda por permiso) | **Construido, data-driven** | `MenuConfigService.cs`, `MenuTreeBuilder.cs`, `MenuPermissionFilter.cs`, `NavMenu.razor`, `ConfiguracionMenu.razor`; entidades `MenuView.cs`/`MenuNode.cs` |

## 2. Las brechas (lo que hay que construir = "el puente")

1. **DOS modelos de "tipo" desconectados.** La tarea (`TaskItem`) se clasifica hoy por
   `ActivityType` (Categoria=texto + Nombre). El catalogo rico **`ActividadSubcategoria`
   (Concepto)** existe pero **no interviene en la creacion de tareas**. Hay que decidir como
   unificarlos (ver [[01 - Arquitectura, decisiones y consumo]] D1). Es LA decision que
   gobierna todo el modulo.
2. **El alta de tarea no consume el concepto.** `TaskWizard.razor` (3 pasos) no lee flags ni
   FKs del concepto: no arranca el flujo (`WorkflowEngine.StartInstance`), no abre el
   formulario (`IniciaModulo`), no asigna por cargo, no usa el tablero del concepto.
3. **Empresa/Area del modal: fuente RESUELTA (PRE-1, 2026-07-11), falta cablearla.** El prototipo
   pide un selector Empresa/Area; hoy el `TaskItem` no tiene FK (la "Categoria" es texto de
   `ActivityType`). **La fuente ya existe**: el modulo "Configuracion de la entidad" (000616,
   `/configuracion-entidad`) administra las `Entidad` del tenant con tipo `Sede` o `Area`, y
   `IEntidadService.ListOptionsAsync()` las lista para el combo. Ojo: es distinto de
   `OrgUnit.Kind=Area` (organigrama 000850), que se usa para asignar pasos por cargo, NO para el
   selector. Pendiente: agregar `TaskItem.EntidadId` (Ola 1) y el combo en el modal (Ola 3).
4. **El menu "Mis Procesos" es estatico**, sembrado en `menu_nodes` (DatabaseSeeder ~2551). El
   objetivo (Mis Procesos -> categoria -> subcategoria -> tablero, generado desde Conceptos) es
   un **menu dinamico nuevo**, mas la opcion en el editor de menu para elegir "que grupo muestra
   las actividades tipo proceso". El fundamento (**PRE-5**) ya esta RESUELTO (2026-07-11, commit
   `66bb60d`): `MenuNode.IsProcessGroup` + checkbox en el editor + badge "procesos" en el sidebar.
   Falta solo el render dinamico (expandir con categorias/subcategorias) = **Ola 4**.
5. **Fidelidad del modal.** El prototipo tiene un wizard de **4 pasos** (Informacion / Contacto /
   Formulario / Documentos) con columna de resumen; el actual es de 3 sin formulario ni
   documentos. Hay que replicarlo (ver [[02 - UX y fidelidad (modal, menu, tableros)]]).
6. **Encargado: marca de jefe/responsable por miembro (Dependencias) = PRE-4. RESUELTO** (2026-07-11,
   commit `66bb60d`): `OrgUnitMember.IsResponsible` (uno por unidad, sincroniza
   `OrgUnit.ResponsibleTenantUserId`) + boton/badge en Dependencias. La Ola 3 lo usara como Encargado
   por defecto del alta.

## 3. Los dos caminos de creacion (segun el dictado)

- **(A) Actividad-proceso**: la que tiene flujo vinculado en el concepto. Se llega desde el menu
  **Mis Procesos -> categoria -> subcategoria**, que carga el **tablero especifico + el modal**
  ya con categoria/subcategoria fijadas (no las vuelve a preguntar). Puede arrancar con un
  **formulario** si la subcategoria tiene `IniciaModulo` (permiso, cotizacion...).
- **(B) Actividad simple**: la que NO tiene flujo. Se crea desde un **tablero**, con el modal
  **Empresa/Area -> Tipo (categoria) -> Actividad (subcategoria) -> Encargado -> descripcion**.
  En el tablero/modal solo se listan las actividades (subcategorias) sin proceso.

Detalle de ambos en [[02 - UX y fidelidad (modal, menu, tableros)]].

## 4. Documentos del capitulo

- [[01 - Arquitectura, decisiones y consumo]] - el puente: decisiones (unificacion de tipos,
  empresa/area, menu dinamico), y como el alta consume el concepto (flujo + formulario + cargo).
- [[02 - UX y fidelidad (modal, menu, tableros)]] - modal wizard 4 pasos (tokens del prototipo),
  menu Mis Procesos dinamico, tableros y los 2 caminos.
- [[03 - Plan por olas y preguntas abiertas]] - backlog verificable + decisiones a cerrar con el
  usuario antes de delegar a un sub-agente.

## 5. Decisiones tomadas (usuario, 2026-07-11) y alcance

- **Pivotar la tarea a Conceptos** (deprecar `ActivityType`) - D1.
- **Empresa/Area sale del modulo Configuracion de la entidad (000616)** - D2; **PRE-1 RESUELTO**
  el 2026-07-11 (entidad `Entidad` con tipo Sede/Area + `ListOptionsAsync` para el combo; ver
  doc 03). Falta desplegar sus migraciones a prod y cablear el FK + el combo en el modal.
- **Menu Mis Procesos dinamico** desde Conceptos - D3.
- **Form-first en v1** (arranque con formulario cuando `IniciaModulo`).
- **Alcance = Actividades + Proyectos JUNTOS** (Proyectos en olas P1-P3 aparte para no mezclar
  riesgos). Plano del ETL de Proyectos: [[Tareas y Proyectos - paginas basicas]] s.2.

## 6. Veredicto de delegacion

Delegable a sub-agente **por olas** (ver doc 03), tras resolver los **prerequisitos** (sobre todo
PRE-1 Areas). Es un alcance grande (Actividades + Proyectos); conviene **un sub-agente por linea**
(Actividades y Proyectos) en worktrees separados para minimizar conflictos, coordinados por el
orquestador. Fidelidad: los tokens del `ECOREX.dc.html` mandan (modal ~4280-4438; menu "Mis
Procesos"/groupDefs ~4711).

Relacionado: [[Tareas y Proyectos - paginas basicas]], [[ctrTareasII - Spec para reconstruir en Claude Design]],
[[ctrVertareasII - Spec para reconstruir en Claude Design]], [[AdmWorkflow - Motor de flujo de la tarea]].
