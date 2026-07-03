---
tipo: spec-modulo
modulo: NEWFRONT_adm_empresas.aspx
carpeta: Configuracion
proposito: Administracion multi-tenant de empresas (SUCURSAL) con modulos, usuarios, parametros, integraciones y reglas
---

# NEWFRONT_adm_empresas - Spec para reconstruir

## 1. Que hace este modulo

Es la consola de administracion multi-tenant de ECOREX. El "tenant" en la BD se llama SUCURSAL y este modulo permite crear, editar y eliminar empresas (tenants) desde una sola pantalla, definiendo su ficha (nombre comercial, razon social, NIT, ciudad, estado, tipo, contrato, contador, revisor fiscal, Slack). El titulo topBar es "CREAR EMPRESAS", codigo de modulo 000072 y ubicacion en el menu "Home/Configuracion/Crear Empresas".

Ademas de la ficha, la pantalla concentra toda la configuracion secundaria del tenant en acordeones: modulos asignados (SUCURSAL_MOD, SUCURSAL_GRUPOS, SUCURSAL_GRUPOS_X_PERMISOS, SUCURSAL_GRUPOS_X_REPORTES), actividades habilitadas (TIPO_TAR_EMPRESA), parametros de configuracion (SUCURSAL_PAR), usuarios asignados (USUARIO.SUCURSAL), integraciones (SUCURSAL_INT con controles de usuario por conector), y reglas de negocio (FORX_DATA + CONTROL_REGLAS). Incluye utilidades poderosas para clonar datos entre empresas (Cargar Datos), copiar formularios completos con sus preguntas y reglas (Copiar formularios), y consultar/traer datos de una base externa (Buscar datos externos).

El segundo tab, "Todas las empresas", es un listado plano de SUCURSAL con filtro por nombre para dar visibilidad rapida a todas las empresas creadas en el sistema.

## 2. Ubicacion / consumidores

Ruta: `Bootstrap/Formularios/Modulos/Configuracion/NEWFRONT_adm_empresas.aspx`. Master: `MasterFormsII.Master`. Namespace codebehind: `GestionMovil` (por convencion del proyecto, sin declararlo).

Grep evidence (busqueda de "NEWFRONT_adm_empresas" y "adm_empresas"): solo aparece dentro de sus 3 archivos y en `Bootstrap.vbproj`. No hay Response.Redirect ni Server.Transfer desde otros modulos; se accede unicamente desde el menu del sistema (modulo 000072). El unico "consumer" es el menu.

## 3. Layout visual (ASCII wireframe)

```
+------------------------------------------------------------------+
| TopBar (CREAR EMPRESAS)   [Guardar] [Limpiar] [Eliminar]         |
+------------------------------------------------------------------+
| [ Creacion de Empresas ]  [ Todas las empresas ]                 |
+------------------------------------------------------------------+
| TAB 1 - Creacion de Empresas                                     |
|                                                                  |
| +-- Card: Datos de la empresa ------------------------------+    |
| | [Codigo v] [Estado v] [Nombre comercial........]          |    |
| | [Ciudad v] [Tipo v]   [Direccion...............]          |    |
| +-----------------------------------------------------------+    |
| +-- Card: Contacto -----------------------------------------+    |
| | [Telefonos] [Email] [Contacto]                            |    |
| +-----------------------------------------------------------+    |
| +-- Card: Datos juridicos ----------------------------------+    |
| | [NIT] [Razon social] [Inicio contrato] [Termino]          |    |
| +-----------------------------------------------------------+    |
| +-- Card: Contador y Revisor Fiscal ------------------------+    |
| | Contador:   [Nombre] [Cedula] [Tarjeta]                   |    |
| | Revisor F.: [Nombre] [Cedula] [Tarjeta]                   |    |
| +-----------------------------------------------------------+    |
| +-- Card: Slack y descripcion ------------------------------+    |
| | [Slack / Token webhook............]                       |    |
| | [Descripcion multiline............]      (LtrError)       |    |
| +-----------------------------------------------------------+    |
|                                                                  |
| >  ACORDEON: Modulos             [Copiar Modulos] [Eliminar]     |
|      combo cmbempresabase + GridModulos + GridViewRmodulos       |
| >  ACORDEON: Actividades         [Cargar actividad] [Eliminar]   |
|      combo cmbListaactividades + GridViewActividades             |
| >  ACORDEON: Cargar Datos        [Listar estructuras][Copiar][X] |
|      combo cmbageciacopia + CheckBoxList cmbtablas               |
| >  ACORDEON: Copiar formularios  [Copiar formulario][Copiar dat] |
|      cmbempresaformulario + cmbfomulariosorigen + GridCuestionarios
| >  ACORDEON: Buscar datos externos [Listar estructuras]          |
|      txtconexion + txtbuscado + CheckBoxTablasCondatos           |
| >  ACORDEON: Cargar datos externos en formularios                |
|      cmbgruporegla + cmbreglas + cmbreglastarea                  |
|      + ctrFormDinamico + [Save][Ejecutar regla]                  |
|      + GridDetalleReglas (cards con orden) + ListViewError       |
| >  ACORDEON: Usuarios            [Agregar] [Quitar]              |
|      combo cmbusuarios + GridViewUsuario                         |
| >  ACORDEON: Configuraciones     [Agregar] [Quitar]              |
|      txtcodigo/txtparamdetalle/txtvalor/Checkemailmasivo         |
|      + GridParametros                                            |
| >  ACORDEON: Integraciones       [Agregar] [Quitar]              |
|      cmbintegracion + GridViewIntegracion                        |
|         cada fila renderiza uno de los 11 ctrl* de integracion   |
+------------------------------------------------------------------+
| TAB 2 - Todas las empresas                                       |
| +-- Card Buscar empresas: [Nombre filtro] [Filtrar]              |
| +-- Card Empresas registradas: Griddatos (Codigo,Nombre,...)     |
+------------------------------------------------------------------+
```

## 4. Controles servidor (tabla)

| Grupo | Control | Tipo | Proposito |
|---|---|---|---|
| Datos empresa | cmbcodigo | DropDownList AutoPostBack | Selector maestro de empresa (SUCURSAL.CODIGO). Al cambiar, dispara `MostrarEmpresa` |
| Datos empresa | cmbestado | DropDownList | SUC_ESTADOS |
| Datos empresa | txtnombre | TextBox | SUCURSAL.NOMBRE |
| Datos empresa | cmbciudad | DropDownList | CIUDADES |
| Datos empresa | cmbtipoempresa | DropDownList | SUC_TIPO |
| Datos empresa | txtdireccion | TextBox | SUCURSAL.DIRECCION |
| Contacto | txttels / txtemail / txtcontacto | TextBox | Telefonos, correo, contacto |
| Juridico | txtnitempresa / txtnombreempresa | TextBox | NIT y razon social (NOMBRE_REAL) |
| Juridico | txtfechainicio / txtfechafin | TextBox TextMode=Date | Vigencia contrato |
| Contador | txtcontador / txtcontadorcedula / txtcontadortarjeta | TextBox | NOMBRE_CONTADOR, ID_CONTADOR, TJ_CONTADOR |
| Revisor | txtrevisor / txtrevisorcedula / txtrevisortarjeta | TextBox | NOMBRE_FISCAL, ID_FISCAL, TJ_FISCAL |
| Otros | txtslack | TextBox | Webhook Slack |
| Otros | txtobservaciones | TextBox MultiLine | Descripcion |
| Otros | LtrError / LiteralErrorsql | Literal | Mensajes de error |
| Modulos | cmbempresabase | DropDownList | Empresa origen para clonar modulos |
| Modulos | GridModulos / GridViewRmodulos | GridView | Modulos asignados + reportes (col REG oculta) |
| Modulos | Linkbutton8 / Linkbutton4 | LinkButton | Copiar / Eliminar modulos |
| Actividades | cmbListaactividades | DropDownList | Categorias disponibles |
| Actividades | GridViewActividades | GridView | Actividades relacionadas (HiddenFieldCategoria) |
| Cargar Datos | cmbageciacopia | DropDownList | Empresa origen para copiar tablas |
| Cargar Datos | cmbtablas | CheckBoxList | Tablas con datos detectadas dinamicamente |
| Copiar Form | cmbempresaformulario | DropDownList AutoPostBack | Empresa origen (dispara `ListaFormularios`) |
| Copiar Form | cmbfomulariosorigen | DropDownList | Formulario a clonar |
| Copiar Form | GridCuestionarios | GridView paginado | Formularios existentes en la empresa destino |
| Externo | txtconexion / txtbuscado | TextBox MultiLine | Cadena conexion externa + lista de IDs |
| Externo | CheckBoxTablasCondatos | CheckBoxList | Resultado del sondeo externo |
| Reglas | cmbgruporegla / cmbreglas / cmbreglastarea | DropDownList AutoPostBack | Grupo -> Regla -> Tarea |
| Reglas | ctrFormDinamico | UserControl | Renderiza el XML de parametros de la regla |
| Reglas | GridDetalleReglas | GridView (cards) | Reglas activas con TextBox de orden AutoPostBack |
| Reglas | ListViewError | ListView | Mensajes de EjecutarRegla |
| Usuarios | cmbusuarios / GridViewUsuario | DropDownList + GridView | Asignacion USUARIO.SUCURSAL |
| Parametros | txtcodigo / txtparamdetalle / txtvalor / Checkemailmasivo | Inputs | SUCURSAL_PAR |
| Parametros | GridParametros | GridView | REG,CODIGO,NOMBRE,VALOR,FLAG_EMAIL_MASIVO |
| Integraciones | cmbintegracion | DropDownList AutoPostBack | Alimentado desde PARAMXML 0202 |
| Integraciones | GridViewIntegracion | GridView | Fila muestra el `ctrl*` correspondiente (RowDataBound) |
| Tab 2 | txtfiltronombre / Griddatos | TextBox + GridView | Listado completo con filtro por nombre |
| Comunes | HiddenFieldsucursal / HiddenFieldConfigura / HiddenDelete | HiddenField | State/hooks JS |
| Comunes | topBar | uc1:MasterTopBar | Guardar / Limpiar / Eliminar (eventos topBar.Save, topBar.Clear, topBar.Delete) |
| Comunes | ModalDialog | uc1:ModalDialog | Mensajes modales |

## 5. Eventos servidor

| Handler | Que hace |
|---|---|
| `Page_Load` | Configura topBar (000072), llena todos los combos con SUCURSAL/CIUDADES/SUC_ESTADOS/SUC_TIPO/USUARIO, dispara `ListaIntegraciones`, `LimpiarFormulario(1)`, `ListadoEmpresas`, `ListaEmpresas`, `ListaReglasGrupos`, `ListaReglas`. Redirige a login si Session("Nombre") vacio |
| `cmbcodigo_SelectedIndexChanged` | `MostrarEmpresa(codigo)` -> carga ficha, actividades, formularios, parametros, modulos, usuarios, integraciones y reglas |
| `topBar_Save` | `CrearCatalogo` -> INSERT o UPDATE en SUCURSAL segun exista el codigo |
| `topBar_Delete` | `Quitarempresa` -> DELETE de SUCURSAL |
| `topBar_Clear` | `LimpiarFormulario(1)` |
| `LinkButtonAgregamodulo_Click` | `agregarModulo` -> clona 4 tablas (SUCURSAL_MOD, SUCURSAL_GRUPOS, SUCURSAL_GRUPOS_X_PERMISOS, SUCURSAL_GRUPOS_X_REPORTES) desde `cmbempresabase` |
| `LinkButtonQuitarModulo_Click` | `eliminarModulo` -> DELETE SUCURSAL_MOD por REG marcado en GridModulos |
| `LinkbuttonAddactividad_Click` / `LinkbuttonDelActividad_Click` | INSERT/DELETE en TIPO_TAR_EMPRESA |
| `Linkbutton1_Click1` | `PrepararbaseDatos` -> escanea sys.tables por columna SUCURSAL, cuenta filas y llena `cmbtablas` |
| `Linkbutton10_Click` | `CopiarTablasSeleccionadas` -> por cada tabla marcada hace `INSERT ... SELECT` reemplazando SUCURSAL |
| `Linkbutton11_Click` | `PrepararbaseDatosExterno` usa `FindDatasetExterno(txtconexion.Text,...)` |
| `cmbempresaformulario_SelectedIndexChanged` | `ListaFormularios` en base a ENCUESTAS_MOV |
| `Linkbutton12_Click` | `GenerarCopiaFormularios` -> clona ENCUESTAS_MOV + preguntas + contenedores + FORX_DATA (bloque grande) |
| `LinkButtonEditarRegla_Click` / `LinkButtonEliminarRegla_Click` | Editar/eliminar filas de FORX_DATA (reglas asociadas) |
| `Linkbutton2_Click_regla` | `AsociarReglaObjeto` -> genera token y llama `ctrFormDinamico.GuardarFormulario()` |
| `Linkbutton14_Click` | `EjecutarRegla` -> instancia `cl_gestion_reglas`, ejecuta y publica mensajes |
| `txtordenejecutar_TextChanged` | Actualiza FORX_DATA.ORDEN_REGLA de la fila |
| `LinkButtonagregarusuario_Click` / `LinkButtonquitarusuario_Click` | UPDATE USUARIO.SUCURSAL |
| `LinkButton2_Click` / `LinkButton3_Click` | Agrega / quita SUCURSAL_PAR |
| `LinkButton5_Click` / `LinkButton6_Click` | Agrega / quita SUCURSAL_INT |
| `LinkButton9_Click` | Filtro TAB 2 -> `ListadoEmpresas()` |
| `GridViewIntegracion_RowDataBound` | Por cada fila, segun `HiddenFieldRowIntegracion` activa el Panel del `ctrl*` (SLACK, WHATSAPP, SIGMA, EMAIL, SENDGRID, AZUREBLOBSTORAGE, AWS, GOOGLECONSOLE, VISIONIA, EVOLUTION API, API MANAGER) |
| `cmbgruporegla_SelectedIndexChanged` / `cmbreglas_SelectedIndexChanged` / `cmbreglastarea_SelectedIndexChanged` | Cascada Grupo -> Regla -> Tarea y refresca plantilla |

## 6. Tablas SQL con snippets

Base logica: `[dbx.GENE].dbo.*` (alias resuelto por AdmDatos).

- **SUCURSAL** (tenant maestro): CODIGO, ESTADO, NOMBRE, CIUDAD, TIPO, DIRECCION, TELS, EMAIL, CONTACTO, NIT, NOMBRE_REAL, FECHA_INICIO, FECHA_FINAL, NOMBRE_CONTADOR, ID_CONTADOR, TJ_CONTADOR, NOMBRE_FISCAL, ID_FISCAL, TJ_FISCAL, OBSERVACIONES, SLACK.

  ```sql
  SELECT NOMBRE + ' ( ' + CODIGO + ' )' AS NOM, CODIGO, NOMBRE
  FROM [dbx.GENE].dbo.SUCURSAL ORDER BY NOMBRE
  ```

- **SUC_ESTADOS / SUC_TIPO / CIUDADES**: catalogos de combos.
- **USUARIO** (SUCURSAL, NOMBRE, EMAIL, ID_USUARIO): asignacion de usuario a tenant.
- **SUCURSAL_MOD** (EMPRESA, MODULO, COSTO, REG): modulos habilitados. Se clona con `INSERT ... SELECT ... NOT EXISTS`.
- **SUCURSAL_GRUPOS** / **SUCURSAL_GRUPOS_X_PERMISOS** / **SUCURSAL_GRUPOS_X_REPORTES**: agrupaciones y permisos por grupo. Se clonan en `agregarModulo`.
- **PRO_MODULOS**: catalogo global de modulos (JOIN para mostrar nombre).
- **SUCURSAL_INT** (SUCURSAL, REG, NOMBRE): integraciones habilitadas del tenant.
- **PARAMXML** (`[dbx.TRABAJO]`, CODIGO='0202'): XML con la lista de integraciones disponibles.
- **SUCURSAL_PAR** (SUCURSAL, CODIGO, NOMBRE, VALOR, FLAG_EMAIL_MASIVO, REG): parametros clave/valor por tenant.
- **TIPO_TAR / TIPO_TAR_R / TIPO_TAR_EMPRESA**: catalogo y asignacion de actividades habilitadas.
- **CONTROL_REGLAS / CONTROL_REGLAS_R**: catalogo de reglas y tareas.
- **FORX_DATA** (SUCURSAL, TABLA, REFERENCIA, CODIGO, PROPIEDAD, TIPO, VALOR, TOKEN, USUARIO, FECHA_REG, ID_REGLA, ORDEN_REGLA): almacen generico key/value (codigo `EJECUTA_PARAM`).
- **ENCUESTAS_MOV / ENCUESTAS_MOV_PREGUNTAS / ENCUESTAS_MOV_PREGUNTAS_T / ENCUESTAS_MOV_PREGUNTAS_CONTENEDOR**: formularios dinamicos (para funcion Copiar formularios).
- **FECHAS**: ano/periodo (ListaAno/ListaPeriodo).

Fragmentos ilustrativos:

```vb
' Alta idempotente de modulos desde una empresa base:
INSERT INTO [dbx.GENE].dbo.SUCURSAL_MOD (EMPRESA,MODULO)
SELECT '<destino>', MODULO
FROM [dbx.GENE].dbo.SUCURSAL_MOD
WHERE EMPRESA = '<origen>'
  AND NOT EXISTS (SELECT * FROM [dbx.GENE].dbo.SUCURSAL_MOD A
                  WHERE A.EMPRESA='<destino>' AND A.MODULO=SUCURSAL_MOD.MODULO)
```

```vb
' Filtro TAB 2:
SELECT CODIGO,NOMBRE,CONTACTO,TELS
FROM [dbx.GENE].dbo.SUCURSAL
WHERE CODIGO <> '' AND SUCURSAL.NOMBRE LIKE '%<filtro>%'
ORDER BY CODIGO DESC
```

## 7. Directivas Register / dependencias

Directivas Register (todas necesarias para reconstruir):

- `MasterTopBar.ascx` (`~/Controles`) - topBar con Save/Clear/Delete
- `AjaxControlToolkit` (assembly)
- `form_contacto.ascx`
- `ModalDialog.ascx`
- `ctrFormDinamico.ascx` (`~/Formularios/Modulos/General/controles`) - motor de formularios dinamicos con XML
- 11 controles de integracion en `~/ControlesIntegradores/`: `ctrlSlack`, `ctrlWhatsapp`, `ctrlSigma`, `ctrlMail`, `ctrlSendgrid`, `CtrlStorage`, `ctrlAWS`, `ctrlGoogleConsole`, `CtrlVisionIA`, `ctrlEvolutionAPI`, `ctrlAPIManager`

Master: `MasterFormsii.Master`. Necesita `sideNavBar` y `navBarTop` en el master.

Codebehind depende de: `MotherData.AdmDatos`, `Funciones.permisos`, `Funciones.ValidaError`, `Funciones.tipdoc`, `Funciones.Token`, `Funciones.Cadena.FormatDate`, `Optimizer`, `mbase_empresa` (variables globales `base_pricipal` y `base_login`), y la clase `cl_gestion_reglas` (motor de reglas).

Session usada: `Session("Empresa")`, `Session("Usuario")`, `Session("Nombre")`.

## 8. Modales / popups

- **ModalDialog** (uc1) global para mensajes.
- Codigo JS declara `myModal1` y `myModalitemsmp` con funciones `ModBuscarNit`, `BuscarClientes`, `ModBuscarRef`, `CerrarBusqueda`, `SeleccionFiltro` que abren/cierran modales de busqueda tipo NIT / item MP, pero en el ASPX visible **no hay markup** para esos modales - vestigio de una version anterior; hoy no se usan (los stubs `if (manejo_tipo == 1) { }` estan vacios).
- No hay modales bootstrap propios en este ASPX; los errores se muestran mediante Literals (`LtrError`, `LiteralErrorsql`) usando `sms_error` (div `.alert-danger`) o `MValidaError.modsms_error/modsms_success`.

## 9. Reglas de negocio

1. **CODIGO como PK del tenant**: si el codigo ya existe en SUCURSAL se hace UPDATE, si no INSERT. Si `cmbcodigo` viene vacio se pide consecutivo tipo `ES1` con `Funciones.tipdoc.Consecutivo`.
2. **Validacion minima**: `CrearCatalogo` solo exige `txtnombre`. `AgregarIntegracion`, `agregarModulo`, `AgregarActividades`, `AsociarReglaObjeto` exigen `cmbcodigo` con valor via `MValidaError.empty_field`.
3. **Copia de modulos**: idempotente con `NOT EXISTS`. Copia 4 tablas relacionadas (SUCURSAL_MOD + SUCURSAL_GRUPOS + SUCURSAL_GRUPOS_X_PERMISOS + SUCURSAL_GRUPOS_X_REPORTES) atomicamente, aunque sin transaccion.
4. **AgregarModulosDesdeUsuario** deriva los modulos desde `DIR_GRUPO` cruzando con USUARIO por ID (permite pre-cargar modulos partiendo de los usuarios ya asignados).
5. **Copia de datos entre empresas** (`PrepararbaseDatos` + `CopiarTablasSeleccionadas`): recorre `sys.tables` filtrando aquellas que tengan columna `SUCURSAL` y no esten en una blacklist muy larga (tablas transaccionales, temporales, catalogos globales). Cuenta filas en origen y en destino - solo ofrece las que tienen filas en origen y 0 en destino. La copia hace `SELECT * FROM <tabla> WHERE SUCURSAL='<origen>'` sustituyendo el codigo de tenant.
6. **Copia de formularios** (`GenerarCopiaFormularios`): 5 INSERT en cascada (ENCUESTAS_MOV, ENCUESTAS_MOV_PREGUNTAS_CONTENEDOR, ENCUESTAS_MOV_PREGUNTAS_T, ENCUESTAS_MOV_PREGUNTAS, FORX_DATA x2), con un UPDATE intermedio para reasignar `ID_PADRE` a los nuevos REG. Si el codigo destino ya existe, saca uno nuevo tipo `03D`.
7. **Reglas** (acordeon "Cargar datos externos en formularios"): asocia una regla al tenant con `token` (Funciones.Token.GenerateToken(30)) y guarda parametros en `FORX_DATA` codigo `EJECUTA_PARAM`. `Ejecutar regla` instancia `cl_gestion_reglas`, alimenta `DataReglas` con FORX_DATA y llama `EjecutarRegla`. Los errores se enumeran en `ListViewError`.
8. **Integraciones**: solo permite una integracion por tenant en modo alta rapida (chequeo `REG=0`). El grid renderiza dinamicamente el UserControl de cada integracion via `RowDataBound` + `Panel.Visible=true`.
9. **Eliminacion de tenant**: DELETE directo a `SUCURSAL` sin cascada ni confirmacion adicional (solo pide codigo).

## 10. Riesgos / deuda tecnica

- **Inyeccion SQL masiva**: casi todos los queries concatenan cadenas directamente desde controles y `Session` (`Session("Empresa")`). Cualquier reconstruccion debe pasar a parametros.
- **Sin transacciones**: `agregarModulo` hace 4 inserts sin `BEGIN TRAN`; si el 3 o el 4 fallan, deja el tenant en estado inconsistente. Igual para `GenerarCopiaFormularios` (5+ inserts y updates).
- **Eliminacion sin cascada**: `Quitarempresa` borra SUCURSAL pero no limpia SUCURSAL_MOD, SUCURSAL_PAR, SUCURSAL_INT, SUCURSAL_GRUPOS*, TIPO_TAR_EMPRESA, USUARIO.SUCURSAL, ENCUESTAS_MOV, FORX_DATA. Deja huerfanos.
- **Blacklist de tablas hardcodeada** en `PrepararbaseDatos`: lista larga de tablas prohibidas embebida como cadena. Fragil.
- **`db3dev.dbo.*` hardcodeado** en `GenerarCopiaFormularios` (subqueries de mapeo REG). Rompe portabilidad de entorno.
- **Codigo remanente de busqueda** (`myModal1`, `myModalitemsmp`, `SeleccionFiltro`, `HrefEliminar`, `HiddenDelete`): JS y controles sin markup ni handler asociado. Limpiar en la reconstruccion.
- **Grid duplicado**: `GridViewRmodulos` (segunda tabla de "Copiar Modulos") no tiene binding en el codebehind - se declara pero nunca se le hace DataBind. Bug latente o feature muerta.
- **AutoPostBack en `txtcodigo`** (parametros) sin handler asociado - genera postbacks vacios.
- **Marcador de superadmin** (`flag_super`) declarado pero no usado. No hay control de permisos de "quien puede crear tenants" en este modulo mas alla del acceso al menu 000072.
- **Encoding**: los archivos originales usan `A` con tilde y `n` con tilde (ver `ListaAno`/`ListaPeriodo`) - la reconstruccion debe respetar UTF-8 con BOM.
- **`ListaReglas` llama `UpdatePaneldatos.Update()`** en vez del panel de reglas - side-effect visual.

## 11. Puntos de reconstruccion en Claude Design

1. **Modelar el dominio "Tenant"**: entidad `Empresa` con subrecursos `Modulos`, `Actividades`, `Parametros`, `Integraciones`, `Usuarios`, `Reglas`, `Formularios`. Cada subrecurso es un tab o un card colapsable.
2. **Layout tabs**: dos tabs de nivel superior (`Editor` y `Listado`). El Editor es la vista maestro-detalle donde arriba va el selector de empresa (`cmbcodigo`) + botones Guardar/Limpiar/Eliminar; abajo el stack de cards.
3. **Selector de empresa reactivo**: al cambiar el codigo debe recargar 8 secciones. En Claude Design usar loaders paralelos y estados de skeleton.
4. **Cards colapsables** con chevron animado, iconos codigo-color (azul/verde/naranja/purpura/gris) siguiendo la paleta `--tronox-*` ya definida. Reutilizar la clase `tronox-card` como referencia visual.
5. **Grids con seleccion masiva**: header con checkbox "Marcar todos", filas con checkbox individual, boton Eliminar procesa marcados. Patron repetido 6 veces (Modulos, Actividades, Usuarios, Parametros, Integraciones, Cuestionarios).
6. **Motor de reglas**: la seccion "Cargar datos externos en formularios" merece un submodulo dedicado con wizard Grupo -> Regla -> Tarea -> Parametros dinamicos (form generado desde XML) -> Guardar/Ejecutar. Reemplazar `ctrFormDinamico` por un JSON-schema-driven form.
7. **Wizard de aprovisionamiento**: en vez de acordeones sueltos, considerar un wizard "Crear empresa nueva" que enlaza en secuencia: (1) datos basicos, (2) copiar modulos de empresa base, (3) copiar tablas de datos, (4) copiar formularios, (5) agregar usuarios, (6) configurar integraciones.
8. **Integraciones como plugins**: cada `ctrl*` (Slack, WhatsApp, Sigma, Mail, SendGrid, Storage, AWS, GoogleConsole, VisionIA, EvolutionAPI, APIManager) es un plugin con schema propio; renderizar como card con logo + form + boton "Probar conexion".
9. **Auditoria y transacciones**: envolver operaciones multi-tabla (crear tenant, copiar modulos, copiar formulario) en transacciones DB y registrar audit log.
10. **Confirmaciones destructivas**: los borrados actuales son inmediatos. En la reconstruccion pedir confirmacion (modal + texto de codigo) para eliminar SUCURSAL, y borrar en cascada todos los subrecursos.
11. **Filtro y paginacion server-side** en TAB 2 (hoy carga todas las empresas de una vez).
12. **Parametrizar TODOS los queries** al reconstruir (inyeccion SQL es sistemica).
