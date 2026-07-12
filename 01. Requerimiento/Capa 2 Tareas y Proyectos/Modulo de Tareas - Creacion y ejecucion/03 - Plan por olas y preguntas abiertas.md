---
tipo: plan-construccion
proyecto: Modulo de Tareas
proposito: Backlog por olas con criterios de aceptacion + las decisiones a cerrar con el usuario antes de delegar a un sub-agente.
---

# 03 - Plan por olas y preguntas abiertas

## ESTADO GENERAL (2026-07-11)

> [!success] DESPLEGADO A PRODUCCION 2026-07-11
> **Actividades (Olas 1-7) y Proyectos (P1 hitos + P3 enlace) estan CONSTRUIDOS y DESPLEGADOS a prod**
> (`root@10.0.0.3`, http://10.0.0.3:5480). Se aplicaron las 6 migraciones pendientes en el arranque:
> `AddEntidadConfig`, `AddEntidadKind`, `AddJefeMemberAndProcessGroupMenu`, `TaskItemConceptoBridge`,
> `AddNotifications`, `AddProjectMilestones` (backup previo `ecorex-2026-07-11-1631.sql.gz`). App sana
> (HTTP 200 /login). Codigo en `fase-0/clon-backbone` @ `e6636d8` (incluye el fix movil del wizard,
> redeploy 2026-07-12).

> [!note] QA VISUAL 2026-07-11 (desktop + movil 375px)
> Se valido como se VE el modulo (estilos/layout/overflow, no solo BD) en desktop y movil. OK: panel de
> Hitos, pagina `/notificaciones` y badge de la campana (desktop y movil, sin overflow). **Se encontro y
> ARREGLO 1 problema**: el wizard de alta (2 columnas + grids) no colapsaba en movil (~37px de overflow);
> ahora apila contenido+resumen y colapsa los grids a 1 columna (<=700px), desktop intacto (commit `e6636d8`).
> El kanban embebido scrollea dentro de su propio contenedor (correcto). **DESPLEGADO a prod 2026-07-12**
> (`e6636d8`, build-from-git + up -d, backup `ecorex-2026-07-12-0510.sql.gz`; CSS `.tk-wz-body` verificado
> en prod). Prod ya tiene el wizard responsive.

### Terminado (en local Y prod)
- **Prerequisitos PRE-1..PRE-5** (Empresa/Area, mapa de consumidores, backfill, jefe/responsable, flag de menu).
- **Olas 1-7 de Actividades**: puente concepto<->tarea, alta que consume el concepto (flujo+auto-titulo+
  notif), wizard 4 pasos milimetrico, menu Mis Procesos dinamico, form-first, tableros, y endurecimiento
  (policies compuestas por vista + entrega real de notificaciones in-app).
- **Proyectos P1 (hitos) + P3 (enlace actividad<->proyecto/hito)**. La cabecera + UI de proyecto (P2) ya existia.

### INVENTARIO DE PENDIENTES (al 2026-07-11)
1. **Config de Conceptos en prod -- HECHO 2026-07-11 (por el editor).** Se cablearon en prod (SKY SYSTEM)
   via el editor de Conceptos, apuntando el dev local (localhost, sin bloqueo) a la BD de prod por el tunel:
   - **Cotizacion de equipos** -> flujo "Cotizacion Comercial" (publicado) + tablero "Comercial -
     Requerimiento Infraestructura".
   - **Requerimiento infraestructura** -> formulario "Solicitud de cotizacion" (form-first).
   Verificado en la BD de prod.
   **Ampliacion 2026-07-11 (por el editor):** se ligo tambien **Operaciones / Visita tecnica** -> flujo
   "Visita tecnica de instalacion" + tablero "Soporte - Mesa de ayuda" (2do concepto-proceso). Y se marco
   en el EDITOR DE MENU el nodo "Mis Procesos" de la vista **Completo** como `IsProcessGroup=true`, de modo
   que el menu dinamico "Mis Procesos" ahora EXPANDE los procesos (verificado: sidebar muestra links
   `actividades?sub=` a "Cotizacion de equipos" y "Visita tecnica"). Tope actual = 2 conceptos-proceso
   porque prod solo tiene 2 flujos PUBLICADOS (Cotizacion Comercial, Visita tecnica de instalacion); para
   mas procesos hay que PUBLICAR mas flujos en el editor BPMN. El resto de sub-categorias siguen sin proceso
   a proposito.
2. ~~**Vistas de menu de los usuarios reales**~~ **RESUELTO 2026-07-12** (`b5964b4`): los 12 usuarios BITCODE
   sin vista YA heredan la "Completo" clonada por el fallback de resolucion; ademas se mejoro ese fallback
   para elegir la vista con MAS nodos visibles cuando el tenant no tiene ninguna IsDefault. Test verde.
   (El seed sistematico de una Completo por cada tenant nuevo sigue como mejora futura.)
3. ~~**DAL-dual SQL Server**~~ **RESUELTO 2026-07-12** (`2a228b1`): `Ecorex.Infrastructure.SqlServer` estaba
   ~14 migraciones atras; migracion catch-up (SqlServerCatchUp20260712) que APLICA de punta a punta contra
   un SQL Server real. Destapo un problema DAL-dual PREEXISTENTE en varios modulos (Contenedor de datos,
   Gestor): cascadas validas en PG pero prohibidas en SQL Server (ciclos por auto-referencia / rutas de
   cascada multiples, error 1785); corregidas provider-condicional (Restrict en SQL Server, Cascade/SetNull
   en PG -> prod=PG sin cambio). Tests SQL Server (Testcontainers 2022): TaskCore 12/12 + 21/21 en aislamiento.
4. **Endurecimiento (Ola 7): PARCIAL 2026-07-12.**
   - ~~Canal EMAIL de notificaciones~~ **HECHO** (`9c86ec9`): AssignAsync entrega por email (best-effort via
     IEmailSender) al encargado y destinatarios del concepto, fuera de la transaccion. Test verde.
   - ~~Policies de gobierno~~ **HECHO** (`fe2626e`): ConfiguracionMenu (editor de menu) restringido a
     Owner/Admin (tenant_role); AdmUsuarios/RolesPermisos ya enforced por el motor Perm:.
   - ~~Badge de notificaciones EN VIVO (SignalR)~~ **HECHO** (`ce78c42`): NotificationHub (/hubs/notifications)
     + broadcaster + MainLayout suscrito; AssignAsync difunde al encargado y destinatarios; la campana
     refresca el conteo sin recargar. Verificado por componente (endpoint negocia WebSockets, badge
     renderiza, assign tests verdes); infra espejo del TaskHub del kanban.
   - Con esto el **endurecimiento de notificaciones (Ola 7) queda COMPLETO**: in-app + email + badge en vivo.
   - PENDIENTE menor: plantilla de correo configurable por tenant (hoy plantilla HTML fija).
5. **Proyectos: PARCIAL -> casi completo 2026-07-12** (`8f978e8`): ~~presupuesto/costos (`PROYECTOS_COS`)
   + DOFA (`PROYECTOS_DOFA`)~~ **HECHO** (entidades ProjectBudgetItem/ProjectDofa + servicio + paneles en
   ProyectoDetalle + timeline de hitos; migracion DUAL PG+SqlServer; tests dual 4/4; validado desktop+movil;
   DESPLEGADO a prod). QUEDA (backlog): vistas **calendario/gantt propias** mas ricas del proyecto (hoy el
   detalle reusa el kanban del tablero y muestra timeline de hitos).
6. **Refinamientos menores**: ~~Ola 4 (link Mis Procesos->tablero)~~ YA RESUELTO en Ola 6. ~~Ola 6 (auto-abrir
   el modal preset AL cargar el tablero via `?sub=`)~~ **DIFERIDO a la fase del modulo de formularios avanzado**
   (decision del usuario 2026-07-12); el modal preset sigue disponible via `crear-actividad?sub=`.
7. **Backlog post-v1 -- DIFERIDO a la fase del modulo de formularios avanzado** (decision del usuario
   2026-07-12; se afinan cuando el constructor de formularios este mas maduro): controles multimedia del
   formulario (foto/firma/GPS/archivo/barcode, hoy placeholder); vista de tareas para cliente final (solo
   lectura); modulos satelite legacy (PQR, encuesta, visitas); catalogo de Sedes/Empresa-cliente adicional.
8. ~~**Redeploy a prod del fix movil del wizard**~~ **HECHO 2026-07-12** (`e6636d8`, build-from-git + up -d,
   backup `ecorex-2026-07-12-0510.sql.gz`, CSS verificado en prod). Prod ya tiene el wizard responsive.
9. ~~**Mas conceptos-proceso en prod**~~ **HECHO 2026-07-12**: se publico el flujo "Mantenimiento y soporte"
   (FLW-001) en el editor BPMN y se ligo "Mantenimiento preventivo" (Operaciones) -> flujo + tablero
   "Soporte - Mesa de ayuda". Ahora hay **3 conceptos-proceso** en prod: Cotizacion de equipos (Comercial),
   Visita tecnica y Mantenimiento preventivo (Operaciones). Para mas, publicar mas flujos y ligar.

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
      - **Despliegue: HECHO 2026-07-11** -- `AddEntidadConfig` + `AddEntidadKind` ya aplicadas en prod
        (ver ESTADO GENERAL arriba).
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

### Ola 2 - Alta que CONSUME el concepto (sin UI nueva aun)  -- HECHA 2026-07-11 (commit `c95c5f5`)
- En `CreateAsync` (transaccional): arranca el `WorkflowInstance` desde el flujo PUBLICADO del
  CONCEPTO (`subcategoria.WorkflowDefinitionId`), preferente sobre el del ActivityType legacy;
  aplica `TituloAuto/DetalleAuto` cuando el alta no los trae (el titulo puede venir vacio si hay
  TituloAuto), con tokens basicos (`@cliente` -> solicitante); dispara notificaciones dejando
  **traza en el historial** de la tarea con los destinatarios (`ActividadSubcategoriaNotificacion`).
- **NOTA:** la ENTREGA real de notificaciones (email/in-app con plantilla) queda para la **Ola 7**
  (no hay infra de entrega hoy). El set completo de tokens de TituloAuto llega con el modal (Ola 3).
- **Aceptacion CUMPLIDA**: tests de integracion (matriz dual, verdes en PG) -- la actividad-proceso
  arranca su `WorkflowInstance` (Running) con TituloAuto/DetalleAuto aplicados y el primer paso
  Pending (lo de `/mis-pasos`); la actividad simple NO crea instancia; y queda la traza de
  notificacion. Sin cambios de esquema (Ola 2 es solo comportamiento).

### Ola 3 - Modal wizard 4 pasos (fidelidad prototipo)  -- HECHA 2026-07-11 (commit `9af2202`)
- `TaskWizard.razor` reescrito al wizard de 4 pasos del prototipo (`ECOREX.dc.html` ~4280-4438):
  dos columnas (contenido 1100px + aside de resumen 280px), steps nav (Informacion / Contacto /
  Formulario / Documentos), cards con los tokens exactos (`--surface`/`--ink`/`--t-*`/`--sh-*`),
  footer con Limpiar + "Guardar y crear otra" + Siguiente/Guardar.
- Cascada real: **Empresa/Area (`Entidad`) -> Proceso/Tipo (`ActividadCategoria`) -> Actividad
  (`ActividadSubcategoria`) -> Encargado** (usuarios derivados de los cargos del concepto via
  `IActividadCatalogoService.ListEncargadoUserIdsAsync`, fallback a todos). Crea por CONCEPTO
  (`SubcategoriaId` + `EntidadId`, `ActivityTypeId` null). Paso 3 = estado del formulario (render
  dinamico = Ola 5); paso 4 = dropzone (adjuntar desde el detalle). Preselccion opcional de
  concepto/entidad para el camino Mis Procesos.
- **Aceptacion CUMPLIDA** (validado en Chrome): modal 1100px centrado milimetrico; la cascada real
  funciona (entidades + categorias de Conceptos reales); crea por concepto (T00208 "Cotizacion de
  equipos"/Agencia Norte y T00209 "Mantenimiento preventivo"/Operaciones Logisticas, ambas con
  `activity_type_id` null); "Guardar y crear otra" deja el modal abierto con el titulo limpio y el
  concepto conservado. Fix incluido: se elimino la regla CSS obsoleta `.tk-wizard{max-width:620px}`
  que aplastaba el overlay del nuevo modal.

### Ola 4 - Menu "Mis Procesos" dinamico + editor  -- HECHA 2026-07-11 (commit `8de3521`)
- El marcado del grupo (`MenuNode.IsProcessGroup` + checkbox en `ConfiguracionMenu`) ya venia de
  PRE-5. Aqui `NavMenu` EXPANDE ese grupo con el arbol dinamico categoria -> subcategoria TIPO
  PROCESO (las que tienen `WorkflowDefinitionId`) desde `IActividadCatalogoService`, cargado en el
  mismo scope aislado que el menu. Cada subcategoria linkea a `crear-actividad?sub=<id>`;
  `CrearActividad` lee `?sub=` y abre el wizard con la categoria/subcategoria FIJADAS
  (`OpenAsync(presetSubcategoriaId)`).
- **Aceptacion CUMPLIDA** (validado en Chrome): bajo "Mis Procesos" aparece el grupo dinamico
  "Comercial (1)" -> subcategoria-proceso "Cotizacion de equipos"; navegar al link abre el wizard
  con Tipo=Comercial y Actividad=Cotizacion de equipos preseleccionadas (no las vuelve a pedir); la
  poda por permisos del menu sigue intacta (el arbol dinamico es aditivo bajo una seccion visible).
  NOTA: "carga el tablero del concepto" -> el wizard deriva el tablero del concepto al crear (Ola 1);
  el link no abre una vista de tablero especifica (refinamiento menor).

### Ola 5 - Arranque form-first  -- HECHA 2026-07-11 (commit `16bf824`)
- Cuando el concepto tiene `IniciaModulo` + `FormDefinitionId`, al pasar al paso "Formulario" el
  wizard CREA la tarea (arranca su flujo, Ola 2) y renderiza el `DynamicFormRenderer` (Fill)
  referenciando el numero de la tarea. El "Enviar" del formulario valida en servidor, persiste la
  `FormResponse` enlazada (reference = numero de tarea) y, si el paso del flujo tiene formulario,
  lo completa (`FormResponseService`). "Diligenciar luego" deja la tarea creada con el formulario
  pendiente. El footer del wizard se adapta (sin Guardar en el paso del formulario).
- **Aceptacion CUMPLIDA** (validado en Chrome): concepto form-first "Requerimiento infraestructura"
  (FRM-001) -> al entrar al paso Formulario se creo la tarea **T00210** y el formulario "Solicitud
  de cotizacion" renderizo con ref T00210; valido en servidor (bloqueo por campo obligatorio) y al
  completar quedo **Submitted** enlazado a la tarea (reference=T00210).

### Ola 6 - Tableros, tabs y tiempo real  -- HECHA 2026-07-11 (commit `4e17144`)
- **Grueso ya construido (ADR-0020):** indice `/actividades` (`ActivityBoardsIndex`) + detalle
  (`ActivityBoardDetail`) con Kanban; alcances "Todas del equipo / Pendientes mios / No asignados"
  con conteos (`ScopeCounters`); vistas Tablero/Lista/Calendario/Gantt; drag&drop
  (`ActivityBoardService.MoveTaskAsync`); filtros usuario/etiqueta; SignalR (`TaskHub` +
  `SignalRTaskBroadcaster`). El tablero se configura en la subcategoria del concepto
  (`TaskBoardId` + columna terminal) y Ola 1 lo hereda al crear.
- **Cerrados los 3 pendientes:** (1) SignalR VIVO -> el detalle se suscribe a `TaskHub.TaskChanged`
  y recarga (ya existia; verificado). (2) `?sub=` -> `/actividades?sub={subcategoriaId}` resuelve el
  `TaskBoardId` del concepto (scope aislado) y CARGA ese tablero; el menu "Mis Procesos" apunta a
  `actividades?sub=`. Fix de concurrencia: mientras resuelve NO pinta el indice (chocaban las cargas
  en el DbContext del circuito). (3) crear-desde-tablero ofrece SOLO conceptos SIN proceso
  (subcategorias sin flujo) y clasifica por concepto (`QuickCreateTaskRequest.SubcategoriaId`).
- **Aceptacion CUMPLIDA** (validado en Chrome): conteos vivos + alcances OK; `?sub=` carga
  "Comercial - Requerimiento Infraestructura"; el crear rapido lista 7 conceptos sin proceso
  (excluye "Cotizacion de equipos" que tiene flujo) y T00211 se creo por concepto.
- NOTA: el auto-abrir del modal preset AL cargar el tablero (`carga tablero + abre modal` del
  prototipo) quedo diferido por la concurrencia de DbContext compartido; el modal preset con cat/sub
  sigue disponible via `crear-actividad?sub=` (Ola 4).

### Ola 7 - Endurecimiento  -- HECHA 2026-07-11 (commit codigo 7111cbb)
- Deuda del encargado (flag principal si se decidio), consecutivos transaccionales, notificaciones
  (plantilla), policies compuestas por vista (multi-permiso del legacy), auditoria.
- **Aceptacion**: pruebas de permisos, concurrencia, y notificacion al asignar.

**Resuelto.** La mayoria del endurecimiento ya venia de olas previas; Ola 7 cerro la pieza
faltante y verifico el resto con tests de integracion (4/4 verdes en PG):
- **Encargado**: flag jefe/responsable = PRE-4 (`OrgUnitMember.IsResponsible`, uno por unidad).
- **Consecutivos transaccionales**: `CreateAsync` emite el correlativo T dentro de la transaccion
  (test `ConcurrentCreates_YieldUniqueCorrelativeNumbers`).
- **Concurrencia optimista**: rowversion/xmin (test `OptimisticConcurrency_SecondStaleUpdate_GetsTypedConflict`).
- **Auditoria**: cada accion deja `TaskItemActivity` (historia inmutable de la tarea).
- **Permisos**: aislamiento cross-tenant por filtro global (test `TaskItems_AreIsolatedBetweenTenants`).
- **NUEVO - notificacion al asignar**: `AssignAsync` deja traza dirigida al encargado
  ("notifico a X: le asignaron la tarea N") y, si la tarea tiene concepto, a los destinatarios
  configurados (`ActividadSubcategoriaNotificacion`) via helper `AddConceptNotificationAsync`.
  Test `Assign_RecordsNotificationTrace_ForAssignee` (verde). La ENTREGA real (email/in-app con
  plantilla) queda como backlog.

**Diferidos de Ola 7 -- RESUELTOS 2026-07-11 (commits `f9ea27b` + `ef9ef06`):**
- **Policies COMPUESTAS por vista -- HECHO** (`f9ea27b`): el motor `Perm:{modulo}:{accion}` (ADR-0033)
  ahora soporta AND multi-permiso (`Perm:m1:a1+m2:a2` = varios `PermissionRequirement`). La familia
  Tareas dejo de ser placeholder: `Tareas.Ver`/`Proyectos.Ver`/`Flujos.Ver` exigen el permiso real y
  `Formularios.Disenar` es COMPUESTA (ver+editar). Sin tocar paginas. 26/26 tests unitarios verdes.
  Aun placeholder (documentado): policies de gobierno (AdmUsuarios/RolesPermisos/ConfiguracionMenu a
  Owner/Admin) y Conceptos.Editar/Dependencias.Ver/MisPasos.Ver.
- **Entrega REAL de notificaciones in-app -- HECHO** (`ef9ef06`): entidad `Notification` (tenant-scoped,
  por usuario, leido/no leido) + `INotificationService` + migracion `AddNotifications` (PG local).
  `AssignAsync` ENTREGA una notificacion al encargado (TaskAssigned) y a los destinatarios del concepto
  (ConceptNotice), en la misma transaccion. Campana del topbar = badge real con conteo -> pagina
  `/notificaciones` (marcar leida / todas / abrir). Test integracion verde + validado en Chrome
  (badge 2, marcar leida -> 1). Backlog: canal EMAIL con plantilla (IEmailSender ya existe) y refresco
  en vivo del badge por SignalR. `AddNotifications` DESPLEGADA a prod 2026-07-11.

## B2. Olas de PROYECTOS (en alcance por decision del usuario)

Se corren DESPUES de la base de actividades (o en paralelo por un sub-agente distinto en worktree,
ya que Proyectos toca entidades propias `PROYECTOS_*`/`DOC_PROYECTOS_*`). Plano del ETL legacy en
[[Tareas y Proyectos - paginas basicas]] seccion 2 (~20 tablas) y destino en su seccion 0.

### Ola P1 - Dominio + cabecera de proyecto  -- HECHA 2026-07-11 (commit `1dd1b90`)
- **Ya existia** (commit AddTaskCore): entidad `Project` (Code/Name/Status/fechas/owner/soft-archive/
  Version) + `ProjectMember` (ACL owner+miembros CanEdit) + `ProjectService` (CRUD + miembros +
  CheckAccess) + UI `Proyectos.razor` (lista+CRUD) y `ProyectoDetalle.razor` (detalle+miembros+kanban
  embebido) + seed `PRJ-001`. => la "cabecera de proyecto" estaba COMPLETA.
- **Nuevo (esta ola): HITOS.** Entidad `ProjectMilestone` (tenant-scoped: ProjectId/Name/DueDate?/
  SortOrder/IsCompleted), migracion `AddProjectMilestones` (PG local); `ProjectService` List/Add/
  Update/SetCompleted/RemoveMilestone (+ `ProjectMilestoneDto` con TaskCount); panel de hitos en
  `ProyectoDetalle` (agregar con fecha / completar / quitar / conteo de actividades); seed de 2 hitos
  para PRJ-001. **DIFERIDO a backlog**: presupuesto/costos (`PROYECTOS_COS`), DOFA (`PROYECTOS_DOFA`)
  -- no se modelaron (fuera del alcance minimo P1/P3).
- **Aceptacion CUMPLIDA**: CRUD de proyecto + hitos tenant-scoped; `AddProjectMilestones` DESPLEGADA a
  prod 2026-07-11 (SqlServer sigue en backlog DAL-dual). Validado en Chrome.

### Ola P2 - UI Proyecto (ProyectoDetalle) + ACL por proyecto
- `Proyectos.razor` (lista/filtro) + `ProyectoDetalle.razor` (cabecera Estado/Asignados/Fecha/
  Etiquetas + tabs Vista Tablero / Timeline / Calendario, fidelidad prototipo). ACL por proyecto
  via authorization handler por recurso (`PROYECTOS_RES.FLAG_ED/FLAG_VER` -> policy por recurso),
  NO chequeo ad-hoc. Las tareas del proyecto son `TaskItem` con `ProjectId`.
- **Aceptacion**: ver/editar/borrar respeta el ACL por proyecto; el tablero del proyecto muestra
  sus tareas; borrar sin FLAG_ED se bloquea.

### Ola P3 - Integracion Actividades <-> Proyectos + tiempo real  -- HECHA 2026-07-11 (commit `1dd1b90`)
- `TaskItem.MilestoneId` (FK nullable a `ProjectMilestone`, Restrict) + DTOs; `TaskItemService` valida
  que el hito pertenezca al proyecto (Invalid si es de otro) y lo persiste; filtro por hito; summary con
  `MilestoneName`. El wizard (paso Contacto) tiene selector **Proyecto -> Hito** que carga los hitos al
  elegir proyecto (`OnProjectChanged`) y pasa `MilestoneId` al crear. La actividad aparece en el
  **tablero del proyecto** (kanban filtrado por ProjectId, ya existente; SignalR `TaskHub` ya vivo) y
  suma al **conteo de actividades del hito**.
- **Aceptacion CUMPLIDA** (test integracion `CreateActivity_LinkedToProjectMilestone...` verde en PG +
  validado en Chrome): T00212 creada por el wizard con PRJ-001 + hito "Kickoff y alcance" aparece en el
  tablero del proyecto y el hito muestra "1 act."; un hito de otro proyecto es rechazado.
- NOTA: timeline/calendario del proyecto = refinamiento menor (el detalle ya reusa el kanban; las vistas
  Lista/Calendario/Gantt existen a nivel de tablero de actividades).

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
