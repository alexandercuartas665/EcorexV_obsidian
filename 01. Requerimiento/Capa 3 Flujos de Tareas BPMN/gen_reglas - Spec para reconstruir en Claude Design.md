---
tipo: spec-reconstruccion
modulo: "000802"
pagina_origen: Bootstrap\Formularios\Modulos\General\gen_reglas.aspx
proposito: Spec autonomo para reconstruir el modulo Reglas en Claude Design / artifact mode / cualquier framework moderno
relacionado: [[Reglas - Motor y discrepancia]] · [[Reglas - Catalogo real y verbos Ensamblado]] · [[Reglas - Quien invoca realmente (cierre)]]
---

# gen_reglas.aspx - Spec completo para reconstruir el modulo Reglas

> Ficha autocontenida con todo lo necesario para rebuiilder el modulo de **Reglas (000802)** desde cero en cualquier stack moderno (HTML/JS, React, Blazor, etc.) o como prompt para Claude Design / Claude Artifact.
>
> Toda la informacion viene de la lectura directa de los archivos `gen_reglas.aspx` + `.aspx.vb` + `.designer.vb` + el esquema SQL real de `db3dev`.

---

## 1. Identidad del modulo

| Campo | Valor |
|---|---|
| Codigo modulo | `000802` |
| Path origen | `Bootstrap\Formularios\Modulos\General\gen_reglas.aspx` |
| Class | `GestionMovil.gen_reglas` (codebehind `gen_reglas.aspx.vb`) |
| Master Page | `MasterFormsII.Master` |
| Tabla principal | `CONTROL_REGLAS` (cabecera) |
| Tabla detalle | `CONTROL_REGLAS_R` (verbos), `CONTROL_REGLAS_F` (formularios asociados), `CONTROL_REGLAS_H` (historial), `PROPIEDADES_REGLAS` |
| Consecutivo | `REG1` |
| Alias BD | `[dbx.GENE]` |
| Topbar title | "REGLAS" |
| Ubicacion (breadcrumb) | "Home / GENERAL / REGLAS" |

---

## 2. Proposito del modulo (en una frase)

Administrador (CRUD) de **documentos de reglas** y sus **verbos** (acciones ejecutables). Cada documento agrupa N verbos; cada verbo tiene un PARAM_XML que define como se ejecuta (modo mDATA / Execute / Ensamblado) + un historial de ejecucion + formularios asociados.

---

## 3. Mapa visual de la pagina (layout ASCII)

```
+================================================================+
| MASTER TOPBAR  [GUARDAR oculto] [DELETE oculto] [LIMPIAR oculto]|  <- masterTopBar: estos 3 botones se OCULTAN al cargar
+================================================================+
| Home / GENERAL / REGLAS                                        |
+================================================================+
|                                                                |
|  [tabs principales]                                            |
|  +--------------------------+ +-------------+                  |
|  | Documento de configuracion | |  Historial  |                |
|  +==========================+ +-------------+                  |
|                                                                |
|  +------------------------------------------------------+      |
|  | Tab 1: Documento de configuracion (activo)            |      |
|  |                                                       |      |
|  |  [Estado v]  [Codigo v]  [Nombre.....] [Grupo......]  |      |  <- form-row 1
|  |   col-md-2    col-md-3    col-md-3      col-md-4      |      |
|  |                                                       |      |
|  |  [Observaciones                                ]      |      |  <- textarea col-md-12
|  |  [                                              ]      |      |     4 filas
|  |                                                       |      |
|  |  > Configuraciones (acordeon, colapsado por default)  |      |
|  |    +-----------------------------------------------+  |      |
|  |    | [Agregar parametros] [Inactivar] [Actualizar] |  |      |
|  |    | -------------------------------------------- |  |      |
|  |    | Tabla GridParametros (CONTROL_REGLAS_F)      |  |      |
|  |    | [ ] | Nombre | Descripcion (textarea ro) | [edit] |    |      |
|  |    +-----------------------------------------------+  |      |
|  |                                                       |      |
|  |  > Tareas (acordeon, colapsado por default)           |      |
|  |    +-----------------------------------------------+  |      |
|  |    | [Agregar nuevo regla] [Inactivar] [Actualizar]|  |      |
|  |    | -------------------------------------------- |  |      |
|  |    | Tabla Gridtareas (CONTROL_REGLAS_R)          |  |      |
|  |    | [ ] | Nombre | Descripcion | Orden | [edit]  |  |      |
|  |    |       (editable: orden -> postback inmediato)|  |      |
|  |    +-----------------------------------------------+  |      |
|  |                                                       |      |
|  +------------------------------------------------------+      |
|                                                                |
+================================================================+
| MODALES (ocultos hasta abrirse)                                |
|                                                                |
|  - PanelTarea          -> editar/crear un VERBO (regla)        |
|  - PanelConfiguracion  -> agregar formulario asociado          |
|  - PanelReglasUsuario  -> reglas usuario-usuario (BPMN)        |
|  - ModalDialog (uc1)   -> mensajes generales                   |
+================================================================+
```

---

## 4. Componentes y campos por seccion

### 4.1 Header (cabecera del documento) - Tab 1, parte superior

| Control | ID original | Tipo | Bind a campo SQL | Comportamiento |
|---|---|---|---|---|
| Estado | `cmbestado` | DropDownList AutoPostBack | `CONTROL_REGLAS.ESTADO` | Opciones FIJAS: `""`, `Activo`, `Desarrollo`, `Inactivo` |
| Codigo | `cmbdocumento` | DropDownList AutoPostBack | `CONTROL_REGLAS.DOCUMENTO` | Carga todos los documentos. Cambio = `CargarDocumento()` |
| Nombre | `txtnombre` | TextBox | `CONTROL_REGLAS.NOMBRE` | varchar(100) |
| Grupo | `txtgrupo` | TextBox | `CONTROL_REGLAS.GRUPO` | varchar(100) - tipo "DOCUMENTALES", "FORMULARIOS", etc. |
| Observaciones | `txtdescripcion` | TextBox MultiLine 4 filas | `CONTROL_REGLAS.OBSERVACION` | ntext, placeholder "Descripción de manejo del la información" |

### 4.2 Acordeon 1: Configuraciones (CONTROL_REGLAS_F)

> Liga formularios al documento de regla. Se renderiza pero queda **colapsado** por default.

**Toolbar (3 botones al lado derecho):**
| Boton | Color | Click handler | Accion |
|---|---|---|---|
| Agregar parametros | `btn-primary` | `Linkbuttonaddconfiguracion_Click` | Abre `PanelConfiguracion` (modal) en modo nuevo |
| Inactivar | `btn-danger` | `LinkbuttonInactivaConfiguracion_Click` | Sobre las filas marcadas, hace `UPDATE CONTROL_REGLAS_F SET ESTADO='Inactivo'` |
| Actualizar | `btn-default` | `LinkbuttonActualizaConfiguracion_Click` | Recarga el grid |

**Grid `GridParametros`** (table-striped, paginacion 30 por pagina):
| Columna | Display | Tipo |
|---|---|---|
| [ ] | checkbox seleccion | con "select all" en header |
| Nombre | `NOMBRE` de `CONTROL_REGLAS_F` | text |
| Descripcion | `DESCRIPCION` | textarea readonly 4 filas |
| Editar | icono `fa-edit` | botton, abre `PanelConfiguracion` con la fila |

### 4.3 Acordeon 2: Tareas (CONTROL_REGLAS_R) -- el principal

> Es el listado de **verbos** del documento. Tambien colapsado por default.

**Toolbar (3 botones):**
| Boton | Click handler | Accion |
|---|---|---|
| "Agregar nuevo regla de trabajo" (btn-primary) | `Linkbutton1_Click` | Abre `PanelTarea` en modo nuevo |
| "Inactivar" (btn-danger) | sin handler | TODO o stub |
| "Actualizar" (btn-default) | `Linkbutton8_Click` | Recarga grid via `ConsultarDetalle()` |

**Grid `Gridtareas`:**
| Columna | Display | Notas |
|---|---|---|
| [ ] | checkbox | "select all" |
| Nombre | `CONTROL_REGLAS_R.NOMBRE` | El "verbo" (EXPANDIR_BARRAS, PASAR_CAMPOS, etc.) |
| Descripcion | `CONTROL_REGLAS_R.DESCRIPCION` | textarea readonly 2 filas |
| Orden | `CONTROL_REGLAS_R.ORDEN` | TextBox AutoPostBack: al cambiar -> `UPDATE ... SET ORDEN=...` |
| Editar | icono `fa-edit` | abre `PanelTarea` con la fila |

### 4.4 Tab 2: Historial

Tablon de auditoria. Por simplicidad muestra `GridHistorico` con:

| Columna | Bind |
|---|---|
| Documento | `CONTROL_REGLAS.DOCUMENTO` (historico de cabeceras) |
| Fecha | `FECHA` |
| Observaciones | `OBSERVACION` |
| Estado | `ESTADO` |

Texto descriptivo a la izquierda: "Configuraciones - Proceso de configuracion del sistema, consulta historial de configuraciones del sistema."

---

## 5. Modal `PanelTarea` (editar/crear verbo) - el componente mas rico

Abre como `modal-lg` con `max-width: 1200px`. Dentro tiene **tabs propios**:

```
+============================================================+
| Modal Header [X]                                           |
+============================================================+
| [Configuracion] [Historial]                                |
+============================================================+
| Tab "Configuracion" (activo)                               |
|                                                            |
| +----------------+ +-----------------+ +-------------+    |
| | Nombre         | | Tipo ejecucion v | | Orden ejec |    |
| | (col-md-4)     | | (col-md-4)       | | (col-md-3) |    |
| +----------------+ +-----------------+ +-------------+    |
|                                                            |
| [Estado de la regla v] (col-md-4)                          |
|                                                            |
| [Descripcion de la funcion................] (col-md-12)   |
| [textarea 3 filas]                                         |
|                                                            |
| [Configuracion de la regla.................] (col-md-12)  |  <- *** EL CAMPO MAS IMPORTANTE
| [textarea 16 filas - aqui va el PARAM_XML]                 |     Placeholder: "INGRESA CONFIGURACION DE LA REGLA"
|                                                            |
| [Ejecutar Script............................] (col-md-12) |
| [textarea 3 filas]                                         |
|                                                            |
| Botones (alineados a la derecha):                          |
| [Crear o actualizar regla] [Limpiar] [Eliminar] [Ejecutar] |
|                                                            |
+============================================================+
| Lado derecho del modal (col-md-5):                         |
| - Vista previa de plantilla ejecucion (ctrFormDinamico)    |
| - Renderiza un formulario dinamico interpretado del XML    |
+============================================================+
| Tab "Historial":                                           |
| GridHistorial: tabla de CONTROL_REGLAS_H del verbo actual  |
| Columnas: REG, USUARIO, FECHA_REG, REGLA, CODIGO,         |
|           LLAMADO, REGISTROS, MSG_ERROR, DURACION         |
+============================================================+
```

### Campos del modal

| Control | ID original | Bind | Opciones / Notas |
|---|---|---|---|
| Nombre | `txtnombreregla` | `CONTROL_REGLAS_R.NOMBRE` | varchar(100) |
| Tipo ejecucion | `cmbtipoejecucion` | `CONTROL_REGLAS_R.TIPO` | `""`, `mDATA`, `Execute`, `Ensamblado` |
| Orden ejecucion | `txtordenejecucion` | `CONTROL_REGLAS_R.ORDEN` | TextBox type=Number, int |
| Estado | `cmbestadotarearegla` | `CONTROL_REGLAS_R.ESTADO` | `""`, `Activo`, `Desarrollo`, `Inactivo` |
| Descripcion funcion | `txtfuncion` | `CONTROL_REGLAS_R.DESCRIPCION` | ntext, 3 filas |
| Configuracion regla | `txtconfiguracionregla` | `CONTROL_REGLAS_R.PARAM_XML` | ntext, **16 filas** -- el XML completo |
| Script | `txtprevioscript` | `CONTROL_REGLAS_R.SCRIPT` | ntext, 3 filas |

### Botones del modal

| Boton | ID | Handler | Accion |
|---|---|---|---|
| Crear o actualizar regla | `Linkbutton4` | `Linkbutton4_Click` -> `AgregarDetalles()` | INSERT o UPDATE en `CONTROL_REGLAS_R` |
| Limpiar | `Linkbutton7` (entre otros) | `Linkbutton7_Click` -> `LimpiarModalReglas(1)` | Resetea campos del modal |
| Eliminar | (variantes) | `LinkbuttonDeleruleUser_Click` -> `EliminarReglas()` | DELETE fila actual |
| **Ejecutar regla (prueba)** | `Linkbutton6` | `Linkbutton6_Click` -> `EjecutarRegla()` | Invoca `cl_gestion_reglas` con el PARAM_XML actual |

### Comportamiento del modal "Vista previa"

Mientras se edita el `txtconfiguracionregla` (XML), el lado derecho renderiza via `ctrFormDinamico`:
```vb
ctrFormDinamico.ModeloXMLString = txtconfiguracionregla.Text
ctrFormDinamico.ModeloClave = "TEST_PARAM"
ctrFormDinamico.Referencia = Me.RegRegla
ctrFormDinamico.Token = Me.RegRegla
ctrFormDinamico.Tabla = topBar.Modulo & "_CAPITALES"
ctrFormDinamico.Sucursal = Session("Empresa")
ctrFormDinamico.IdRegla = Me.RegRegla
Call ctrFormDinamico.CargarModeloFromString()
```

→ es decir, el control toma el XML y construye un formulario en vivo a partir de los nodos `<CorXml NOMBRE="TEST_PARAM">...</CorXml>` (vease catalogo de PARAM_XML en [[Reglas - Catalogo real y verbos Ensamblado]]).

---

## 6. Modal `PanelConfiguracion` (asociar formulario)

Mas pequeno. Edita una fila de `CONTROL_REGLAS_F`.

| Campo | Bind |
|---|---|
| Formulario | `cmbformualario` -> `CONTROL_REGLAS_F.FORMULARIO` (dropdown de `ENCUESTAS_MOV.CODIGO`) |
| Nombre config | `txtnombreconfiguracion` -> `NOMBRE` |
| Descripcion | `txtdetalleconfiguracon` -> `DESCRIPCION` |
| Estado | `cmbestadoconfiguracion` -> `ESTADO` |

Si esta editando (no creando) el dropdown de Formulario se deshabilita (no se permite cambiar el formulario, solo los metadatos).

Botones: Crear/Actualizar, Eliminar, Limpiar.

Se renderiza tambien un `crtCargaEncuestaII` (renderer del formulario completo asociado) para que el usuario vea como se ve el formulario.

---

## 7. Datos reales en produccion (para el seed)

Estos 8 documentos son los que existen en la BD `db3dev`:

| DOCUMENTO | NOMBRE | GRUPO | ESTADO |
|---|---|---|---|
| `00001` | GENERAR MATRIZ DE BARRAS | DOCUMENTALES | Desarrollo |
| `00002` | LLENAR DATOS CON IA | INTELIGENCIA ARTIFICIAL | Activo |
| `00003` | GENERAR ACTIVIDADES | SISTEMA ACTIVIDADES | Desarrollo |
| `00005` | OPERACIONES DE FORMULARIOS | FORMULARIOS | Activo |
| `00006` | MIGRAR SIGO | SIGO | Desarrollo |
| `00007` | PROCESOS DE IMPORTACION BOTS | PROCESOS | Activo |
| `00008` | PLANTILLAS DE IMPRESION | IMPRESIONES | Desarrollo |
| `00009` | SISTEMA ECOREX | GESTION DE USUARIOS | Desarrollo |

Y 21 verbos asociados (ver tabla completa en [[Reglas - Catalogo real y verbos Ensamblado]] seccion 3).

Ejemplo de PARAM_XML real (verbo `EXPANDIR_BARRAS`):
```xml
<PagXml>
  <CorXml>
    <NOMBRE>ENSAMBLADO_DATA</NOMBRE>
    <PROCESO><![CDATA[Funciones.cl_doc_reglas_documental, Funciones]]></PROCESO>
    <EVENTO><![CDATA[doc_asignar_barras]]></EVENTO>
  </CorXml>
  <CorXml>
    <NOMBRE>TEST_PARAM</NOMBRE>
    <CAMPO>CAMPO_FORMULARIO</CAMPO>
    <OBLIGATORIO>NO</OBLIGATORIO>
    <DESCRIPCION>Variables fijas a cargar</DESCRIPCION>
    <ETIQUETA>Formulario destino:</ETIQUETA>
    <TIPO>TEXTO</TIPO>
    <ROW>col-md-12</ROW>
  </CorXml>
  <CorXml>
    <NOMBRE>RETURN_SMS</NOMBRE>
    <DATA>Se procesaron @@CANTIDAD@@ registros</DATA>
  </CorXml>
</PagXml>
```

---

## 8. Esquemas SQL exactos

### `CONTROL_REGLAS` (cabecera)
```
REG          int           PK
DOCUMENTO    varchar(25)   PK_LOGICA (codigo visible: "00001", "00002", ...)
SUCURSAL     varchar(5)
FECHA        datetime
FECHA_INI    datetime      vigencia desde
FECHA_FIN    datetime      vigencia hasta
OBSERVACION  ntext
USUARIO      varchar(25)
FECHA_NOV    datetime
ESTADO       varchar(20)   "Activo" | "Desarrollo" | "Inactivo"
NOMBRE       varchar(100)
GRUPO        varchar(100)
```

### `CONTROL_REGLAS_R` (verbos - detalle)
```
REG          int           PK
REGLA        varchar(25)   FK -> CONTROL_REGLAS.DOCUMENTO
NOMBRE       varchar(100)  el "verbo" (EXPANDIR_BARRAS, etc.)
SUCURSAL     varchar(5)
TIPO         varchar(20)   "mDATA" | "Execute" | "Ensamblado"
ORDEN        int           secuencia
PARAM_XML    ntext         el XML completo de configuracion
DESCRIPCION  ntext
ESTADO       varchar(50)   "Activo" | "Desarrollo" | "Inactivo"
SCRIPT       ntext         script JS pre-ejecucion (opcional)
```

### `CONTROL_REGLAS_F` (formularios asociados)
```
REG          int           PK
REGLA        varchar       FK
SUCURSAL     varchar
FORMULARIO   varchar       FK -> ENCUESTAS_MOV.CODIGO
NOMBRE       varchar
DESCRIPCION  ntext
ESTADO       varchar       "Activo" | "Inactivo"
```

### `CONTROL_REGLAS_H` (historial)
```
REG          int
USUARIO      varchar(25)
FECHA_REG    datetime
REGLA        varchar       FK
CODIGO       int           id del verbo ejecutado
LLAMADO      ntext         XML de invocacion
REGISTROS    int           filas afectadas
MSG_ERROR    ntext
DURACION     int           ms
```

---

## 9. Estados y flujo de datos

```
[Usuario abre la pagina]
        |
        v
[Page_Load]
   - topBar.Modulo = "000802"
   - topBar.Title = "REGLAS"
   - topBar.Ubicacion = "Home/GENERAL/REGLAS"
   - Carga combos:
       cmbestadoconfiguracion       <- ["", Activo, Desarrollo, Inactivo]
       cmbestado                     <- ["", Activo, Desarrollo, Inactivo]
       cmbtipoejecucion              <- ["", mDATA, Execute, Ensamblado]
       cmbestadotarearegla           <- ["", Activo, Desarrollo, Inactivo]
       cmbdocumento                  <- SELECT documentos de CONTROL_REGLAS
       cmbformualario                <- SELECT formularios de ENCUESTAS_MOV
        |
        v
[Usuario selecciona un documento en cmbdocumento]
   - Dispara CargarDocumento()
     - SELECT * FROM CONTROL_REGLAS WHERE DOCUMENTO = X
     - Setea: cmbestado, txtdescripcion, txtgrupo, txtnombre
   - Dispara ConsultarDetalle()       (cargar grid Tareas/verbos)
   - Dispara ConsultarDetalleCongf()  (cargar grid Configuraciones/formularios)
        |
        v
[Usuario click "Editar" en una fila de Gridtareas]
   - LinkButton13_Click guarda RegRegla = HiddenFieldRowregregla.Value
   - Llama CargarModalReglas() que SELECT del verbo
   - Setea controles del modal: cmbtipoejecucion, txtordenejecucion,
     txtconfiguracionregla, txtnombreregla, cmbestadotarearegla,
     txtfuncion, txtprevioscript
   - Llama MostrarPlantillaEjecucion() (renderiza preview XML)
   - Llama HistorialEjecucion() (carga GridHistorial)
   - Muestra modal PanelTarea via Bootstrap
```

---

## 10. Prompt directo para Claude Design / Artifact

> Copia-pega esto como prompt si quieres que Claude reconstruya el modulo:

```
Construye un modulo CRUD moderno llamado "Reglas" (codigo 000802) con tema claro
moderno (colores: fondo #f7f8fa, primary #2563eb, success #10b981, danger #ef4444).

LAYOUT:
- Header de pagina: titulo "Reglas" con subtitulo "Modulo 000802 - Tabla CONTROL_REGLAS"
- 2 tabs principales: "Documento de configuracion" y "Historial"

TAB "Documento de configuracion":
- Fila 1 con 4 campos:
  - Estado (dropdown): vacio, Activo, Desarrollo, Inactivo
  - Codigo (dropdown): lista de 8 documentos (ver datos abajo)
  - Nombre (input text)
  - Grupo (input text)
- Fila 2: Observaciones (textarea 4 filas, ancho completo)
- Acordeon 1 "Configuraciones" (colapsado por default):
  - 3 botones: [Agregar parametros (azul)] [Inactivar (rojo)] [Actualizar (gris)]
  - Tabla con columnas: [checkbox] Nombre, Descripcion (textarea readonly), Editar
- Acordeon 2 "Tareas" (colapsado por default):
  - 3 botones: [Agregar nuevo regla (azul)] [Inactivar (rojo)] [Actualizar (gris)]
  - Tabla con columnas: [checkbox] Nombre, Descripcion, Orden (editable!), Editar

TAB "Historial":
- Tabla simple: [checkbox] Documento, Fecha, Observaciones, Estado

MODAL "Editar regla" (se abre desde editar en tabla Tareas):
- Tamano grande (max 1200px)
- 2 tabs internos: "Configuracion" y "Historial"
- Tab "Configuracion" - layout 2 columnas (col-md-7 izq + col-md-5 der):
  - Izquierda:
    - Fila 1: Nombre (col-md-4) | Tipo ejecucion (col-md-4) | Orden (col-md-3)
    - Fila 2: Estado regla (col-md-4)
    - Fila 3: Descripcion funcion (col-md-12, textarea 3 filas)
    - Fila 4: Configuracion de la regla (col-md-12, textarea **16 filas**)
             placeholder: "INGRESA CONFIGURACION DE LA REGLA"
    - Fila 5: Ejecutar Script (col-md-12, textarea 3 filas)
    - Botones derecha: [Crear o actualizar (azul)] [Limpiar] [Eliminar (rojo)] [Ejecutar prueba (verde)]
  - Derecha: Preview en vivo del formulario que genera el PARAM_XML del campo
    "Configuracion de la regla" (parsear nodos <CorXml NOMBRE="TEST_PARAM">
    con CAMPO, ETIQUETA, TIPO, OBLIGATORIO, ROW)
- Tab "Historial":
  - Tabla con columnas: Usuario, Fecha, Llamado, Registros, Duracion(ms), MsgError

DROPDOWNS - opciones fijas:
- Estado: "", "Activo", "Desarrollo", "Inactivo"
- Tipo ejecucion: "", "mDATA", "Execute", "Ensamblado"

DATOS DEMO (los 8 documentos reales):
1. 00001 GENERAR MATRIZ DE BARRAS - DOCUMENTALES - Desarrollo
2. 00002 LLENAR DATOS CON IA - INTELIGENCIA ARTIFICIAL - Activo
3. 00003 GENERAR ACTIVIDADES - SISTEMA ACTIVIDADES - Desarrollo
4. 00005 OPERACIONES DE FORMULARIOS - FORMULARIOS - Activo
5. 00006 MIGRAR SIGO - SIGO - Desarrollo
6. 00007 PROCESOS DE IMPORTACION BOTS - PROCESOS - Activo
7. 00008 PLANTILLAS DE IMPRESION - IMPRESIONES - Desarrollo
8. 00009 SISTEMA ECOREX - GESTION DE USUARIOS - Desarrollo

VERBOS DEMO (al seleccionar documento 00005):
- PASAR_CAMPOS (Ensamblado, orden 1, Activo)
- BLOQUEAR_CAMPO_XCONDICION (Ensamblado, orden 1, Activo)
- IMPORTAR_FORMULARIO_DB (Ensamblado, orden 1, Activo)
- ASIGNAR_CONSECUTIVO (Ensamblado, orden 1, Activo)
- EJECUTAR CONSULTA SQL (Execute, orden 1, Activo)
- ABRIR_MODAL (Ensamblado, orden 5, Activo)

EJEMPLO PARAM_XML para editor (al cargar verbo PASAR_CAMPOS):
<PagXml>
  <CorXml>
    <NOMBRE>ENSAMBLADO_DATA</NOMBRE>
    <PROCESO>GestionMovil.cl_gestion_campo</PROCESO>
    <EVENTO>pasar_campos</EVENTO>
  </CorXml>
  <CorXml>
    <NOMBRE>TEST_PARAM</NOMBRE>
    <CAMPO>CAMPO_ORIGEN</CAMPO>
    <ETIQUETA>Campo origen:</ETIQUETA>
    <TIPO>TEXTO</TIPO>
    <ROW>col-md-6</ROW>
  </CorXml>
  <CorXml>
    <NOMBRE>TEST_PARAM</NOMBRE>
    <CAMPO>CAMPO_DESTINO</CAMPO>
    <ETIQUETA>Campo destino:</ETIQUETA>
    <TIPO>TEXTO</TIPO>
    <ROW>col-md-6</ROW>
  </CorXml>
</PagXml>

ESTILO:
- Cards con sombra suave, border-radius 12px
- Botones grandes en acciones primarias (btn-lg)
- Tipografia Inter/system, peso 500 para labels
- Sin scroll horizontal, responsive
- Icono "lapiz" (edit) en columnas de accion
- Toasts para confirmaciones (top-right)

STACK SUGERIDO: HTML + JS vanilla + Bootstrap 5 o Tailwind. Persistencia en
localStorage. Que sea un solo archivo HTML autocontenido.
```

---

## 11. Reglas de comportamiento criticas

1. **Los 3 botones del MasterTopBar (Guardar, Delete, Limpiar) se OCULTAN al cargar la pagina** (este modulo tiene su propia logica de guardado por seccion).
2. **AutoPostBack en `txtroworden` del grid Tareas**: al cambiar el orden, INSTANTANEAMENTE actualiza la BD (`UPDATE CONTROL_REGLAS_R SET ORDEN = ... WHERE REG = ...`).
3. **Acordeones cerrados por default** - el usuario debe expandir manualmente Configuraciones / Tareas.
4. **El editor de PARAM_XML es texto plano**, 16 filas, sin syntax highlighting (mejorable: usar Monaco editor o CodeMirror).
5. **Preview del XML en vivo**: parsea `<CorXml NOMBRE="TEST_PARAM">` y renderiza un formulario interpretado (campo, etiqueta, tipo, obligatorio, ROW de grid Bootstrap).
6. **Mensajes de exito/error** se inyectan en `LiteralGuardado`, `LiteralErrorsql`, `LiteralConfiguraciones` que son `<asp:Literal>` dentro de UpdatePanels. En el rebuild moderno -> usar toast / banner.
7. **Validacion**: el campo `cmbdocumento` es obligatorio para todas las operaciones; si esta vacio se muestra "Debe idicar el codigo" (sic, mantener typo si se busca fidelidad).
8. **Multi-tenant**: todo SQL filtra por `WHERE SUCURSAL = Session("Empresa")`.

---

## 12. Lo que **no** se reconstruye (funcionalidad fuera de scope)

- El boton "Ejecutar regla" depende de `cl_gestion_reglas` (clase .NET no portable). En el rebuild simular con mock que muestre "Ejecutando..." y luego un resultado tipo "Se procesaron N registros (XX ms)".
- El sistema de `ctrFormDinamico` que parsea XML es complejo (parser custom de `<CorXml>`). En el rebuild basta con un parser simple que renderice los campos `TEST_PARAM`.
- El acordeon "Reglas usuario-usuario" (`PanelReglasUsuario`) es opcional - es una vista de reglas en transiciones de flujo BPMN.
- El modal de "Buscar NIT" / "Buscar cliente" del HTML es legacy, no se usa en este modulo.

---

## 13. Checklist de cobertura para validar el rebuild

- [ ] Tabs principales (Documento + Historial) renderizan
- [ ] Cabecera con 4 campos (Estado, Codigo, Nombre, Grupo) + Observaciones
- [ ] Combo Estado tiene exactamente 4 opciones (incluyendo vacio)
- [ ] Combo Codigo carga 8 documentos demo
- [ ] Cambiar Codigo dispara carga de Nombre, Grupo, Observaciones
- [ ] Acordeon Configuraciones se abre/cierra
- [ ] Acordeon Tareas se abre/cierra
- [ ] Tabla Tareas carga verbos del documento seleccionado
- [ ] Editar orden de un verbo persiste el cambio
- [ ] Click "Editar" en verbo abre modal grande
- [ ] Modal tiene textarea de 16 filas para PARAM_XML
- [ ] Modal renderiza preview en vivo del XML
- [ ] Combo Tipo ejecucion tiene 4 opciones (incluyendo vacio)
- [ ] Boton "Crear o actualizar" guarda y cierra modal
- [ ] Boton "Ejecutar prueba" simula ejecucion (toast con resultado)
- [ ] Tab "Historial" muestra GridHistorico con datos
- [ ] Layout responsive en desktop y tablet
- [ ] Botones del MasterTopBar (Guardar/Delete/Limpiar) ocultos
- [ ] Validacion: sin Codigo seleccionado, mostrar error al intentar agregar
- [ ] Tema claro coherente con paleta especificada

---

## 14. Recursos relacionados en el vault

- [[Reglas - Motor y discrepancia]] - el motor de ejecucion y el bug del manejador legacy
- [[Reglas - Catalogo real y verbos Ensamblado]] - los 21 verbos reales con su PARAM_XML
- [[Reglas - Quien invoca realmente (cierre)]] - cierre del hilo: cl_gestion_reglas es el motor moderno
- [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]] - relacion con el constructor de formularios

---

*Spec generado leyendo: gen_reglas.aspx + gen_reglas.aspx.vb + gen_reglas.aspx.designer.vb + el esquema SQL real de db3dev en una sesion de documentacion.*
