---
tipo: indice-proyecto
proyecto: Modulo de Tareas - Creacion y ejecucion de actividades
modulos_web: /actividades (000636), /crear-actividad (000038), /proyectos (000042), /conceptos (000270), /dependencias (000850), /flujos (000291), /formularios (000131), /configuracion-menu (000194)
estado: DESPLEGADO A PROD (2026-07-11) - Actividades Olas 1-7 + Proyectos P1/P3
fecha: 2026-07-11
autor: documentado por agente IA (4 exploradores read-only sobre el codigo real) a partir de la dictado del usuario + prototipo + Capa 2
---

# Modulo de Tareas - Creacion y ejecucion de actividades

> Capitulo de construccion. El objetivo es que crear una actividad quede **milimetricamente
> igual al prototipo** (`01. Requerimiento/Prototipo/ECOREX.dc.html`) y que la actividad
> **consuma el concepto** (categoria/subcategoria) para arrancar su flujo, su formulario y su
> tablero, con la asignacion por cargo. Los motores ya existian; **el puente YA SE CONSTRUYO**.

> [!success] ESTADO 2026-07-11: DESPLEGADO A PROD (Olas 1-7 + Proyectos P1/P3)
> Los 5 prerequisitos (PRE-1..PRE-5), las 7 olas de Actividades y Proyectos P1 (hitos) + P3 (enlace
> actividad<->hito) estan implementados y **DESPLEGADOS a prod** (http://10.0.0.3:5480; 6 migraciones
> aplicadas el 2026-07-11, backup previo). Tests de integracion verdes (PG) + validacion en Chrome. El
> detalle por ola/P, commits y el INVENTARIO DE PENDIENTES estan en
> [[03 - Plan por olas y preguntas abiertas]] (fuente de verdad del avance). La seccion 2 de este doc
> describe las brechas TAL COMO ESTABAN al inicio; cada una lleva ahora su marca de resuelta.
> **Pendientes principales** (ver inventario en doc 03): (1) cablear la config de Conceptos en prod via
> el editor (flujo/form/tablero por subcategoria); (2) vistas de menu de los demas usuarios reales;
> (3) DAL-dual SQL Server; (4) backlog de endurecimiento (email+plantilla, badge en vivo, policies de
> gobierno) y de Proyectos (finanzas/DOFA, timeline).

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

## 2. Las brechas (como estaban al inicio) -- TODAS CERRADAS 2026-07-11

> Se dejan descritas como estaban para trazabilidad; cada una lleva su marca de resuelta y el commit.

1. ~~**DOS modelos de "tipo" desconectados.**~~ **RESUELTO (Ola 1, `a60252e`).** `TaskItem` pivota a
   `SubcategoriaId` (concepto) + `EntidadId`; `ActivityTypeId` queda nullable/deprecado (D1). El alta
   exige >=1 clasificacion.
2. ~~**El alta de tarea no consume el concepto.**~~ **RESUELTO (Olas 2-3-5).** El nuevo `TaskWizard`
   (4 pasos) lee flags/FKs del concepto: arranca el flujo (`WorkflowDefinitionId`, Ola 2), aplica
   `TituloAuto`/`DetalleAuto` (Ola 2), asigna por cargo (`ListEncargadoUserIdsAsync`, Ola 3), abre el
   formulario si `IniciaModulo` (form-first, Ola 5) y hereda el tablero del concepto (Ola 1/6).
3. ~~**Empresa/Area del modal.**~~ **RESUELTO.** Fuente = `Entidad` (Sede/Area) de Config de la entidad
   (000616), PRE-1 (`5590545`); FK `TaskItem.EntidadId` (Ola 1) + combo en el modal (Ola 3).
4. ~~**El menu "Mis Procesos" es estatico.**~~ **RESUELTO (Ola 4, `8de3521`).** `NavMenu` expande el
   grupo `IsProcessGroup` (PRE-5, `66bb60d`) con el arbol dinamico categoria->subcategoria-proceso
   desde Conceptos; cada subcategoria linkea al tablero/alta con cat/sub fijadas.
5. ~~**Fidelidad del modal.**~~ **RESUELTO (Ola 3, `9af2202`).** `TaskWizard.razor` reescrito al wizard
   de 4 pasos (Informacion/Contacto/Formulario/Documentos + aside resumen), milimetrico a
   `ECOREX.dc.html` (tokens `--surface/--ink/--t-*/--sh-*`).
6. ~~**Encargado: marca de jefe/responsable.**~~ **RESUELTO** (PRE-4, `66bb60d`):
   `OrgUnitMember.IsResponsible` (uno por unidad, sincroniza `OrgUnit.ResponsibleTenantUserId`); la Ola 3
   lo usa como Encargado por defecto y la Ola 7 (`7111cbb`) notifica al asignar.

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

**Actividades: TERMINADO en local (Olas 1-7 HECHAS 2026-07-11, ver doc 03).** Queda el pendiente
operativo de **desplegar a prod** las migraciones acumuladas + la config demo via editor de Conceptos,
y el **DIFERIDO de endurecimiento** (policies compuestas por vista + entrega real de notificaciones).
**Proyectos: sin empezar** (olas P1-P3); conviene **un sub-agente** en worktree aparte para minimizar
conflictos. Fidelidad: los tokens del `ECOREX.dc.html` mandan (modal ~4280-4438; menu "Mis
Procesos"/groupDefs ~4711).

Relacionado: [[Tareas y Proyectos - paginas basicas]], [[ctrTareasII - Spec para reconstruir en Claude Design]],
[[ctrVertareasII - Spec para reconstruir en Claude Design]], [[AdmWorkflow - Motor de flujo de la tarea]].
