---
tipo: spec-modulo
modulo: gen_reglas.aspx
carpeta: General
proposito: Motor de reglas de negocio parametrizables por documento; define reglas ejecutables (tareas) con configuraciones de formulario asociadas.
---

# gen_reglas - Spec para reconstruir

Documento auto-contenido para reconstruir el modulo `gen_reglas.aspx` (motor de
reglas de negocio) sin depender de otros specs. Fuente: codigo VB.NET
WebForms de la solucion ECOREX (namespace `GestionMovil`, .NET 4.8.1).

## 1. Que hace este modulo (motor de reglas)

`gen_reglas.aspx` es el CRUD y consola de ejecucion del motor de reglas. Un
"documento de reglas" (identificado por `DOCUMENTO`) agrupa:

- Un conjunto de reglas ejecutables (tareas) con un tipo de ejecucion, orden,
  parametros XML y un script previo opcional.
- Un conjunto de configuraciones de formulario (encuestas dinamicas) que
  definen que datos captura el usuario al invocar la regla.
- Un historial de ejecuciones con usuario, fecha, duracion, cantidad de
  registros procesados y mensaje de error.

El modulo permite crear, editar, ordenar, inactivar y ejecutar reglas de forma
interactiva. La ejecucion real la delega en la clase `cl_gestion_reglas` del
namespace `GestionMovil` (Bootstrap/Formularios/Modulos/Documental/Clases).

- URL: `~/Formularios/Modulos/General/gen_reglas.aspx`
- Modulo topBar: `000802`
- Titulo: `REGLAS`, Ubicacion en menu: `Home/GENERAL/REGLAS`
- Master: `~/Formularios/Modulos/Dashboard/MasterFormsii.Master`
- Base fisica: `SERVERI_MAR`, alias SQL: `[dbx.GENE]`

## 2. Modelo mental (Regla + Condicion + Accion + Modo Ejecucion)

En este motor no existen tres entidades separadas (Condicion / Accion /
Ejecucion). Toda una regla es un unico registro `CONTROL_REGLAS_R` con:

- `TIPO` (modo de ejecucion): `mDATA`, `Execute` o `Ensamblado`.
- `PARAM_XML`: XML libre que contiene tanto las condiciones/filtros como las
  acciones y el nombre del proceso a invocar. Se edita como texto plano en un
  textarea de 16 filas.
- `SCRIPT`: script previo (SQL o snippet) que se ejecuta antes.
- `ORDEN`: numero entero. Las reglas se ejecutan en orden ascendente.
- `ESTADO`: `Activo`, `Desarrollo`, `Inactivo`. Solo `Activo` y `Desarrollo`
  entran al motor.

El motor construye la instancia a invocar leyendo `PARAM_XML`:

```xml
<CorXml>
  <NOMBRE>ENSAMBLADO_DATA</NOMBRE>
  <PROCESO>Namespace.Clase, Ensamblado</PROCESO>
  <EVENTO>NombreDelMetodo</EVENTO>
</CorXml>
```

Las "configuraciones" (`CONTROL_REGLAS_F`) son formularios dinamicos
(encuestas) que capturan variables de entrada del usuario al ejecutar. Estos
valores se almacenan en `FORX_DATA` (tabla del motor de encuestas) y se pasan
al ensamblado como `DocParametros`.

## 3. Ubicacion en el menu / consumidores (grep evidence)

- `topBar.Modulo = "000802"`, `topBar.Ubicacion = "Home/GENERAL/REGLAS"`.
- Referencia en `Bootstrap/Bootstrap.vbproj`: solo `Content Include` y sus
  dependencias designer/vb. No hay `Redirect` ni `href` desde otro `.aspx`.
- Consumidores de `cl_gestion_reglas` (quien ejecuta reglas en produccion):
  - `Bootstrap/Formularios/Modulos/Visal/clases/cl_historias.vb`
  - `Bootstrap/Formularios/Modulos/Tareas/workflow/AdmWorkflow.vb`
  - `Bootstrap/Formularios/Modulos/Documental/Reglas/cl_form_reglas_pruebas.vb`
  - `Bootstrap/Formularios/Modulos/Documental/Controles/ctrCargaTablas.ascx.vb`
  - `Bootstrap/Formularios/Modulos/Documental/Controles/crtCargaEncuestaII.ascx.vb`
  - `Bootstrap/Formularios/Modulos/Documental/Clases/cl_gestion_formularios.vb`
  - `Bootstrap/Formularios/Modulos/Documental/Clases/cl_gestion_campo.vb`
  - `Bootstrap/Formularios/Modulos/Configuracion/NEWFRONT_adm_empresas.aspx.vb`

En resumen: gen_reglas es el editor. La ejecucion ocurre desde el resto del
sistema (Visal historias, Tareas workflow, controles Documental, admin de
empresas) invocando a `cl_gestion_reglas` con el `RegRegla` correspondiente.

## 4. Layout visual (ASCII art wireframe: tabs, grids, modales)

```
+===========================================================================+
| MasterTopBar (Titulo REGLAS, botones Save/Delete/Clear ocultos via CSS)   |
+===========================================================================+
| [ Documento de configuracion ] [ Historial ]                              |
+---------------------------------------------------------------------------+
| Estado[v] Codigo[v] Nombre[____________] Grupo[__________]                |
| Observaciones [_______________________________________________]           |
|                                                                           |
| > Configuraciones (accordion collapse)                                    |
|   [Agregar parametros] [Inactivar] [Actualizar]                           |
|   +------------------------------------------------------------------+   |
|   | GridView1 (combinaciones - vestigio: columnas Grupo/Marca/Medio)  |   |
|   +------------------------------------------------------------------+   |
|   +------------------------------------------------------------------+   |
|   | GridParametros [x] Nombre | Descripcion | [Editar]                |   |
|   +------------------------------------------------------------------+   |
|                                                                           |
| > Tareas (accordion collapse)                                             |
|   [Agregar nuevo regla] [Inactivar] [Actualizar]                          |
|   +------------------------------------------------------------------+   |
|   | Gridtareas [x] Nombre | Descripcion | Orden(edit) | [Editar]      |   |
|   +------------------------------------------------------------------+   |
+---------------------------------------------------------------------------+
| TAB Historial: GridHistorico Documento | Fecha | Observaciones | Estado   |
+===========================================================================+

MODAL PanelTarea (Regla) - Tabs [Configuracion | Historial]  1200px wide
+---------------------------------------------------------------------------+
| IZQUIERDA (col-7)               | DERECHA (col-5)                         |
| Nombre  Tipo  Orden             | ctrFormDinamico (form dinamico          |
| Estado                          |   que renderiza PARAM_XML como          |
| Descripcion funcion             |   inputs y guarda a FORX_DATA)          |
| PARAM_XML (textarea x16)        | [Ejecutar regla ->]                     |
| Script previo (textarea x3)     | LiteralErrorsql (toastr like)           |
| [Crear/actualizar] [Nueva]      | ListViewError (lista de mensajes)       |
| [Eliminar]                      |                                         |
+---------------------------------------------------------------------------+
| Tab Historial: GridHistorial Usuario/Fecha/Registros/Duracion/Llamado/Err |
+---------------------------------------------------------------------------+

MODAL PanelConfiguracion (Configuracion de formulario) 1400px wide
+---------------------------------------------------------------------------+
| IZQUIERDA (col-4)               | DERECHA (col-8)                         |
| Nombre    Estado                | crtCargaEncuestaII (editor de encuesta) |
| Formulario [v]                  |                                         |
| Descripcion                     |                                         |
| [Agregar] [Nueva] [Eliminar]    |                                         |
+---------------------------------------------------------------------------+
```

## 5. Controles servidor (tabla: ID | Tipo | Proposito)

| ID | Tipo | Proposito |
|---|---|---|
| topBar | MasterTopBar | Barra superior con Save/Delete/Clear |
| ModalDialog | ModalDialog | Popup de mensajes globales |
| HiddenFieldsucursal | HiddenField | Sucursal del usuario logueado |
| HiddenDelete | HiddenField | Marca de fila a eliminar |
| cmbestado | DropDownList | Estado del documento (Activo/Desarrollo/Inactivo) |
| cmbdocumento | DropDownList (AutoPostBack) | Codigo del documento de reglas |
| txtnombre | TextBox | Nombre del documento |
| txtgrupo | TextBox | Agrupador logico |
| txtdescripcion | TextBox multiline | Observaciones |
| Linkbuttonaddconfiguracion | LinkButton | Abre modal Configuracion (nueva) |
| LinkbuttonInactivaConfiguracion | LinkButton | Inactiva configuraciones marcadas |
| LinkbuttonActualizaConfiguracion | LinkButton | Recarga grid de parametros |
| GridView1 | GridView | Grid heredado (Grupo/Marca/Medio/Dias) sin data-binding activo |
| GridParametros | GridView | Grid de configuraciones (`CONTROL_REGLAS_F`) |
| Linkbutton1 | LinkButton | Abre modal Regla (nueva) |
| Linkbutton8 | LinkButton | Actualiza grid de tareas |
| Gridtareas | GridView | Grid de reglas (`CONTROL_REGLAS_R`), campo Orden editable |
| GridHistorico | GridView | Historial de documentos (tab Historial) |
| PanelTarea | Panel (modal) | Modal de edicion de regla |
| txtnombreregla | TextBox | Nombre de la regla |
| cmbtipoejecucion | DropDownList | mDATA / Execute / Ensamblado |
| txtordenejecucion | TextBox Number | Orden de ejecucion |
| cmbestadotarearegla | DropDownList | Estado de la regla |
| txtfuncion | TextBox multiline (3) | Descripcion de la funcion |
| txtconfiguracionregla | TextBox multiline (16) | `PARAM_XML` |
| txtprevioscript | TextBox multiline (3) | Script previo |
| ctrFormDinamico | UserControl | Editor de formulario dinamico derivado del XML |
| Linkbutton4 | LinkButton | Crear/actualizar regla |
| Linkbutton7 | LinkButton | Limpiar modal (Nueva) |
| Linkbutton5 | LinkButton | Eliminar (sin handler cableado) |
| Linkbutton6 | LinkButton | Ejecutar regla |
| LiteralErrorsql | Literal | Resultado de la ejecucion |
| ListViewError | ListView | Lista de mensajes usuario/mensaje/tecnico |
| GridHistorial | GridView | Historial de ejecuciones de la regla actual |
| PanelConfiguracion | Panel (modal) | Modal de configuracion de formulario |
| cmbformualario | DropDownList | Encuesta origen (`ENCUESTAS_MOV.CODIGO`) |
| txtnombreconfiguracion | TextBox | Nombre de la configuracion |
| cmbestadoconfiguracion | DropDownList | Estado de la configuracion |
| txtdetalleconfiguracon | TextBox multiline | Descripcion |
| crtCargaEncuestaII | UserControl | Editor de la encuesta seleccionada |
| LinkbuttonModAddconfiguracion | LinkButton | Guardar configuracion |
| LinkbuttonModNuevaconfiguracion | LinkButton | Limpiar modal configuracion |
| LinkbuttonModDelConfiguracion | LinkButton | Eliminar configuracion |
| LiteralGuardado / LiteralConfiguraciones | Literal | Feedback de guardado |

## 6. Eventos servidor (cada handler y su logica)

- `Page_Load`: si `Not IsPostBack` fija topBar, carga combos (`TipoEjecucion`,
  `ListaEstado`, `ListaDocumento`, `ListaEstadoRegla`, `ListaFomularios`,
  `ListaEstadoConfiguracion`). Redirige a login si no hay sesion.
- `cmbdocumento_SelectedIndexChanged` -> `CargarDocumento()` (carga cabecera y
  ambos grids).
- `topBar_Save` -> `GuardarDocumento()` (upsert de `CONTROL_REGLAS`).
- `topBar_Delete` -> `EliminarDocumento()` + `LimpiarDoc(1)` + `ListaDocumento()`.
- `topBar_Clear` -> `LimpiarDoc(1)` + `ListaDocumento()`.
- `Linkbutton8_Click` -> `ConsultarDetalle()` (refresca `Gridtareas`).

Modal Regla:

- `Linkbutton1_Click` -> `LimpiarModalReglas(1)` y abre modal via
  `ScriptManager.RegisterStartupScript` (`bootstrap.Modal(...).show()`).
- `LinkButton13_Click` (fila Editar) -> setea `Me.RegRegla` con el `REG` de la
  fila, llama `CargarModalReglas()` y abre modal.
- `Linkbutton4_Click` -> `AgregarDetalles()` (upsert en `CONTROL_REGLAS_R`).
- `Linkbutton7_Click` -> `LimpiarModalReglas(1)`.
- `Linkbutton6_Click` -> `EjecutarRegla()`.
- `txtroworden_TextChanged` -> UPDATE inline de `CONTROL_REGLAS_R.ORDEN`.

Modal Configuracion:

- `Linkbuttonaddconfiguracion_Click` -> `LimpiarModalConfig(1)` y abre modal.
- `LinkButton13_Click2` (fila Editar) -> `CargarModalConfig()` y abre modal.
- `LinkbuttonInactivaConfiguracion_Click` -> UPDATE `ESTADO='Inactivo'` en
  `CONTROL_REGLAS_F` para filas marcadas.
- `LinkbuttonActualizaConfiguracion_Click` -> `ConsultarDetalleCongf()`.
- `LinkbuttonModAddconfiguracion_Click` -> `AgregarDetallesFormulario()`.
- `LinkbuttonModNuevaconfiguracion_Click` -> `LimpiarModalConfig(1)`.
- `LinkbuttonModDelConfiguracion_Click` -> `EliminarFormulario(Me.RegConfig)`.

## 7. Tablas SQL: CONTROL_REGLAS* (esquema completo + relaciones)

Base `[dbx.GENE].dbo`. Solo se detectaron **3 tablas** en uso:
`CONTROL_REGLAS`, `CONTROL_REGLAS_R`, `CONTROL_REGLAS_F`, mas `CONTROL_REGLAS_H`
para historial. No hay `CONTROL_REGLAS_D` en el codigo.

### CONTROL_REGLAS (cabecera / documento)

- `SUCURSAL`, `DOCUMENTO` (PK logica), `USUARIO`, `FECHA_NOV`, `OBSERVACION`,
  `ESTADO`, `NOMBRE`, `GRUPO`, `REG` (autoincremental), `FECHA`.

### CONTROL_REGLAS_R (reglas / tareas)

- `SUCURSAL`, `REGLA` (FK -> `CONTROL_REGLAS.DOCUMENTO`), `REG` (PK).
- `TIPO` (mDATA / Execute / Ensamblado), `ORDEN` (int).
- `PARAM_XML` (nvarchar large), `SCRIPT` (nvarchar).
- `NOMBRE`, `DESCRIPCION`, `ESTADO`.

### CONTROL_REGLAS_F (configuraciones de formulario)

- `SUCURSAL`, `REGLA` (FK), `REG` (PK).
- `FORMULARIO` (FK -> `ENCUESTAS_MOV.CODIGO`).
- `NOMBRE`, `DESCRIPCION`, `ESTADO`.

### CONTROL_REGLAS_H (historial de ejecucion)

- `REG` (PK), `USUARIO`, `FECHA_REG`.
- `REGLA` (documento), `CODIGO` (= `CONTROL_REGLAS_R.REG`).
- `LLAMADO` (metodo/parametros invocados), `REGISTROS` (int), `MSG_ERROR`,
  `DURACION` (ms).

### Tablas relacionadas fuera del motor

- `FORX_DATA`: valores capturados por `ctrFormDinamico` (llave logica
  SUCURSAL + TABLA + REFERENCIA + CODIGO + ID_REGLA + PROPIEDAD).
- `ENCUESTAS_MOV`: catalogo de encuestas/formularios origen.

Relaciones:

```
CONTROL_REGLAS 1 --- N CONTROL_REGLAS_R  (via REGLA = DOCUMENTO)
CONTROL_REGLAS 1 --- N CONTROL_REGLAS_F  (via REGLA = DOCUMENTO)
CONTROL_REGLAS_R 1 --- N CONTROL_REGLAS_H (via CODIGO = REG)
CONTROL_REGLAS_R 1 --- N FORX_DATA        (via ID_REGLA = REG)
CONTROL_REGLAS_F N --- 1 ENCUESTAS_MOV   (via FORMULARIO = CODIGO)
```

## 8. Los "verbos" reales encontrados (con conteo)

Grep de `Ensamblado` en `Formularios/Modulos`:

- `Bootstrap/Formularios/Modulos/General/gen_reglas.aspx.vb`: 1 (lista combo).
- `Bootstrap/Formularios/Modulos/Documental/Clases/cl_gestion_reglas.vb`: 2
  (dos `Select Case tipo_ejecucion` con branch `"Ensamblado"`).
- `Bootstrap/Formularios/Modulos/Utilidades/NEWFRONT_web_scraping.aspx.vb`: 1
  (contexto no relacionado).
- `Bootstrap/Formularios/Modulos/Turnos/tur_reglastraslado.aspx.vb`: 1
  (contexto no relacionado).

Total en el motor: **1 verbo implementado** (`Ensamblado`). `mDATA` y `Execute`
aparecen en el combo `cmbtipoejecucion` pero **no tienen branch** en el
`Select Case` de `cl_gestion_reglas.EjecutarRegla`.

## 9. 3 modos de ejecucion: mDATA / Execute / Ensamblado

El combo `cmbtipoejecucion` (llenado en `TipoEjecucion()`) ofrece tres modos:

- **mDATA**: pensado para consultas SQL parametrizadas via `AdmDatos`. **No
  implementado** en `cl_gestion_reglas`. Actualmente no hace nada si se
  selecciona.
- **Execute**: pensado para ejecutar `SCRIPT` (comando SQL directo). **No
  implementado**. `SCRIPT` se guarda en la tabla pero no se ejecuta.
- **Ensamblado**: unico modo funcional. Instancia por reflexion la clase
  indicada en `<PROCESO>` del `PARAM_XML` (formato `Namespace.Clase, Ensamblado`),
  setea sus propiedades (`DocParametros`, `Usuario`, `Sucursal`, `Formulario`,
  `Referencia`, `Referenciaii`, `RegTabla`, `FormGuardado`, `Produccion`,
  `Modulo`, `DATOS_UNICOS_POR_USUARIO`) e invoca el metodo `<EVENTO>` via
  `MethodInfo.Invoke`. Los mensajes emergentes de la clase se agregan a
  `man_MensajeEmergente` para mostrarlos en `ListViewError`.

`EjecutarRegla` filtra por `ESTADO IN ('Activo','Desarrollo')` y respeta el
`ORDEN`. Detiene la cadena en el primer error emitido.

## 10. Directivas Register / dependencias

```asp
<%@ Page Language="vb" AutoEventWireup="false"
    CodeBehind="gen_reglas.aspx.vb"
    Inherits="GestionMovil.gen_reglas"
    MasterPageFile="~/Formularios/Modulos/Dashboard/MasterFormsii.Master"
    ValidateRequest="false" Async="true" %>
<%@ Register Assembly="AjaxControlToolkit" Namespace="AjaxControlToolkit" TagPrefix="cc1" %>
<%@ Register Src="~/Controles/ModalDialog.ascx" TagPrefix="uc1" TagName="ModalDialog" %>
<%@ Register Src="~/Controles/MasterTopBar.ascx" TagPrefix="uc1" TagName="MasterTopBar" %>
<%@ Register Src="~/Formularios/Modulos/General/controles/ctrFormDinamico.ascx"
    TagPrefix="uc1" TagName="ctrFormDinamico" %>
<%@ Register Src="~/Formularios/Modulos/Documental/Controles/crtCargaEncuestaII.ascx"
    TagPrefix="uc1" TagName="crtCargaEncuestaII" %>
```

Dependencias de codigo:

- `MotherData.AdmDatos` (DAL), `Funciones.ValidaError`, `Funciones.tipdoc`
  (consecutivos), `Optimizer.ComboFill` (llenado de combos), `carga_empresa`
  + `mbase_empresa` (bootstrap de conexion), `sideNavBar`, `navBarTop`
  (controles del master).
- `cl_gestion_reglas` (Documental/Clases) para ejecutar.

Nota: **no se registra `ctrDependencias`** en este ASPX.

## 11. Modales / popups (agregar condicion, agregar accion, etc)

Solo hay dos modales, ambos abiertos con `bootstrap.Modal(...).show()` via
`ScriptManager.RegisterStartupScript`:

- **PanelTarea** (`modal-lg` 1200px). Doble pestana:
  - "Configuracion": lado izquierdo con los campos escalares de la regla y
    lado derecho con `ctrFormDinamico` que renderiza `PARAM_XML` como
    formulario dinamico. Boton "Ejecutar regla" invoca `EjecutarRegla()`.
  - "Historial": `GridHistorial` con las ejecuciones (`CONTROL_REGLAS_H`).
- **PanelConfiguracion** (`modal-lg` 1400px). Sin pestanas. Lado izquierdo con
  los campos de la configuracion; lado derecho con `crtCargaEncuestaII`
  (editor completo de la encuesta seleccionada).

No existe un modal "agregar condicion" ni "agregar accion" separados: toda la
logica se edita como XML libre dentro de `txtconfiguracionregla`.

## 12. Reglas de negocio detectadas

- Consecutivo automatico del documento: si `cmbdocumento.SelectedValue` esta
  vacio, `Funciones.tipdoc.Consecutivo("REG1", "", Session("Empresa"))` genera
  el nuevo codigo.
- Upsert por `COUNT(*)`: se decide INSERT/UPDATE segun cuente `0` filas en
  `CONTROL_REGLAS` / `CONTROL_REGLAS_R` / `CONTROL_REGLAS_F` para el `REG`
  actual. Escape simple `Replace("'", "''")` unicamente sobre `PARAM_XML` y
  `DESCRIPCION` de configuraciones.
- El estado inicial al crear es siempre `Desarrollo`.
- Al eliminar cabecera se borra tambien `CONTROL_REGLAS_R` (pero **no** se
  borra `CONTROL_REGLAS_F` ni `CONTROL_REGLAS_H`).
- El motor solo ejecuta reglas con `ESTADO IN ('Activo','Desarrollo')`.
- El orden de ejecucion viene de `CONTROL_REGLAS_R.ORDEN` y es editable inline
  en `Gridtareas`.
- Al cargar una configuracion existente, `cmbformualario` se bloquea
  (`disabled`) para impedir cambio de formulario origen.
- La ejecucion desde el modal usa `topBar.Modulo & "_CAPITALES"` como nombre de
  tabla contenedora en `FORX_DATA` (tabla del motor de encuestas).

## 13. Riesgos / deuda tecnica

- **SQL injection**: todas las consultas concatenan valores de `Session`,
  `TextBox` y `SelectedValue` sin parametrizar. `PARAM_XML` solo aplica
  duplicado de apostrofe.
- **Verbos declarados pero no implementados** (`mDATA`, `Execute`) son trampas
  silenciosas: si el usuario los selecciona, la regla no hace nada y no arroja
  error visible.
- **Grid huerfano** `GridView1` con columnas de logistica
  (Grupo/Marca/Medio/Tiempo alistamiento...) que no reciben binding en el
  code-behind: vestigio de otro modulo copiado.
- **`Linkbutton5` (Eliminar regla)** no tiene handler cableado en `OnClick`.
- **Borrado incompleto** de cabecera: quedan huerfanos en `CONTROL_REGLAS_F` y
  `CONTROL_REGLAS_H`.
- **No hay filtro por SUCURSAL** al listar documentos (`ListaDocumento`), pero
  si al leer configuraciones y reglas.
- **JS inline duplicado**: tres bloques `BeginRequestHandler` / `EndRequestHandler`
  con el mismo nombre en el archivo (sobreescriben handlers previos).
- **Configuraciones (`CONTROL_REGLAS_F`)** no se muestran en un solo lugar
  claro y comparten grid con reglas.
- **Estructura del PARAM_XML** solo esta documentada de facto por
  `cl_gestion_reglas` (busca `CorXml` con `NOMBRE="ENSAMBLADO_DATA"`); no hay
  esquema publicado.

## 14. Puntos de reconstruccion en Claude Design

Al reconstruir en Claude Design conviene:

1. **Modelar 4 entidades explicitas**: RulesDocument, Rule (con `type`, `order`,
   `paramXml`, `script`, `status`), FormBinding (formulario+regla) y
   ExecutionHistory.
2. **Sustituir `PARAM_XML` de texto libre** por un editor estructurado:
   - Modo Assembly: campos `assemblyType`, `methodName`, y lista de propiedades
     inyectadas (map key/value).
   - Modo Sql: editor SQL con parametros nombrados.
   - Modo Http/Function (nuevo): endpoint + payload.
3. **Implementar de verdad los tres modos** (`mDATA` / `Execute` /
   `Ensamblado`) o eliminar del combo los no soportados.
4. **Encuestas dinamicas**: reemplazar `ctrFormDinamico` + `FORX_DATA` por un
   JSON Schema Form (react-jsonschema-form o equivalente).
5. **Ejecucion asincrona con jobs**: encolar ejecuciones en vez de correr
   sincronamente en el postback. Persistir historial con `startedAt`,
   `finishedAt`, `duration`, `rowsAffected`, `errorMessage`, `logs`.
6. **Seguridad**: parametrizar todo SQL, validar `assemblyType` contra un
   allowlist, sandboxear la carga por reflexion o mover a plugins declarados.
7. **UI/UX**:
   - Reemplazar los 2 modales por un split pane (lista de reglas a la
     izquierda, editor a la derecha) con tabs `Editor | Formulario | Historial`.
   - Ordenar reglas via drag-and-drop, no textbox inline.
   - Estados como pills (Activo verde, Desarrollo amarillo, Inactivo gris).
   - Boton "Test run" con dry-run que devuelve el plan de ejecucion.
8. **CRUD consistente**: borrar cabecera debe cascadear a `_R`, `_F` y `_H`.
9. **Audit trail**: `createdBy/updatedBy/updatedAt` en cada entidad.
10. **APIs REST**: `GET/POST/PATCH/DELETE /rules-documents`, `/rules`,
    `/rules/{id}/executions`, `/rules/{id}/run`.

Con esto un dev puede reconstruir el motor manteniendo la semantica actual
(documento -> reglas -> historial + formulario dinamico) y a la vez cerrar la
deuda tecnica listada en la seccion 13.
