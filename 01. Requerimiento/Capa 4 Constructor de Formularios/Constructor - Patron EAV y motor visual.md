---
tipo: ficha-tecnica
capa: Capa 4 - Constructor de Formularios
foco: patron de persistencia (EAV origen -> jsonb destino) y motor visual
motor_destino: DynamicFormRenderer + FormBuilder
persistencia_destino: form_response.data jsonb (Postgres) / nvarchar(max) (SQL Server)
estado: documentado en profundidad (destino + referencia origen)
---

# Persistencia y motor visual del Constructor de Formularios

> Ficha tecnica que profundiza en [[00 - Visión Formularios]]. Explica **como se
> almacena** un formulario y sus respuestas -- el destino document-oriented
> (`jsonb`/`nvarchar(max)`) y el origen EAV (`FORX_DATA`) que se conserva como
> plano del ETL -- y **como se renderiza** (motor visual polimorfico). Stack y
> arquitectura en [[Visión y entorno]]; aspecto en [[00 - Prototipo Final ECOREX]].

---

## 1. Modelo de persistencia DESTINO (document-oriented)

El destino abandona el EAV puro (una fila por atributo) por un modelo
**document-oriented** gobernado por el DAL dual `IEcorexDbContext`: cada respuesta
es **una fila** con un documento JSON. Esto preserva la flexibilidad del EAV
(cero migracion de esquema al cambiar un formulario) sin sus trampas de lectura.

### 1.1 Definicion del formulario (esquema)

```txt
form_definition
   id (uuid PK), tenant_id, code, title, description, version, form_type, status,
   created_at, updated_at, row_version (concurrencia optimista)

form_question
   id, tenant_id, definition_id (FK), control_type (enum), label, caption,
   help_text, options (jsonb), required, order, container_id (FK, nullable),
   grid_col, border_color, bg_color, is_label_only

form_container   (arbol de secciones/tablas)
   id, tenant_id, definition_id, name, container_type, parent_id (self-ref),
   order, style

form_definition_version   (historial inmutable)
   id, tenant_id, definition_id, version, snapshot (jsonb), created_at, created_by
```

### 1.2 Respuestas (una fila = un envio)

```txt
form_response
   id (uuid PK), tenant_id, form_code, reference, reference2, token,
   user_id, status, created_at, finalized_at, row_version,
   data jsonb            -- Postgres: columna jsonb indexada GIN
                          -- SQL Server: nvarchar(max) con JSON (OPENJSON/JSON_VALUE)
```

El documento `data` tiene forma `{ "campoX": { "value": ..., "ruleId": ..., "type": ... } }`.
El origen del valor (usuario vs regla) se guarda dentro del propio JSON
(`ruleId`), reemplazando las columnas `FORX_DATA.ID_REGLA` / `ORDEN_REGLA`.

### 1.3 Union con flujo

```txt
form_flow_link   (sucesor de FORX_DATA_FLUJO)
   id, tenant_id, form_response_id (FK), workflow_instance_id (FK), node_id,
   status
```

Vincula la respuesta con la instancia de flujo y su nodo actual. El Workflow
Engine bloquea el avance hasta que la respuesta este completa y valida.

### 1.4 Por que jsonb / nvarchar(max) y no EAV

| Ventaja | Detalle |
|---|---|
| **Lectura O(1)** | Pintar un form de 30 campos = leer 1 fila, no 30 SELECT sobre EAV. |
| **Cero ALTER TABLE** | Agregar/quitar un campo del formulario no toca el esquema fisico. |
| **Indexable** | Postgres indexa el `jsonb` con GIN; SQL Server permite indices computados sobre `JSON_VALUE`. |
| **Consultable** | `data @> '{"ciudad":"X"}'` (Postgres) / `JSON_VALUE(data,'$.ciudad')` (SQL Server) sin EXISTS por campo. |
| **Tipado en app** | El `control_type` de `form_question` dicta el parseo; el JSON conserva el valor tipado. |

### 1.5 ETL EAV -> jsonb (round-trip validado)

El ETL agrupa las filas `FORX_DATA` (o `ENCUESTA_RESP`) de un mismo
`(TABLA, REFERENCIA)` y las pivota a un solo documento `data`. Riesgo principal:
perdida de campos por typos historicos en `PROPIEDAD`. Mitigacion: tests de
round-trip campo por campo (ver riesgos en [[Visión y entorno]] seccion 17).

---

## 2. Motor visual DESTINO (renderer polimorfico)

El `DynamicFormRenderer` (Blazor, sucesor de `crtCargaEncuestaII`) recibe una
`form_definition` con sus `form_question` y `form_container`, y compone la UI
resolviendo cada `control_type` a un **componente Blazor**. Es un renderer
polimorfico: agregar un tipo de control nuevo = registrar un componente, sin
tocar el motor.

```txt
DynamicFormRenderer(definition)
  para cada container (arbol, ordenado):
     para cada question (ordenada, respetando grid_col):
        resolver control_type -> componente Blazor
        bindear data[question.code] <-> componente
        enganchar eventos (change/blur/click) -> FormRuleDispatcher
```

- **Modos** (heredados de `crtCargaEncuestaII`): disenio vs produccion, con tabs
  vs plano (`FORM_CARPETA_VISOR`), auto-guardado, botones visibles/ocultos.
- **Reglas por campo**: cada control emite eventos al `FormRuleDispatcher`, que
  consulta las reglas asignadas y delega en el `RulesEngine` (show/hide, calculo,
  autollenado, IA). Reemplaza el patron origen `FORX_DATA WHERE CODIGO='EJECUTA_PARAM'`.
- **El constructor** (`FormBuilder`) es el lado de disenio: CRUD de preguntas y
  contenedores, drag&drop, preview en vivo del mismo renderer.

Detalle de los motores y su equivalencia origen->destino en
[[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]].

---

## 3. Tipos de control soportados

19 tipos en el origen (string-literales en `TIPO_RESPUESTA`) -> **enum tipado**
en el destino, un componente Blazor por tipo. Catalogo completo con conteos de
uso reales en [[Esquema completo - Tablas y tipos de control]]. Los 6 mas usados
(Texto, Titulo, Lista, Abierto, Numero, Fecha) cubren ~95% del volumen y se
priorizan. El origen no tiene catalogo en BD (`ADM_CONTROLES` es del shell, no del
form engine) -- el destino lo formaliza como enum/registro.

---

## 4. Errores del origen corregidos aqui

| Error origen | Correccion destino |
|---|---|
| `VALOR ntext` (deprecated, sin LEN, sin full-text) | `jsonb` / `nvarchar(max)` con JSON |
| Lectura O(n x m) del form (N SELECT por EAV) | 1 fila `jsonb` = O(1) |
| Sin integridad referencial (`PROPIEDAD` string libre) | `form_question` tipada + FK a definicion |
| SQLi en persist de preguntas | EF Core parametrizado |
| Multi-tenant por `SUCURSAL` | `TenantId` + `HasQueryFilter` + RLS |
| Sin transaccion multi-tabla | comando en transaccion + concurrencia optimista (`row_version`) |

---

# ORIGEN (referencia ETL) - patron EAV `FORX_DATA`

> El sistema legacy almacena definiciones y respuestas en un conjunto de tablas
> EAV. Se conserva como reglas de negocio a preservar y plano del ETL.

## O.1 Anatomia de un formulario en la BD origen

Un formulario disenado toca **al menos 7 tablas** en 3 capas:

### Capa Esquema (definicion)

```txt
ENCUESTAS_MOV  (cabecera)
   REG, CODIGO, TITULO, DESCRIPCION, CODIGO_HSEQ, VERSION, TIPO_FORMATO, SUCURSAL

ENCUESTAS_MOV_PREGUNTAS  (preguntas Q1, Q2, ...)
   PREGUNTA, TIPO_RESPUESTA, OBLIGATORIO, ORDEN, RESPUESTAS (opciones), ...

ENCUESTAS_MOV_PREGUNTAS_T  (contenedores/secciones, arbol ID_PADRE)

ENCUESTAS_MOV_HISTORIAL  (versiones)
```

### Capa Asociacion

```txt
ENCUESTAS_MODULOS              form <-> modulo
ENCUESTAS_GRUPOS (+_X_MODULOS)  familias de formularios
ENCUESTAS_DOCUMENTOS / _RECURSOS / _DOCU_ARCHIVOS   adjuntos
```

### Capa Respuestas (EAV)

```txt
ENCUESTA_RESP    respuestas estructuradas (1 fila por pregunta respondida)

FORX_DATA        almacen EAV generico (el mas interesante)
   REG, TABLA, REFERENCIA, CODIGO, PROPIEDAD, TIPO, VALOR (ntext),
   TOKEN, USUARIO, FECHA_REG, SUCURSAL, ID_REGLA, ORDEN_REGLA

FORX_DATA_FLUJO  estado del form en un flujo BPMN (-> DOC_PROCESOS_R)
```

### Capa Soporte

```txt
ADM_CONTROLES + _I + _V   (catalogo del SHELL, no del form engine)
GEN_TOKEN / GEN_PARAMETROS  tokens de visor
PARAMXML                    defs XML por modulo
BIB_BIBLIOTECA + BIB_GRUPOS  snippets reutilizables
```

## O.2 El patron EAV explicado

`FORX_DATA` implementa Entity-Attribute-Value puro:

| Columna | Concepto EAV | Migra a |
|---|---|---|
| `TABLA` | Tipo de entidad | contexto de `form_response` |
| `REFERENCIA` | ID del registro padre | `form_response.reference` |
| `CODIGO` | Sub-codigo del campo | clave dentro de `data` |
| `PROPIEDAD` | Atributo (nombre del campo) | clave del objeto `data` |
| `TIPO` | Tipo de dato | `control_type` / metadato en JSON |
| `VALOR` (ntext) | El valor (string) | valor tipado en `data` |
| `TOKEN` | Sesion/visor | `form_response.token` |
| `USUARIO`, `FECHA_REG`, `SUCURSAL` | Auditoria | columnas + `tenant_id` |
| `ID_REGLA`, `ORDEN_REGLA` | Regla que produjo el dato | `data->'$.campo.ruleId'` |

### Por que EAV (origen)

- Cero migracion de esquema al cambiar campos del formulario.
- Unico almacen para el 100% de los formularios del tenant.
- Trazabilidad por regla (`ID_REGLA` + `ORDEN_REGLA`).

### Las trampas del EAV origen (motivan la migracion a jsonb)

| Trampa | Detalle |
|---|---|
| Lectura lenta | 30 campos = 30 SELECT (o PIVOT) sobre `FORX_DATA`. O(n x m) sin indice cuidadoso `(TABLA, REFERENCIA, PROPIEDAD)`. |
| Sin tipos fisicos | Todo `ntext`; comparar fechas/numeros exige parsear en app. |
| Sin integridad referencial | `TABLA`/`PROPIEDAD` strings libres; un typo crea un campo fantasma. |
| Joins complejos | Reportes con condiciones = un EXISTS por campo. |
| `VALOR ntext` | Deprecated desde SQL 2005; sin `LEN` directo, sin full-text estandar. |

### Estrategia de migracion (confirmada en destino, seccion 1)

- Postgres `jsonb` (1 columna por respuesta, lectura O(1), indice GIN) /
  SQL Server `nvarchar(max)` con JSON.
- NO migrar a tablas dinamicas por formulario (el constructor admite cambios
  constantes, no serian estables).

## O.3 Controles disponibles en el origen (`ADM_CONTROLES`)

`ADM_CONTROLES` (+ `_I`, `_V`) administrado en el modulo 000282
(`NEWFRONT_adm_controles.aspx`). **Aclaracion clave:** sus 7 filas son controles
del **shell** (`TOPBAR`, `SIDENAVBAR`, `NAVBARTOP`, tiles), **no** el catalogo de
tipos de control de formularios. El catalogo real de form-engine vive
hard-coded en el codigo (`cl_FormCreator` y similares) -> el destino lo formaliza
como enum. Detalle en [[Esquema completo - Tablas y tipos de control]].

## O.4 UserControls del constructor (origen)

| Control | Rol | Sucesor destino |
|---|---|---|
| `ctrFormCreator1` | Disenador (drag&drop) | `FormBuilder` |
| `ctrFormCreator2` | Listado de cuestionarios | listado en `FormBuilder` |
| `ctrModulos` | Asociador form <-> modulos | Module Registry |
| `cl_tree_componentes` | Arbol de secciones (clase VB) | modelo `form_container` |

Eventos wired en `Page_Load`:

```vb
AddHandler ctrFormCreator1.SolicitarRedibujo, AddressOf ctrFormCreator1_SolicitarRedibujo
AddHandler ctrFormCreator2.SolicitarCargaEncuesta, AddressOf CargarEncuestaDesdeCtrFormCreator
```

`HtmlAgilityPack` parsea el HTML que el disenador genera (superficie XSS; en el
destino la definicion es estructurada, no HTML).

## O.5 Visor por token y motor de reglas (origen)

- **Visor por token** (`docu_viewform.aspx?token=...`): resuelve el token contra
  `GEN_PARAMETROS` (no `GEN_TOKEN`). Detalle completo en
  [[Visor por token - docu_viewform]].
- **Reglas por campo**: filas `FORX_DATA` con `CODIGO='EJECUTA_PARAM'` asocian una
  pregunta (`REFERENCIA`) a una regla (`ID_REGLA`). El renderer detecta eventos y
  ejecuta via `cl_gestion_reglas`. Detalle en
  [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]
  y [[Reglas - Quien invoca realmente (cierre)]].

## O.6 Patron del Page_Load (origen)

```vb
Sub Page_Load:
   topBar.Modulo = "000131"
   ' ctrFormCreator1 (disenador): ManejoTabla="[ENCUESTAS_MOV]", Empresa=Session("Empresa")
   '                              MostrarListadoPreguntas(True)
   ' ctrFormCreator2 (listado):   MostrarListadoCuestionarios(True)
   ' Wire: SolicitarRedibujo -> UpdatePaneldatos.Update
   '       SolicitarCargaEncuesta -> CargarEncuestaDesdeCtrFormCreator
   If Not IsPostBack:
      ListaCodigo() : ctrFormCreator1.InicializarEditor() : ctrFormCreator2.CargarCuestionarios()

Topbar: Save -> AgregaEncuesta(); Delete -> QuitarEncuesta(); Clear -> LimpiarEncuesta()
Modal "Asociar a modulo": ctrModulos.ListarModulos() + RegisterStartupScript(bootstrap.Modal.show)
```

En el destino esto se reduce a inyectar `FormBuilder`/`DynamicFormRenderer` con
`TenantId`, el `code` del formulario y el modo; el wiring por eventos VB
desaparece a favor del binding declarativo de Blazor.
