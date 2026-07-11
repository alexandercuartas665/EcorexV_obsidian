---
tipo: ficha-modulos-cliente
modulos: ["000038 Crear Actividad", "000042 Proyectos", "000636 Administrar Actividades", "000620", "000039"]
stack_destino: .NET 10 / ASP.NET Core / EF Core 10 / Blazor / SignalR
---

# Tareas y Proyectos - paginas basicas del modulo OPERACIONES

> Nota de la Capa 2 (nucleo Tareas y Proyectos). Documenta las paginas
> operativas centrales con las que el usuario interactua dia a dia. Sigue el
> patron del vault: primero la VISION DESTINO (.NET 10 / Blazor, seccion 0) y
> luego el ANALISIS DEL ORIGEN legacy (WebForms) como plano del ETL y catalogo
> de reglas de negocio a preservar. Aspecto y navegacion definitivos en
> [[Visión y entorno|Prototipo Final ECOREX]]; vision maestra en [[Visión y entorno]].

---

## 0. Vision destino (.NET 10 / Blazor)

En el destino, estas tres paginas dejan de ser `.aspx` con code-behind y se
convierten en **rutas y componentes Blazor** dentro del Task Service, sobre la
Clean Architecture descrita en [[Visión y entorno]] (secciones 3 y 5). El mapeo
conceptual es:

| Origen (WebForms) | Destino (.NET 10) | Ruta Blazor destino |
|---|---|---|
| `NEWFRONT_admtareas.aspx` (000636) | `TareasBoard` (lista + Kanban del tenant) | `/tareas` |
| `NEWFRONT_proyectos.aspx` (000042) | `ProyectoDetalle` (cabecera + tabs + Kanban) | `/proyectos/{id}` |
| `NEWFRONT_tar_tareas.aspx` (000038) | `NuevaActividadDialog` (wizard `DynamicFormRenderer`) | dialog modal sobre `/tareas` |

Principios que gobiernan las tres paginas en el destino:

1. **Multi-tenant real**: donde el origen filtra por columna `SUCURSAL` +
   disciplina del desarrollador, el destino aplica `TenantId` con
   `HasQueryFilter` global + RLS en la BD. Ningun query de estas paginas puede
   ver tareas o proyectos de otro tenant aunque el desarrollador lo olvide. Ver
   [[Gestion de Empresas - Admin multi-tenant]].
2. **DAL dual**: toda lectura/escritura pasa por `IEcorexDbContext` (PostgreSQL o
   SQL Server segun `Database:Provider`), nunca por SQL concatenado. Los 9 SQL
   concatenados del origen se reescriben como consultas EF Core parametrizadas.
3. **Tiempo real**: los grids y el Kanban se actualizan por **SignalR** (grupo
   por tenant + proyecto), sin postbacks ni `UpdatePanel`. Un cambio de estado en
   un dispositivo se propaga a los demas al instante.
4. **Motores portados**: la instanciacion de flujo al crear una tarea la resuelve
   `WorkflowEngine` (antes `AdmWorkflow`); el formulario dinamico lo renderiza
   `DynamicFormRenderer` (antes `crtCargaEncuestaII`); las reglas de cierre de
   paso las ejecuta `RulesEngine` (antes `cl_gestion_reglas`).
5. **Permisos como policies**: los codigos `PERMITE_*` del origen se convierten en
   **policies .NET** consumidas por el PermissionsManager (autorizacion por
   `[Authorize(Policy = ...)]` y directivas `<AuthorizeView>` en Blazor), no por
   llamadas ad-hoc a `ValidaPermiso`.
6. **Correccion de errores heredados**: operaciones multi-tabla (crear tarea +
   instancia de flujo + adjuntos + notificacion) van en **una transaccion**;
   borrado es **soft-delete**; edicion concurrente usa **concurrencia optimista**
   (rowversion / xmin); toda accion queda en **auditoria**.

El detalle de UX de los dos controles centrales que estas paginas hospedan vive
en [[ctrTareasII - Spec para reconstruir en Claude Design]] (crear/editar) y
[[ctrVertareasII - Spec para reconstruir en Claude Design]] (trabajar el caso).

> Lo que sigue (secciones 1 a 6) es el **analisis del sistema ORIGEN** legacy. Se
> conserva como plano del ETL y como catalogo de reglas de negocio a preservar:
> cada tabla, permiso y cascada documentado abajo es un requisito que el destino
> debe reproducir con la arquitectura de la seccion 0. No es la especificacion de
> implementacion destino.

---

## 1. [ORIGEN] `NEWFRONT_admtareas.aspx` (módulo `000636;000620;000039`)

**Path:** `Bootstrap\Formularios\Modulos\Tareas\NEWFRONT_admtareas.aspx`
**Destino:** componente Blazor `TareasBoard` en ruta `/tareas` (ver seccion 0).

> La **vista principal** del usuario operativo. Lista todas las tareas pendientes, asignadas, todas, en formato grid + kanban. Es la pantalla a la que el usuario llega por defecto.
>
> **A preservar en destino:** los 3 codigos de modulo `000636;000620;000039` (patron
> multi-permiso) se traducen a un conjunto de policies .NET evaluadas por el
> PermissionsManager; los 4 tabs (No Asignados / Asignados / Todos / Kanban) se
> mantienen como pestanas del componente con conteos vivos por SignalR; la
> preseleccion por `?cat=&sub=` se conserva como query parameters de la ruta
> Blazor. El modo soporte de `SUCURSAL="01"` (proveedor 3DEV) se modela como un
> **flag de tenant** en el registro de empresas, no como comparacion literal de
> string.

### IDs de módulo

Tiene **3 códigos separados por `;`**: `"000636;000620;000039"`. Esto es un patrón único — un solo `.aspx` representa varios "módulos" del sistema para fines de permisos. El `PermissionsManager.ValidaPermiso(...)` consulta los 3 códigos cuando se valida una acción.

### Tabla y consecutivo

| Campo | Valor |
|---|---|
| `manejo_tabla` | `"SISTEMA"` (tabla de tareas/casos) |
| `manejo_consecutivo` | `"T05"` |
| `manejo_dbx` | `"GENE"` |

### Carga de parámetros XML al iniciar

Carga 2 XMLs de `GEN_PARAMETROS` (módulo `000636`):

| `CODIGO` | Para qué |
|---|---|
| `XML_NUEVA_TAREA` | Plantilla XML que define los campos del modal "Crear Nueva Tarea" |
| `XML_NOTIFICACION_TAREA` | Plantilla XML para construir el body del email de notificación al asignar tarea |

### Permisos consultados (vía `PermissionsManager.ValidaPermiso`)

| Permiso | Si true |
|---|---|
| `PERMITE_ASIGNAR` | Activa botones de asignación en los grids |
| `PERMITE_VER_TODAS_LAS_ACTIVIDADES` | Muestra grid "TODOS" + permite filtros amplios |
| `PERMITE_VER_MIS_PROYECTOS` | Permite filtrar por proyectos propios |
| `PERMITE_VER_PROYECTOS_TODOS` | Permite filtrar por todos los proyectos |
| `SOLO_PROYECTOS_DE_ORGANIZACION` | Restringe a proyectos de la propia organización |

### UserControls hijos

| Control | Rol |
|---|---|
| `topBar` | Topbar con título/breadcrumb/acciones |
| `sideNavBar`, `navBarTop` | Master controls |
| `ctrTareas1` | Form para crear/editar una tarea individual |
| `ctrVertareasII` | Vista detallada de una tarea |
| `ctrGridTareasAsignados` | Grid (tab "ASIGNADO") |
| `ctrGridTareassnAsignados` | Grid (tab "NO ASIGNADO") |
| `ctrGridTareasTodas` | Grid (tab "TODOS") |
| **`ctrKanbanTablero`** | **Tablero kanban** (drag&drop entre estados) |

### Tabs visibles

| Tab | Contenido | Variable de conteo |
|---|---|---|
| `lbltab1` | No Asignados | `ctrGridTareassnAsignados.Cantidadtareas` |
| `lbltab2` | Asignados | `ctrGridTareasAsignados.Cantidadtareas` |
| `lbltab5` | Todos | `ctrGridTareasTodas.Cantidadtareas` |
| `lblTabKanban` | Kanban | `ctrKanbanTablero.Cantidadtareas` |

### Query string

Acepta:
- `?cat={categoria}&sub={subcategoria}` — preselecciona filtro al cargar (usado por links del dashboard como "Requerimientos de equipos")

### Sucursal especial `"01"`

```vb
If Session("Empresa") = "01" Then
    ctrTareas1.TareaSoporte = "1"   ' modo soporte de 3DEV (proveedor)
    ctrVertareasII.TareaSoporte = "1"
    ctrGridTareassnAsignados.TresDev = "1"
Else
    ' modo cliente normal
End If
```

Sucursal `"01"` es la del **proveedor (3DEV/Bitcode)** — activa funciones de soporte. Todos los demás tenants son clientes.

### Modal botón GPT
`topBarBotGPT` comentado en el código — sugiere que hubo un asistente GPT directo en la topbar (deshabilitado).

---

## 2. [ORIGEN] `NEWFRONT_proyectos.aspx` (módulo `000042;000655`)

**Path:** `Bootstrap\Formularios\Modulos\Tareas\NEWFRONT_proyectos.aspx`
**Destino:** componente Blazor `ProyectoDetalle` en ruta `/proyectos/{id}` (ver seccion 0).

> Administra **proyectos** (contenedores de tareas relacionadas con presupuesto, hitos, costos).
>
> **A preservar en destino:** el ACL por proyecto de `PROYECTOS_RES` (flags
> `FLAG_ED` editar / `FLAG_VER` ver) se conserva como regla de negocio, pero se
> implementa con un **authorization handler** por recurso (no un `FindReader`
> suelto antes de borrar) y el borrado es **soft-delete** auditado. La cabecera
> del prototipo (Estado, Asignados con avatares, Fecha limite, Etiquetas) y las
> pestanas `Vista Tablero / Timeline / Calendario` de [[Visión y entorno|Prototipo Final ECOREX]]
> reemplazan el layout WebForms; las ~20 tablas `PROYECTOS_*` / `DOC_PROYECTOS_*`
> mapean a entidades EF Core `ITenantScoped` (ver plano del ETL abajo).

### Tabla y consecutivo

| Campo | Valor |
|---|---|
| `manejo_tabla` | `"TRA_TIPOSPRO"` (tipos de proyecto) |
| `manejo_consecutivo` | `"T05"` |
| `manejo_dbx` | `"GENE"` |

### IDs de módulo

`"000042;000655"` — dos códigos por las mismas razones de `NEWFRONT_admtareas`.

### Catálogos cargados en combos

- **DOFA**: vacío, DEBILIDADES, OPORTUNIDADES, FORTALEZAS, AMENAZAS
- **Costos**: vacío, PROVEEDOR EXTERNO, OPERADORES INTERNOS, EQUIPOS Y MATERIALES, OTROS

### Permisos

| Permiso | Si true |
|---|---|
| `VER_FILTRO_PROYECTOS` | Muestra `ctrFiltroProyecto` (filtro avanzado) |

### UserControls hijos

- `topBar`, `sideNavBar`, `navBarTop`
- `ctrTareas1` — para tareas dentro del proyecto
- `ctrFiltroProyecto` — filtro avanzado

### Tablas SQL principales

| Tabla | Cols | Para qué |
|---|---|---|
| `DOC_PROYECTOS` | 19 | Cabecera de proyecto |
| `DOC_PROYECTOS_FIRMAS` | 8 | Firmas asociadas |
| `DOC_PROYECTOS_FORMS` | 5 | Formularios asociados |
| `DOC_PROYECTOS_ORG` | 8 | Organización |
| `DOC_PROYECTOS_PRORROGAS` | 13 | Prórrogas |
| `DOC_PROYECTOS_ROLES` | 8 | Roles dentro del proyecto |
| `DOC_PROYECTOS_SER` | 18 | Servicios |
| `DOC_PROYECTOS_SER_HIS` | 21 | Historial de servicios |
| `DOC_PROYECTOS_SERIE` | 6 | Serialización |
| `PROYECTOS_RES` | (varios) | **Permisos por proyecto/usuario** (FLAG_ED edit, FLAG_VER ver) |
| `PROYECTOS_DOFA` | | DOFA |
| `PROYECTOS_HITO` | | Hitos |
| `PROYECTOS_PRESUPUESTO` | | Presupuesto |
| `PROYECTOS_COS` | | Costos |
| `PROYECTOS_NOVEDADES` | | Novedades |
| `PROYECTOS_OBS` | | Observaciones |
| `PROYECTOS_TELECOMU` | | Telecomunicaciones (?) |
| ... | | (varios más con prefijo `PROYECTOS_`) |

### Validación de eliminación

```vb
If tbrec.FindReader("SELECT FLAG_ED FROM PROYECTOS_RES WHERE SUCURSAL=@e AND USUARIO=@u AND PROYECTO=@p", ...) = "1" Then
    Call delProyecto()
Else
    topBar.ShowError("Error", "No puedes eliminar un proyecto el cual no puedes editar.")
End If
```

→ **ACL por proyecto** vía `PROYECTOS_RES.FLAG_ED`. Si el usuario no tiene flag de edición, no puede borrar.

---

## 3. [ORIGEN] `NEWFRONT_tar_tareas.aspx` (módulo `000038`)

**Path:** `Bootstrap\Formularios\Modulos\Tareas\NEWFRONT_tar_tareas.aspx`
**Destino:** dialog Blazor `NuevaActividadDialog` (wizard) invocado desde `/tareas`.

> Página para **agregar una nueva actividad/caso** específicamente.
>
> **A preservar en destino:** la plantilla `XML_NUEVA_TAREA` de `GEN_PARAMETROS`
> deja de ser XML crudo y se modela como definicion versionada del formulario que
> `DynamicFormRenderer` interpreta; el consecutivo `A06` lo genera un servicio de
> consecutivos transaccional por tenant. El "bug menor" anotado abajo
> (`TareaSoporte = 1` para toda sucursal) se corrige explicitamente en el destino:
> el flag de soporte deriva del flag de tenant, no se fuerza por igual.

### Tabla y consecutivo

| Campo | Valor |
|---|---|
| `manejo_tabla` | `"ACU_BAJAS"` (interesante: el nombre sugiere "Acumulado de bajas" pero realmente almacena casos) |
| `manejo_consecutivo` | `"A06"` |
| `manejo_dbx` | `"GENE"` |

### IDs de módulo

Único: `"000038"`.

### Permisos consultados

| Permiso | Efecto |
|---|---|
| `PERMITE_ASIGNAR` | Habilita asignar |
| `PERMITE_VER_MIS_PROYECTOS` | Filtro proyectos propios |
| `PERMITE_VER_PROYECTOS_TODOS` | Filtro todos |

### Carga inicial

- Lee `XML_NUEVA_TAREA` y `XML_NOTIFICACION_TAREA` de `GEN_PARAMETROS` (mismo módulo `000636` que admtareas — comparte la plantilla)
- Llena combos:
  - **Estado tarea**: Pendiente / Terminado / Todos
  - **Estado programación**: vacío / Sin Programar / Programado
  - **Prioridad** (combo `cmbfiltroprioridad`) desde `PRIORIDADCCS`
  - **Encargado / Asignado** desde `[dbx.GENE].dbo.USUARIO WHERE FLAG_3DEV = 1`
  - Filtros categoría/subcategoría

### Detalle "soporte" especial
Para sucursal `"01"`: `ctrTareas1.TareaSoporte = 1`
Para otras: `ctrTareas1.TareaSoporte = 1` (igual — bug menor o caso histórico)

---

## 4. Otras páginas relevantes del módulo Tareas

Detectadas en `Bootstrap\Formularios\modulos\tareas\`:

| Página | Para qué |
|---|---|
| `NEWFRONT_adm_calendar.aspx` | Vista calendario de tareas |
| `NEWFRONT_adm_jefetareas.aspx` | Vista para jefes (supervisión) |
| `NEWFRONT_admtareas_compacto.aspx` | Vista compacta (probable para móvil/tablet) |
| `NEWFRONT_tar_conceptos.aspx` | Catálogo "Actividades" (000270) |
| `NEWFRONT_tar_estados.aspx` | Catálogo estados (000653) |
| `NEWFRONT_tar_prioridades.aspx` | Catálogo prioridades (000621) |
| `NEWFRONT_tar_notificaciones.aspx` | Notificaciones (000288) — usa SignalR |
| `NEWFRONT_tar_visita.aspx` | Visitas (000?) |
| `NEWFRONT_tar_pqr.aspx` | PQRs |
| `NEWFRONT_tar_encuesta_servicio.aspx` | Encuesta de servicio post-cierre |
| `NEWFRONT_tar_tareascli.aspx` | Vista de tareas para el cliente final |
| `tar_tipoproyecto.aspx` | Catálogo tipos proyecto (000690) |
| `tar_notificaciones.aspx`, `tar_pqr.aspx`, `tar_visita.aspx`, `tar_encuesta_servicio.aspx` | Versiones antiguas |
| `TestSchedule.aspx` | Página de prueba de scheduling |
| `taskChecker.aspx` | Endpoint de verificación periódica |

---

## 5. Patrón común detectado

Las 3 páginas siguen el mismo patrón documentado en [[Manejo de Datos - Alias, parametros, UDFs, consecutivos]] §9:

1. `tbrec As New MotherData.AdmDatos`
2. `BASE_SISTEMA = mbase_empresa.base_pricipal` en `Sub New()`
3. Variables `manejo_tabla`, `manejo_consecutivo`, `manejo_dbx`
4. `Page_Load`:
   - Setea `topBar.Modulo`
   - Carga combos via `Optimize.ComboFill(sql, default, combo, BASE_SISTEMA)`
   - Valida permisos via `PermManager.ValidaPermiso(...)`
   - Redirect a login si `Session("Nombre") = ""`
5. `topBar_Save/Delete/Clear/Nuevo` handlers que invocan los métodos CRUD locales
6. Lectura/escritura via SQL concatenado contra `tbrec.Execute/FindReader/FindDataset`

---

## 6. Patrones de integración Tareas <-> Flujos <-> Formularios

Las integraciones del origen son **reglas de negocio a preservar**. La columna
destino indica el motor portado que las asume (ver [[Visión y entorno]] 3.2):

| Integración | Origen (WebForms) | Destino (.NET 10) |
|---|---|---|
| Crear tarea -> instancia proceso BPMN | `AdmWorkflow.GuardarTablaSeguimiento(Sucursal, Caso, Categoria, SubCategoria)` | `WorkflowEngine.StartInstance(tenantId, taskId, activityTypeId)` dentro de la misma transaccion del INSERT de la tarea |
| Tarea consume formulario | `GEN_COMPONENTES_R.MODULO='FLUJO_PROCESO'` + `REFERENCIA=ID_ELEMENTO` + `FORMULARIO=ENCUESTAS_MOV.CODIGO` | `DynamicFormRenderer` resuelve la definicion por (tenant, activityType, nodo BPMN); respuestas a `jsonb`/`nvarchar(max)` via DAL dual |
| Cierre de paso ejecuta reglas | `AdmWorkflow.ActualizaEstadoTareaInicial` -> `SiguienteEstado` -> reglas autonomas | `WorkflowEngine.Advance()` invoca `RulesEngine` (verbos Ensamblado tipados); emite evento `task.state.changed` por SignalR + Event Bus |
| Vista de la tarea para cliente | `NEWFRONT_tar_tareascli.aspx` (interfaz simplificada) | vista Blazor de solo lectura con policy de cliente; mismo componente base, `AuthorizeView` recorta acciones |

---

## 7. Reglas de negocio a preservar (resumen para el ETL)

Consolidado de lo que el destino DEBE reproducir, extraido de las secciones 1-6:

1. **Multi-permiso por pagina**: una vista puede exigir varios codigos de modulo a
   la vez (000636;000620;000039). Traducir a policies compuestas.
2. **ACL por proyecto** (`PROYECTOS_RES.FLAG_ED` / `FLAG_VER`): autorizacion por
   recurso, no global.
3. **Consecutivos por tipo de documento** (`A06`, `T05`): servicio transaccional
   por tenant, sin colisiones concurrentes.
4. **Cascadas de combos** Empresa -> Tipo/Area -> Subcategoria -> Encargado (por
   cargo del nodo BPMN): logica de dominio, no del componente UI.
5. **Notificacion al asignar** (plantilla `XML_NOTIFICACION_TAREA`): pasa a
   Notification Service (SignalR + email por SendGrid) disparado por evento.
6. **Modo soporte del proveedor** (tenant `01` / 3DEV): flag de tenant, no
   comparacion de string literal.

## 8. TODO (migracion)

- [ ] Definir el conjunto exacto de policies .NET que reemplazan cada `PERMITE_*`
      (mapa completo del catalogo de permisos).
- [ ] Especificar el contrato de `WorkflowEngine.StartInstance` y su transaccion
      con el INSERT de la tarea.
- [ ] Modelar las ~20 tablas `PROYECTOS_*` / `DOC_PROYECTOS_*` como entidades EF
      Core `ITenantScoped` y definir el ETL campo por campo.
- [ ] Disenar los grupos SignalR (por tenant + proyecto) para los conteos vivos de
      los 4 tabs y el drag&drop del Kanban.
- [ ] Portar `ctrKanbanTablero` a un componente Blazor con drag&drop y RLS.
