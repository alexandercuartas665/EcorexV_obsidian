---
titulo: Clases del nucleo de formularios (Documental)
capa: Capa 4 - Constructor de Formularios
origen: Bootstrap/Formularios/Modulos/Documental/Clases
tipo: documentacion-tecnica
---

# Clases del nucleo del motor de formularios dinamicos (modulo Documental)

Doble encuadre en cada seccion: **ORIGEN** describe lo que hace hoy la clase en VB.NET (.NET Framework 4.8.1, ASP.NET WebForms); **DESTINO** describe como se reconstruye en la plataforma objetivo (.NET 10, servicios de aplicacion desacoplados de la UI). Ver contexto en [[00 - Visión Formularios]], [[Visión y entorno]] y [[Constructor - Patron EAV y motor visual]].

---

## 1. Que es este conjunto de clases y como se relaciona con las tablas

Este directorio (`Bootstrap\Formularios\Modulos\Documental\Clases\`) es el **backend de dominio** del motor de formularios dinamicos del modulo Documental (modulo `000131`, encuestas moviles). No son controles de UI: son clases de negocio que las paginas y controles ASCX (por ejemplo `crtCargaEncuestaII`, ver [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]) instancian para persistir respuestas, calcular campos derivados, ejecutar reglas, construir el arbol de componentes y generar el JavaScript de cliente.

Todas comparten el patron `tbrec` (instancia de `MotherData.AdmDatos`) y el alias `[dbx.GENE]` para portabilidad de entorno. El modelo de datos es **EAV** (Entity-Attribute-Value): la definicion del formulario vive en `ENCUESTAS_MOV_PREGUNTAS` (campos) y `ENCUESTAS_MOV_PREGUNTAS_T` (contenedores), las **propiedades** de cada campo viven en `FORX_DATA` bajo `TABLA='000131_ORIGEN'` (columnas TABLA/REFERENCIA/TOKEN/CODIGO/PROPIEDAD/VALOR), y las **respuestas** capturadas viven en `ENCUESTA_RESP`. Ver el detalle del esquema en [[Esquema completo - Tablas y tipos de control]].

Ejes de identidad recurrentes en casi todos los queries: `SUCURSAL` (tenant), `ENCUESTA`/`Formulario` (id del formulario), `CAMPANA`/`Referencia` (instancia de diligenciamiento), `REFERENCIAII` (sub-instancia), `REFERENCIA_LLAVE` (llave temporal por usuario), `TABLA`/`TABLA_REG` (fila cuando el campo esta dentro de una tabla repetible) y `PEDREG` (REG de la pregunta).

---

## 2. Clases una por una

### 2.1 cl_gestion_formularios (nucleo, 128 KB)

**ORIGEN.** Es la clase central de captura y renderizado. Expone ~30 propiedades de estado (LLAVE_TABLA_MAESTRA, CONSECUTIVO_ASIGNADO, Formulario, Referencia, RegSegmento, ModoVisor, FORM_AUTOGUARDADO, FORM_MOBIL, DATOS_UNICOS_POR_USUARIO, IdProceso/IdFlujoProceso, etc.) que actuan como contexto de ejecucion. Metodos publicos clave:

- **`ProcesarElemento(...)`**: persiste una respuesta individual. Recibe `jsonData` (deserializado a `Dictionary(Of String,String)`), y decide entre INSERT o UPDATE sobre `ENCUESTA_RESP` segun `count(*)` previo. Antes gestiona propiedades del campo (autocompletado) leyendo `FORX_DATA` y llamando `GestionarPropiedades`; si un campo tiene AUTOCOMPLETADO genera un segundo registro oculto (`FLAG_OCULTO=1`, patron "campo duo"). Guarda `RESPUESTA_USU` con escape de comillas.
- **`SqlCamposNew(...)`**: construye el gran SELECT que arma la definicion visible del formulario desde `ENCUESTAS_MOV_PREGUNTAS`, cruzando el tipo de contenedor (`ENCUESTAS_MOV_PREGUNTAS_T`), si el campo tiene formula (`ENCUESTAS_MOV_PREGUNTAS_F`, subconsultas CONTIENE_FORMULA/REG_FORMULA) y las propiedades EAV (carga `FORX_DATA` a una tabla temporal `@FORX_DATA` y la proyecta como XML `PROPIEDADES_CAMPO`). Aplica filtro por segmento via `ENCUESTAS_MOV_PREGUNTAS_CONTENEDOR_R`.
- **`AdornoDetalle(...)`**: orquestador de renderizado por fila del ListView; recibe y coordina `cl_gestion_campo`, `cl_gestion_script` y `cl_gestion_reglas` (inyeccion de dependencias por parametro).
- **`FormularioProcesarElemento(...)`** (dos sobrecargas, ListView y DataTable): recorre los controles del formulario, valida obligatorios y persiste en lote.
- **`Consecutivo` / `RecuperaConsecutivo`**: generan el numero de documento (usa `TIPDOC`, `GEN_PARAMETROS.CONSECUTIVO_FOR_VIRTUAL` y `FORX_DATA.CONSECUTIVO_TEST_TABLA_MAESTRA`).
- **`AgregarRegistrotabla`**: agrega una fila a una tabla repetible.
- **`ListaDinamico` / `GestionarDataset` / `FindPropiedades` / `GestionarPropiedades`**: resuelven combos dinamicos y propiedades desde tabla maestra y XML EAV.
- **`ObtenerSegmentoProceso`**: resuelve el segmento activo segun proceso/actividad BPMN.

**Tablas que toca:** `ENCUESTA_RESP` (INSERT/UPDATE/DELETE), `ENCUESTAS_MOV_PREGUNTAS`, `ENCUESTAS_MOV_PREGUNTAS_T`, `ENCUESTAS_MOV_PREGUNTAS_F`, `ENCUESTAS_MOV_PREGUNTAS_CONTENEDOR_R`, `FORX_DATA`, `GEN_PARAMETROS`, `TIPDOC`.

**DESTINO (.NET 10).** Se parte en dos servicios: **FormRenderService** (equivalente a SqlCamposNew + AdornoDetalle: arma el modelo de formulario para el cliente) y **AnswerService/ResponseService** (equivalente a ProcesarElemento/FormularioProcesarElemento: persiste respuestas). El estado en propiedades se reemplaza por un objeto `FormContext` inmutable (record) pasado por argumento. El SQL string-concatenado se reemplaza por consultas parametrizadas (Dapper/EF Core), y la generacion de consecutivos por un `ConsecutiveService` con transaccion/lock.

### 2.2 cl_gestion_campo (gestion de campos/preguntas, 24 KB)

**ORIGEN.** Traduce un campo de la definicion a un control concreto de UI. Propiedades de contexto (Sucursal, Formulario, Usuario, Referencia, ReferenciaII, Produccion, FormCerrada).

- **`AgregarPropiedades(control As ListViewItem, XmlParam, ValorCampo, CampoSeleccion, mcl_gestion_reglas)`**: aplica al control las propiedades EAV (obligatoriedad, visibilidad, estilos, reglas) parseando el XML de propiedades del campo; enlaza con `cl_gestion_reglas`.
- **`TraerControl(control, TipoControl) As Object`**: fabrica/localiza el control segun el tipo de respuesta (texto, numero, combo, fecha, firma, imagen, audio, GPS, etc.).

**DESTINO.** Un **FieldFactory / FieldRendererRegistry**: mapa tipo-de-campo -> renderer. En una SPA (Blazor/React) esto es metadata que el cliente interpreta; el servidor solo entrega el descriptor del campo y sus propiedades ya resueltas.

### 2.3 cl_base_formulario (clase base / modelo, 14 KB)

**ORIGEN.** DTO/POCO del campo: expone ~40 propiedades publicas (REG, ENCUESTA, PREGUNTA, CAPTION, TIPO_RESPUESTA, RESPUESTAS, ORDEN, OBLIGATORIO, ESQUEMA_CAMPO, CONTIENE_FORMULA, REG_FORMULA, ID_CAMPO, PROPIEDADES_CAMPO, CONTAINER_HTML_open/closed, ANCHO_CAMPO, etc.). Metodos: `ConvertirDatasetALista` (DataSet -> `List(Of cl_base_formulario)`), `ListarNiveles` / `AgregarsinNivel` (arma jerarquia padre-hijo), `CrearFormularioDesdeFila`.

**DESTINO.** Es el **modelo de dominio** (record `FieldDefinition` / `FormNode`). `ListarNiveles` se vuelve construccion de un arbol tipado en memoria. Ideal candidato para serializarse directo a JSON del API.

### 2.4 cl_FormCreator (API CRUD del constructor, 27 KB)

**ORIGEN.** Fachada de persistencia del **disenador** de formularios. Todos los metodos reciben `BASE_SISTEMA` explicito. Grupos:

- **Preguntas:** `GuardarPregunta` (INSERT con `OUTPUT INSERTED.REG` o UPDATE sobre `ENCUESTAS_MOV_PREGUNTAS`), `EliminarPregunta`, `ActualizarOrdenPregunta`, `ObtenerPreguntasFormulario`, `ObtenerPreguntaPorReg`, `ObtenerPreguntaPorUbicacion`, `ObtenerPreguntaPorFilaExacta`.
- **Contenedores/segmentos:** `ExisteContenedor`, `GuardarContenedor`, `ObtenerContenedoresFormulario`, `ObtenerContenedor`, `ObtenerStyleContenedor`, `ObtenerObjetosPorContenedor`, `ObtenerTiposContenedor`, `ObtenerSegmentosFormulario`, `ObtenerObjetosTipologiaSegmento` (sobre `ENCUESTAS_MOV_PREGUNTAS_T`).
- **Encuestas/formularios:** `ObtenerCuestionarios`, `ObtenerEncuesta`, `GuardarEncuesta`, `EliminarEncuesta`, `ObtenerTiposFormulario`, `ObtenerCodigosFormulario`, `ContarRespuestasEncuesta`, `ObtenerProcesosAsociadosEncuesta`.
- **Archivos/tipos:** `ObtenerNombreArchivoPorGuid`, `GuardarRelacionArchivoPregunta`, `ObtenerTablasImportacionFormulario`, `ObtenerTablaTipo`.

**DESTINO.** Es el candidato mas limpio a **FormDefinitionService / FormBuilderService** con repositorios (`IPreguntaRepository`, `IContenedorRepository`, `IEncuestaRepository`). Ya recibe `BASE_SISTEMA` por parametro, lo que facilita inyectar un `ITenantConnection`. Se expone como API REST (`/api/forms`, `/api/forms/{id}/fields`).

### 2.5 cl_gestion_reglas (motor de reglas, 15 KB)

**ORIGEN.** Ejecuta reglas configurables asociadas al formulario. Propiedades: contexto + `DataReglas` (DataSet cacheado de reglas), `preDOC_PARAMXML` (parametros pre-cargados), `ErrorSql`.

- **`EjecutarRegla(RegRegla, Tabla, RegTabla, Optional OrdenEjecutar)`**: filtra en `DataReglas` las reglas de esa referencia (LINQ, agrupadas por `ID_REGLA`/`ORDEN`), lee parametros de `FORX_DATA` (`CODIGO='EJECUTA_PARAM'`) como XML, obtiene la definicion en `CONTROL_REGLAS_R` (columna `PARAM_XML`, `TIPO`, estados 'Activo'/'Desarrollo'). Para `TIPO='Ensamblado'` usa **reflexion**: `Type.GetType`, `Activator.CreateInstance`, inyecta propiedades (DocParametros, Usuario, Sucursal, Formulario, etc.) por `SetValue` e invoca el metodo (`EVENTO`) via `MethodInfo.Invoke`. Registra trazas con `LogDebug`.

**Tablas:** `FORX_DATA`, `CONTROL_REGLAS_R`, `CONTROL_REGLAS`.

**DESTINO.** **RuleEngine** con estrategia de plugins: en vez de `Type.GetType` + reflexion cruda, un registro de `IFormRule` resueltos por DI (keyed services). El `PARAM_XML` se migra a JSON. La invocacion dinamica se reemplaza por un contrato tipado `Execute(RuleContext ctx)`. Ver como se articula hoy con el renderer en [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]].

### 2.6 cl_tree_componentes (arbol de componentes/contenedores, 9.5 KB)

**ORIGEN.** Maneja la jerarquia visual de contenedores (segmentos, filas, columnas) sobre `ENCUESTAS_MOV_PREGUNTAS_T`. Metodos: `ListarNiveles(TreeView, RegSeleccionado)` y `AutoListarxNiveles` (recursivo padre->hijos), `ListaComponentes(DropDownList)`, `EditarComponente`, `StyloComponente`, `MoverComponentes` / `MoverNodos` (reasigna `ID_PADRE`), `EliminarComponentes` / `EliminarNodos`.

**DESTINO.** **ContainerTreeService** que devuelve/actualiza un arbol JSON. El `TreeView` WebForms desaparece; el arbol se serializa y el drag-and-drop de mover/reordenar es una operacion `PATCH` de `parentId`/`order`. Logica recursiva pura y facil de portar.

### 2.7 cl_gestion_script (generacion de JavaScript de cliente, 11 KB)

**ORIGEN.** Genera el JS que el navegador ejecuta para campos especiales. `PrepararbaseScrip()` lee **plantillas** desde `PARAMXML` (`[dbx.TRABAJO]`, `CODIGO='S016'`), extrae bloques por nombre (SCRIPT.CAMPOS, SCRIPT.HTML, SCRIPT.IMAGENES, SCRIPT.FIRMA, SCRIPT.AUDIO, SCRIPT.GPS) y reemplaza tokens `@@FORMULARIO@@`, `@@URL_BASE@@`, `@@NOMBRE@@`, etc. Metodos `PrepararscriptHtml/Image/Firma/Audio/GPS` iteran las colecciones de controles registrados (`mControlesHtml`, `mControlesImage`, `mControlesFirma`, `mControlesAudio`) y emiten un fragmento por control.

**DESTINO.** Idealmente **desaparece**: en una SPA los comportamientos de firma/imagen/audio/GPS son componentes de cliente reutilizables. Si se conserva plantillado servidor, se usa un motor de templates (Razor/Scriban) y las plantillas se versionan como archivos, no como filas en `PARAMXML`.

### 2.8 cl_Transcripcion_audio (tipo de campo especial: audio -> texto, 9.5 KB)

**ORIGEN.** Convierte un campo de audio en texto. `ProcesarAudio(GuidAudio)`: lee metadata del archivo en `GEN_ARCHIVOS` por GUID, descarga el binario desde Azure Blob (`Funciones.AzureBlobStorage.GetFileBlobAsHex`), parte el audio en trozos con `InMemoryAudioSplitter.SplitToMp3Chunks` (~1400 s) y transcribe cada trozo con `TranscribirAudioBytesAsync`, que hace POST multipart a Azure OpenAI `gpt-4o-transcribe`. `ProcesarAudioNuevo` es la variante que delega en la clase `ChatGPT` (Funciones).

**Riesgo inmediato:** la `AZURE_API_KEY` esta **hardcodeada** en el codigo fuente.

**Tablas:** `GEN_ARCHIVOS`. Servicios: Azure Blob Storage, Azure OpenAI transcribe.

**DESTINO.** **AudioTranscriptionService** con `HttpClient` inyectado (`IHttpClientFactory`), clave via `IConfiguration`/Key Vault, y ejecucion asincrona real (`await`, no `.Result`/`.GetAwaiter().GetResult()`). Idealmente encolado (background worker) para audios largos.

---

## 3. Motor de calculos (cl_gestion_calculos, 44 KB)

**ORIGEN.** Calcula campos derivados (formulas entre campos). Contexto: Usuario, Sucursal, Formulario, Tabla, RegSegmento.

- **`ProcesarFormulas(RegProcesado, Tabla_id, Tabla_row, Referencia, Referenciaii, TipoLlamado, jsonData)`**: dado el campo que cambio (`RegProcesado`), busca en `ENCUESTAS_MOV_PREGUNTAS_F` todos los campos **destino** cuya formula depende de el (`REG_ORIGEN=RegProcesado -> REG_DESTINO`). Lee la propiedad `FORMULA` desde el XML EAV (`FORX_DATA`/`@FORX_DATA`, TABLA `000131_ORIGEN`). Por cada campo destino: `ExtractFieldFunctions` parsea la formula en referencias de campo (`FieldOnFunctions`: Field, Tabla, FieldTipo, FunctionName, FullName, SolicitudAgrupado). Para agregaciones ejecuta el stored proc `sp_visualizar_data_formulario_sql` (SUM/AVG/COUNT sobre las respuestas), sustituye el resultado en la formula, limpia la expresion y la evalua con **`Funciones.FormulaInterpreter.EvaluateFormula`**. Formatea numeros y acumula en `mResultFunctions` (resultado, error, IdCampo, Reg, Pregunta). Si el segmento esta oculto (`UBICA_SEGMENTO`, permisos en `SUCURSAL_GRUPOS_X_PERMISOS_X_MODULOS`), marca `SegmentoOculto` y reenvia via `ReenviarFormulas`.
- **`ExtraerValoresFormula` / `ExtraerCampoFormula`**: resuelven el valor de un campo o una lista de campos referenciados.
- **`CamposConformulas` / `ExtractFieldFunctions` / `FindIdTable` / `ProcesarTotalTabla` / `ProcesarEnlaceDatos` / `PrepararTabla`**: soporte de totales de tabla, enlaces de datos y parsing.

**DESTINO.** **CalculationEngine** con grafo de dependencias explicito (topological sort de `REG_ORIGEN->REG_DESTINO`) para recalcular solo lo afectado y detectar ciclos. El evaluador de expresiones (hoy `FormulaInterpreter`) se conserva o reemplaza por una libreria (p.ej. NCalc / Jint). Las agregaciones dejan de depender de un stored proc y se resuelven con LINQ/consultas parametrizadas. Debe ser **puro y testeable** (sin acceso directo a `HttpContext`).

---

## 4. Tablas SQL que tocan (resumen)

| Tabla ([dbx.GENE]) | Rol | Clases que la usan |
|---|---|---|
| `ENCUESTAS_MOV_PREGUNTAS` | Definicion de campos/preguntas | formularios, FormCreator, calculos |
| `ENCUESTAS_MOV_PREGUNTAS_T` | Contenedores/segmentos (arbol) | FormCreator, tree_componentes, formularios |
| `ENCUESTAS_MOV_PREGUNTAS_F` | Relaciones de formula (origen->destino) | calculos, formularios |
| `ENCUESTAS_MOV_PREGUNTAS_CONTENEDOR_R` | Campos por segmento | formularios |
| `FORX_DATA` | Propiedades EAV (TABLA `000131_ORIGEN`), parametros de regla | formularios, calculos, reglas |
| `ENCUESTA_RESP` | Respuestas capturadas | formularios |
| `CONTROL_REGLAS_R` / `CONTROL_REGLAS` | Definicion de reglas (PARAM_XML) | reglas |
| `GEN_PARAMETROS` / `TIPDOC` | Consecutivos | formularios |
| `GEN_ARCHIVOS` | Archivos (audio) en blob | Transcripcion_audio |
| `PARAMXML` ([dbx.TRABAJO]) | Plantillas de script (S016) | script |
| `SUCURSAL_GRUPOS_X_PERMISOS_X_MODULOS` | Permisos de segmento | calculos |

---

## 5. Riesgos / deuda tecnica

1. **Inyeccion SQL generalizada:** casi todo el SQL se construye por concatenacion de strings con valores de usuario (Formulario, Referencia, jsonData). Migrar obliga a parametrizar.
2. **Secreto hardcodeado:** `AZURE_API_KEY` en texto plano en `cl_Transcripcion_audio.vb`.
3. **Acoplamiento a WebForms:** varias clases reciben `ListView`, `DropDownList`, `TreeView`, `ListViewItem` como parametros; la logica de negocio esta mezclada con la UI.
4. **Reflexion sin contrato (reglas):** `EjecutarRegla` invoca ensamblados por nombre/string; fragil y sin verificacion en compilacion.
5. **Async mal implementado:** transcripcion usa `.Result` / `.GetAwaiter().GetResult()` (riesgo de deadlock).
6. **Estado por propiedades mutables** en clases de servicio (no thread-safe si se reusan).
7. **Dependencia de stored procs** (`sp_visualizar_data_formulario_sql`, `getdate3dev`) y XML EAV consultado con XPath en SQL (dificil de testear).
8. **Columnas con enye** (`CAMPANA` aparece como CAMPA[enye]A) y literales de modulo (`000131_ORIGEN`) dispersos.

---

## 6. DESTINO .NET 10 - mapeo por clase

| Clase VB (ORIGEN) | Servicio destino (.NET 10) |
|---|---|
| `cl_gestion_formularios` | FormRenderService + ResponseService + ConsecutiveService |
| `cl_FormCreator` | FormDefinitionService / FormBuilderService (+ repositorios) |
| `cl_base_formulario` | Modelos de dominio (records FieldDefinition/FormNode) |
| `cl_gestion_campo` | FieldFactory / FieldRendererRegistry |
| `cl_gestion_calculos` | CalculationEngine (grafo de dependencias + evaluador) |
| `cl_gestion_reglas` | RuleEngine (IFormRule via DI, params JSON) |
| `cl_tree_componentes` | ContainerTreeService (arbol JSON, PATCH orden/padre) |
| `cl_gestion_script` | (se elimina) componentes de cliente / templates |
| `cl_Transcripcion_audio` | AudioTranscriptionService (HttpClientFactory, Key Vault, async) |

Principios de la migracion: separar dominio de UI, consultas parametrizadas, EAV -> tablas/columnas tipadas o JSON, DI en vez de reflexion, contratos async correctos y cobertura de pruebas sobre CalculationEngine y RuleEngine.

---

## 7. Enlaces

- [[00 - Visión Formularios]]
- [[Constructor - Patron EAV y motor visual]]
- [[Esquema completo - Tablas y tipos de control]]
- [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]
- [[Visor por token - docu_viewform]]
- [[Visión y entorno]]
