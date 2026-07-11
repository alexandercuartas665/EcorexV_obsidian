---
tipo: vision-modulo
capa: Capa 4 - Constructor de Formularios
modulo_codigo_origen: "000131"
motor_destino: DynamicFormRenderer + FormBuilder (Blazor)
stack_destino: .NET 10 / ASP.NET Core / EF Core 10 / Blazor
tabla_origen: ENCUESTAS_MOV / FORX_DATA
persistencia_destino: form_response.data jsonb (Postgres) / nvarchar(max) (SQL Server)
estado: vision del sistema destino + analisis del origen como referencia ETL
---

# Vision - Constructor de Formularios (Capa 4)

> Vision del **motor de formularios dinamicos** del nuevo ECOREX sobre .NET 10.
> Este documento describe el sistema DESTINO -- el constructor visual y el
> renderer que se van a construir -- y conserva el analisis del sistema ORIGEN
> (`ENCUESTAS_MOV`, `FORX_DATA`, `crtCargaEncuestaII`, `cl_FormCreator`) como
> reglas de negocio a preservar y plano del ETL. El stack, la arquitectura y el
> aislamiento multi-tenant se definen en [[Visión y entorno]]; el aspecto y la
> navegacion definitivos en [[Visión y entorno|Prototipo Final ECOREX]].

## 1. Que es y por que existe

El Constructor de Formularios es uno de los tres motores diferenciadores de
ECOREX (junto al motor BPMN y al motor de Reglas). Su promesa: **un administrador
disenia un formulario -- preguntas, tipos de control, validaciones, secciones --
sin escribir codigo, y el sistema lo renderiza, lo asocia a una actividad o a un
paso de un flujo, captura las respuestas y las hace disponibles para reportes y
reglas.** No es un formulario estatico por pantalla: es un formulario como dato,
disenado en runtime y versionado.

El motor cubre todo el ciclo:

1. **Disenio** -- constructor visual (`FormBuilder`, sucesor de `cl_FormCreator` +
   `ctrFormCreator`) donde se arman preguntas y contenedores por arrastre.
2. **Renderizado** -- `DynamicFormRenderer` (sucesor de `crtCargaEncuestaII`)
   pinta la UI a partir de la definicion y soporta N tipos de control.
3. **Captura** -- las respuestas se persisten (patron EAV en el origen -> `jsonb`
   en el destino) con auditoria por tenant y por regla.
4. **Union con flujo** -- via `FORX_DATA_FLUJO` un paso del proceso captura datos
   estructurados; el formulario se convierte en la interfaz de un nodo BPMN.
5. **Reglas** -- cada campo puede disparar reglas (show/hide, calculo, validacion,
   autollenado, IA) resueltas por el `RulesEngine` (ver [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]).
6. **Publicacion por token** -- un formulario se comparte por URL con token; el
   visor lo renderiza para responderlo (ver [[Visor por token - docu_viewform]]).

## 2. Arquitectura destino del motor

El motor se descompone en el **Dynamic Forms Engine** de la solucion .NET 10
(seccion 8 de [[Visión y entorno]]), con estos componentes:

| Componente destino | Sucesor de (origen) | Responsabilidad |
|---|---|---|
| `FormBuilder` (Blazor) | `cl_FormCreator` + `ctrFormCreator1/2` | Constructor visual: CRUD de preguntas y contenedores, drag&drop, preview. |
| `DynamicFormRenderer` (Blazor) | `crtCargaEncuestaII` | Renderiza la definicion en UI viva y captura respuestas. Un componente Blazor por tipo de control. |
| `FormDefinitionService` | `cl_FormCreator.ObtenerCuestionarios/GuardarPregunta` | Persistencia de la definicion (cabecera, preguntas, contenedores, versiones). |
| `FormResponseService` | logica de `FORX_DATA` / `ENCUESTA_RESP` | Persistencia de respuestas en `form_response.data jsonb`. |
| `FormRuleDispatcher` | disparo de reglas en `crtCargaEncuestaII` | Detecta eventos de campo y delega en el `RulesEngine`. |
| `FormTokenService` | visor por token (`GEN_PARAMETROS`) | Emite y valida tokens de visor con caducidad, uso unico y revocacion. |

Toda entidad del motor implementa `ITenantScoped`; las consultas pasan por
`IEcorexDbContext` (DAL dual Postgres/SQL Server) con `HasQueryFilter` por
`TenantId` y RLS en BD. Cero SQL concatenado: la mayor deuda del origen
(SQLi masiva en el persist de preguntas y en el visor por token) se cierra con
EF Core parametrizado.

## 3. Modelo de datos destino (resumen)

El origen usa **EAV puro** en `FORX_DATA` (una fila por atributo). El destino
migra a un modelo document-oriented gobernado por el DAL dual:

```txt
form_definition        (cabecera: tenant_id, code, title, version, type, status)
   -> form_question    (pregunta: control_type, label, options, required, order, container_id, grid_col, style)
   -> form_container    (seccion/tabla: tipo, parent_id -> arbol, style)
   -> form_definition_version  (snapshot al editar; historial inmutable)

form_response          (respuesta: tenant_id, form_code, reference, token, user_id, created_at,
                        data jsonb / nvarchar(max))   <- 1 fila por envio, NO 1 por campo
   -> field origin     (data->'$.fieldX.ruleId' registra la regla que produjo el valor)

form_flow_link         (sucesor FORX_DATA_FLUJO: form_response <-> workflow_instance/node)
```

**Decision clave:** una respuesta = **una fila** con un `jsonb` (Postgres,
indexado GIN) o `nvarchar(max)` con JSON (SQL Server), en lugar de N filas EAV.
Esto convierte la lectura de un formulario de O(n campos) a O(1). El detalle del
patron, sus trampas y la equivalencia origen->destino esta en
[[Constructor - Patron EAV y motor visual]]. El esquema completo del origen (con
columnas reales y el catalogo de 19 tipos de control) se conserva como plano del
ETL en [[Esquema completo - Tablas y tipos de control]].

## 4. Tipos de control (renderer polimorfico)

El `DynamicFormRenderer` renderiza **N tipos de control** a partir del campo
`control_type` de cada pregunta. En el origen son 19 string-literales
hard-codeados en `TIPO_RESPUESTA`; en el destino se formalizan como un **enum
tipado** con un componente Blazor por tipo:

| Familia | Tipos | Componente destino |
|---|---|---|
| Texto | Texto, Abierto (textarea), Html (WYSIWYG), Literal | `TextControl`, `TextAreaControl`, `RichTextControl` |
| Seleccion | Lista (dropdown), Multiples Opciones, Lista RoundBoton (radio), Si/NO | `SelectControl`, `MultiCheckControl`, `RadioControl`, `ToggleControl` |
| Numerico / Fecha | Numero, Fecha | `NumberControl`, `DateControl` |
| Multimedia | Imagen, Foto, Audio, Firma | `ImageControl`, `PhotoControl`, `AudioControl`, `SignatureControl` |
| Geo / Accion | Gps, Button | `GpsControl`, `ActionButtonControl` |
| Estructura | Titulo, Grafico, GridDetalle | `HeadingControl`, `ChartControl`, `GridControl` |

Los 6 primeros (Texto, Titulo, Lista, Abierto, Numero, Fecha) cubren ~95% del
volumen del origen -- se priorizan en la migracion. El catalogo completo con
conteos de uso reales esta en [[Esquema completo - Tablas y tipos de control]].

## 5. Union formulario-flujo (`FORX_DATA_FLUJO` -> `form_flow_link`)

Un formulario cobra su maximo valor cuando es la interfaz de un **nodo BPMN**: el
flujo se detiene en un paso, presenta el formulario, el usuario lo responde y la
respuesta alimenta las reglas de la transicion. En el origen esto lo vincula
`FORX_DATA_FLUJO` (8 columnas) contra `DOC_PROCESOS`. En el destino,
`form_flow_link` asocia una `form_response` con una `workflow_instance` y su nodo
actual, de modo que:

- El Workflow Engine sabe que un nodo requiere un formulario y bloquea el avance
  hasta que la `form_response` este completa y valida.
- Las reglas del nodo leen los datos capturados para decidir la transicion.
- La trazabilidad es total: que se capturo, en que paso, por quien y cuando.

Ver [[00 - Visión Flujos]] para el lado del motor de proceso.

## 6. Reglas sobre formularios

Cada campo puede tener reglas asociadas que se ejecutan al cambiar valor, al
avanzar de seccion o al enviar. En el origen el vinculo se guarda como filas
`FORX_DATA` con `CODIGO='EJECUTA_PARAM'` (`REFERENCIA`=preguntaID,
`ID_REGLA`=regla) y las reglas se ejecutan via `cl_gestion_reglas` (reflexion
sobre verbos `Ensamblado`, p.ej. `cl_ia_reglas_formularios` para IA). En el
destino, el `FormRuleDispatcher` emite eventos de campo al `RulesEngine`, que
resuelve verbos tipados en un sandbox (sin SQL directo). Ver
[[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]
y [[Reglas - Quien invoca realmente (cierre)]].

## 7. Errores del origen que el destino corrige

1. **SQLi masiva** en el persist de preguntas (`cl_FormCreator`) y en el visor
   por token -> EF Core parametrizado en todo el motor.
2. **EAV sobre `ntext`** (deprecated, sin `LEN`, sin indexado) -> `jsonb`/`nvarchar(max)`
   con JSON, indexado GIN.
3. **Tokens de visor eternos** (sin caducidad, uso unico ni revocacion) ->
   `FormTokenService` con `expires_at`, `single_use`, `revoked_at`.
4. **Visor no anonimo** (exige `Session("Nombre")`) -> el destino soporta acceso
   por token con o sin login segun politica del formulario.
5. **HTML como vehiculo de almacenamiento** (parseo con HtmlAgilityPack, superficie
   XSS) -> definicion estructurada + sanitizacion en render.
6. **Multi-tenant por `Session("Empresa")` / `SUCURSAL`** -> `TenantId` +
   `HasQueryFilter` + RLS. Los tokens siempre filtran por tenant.
7. **Sin transacciones** en operaciones multi-tabla del guardado -> comando
   multi-entidad envuelto en transaccion.

## 8. Notas relacionadas

- [[Visión y entorno]] -- vision maestra, stack y arquitectura destino
- [[Visión y entorno|Prototipo Final ECOREX]] -- aspecto y navegacion definitivos
- [[Constructor - Patron EAV y motor visual]] -- patron EAV -> jsonb, motor visual
- [[Esquema completo - Tablas y tipos de control]] -- plano ETL (tablas y 19 tipos)
- [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]] -- motores del origen -> destino
- [[Visor por token - docu_viewform]] -- publicacion por token
- [[00 - Visión Flujos]] -- union formulario-flujo (BPMN)

---

# ORIGEN (referencia de migracion) - Constructor de Formularios (000131)

> Lo siguiente describe el modulo legacy `000131` sobre WebForms/VB.NET que se
> esta migrando. Se conserva como reglas de negocio a preservar y plano del ETL.

## O.1 Ubicacion y UI viva (origen)

| Ruta menu | URL |
|---|---|
| AUTOMATIZACION -> Formularios | `../documental/NEWFRONT_docu_formulario.aspx` |
| Visor por token | `../documental/docu_viewform.aspx?token=...` |

Hallazgos confirmados en produccion (`app.bitcode.com.co`):

- **Tabs principales**: `Recursos` (#tab-1) y `Detalle` (#tab-2).
- **Acordeones**: `Configurar pagina` (#collapse0) y `Disenio formulario` (#collapse1).
- **Botones topbar**: `Guardar`, `Eliminar`, `Limpiar`.
- **13 modales** (agregar pregunta, contenedor, control, asociar a modulo,
  configurar regla, vista previa, etc.) y **6 grids**.
- **425 elementos** con id `FormCreator`/`formcreator`/`Cuestionario`.
- **Campos cabecera**: Consecutivo, Codigo, Nombre, Version, Codigo, Tipo
  Formulario, Descripcion.
- **Formularios reales**: `00001` (desarrollo, AGENTE IA gestion comercial),
  `00031` (terminado, SKY SYSTEM COMERCIAL CAPTURA REQUERIMIENTO).
- **Estados observados**: `desarrollo`, `terminado` -- analogos al `ESTADO` de
  `CONTROL_REGLAS` (enum probable Desarrollo/Activo/Inactivo/terminado).

El visor (`docu_viewform.aspx?token=...`) es compartido por Autocompletado
(000801), Agentes IA (000867) y Pruebas Formularios (000891) -- todos usan el
mismo runtime.

## O.2 Page_Load del origen (regla de negocio a preservar)

```vb
topBar.Modulo = "000131"
topBar.Ubicacion = "Home/Documental/Creacion de Formularios"

' ctrFormCreator1 = DISENIO (preguntas)  -> FormBuilder en destino
ctrFormCreator1.BaseSistema = BASE_SISTEMA
ctrFormCreator1.ManejoTabla = "[ENCUESTAS_MOV]"
ctrFormCreator1.Empresa = Session("Empresa")            ' -> TenantId
ctrFormCreator1.CodigoFormulario = cmbcodigo.SelectedValue
ctrFormCreator1.ManejoConsecutivo = "03D"
ctrFormCreator1.MostrarListadoCuestionarios(False)
ctrFormCreator1.MostrarListadoPreguntas(True)

' ctrFormCreator2 = LISTADO (gestor)
ctrFormCreator2.MostrarListadoCuestionarios(True)
ctrFormCreator2.MostrarListadoPreguntas(False)

AddHandler ctrFormCreator1.SolicitarRedibujo, AddressOf ctrFormCreator1_SolicitarRedibujo
AddHandler ctrFormCreator2.SolicitarCargaEncuesta, AddressOf CargarEncuestaDesdeCtrFormCreator
```

Dependencia: `HtmlAgilityPack` (parseo del HTML del constructor). Eventos topbar:
Save -> `AgregaEncuesta()`, Delete -> `QuitarEncuesta()`, Clear -> reset +
`LimpiarEncuesta()`.

## O.3 Persistencia del origen (plano del ETL)

Definicion (esquema):

| Tabla | Rol | Migra a |
|---|---|---|
| `ENCUESTAS_MOV` | Cabecera del formulario | `form_definition` |
| `ENCUESTAS_MOV_HISTORIAL` | Versiones/historial | `form_definition_version` |
| `ENCUESTAS_MOV_PREGUNTAS` | Preguntas | `form_question` |
| `ENCUESTAS_MOV_PREGUNTAS_T` | Contenedores (arbol) | `form_container` |
| `ENCUESTAS_MODULOS` | Asociacion form <-> modulo | tabla de asociacion + module registry |
| `ENCUESTAS_GRUPOS` (+ `_X_MODULOS`) | Familias de formularios | catalogo de grupos |
| `ENCUESTAS_DOCUMENTOS` / `_RECURSOS` / `_DOCU_ARCHIVOS` | Adjuntos | object storage + metadata |

Respuestas:

| Tabla | Rol | Migra a |
|---|---|---|
| `ENCUESTA_RESP` | Respuestas estructuradas (N filas/pregunta) | `form_response.data` |
| `FORX_DATA` (13 cols) | EAV generico | `form_response.data jsonb` |
| `FORX_DATA_FLUJO` (8 cols) | Estado del form en flujo BPMN | `form_flow_link` |

Soporte/runtime: `PARAMXML` (defs XML por modulo), `GEN_TOKEN`/`GEN_PARAMETROS`
(tokens de visor), `ADM_CONTROLES` (catalogo del shell -- NO del form engine),
`BIB_BIBLIOTECA`/`BIB_GRUPOS` (snippets).

## O.4 Componentes UI del origen

| Componente | Rol | Sucesor destino |
|---|---|---|
| `ctrFormCreator1` (UserControl) | Constructor visual (preguntas, drag&drop) | `FormBuilder` |
| `ctrFormCreator2` (UserControl) | Gestor (listado de cuestionarios) | listado en `FormBuilder` |
| `ctrModulos` (UserControl) | Asociador form <-> modulos | integracion Module Registry |
| `cl_tree_componentes` (clase VB) | Arbol de secciones/contenedores | modelo de `form_container` |
| `HtmlAgilityPack` | Parseo del HTML del constructor | definicion estructurada |

## O.5 Riesgos del origen (todos corregidos en destino -- ver seccion 7)

- SQLi masiva en el persist de preguntas (SQL concatenado).
- Estado del editor en JS del cliente; postback puede perder cambios.
- HTML como almacenamiento (superficie XSS).
- `GEN_TOKEN`/`GEN_PARAMETROS` sin caducidad ni revocacion.
- Multi-tenant fragil por `Session("Empresa")` / `SUCURSAL`.
