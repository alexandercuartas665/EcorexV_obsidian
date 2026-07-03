---
tipo: ficha-esquema-completo
capa: Capa 4 - Constructor de Formularios
fuente_origen: consultas directas a db3dev (INFORMATION_SCHEMA + SELECT real de tipos)
proposito: plano del ETL (esquema origen) + modelo destino (jsonb / EF Core 10)
estado: barrido SQL del origen + mapeo a destino
---

# Esquema completo del Constructor de Formularios -- ETL origen y modelo destino

> Plano del ETL. Conserva el **esquema exacto del origen** (tablas, columnas y el
> catalogo real de 19 tipos de control tras barrido SQL a `db3dev`) como
> referencia de migracion, y agrega el **modelo destino** (EF Core 10 + jsonb).
> Complementa [[00 - Visión Formularios]] y [[Constructor - Patron EAV y motor visual]].

---

# PARTE A -- Modelo DESTINO (.NET 10 / EF Core 10)

## A.1 Entidades y mapeo desde el origen

| Entidad destino | Tabla origen | Nota de migracion |
|---|---|---|
| `FormDefinition` | `ENCUESTAS_MOV` | cabecera; `SUCURSAL` -> `TenantId` |
| `FormQuestion` | `ENCUESTAS_MOV_PREGUNTAS` | `TIPO_RESPUESTA` (string) -> `ControlType` (enum) |
| `FormContainer` | `ENCUESTAS_MOV_PREGUNTAS_T` | arbol via `ID_PADRE` -> `ParentId` self-ref |
| `FormDefinitionVersion` | `ENCUESTAS_MOV_HISTORIAL` | snapshot inmutable |
| `FormResponse` | `FORX_DATA` + `ENCUESTA_RESP` | **N filas EAV -> 1 fila con `data jsonb`** |
| `FormFlowLink` | `FORX_DATA_FLUJO` | -> `WorkflowInstance` + `NodeId` |

## A.2 `FormResponse` con columna JSON dual

```csharp
public class FormResponse : ITenantScoped {
    public Guid Id { get; set; }
    public Guid TenantId { get; set; }
    public string FormCode { get; set; }
    public string? Reference { get; set; }
    public string? Token { get; set; }
    public Guid? UserId { get; set; }
    public string Status { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public JsonDocument Data { get; set; }   // { "campoX": { value, ruleId, type } }
    public uint RowVersion { get; set; }     // concurrencia optimista (xmin / rowversion)
}
```

Configuracion por provider en `IEcorexDbContext`:

- **Postgres** (`PostgresDbContext`): `Data` -> columna `jsonb`, indice `GIN`.
  Consulta: `WHERE Data @> '{"ciudad":"X"}'`.
- **SQL Server** (`SqlServerDbContext`): `Data` -> `nvarchar(max)` con constraint
  `ISJSON`; indices computados sobre `JSON_VALUE(Data,'$.campo')`.

Filtro global obligatorio: `HasQueryFilter(x => x.TenantId == _tenant.CurrentTenantId)`
+ RLS en BD.

## A.3 `ControlType` (enum destino) <- 19 string-literales del origen

```csharp
public enum ControlType {
    Texto, Titulo, Lista, Abierto, Numero, Fecha,      // ~95% del volumen
    MultiplesOpciones, SiNo, ListaRoundBoton, Grafico, Button, Imagen,
    Firma, Foto, Html, Audio, Literal, Gps, GridDetalle
}
```

Cada valor mapea a un componente Blazor del `DynamicFormRenderer` (ver tabla en
[[00 - Visión Formularios]] seccion 4). El origen NO tiene catalogo en BD; el
destino lo formaliza como este enum (o un registro versionado en BD si se quiere
extensibilidad por tenant).

---

# PARTE B -- Esquema ORIGEN (plano del ETL)

> Datos exactos tras barrido SQL a `db3dev`. Se conservan como referencia de
> migracion: nombres, tipos y conteos de uso reales.

## B.1 Las 4 tablas centrales de DEFINICION

### B.1.1 `ENCUESTAS_MOV` -- Cabecera del formulario

| Columna | Tipo | Notas | Destino |
|---|---|---|---|
| `REG` | int | PK | `Id` (uuid) |
| `CODIGO` | varchar(50) | Codigo visible | `Code` |
| `TITULO` | varchar(300) | | `Title` |
| `DESCRIPCION` | text | Largo libre | `Description` |
| `CODIGO_HSEQ` | varchar(50) | Codigo HSEQ (calidad) | `HseqCode` |
| `VERSION` | varchar(50) | | `Version` |
| `TIPO_FORMATO` | varchar(50) | Clasificacion | `FormType` |
| `SUCURSAL` | varchar(10) | Tenant | `TenantId` |

### B.1.2 `ENCUESTAS_MOV_PREGUNTAS` -- Preguntas (18 columnas)

| Columna | Tipo | Notas | Destino |
|---|---|---|---|
| `REG` | int | PK | `Id` |
| `ENCUESTA` | varchar(50) | FK -> `ENCUESTAS_MOV.CODIGO` | `DefinitionId` |
| `PREGUNTA` | text | Enunciado | `Label` |
| `CAPTION` | ntext | Texto extendido | `Caption` |
| `TIPO_RESPUESTA` | varchar(50) | **Tipo de control** (string, ver B.3) | `ControlType` (enum) |
| `AYUDA_PREGUNTA` | text | Tooltip | `HelpText` |
| `RESPUESTAS` | text | Opciones (separadas por `\|`) | `Options` (jsonb array) |
| `CORRECTAS` | text | Opciones correctas (cuestionarios) | `CorrectOptions` |
| `ORDEN` | int | Posicion | `Order` |
| `OBLIGATORIO` | varchar(2) | SI/NO | `Required` (bool) |
| `SUCURSAL` | varchar(10) | Tenant | `TenantId` |
| `NUMERAL` | varchar(5) | Numeracion (1.1, 1.2) | `Numeral` |
| `PREGUNTA_OTRA` | varchar(100) | Texto campo "Otra" | `OtherLabel` |
| `COLUMNA` | varchar(10) | Bootstrap grid (col-md-6) | `GridCol` |
| `REG_TABLA` | int | FK -> `_T.REG` (contenedor) | `ContainerId` |
| `COLOR_BORDE` | varchar(30) | | `BorderColor` |
| `COLOR_FONDO` | varchar(30) | | `BgColor` |
| `FLAG_LABEL` | tinyint | Solo label, no input | `IsLabelOnly` |

### B.1.3 `ENCUESTAS_MOV_PREGUNTAS_T` -- Contenedores tipo arbol (9 columnas)

> Tabla que `cl_tree_componentes` lee para el TreeView de secciones. `_T` =
> "Tipo"/"Tree". Migra a `FormContainer` con self-reference.

| Columna | Tipo | Notas | Destino |
|---|---|---|---|
| `REG` | int | PK | `Id` |
| `SUCURSAL` | varchar | Tenant | `TenantId` |
| `NOMBRE` | varchar | Etiqueta | `Name` |
| `ENCUESTA` | varchar | FK -> `ENCUESTAS_MOV.CODIGO` | `DefinitionId` |
| `PEDREG` | int | (reg padre?) | - |
| `TIPO` | varchar | Tipo de contenedor (`TABLA`, `SEGMENTO`) | `ContainerType` |
| `ORDEN` | int | Orden | `Order` |
| `ID_PADRE` | int | Auto-referencia (arbol) | `ParentId` (self-ref) |
| `STYLE` | ntext | CSS del contenedor | `Style` |

> El visor usa `WHERE TIPO='TABLA'` para listar tablas importables desde Excel
> (`ListaTablas()`). Ver [[Visor por token - docu_viewform]].

### B.1.4 `ENCUESTAS_MOV_PREGUNTAS_CONTENEDOR` -- Contenedor legacy (5 cols)

| Columna | Tipo |
|---|---|
| `REG` | int |
| `SUCURSAL` | varchar(5) |
| `NOMBRE` | varchar(20) |
| `ENCUESTA` | varchar(5) |
| `TIPO_CONTENEDOR` | varchar(20) |

> Solo `TIPO_CONTENEDOR='SEGMENTO'` (28 usos). Tabla legacy; el modelo moderno usa
> `_T`. En el destino se unifica todo en `FormContainer` -- no se migra por
> separado.

## B.2 Persistencia de RESPUESTAS (2 caminos origen -> 1 destino)

### B.2.1 `ENCUESTA_RESP` -- Respuestas estructuradas (23 columnas)

| Columna | Tipo | Notas |
|---|---|---|
| `REG` | int | PK |
| `ENCUESTA` | varchar | FK -> `ENCUESTAS_MOV.CODIGO` |
| `PREGUNTA` | varchar | FK -> `ENCUESTAS_MOV_PREGUNTAS.REG` |
| `TIPO_REG` | varchar | Tipo de registro |
| `TIPO_RESPUESTA` | text | Copia denormalizada del tipo |
| `IDENTIFICACION` | varchar | Identificador del respondedor |
| `FECHA` | datetime | Cuando |
| `RESPUESTA_USU` | nvarchar | **El valor capturado** |
| `RESPUESTA_USU_OTRA` | varchar | Valor "Otra" |
| `PEDREG` | int | Reg del padre (lote) |
| `CAMPANIA` | varchar | Agrupar campanias |
| `REFERENCIAII` | varchar | Identificador adicional |
| `REFERENCIA_LLAVE` | varchar | Llave externa (ej. ID_CASO de tarea) |
| `REQUERIDA` | varchar | Si era obligatoria |
| `FECHA_FINALIZACION` | datetime | Cierre de la encuesta |
| `SUCURSAL` | varchar | Tenant |
| `CLI_NOM` | varchar | Nombre del cliente |
| `ESTADO` | varchar | Estado del registro |
| `ENCUESTA_RESP` | varchar | Identificador del lote |
| `AYUDA_PREGUNTA` / `ORDEN` / `NUMERAL` / `PREGUNTA_OTRA` | - | Snapshots de la pregunta |

> Patron origen: N filas por encuesta (una por pregunta). ETL: agrupar por lote y
> pivotar a un solo `data jsonb` en `FormResponse`.

### B.2.2 `FORX_DATA` -- EAV generico (camino moderno del origen)

| Columna | Tipo | Notas | Destino |
|---|---|---|---|
| `REG` | int | PK | - |
| `TABLA` | varchar | Tabla destino logica | contexto |
| `REFERENCIA` | varchar | ID del padre | `FormResponse.Reference` |
| `CODIGO` | varchar | Sub-codigo (ej. `EJECUTA_PARAM`) | clave en `data` |
| `PROPIEDAD` | varchar | Atributo | clave del objeto `data` |
| `TIPO` | varchar | Tipo de dato | metadato JSON |
| `VALOR` | ntext | El valor (string) | valor tipado en `data` |
| `TOKEN` | varchar | Token/sesion | `FormResponse.Token` |
| `USUARIO` | varchar | Quien | `UserId` |
| `FECHA_REG` | datetime | Cuando | `CreatedAt` |
| `SUCURSAL` | varchar | Tenant | `TenantId` |
| `ID_REGLA` | int | -> `CONTROL_REGLAS_R.REG` (regla que genero el dato) | `data->'$.campo.ruleId'` |
| `ORDEN_REGLA` | int | Orden de ejecucion | metadato JSON |

> `CODIGO='EJECUTA_PARAM'` (usado por `ctrReglas`): una fila con
> `REFERENCIA`=preguntaID, `ID_REGLA`=regla, `TOKEN`=id = "la pregunta X tiene la
> regla Y". En el destino esta asociacion vive en la definicion de la pregunta, no
> en la respuesta.

### B.2.3 `FORX_DATA_FLUJO` -- Estado del form en flujo BPMN (8 cols)

Conecta el formulario con `DOC_PROCESOS` cuando el form es un paso del flujo.
Migra a `FormFlowLink` (-> `WorkflowInstance` + `NodeId`).

## B.3 Catalogo REAL de `TIPO_RESPUESTA` (19 tipos en produccion)

Tipos que el constructor renderiza. **No hay catalogo en BD** -- son string
literales en `ENCUESTAS_MOV_PREGUNTAS.TIPO_RESPUESTA`. Conteo real en `db3dev`
agregando todos los tenants:

| Tipo (origen) | Uso | Categoria | Componente destino |
|---|---|---|---|
| `Texto` | 1737 | Input string | `TextControl` |
| `Titulo` | 1326 | Display | `HeadingControl` |
| `Lista` | 269 | Dropdown | `SelectControl` |
| `Abierto` | 232 | Textarea | `TextAreaControl` |
| `Numero` | 116 | Number | `NumberControl` |
| `Fecha` | 104 | Date picker | `DateControl` |
| `Multiples Opciones` | 79 | Checkbox group | `MultiCheckControl` |
| `Si/NO` | 68 | Toggle | `ToggleControl` |
| `Lista RoundBoton` | 41 | Radio group | `RadioControl` |
| `Grafico` | 34 | Chart | `ChartControl` |
| `Button` | 27 | Action | `ActionButtonControl` |
| `Imagen` | 21 | Image | `ImageControl` |
| `Firma` | 7 | Signature pad | `SignatureControl` |
| `Foto` | 4 | Photo capture | `PhotoControl` |
| `Html` | 4 | Rich HTML | `RichTextControl` |
| `Audio` | 4 | Audio capture | `AudioControl` |
| `Literal` | 3 | Plain literal | `LiteralControl` |
| `Gps` | 2 | Geo-location | `GpsControl` |
| `GridDetalle` | 2 | Detail grid | `GridControl` |

> Total: 4080 preguntas vivas en 19 tipos. Los 6 primeros cubren ~95% -> prioridad
> de migracion. Filtrando un tenant unico el conteo baja a 18 (no todos usan
> `GridDetalle`); el total global es 19.

### Aclaracion: `ADM_CONTROLES` NO es el catalogo de tipos de form

`ADM_CONTROLES` (7 filas) son controles del **shell** de la app (`TOPBAR`,
`SIDENAVBAR`, `NAVBARTOP`, `TILE MODULOS`, `TILE REPORTES`, `TILE FORMACION`,
`NEW MASTERTOPBAR`), no del form engine. El catalogo de form-engine vive
hard-coded en `cl_FormCreator` y similares -> en el destino es el enum
`ControlType` (A.3).

## B.4 Catalogo de `TIPO_CONTENEDOR` (1 tipo)

| Tipo | Uso |
|---|---|
| `SEGMENTO` | 28 |

> Solo `SEGMENTO`. El catalogo real de contenedores vive en
> `ENCUESTAS_MOV_PREGUNTAS_T.TIPO`, no aqui.

## B.5 Tablas de soporte (origen)

- **`PARAMXML`** (3 cols): definiciones XML por (`SUCURSAL`, `MODULO`) -- plantillas
  y configs estructuradas.
- **`GEN_TOKEN`** / **`GEN_PARAMETROS`**: tokens de visor. El visor
  `docu_viewform.aspx` busca el token en `GEN_PARAMETROS` (NO en `GEN_TOKEN`), con
  join a `[dbx.TRABAJO].dbo.PRO_MODULOS`. Ver [[Visor por token - docu_viewform]].
- **`BIB_BIBLIOTECA`** + **`BIB_GRUPOS`**: snippets de codigo reutilizables.

## B.6 Pendientes de barrido origen

- Detalle de `PARAMXML` (que modulos lo usan) -- problema con `-W` + `-y` en sqlcmd.
- Parser del campo `RESPUESTAS` (opciones tipo Lista/Multiples).
- Detalle del campo `STYLE` en `_T` (CSS por contenedor).
- Otros valores de `FORX_DATA.CODIGO` mas alla de `EJECUTA_PARAM`.
