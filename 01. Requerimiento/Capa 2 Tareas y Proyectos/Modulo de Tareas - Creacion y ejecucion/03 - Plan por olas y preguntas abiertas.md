---
tipo: plan-construccion
proyecto: Modulo de Tareas
proposito: Backlog por olas con criterios de aceptacion + las decisiones a cerrar con el usuario antes de delegar a un sub-agente.
---

# 03 - Plan por olas y preguntas abiertas

## A. Decisiones tomadas (usuario, 2026-07-11)

- **[D1] Unificacion de tipos = PIVOTAR A CONCEPTOS.** `TaskItem` referencia
  `ActividadSubcategoria` (concepto) y se deprecia `ActivityType`. Requiere migracion + backfill.
- **[D2] Empresa/Area = MODULO CONFIGURACION DE LA ENTIDAD (000616).** El dato sale del modulo de
  configuracion de la entidad (no del organigrama directamente). Prerequisito **RESUELTO** el
  2026-07-11 (ver PRE-1). La fuente concreta es la entidad `Entidad` via
  `IEntidadService.ListOptionsAsync()`.
- **[D3] Menu Mis Procesos = grupo dinamico** desde Conceptos (categoria->subcategoria con flujo),
  marcado en el editor de menu. "Actividad tipo proceso" = subcategoria con `WorkflowDefinitionId`.
- **[Form-first] = SI, EN V1.** El paso "Formulario" del wizard se cablea con `DynamicFormRenderer`.
- **[Alcance] = ACTIVIDADES + PROYECTOS JUNTOS.** El modulo Proyectos (000042) entra en esta gran
  tarea (ver olas P1-P3). Es un alcance grande; olas separadas para no mezclar riesgos.
- **[Encargado]** El usuario SI quiere la marca de jefe/responsable por miembro -> ahora es
  **PRE-4** (ya no "no bloqueante"). Usar ese flag como Encargado por defecto y reconciliar con
  `OrgUnit.ResponsibleTenantUserId`.

## A2. Prerequisitos - checklist (resolver ANTES de las olas del modal)

Tareas previas, cada una verificable, que desbloquean el modulo:

- [x] **PRE-1 Areas en la config de la empresa (D2). RESUELTO 2026-07-11** (commit `5590545`;
      modulo "Configuracion de la entidad" 000616, pagina `/configuracion-entidad`). Se creo la
      entidad `Entidad` (legacy SUCURSAL_R = agencias/areas/sedes del tenant, distinta del tenant
      mismo = "Mi cuenta" = SUCURSAL) con un discriminador `EntidadKind { Sede, Area }`: el modal
      **pregunta primero el tipo** y adapta los campos (Sede = identidad legal + ubicacion + logo +
      config; Area = solo nombre / sigla / responsable / contacto + config). CRUD administrable
      (crear / editar / archivar / marcar principal) + **campos dinamicos por tenant**
      (`EntidadFieldDefinition`, mismo patron que las fichas del Directorio). El servicio
      **`IEntidadService.ListOptionsAsync()`** (devuelve `EntidadOptionDto { Id, Label }`) es la
      fuente lista-para-combo del selector "Empresa/Area" del alta.
      - **PENDIENTE de despliegue:** las migraciones `AddEntidadConfig` + `AddEntidadKind` estan
        aplicadas SOLO en local; hay que desplegarlas a prod antes de que el modulo de Tareas las
        consuma en produccion.
      - **Reconciliacion (clave para la Ola 1):** existen DOS conceptos de "area" y se quedan
        SEPARADOS a proposito:
        - **(a) `Entidad.Kind=Area`** = area/agencia/sede organizativa de la cuenta. Es el ORIGEN
          del selector "Empresa/Area" del alta de actividades. El FK del `TaskItem` apunta aqui
          (`EntidadId -> Entidad`).
        - **(b) `OrgUnit.Kind=Area`** (Dependencias / organigrama 000850) = nodo del organigrama
          para **asignar los pasos por cargo -> funcionario**. NO es lo que elige el usuario en el
          alta; lo consume el runtime del flujo del concepto.
        => **Correccion al plan de la Ola 1:** el FK nuevo del `TaskItem` es **`EntidadId -> Entidad`**
        (NO `AreaOrgUnitId -> OrgUnit`, como decia el borrador). La asignacion de responsables de
        pasos sigue saliendo del organigrama a traves del flujo, no de este FK.
- [x] **PRE-2 Consumidor de los flags/FKs del concepto. RESUELTO 2026-07-11 -> mapa de lectores VACIO.**
      Auditoria de codigo: los flags/FKs de `ActividadSubcategoria` (`IniciaModulo`, `TituloAuto`,
      `DetalleAuto`, `RequiereCliente`, `CierreManual`, `WorkflowDefinitionId`, `FormDefinitionId`,
      `TaskBoardId`, `TaskBoardColumnId`, `Sedes`, `Cargos`, `Terceros`, `Notificaciones`) solo se
      leen/escriben en el **modulo de config** (`Conceptos.razor` <-> `ActividadCatalogoService` +
      `ActividadDtos`) y en el snapshot/migraciones EF. **`TaskItemService` NO referencia el concepto**
      (el unico hit de "Subcategoria" en `TerceroService` es un string de nota de contacto, no el
      concepto). Confirmado en Chrome: el alta (`TaskWizard`) clasifica por **ActivityType** -- el
      selector "Categoria" ofrece "Direccion Comercial / Direccion General / Gestion Humana"
      (categorias de ActivityType), NO las del concepto -- y no ofrece subcategorias. => Este modulo
      sera el PRIMER consumidor; no hay nada que romper.
- [x] **PRE-3 Backfill de tareas existentes. RESUELTO 2026-07-11 -> plan revisado.** Datos reales
      (BD demo local): **206 tareas, todas con `ActivityTypeId`** (FK NOT NULL). 4 tipos en uso:
      Gestion Humana/Solicitud (116), Direccion Comercial/Cotizacion (88), Direccion Comercial/
      Seguimiento (1), Direccion General/Requerimiento (1). Catalogo de Conceptos: 4 categorias
      (Comercial / Operaciones / Gestion Humana / Financiera) + 8 subcategorias, **ninguna con flujo
      aun**. **Match exacto (Category,Name) -> (Categoria,Subcategoria) = 0 de 206**: los dos seeds
      divergieron ("Direccion Comercial" != "Comercial"; "Cotizacion" != "Cotizacion de equipos";
      "Solicitud" no tiene equivalente). **DECISION de backfill:** `TaskItem.SubcategoriaId` NULLABLE;
      las tareas existentes -> **NULL** (NO fabricar mapeo sobre datos demo divergentes). Conservar
      `ActivityTypeId` durante la transicion (deprecar, no dropear, en Ola 1). Migracion no
      destructiva; las tareas nuevas (post Ola 3) reciben `SubcategoriaId` real. Si en PROD hubiera
      datos reales que ameriten clasificar la historia, se hara un **mapa curado manual** (tabla de
      equivalencias), nunca automatico por nombre.
- [x] **PRE-4 Marca de "jefe / responsable" por miembro en Dependencias (000850). RESUELTO
      2026-07-11** (commit `66bb60d`). Se agrego `OrgUnitMember.IsResponsible`: a lo sumo un jefe por
      unidad; al marcarlo se sincroniza `OrgUnit.ResponsibleTenantUserId` con ese usuario, al
      desmarcarlo/quitarlo se limpia (reconciliacion resuelta: el flag por miembro es la fuente y la
      unidad refleja). Nuevo `IOrgUnitService.SetMemberResponsibleAsync` (multi-tabla en una
      transaccion) + boton "Hacer jefe" / "Jefe" y badge en `Dependencias.razor`. Sera el Encargado
      propuesto por defecto al crear actividades (lo consumira la Ola 3). Validado en Chrome:
      marcar a un miembro como jefe reasigna el Responsable de la unidad y desmarca a los demas.
- [x] **PRE-5 Item de menu "despliega procesos" (grupo dinamico) - fundamento de datos. RESUELTO
      2026-07-11** (commit `66bb60d`). Se agrego `MenuNode.IsProcessGroup` (flag, no un `Kind`
      nuevo): marca un grupo (Section/Subgroup) como "despliega los procesos". Cableado de punta a
      punta: editor (`MenuEditorNodeDto`/`MenuNodeEditDto` + checkbox en `ConfiguracionMenu.razor`),
      sidebar (`MenuNodeDto`/`MenuTreeBuilder.FlatNode` + badge "procesos" en `NavMenu`), clon y
      export/import. El **render dinamico real** (expandir con categorias/subcategorias de proceso
      via `IActividadCatalogoService`) sigue siendo la **Ola 4**; este PRE es el esquema + editor.
      Validado en Chrome: se marca el grupo, persiste tras recargar y el sidebar muestra el badge
      PROCESOS.

## B. Plan por olas (cada una entregable y verificable)

Reglas transversales (toda ola): multi-tenant real; DAL dual PG/SQL Server; SQL parametrizado;
transaccion en operaciones multi-tabla; soft-delete + auditoria; concurrencia optimista; SignalR
para tiempo real; permisos como policies; ASCII en archivos nuevos; PROGRESO.md por sesion; tests
(unit + integracion dual). Fidelidad milimetrica al prototipo.

### Ola 0 - Cerrar decisiones + spec fina
- Resolver A.1-A.6 con el usuario. Extraer del `ECOREX.dc.html` los tokens exactos del modal y del
  menu. **Aceptacion**: decisiones firmadas + hoja de tokens.

### Ola 1 - Datos: puente Concepto <-> Tarea  -- HECHA 2026-07-11 (commit `a60252e`)
- `TaskItem`: `ActivityTypeId` pasa a NULLABLE (deprecado D1) + `SubcategoriaId` (FK
  ActividadSubcategoria) + `EntidadId` (FK `Entidad`, Empresa/Area; NO `OrgUnit`), ambos nullable,
  FK Restrict. Migracion `TaskItemConceptoBridge` (PG; SqlServer sigue en backlog DAL-dual).
- `CreateAsync`: exige al menos una clasificacion (ActivityType o concepto); valida subcategoria/
  entidad; si el request no fijo tablero y el concepto tiene `TaskBoardId`, la tarea hereda ese
  tablero y cae en su PRIMERA columna (no la "terminado"). `UpdateAsync`/`ListAsync` + DTOs (summary
  + filtro) tambien aceptan concepto/entidad. El arranque de flujo desde el concepto es la Ola 2.
- Backfill PRE-3 aplicado: las 206 tareas existentes quedan con `SubcategoriaId`/`EntidadId` NULL y
  conservan `ActivityTypeId`.
- **Aceptacion CUMPLIDA**: tests de integracion (matriz dual, verdes en PG) -- alta por concepto
  hereda tablero+primera columna, y "sin clasificacion -> Invalid". Regresion en Chrome: el alta
  legacy por ActivityType sigue creando tareas (T00207). Migracion aplicada en local (falta prod +
  SqlServer). NOTA: el alta por concepto aun no tiene UI (llega en la Ola 3, el modal).

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

## B2. Olas de PROYECTOS (en alcance por decision del usuario)

Se corren DESPUES de la base de actividades (o en paralelo por un sub-agente distinto en worktree,
ya que Proyectos toca entidades propias `PROYECTOS_*`/`DOC_PROYECTOS_*`). Plano del ETL legacy en
[[Tareas y Proyectos - paginas basicas]] seccion 2 (~20 tablas) y destino en su seccion 0.

### Ola P1 - Dominio + cabecera de proyecto
- Modelar las tablas legacy `DOC_PROYECTOS` (cabecera) + `PROYECTOS_HITO`, `PROYECTOS_PRESUPUESTO`,
  `PROYECTOS_COS` (costos), `PROYECTOS_DOFA`, `PROYECTOS_RES` (ACL) como entidades EF Core
  `ITenantScoped`; migracion dual. Servicio + DTOs de proyecto.
- **Aceptacion**: CRUD de proyecto (cabecera + hitos + presupuesto) tenant-scoped, con soft-delete
  y auditoria; migracion aplica en PG y SQL Server.

### Ola P2 - UI Proyecto (ProyectoDetalle) + ACL por proyecto
- `Proyectos.razor` (lista/filtro) + `ProyectoDetalle.razor` (cabecera Estado/Asignados/Fecha/
  Etiquetas + tabs Vista Tablero / Timeline / Calendario, fidelidad prototipo). ACL por proyecto
  via authorization handler por recurso (`PROYECTOS_RES.FLAG_ED/FLAG_VER` -> policy por recurso),
  NO chequeo ad-hoc. Las tareas del proyecto son `TaskItem` con `ProjectId`.
- **Aceptacion**: ver/editar/borrar respeta el ACL por proyecto; el tablero del proyecto muestra
  sus tareas; borrar sin FLAG_ED se bloquea.

### Ola P3 - Integracion Actividades <-> Proyectos + tiempo real
- Vincular actividades a proyecto e hito (paso 2 del wizard: Proyecto / Hito del proyecto);
  conteos/tableros del proyecto por SignalR; timeline/calendario.
- **Aceptacion**: crear una actividad dentro de un proyecto la enlaza a su hito y aparece en el
  tablero del proyecto en tiempo real.

## C. Backlog (post v1)
- Catalogo de Sedes/Empresa-cliente adicional (mas alla de Areas del tenant), si se necesita.
- Tipos de control multimedia del formulario (foto/firma/GPS/archivo/barcode) hoy placeholder.
- Vista de tareas para cliente final (interfaz simplificada de solo lectura).
- Modulos satelite del legacy (PQR, encuesta de servicio, visitas) si aplican al negocio.

## D. Riesgos
- **Migracion `ActivityType` -> Subcategoria**: RESUELTO por PRE-3 -> `SubcategoriaId` nullable, las
  206 tareas existentes quedan en NULL (0 auto-match real), se conserva `ActivityTypeId` en la
  transicion. Sin perdida de datos.
- **Doble modelo si no se decide D1**: mantener ambos "tipos" a la vez genera inconsistencias.
- **Fidelidad**: sin extraer tokens exactos del prototipo, un sub-agente puede desviarse; la Ola 0
  (hoja de tokens) mitiga.
- **Alcance**: Proyectos y multimedia pueden inflar el modulo; mantenerlos en backlog.
