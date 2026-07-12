---
tipo: spec-ux
proyecto: Propiedades avanzadas del Constructor de Formularios
proposito: Como se configuran las propiedades avanzadas en el FormDesigner (propiedades del formulario + tab Datos/Calculo del campo), el GridDetail con totales y la bandeja del formulario-modulo con filtros.
---

# 02 - UX y paneles de propiedades (Formularios avanzados)

> El `FormDesigner.razor` ya tiene un panel de propiedades por campo con tabs
> **Diseno / Datos / Reglas** y un lienzo con paleta drag and drop. Esta spec dice **que
> agregar** a esos paneles (fiel a los tokens del prototipo `ECOREX.dc.html`), sin rehacer
> lo existente. La vista previa sigue siendo el `DynamicFormRenderer` real. Ver decisiones en
> [[01 - Arquitectura, decisiones y datos (Formularios avanzados)]].

## 1. Panel nuevo: Propiedades del FORMULARIO

Hoy el disenador edita cabecera (codigo, titulo, version) y el arbol. Se agrega una seccion
**"Propiedades del formulario"** (engranaje del topbar o tab de nivel formulario):

- **Es un modulo del sistema** (toggle `IsModule`): al activar, pide **grupo de menu**, **icono**
  y genera las policies `Form.{code}.Ver/Crear/Editar/Eliminar`. Muestra la ruta resultante
  `/m/{code}`.
- **Es transaccional** (toggle `IsTransactional`): al activar, despliega la config de identidad.
- **Identidad del registro** (`IdentityMode`):
  - **Ninguna** (default).
  - **Por dato (clave natural)**: selector del/los campo(s) clave (`IdentitySourceFieldCode`,
    `UniqueKeyFieldCodes`) + toggle "validar unicidad".
  - **Consecutivo del sistema**: selector/creador de **secuencia** con prefijo, longitud,
    reinicio (nunca/anual/mensual) y "numeracion por entidad" (una por sede/area). Vista previa
    del proximo numero (ej. `COT-2026-000124`).
- **Bandeja**: elegir las **columnas** del listado y los **campos de filtro** (D6).

> [!note] Regla de UX: la config transaccional NO aparece si el formulario no es transaccional.
> Progressive disclosure: cada toggle revela solo lo suyo, para no abrumar al configurador
> (persona Andres, ver [[02 - Historias del Configurador]]).

## 2. Panel de CAMPO: tab "Datos" (lookup / autocompletado)

En el tab **Datos** del campo seleccionado se agrega el bloque **Origen de datos**:

- **Origen** (`SourceKind`): `Opciones fijas` (como hoy) | `Contenedor de datos` | `Directorio
  (terceros)` | `Inventario (items)` | (extensible).
- Si es una fuente de datos:
  - **Fuente** (`SourceRef`): selector del contenedor o entidad.
  - **Mostrar / Guardar** (`DisplayField` / `ValueField`): que ve el usuario vs que id se guarda.
  - **Presentacion**: `Autocompletar` (typeahead) | `Lista` (dropdown) | `Buscador` (modal).
  - **Filtro** (`FilterJson`): condiciones del catalogo (ej. `EsCliente = true`). Siempre
    acotado al tenant.
  - **Autollenar campos** (`AutofillMapJson`): tabla de mapeo "campo de la fuente -> campo del
    formulario" (ej. `NIT -> nit`, `Direccion -> direccion`). Incluye campos dinamicos del
    tercero/item.

Ejemplo (captura de cliente): campo "Cliente" con Origen = Directorio, Presentacion =
Autocompletar, Autollenar: al elegir, se completan NIT, telefono y ciudad automaticamente.

## 3. Panel de CAMPO: bloque "Calculo" (formulas y formato)

En el tab **Datos** (o un tab **Calculo**) del campo:

- **Campo calculado** (`CalcExpression`): editor de expresion con ayuda de campos disponibles
  (`{cantidad} * {precio}`), preview del resultado con datos de ejemplo. Solo lectura en el
  render (el usuario no lo edita).
- **Formato / mascara** (`Format`): moneda, porcentaje, entero, telefono, documento, fecha.
- **Valor por defecto dinamico** (`DefaultDynamic`): Usuario actual, Hoy, Entidad del contexto,
  Proximo consecutivo (peek).
- **Visibilidad por rol** (`FieldVisibilityJson`): ver/editar por rol (campos sensibles).

## 4. GridDetail (tabla) con totales

El control `GridDetail` (ya funcional) suma:

- Por **columna**: tipo de dato, **columna calculada** (subtotal por fila) y **agregado**
  (`Aggregate`: sum/count/avg/min/max) que pinta una **fila de totales** al pie.
- **Roll-up**: el total de una columna alimenta un campo del encabezado (ej. `total_general`).
- Las columnas tambien pueden ser **lookup** (una columna "Item" que autocompleta desde
  Inventario y autollena "precio" en la fila).

Ejemplo (cotizacion): tabla de lineas con columnas Item (lookup) -> Precio (autollenado) x
Cantidad = Subtotal (calculado); fila de totales suma Subtotal; roll-up a `total` del encabezado.

## 5. Bandeja del formulario-modulo (listado + filtros + KPIs)

La ruta `/m/{code}` del formulario-modulo (D1/D6):

- **Barra de filtros dinamicos**: se construye sola a partir de `FilterFieldsJson` (los campos
  elegidos como filtro), + rango de fecha de transaccion + busqueda por numero/texto.
- **Grid de registros**: columnas de `ListColumnsJson` (numero, fecha, estado + campos), con
  acciones por fila (ver / editar / anular / imprimir) segun permiso.
- **KPIs**: tarjetas arriba (cantidad, suma de un campo monetario, % de crecimiento vs periodo
  anterior) - el usuario los describio como "indicadores de crecimiento de cada bolsa".
- **Export** a Excel/CSV; boton "Nuevo" abre el `DynamicFormRenderer` en modo Fill.
- **En vivo**: la bandeja se actualiza por SignalR cuando entra un registro (record.created).

## 6. Fidelidad visual

- Reusar los tokens del prototipo `ECOREX.dc.html` (`--surface`, `--ink`, `--t-*`, `--sh-*`)
  como hace el resto del modulo de formularios (CSS isolation `.razor.css` con fallback literal).
- La **vista previa** del disenador y el **render** real son el mismo `DynamicFormRenderer`
  (ya es asi): lo que se configura se ve igual al llenar.
- Los paneles nuevos siguen el patron de tabs y modales ya presentes en `FormDesigner.razor`
  (no se inventa un lenguaje visual nuevo).

## 7. Ejemplos de referencia (para aterrizar el diseno)

| Caso | IsModule | IsTransactional | Identidad | Lookups | Calculo |
|---|---|---|---|---|---|
| Cotizacion | Si | Si | Consecutivo (COT-AAAA-NNNNNN) | Cliente (Directorio), Item (Inventario) | Subtotales + total (roll-up) |
| Permiso / solicitud | Si | Si | Consecutivo | Solicitante (usuario) | Dias = fin - inicio |
| Ficha de cliente | Si | Si | Clave natural (NIT) | Ciudad (contenedor), Vendedor (usuario) | - |
| Encuesta simple | No | No | Ninguna | - | - |

Cada caso mapea a las banderas de la definicion (doc 01 seccion 2) sin codigo nuevo por caso.

Sigue en [[03 - Plan por olas y preguntas abiertas (Formularios avanzados)]].
