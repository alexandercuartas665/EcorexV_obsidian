---
tipo: inventario
fuente: dashboard `das_module.aspx` + glob de páginas .aspx
estado: completo para módulos visibles al usuario `1048064705`
stack_destino: .NET 10 / ASP.NET Core 10 / EF Core 10 / Blazor
---

# Inventario General de modulos - proyecto Tareas

> Este inventario tiene DOS lecturas. Como plano del sistema DESTINO (.NET 10)
> es el catalogo de capacidades que el nuevo ECOREX debe cubrir, y como
> referencia del sistema ORIGEN es el mapa exacto del legacy que se migra: cada
> modulo `000XXX`, su pagina fisica y su tabla. La vision maestra esta en
> [[Visión y entorno]]; el aspecto y la navegacion definitivos en
> [[00 - Prototipo Final ECOREX]]. Separar ORIGEN vs DESTINO es la disciplina de
> todo el vault.

## 0. Vision DESTINO del inventario (.NET 10)

En el destino el inventario deja de ser una lista de paginas `.aspx` y se vuelve
un **registro de modulos por tenant** gobernado por el Module Registry (origen
`ADM_WEB`, modulo `000109`). Cada modulo del origen se reproyecta asi:

1. **Codigo `000XXX` preservado como clave de negocio.** El codigo de 6 digitos
   NO se descarta: viaja como `LegacyModuleCode` en la entidad `Module` y en el
   prototipo se sigue mostrando ("MODULO 000XXX", "variables por tenant"). Es el
   puente ORIGEN <-> DESTINO y la clave del ETL de permisos y parametros.
2. **Pagina fisica `.aspx` -> feature Blazor + endpoints API.** El `.aspx.vb`
   monolitico se descompone en Domain/Application/Infrastructure segun Clean
   Architecture (seccion 12 de [[Visión y entorno]]).
3. **Tabla principal -> entidad EF Core `ITenantScoped`.** Toda tabla `SUCURSAL`
   listada aqui se mapea a una entidad con `TenantId` + `HasQueryFilter` + RLS.
   El detalle de ese mapeo por familia esta en
   [[Modelo Entidad-Relacion logico]].
4. **Permisos por menu -> policies .NET.** `PermissionsManager.ValidaPermiso`
   pasa a policies del PermissionsManager portado (modulo `000109` +
   [[Gestion de Empresas - Admin multi-tenant]]).
5. **Errores heredados corregidos.** El SQL concatenado del patron comun (ver
   seccion final) desaparece: EF Core parametrizado, transacciones en comandos
   multi-tabla, secretos cifrados, sin `[dbx.GENE]` concatenado (resuelto por
   tenant), consecutivos `Funciones.tipdoc` -> servicios de secuencia.

La columna "clasificacion" del origen (Operativo, Catalogo, Plataforma,
Desarrollo) se conserva porque orienta la agrupacion de features y el orden de
migracion: el "corazon" (Flujos `000291`, Formularios `000131`, Reglas `000802`)
son los tres motores portados (seccion 3.2 de [[Visión y entorno]]) y tienen
prioridad sobre los catalogos.

## 0.1 Tracker de construccion DESTINO (actualizado 2026-07-05)

Estado real del codigo en EcorexV (rama `fase-0/clon-backbone`, commits a482b47..39b066e).
Los ADR de implementacion (0011..0029) viven en `docs/decisiones/` del repo destino; el
detalle de cada sesion en `PROGRESO.md` y las corridas de prueba en el vault
([[00 - Registro de corridas]]).

| Modulo / capacidad | Legacy | Estado | Notas |
|---|---|---|---|
| Plataforma multi-tenant + Super Admin + planes | - | CONSTRUIDO | Backbone adaptado; tenant demo SKY SYSTEM; DAL dual + test aislamiento 2 motores |
| Nucleo tareas (TaskItem, estados, worklog, kanban, wizard) | 000038/000636 | CONSTRUIDO | ADR-0013; consecutivos por tenant; concurrencia optimista |
| Tableros unificados (indice + detalle + filtros + alcances + vistas Tablero/Lista/Calendario/Gantt) | 000636 | CONSTRUIDO | ADR-0021; "Administrar actividades" y el menu rapido del rail son EL MISMO sistema de tableros |
| Proyectos (con ACL de miembros) | 000042 | CONSTRUIDO (base) | Familia DOC_PROYECTOS_* extendida pendiente de ETL |
| WorkflowEngine (motor BPMN) + editor canvas | 000291 | CONSTRUIDO | ADR-0014/0022; editor canvas propio funcional; BpmnXml regenerado (portable bpmn.io) |
| DynamicFormRenderer + constructor + visor por token | 000131 | CONSTRUIDO | ADR-0015/0021; constructor drag-and-drop funcional; tipos multimedia avanzados pendientes |
| RulesEngine (verbos tipados) + modulo Gestion de reglas | 000802 | CONSTRUIDO | ADR-0016/0023; layout 3 paneles + PARAM_XML editable; verbos IA/importacion pendientes |
| Conceptos de actividad | 000270 | CONSTRUIDO | ADR-0024a; sobre ActivityType real |
| Cargador de contactos (CSV -> mapeo -> dedup -> Leads) | 000873 | CONSTRUIDO | ADR-0024; carga transaccional con historial |
| Extraccion de datos / web scraping | 000730 | CONSTRUIDO | ADR-0025; guard SSRF estricto (fail-closed); scheduler + multipaso pendientes |
| Ficha de empresa (adm_empresas, en PlatformAdmin) | 000072 | CONSTRUIDO | ADR-0026; ficha real del tenant; 9 secciones placeholder (flujos SQL peligrosos NO reconstruidos) |
| Inventario: Items + catalogos normalizados | 000066/000556/000506/000502/000606/000498 | CONSTRUIDO | ADR-0027; Item + Warehouse/Brand/Group/Subgroup/Type + stock por bodega; migracion dual; portado de CUBOT.nails |
| Plantillas WhatsApp HSM | - | CONSTRUIDO | ADR-0029; portado de CUBOT.travels; Submit/SyncStatus STUBS (sin integracion Meta real) |
| Infraestructura IA (Agentes, Lineas, Conversaciones, Bitacora) | 000867 | CONSTRUIDO (heredado + reorg) | ADR-0028; grupo propio "Infraestructura IA"; crear_lead desacoplado del CRM (IAgentLeadSink: NoOp + PipelineLeadSink) |
| Dependencias (organigrama) | 000850 | CONSTRUIDO | ADR-0017 |
| Modulos web (module registry) | 000109 | CONSTRUIDO (base) | ADR-0017; menu derivado del registry + policies por modulo en "paso 1" (derivacion del registry = paso 2 pendiente, transversal) |
| UI fiel al Prototipo Final (ECOREX.dc.html) | - | CONSTRUIDO | Tokens exactos + acordeones + dark mode; login "ventana al producto" (mockup del tablero) |
| CI/CD pr-check (gitleaks + build Release + format + matriz dual) | - | CONSTRUIDO | ADR-0018/0019; gates de merge activos en GitHub Actions |
| Comercial / CRM del workspace | 000477 | HEREDADO CRM | Pipeline/leads del backbone; grupo "CRM (heredado)" reducido a Asesores/Automatizaciones/Lista negra tras extraer la infra IA |
| Programar actividad | 000889 | PENDIENTE | |
| Power BI Service | 000788 | PENDIENTE | Catalogo del registry ya lo lista como placeholder |
| ETL desde db3dev (tenant 01=BITCODE) | - | PENDIENTE | FASE 6; requiere conexion (solo lectura) |

> **Deuda transversal de policies (paso 1 -> paso 2):** cada modulo nuevo registra su
> policy como `RequireClaim("tenant_id")` (equivalente a `TenantMember`) con un nombre
> estable (`Inventario.Ver`, `PlantillasWhatsApp.Editar`, `ExtraccionDatos.Editar`,
> `AdmEmpresas.Ver`...). El "paso 2" comun es derivarlas del Module Registry (000109) y
> anadir MFA en acciones criticas. No es especifico de un modulo.

## 1. Referencia ORIGEN: mapa de modulos legacy

> Lo que sigue es el levantamiento del sistema ORIGEN (WebForms `GestionMovil`).
> Se conserva INTACTO como plano del ETL: cada fila dice que capacidad migrar,
> desde que pagina y sobre que tabla. NO es el diseno destino.

Este inventario mapea cada **link del menu lateral** del dashboard (`das_module.aspx`) a:
- el **codigo de modulo** numerico (6 digitos) -> `Module.LegacyModuleCode` en destino
- el **path fisico** del `.aspx` y su `.aspx.vb` -> feature Blazor + API en destino
- la **tabla SQL principal** que opera -> entidad EF Core `ITenantScoped` en destino
- una **clasificacion** (Operativo, Catalogo, Plataforma, Desarrollo, etc.)

> Convención: todos los módulos viven bajo `Bootstrap\Formularios\Modulos\` (la carpeta `modulos` aparece en minúsculas en algunos paths del menú pero Windows lo trata igual).

---

## Sección OPERACIONES

| Módulo | Cod | Página .aspx | Tabla principal | Notas |
|---|---|---|---|---|
| Crear Una Actividad | `000038` | `modulos\tareas\NEWFRONT_tar_tareas.aspx` | `TAR_*` + `[dbx.GENE].dbo.USUARIO` + `PRIORIDADCCS` | Carga XML de plantilla desde `GEN_PARAMETROS` (`XML_NUEVA_TAREA`, `XML_NOTIFICACION_TAREA`) |
| Proyectos | `000042` | `modulos\tareas\NEWFRONT_proyectos.aspx` | `DOC_PROYECTOS` + 9 tablas derivadas | Roles, firmas, prórrogas, formas, series, organización |
| Administrar Actividades | `000636` | `modulos\tareas\NEWFRONT_admtareas.aspx` | `SISTEMA` + `TAR_*` + grids por estado | Consecutivo `T05`. Usa SignalR para refresco vivo. Componente `ctrGridTareas` |
| Programar Una Actividad | `000889` | `modulos\utilidades\adm_programador.aspx` | (revisar) | Recordatorios/repetición |
| Requerimientos de equipos | (cat=32, sub=00477) | `modulos\tareas\NEWFRONT_admtareas.aspx?cat=32&sub=00477` | igual que admin | Filtro especializado |

---

## Sección NEGOCIO

| Módulo | Cod | Página | Tabla |
|---|---|---|---|
| Creación Clientes | `000232` | `modulos\documentos\NEWFRONT_doc_clientes.aspx` | (revisar) |
| Seguimiento Clientes | `000123` | `modulos\cartera\car_seguimientoNEWFRONT.aspx` | (revisar) |
| Items Inventarios | `000066` | `modulos\inventarios\NEWFRONT_items.aspx` | (revisar) |

---

## Sección AUTOMATIZACIÓN ⭐

| Módulo | Cod | Página | Tabla principal | Notas |
|---|---|---|---|---|
| **Flujos Del Proceso** ⭐ | `000291` | `modulos\Documentos\NEWFRONT_doc_procesos.aspx` | `DOC_PROCESOS` + 9 derivadas + `TAR_WORKFLOW_*` | Consecutivo `0D7`. Integra ChatGPT (`Funciones.clChatGPT`). Componente `crtFlujoProcesos` + clase `AdmWorkflow`. Ver [[00 - Visión Flujos]] |
| **Formularios** ⭐ | `000131` | `modulos\Documental\NEWFRONT_docu_formulario.aspx` | `[ENCUESTAS_MOV]` + `ENCUESTAS_MOV_*` + `ENCUESTA_RESP` + `FORX_DATA` | Consecutivo `03D`. Dos controles paralelos: `ctrFormCreator1` (diseño preguntas) y `ctrFormCreator2` (listado). Usa `HtmlAgilityPack`. Ver [[00 - Visión Formularios]] |
| Power Bi Service | `000788` | `modulos\PowerBI\bi_servicios.aspx` | (revisar) | Catálogo de dashboards Power BI por empresa |
| Agentes IA | `000867` | `modulos\documental\docu_viewform.aspx?token=REDACTED` | Vía visor por token | Tabla `GEN_TOKEN` resuelve el formulario |

---

## Sección SISTEMA › Inventarios (catálogo)

| Módulo | Cod | Página | Tabla |
|---|---|---|---|
| Bodegas | `000556` | `modulos\inventarios\NEWFRONT_inv_bodegas.aspx` | `INV_BODEGAS` |
| Grupo Inventarios | `000506` | `modulos\Inventarios\NEWFRONT_catgrupos.aspx` | `CATGRUPOS` |
| Marcas | `000502` | `modulos\Inventarios\NEWFRONT_catmarcas.aspx` | `CATMARCAS` |
| Subgrupos Inventarios | `000606` | `modulos\Inventarios\NEWFRONT_catsubgrupos.aspx` | `CATSUBGRUPOS` |
| Tipos Inventarios | `000498` | `modulos\Inventarios\NEWFRONT_catgrupoinvetaros.aspx` | `CATGRUPOINVETAROS` |

## Sección SISTEMA › Actividades (parametría tareas)

| Módulo | Cod | Página | Tabla |
|---|---|---|---|
| Prioridades | `000621` | `modulos\tareas\NEWFRONT_tar_prioridades.aspx` | `PRIORIDADCCS` / `TAR_PRIORIDADES` |
| Tipos Proyecto | `000690` | `modulos\tareas\tar_tipoproyecto.aspx` | `TAR_TIPOS_PROYECTO` + `_HITOS` + `_HITOS_ACT` + `_R` + `_RES` (5 tablas) |
| Actividades (conceptos) | `000270` | `modulos\tareas\NEWFRONT_tar_conceptos.aspx` | (revisar) |
| Estados | `000653` | `modulos\tareas\NEWFRONT_tar_estados.aspx` | `TAR_ESTADOS` |

## Sección SISTEMA › CRM

| Módulo | Cod | Página | Tabla |
|---|---|---|---|
| Conceptos Actividades | `000125` | `modulos\cartera\car_actividadcomerNEWFRONT.aspx` | (revisar) |
| Estados CRM | `000272` | `modulos\cartera\car_estadosNEWFRONT.aspx` | (revisar) |
| Perfiles Cliente/Prov | `000166` | `modulos\cartera\car_perfilesNEWFRONT.aspx` | (revisar) |
| Servicios/Productos | `000249` | `modulos\cartera\car_canalesNEWFRONT.aspx` | (revisar) |
| Tipos Empresas | `000231` | `modulos\documentos\NEWFRONT_doc_tiposempresa.aspx` | (revisar) |
| Vendedores | `000124` | `modulos\cartera\car_vendedoresNEWFRONT.aspx` | (revisar) |
| Origen Clientes | `000324` | `modulos\comercial\comer_origen.aspx` | (revisar) |
| Grupos Actividades | `000126` | `modulos\cartera\car_actividadgrupNEWFRONT.aspx` | (revisar) |

## Sección SISTEMA › General

| Módulo | Cod | Página | Tabla |
|---|---|---|---|
| Configuración Entidad | `000615` | `modulos\general\gen_config_empresa_cli.aspx` | `GEN_PARAMETROS` |
| Plantillas | `000893` | `modulos\general\gen_admplantillas.aspx` | `GEN_PLANTILLAS_IMPRESION` |
| Dependencias | `000850` | `modulos\general\gen_dependencias.aspx` | `GEN_DEPENDENCIA_ORGANICA` |
| Roles y Permisos | `000194` | `modulos\general\gen_rol_administrador.aspx` | (revisar — relacionado con `Funciones.permisos` y `PermissionsManager`) |
| Administración Usuarios | `000073` | `modulos\General\NEWFRONT_gen_usuarios.aspx` | `USUARIO` en `[dbx.GENE]` |

## Sección SISTEMA › **Desarrollo** ⭐ (configuración avanzada)

| Módulo | Cod | Página | Tabla principal | Función |
|---|---|---|---|---|
| Autocompletado Formularios | `000801` | `modulos\documental\docu_viewform.aspx?token=REDACTED` | `ENCUESTAS_MOV` (visor token) | Auto-llenado de campos |
| Biblioteca Recursos | `000133` | `modulos\configuracion\NEWFRONT_bib_biblioteca.aspx` | `BIB_BIBLIOTECA` | Snippets de código |
| Controles | `000282` | `modulos\configuracion\NEWFRONT_adm_controles.aspx` | `ADM_CONTROLES` + `_I` + `_V` (3 tablas) | Definición visual de controles del form engine |
| Flujos del Proceso | `000291` | (mismo que automatización) | `DOC_PROCESOS` | Duplicado en Desarrollo |
| Grupos Biblioteca | `000132` | `modulos\configuracion\NEWFRONT_bib_grupos.aspx` | `BIB_GRUPOS` | Agrupador |
| Mensajes Error | `000301` | `modulos\configuracion\NEWFRONT_adm_errores.aspx` | `GEN_ERROR` + `GEN_ERROR_R` (admin de catálogo de errores) | Catálogo |
| Módulos Web | `000109` | `modulos\Configuracion\NEWFRONT_adm_modulos.aspx` | `ADM_WEB` | Define los códigos `000xxx` |
| Notificaciones | `000288` | `modulos\tareas\NEWFRONT_tar_notificaciones.aspx` | (revisar — SignalR) | Emisor SignalR |
| Objetos Sistemas | `000137` | `modulos\Configuracion\newfront_adm_objetos.aspx` | `ADM_HTML` + `_R` (HTML/CSS storage) | Recursos visuales |
| Parámetros XML | `000057` | `modulos\configuracion\newfront_adm_par_xml.aspx` | `PARAMXML` (3 cols) | Configs por módulo |
| **Reglas** ⭐ | `000802` | `modulos\general\gen_reglas.aspx` | (motor de reglas para formularios) | Crítico para [[00 - Visión Formularios]] |
| Reportes | `000119` | `modulos\reportes\newfront_reportes_admin.aspx` | (revisar) | Admin de reports SQL Reporting |
| Servicios Web | `000053` | `modulos\configuracion\NEWFRONT_adm_webservice.aspx` | `ADM_SERVICIOS` | |
| **SQL Admin** ⭐ | `000077` | `modulos\Configuracion\NEWFRONT_adm_sql.aspx` | (ejecutor SQL desde la nube) | Permite ejecutar ops sobre BD desde la app |
| Tipos Documentos | `000136` | `modulos\Configuracion\newfront_gen_tiposdocumento.aspx` | (revisar) | Consecutivos |
| Tipos Tecnologías | `000134` | `modulos\configuracion\bib_tecnologia.aspx` | `BIB_TECNOLOGIA` | |
| Pruebas Formularios | `000891` | `modulos\documental\docu_viewform.aspx?token=REDACTED` | Visor token | Test runner |

---

## Patrón común detectado en TODAS las páginas

Cada `.aspx.vb` sigue la misma estructura (visto en `NEWFRONT_tar_tareas`, `NEWFRONT_admtareas`, `NEWFRONT_doc_procesos`, `NEWFRONT_docu_formulario`):

```vb
Imports MotherData
Public Class NEWFRONT_xxx
    Inherits System.Web.UI.Page

#Region "variables de manejo de datos"
    Dim tbrec As New MotherData.AdmDatos       ' DAL
    Dim mdata As DataSet                       ' carga
    Dim sql_t As String = ""
    Dim BASE_SISTEMA As String = "SERVERI_MAR" ' alias de Config.xml
    Dim PAG_LOGIN As String = "index.aspx"
#End Region

#Region "Inicia sistema {}"
    Sub New()
        Call carga_empresa()                   ' lee Empresa.xml
        BASE_SISTEMA = mbase_empresa.base_pricipal
        PAG_LOGIN    = mbase_empresa.base_login
    End Sub
#End Region

#Region "variables modulos comunes"
    Dim manejo_tabla       As String = "XXX"   ' tabla principal
    Dim manejo_consecutivo As String = "XXX"   ' codigo consecutivo (Funciones.tipdoc)
    Dim manejo_dbx         As String = "GENE"  ' alias de base (resuelto por AdmDatos.PreparaCadena)
#End Region

    Protected Sub Page_Load(...) Handles Me.Load
        ' topBar.Modulo = "000xxx"   <-- aquí se ve el código del módulo
        ' Permisos via PermissionsManager.ValidaPermiso(...)
        ' Carga de combos con Optimize.ComboFill(...)
    End Sub
End Class
```

**Patrón de query típico (riesgo SQL injection masivo):**
```vb
sql_t = "INSERT INTO [dbx." & manejo_dbx & "].dbo." & manejo_tabla & " (..) VALUES ('" & txtnombre.Text & "', '" & Session("Empresa") & "')"
tbrec.Execute(sql_t, BASE_SISTEMA)
```
→ todos los valores se interpolan como strings (cero parametrización). Documentado también en DokTrino origen (~713 puntos) y Visal.

**Correccion en el DESTINO (error #1 heredado).** Este patron es el error sistemico
que se elimina en .NET 10: cada `INSERT/UPDATE` concatenado pasa a EF Core
parametrizado (o `FromSqlInterpolated` cuando haga falta SQL crudo). El
`[dbx." & manejo_dbx & "]` deja de concatenarse: la base fisica la resuelve el
`IConnectionStringResolver` por tenant. El `Session("Empresa")` deja de viajar en
el string: el `TenantId` lo inyecta el `HasQueryFilter` global. Comandos que hoy
tocan varias tablas sin transaccion se envuelven en un Unit of Work. Ver
[[Visión y entorno]] seccion 14 y [[Gestion de Empresas - Admin multi-tenant]].

---

## Componentes/Controles propios reutilizados (vistos en código)

- `topBar` — barra superior con título, breadcrumb, eventos Save/Delete/Clear
- `sideNavBar` — menú lateral por permisos
- `navBarTop` — topbar de usuario
- `Optimizer` — utilidades visuales (`ComboFill`, etc.)
- `Funciones.permisos` y `PermissionsManager.ValidaPermiso(Usuario, Empresa, Permiso, Modulo)`
- `Funciones.ValidaError` — validación de campos con UI integrada (`empty_field`, `sms_nerror`)
- `Funciones.tipdoc.Consecutivo(tipo, "", Empresa)` — generador de IDs
- `Funciones.clChatGPT` — wrapper de OpenAI (usado en Flujos)
- `ctrFormCreator1/2` — controles del constructor de formularios
- `crtFlujoProcesos` — control del editor BPMN
- `AdmWorkflow` — clase de negocio del motor de workflow
- `ctrGridTareas`, `ctrTareas1` — grids de tareas
- `cl_tree_componentes` — árbol del builder de formularios
- `TimerLine.TimeLine` — log
- `HtmlAgilityPack` — parseo HTML (usado en Formularios)

Ver detalles en [[00 - Mapa Funciones]].
