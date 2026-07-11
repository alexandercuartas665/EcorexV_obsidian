---
tipo: ficha-tecnica
capa: Capa 4 - Constructor de Formularios
foco: endpoint HTTP tipo-MCP que expone el constructor de formularios a agentes de IA
componentes_origen: FormCreatorMCP.ashx + FormCreatorMCP.vb + cl_FormCreator + Gemini 2.0 Flash
persistencia: ENCUESTAS_MOV (mod 000131) y tablas satelite _PREGUNTAS_T / _PREGUNTAS / _HISTORIAL
protocolo_origen: JSON-over-HTTP con envoltura propietaria (NO JSON-RPC 2.0 estandar)
estado: documentado en profundidad (origen fiel + proyeccion destino)
nota: pieza estrella - primer punto donde un agente de IA OPERA el constructor
---

# FormCreatorMCP - Servidor MCP del constructor

> Ficha estrella. Documenta el unico punto del sistema actual donde **un agente de
> IA opera el constructor de formularios de extremo a extremo**: crea el formulario,
> agrega contenedores y campos, aplica estilos, previsualiza, publica y hace
> rollback -- todo por peticiones JSON contra un `.ashx`. Es el ancestro directo del
> Copiloto de Configuracion de la [[Agentes de IA - Arquitectura y Operacion|Capa 7 de IA]].
> El motor de datos que hay debajo se explica en [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]];
> el modelo EAV que persiste en [[Constructor - Patron EAV y motor visual]]; la vision
> del modulo en [[00 - Visión Formularios]]; el stack en [[Visión y entorno]].

Archivos origen: `Bootstrap/Formularios/Modulos/Documental/MCP/FormCreatorMCP.ashx`,
`FormCreatorMCP.ashx.vb` (handler HTTP, clase `GestionMovil.FormCreatorMCPHandler`) y
`FormCreatorMCP.vb` (clase `FormCreatorMCP`, ~1345 lineas, 20 tools).

---

## 1. Que es y por que importa

FormCreatorMCP es un **servidor de herramientas para IA** montado sobre el modulo
Documental. Su premisa: en vez de que un humano arrastre controles en el constructor
visual, un agente (o cualquier cliente HTTP) invoca "tools" atomicas -- `create_form`,
`create_container`, `create_field`, `set_style`, `publish_form` -- y compone el
formulario programaticamente. La pieza que lo vuelve notable es la tool
`import_form_from_image`: recibe la **foto o captura de un formulario** (papel o
pantalla), la manda a Gemini 2.0 Flash Vision y devuelve un *blueprint* JSON con
titulo, contenedores jerarquicos, campos tipados y estilos inferidos. Un agente puede
encadenar entonces `validate_blueprint -> create_form -> create_container* ->
create_field* -> preview_form -> publish_form` y materializar en la base de datos un
formulario que solo existia como imagen. Ese es el flujo estrella: **de imagen a
formulario ejecutable, orquestado por IA**, usando `cl_FormCreator` como capa de datos.

Importa porque es el prototipo funcional del concepto que en el destino se llama
Copiloto de Configuracion: la IA no solo conversa, sino que **muta el esquema del
producto** a traves de un contrato de herramientas verificable. Todo lo que hoy hace
este `.ashx` es la lista de capacidades que el copiloto del destino debe reproducir.

---

## 2. Arquitectura: el endpoint, el request y el response

**El .ashx.** El markup es una sola linea que enruta a la clase code-behind:
`<%@ WebHandler Language="VB" CodeBehind="FormCreatorMCP.ashx.vb" Class="GestionMovil.FormCreatorMCPHandler" %>`.
El handler implementa `IHttpAsyncHandler` e `IRequiresSessionState`; por ser .NET 4.x
adapta `Task` a `IAsyncResult` con una clase interna `TaskAsyncResult` (patron
Begin/EndProcessRequest). Es asincrono de verdad porque las llamadas a Gemini pueden
tardar hasta 300 s (el `HttpClient.Timeout` esta puesto en 300 segundos).

**El protocolo NO es MCP estandar.** Pese al nombre, esto **no** implementa Model
Context Protocol (no hay handshake `initialize`, ni `tools/list`, ni envoltura
JSON-RPC 2.0, ni transporte stdio/SSE). Es una **API RPC propietaria JSON-sobre-HTTP**:
un unico verbo POST con una envoltura plana `{tool, params}`. El nombre "MCP" es
aspiracional/arquitectonico -- de hecho el propio codigo cita como modelo de referencia
a `OrquestadorMCP/GeminiAgent.vb`. Conviene saber que en el repositorio SI existe un
MCP real y separado: el proyecto **`MCPClient`** (`src/ProtocoloMCP/JsonRpcHandler.vb`,
`MCPProtocolHandler.vb`) habla JSON-RPC 2.0 con metodos `initialize`, `listinputs`,
`close` sobre TCP. FormCreatorMCP y ese cliente comparten espiritu pero **no** el mismo
protocolo; documentar esta distincion evita confusion en la migracion.

**Formato de peticion.** POST con cuerpo JSON:

```json
{
  "tool": "create_field",
  "base_sistema": "<cadena de conexion / BASE_SISTEMA>",
  "empresa": "01",
  "api_key": "<API key de Gemini>",
  "params": { "encuesta": "F0001", "tipo_respuesta": "text", "pregunta": "Nombre" }
}
```

El handler valida que existan `tool`, `base_sistema`, `empresa` y `api_key` (400 si
falta alguno), parsea `params` como `JObject`, instancia
`New FormCreatorMCP(baseSistema, empresa, apiKey)` y llama
`Await mcp.ExecuteToolAsync(tool, params)`. Notas de transporte: fija
`Content-Type: application/json; charset=utf-8`, **responde con CORS abierto**
(`Access-Control-Allow-Origin: *`), atiende `OPTIONS` (preflight -> 200), rechaza todo
metodo distinto de POST con **405**, y devuelve 400 ante body no-JSON.

**Formato de respuesta.** Envoltura uniforme, serializada con Newtonsoft
(`Formatting.None`):

```json
{ "success": true,  "data": { ... } }
{ "success": false, "error": "mensaje" }
```

El *dispatcher* es un `Select Case toolName.ToLower().Trim()` dentro de
`ExecuteToolAsync`, envuelto en un `Try/Catch` global que captura cualquier excepcion
y la reempaqueta como `{success:false, error:"Error interno en tool '...': ..."}`. Una
tool desconocida devuelve `Tool '...' no reconocida`.

---

## 3. Catalogo de las 20 herramientas

Agrupadas por dominio. "Clase/SQL" indica si delega en `cl_FormCreator` o ejecuta SQL
directo via `MotherData.AdmDatos`.

| Tool | Parametros clave | Que hace | Clase / SQL |
|------|------------------|----------|-------------|
| `import_form_from_image` | `image_base64` (req), `mime_type`, `form_name` | Gemini Vision analiza la imagen y devuelve blueprint JSON (titulo, containers[], fields[], styles, confidence). NO persiste | `CallGeminiVisionAsync` -> API Gemini |
| `validate_blueprint` | `blueprint` (req) | Valida IDs unicos, referencias container/parent existentes, ciclos padre-hijo, tipos validos de campo y contenedor. NO toca BD | Logica en memoria |
| `create_form` | `titulo` (req), `descripcion`, `codigo`, `tipo_doc`, `tipo_formato`, `version`, `codigo_hseq` | Inserta el documento raiz del formulario; genera codigo por consecutivo si se omite | `cl_FormCreator.GuardarEncuesta` |
| `update_form` | `codigo` (req), `titulo`, `descripcion`, `version`, `tipo_formato` | Lee la fila actual y regraba con los campos cambiados (merge sobre valores previos) | `ObtenerEncuesta` + `GuardarEncuesta` |
| `list_forms` | `tabla` (def `ENCUESTAS_MOV`) | Lista formularios de la empresa activa | `ObtenerCuestionarios` |
| `get_form_structure` | `codigo` (req) | Arbol completo: metadatos raiz + contenedores + campos en un JSON | `ObtenerEncuesta` + `ObtenerContenedoresFormulario` + `ObtenerPreguntasFormulario` |
| `create_container` | `encuesta` (req), `nombre` (req), `tipo`, `id_padre`, `orden`, `style` | Inserta contenedor (row/tab/accordion/panel/column/group) y recupera su REG | INSERT directo en `..._PREGUNTAS_T` + `FindReader` |
| `update_container` | `reg` (req), `nombre`, `tipo`, `orden`, `style` | UPDATE parcial (solo campos provistos) del contenedor | UPDATE directo `..._PREGUNTAS_T` |
| `delete_container` | `reg` (req), `encuesta` (req), `strategy` (block/cascade) | Borra contenedor; `block` falla si tiene campos, `cascade` los desasocia (REG_TABLA='') | COUNT + UPDATE + DELETE directos |
| `move_node` | `reg` (req), `nuevo_orden` (req), `tipo_nodo` (campo/contenedor) | Reordena un campo o contenedor | `ActualizarOrdenPregunta` (campo) / UPDATE directo (contenedor) |
| `clone_node` | `reg` (req), `encuesta` (req), `deep` (bool) | Clona contenedor con nuevo REG; con `deep` copia tambien sus campos | INSERT directo + `GuardarPregunta` por campo |
| `create_field` | `encuesta` (req), `tipo_respuesta` (req), `pregunta` (req), `contenedor`, `caption`, `orden`, `obligatorio`, `opciones`, `ayuda`, `columna` | Crea campo de captura (text/number/date/email/phone/select/multiselect/checkbox/radio/file/signature) | `cl_FormCreator.GuardarPregunta` (INSERT) |
| `update_field` | `reg` (req), `encuesta` (req), + props opcionales | Lee campo actual y regraba con merge | `ObtenerPreguntaPorReg` + `GuardarPregunta` (UPDATE) |
| `delete_field` | `reg` (req) | Elimina campo permanentemente | `cl_FormCreator.EliminarPregunta` |
| `list_fields` | `encuesta` (req), `contenedor` (filtro) | Lista campos, opcionalmente filtrados por contenedor | `ObtenerPreguntasFormulario` + filtro en memoria |
| `set_style` | `reg` (req), `styles` (obj, req) | Guarda tokens CSS normalizados (width, padding, color, background, border, shadow...) como JSON en columna STYLE | UPDATE directo `..._PREGUNTAS_T` |
| `set_layout` | `reg` (req), `layout` (obj, req) | Mergea reglas de distribucion (columns, gap, align, breakpoint, responsive) dentro del STYLE existente | `ObtenerStyleContenedor` + UPDATE directo |
| `preview_form` | `codigo` (req) | Arma representacion renderizable: contenedores enriquecidos con estilo + campos agrupados por contenedor. NO persiste | `ObtenerEncuesta` + `...Contenedores` + `...Preguntas` + `ObtenerStyleContenedor` |
| `publish_form` | `codigo` (req), `usuario` (def `MCP`) | Snapshot JSON completo a HISTORIAL (para rollback) + incrementa VERSION | SELECT + INSERT `..._HISTORIAL` + UPDATE `ENCUESTAS_MOV` |
| `rollback_form_version` | `codigo` (req), `version` (opc; vacio = ultimo) | Restaura desde snapshot: borra estructura actual, recrea contenedores (con mapping oldReg->newReg) y campos, reajusta VERSION | DELETE + INSERT/`FindReader` + `GuardarPregunta` + UPDATE |

Contrato de tipos que el modelo debe respetar: **campos** = text, number, date, email,
phone, select, multiselect, checkbox, radio, file, signature; **contenedores** = row,
tab, accordion, panel, column, group. `validate_blueprint` es el guardian que rechaza
cualquier tipo fuera de esas listas antes de tocar la base.

---

## 4. Integracion con el motor de formularios

FormCreatorMCP es deliberadamente **delgado**: no reimplementa la logica de negocio,
la delega en `cl_FormCreator` (documentado en
[[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]).
Instancia `Private ReadOnly _formCreator As New cl_FormCreator()` y llama a sus metodos
publicos: `GuardarEncuesta`, `ObtenerEncuesta`, `ObtenerCuestionarios`,
`ObtenerContenedoresFormulario`, `ObtenerPreguntasFormulario`, `GuardarPregunta`,
`ObtenerPreguntaPorReg`, `EliminarPregunta`, `ActualizarOrdenPregunta`,
`ObtenerStyleContenedor`. Un detalle de implementacion clave: `create_field` pasa
`"0"` como `regPregunta`; como REG es INT IDENTITY y nunca existe REG=0,
`GuardarPregunta` siempre entra al branch INSERT y retorna el REG real generado.

Donde `cl_FormCreator` no ofrece metodo (contenedores), el MCP **baja a SQL crudo** con
`New MotherData.AdmDatos()`: los contenedores se manipulan con INSERT/UPDATE/DELETE
directos sobre `ENCUESTAS_MOV_PREGUNTAS_T`, recuperando el REG recien insertado con un
`SELECT TOP 1 ... ORDER BY REG DESC`. Todas las consultas usan el alias de portabilidad
`[dbx.GENE].dbo.<tabla>` (ver convencion en CLAUDE.md), de modo que el MCP hereda la
misma resolucion de catalogo fisico que el resto del sistema. Este es el gran acoplamiento:
el MCP conoce nombres de columna concretos (NOMBRE, TIPO, ORDEN, ID_PADRE, STYLE,
REG_TABLA, PREGUNTA, CAPTION, TIPO_RESPUESTA, RESPUESTAS...) y por tanto es fragil ante
cualquier cambio de esquema.

---

## 5. Uso de IA

La IA es **Gemini 2.0 Flash** de Google (no OpenAI/ChatGPT), invocada por REST directo a
`generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`. Hay
dos helpers: `CallGeminiVisionAsync` (imagen `inlineData` base64 + prompt, temp 0.2,
4096 tokens) usado por `import_form_from_image`, y `CallGeminiTextAsync` (solo texto,
temp 0.1, 2048 tokens) disponible para validaciones/transformaciones. El prompt de vision
fuerza al modelo a devolver **unicamente** un JSON con el esquema `form_title / containers
/ fields / styles / confidence`; la respuesta se limpia con `LimpiarRespuestaGemini` (que
quita cercas markdown ```` ```json ````) y se reparsea. Si Gemini no devuelve JSON valido,
la tool responde error con los primeros 200 caracteres crudos. La IA aqui **solo interpreta
imagenes**: no genera formularios desde lenguaje natural puro (para eso el agente cliente
tendria que orquestar las tools). La API key **no vive en Web.config**: viaja como
parametro `api_key` en cada request, decision explicita del diseno.

---

## 6. Seguridad y autenticacion

Este es el punto mas debil y hay que decirlo sin adornos. **No hay autenticacion de
usuario ni autorizacion por rol**: el unico "secreto" es la `api_key` de Gemini, que es
credencial del proveedor de IA, no del sistema -- cualquiera que la conozca (o que apunte a
su propia key) puede invocar todas las tools contra cualquier `base_sistema`/`empresa` que
sepa nombrar. El endpoint expone **CORS `*`**, es decir, es llamable desde cualquier
origen de navegador. Y sobre todo: **casi todas las tools de contenedor/estilo construyen
SQL por concatenacion de strings**. Hay un `Replace("'", "''")` en varios sitios (nombre,
style, snapshot) que mitiga parcialmente, pero `orden`, `tipo`, `id_padre`, `reg`,
`encuesta`, `version` entran crudos a las consultas -> **superficie de inyeccion SQL**. El
handler pide sesion (`IRequiresSessionState`) pero no valida su contenido. En resumen: apto
para un entorno cerrado/confianza-alta, inseguro para exponer a Internet tal cual.

---

## 7. Tablas SQL que toca

- **`ENCUESTAS_MOV`** (mod 000131, raiz del formulario): columnas usadas CODIGO,
  SUCURSAL, TITULO, DESCRIPCION, VERSION, TIPO_FORMATO, CODIGO_HSEQ. Insert/update via
  `GuardarEncuesta`; VERSION se pisa en publish/rollback.
- **`ENCUESTAS_MOV_PREGUNTAS_T`** (contenedores/tablas visuales): REG (INT IDENTITY),
  SUCURSAL, NOMBRE, ENCUESTA, PEDREG (INT DEFAULT 0), TIPO, ORDEN, ID_PADRE, STYLE.
  Manipulada con SQL directo.
- **`ENCUESTAS_MOV_PREGUNTAS`** (campos, patron EAV): REG, ENCUESTA, SUCURSAL, PREGUNTA,
  CAPTION, TIPO_RESPUESTA, AYUDA_PREGUNTA, RESPUESTAS, CORRECTAS, ORDEN, OBLIGATORIO,
  PREGUNTA_OTRA, COLUMNA, REG_TABLA (FK logica al contenedor). Via `GuardarPregunta` /
  `EliminarPregunta`.
- **`ENCUESTAS_MOV_HISTORIAL`** (versionado): ID, ENCUESTA, SUCURSAL, VERSION,
  SNAPSHOT_JSON, FECHA, USUARIO. El snapshot es el JSON completo del formulario; habilita
  el rollback. (Tabla especifica de este MCP -- verificar que exista/este creada.)

El esquema completo y los tipos de control estan en
[[Constructor - Patron EAV y motor visual]] y en la ficha "Esquema completo - Tablas y
tipos de control".

---

## 8. Riesgos y deuda tecnica

1. **Inyeccion SQL** en las tools de contenedor/estilo/publicacion (concatenacion; escape
   incompleto). Prioridad alta si el endpoint se expone.
2. **Sin autenticacion propia ni multitenencia forzada**: `empresa` y `base_sistema`
   llegan del cliente; nada impide operar sobre otra sucursal.
3. **CORS abierto** (`*`).
4. **"MCP" enganoso**: no es el protocolo MCP; un integrador podria esperar `tools/list` y
   no lo hay. Documentar como "API RPC estilo MCP".
5. **Acoplamiento a esquema**: nombres de columna hardcodeados; fragil a migraciones.
6. **Recuperar REG por `ORDER BY REG DESC` + NOMBRE** es carrera-propenso bajo
   concurrencia (dos contenedores del mismo nombre casi simultaneos).
7. **Rollback destructivo**: borra toda la estructura antes de recrear; si falla a mitad,
   queda inconsistente (sin transaccion envolvente).
8. **API key en el request**: comoda pero expone la credencial de IA a cada cliente y a los
   logs si no se filtra.

---

## 9. DESTINO (.NET 10): del .ashx al Copiloto de Configuracion (Capa 7)

En el destino esta pieza **no se porta como `.ashx`**: se disuelve en dos capas limpias.
Primero, un **servicio de aplicacion `FormBuilderService`** que expone exactamente estas
20 operaciones como casos de uso tipados (CQRS commands/queries) sobre el
`DynamicFormRenderer`/`FormBuilder` del destino, con validacion via FluentValidation,
persistencia por `IEcorexDbContext` (Postgres `jsonb` / SQL Server) y, crucialmente,
**parametrizacion completa** (adios concatenacion), transacciones envolventes en publish/
rollback y multitenencia forzada por `tenant_id` del contexto autenticado (no del body).
El versionado migra a `form_definition_version` (snapshot `jsonb`), lo que ya hace hoy
`ENCUESTAS_MOV_HISTORIAL` pero de forma tipada.

Segundo, y aqui esta la promesa: estas 20 tools son el **catalogo de acciones del Copiloto
de Configuracion de la Capa 7** (ver [[Agentes de IA - Arquitectura y Operacion]]). El
copiloto que "propone formularios" es precisamente un agente que dispone de `create_form`,
`create_container`, `create_field`, `set_style`, `preview_form`, `publish_form` como su
toolset -- el patron imagen->blueprint->validate->materializar ya esta probado aqui y se
generaliza a texto->blueprint (lenguaje natural: "necesito un formulario de admision con
datos del paciente, contacto de emergencia y consentimiento"). La eleccion de proveedor de
IA se abstrae detras de un `IAiProvider` (Gemini hoy, pero intercambiable), no clavado a
`generativelanguage.googleapis.com`.

Sobre **si conviene exponerlo como servidor MCP real**: si, y es la evolucion natural. En
.NET 10 el `FormBuilderService` puede envolverse con un **servidor MCP conforme** (JSON-RPC
2.0, `initialize` + `tools/list` + `tools/call`, transporte stdio o HTTP+SSE), reutilizando
literalmente el mismo catalogo de 20 tools con sus JSON Schemas de parametros. Eso permitiria
que **cualquier** agente MCP-compatible (no solo el copiloto propio) opere el constructor con
descubrimiento automatico de capacidades -- justo lo que este `.ashx` promete en el nombre
pero no cumple en el protocolo. El repositorio ya tiene el ladrillo del lado cliente
(`MCPClient` con `JsonRpcHandler`), asi que el destino consiste en **cerrar el circulo**:
convertir la envoltura propietaria `{tool, params}` en JSON-RPC estandar y dejar que el
copiloto de la Capa 7 y el servidor MCP compartan un unico `FormBuilderService`.

---

## 10. Enlaces

- [[00 - Visión Formularios]] -- vision del modulo constructor.
- [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]] -- la capa de datos que el MCP delega.
- [[Constructor - Patron EAV y motor visual]] -- persistencia EAV origen -> jsonb destino.
- [[Agentes de IA - Arquitectura y Operacion]] -- Capa 7; el Copiloto de Configuracion que hereda estas tools.
- [[Visión y entorno]] -- stack y arquitectura general (.NET 10 destino).
