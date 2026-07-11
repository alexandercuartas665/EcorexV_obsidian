---
tipo: plan-construccion
proyecto: Modulo de Tareas
proposito: Backlog por olas con criterios de aceptacion + las decisiones a cerrar con el usuario antes de delegar a un sub-agente.
---

# 03 - Plan por olas y preguntas abiertas

## A. Preguntas / decisiones a cerrar ANTES de delegar (bloqueantes)

1. **[D1] Unificacion de tipos.** Confirmar: pivotar `TaskItem` a `ActividadSubcategoria`
   (concepto) y deprecar `ActivityType`? (recomendado). Define el esquema de todo el modulo.
2. **[D2] Empresa/Area.** Confirmar: el selector "Empresa/Area" sale del organigrama `OrgUnit`
   (Kind=Area/Dependencia) (recomendado), y no de "Configuracion de entidad" (000615). Si se
   quiere un catalogo de sedes propio, es alcance extra.
3. **[D3] Menu Mis Procesos.** Confirmar: grupo dinamico marcado en el editor de menu que expande
   categorias/subcategorias (proceso) desde Conceptos (recomendado), y "actividad tipo proceso" =
   subcategoria con `WorkflowDefinitionId != null`.
4. **[Encargado]** Basta con `OrgUnit.ResponsibleTenantUserId` (encargado por unidad), o se quiere
   un flag "principal" por miembro en `OrgUnitMember`?
5. **[Alcance v1]** El arranque form-first (`IniciaModulo`) entra en v1 o backlog? (el motor de
   formularios existe; falta cablearlo al wizard).
6. **[Proyectos]** El modulo Proyectos (000042, ~20 tablas legacy) se aborda en este capitulo o
   aparte? (recomendado: aparte; aqui foco en actividades).

## B. Plan por olas (cada una entregable y verificable)

Reglas transversales (toda ola): multi-tenant real; DAL dual PG/SQL Server; SQL parametrizado;
transaccion en operaciones multi-tabla; soft-delete + auditoria; concurrencia optimista; SignalR
para tiempo real; permisos como policies; ASCII en archivos nuevos; PROGRESO.md por sesion; tests
(unit + integracion dual). Fidelidad milimetrica al prototipo.

### Ola 0 - Cerrar decisiones + spec fina
- Resolver A.1-A.6 con el usuario. Extraer del `ECOREX.dc.html` los tokens exactos del modal y del
  menu. **Aceptacion**: decisiones firmadas + hoja de tokens.

### Ola 1 - Datos: puente Concepto <-> Tarea
- `TaskItem`: agregar `SubcategoriaId` (FK ActividadSubcategoria) + `AreaOrgUnitId` (FK OrgUnit);
  migracion EF + backfill de tareas existentes. Ajustar `TaskItemService.CreateAsync` para aceptar
  subcategoria y derivar board/columna/flujo/flags del concepto.
- **Aceptacion**: se puede crear un TaskItem ligado a una subcategoria; el board/columna se toman
  del concepto; tests unit del servicio verdes; migracion aplica en PG y SQL Server.

### Ola 2 - Alta que CONSUME el concepto (sin UI nueva aun)
- En `CreateAsync` (transaccional): si la subcategoria tiene `WorkflowDefinitionId`, llamar
  `WorkflowEngine.StartInstance`; aplicar `TituloAuto/DetalleAuto`; disparar notificaciones a
  `Subcategoria.Notificaciones`.
- **Aceptacion**: crear una actividad-proceso arranca su `WorkflowInstance` con el primer paso
  pendiente asignado por cargo (visible en `/mis-pasos`); una simple no crea instancia. Test de
  integracion del alta + StartInstance en la misma transaccion.

### Ola 3 - Modal wizard 4 pasos (fidelidad prototipo)
- Reescribir `TaskWizard.razor` al wizard de 4 pasos del prototipo: Informacion (Empresa/Area ->
  Tipo -> Actividad -> Encargado -> Fecha + Descripcion + Prioridad + Etiquetas), Contacto,
  Formulario (condicional, ver Ola 5), Documentos; columna de resumen; footer con "Guardar y crear
  otra" (reusar patron del Contenedor de datos).
- Cascada real: Tipo(categoria) -> Actividad(subcategoria) -> Encargado (cargos + responsable).
- **Aceptacion**: el modal se ve milimetrico al prototipo (validacion visual contra el HTML), la
  cascada funciona, crea la actividad por ambos caminos, "Guardar y crear otra" no cierra.

### Ola 4 - Menu "Mis Procesos" dinamico + editor
- `MenuNode`: soporte de grupo dinamico "muestra actividades tipo proceso"; `ConfiguracionMenu`
  gana la opcion para marcar el grupo. `NavMenu` expande ese grupo con categorias/subcategorias
  (proceso) desde `IActividadCatalogoService`. Al clic: carga el tablero del concepto + abre el
  modal con categoria/subcategoria fijadas.
- **Aceptacion**: en el menu aparece Mis Procesos -> categoria -> subcategoria (dinamico, como la
  imagen); al entrar carga el tablero correcto y el modal no vuelve a pedir categoria/subcategoria;
  la poda por permisos sigue funcionando.

### Ola 5 - Arranque form-first (si entra en v1)
- Paso "Formulario" del wizard: si la subcategoria tiene `IniciaModulo` + `FormDefinitionId`,
  renderizar `DynamicFormRenderer` (Fill) + `FormResponseService` (draft->submit); enlazar la
  respuesta a la tarea/paso.
- **Aceptacion**: una subcategoria "inicia modulo" abre el formulario al crear; al enviar, la
  tarea queda creada con la respuesta y el flujo continua.

### Ola 6 - Tableros, tabs y tiempo real
- Tabs No Asignados/Asignados/Todos/Kanban con conteos vivos por SignalR; drag&drop kanban;
  preseleccion `?cat=&sub=`; lista de "crear desde tablero" solo con actividades sin proceso.
- **Aceptacion**: mover una tarjeta se propaga a otros clientes al instante; los conteos son
  vivos; los filtros/preseleccion funcionan.

### Ola 7 - Endurecimiento
- Deuda del encargado (flag principal si se decidio), consecutivos transaccionales, notificaciones
  (plantilla), policies compuestas por vista (multi-permiso del legacy), auditoria.
- **Aceptacion**: pruebas de permisos, concurrencia, y notificacion al asignar.

## C. Backlog (post v1)
- Modulo Proyectos (000042) completo (cabecera, hitos, presupuesto, ACL por proyecto).
- Catalogo de Sedes/Empresa-cliente propio (si se decide en D2).
- Tipos de control multimedia del formulario (foto/firma/GPS/archivo/barcode) hoy placeholder.
- Vista calendario / Gantt de actividades; vista de tareas para cliente final.

## D. Riesgos
- **Migracion `ActivityType` -> Subcategoria**: hay tareas existentes con `ActivityTypeId`;
  definir el backfill (mapear o dejar null) para no perder datos.
- **Doble modelo si no se decide D1**: mantener ambos "tipos" a la vez genera inconsistencias.
- **Fidelidad**: sin extraer tokens exactos del prototipo, un sub-agente puede desviarse; la Ola 0
  (hoja de tokens) mitiga.
- **Alcance**: Proyectos y multimedia pueden inflar el modulo; mantenerlos en backlog.
