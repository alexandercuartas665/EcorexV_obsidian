---
tipo: spec-arquitectura
proyecto: Propiedades avanzadas del Constructor de Formularios
proposito: Decisiones de arquitectura (formulario-modulo, transaccionalidad, identidad/consecutivo, lookups de datos, calculo, filtros) + modelo de datos objetivo y la secuencia de consumo de un registro transaccional.
---

# 01 - Arquitectura, decisiones y datos (Formularios avanzados)

> Este doc asume el motor validado en
> [[00 - INDICE y objetivo (Formularios avanzados)]] seccion 1. Cada decision lleva
> opciones y una recomendacion (patron del vault). Las reglas transversales del proyecto
> (multi-tenant real, DAL dual, transaccion, nada de SQL concatenado ni formulas arbitrarias)
> son obligatorias y estan al final.

## 1. Decisiones de arquitectura

### D1 - Formulario como MODULO del sistema
Hoy todo formulario vive dentro del gestor generico `/formularios`. El usuario quiere
**promover** ciertos formularios a modulo de primer nivel.

- **(a) Bandera `IsModule` + nodo de menu generado (recomendado).** En las propiedades del
  formulario se marca "Es un modulo del sistema": se crea/asocia un `MenuNode`
  (`Kind=FormModule`, con `FormDefinitionId`, icono y titulo), se generan **policies**
  `Form.{code}.Ver/Crear/Editar/Eliminar` y la ruta `/m/{code}` renderiza la **bandeja**
  del formulario (ver D6). Reusa el menu data-driven (`MenuConfigService`,
  `MenuTreeBuilder`, `MenuPermissionFilter`) y el modelo de permisos ya existente.
- (b) Modulo "hardcoded" por cada formulario (no escala, rompe el data-driven).

Recomendacion: **(a)**. Un formulario-modulo es, en esencia, "definicion + bandeja + permisos
+ nodo de menu", todo derivado de la definicion.

### D2 - Transaccionalidad: la respuesta se vuelve un REGISTRO (hecho)
Hoy `FormResponse` es "una respuesta". Para "crear transacciones" el registro necesita
identidad de negocio, estado y fecha.

- **(a) Bandera `IsTransactional` en la definicion (recomendado).** Cuando esta activa,
  cada envio confirmado es un **registro** con:
  - `RecordNumber` (segun D3),
  - `Status { Draft, Confirmed, Voided }` (borrador -> confirmado; anular NO borra),
  - `TransactionDate` (fecha del hecho, por defecto hoy, editable segun politica),
  - `data jsonb` (los campos, como hoy) + **dimensiones** (ver D4: los campos lookup
    guardan el id del tercero/item/etc., que son las FKs de la tabla de hechos).
  Un formulario NO transaccional se comporta como hoy (captura suelta).
- (b) Tabla de hechos fisica por formulario (una tabla real por definicion). Potente para BI
  pero rompe el modelo generico y complica el DAL dual; se descarta para v1 (se puede exponer
  despues como **vista/proyeccion** desde el `jsonb`).

Recomendacion: **(a)**. "Tabla de hechos" = el conjunto de `form_record` de una definicion
transaccional; las **dimensiones** son los campos lookup (FKs). Para BI/Power BI se expone
una **vista** que aplana `data` + dimensiones (sin duplicar almacenamiento).

### D3 - Identidad del registro (los 2 modos que pidio el usuario)
Propiedad `IdentityMode` en la definicion:

- **`NaturalKey` (identidad externa).** El numero/clave sale de **un campo** del propio
  formulario (`IdentitySourceFieldCode`): un SKU, el id de un tercero, un consecutivo
  externo. Se valida **unicidad** por tenant (`UniqueKeyFieldCodes`, uno o varios campos).
  Caso: "el id lo pone el dato" (una ficha por SKU, una por cliente).
- **`Sequence` (consecutivo del sistema).** En la config del formulario se elige/crea una
  **secuencia** (`TenantSequence`): prefijo, longitud, reinicio (nunca/anual/mensual),
  y opcional **por entidad** (una numeracion por sede/area). Al **confirmar** el registro se
  **consume** el siguiente numero via `ISequenceService` (el mismo que usan las tareas y el
  verbo `AsignarConsecutivoVerb`). Caso: cotizacion COT-2026-000123, permiso, remision.
- `None` (default): formulario no transaccional, sin identidad de negocio.

Reglas del consecutivo: se consume **solo al confirmar** (no en borrador), dentro de la
transaccion; anular un registro **no libera** el numero (queda el hueco, trazable); reintentos
son idempotentes por `FormResponse.Id`.

### D4 - Campos de datos: lookup / autocompletado (dominio del tenant)
Nuevo comportamiento de campo gobernado por sus **propiedades de datos**. Se propone un tipo
de control `Lookup` (o `EntityRef`) y/o extender Select con un **origen de datos**:

- **`SourceKind`**: `Options` (fijas, como hoy) | `DataContainer` | `Tercero` (Directorio) |
  `Item` (Inventario) | (extensible a otras entidades del tenant).
- **`SourceRef`**: id del contenedor de datos o la entidad objetivo.
- **`DisplayField` / `ValueField`**: que se muestra vs que se guarda (se guarda el **id**).
- **`FilterJson`**: scope del catalogo (ej. solo terceros `EsCliente=true`, items de una
  bodega). Siempre **filtrado por tenant** (dominio del cliente) por el filtro global.
- **`AutofillMapJson`**: al elegir una opcion, **autollenar** otros campos
  (ej. cliente -> NIT, direccion, ciudad; item -> precio, unidad). Mapea campos de la fuente
  (incluidos los **campos dinamicos** de `TerceroFieldService`/`ItemFieldService`) a campos
  del formulario.
- **Presentacion**: `autocomplete` (typeahead server-side, paginado), `dropdown` (lista) o
  `modal` (buscador). El usuario pidio los tres modos.

Reusa: `DataContainerService` (contenedor generico), `TerceroService`/`ItemService`
(entidades del negocio) y sus `*FieldService` para el autollenado. La busqueda es
**server-side** y parametrizada (nada de traer todo el catalogo al cliente).

### D5 - Calculo y agregacion (formulas, totales)
- **Campo calculado**: `CalcExpression` sobre otros campos (aritmetica, condicional, fechas,
  texto). Ej. `total = cantidad * precio_unitario * (1 - descuento)`.
- **Totales en tabla (GridDetail)**: por columna, un `Aggregate { None, Sum, Count, Avg, Min,
  Max }` que pinta una **fila de totales**. Ademas **columna calculada** por fila (subtotal =
  cant * precio).
- **Roll-up detalle -> encabezado**: el total de una columna del GridDetail alimenta un campo
  del formulario (ej. `total_general` = suma de subtotales del detalle).
- **Motor**: expresiones **tipadas en sandbox**, NO codigo arbitrario (se evita el RCE de
  facto del legacy). Se alinea con el `RulesEngine` (mismo evaluador de expresiones). Se
  **evalua en cliente** para UX inmediata y se **revalida en servidor** al guardar (fuente de
  verdad; el cliente nunca es confiable para montos).

### D6 - Filtros / bandeja del formulario-modulo
Cuando un formulario es modulo (D1) y/o transaccional (D2), su ruta muestra una **bandeja**:

- **Grid de registros** con columnas configurables (numero, fecha, estado + campos elegidos).
- **Filtros dinamicos** por los campos del formulario (incluidos los dinamicos y los lookup),
  rangos de fecha y busqueda; los filtros persisten como "indicadores de crecimiento" (el
  usuario los describio en el gestor de contactos: cantidad, % crecimiento por bolsa).
- **KPIs** derivados de los filtros (conteo, suma de un campo monetario, % vs periodo previo).
- **Export** (Excel/CSV) y base para **BI** (vista aplanada, D2). El stack ya tiene Power BI
  en el ecosistema.
- Acciones por registro segun permisos: ver, editar (si Draft/politica), anular, imprimir (D8).

### D7 - Maestro-detalle entre formularios
Mas alla del GridDetail (detalle embebido en el mismo `data`), a veces el detalle son
**registros de OTRO formulario** (cotizacion -> muchas lineas que son fichas de item).

- **(a) Campo `Subform`/relacion (recomendado para v2).** Un campo referencia una definicion
  hija; sus filas son `form_record` hijos con FK al padre. Habilita reportar el detalle por
  separado y reusarlo.
- (b) Solo GridDetail (todo en el `jsonb` del padre) para v1: mas simple, suficiente para la
  mayoria de casos. Recomendado empezar por (b) y subir a (a) cuando se necesite.

### D8 - Transversales creativos (que completan "el mejor sistema")
- **Valores por defecto dinamicos**: `DefaultDynamic` = `CurrentUser`, `Today`,
  `CurrentEntidad`, `Sequence.peek`, etc.
- **Mascaras y formato**: `Format/Mask` por campo (moneda, %, telefono, documento, fecha).
- **Permisos a nivel de campo**: `FieldVisibilityJson` (ver/editar por rol) para campos
  sensibles (precio de costo, margen).
- **Impresion / PDF con plantilla**: generar el PDF del registro desde una plantilla (sucesor
  de las plantillas PDF->Blob del legacy); adjuntarlo al object storage.
- **Webhooks / eventos al guardar**: al confirmar, emitir evento (SignalR + cola) y opcional
  **webhook tipado** a integraciones (Siigo, WhatsApp HSM, correo). Reusa el patron de verbos
  del `RulesEngine` (accion `Ensamblado` tipada, con allow-list, NO reflexion abierta).
- **Adjuntos Tier 2**: activar captura real de foto/firma/GPS/archivo/barcode + object storage
  (cierra los placeholders del renderer).
- **Import desde imagen (IA)**: foto -> blueprint -> formulario; pertenece a
  [[Agentes de IA - Arquitectura y Operacion]] (Capa 7), se referencia aqui como consumidor.

## 2. Modelo de datos objetivo (cambios sobre lo existente)

Sobre las entidades actuales (`FormDefinition`, `FormContainer`, `FormQuestion`,
`FormResponse`, `FormToken`, `FormFlowLink`, `WorkflowNodeForm`):

**`FormDefinition`** (+ propiedades de modulo y transaccion):
- `IsModule (bool)`, `ModuleMenuNodeId (Guid?)`, `ModuleIcon (string?)`.
- `IsTransactional (bool)`.
- `IdentityMode (enum None|NaturalKey|Sequence)`.
- `IdentitySourceFieldCode (string?)` (NaturalKey), `UniqueKeyFieldCodes (string[]?)`.
- `SequenceId (Guid?)` -> `TenantSequence` (Sequence).
- `ListColumnsJson`, `FilterFieldsJson` (config de la bandeja D6).

**`FormQuestion`** (+ propiedades de datos y calculo):
- `SourceKind (enum)`, `SourceRef (Guid?/string)`, `DisplayField`, `ValueField`,
  `FilterJson`, `AutofillMapJson`, `Presentation (autocomplete|dropdown|modal)`.
- `CalcExpression (string?)`, `Aggregate (enum)`.
- `Format (string?)`, `DefaultDynamic (enum?)`, `FieldVisibilityJson`.

**`FormResponse` -> se comporta como `form_record` cuando la definicion es transaccional:**
- `RecordNumber (string?)`, `Status (enum Draft|Confirmed|Voided)`, `TransactionDate`,
  `VoidedAt/VoidedBy/VoidReason`.
- Dimensiones: no requiere columnas nuevas por dimension (los ids viven en `data`), pero se
  crean **indices** sobre las claves lookup mas consultadas para la bandeja/BI.

**Nuevas piezas** (minimas, la mayoria reusa infra):
- `FormModulePermission` (o derivar policies del code, sin tabla nueva) - D1.
- Vinculo `FormDefinition.SequenceId -> TenantSequence` - D3 (la secuencia YA existe).
- (v2) `FormRecordLink` para maestro-detalle entre formularios - D7.

## 3. Consumo: confirmar un registro transaccional (secuencia)

Al **confirmar** (no en autosave de borrador), dentro de UNA transaccion:

```
1. Validar el registro en servidor (FormFieldValidator + reglas de campo activas).
   - Recalcular en servidor los campos con CalcExpression y los Aggregate (no confiar en el cliente).
2. Resolver identidad segun IdentityMode:
   - NaturalKey: leer el/los campos clave, validar unicidad por tenant (indice unico).
   - Sequence:   ISequenceService.Next(SequenceId[, entidad]) -> RecordNumber. Consumo real.
   - None:       sin numero.
3. Persistir FormResponse como Confirmed: RecordNumber, TransactionDate, data jsonb,
   Status=Confirmed. (Los campos lookup ya guardan los ids = dimensiones del hecho.)
4. Disparar reglas/acciones de post-guardado (RulesEngine): webhooks tipados, PDF, notif.
5. Si el formulario esta unido a un nodo BPMN (FormFlowLink Pending): completar el paso
   via IWorkflowEngine.CompleteStepAsync en la MISMA transaccion (comportamiento ya existente).
6. Auditoria + commit. Emitir evento SignalR (record.created) al grupo del tenant (bandeja en vivo).
```

Anular: `Status=Voided` + motivo + auditoria; **no** borra ni libera el consecutivo.

## 4. Reglas transversales (del proyecto, obligatorias)

- **Multi-tenant real** (`HasQueryFilter` + RLS): catalogos de lookup y registros SIEMPRE
  filtrados por tenant; imposible autocompletar con datos de otro cliente.
- **DAL dual (PG / SQL Server)** via `IEcorexDbContext`; la matriz dual es condicion de merge.
  El `jsonb` (PG) / `nvarchar(max)` con JSON (SQL Server) ya es el patron; los indices de
  dimension se generan en ambos.
- **Transaccion** para confirmar registro + consecutivo + reglas + (paso de flujo) + notif:
  todo o nada.
- **Nada de SQL concatenado**; los filtros de lookup y de bandeja se traducen a consultas
  parametrizadas / expresiones tipadas.
- **Formulas y acciones en sandbox tipado** (evaluador del RulesEngine, allow-list de verbos):
  se evita el RCE de facto por reflexion del legacy (ver
  [[Clases de Reglas del modulo Documental]]).
- **Auditoria + soft-delete + concurrencia optimista** (`Version`) en todo registro.
- **Permisos como policies** (`[Authorize(Policy=Form.{code}.*)]`), no chequeos ad-hoc.

Sigue en [[02 - UX y paneles de propiedades (Formularios avanzados)]] (como se configura) y
[[03 - Plan por olas y preguntas abiertas (Formularios avanzados)]] (en que orden y que
decidir).
