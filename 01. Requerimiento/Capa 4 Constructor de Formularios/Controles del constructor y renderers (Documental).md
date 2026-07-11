---
capa: "Capa 4 - Constructor de Formularios"
modulo: Documental
tipo: Ingenieria inversa
origen: ASP.NET WebForms (VB.NET) - UserControls .ascx
destino: .NET 10 / Blazor (DynamicFormRenderer + FormBuilder + componentes por tipo de campo)
ruta_origen: Bootstrap/Formularios/Modulos/Documental/Controles/
---

# Controles del constructor y renderers (Documental)

> Doble encuadre permanente. **ORIGEN** = la carpeta `Documental/Controles/` (38 archivos, UserControls WebForms VB.NET, namespace `GestionMovil`). **DESTINO** = componentes Blazor de .NET 10: un `FormBuilder` (disenador), un `DynamicFormRenderer` (runtime polimorfico) y **un componente por `ControlType`**. Cada seccion contrasta como esta hecho hoy y como deberia quedar.

Relacionado: [[00 - Visión Formularios]] - [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]] - [[Constructor - Patron EAV y motor visual]] - [[Esquema completo - Tablas y tipos de control]] - [[Visión y entorno]]

---

## 1. Panorama: constructor (disenar) vs renderer (responder)

El modulo Documental separa dos mundos que comparten el mismo modelo EAV (ver [[Constructor - Patron EAV y motor visual]]):

- **Constructor / disenador** = `ctrFormCreator.ascx`. Es la herramienta de autor: define contenedores, columnas, campos (preguntas), tipo de cada campo, orden, obligatoriedad, opciones de lista, propiedades, reglas, segmentos e imagenes. Persiste todo en tablas de definicion (`ENCUESTAS_MOV`, `ENCUESTAS_MOV_PREGUNTAS`, `FORX_DATA`, `CONTROL_REGLAS*`).
- **Renderer / runtime** = `crtCargaEncuestaII.ascx` (version vigente) y `crtCargaEncuesta.ascx` (version anterior). Leen la definicion y **dibujan el formulario para ser respondido**, capturan respuestas, validan obligatorios, aplican logica condicional (reglas) y guardan.

El resto de los 38 archivos son o bien **sub-controles auxiliares** que el constructor incrusta (panel de propiedades, editor de reglas, editor de campos, modulos, flujo) o bien **controles de tipo de campo** que el renderer instancia segun `TIPO_RESPUESTA` (firma, audio, imagen, GPS, texto rico, grid, tabla, captcha, filtro).

Clave arquitectonica: en ORIGEN el renderer nuevo NO tiene un `switch` gigante en su code-behind; delega el "que control mostrar" al metodo `cl_gestion_formularios.AdornoDetalle(...)` (capa `Funciones`/`MotherData`) que recorre cada item del `ListView` y activa el sub-control adecuado poniendo `Visible=true`. En DESTINO esto se vuelve **despacho polimorfico**: el `DynamicFormRenderer` itera el modelo y por cada campo renderiza `<DynamicField Model="campo" />`, que a su vez hace `switch` sobre `ControlType` y emite `<TextField>`, `<SignatureField>`, etc.

---

## 2. ctrFormCreator (el disenador)

`ctrFormCreator.ascx` (83 KB markup, 58 KB code-behind, 40 KB designer) es el componente mas grande de la carpeta. Estructura:

**Nivel 0 - Listado de formularios.** Un `GridView GridCuestionarios` lista los cuestionarios existentes (CODIGO, TITULO, DESCRIPCION); el `LinkButton lnkSeleccionarFormulario` con `CommandName="SeleccionarFormulario"` abre uno para editarlo (`GridCuestionarios_RowCommand`).

**Nivel 1 - Modal editor (`PanelPregunta`, modal-lg hasta 1800px).** Es el editor visual propiamente dicho, dividido en tres columnas:

- **Barra de herramientas superior (`fe-toolbar`)**: navegacion previo/siguiente entre registros de prueba (`LinkButtonNavegaratras` / `LinkButtonNavegarAdelante` / `txtubicareg`), borrar datos de prueba, guardar datos, **importar desde Excel** (`LinkButton17` -> `gen_ctrExcelFormularios` + `ProcesarArchivoFormulario` / `GenerarCampoDesdeCaption`), nuevo registro, eliminar, compartir (`Linkcompartir`), impresion, iframe.
- **Columna izquierda de configuracion (`fe-config-panel`, tabs `fe-tabs`)** con dos pestanas:
  - **Tab "Campos" (`#tab-x1`)**: define un campo. Controles: `cmbcolumna` (ancho/columna, incluye valores especiales `espacio` y `linea`), `cmbtabla` (contenedor/tabla destino, con pin `CheckBox1`), `txtCaption` (etiqueta visible, `AutoPostBack` -> `txtCaption_TextChanged` que autogenera el nombre de campo), `txtPregunta` (nombre interno del campo), `cmbTipoResp` (**el selector de tipo de control**), `txtOrden`, `cmbObligatorio`, `txtRespuestas` (opciones de lista separadas por `;`). Botones guardar/nuevo/eliminar pregunta. Acordeones anidados: *Opciones respuestas* (`txtAyuda`, `txtopcionotra` opcion "otra", `txtCorrectas` respuestas correctas para modo evaluacion), *Propiedades* (`ctrPropertyForm`), *Reglas* (`ctrReglas1`), *Imagenes* (`ctrImagenesForm`).
  - **Tab "Formulario" (`#tab-x2`)**: propiedades globales del formulario via un segundo `ctrPropertyForm` (`ctrPropertyForm_allform`) + boton guardar propiedades.
- **Columna central**: la **vista previa en vivo** = una instancia embebida de `crtCargaEncuestaII` (el mismo renderer runtime), de modo que el autor ve el formulario dibujado tal como lo vera el usuario final (WYSIWYG por reutilizacion del renderer).
- **Columna derecha (`col-md-2`)**: *Segmentar* (`TreeViewSegmentos` con checkboxes: agrupa campos en segmentos reutilizables; menu con Guardar/Incluir/Incluir todo/Retirar/Visualizar/Eliminar segmento) y *Contenedores* (`cmbtipoComponente` + nombre/orden: crea contenedores/tablas hijas).

**Persistencia.** El metodo central es `AgregaPregunta()`, que valida (salvo `espacio`/`linea`) y llama a `cl_FormCreator.GuardarPregunta(BaseSistema, Empresa, CodigoFormulario, Pregunta, nombre, caption, tipo, ayuda, respuestas, correctas, orden, obligatorio, opcionOtra, columna, contenedor)` -> tabla `ENCUESTAS_MOV_PREGUNTAS`. Despues: `ctrPropertyForm.GuardarFormulario()`, marca campos con formula (`cl_gestion_calculos.CamposConformulas()`), detecta si el contenedor es tabla (`cl_FormCreator.ObtenerTablaTipo`), carga propiedades del control (`CargarPropiedadesControl`), refresca reglas (`ctrReglas1.MostrarReglasCargadas`) y redibuja la vista previa (`RaiseEvent SolicitarRedibujo`, `CargarFormulario`). Otros metodos relevantes: `CargarReg` (carga un campo a los inputs para editar), `ListaTablas`/`ListaColumnas`/`ListaContenedores`, `AgregarSegmento`/`AgregarObjetoalSegmento`, `AgregarTabla`/`EliminarTabla` (con `TreeViewNiveles`), `TiposSolicitud`, `InicializarEditor`.

> **DESTINO**: `FormBuilder.razor` = un componente de autor con panel de propiedades reactivo (bindeo bidireccional), lista drag-and-drop de campos, y **preview reutilizando el mismo `DynamicFormRenderer`** (misma estrategia WYSIWYG, ahora sin doble `UpdatePanel`). `GuardarPregunta` se vuelve un `IFormDefinitionService.SaveFieldAsync(FieldDefinitionDto)`. Los `TreeView` de segmentos/contenedores -> componentes de arbol Blazor o `MudTreeView`.

---

## 3. crtCargaEncuestaII (el renderer runtime)

`crtCargaEncuestaII.ascx` (19 KB markup, 60 KB code-behind) es el corazon del runtime. Patron de renderizado:

**Un `ListView ListViewPropiedades`** cuyo `ItemTemplate` contiene **todos los tipos de control posibles pre-declarados y ocultos** (`Visible="false"`): `TextBoxRespuesta` (+ `AutoCompleteExtender` contra `fm_autocompletado.asmx`), `DropDownListRespuesta`, `CheckBoxListRespuestas`, `RadioButtonListRespuestas`, `ctrlTextGps`, un `LinkButton` (boton dinamico), `LiteralHtml`, y paneles que envuelven los sub-controles pesados: `panelHtml`->`ctrTextEditor`, `panelImagen`->`ctrImagenesForm`, `panelFirma`->`ctrSignature`, `panelAudio`->`ctrAudio`, `panelGridDetalle`->`ctrGridDetalle`, y aparte `panelRespuestaTabla`->`ctrCargaTablas`. Cada item lleva envoltura HTML de contenedor via `Eval("CONTAINER_HTML_open")` / `CONTAINER_HTML_closed` (layout por contenedor/columna).

**El despacho.** `CargarModelo()` arma el `DataSource`: obtiene los campos con `cl_gestion_formularios.SqlCamposNew(...)`, los convierte con `cl_base_formulario.ConvertirDatasetALista(mdata)` y hace `DataBind`. En `ListViewPropiedades_ItemDataBound` se invoca `mcl_gestion_formulario.AdornoDetalle(e, 0, ListViewPropiedades, ...)`: **ahi vive la logica que, segun `TIPO_RESPUESTA`, pone visible el control correcto y lo puebla** (opciones de lista, valor guardado, formato). Si el campo es una tabla hija, `AdornoDetalle` devuelve `proceso_tabla.Procesar=true` y el renderer configura e inicializa `ctrCargaTablas` (`PrepararTablas` / `PrepararRegistro`) enlazando su evento `GuardarFormulario`.

**Propiedades del formulario.** `PrepararPropiedades()` recorre `FORX_DATA` y mapea propiedades a estado: `LLAVE_TABLA_MAESTRA`, `CLAVE_PRIMARIA`, `ORDEN_TABLA_MAESTRA`, `LISTA_TABLA_MAESTRA`, y bajo `CODIGO='FORM.FORM'` las globales `AUTOGUARDADO`, `BOTONES`, `EDITAR_EXTERNO`, `FORM_MOBIL`, `DATOS_UNICOS_POR_USUARIO`, `OCULTAR_BOTON_GUARDADO`, `FORM_CARPETA_VISOR`. Genera tokens (`Funciones.Token`) para claves maestras/consecutivos.

**Claves y eventos.** `LlamadoObjetoclavePrincipal()` engancha handlers dinamicos: si hay clave maestra por combo -> `CargarllavedeCOMBO`; por textbox -> `CargarllavedeTEXBOX`; y para cada item conecta los eventos de los sub-controles: `ctrSignature.CapturarFirma`->`BloquearDatosFirmados`, `ctrSignature.IniciarFirma`->`ValidaPreviofirma`, `ctrCargaTablas.GuardarFormulario`->`GuardarFormularioTabla`, `ctrAudio.TranscripcionLista`->`CargarModelo`.

**Validacion y guardado.** `Validar()` consulta `ENCUESTA_RESP` por campos `REQUERIDA='SI'` sin respuesta y arma el mensaje de obligatorios. `GuardarFormulario(Validar, ValidarVacio)` persiste; `EliminarRegistro`, `NuevoRegistro`, `CopiarDatosFormulario`, `LimpiarModelo` completan el ciclo. `Linkcompartir` / `LinkImpresion` generan enlaces publicos/PDF.

**Reglas / logica condicional.** `PrepararFormulas()` y `EjecutarModal(...)` disparan reglas (ver seccion 4). `CargarFormularioEnModal(Place, mPage, rutaControl, JsonParam)` carga sub-formularios dentro del modal `PanelVisor` (composicion recursiva de formularios).

**Version anterior - `crtCargaEncuesta.ascx`** (7 KB / 18 KB): mismo proposito pero con **`Select Case man_tiporespuesta` explicito** en el code-behind (`MostrarEncuesta`): casos `"Lista RoundBoton"` (radio), `"Lista"`/`"Multiples Opciones"` (checkbox/multi), `"Si/NO"`, `"Abierto"` (texto), `"Fecha"`, `"Titulo"`. Es el ancestro directo del despacho: util para leer los nombres canonicos de tipo. `crtCargaEncuestaII` extrajo ese switch a `AdornoDetalle` y sumo los tipos ricos (firma, audio, imagen, GPS, HTML, grid, tabla, boton).

> **DESTINO**: `DynamicFormRenderer.razor` recibe un `FormModel` (lista de `FieldModel`). En vez de declarar todos los controles ocultos, hace **render condicional real**: `@foreach (var f in Model.Fields) { <DynamicField Field="f" @bind-Value="f.Value" /> }`. `DynamicField` contiene el `switch (f.ControlType)`. Sin `UpdatePanel`, la reactividad la da Blazor. `AdornoDetalle` desaparece: su logica de "poblar opciones y valor" pasa a la inicializacion de cada componente tipado. Validacion via `EditForm` + `DataAnnotations`/`FluentValidation`. Guardado via servicio async. El modal de sub-formularios -> `<DynamicFormRenderer>` anidado.

---

## 4. Editor de reglas (ctrReglas / ctrlGestorReglas)

Dos controles casi gemelos implementan el editor de **logica condicional / reglas** que se adjuntan a un campo o al formulario:

- **`ctrReglas.ascx`**: panel colapsable "Reglas" cableado con eventos (`OnClick`, `OnTextChanged`). Selectores en cascada: `cmbgruporegla` (grupo) -> `cmbreglas` (regla) -> `cmbreglastarea` (tarea). Un `ctrFormDinamico` embebido renderiza los **parametros** de la regla seleccionada (formulario dinamico de parametros). Botones Guardar y "Ejecutar regla". `GridDetalleReglas` lista las reglas ya asociadas (tarjetas con REGLA, DESCRIPCION, `txtordenejecutar` orden de ejecucion, editar/eliminar via `LinkButtonEditarRegla`/`LinkButtonEliminarRegla`). `ListViewError` muestra la bitacora de ejecucion.
- **`ctrlGestorReglas.ascx`**: variante "gestor" con el **mismo markup** pero code-behind minimo (1 KB, los botones sin handler cableado): metodos `ListaReglas`, `ListaReglasTarea`, `ListaReglasGrupos`, `MostrarPlantillaEjecucion`, `AsociarReglaObjeto`, `MostrarReglasCargadas`, `LimpiarSeleccionReglas`. Es la version reutilizable/desacoplada del mismo editor.

Motor: ambos se apoyan en `cl_gestion_reglas` (ver [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]) y en las tablas `CONTROL_REGLAS` / `CONTROL_REGLAS_R` / `FORX_DATA` (parametros con `CODIGO='EJECUTA_PARAM'`). El renderer las ejecuta en `PrepararFormulas` / `EjecutarModal`.

> **DESTINO**: un `RuleEditor.razor` (autor) + un `RuleEngine` de servicio. Los tres combos en cascada -> selects bindeados con carga async. `ctrFormDinamico` de parametros -> un mini `DynamicFormRenderer` reutilizado. La ejecucion de reglas en runtime -> servicio inyectado que corre sobre el modelo en memoria y actualiza visibilidad/valores (logica condicional declarativa: `Condition`, `Action`, `TargetField`).

---

## 5. Catalogo de controles de tipo de campo

Cada valor de `cmbTipoResp` / `TIPO_RESPUESTA` mapea a un control. Tabla ORIGEN -> DESTINO Blazor:

| Control ORIGEN (.ascx) | Tipo de campo que renderiza | Como funciona (resumen) | Componente DESTINO |
|---|---|---|---|
| (inline) `TextBoxRespuesta` | Texto abierto / numero / fecha | `TextBox` del ItemTemplate + `AutoCompleteExtender` (asmx) | `TextField` / `NumberField` / `DateField` |
| (inline) `DropDownListRespuesta` | Lista desplegable | opciones desde `RESPUESTAS` (`;`) | `SelectField` |
| (inline) `CheckBoxListRespuestas` | Multiples opciones | multi-seleccion | `CheckboxGroupField` |
| (inline) `RadioButtonListRespuestas` | Lista tipo radio ("Lista RoundBoton") | opcion unica | `RadioGroupField` |
| (inline) `LinkButton` | Boton dinamico / accion | dispara regla (`Linkbutton_Click1`) | `ActionButtonField` |
| `ctrTextEditor.ascx` | Texto rico HTML | editor **Quill 2.0.3** (CDN), sincroniza a `txtEditorContent` (hidden) | `RichTextField` (editor JS via interop) |
| `ctrSignature.ascx` | Firma manuscrita | `<canvas id="SignaturePad">` -> base64 en `HiddenFieldSignature` -> blob; eventos `CapturarFirma`/`IniciarFirma`; bloqueo tras firmar | `SignatureField` (canvas + JS interop) |
| `ctrImagenesForm.ascx` | Imagenes / archivos / foto | `TDFileUpload` + `ctrlDocumentosCentral` + `CtrlArchivoDirectorios`; guarda blob, sirve via `ShowImageGenerico.ashx`; anotaciones | `ImageUploadField` / `FileField` |
| `ctrAudio.ascx` | Audio | grabacion `MediaRecorder` (JS) -> base64 `hdnAudioData` -> `TDFileUpload`; grid de audios con flag transcripcion (`FLAG_TRANSCRIPCION`); evento `TranscripcionLista` | `AudioField` (MediaRecorder interop + transcripcion) |
| `ctrlTextGps.ascx` | Geolocalizacion | `txtubicaciongps` (readonly) + `hfLatitud`/`hfLongitud`; `AsyncPostBackTrigger` al cambiar latitud | `GeoField` (Geolocation API) |
| `ctrCatchap.ascx` | Captcha | **Google reCAPTCHA v2** (sitekey embebido); `hfCaptchaVerified`; bloquea submit sin marcar | `CaptchaField` (reCAPTCHA/Turnstile) |
| `ctrGridDetalle.ascx` | Grid de detalle (seleccion) | filtros dinamicos (`cmbfiltrocampo`+`txtfiltrovalor`, lista de filtros activos, orden), `GridFiltro` con columnas generadas en code-behind, checkbox masivo + seleccionar | `DetailGridField` / `LookupGridField` |
| `ctrCargaTablas.ascx` | Tabla hija (master-detail) | sub-grid editable dentro del form (nuevo/guardar/eliminar fila), `FiltroCondicion`/`ListaFiltros`, `VISTA_DESC`; evento `GuardarFormulario`/`RecargarScript` | `ChildTableField` (grid editable) |
| `ctrlField.ascx` | (autor) edicion masiva de campos | `CheckBoxList cmbcontroles` + filtro caption/nombre/orden; `GridPreguntas` edita caption/nombre/tipo/respuestas/orden en lote | herramienta del `FormBuilder` (no runtime) |
| `ctrModulos.ascx` | (autor) gestion de modulos | crea modulo, `GridDetalle` edita nombre/descripcion/consecutivo | pantalla admin (no runtime) |
| `ctrlEjemploFiltro.ascx` | Filtro/busqueda CIE-11 (medico) | busca diagnosticos en API CIE-11, `gvResultadosCIE11`, selecciona `EntityId` | `Cie11LookupField` (ejemplo de campo con API externa) |
| `ctrFiltrar.ascx` | (stub) | `.ascx` vacio, code-behind minimo (194 B) - placeholder/experimental | descartar en migracion |

---

## 6. Sub-controles auxiliares

- **`ctrPropertyForm.ascx`**: panel de propiedades generico key-value. `ListViewPropiedades` con `DataKeyNames="ETIQUETA,CAMPO"` renderiza por cada propiedad una `TextBox txttexto` o un `DropDownList cmbcombo` (segun `TIPO`), con hiddens `ETIQUETA/CAMPO/DATA/VALOR/TIPO`. `GuardarFormulario()` persiste. Se usa dos veces en el constructor: propiedades del campo y propiedades del formulario. -> **DESTINO**: `PropertyPanel.razor` con render dinamico por `PropertyType` y bindeo bidireccional.
- **`ctrFlujoActividades.ascx`**: asocia un campo/formulario a un **sub-proceso BPMN** (integra con el motor de flujos, mod `DOC_PROCESOS`). Props `SubFlujoSucursal`, `ProcesoSeleccionadoId`; metodos `AsociarSubProcesoObjeto`, `CargarFlujoGuardado`, `btnEliminarFlujoActividad_Click`, `ddlSubFlujoProceso_SelectedIndexChanged`. -> **DESTINO**: `WorkflowActivityBinder.razor` acoplado al motor BPMN de destino.
- **`ctrFormDinamico`** (registrado desde `General/Controles`, fuera de esta carpeta): renderiza los parametros de una regla; es un mini-renderer reutilizado por los editores de reglas.

---

## 7. Riesgos / deuda tecnica

1. **Despacho oculto en `AdornoDetalle`**: el mapeo `TIPO_RESPUESTA -> control` no esta en el renderer sino en `cl_gestion_formularios` (capa de datos). Migrar exige extraer y documentar ese switch (crtCargaEncuesta viejo sirve de referencia de los nombres canonicos).
2. **Todos los controles pre-declarados y ocultos** en el ItemTemplate: cada fila instancia firma+audio+imagen+grid+tabla aunque solo use uno. Costo de arbol de controles y de ViewState alto. En Blazor esto se elimina con render condicional.
3. **Dependencias externas por CDN** dentro de controles: Quill (`ctrTextEditor`), reCAPTCHA (`ctrCatchap`) cargan `<script src>` remotos; el `ctrTextEditor` incluso emite `<html><head><body>` completos dentro de un `.ascx` (anti-patron). CSP/offline lo rompen.
4. **Sitekey de reCAPTCHA y rutas de servicios (.asmx/.ashx) hardcodeadas** en el markup.
5. **Duplicacion `ctrReglas` vs `ctrlGestorReglas`** (markup identico, code-behind divergente) y **`crtCargaEncuesta` vs `crtCargaEncuestaII`** (dos renderers coexistiendo). Migrar a uno solo.
6. **`ctrFiltrar` es un stub vacio**; `ctrlEjemploFiltro` es un ejemplo especifico (CIE-11) mezclado con controles genericos: separar dominio medico de la infraestructura de formularios.
7. **Doble/triple `UpdatePanel` anidado** por todos lados: fuente de bugs de refresco parcial; desaparece en Blazor.
8. Acoplamiento del renderer a `Session("ControlFormulario")` y a tokens generados en linea.

---

## 8. Destino .NET 10 (renderer polimorfico Blazor)

Diseno objetivo (ver tambien [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]):

- **`DynamicFormRenderer.razor`** (reemplaza `crtCargaEncuestaII`): recibe `FormModel`, itera `Fields`, y por cada uno emite `<DynamicField>`. Envuelto en `EditForm` con validacion declarativa. Reutilizable como preview en el builder y como runtime.
- **`DynamicField.razor`**: unico punto con `switch (field.ControlType)` que resuelve **un componente por tipo** (`TextField`, `SelectField`, `RadioGroupField`, `CheckboxGroupField`, `SignatureField`, `AudioField`, `ImageUploadField`, `GeoField`, `RichTextField`, `CaptchaField`, `DetailGridField`, `ChildTableField`, `ActionButtonField`). Cada componente implementa una interfaz comun `IFormField` (`Value`, `Validate()`, `Render`), con JS interop encapsulado (canvas de firma, MediaRecorder de audio, Geolocation, editor rico).
- **`FormBuilder.razor`** (reemplaza `ctrFormCreator`): edicion de campos con drag-and-drop, `PropertyPanel`, `RuleEditor`, arbol de segmentos/contenedores, e importacion Excel; preview con el mismo `DynamicFormRenderer`.
- **`RuleEngine`** de servicio: ejecuta la logica condicional sobre el modelo en memoria (visibilidad, calculo, disparo de acciones), reemplazando `PrepararFormulas`/`AdornoDetalle`/`cl_gestion_reglas`.
- **Servicios**: `IFormDefinitionService` (CRUD de definicion, reemplaza `cl_FormCreator`), `IFormDataService` (respuestas, reemplaza el `GuardarFormulario` del renderer), `IFileStorageService` (blobs, reemplaza `ShowImageGenerico.ashx` + `TDFileUpload`).
- Persistencia: mantener modelo EAV (`ENCUESTAS_MOV*`, `FORX_DATA`) tras una fachada, o normalizar a `FormDefinition/FieldDefinition/FieldProperty/Rule` (ver [[Esquema completo - Tablas y tipos de control]]).

---

## 9. Enlaces

- [[00 - Visión Formularios]]
- [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]
- [[Constructor - Patron EAV y motor visual]]
- [[Esquema completo - Tablas y tipos de control]]
- [[Visión y entorno]]
