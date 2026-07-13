---
tipo: registro-esquema
proyecto: Propiedades avanzadas del Constructor de Formularios
proposito: Bitacora de cada cambio de esquema (tablas/columnas) que el agente del worktree de formularios va aplicando en su BD local de trabajo, para que la SESION PRINCIPAL lo replique en la remota (prod) cuando pueda. Handoff ASINCRONO (la principal esta ocupada).
---

# 04 - Registro de tablas y cambios de esquema (Formularios avanzados)

> [!important] PARA LA SESION PRINCIPAL - COMO REPLICAR EN PROD
> El agente del worktree de formularios genera y aplica sus migraciones en una BD LOCAL de
> trabajo (`ecorex_forms`). NO toca la remota/prod. Este documento lista cada cambio para que
> la sesion principal lo aplique en prod cuando pueda. **Regla de oro:** el agente del worktree
> es el UNICO dueno de estas migraciones; la sesion principal NO debe generar otra migracion
> para las mismas columnas: solo trae la del worktree (rebase / cherry-pick de la rama
> `claude/briefing-worktree-formularios-f50017`) y la aplica a prod. Dos migraciones para el
> mismo cambio = snapshots divergentes + conflicto de git.
>
> Protocolo completo: ver seccion D de [[03 - Plan por olas y preguntas abiertas (Formularios avanzados)]].

## Leyenda de estado

- **Aplicada (local)**: ya corrio en la BD de trabajo del worktree (`ecorex_forms`, Postgres local 5442).
- **Pendiente (prod)**: falta que la sesion principal la aplique en la remota (`10.0.0.3`).
- **Reconciliada**: aplicada en prod y confirmada; el worktree ya trabaja sobre prod si aplica.

---

## OLA F1 - Lookups / autocompletado

### Cambio F1.1 - `form_questions`: propiedades de origen de datos (lookup)

| Metadato | Valor |
|---|---|
| Fecha | 2026-07-12 |
| Tabla | `form_questions` (EXISTENTE - solo se agregan columnas) |
| Naturaleza | **Aditiva** (no altera/borra nada; cero riesgo para la sesion principal) |
| Migracion PG | `20260712154152_AddFormLookupFields` (proyecto `Ecorex.Infrastructure`) |
| Migracion SQL Server | `20260712154251_AddFormLookupFields` (proyecto `Ecorex.Infrastructure.SqlServer`) |
| Rama | `claude/briefing-worktree-formularios-f50017` |
| Estado | Aplicada (local) / **Pendiente (prod)** |

**Columnas nuevas:**

| Columna | Tipo PG | Tipo SQL Server | Null | Default |
|---|---|---|---|---|
| `source_kind` | varchar(40) | nvarchar(40) | no | `'Options'` |
| `source_ref` | varchar(200) | nvarchar(200) | si | - |
| `display_field` | varchar(120) | nvarchar(120) | si | - |
| `value_field` | varchar(120) | nvarchar(120) | si | - |
| `filter_json` | jsonb | nvarchar(max) | si | - |
| `autofill_map_json` | jsonb | nvarchar(max) | si | - |
| `presentation` | varchar(40) | nvarchar(40) | no | `'Autocomplete'` |

**Script SQL (PostgreSQL, lo que corre en prod):**

```sql
ALTER TABLE form_questions ADD autofill_map_json  jsonb;
ALTER TABLE form_questions ADD display_field       character varying(120);
ALTER TABLE form_questions ADD filter_json         jsonb;
ALTER TABLE form_questions ADD presentation        character varying(40)  NOT NULL DEFAULT 'Autocomplete';
ALTER TABLE form_questions ADD source_kind         character varying(40)  NOT NULL DEFAULT 'Options';
ALTER TABLE form_questions ADD source_ref          character varying(200);
ALTER TABLE form_questions ADD value_field         character varying(120);
```

> En prod se aplica via EF (`dotnet ef database update` con la migracion `AddFormLookupFields`),
> NO con el SQL a mano, para que `__EFMigrationsHistory` quede consistente. El SQL de arriba es
> solo referencia de lo que hace la migracion.

**Enums nuevos** (persistidos como string, ya registrados en `ConfigureConventions` de `EcorexDbContext`):
- `FormSourceKind` = `Options | DataContainer | Tercero | Item`
- `FormFieldPresentation` = `Autocomplete | Dropdown | Modal`

**Para que se usa:** habilita que un campo de formulario se llene desde una tabla de datos del
tenant (Directorio / Inventario / Contenedor de datos) con autollenado por copia. `source_kind` +
`source_ref` dicen de donde sale el catalogo; `display_field`/`value_field` que se muestra vs que
id se guarda; `filter_json` acota el catalogo (siempre por tenant); `autofill_map_json` copia
campos dependientes al elegir (ej. cliente -> NIT/ciudad); `presentation` = autocompletar/lista/modal.

---

## OLA F2 - Calculo y agregacion

### Cambio F2.1 - `form_questions`: campo calculado + agregado

| Metadato | Valor |
|---|---|
| Fecha | 2026-07-12 |
| Tabla | `form_questions` (EXISTENTE - solo se agregan columnas) |
| Naturaleza | **Aditiva** (cero riesgo para la sesion principal) |
| Migracion PG | `20260712202108_AddFormCalcFields` (proyecto `Ecorex.Infrastructure`) |
| Migracion SQL Server | `AddFormCalcFields` (proyecto `Ecorex.Infrastructure.SqlServer`) |
| Rama | `claude/briefing-worktree-formularios-f50017` |
| Estado | Aplicada (local) / **Pendiente (prod)** |

**Columnas nuevas:**

| Columna | Tipo PG | Tipo SQL Server | Null | Default |
|---|---|---|---|---|
| `calc_expression` | varchar(1000) | nvarchar(1000) | si | - |
| `aggregate` | varchar(40) | nvarchar(40) | no | `'None'` |

**Enum nuevo** (persistido como string, en `ConfigureConventions`): `FormAggregate` = `None | Sum | Count | Avg | Min | Max`.

**Para que se usa:** `calc_expression` es la formula de un campo calculado (aritmetica/condicional
sobre otros campos, evaluada en sandbox tipado); `aggregate` es el total de una columna de GridDetail
(fila de totales + roll-up al encabezado). Ambas de la ola F2 (doc 01 D5).

---

## OLA F3 - Transaccionalidad (registro: identidad, estado, fecha)

### Cambio F3.1 - `form_definitions` + `form_responses`: campos transaccionales

| Metadato | Valor |
|---|---|
| Fecha | 2026-07-12 |
| Tablas | `form_definitions` y `form_responses` (EXISTENTES - solo columnas nuevas) |
| Naturaleza | **Aditiva** (cero riesgo para la sesion principal) |
| Migracion PG | `20260712213148_AddFormTransactional` (`Ecorex.Infrastructure`) |
| Migracion SQL Server | `AddFormTransactional` (`Ecorex.Infrastructure.SqlServer`) |
| Rama | `claude/briefing-worktree-formularios-f50017` |
| Estado | Aplicada (local) / **Pendiente (prod)** |

**`form_definitions` (5 columnas):**

| Columna | Tipo PG | Tipo SQL Server | Null | Default |
|---|---|---|---|---|
| `is_transactional` | boolean | bit | no | `false` |
| `identity_mode` | varchar(40) | nvarchar(40) | no | `'None'` |
| `identity_source_field_code` | varchar(60) | nvarchar(60) | si | - |
| `unique_key_fields_json` | jsonb | nvarchar(max) | si | - |
| `sequence_id` | uuid | uniqueidentifier | si | - |

**`form_responses` (6 columnas):**

| Columna | Tipo PG | Tipo SQL Server | Null | Default |
|---|---|---|---|---|
| `record_number` | varchar(100) | nvarchar(100) | si | - |
| `record_status` | varchar(40) | nvarchar(40) | no | `'Draft'` |
| `transaction_date` | timestamptz | datetimeoffset | si | - |
| `voided_at` | timestamptz | datetimeoffset | si | - |
| `voided_by_tenant_user_id` | uuid | uniqueidentifier | si | - |
| `void_reason` | varchar(500) | nvarchar(500) | si | - |

**Indice nuevo:** unico filtrado sobre `form_responses (tenant_id, definition_id, record_number)` donde
`record_number IS NOT NULL` (no repetir numero de registro por definicion).

**Enums nuevos** (string, en `ConfigureConventions`):
- `FormIdentityMode` = `None | NaturalKey | Sequence`
- `FormRecordStatus` = `Draft | Confirmed | Voided`

**Para que se usa:** F3 vuelve el envio confirmado un REGISTRO con numero (consecutivo via
`ISequenceService`/`TenantSequence`, o clave natural), estado (Draft/Confirmed/Voided) y fecha de
transaccion. `RecordStatus` es INDEPENDIENTE del `status` (Draft/Submitted, ciclo de envio).
`sequence_id` es referencia logica a `TenantSequence` (sin FK fisica, se resuelve en servicio).

> DECISION a validar con el usuario: se agrego un ciclo transaccional aparte (`record_status`) en
> vez de extender el `status` existente (Draft/Submitted), para no tocar el flujo de envio actual.

---

## OLA F4 - Formulario como modulo

### Cambio F4.1 - `form_definitions`: modulo + bandeja

| Metadato | Valor |
|---|---|
| Fecha | 2026-07-13 |
| Tabla | `form_definitions` (EXISTENTE - solo columnas) |
| Naturaleza | **Aditiva** |
| Migracion PG | `20260713121447_AddFormModule` (`Ecorex.Infrastructure`) |
| Migracion SQL Server | `AddFormModule` (`Ecorex.Infrastructure.SqlServer`) |
| Estado | Aplicada (local) / **Pendiente (prod)** |

**Columnas nuevas:** `is_module` (bool, def false), `module_menu_node_id` (uuid null),
`module_icon` (varchar(60) null), `list_columns_json` (jsonb/nvarchar(max) null),
`filter_fields_json` (jsonb/nvarchar(max) null).

**Para que:** promover un formulario a modulo del sistema. `module_menu_node_id` guarda el nodo de
menu generado; **el usuario elige DONDE colgarlo** (vista + grupo padre) al promover, reusando
`MenuConfigService.CreateNodeAsync` (Kind=Item, Route=/m/{code}). La bandeja usa list/filter json.

---

## OLA F5 - Maestro-detalle entre formularios

### Cambio F5.1 - `form_record_links` (TABLA NUEVA) + `form_questions.subform_definition_id`

| Metadato | Valor |
|---|---|
| Fecha | 2026-07-13 |
| Migracion PG | `20260713140004_AddFormRecordLink` (`Ecorex.Infrastructure`) |
| Migracion SQL Server | `AddFormRecordLink` (`Ecorex.Infrastructure.SqlServer`) |
| Naturaleza | **1 tabla nueva + 1 columna aditiva** |
| Estado | Aplicada (local) / **Pendiente (prod)** |

**Tabla nueva `form_record_links`** (TenantEntity): `id`, `tenant_id`, `parent_response_id` (FK
form_responses, Restrict), `parent_field_code` (varchar(60)), `child_response_id` (FK form_responses,
Restrict), `sort_order`, `created_at`+auditoria. Indice unico (parent_response_id, parent_field_code,
child_response_id).

**Columna nueva `form_questions.subform_definition_id`** (uuid null): la definicion HIJA que embebe un
campo Subform.

**Enum**: `FormControlType` += `Subform` (persistido como string; seguro al final).

**Para que:** el campo Subform del formulario padre agrupa N registros HIJOS (respuestas de otra
definicion) enlazados por `form_record_links`. A diferencia del GridDetail (filas en el jsonb del
padre), cada hijo es un `FormResponse` propio, reportable aparte.

---

Relacionado: [[03 - Plan por olas y preguntas abiertas (Formularios avanzados)]],
[[01 - Arquitectura, decisiones y datos (Formularios avanzados)]].

---

## F6 mascaras + captura Tier 2 (2026-07-13): SIN cambio de esquema

Los 2 pendientes de F6 (mascaras de entrada phone/document y captura Tier 2 real de firma/GPS/archivo)
**NO agregan tablas ni columnas**. Reutilizan columnas F6 ya registradas antes:

- **Mascara**: usa `form_questions.format` (valores `phone` y `document`, ademas de currency/percent/
  integer/decimal). Es presentacion; el valor guardado en `form_responses.data` queda crudo.
- **Captura Tier 2**: firma/GPS/archivo se guardan INLINE en `form_responses.data` (jsonb) como el resto
  de campos: firma y archivo como `data:...;base64,...` (tope 1 MB), GPS como texto "lat, lng". No hay
  tabla de adjuntos ni bucket; el object storage para adjuntos grandes es un incremento posterior.

**Implicacion para la sesion principal:** al desplegar F6 a prod NO hay migracion nueva por estos 2 items;
basta el codigo (renderer + `wwwroot/js/form-capture.js` + `App.razor`). Las columnas F6 (`format`,
`default_dynamic`, `field_visibility_json`) ya estan en el registro previo de este doc.
