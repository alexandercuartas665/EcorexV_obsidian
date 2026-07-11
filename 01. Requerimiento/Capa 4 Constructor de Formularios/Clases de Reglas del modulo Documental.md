---
capa: 4
tema: Constructor de Formularios
subtema: Subsistema de Reglas (Ensamblados)
tipo: ingenieria-inversa
origen: Bootstrap/Formularios/Modulos/Documental/Reglas
destino_net: .NET 10 RulesEngine con verbos tipados en DI
---

# Clases de Reglas del modulo Documental

> [!abstract] Que es esta nota
> Documenta las **11 clases de la carpeta `Reglas/`** del modulo Documental. Son los **"ensamblados"** (acciones ejecutables) que el motor de reglas [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas|cl_gestion_reglas]] invoca **por reflexion** cuando una regla es de tipo `Ensamblado`. Cada clase implementa un **verbo** concreto: calcular una cotizacion, imprimir una plantilla, llamar la API de Siigo, migrar datos entre formularios, generar contenido con IA, ejecutar SQL crudo, etc. Cierra el circuito abierto en [[Reglas - Quien invoca realmente (cierre)]] y [[Reglas - Catalogo real y verbos Ensamblado]].

---

## 1. Que es este subsistema: las acciones ejecutables del motor

El motor de reglas no contiene la logica de negocio de cada accion. Contiene un **despachador por reflexion**. Toda la logica vive en estas clases sueltas, cada una escrita para un tipo de accion. La union entre motor y clase es un contrato de XML + convenciones muy estricto, no interfaces ni tipos: aqui esta el corazon del diseno (y de la deuda tecnica).

**Contrato de invocacion.** En `cl_gestion_reglas.EjecutarRegla` (region `Ejecutar Reglas`), cuando `CONTROL_REGLAS_R.TIPO = "Ensamblado"`, el motor:

1. Parsea el XML `PARAM_XML` de la regla y extrae el nodo `CorXml` cuyo `NOMBRE = "ENSAMBLADO_DATA"`, obteniendo dos strings: **`PROCESO`** (nombre completo de la clase, p.ej. `GestionMovil.cl_form_reglas_formularios`) y **`EVENTO`** (nombre del metodo/verbo a invocar, p.ej. `doc_migrar_aformulario`).
2. `Type.GetType(PROCESO)` -> `Activator.CreateInstance(tipo)`: instancia la clase por su nombre-string.
3. Inyecta por reflexion **11 propiedades fijas** en la instancia (el "contexto de ejecucion"): `DocParametros`, `Usuario`, `Sucursal`, `Formulario`, `Referencia`, `Referenciaii`, `RegTabla`, `FormGuardado`, `Produccion`, `Modulo`, `DATOS_UNICOS_POR_USUARIO`.
4. `tipo.GetMethod(EVENTO).Invoke(instancia, Nothing)`: llama el verbo **sin argumentos**. El verbo es siempre un `Public Sub` parametroless.
5. Tras la invocacion lee de vuelta dos salidas de la instancia: `instancia.Referenciaii` (por si el verbo genero un nuevo consecutivo/agrupador) y `instancia.man_mensajeEmergente` (lista de `MensajeEmergente` que se propaga al usuario; si alguno trae `flag_error = True` se detiene la cadena de reglas).

**Patron comun a las 11 clases.** Todas comparten la misma anatomia:

- Las **11 propiedades** publicas de contexto (el motor las setea).
- Una `Structure MensajeEmergente { Usuario, Tecnico, Mensaje, flag_error }` y una lista `Public man_mensajeEmergente` que es el canal de salida hacia el usuario.
- Un `MotherData.AdmDatos` privado (`tbrec`) y el patron `[dbx.GENE]` para portabilidad de catalogo.
- El **verbo** (uno o mas `Public Sub` parametroless) que: (a) parsea `Me.DocParametros` (XML de `<Datos>` con tripletes `PROPIEDAD` / `VALOR` / `TIPO`), (b) mapea cada `PROPIEDAD` (los famosos `CAMPO_*`) a variables locales via `Select Case`, y (c) delega en metodos publicos auxiliares que ejecutan SQL o llaman servicios externos.

Los `CAMPO_*` son la **firma logica** de cada verbo: no hay parametros tipados, la firma se descubre leyendo el `Select Case`. Este es exactamente el punto que .NET 10 debe formalizar (seccion 9).

---

## 2. El nucleo: reglas de formularios y de tareas

### 2.1 `cl_form_reglas_formularios` (nucleo de formularios)

La clase mas central. Su verbo principal es **`doc_migrar_aformulario`** (EVENTO), que copia datos de un formulario origen a un formulario destino. Parametros que consume del XML: `CAMPO_FORMULARIO` (formulario destino), `JSON_ORIGEN_DESTINO` (mapeo campo-a-campo en JSON con `tabla_origen`/`tabla_destino`/`origen`/`destino`), `CAMPAnA_DESTINO`. Delega en `CargarDatosTabla`, el metodo mas pesado del subsistema.

Metodos publicos relevantes:

- **`doc_migrar_aformulario()`** - verbo. Desensambla parametros y llama `CargarDatosTabla`.
- **`CargarDatosTabla(json_destino, FormularioDestino, campana_destino)`** - motor de copia. Deserializa el JSON de mapeo, agrupa campos por par (tabla_origen, tabla_destino), resuelve **tokens dinamicos** `@@FECHA_CORTA@@`, `@@FECHA_LARGA@@`, `@@HORA@@`, `@@EMPRESA_NOMBRE@@`, `@@USUARIO_NOMBRE@@`, evalua **formulas** (campos que empiezan por `=`) via `SetearPrompt`, invoca el SP `sp_visualizar_data_formulario_sql` para leer datos origen, evita duplicados con un `COUNT(*)` y escribe respuestas con `UPDATE ... ENCUESTA_RESP` (modelo EAV, ver [[Constructor - Patron EAV y motor visual]]). Usa `cl_gestion_formularios` para crear registros de tabla.
- **`EnviarDatosFormulario(json_destino, Usuario)`** - variante que empuja datos a un formulario concreto generando consecutivo.
- **`CargarConsecutivoFormulario(...)`** / **`doc_asignar_consecutivo()`** / **`Asignar_consecut(...)`** - verbo y auxiliares para numeracion consecutiva de documentos.
- **`doc_aspecto_aformulario()`** - verbo que aplica aspecto/diseno.
- **`ValidarReglasdiseno(json_destino, FormularioOrigen)`** - valida reglas de diseno.
- **`SetearPrompt(BasePrompt, RegTabla)`** - puente a `cl_gestion_formularios.ExtraerValoresFormula` para resolver formulas y variables de celda.
- **`PrepararTabla(NombreTabla, Formulario)`** - normaliza nombre de tabla EAV.

### 2.2 `cl_tar_reglas_tareas` (reglas de tareas)

Verbo principal **`tar_generar_tareas_tabla`** (EVENTO): a partir de las filas de una tabla dinamica o de un formulario, genera tareas/actividades. Primero lee de `GEN_PARAMETROS` (MODULO `000636`, codigo `XML_NOTIFICACION_TAREA`) la plantilla de notificacion. Parametros: `CAMPO_TABLA`, `CAMPO_MENSAJE`, `CAMPO_CERRAR_FORMULARIOS`, `CAMPO_ASIGNAR_REFERENCIAII`. Si hay tabla llama `ListarTablatareas`, si no `ListarFormTareas`.

Metodos publicos:

- **`tar_generar_tareas_tabla()`** - verbo despachador.
- **`ListarTablatareas(...)`** / **`ListarFormTareas(...)`** - recorren filas (celda a celda) leyendo campos por convencion `CAMPO_USUARIO`, `CAMPO_PRIORIDAD`, `CAMPO_CATEGORIA`, `CAMPO_SUBCATEGORIA`, `CAMPO_ENTREGA`, `CAMPO_DETALLE`, `CAMPO_SUGERENCIA_IA`, `CAMPO_ACTIVIDAD_GENERADA`, `CAMPO_COPIAR`, `CAMPO_PLANTILLA`, etc., y crean la tarea. Usan `cl_gestion_calculos.FindIdTable` para resolver la tabla EAV.
- **`ExtaerVariable(...)`** / **`ExtaerNombreCampo(...)`** / **`CampoHomologar(...)`** - extraccion de valores de celda y homologacion contra catalogos.
- **`addConsumo(...)`** / **`ValidaComponentes(...)`** - registran consumo (inventario/facturable) asociado a la tarea.
- **`NofificarAsignado(Caso, Mensaje, Anotacion)`** / **`NofificarCasoCliente(Caso, Plantilla, Operadores, itemFormulario)`** - envian notificaciones al asignado y al cliente usando la plantilla XML cargada.
- Propiedad extra: `XMLTareaNotificacion` (cache de la plantilla de notificacion).

Estas dos clases son el 80% del trafico real de reglas: migrar datos entre formularios y disparar tareas.

---

## 3. Integraciones externas

### 3.1 `cl_sigo_reglas_api` (integracion Siigo / "SIGO")

Integra con **Siigo**, SaaS contable/de facturacion electronica colombiano (REST). Dos verbos:

- **`sigo_crear_terceros()`** - mapea un tercero (cliente) desde los `CAMPO_*` / campos del formulario a la estructura `TerceroSIGO` (type, person_type, name, id_type, identification, comercial_name, direccion, telefonos, contactos, fiscal_responsibilities_code, etc.) y hace `POST /v1/customers`.
- **`sigo_crear_facturas()`** - arma `FacturaSIGO` y hace `POST /v1/invoices`.
- **`CrearTerceroSIGO(mTercero, Token)`** / **`CrearFacturaSIGO(mFactura, Token)`** - clientes HTTP (`HttpClient`, header `Authorization: Bearer <Token>`, JSON via `System.Text.Json`).
- **`ObtenerTokenSIGO()`** - autentica contra `/auth` y devuelve el token.
- Propiedades extra: `SIGO_TOKEN`, `ENTORNO`, `CUENTA` (credenciales/entorno). Los endpoints reales (`https://api.siigo.com/...`) conviven comentados con mocks de Apiary.

### 3.2 `cl_bitcode_cotizador` (motor de cotizacion)

Verbo unico **`crear_cotizacion()`**: la clase con mas `CAMPO_*` (mas de 25). Recoge datos de cliente (`CAMPO_CLIENTE_NOMBRE/EMPRESA/EMAIL/TELEFONO/DIRECCION`), vendedor (`CAMPO_VENDEDOR_*`), condiciones (`CAMPO_TASA_CAMBIO`, `CAMPO_VIGENCIA`, `CAMPO_TITULO`, `CAMPO_CONDICIONES_PAGO`, `CAMPO_TERMINOS`, `CAMPO_NOTAS`) y las lineas de items (`CAMPO_ITEM_OPCION/NOMBRE/DESC/CANT/PRECIO/IMP`). Llama una API de cotizacion (`CAMPO_API_URL` + `CAMPO_API_KEY`), recibe el documento y lo persiste segun `CAMPO_MODO_GUARDADO` (por defecto `ZIP`) en `CAMPO_RUTA_GUARDAR`, subiendo a **Azure Blob Storage** (mismo patron `SubirABlobStorage` que la clase de impresion). Propiedad extra: `ModoGuardado`.

---

## 4. Impresion de plantillas: `cl_impresion_plantillas`

Genera documentos imprimibles a partir de HTML parametrizado por SQL. Verbo/entry principal **`CrearPlantilla()`**. Parametros: `DOCUMENTO` (plantilla base), `SQL_CABECERA1..5` (consultas que rellenan bloques de cabecera/detalle), `GENERAR_PDF` (`SI`/`NO`).

Flujo:

- **`CrearPlantilla()`** - desensambla parametros y llama `CargaCrearPlantilla`.
- **`CargaCrearPlantilla(documento, cabecera1, detalle1..4)`** - ejecuta las consultas SQL, interpola resultados en el HTML de la plantilla y produce `HtmlResultado`.
- **`GuardarComoPdf()`** - convierte el HTML a PDF y lo sube a **Azure Blob Storage** (`Funciones.AzureBlobStorage`, cuenta `AZUREBLOBSTORAGE`), registrando un GUID en `GEN_ARCHIVOS` (via `PostFileGenArchivos`).
- **`GuardarPdfEnTemp()`** - variante que deja el PDF en temporal.
- Propiedad extra: `HtmlResultado`.

Es el generador de "documentos" reales (cotizaciones, ordenes, actas) que el usuario descarga o imprime.

---

## 5. Reglas de IA: `cl_ia_reglas_formularios`

Verbo **`doc_generar_tabla()`**: usa el motor `Funciones.ChatGPT` para generar el contenido de una tabla del formulario a partir de un prompt. Parametros: `CAMPO_FORMULARIO`, `CAMPO_TABLA` (ambos normalizados a estructura via `PrepararTabla`, que inyecta el esquema de campos en el prompt) y `CAMPO_PROMPT` (procesado por `SetearPrompt`, que resuelve formulas y variables del formulario). Configura el bot (`BotSucursal`, `BotUserName`, `RestoreConfiguration`, `FormatoHtml = False`), invoca `ChatWithGPT(prompt)`, toma el ultimo mensaje del chat y lo materializa via **`CargarTabla(XML)`**, que escribe las filas generadas en la tabla EAV del formulario. Es el punto de contacto entre el constructor de formularios y los [[Agentes de IA - Arquitectura y Operacion|agentes de IA]].

---

## 6. Migracion, pruebas, SQL, usuarios y modales

- **`cl_form_reglas_migracion`** - variante de la clase de formularios para migrar desde **otra base de datos externa**. Verbo **`doc_migrardb_aformulario()`**; su `CargarDatosTabla` recibe una `cadena_conexion` explicita (ademas de `man_campana`/`man_sucursal`), de modo que lee de un catalogo ajeno y escribe en el formulario destino. Repite `PrepararTabla`, `doc_aspecto_aformulario`, `ValidarReglasdiseno`, `SetearPrompt`. Es un fork casi identico a `cl_form_reglas_formularios` (deuda: duplicacion de codigo).

- **`cl_form_reglas_pruebas`** - verbo **`doc_probar_regla()`**: banco de pruebas que, dado `CAMPO_TELEFONO` y `CAMPO_MENSAJE`, envia un WhatsApp via `clEvoApi.SendMensaje("EvolutionAPI", ...)`. Propiedad extra `OrdenEjecutar`. Sirve para validar la cadena de reglas en aislamiento.

- **`cl_operaciones_sql`** - verbo **`sql_ejecutar_consulta()`**: toma el parametro `CONSULA_SQL` (SQL **crudo** definido en la regla), sustituye `@@SUCURSAL@@` y `@@USUARIO@@`, y lo ejecuta directo con `tbrec.Execute`. Es un "escape hatch" que permite meter SQL arbitrario en una regla. Maxima flexibilidad, maximo riesgo (seccion 8).

- **`cl_bitcode_user`** - administracion de usuarios/conexiones. Verbos **`crear_conexion()`** (lee `CAMPO_USUARIO/TIPO/NOM_CONX/ID_CONX/PASS_CONX` de los campos del formulario via `LeerCampoFormulario` y llama `AddConxUser`, un SP en `sweb_biblioteca`) y **`crear_usuario()`** (`CreateUser(Nombre, Identificacion, ...)`). Nota: maneja credenciales/passwords de conexion como texto de formulario.

- **`cl_gestion_modales`** - **no** se invoca por el despachador de `EjecutarRegla`; lo llama directamente `crtCargaEncuestaII`. Verbo **`EjecutarModal(mPage, RegRegla, Tabla, RegTabla)`** (con parametros): busca en `FORX_DATA` el codigo `ABRIR_FORMULARIO_MODAL` y, via `ScriptManager` + Bootstrap Modal, abre un formulario anidado dentro de un modal (`CargarFormularioEnModal`). Es la unica clase de la carpeta orientada a UI/servidor en vez de datos. Propiedades distintas al resto: `DataReglas As DataSet`, `ErrorSql`, `FormularioGuardado`.

---

## 7. Tablas SQL involucradas

| Tabla / SP | Uso |
|---|---|
| `CONTROL_REGLAS` / `CONTROL_REGLAS_R` | Definicion de reglas; `CONTROL_REGLAS_R.PARAM_XML` trae el nodo `ENSAMBLADO_DATA` con `PROCESO`/`EVENTO`; `TIPO = Ensamblado` |
| `FORX_DATA` | Parametros de cada regla (`CODIGO = EJECUTA_PARAM`), tripletes `PROPIEDAD`/`VALOR`/`TIPO` -> `DocParametros` |
| `ENCUESTA_RESP` | Respuestas EAV del formulario (destino de escritura en migracion) |
| `sp_visualizar_data_formulario_sql` | SP que lee datos de un formulario/tabla EAV |
| `GEN_PARAMETROS` | Parametros por modulo (p.ej. `000636` / `XML_NOTIFICACION_TAREA`) |
| `GEN_ARCHIVOS` | Registro de archivos (PDF/ZIP) subidos a Blob con su GUID |
| `SUCURSAL`, `USUARIO` | Resolucion de tokens `@@EMPRESA_NOMBRE@@`, `@@USUARIO_NOMBRE@@` |
| `sweb_biblioteca` (SP) | Alta de conexiones de usuario (`cl_bitcode_user`) |

---

## 8. Riesgos y deuda tecnica

1. **Reflexion = vector de ejecucion.** `Type.GetType(PROCESO)` + `Activator.CreateInstance` + `GetMethod(EVENTO).Invoke` con `PROCESO`/`EVENTO` provenientes de **datos de BD** (`CONTROL_REGLAS_R.PARAM_XML`). Quien pueda escribir una regla puede instanciar cualquier tipo cargado e invocar cualquier metodo publico parametroless: es un **RCE de facto** limitado solo por permisos de la tabla de reglas. No hay allow-list de clases/metodos.
2. **SQL concatenado en todas partes.** `Sucursal`, `Usuario`, `Referencia`, `Referenciaii`, valores de celda y hasta el `CONSULA_SQL` completo se interpolan por concatenacion de strings a los queries. **Inyeccion SQL** sistematica; `cl_operaciones_sql` la vuelve una feature.
3. **Contrato implicito y fragil.** La firma de cada verbo son strings `CAMPO_*` en un `Select Case`. Un typo (`CAMPAnA_DESTINO` vs `CAMPANA_DESTINO`, o el `Case "CAMPO_MENSAJE"` duplicado en `cl_tar_reglas_tareas`) falla en silencio. Cero validacion de esquema.
4. **Duplicacion.** `cl_form_reglas_formularios`, `cl_form_reglas_migracion` e `cl_ia_reglas_formularios` repiten casi los mismos `PrepararTabla`/`SetearPrompt`/`CargarDatosTabla`.
5. **Secretos en claro.** `SIGO_TOKEN`, `CAMPO_API_KEY`, passwords de conexion viajan como texto de parametro/campo.
6. **Errores tragados.** `Try/Catch` que solo empujan un `MensajeEmergente` generico; el `Tecnico` guarda `ex.ToString` pero no hay logging estructurado consistente (solo `Debug/Trace` en el motor).

---

## 9. DESTINO .NET 10: verbos tipados en un RulesEngine con DI

El objetivo no es portar la reflexion, es **eliminarla**. El mapeo propuesto:

- **De string-a-clase a verbo tipado registrado en DI.** Cada verbo (`doc_migrar_aformulario`, `crear_cotizacion`, `sigo_crear_terceros`, `CrearPlantilla`, `doc_generar_tabla`, `sql_ejecutar_consulta`, ...) se vuelve una implementacion de `IRuleAction` (o un handler MediatR) con una **key estable** (`[RuleVerb("migrar_a_formulario")]`). El RulesEngine resuelve la key contra un `IReadOnlyDictionary<string, IRuleAction>` poblado por DI. Se **elimina `Type.GetType`/`Activator`**: si una regla nombra un verbo que no existe, es un error de arranque/validacion, no un `Invoke` en runtime.
- **De `CAMPO_*` en Select Case a parametros tipados.** Cada verbo declara un **record de parametros** (`MigrarAFormularioParams { string FormularioDestino; MapeoCampos Json; string CampanaDestino; }`) deserializado y **validado** (FluentValidation / DataAnnotations) desde el XML/JSON de la regla antes de ejecutar. El contrato deja de ser implicito.
- **De las 11 propiedades inyectadas por reflexion a un `RuleContext` inmutable.** `record RuleContext(Usuario, Sucursal, Formulario, Referencia, Referenciaii, RegTabla, Produccion, Modulo, ...)` pasado por constructor/parametro. Salidas (`Referenciaii` nuevo, mensajes) via `RuleResult` de retorno, no mutando la instancia.
- **De SQL concatenado a repositorios parametrizados.** `AdmDatos` -> repos EF Core / Dapper con parametros; `sql_ejecutar_consulta` se retira o se restringe a consultas whitelisted parametrizadas. Allow-list explicita de verbos permitidos por tenant.
- **Integraciones como `HttpClient` tipados.** Siigo y el cotizador pasan a `IHttpClientFactory` + clientes tipados con Polly (retry/timeout) y secretos en `IConfiguration`/Key Vault, no en parametros de regla.
- **IA como servicio inyectado.** `doc_generar_tabla` consume un `IChatCompletionService` (ver [[Agentes de IA - Arquitectura y Operacion]]) en vez del singleton `Funciones.ChatGPT`.
- **Modales fuera del motor.** `cl_gestion_modales` no pertenece al RulesEngine; en la version web moderna es responsabilidad del componente de UI (Blazor/Razor), no de una "regla".

Resultado: mismas acciones de negocio, pero con verbos descubribles, contratos validados, sin reflexion abierta ni SQL/secretos embebidos en datos.

---

## 10. Enlaces

- [[00 - Visión Formularios]]
- [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]
- [[Reglas - Quien invoca realmente (cierre)]]
- [[Reglas - Catalogo real y verbos Ensamblado]]
- [[Agentes de IA - Arquitectura y Operacion]]
- [[Visión y entorno]]
- Relacionadas: [[Constructor - Patron EAV y motor visual]], [[Esquema completo - Tablas y tipos de control]]
