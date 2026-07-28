---
tipo: nota-desarrollo
capa: Capa 2 - Formularios dinamicos
proposito: Bitacora de necesidades de DESARROLLO del motor de formularios, detectadas al DISENAR formularios reales de clientes. La sesion de diseno anota aqui; la sesion de desarrollo las implementa.
estado: activo
---

# Notas para el programador - ECOREX Formularios

> Convencion: la **sesion de diseno** (que arma formularios reales sobre el motor) escribe aqui cada
> vez que el diseno pide algo que el motor todavia no cubre. La **sesion de desarrollo** toma estos
> apuntes, los implementa y marca el estado. Solo ASCII. Referencias de codigo con ruta relativa al
> repo `apps/backend/src/...`.
>
> Estados: `[ ] pendiente`  `[~] en curso`  `[x] hecho`  `[-] descartado`.
> Prioridad: **P1** bloquea el formulario / **P2** mejora fuerte / **P3** deseable.

---

## 2026-07-17 - Diseno de CONTACTO CLIENTE (SOLDARCO, FRM-00005)

Contexto: primer formulario real que se disena sobre el motor nuevo. Origen legacy
`M700_GEN.dbo.ENCUESTAS_MOV` (SOLDARCO). Cabezotes ya migrados a `form_definitions`; las preguntas se
construyen a partir del mapa de diseno. El diseno visual y el mapa de campos completo estan en el
artefacto de diseno entregado al usuario (maqueta + decisiones + mapa legacy->nuevo).

Decisiones de diseno ya cerradas con el usuario (NO requieren desarrollo, son configuracion):
- Consecutivo: el formulario es **transaccional** (`identity_mode = Sequence`); el numero lo asigna el
  motor de consecutivos y se muestra read-only. Ya soportado (el renderer pinta el chip de
  `FormResponse.RecordNumber`).
- Cliente (nombre/codigo/NIT): **texto libre** por ahora (sin lookup a Terceros).
- Personas de contacto: **GridDetail** (filas en el mismo registro).
- Valor: visible/obligatorio **solo si** "Concreto venta = Si".
- Fecha default = Hoy (`DefaultDynamic`); Asesor comercial default = usuario actual.

### Necesidades de desarrollo detectadas

**D1 [x] P2 - Control de HORA (Time).** HECHO 2026-07-17 (opcion A). Se agrego `Time` y `DateTime`
al enum (input `type=time`/`datetime-local`), validacion `ValidateTime`, entrada en la paleta y en
la lista de tipos elegibles. Verificado en Chrome: un campo Time se pinta como selector de hora.
El enum `Ecorex.Domain/Enums/FormControlType.cs` tiene `Date` pero NO tiene `Time` ni `DateTime`. El
formato viejo separa Fecha y Hora (ambas venian como tipo "Fecha" en el legacy, pero la 2a es una hora).
- Opcion A (recomendada): agregar `FormControlType.Time` (input `type=time`) + su rama en
  `Components/Shared/Forms/DynamicFormRenderer.razor` (hoy `Date` esta en el `case` ~linea 557) y en el
  `FormDesigner.razor` (paleta de controles).
- Opcion B: un solo `DateTime`. Menos fiel al formato viejo.
- Mientras no exista: el campo Hora queda FUERA del formulario (no se pone texto-mascara como parche,
  por decision del usuario).

**D2 [x] P2 - Consecutivo visible en BORRADOR.** HECHO 2026-07-17. Decision del usuario:
PREVISUALIZAR (no reservar, para no dejar huecos en la secuencia si se descarta un borrador). El
numero se sigue emitiendo al confirmar; en el borrador de un formulario transaccional el renderer
muestra el chip "N.o por asignar". Verificado en Chrome.
`FormResponse.RecordNumber` se asigna al **confirmar** el registro (es null en Draft; ver
`Domain/Entities/FormResponse.cs` y la ola F3). El renderer ya muestra el chip cuando existe
(`DynamicFormRenderer.razor` ~linea 54-56), pero durante el llenado de un registro nuevo aparece vacio.
El formato viejo muestra el consecutivo desde el inicio.
- Necesidad: **reservar/mostrar el numero al crear el borrador** (o pintar un placeholder tipo
  "se asignara al guardar"). Definir con negocio si el numero se reserva (y por tanto se puede
  "quemar" si se descarta el borrador) o si solo se previsualiza.

**D3 [x] P1 - Columnas tipo LISTA dentro del GridDetail.** HECHO 2026-07-17. Una columna del grid
declara ahora su tipo (`text`/`select`), sus opciones y si es `required`. El JSON de columna quedo
`{id,label,type,required,options:[{id,label}]}` (las viejas `[{id,label}]` siguen valiendo). El
renderer pinta `<select>` en las columnas lista; el disenador tiene editor por columna (nombre +
tipo + opciones + obligatoria); el servidor valida requerido/opcion por celda. Verificado en Chrome
con una columna Estado (Instalado/Pendiente) obligatoria.
La grilla de contactos necesita columnas Select con opciones fijas: "Medio contacto"
(MAIL / VISITA PRESENCIAL / TELEFONO / REDES) y "Otros/Resultado" (PQRS / Soporte / Oportunidad), ademas
de "obligatorio" por columna (Nombre persona y Descripcion son requeridas).
- `GridDetail` (ADR-0021) guarda columnas en `OptionsJson` como `[{id,label}]` y el valor es un arreglo
  de filas `[{colId:"valor"}]`. Hay que CONFIRMAR/EXTENDER que una columna pueda declarar su propio
  tipo de control (texto vs select) y su lista de opciones, y que valide "requerido" por columna.
- Es P1 porque la grilla es el corazon del formulario; si las columnas solo son texto, se pierde la
  parametrizacion de Medio/Resultado.

**D4 [x] P2 - Regla condicional "mostrar Valor si venta = Si".** HECHO 2026-07-17. El runtime YA
existia (dispatcher + estado de UI aplican hide/show/required); el verbo BLOQUEAR_CAMPO_XCONDICION
ya evaluaba la condicion con accion inversa. Faltaba la AUTORIA: el tab Reglas del constructor tiene
ahora un editor inline "si {campo} {=/!=/vacio/con valor} {valor} -> {mostrar/ocultar/obligatorio/
opcional} {campo}", que crea y vincula la regla sin salir al modulo de Reglas. Verificado en Chrome:
escribir el valor oculta el campo y cambiarlo lo vuelve a mostrar.
El motor ya tiene el vinculo `Domain/Entities/FormFieldRule.cs` (pregunta -> `Rule` del RulesEngine, con
acciones de UI ocultar/mostrar/set/required al cambiar el valor). Falta CONFIRMAR que el **FormDesigner**
permita configurar esto sin codigo: elegir campo disparador ("Concreto venta"), condicion (= "Si") y
accion (mostrar + hacer requerido "Valor"). Si el disenador aun no expone reglas de campo, esa es la
tarea.

### Puntos menores / a definir (no bloquean)
- Branding por formulario: el formato viejo tiene el logo de SOLDARCO en el encabezado. Hoy `Heading` es
  solo texto e `Image` es placeholder. Definir si el encabezado del formulario admite logo/imagen del
  tenant. P3.
- `VERSION` del legacy (V01/V0/0.0) no se migro (revision es numerico). Si se quiere conservar la etiqueta
  de version textual, definir donde. P3.

---

## 2026-07-17 - CONTACTO CLIENTE (FRM-00005) CONSTRUIDO en prod

Con D1-D4 ya desplegados en prod (redeploy desde `fase-0/clon-backbone`), la sesion de diseno armo el
formulario CONTACTO CLIENTE en SOLDARCO. Se hizo por SQL directo (13 preguntas + config transaccional +
regla), copiando las formas EXACTAS que produce el motor (verificadas en codigo: `FormGridCalculator`,
`BloquearCampoPorCondicionVerb`, `SaveFormQuestionRequest`). Estado: los 13 campos, la transaccionalidad
y la regla quedaron verificados en BD. Falta la verificacion VISUAL en el navegador (ver nota de login abajo).

Lo construido:
- Definicion **transaccional** (`is_transactional=true`, `identity_mode=Sequence`): el consecutivo lo emite
  el motor al confirmar (`FRM-00005-000001`, auto-crea la `tenant_sequence`).
- 13 campos: fecha (Date, default Hoy), **hora (Time, usa D1)**, asesor_comercial (Text, default usuario
  actual), nombre/codigo/nit (Text), perfil y estado_comercial (Select), sec_contactos (Heading),
  **contactos (GridDetail con columnas tipadas: 3 text + 2 select con opciones, usa D3)**, detalle_general
  (TextArea), concreto_venta (Select Si/No), valor (Number, formato moneda).
- **Regla D4**: `BLOQUEAR_CAMPO_XCONDICION` con `effect=show`, `operator=equals`, `value=si`,
  `sourceField=concreto_venta`, `targetField=valor`. El verbo es de DOBLE VIA (si cumple Show, si no Hide),
  asi que una sola regla basta. Se creo el 1er `rule_document` de SOLDARCO (RUL-001).

### Hallazgo nuevo para desarrollo

**D5 [ ] P3 - Reglas de campo NO se evaluan al CARGAR el formulario.**
El renderer (`DynamicFormRenderer.razor`) solo dispara `DispatchFieldRulesAsync` en el `OnFieldChanged`
(no hay pase inicial). Ademas la visibilidad es un OR: `_ruleUiState.IsHidden(fc) || q.IsHidden || rolHide`
(linea ~463), asi que un `is_hidden` estatico NO se puede revelar por regla. Consecuencia: el campo "valor"
arranca VISIBLE y solo se oculta cuando el usuario elige concreto_venta != "si". No queda oculto de entrada.
- Mejora deseable: evaluar las reglas de campo una vez al montar el formulario (con los valores por
  defecto) para fijar el estado inicial de UI. Asi "valor" nace oculto hasta que haya venta.

**Nota de QA - login por automatizacion: RESUELTO (causa raiz + como).**
Sintoma: por automatizacion (Chrome MCP) los clics/teclas sobre el login NO hacian submit (sin error).
CAUSA RAIZ: el login NO es un handler interactivo de Blazor; es un **POST HTTP nativo** (`<form
action="/auth/login" method="post">`), porque la cookie de auth debe fijarse en una respuesta HTTP real,
no por el circuito SignalR. Por eso clicar el boton sobre el circuito no envia nada. Ademas los inputs se
llenan con `@bind` (onchange), asi que setear `.value` por JS sin evento no vale.
SOLUCION que funciona (verificado, se entro como administracion@soldarco.com): fijar los valores con el
setter nativo + `dispatchEvent(new Event('change',{bubbles:true}))` y luego **`form.requestSubmit()`** sobre
el form de `/auth/login`. Tras el POST la sesion queda activa y el resto de la app (interactiva) va normal.
Para el plan de pruebas E2E: encapsular esto en un helper de login (o exponer un endpoint de sesion de test).
Nota aparte: el CONSTRUCTOR de formularios es una pagina Blazor Server PESADA; bajo automatizacion el
renderer se congela al capturar screenshot (no es bug funcional, es peso de pagina). Verificacion via texto
(get_page_text) o por BD es mas fiable en esa pantalla.

**Verificacion de FRM-00005 en la UI (2026-07-17):** con el login ya resuelto, el modulo Formularios de
SOLDARCO muestra la tarjeta CONTACTO CLIENTE (FORX-FRM-00005) con **13 campos** y **1 regla**, y el KPI
"1 Con reglas". Confirma que lo construido por SQL quedo bien y el motor lo reconoce.

---

## 2026-07-17 - SIMULADOR DE COTIZACIONES (SKY SYSTEM, codigo `COT`)

Origen: Excel `048. SKy System/Cotizador Formulario.xlsx`, hoja **SIMULADOR** (+ hojas BASE_PRODUCTOS
~1019 productos, BASE_CLIENTES ~38 clientes, SEGUIMIENTO_COTIZACIONES, FORMATO_COTIZACION).
FASE 1 (hecha): **diseno/estructura**. FASE 2 (pendiente): las formulas.

Construido en prod (definition `59a91ffe-01ef-4e28-9b3e-2b4ae9935612`): formulario **transaccional**
(consecutivo `COT-000001`, equivale al N. COT del Excel), 18 campos:
encabezado (fecha, cliente, IVA%), parametros del cliente (pasaje, horas parq., parq. x hora,
total parq. **calculado**, margen Sky), la **tabla `items` con 18 columnas**, los 5 totales
(alimentados por **rollup** desde la tabla) y observaciones.

### Que formulas del Excel YA funcionan y cuales NO

El motor de calculo (`FormExpressionEvaluator`) es un **sandbox estricto**: solo numeros, referencias
`{campo}` y `+ - * /` con parentesis y menos unario. **NO tiene funciones** (decision de diseno para
evitar el RCE del legacy). Resultado:

YA CONFIGURADAS (aritmetica pura, quedaron activas en el formulario):
- `precio_base = {costo}/{margen}`
- `subtotal = {cantidad}*{p_unitario}`
- `descuento = {subtotal}*{desc_pct}`
- `subt_desc = {subtotal}-{descuento}`
- `total = {subt_desc}+{iva}`
- `total_parq = {horas_parq}*{parq_hora}` (encabezado)
- Totales por `agg:"Sum"` + `rollup` a los campos del encabezado.

### Necesidades de desarrollo (fase 2 del simulador)

**D6 [ ] P1 - Lookup/autollenado en COLUMNA de tabla (el VLOOKUP del Excel).**
El corazon del simulador: se escribe el CODIGO y se autollenan producto, detalle, marca, proveedor,
costo y stock desde BASE_PRODUCTOS. Hoy `FormGridColumn.Kind` solo admite `text` o `select`; el lookup
con autollenado existe **solo a nivel de CAMPO** (`SourceKind` + `AutofillMapJson`), no de columna.
Sin esto el usuario teclea a mano las 6 columnas por fila.

**D7 [ ] P1 - Funciones en el motor de calculo.**
El Excel usa `IF`, `IFERROR`, `CEILING(x,1000)`, `ISNUMBER`, `SUMIF`, `TEXT`, `CHAR`. Minimo viable
para el simulador: **IF (condicional)** y **CEILING/ROUND a multiplo** (el P.UNITARIO redondea a miles).
Mantener el sandbox: agregar una allow-list de funciones puras, sin acceso al host.
Casos concretos que hoy no se pueden expresar:
- `p_unitario = CEILING(precio_base + pasaje + total_parq + mano_obra, 1000)`
- `subtotal = IF(cantidad > stock, "SIN STOCK", cantidad * p_unitario)`
- `iva = IF(producto_exento, 0, subt_desc * iva_pct)`

**D8 [ ] P2 - Una columna calculada no puede referenciar campos del ENCABEZADO.**
`FormGridCalculator.Recompute(rows, cols)` calcula **por fila**, con las columnas de esa fila. El
simulador necesita que la fila use el `iva_pct`, `pasaje` y `total_parq` del encabezado. Workaround
actual: el IVA de la columna quedo con el literal `0.19` (si cambian el IVA del encabezado, la columna
no se entera).

**D9 [ ] P2 - Agregado condicional (SUMIF).** El total del Excel excluye las filas "SIN STOCK"
(`SUMIF(N5:N24,"<>SIN STOCK",...)`). Hoy `agg` solo suma todo.

**D10 [ ] P3 - Generacion de texto/plantilla.** El Excel arma un mensaje de WhatsApp concatenando los
items y totales. Seria un "campo texto por plantilla" con placeholders.

**DATOS (no es codigo, es ETL):** los maestros NO van a contenedores genericos (correccion del usuario):
los **clientes van al Directorio General (Terceros)** y los **productos a Items de inventario**. El motor
ya soporta ambos como fuente (`FormSourceKind.Tercero` / `.Item`) con sus campos dinamicos. El lookup del
CLIENTE en el encabezado es configurable HOY (sin codigo) en cuanto existan los terceros.

### Decisiones de diseno cerradas con el usuario (2026-07-17, revision campo por campo)

1. **Costo**: el simulador usa el **COSTO SIN IVA** (col 8 de la base), no el costo con IVA. Es
   intencional; el precio base se calcula sobre ese.
2. **Columnas del lookup EDITABLES**: producto/detalle/marca/proveedor/costo/stock se autollenan como
   **snapshot** y el asesor puede ajustarlas en una cotizacion puntual (no es vinculo vivo).
3. **PASAJE: una sola vez por cotizacion**, NO por fila. El Excel lo sumaba dentro del precio unitario
   de cada item (5 productos = 5 pasajes): era un error. Consecuencia buena: la formula de la fila ya
   no necesita leer el encabezado para el pasaje/parqueadero.
4. **MARGEN en %**: se captura 20 (no el divisor 0,8). Formula activa
   `precio_base = {costo}/(1-{margen}/100)`. Idem el descuento: `{subtotal}*{desc_pct}/100`.
   OJO: en BASE_CLIENTES la columna MARGEN SKY esta **vacia en los 38 clientes** (y MANO DE OBRA
   tambien), asi que hoy el margen es un valor por defecto, no un dato del cliente.
5. **Datos faltantes del cliente = 0** (5 clientes sin TIPO PARQ, 2 sin pasaje): nunca bloquea.
6. **PARQ $ es de doble uso**: en clientes FIJO es el valor fijo; en X HORA es la tarifa por hora.
   Resuelto SIN codigo: regla condicional (muestra "Horas" solo si es X HORA) + horas=1 por defecto,
   de modo que `total_parq = horas * valor` sirve para ambos casos.
7. **SIN STOCK**: el renglon se **marca y NO suma** (como el Excel). Implementado como marca numerica
   `sin_stock = SI(cantidad > stock; 1; 0)` + agregado condicional, en vez del hack del Excel de meter
   el texto "SIN STOCK" en una columna numerica.
8. **IVA editable por cotizacion** (campo `iva_pct` del encabezado) -> exige que una formula de columna
   pueda leer el encabezado.
9. **Redondeo con multiplo como PARAMETRO** (`REDONDEAR.SUPERIOR(valor; multiplo)`), no fijo a 1000.
10. Fuera de alcance de la primera tanda: la plantilla del mensaje de WhatsApp.

### Estado del formulario COT (lo que YA calcula vs lo que espera codigo)

Activas hoy: `precio_base`, `subtotal`, `descuento`, `subt_desc`, `total`, `total_parq` y
`total_cotizacion = {tot_total}+{pasaje}+{total_parq}`, mas los 5 totales por rollup.
Se dejaron a proposito las versiones que FUNCIONAN de `p_unitario` (`{precio_base}+{mano_obra}`) e
`iva` (`{subt_desc}*0.19`): poner ya las formulas objetivo devolveria null y tumbaria toda la cadena.
La sesion de codigo debe cambiarlas al cerrar C2/C3:
```
p_unitario = REDONDEAR.SUPERIOR({precio_base}+{mano_obra}; 1000)
iva        = SI({exento_iva}=1; 0; {subt_desc}*{#iva_pct}/100)
```

### Capacidades a implementar (prompt entregado al usuario para la sesion de codigo)
- **C1** = D6 lookup/autollenado en columna de tabla (fuente Item; generico para Tercero/Contenedor).
- **C2** = D7 funciones en el sandbox: `SI`, comparaciones, `REDONDEAR.SUPERIOR/INFERIOR/REDONDEAR`
  con multiplo parametrico, y `MIN`/`MAX` deseables.
- **C3** = D8 una formula de columna puede leer un campo del ENCABEZADO (sintaxis propuesta `{#campo}`).
- **C4** = D9 agregado condicional (`aggWhen`) para excluir filas marcadas de los totales.
- **C5** = D11 **valor por defecto por columna** de tabla (cantidad=1, margen=20, desc=0). Hoy al
  añadir fila todas las celdas nacen vacias.

### ESTADO REAL tras revisar el codigo desplegado (2026-07-27)
- **C2 HECHO**: `FormExpressionEvaluator` es ahora un sandbox con funciones `SI`, `REDONDEAR`,
  `REDONDEAR.SUPERIOR`, `REDONDEAR.INFERIOR`, `MIN`, `MAX`, comparadores `> < >= <= = <>`, `SI` perezosa.
  Multiplo del redondeo como PARAMETRO. Allow-list cerrada, sandbox intacto.
- **C3 HECHO**: referencias al encabezado `{#campo}`; `FormGridCalculator.Compute/Recompute` reciben
  `headerValues`.
- **C4 HECHO**: `FormGridColumn.AggWhen` + `Compute` devuelve `ExcludedRowIndexes`. Ademas se agrego
  ancho por columna (`width`) como bonus.
- **C1 HECHO** (codigo, 2026-07-27): lookup por COLUMNA en `FormGridColumnLookup.cs`
  (`FormGridLookupConfig`+`FormGridColumnExtras`+parser), en paralelo a `FormGridCalculator`. JSON de
  columna: `"lookup":{"source":"Item","valueField":"sku","displayField":"name","autofill":{campoFuente:
  idColumnaDestino}}`. La celda guarda el `valueField` (SKU legible), no el id. Reusa `IFormLookupService`.
- **C5 HECHO**: `"default":"1"` por columna (`FormGridColumnExtras`), + `"stockCheck":{"against":"cantidad"}`.
- **DATOS (2026-07-27, sesion de datos - HECHO):** SKY SYSTEM tiene 43 terceros (clientes, OK) y el
  catalogo de items del cotizador ya quedo LISTO para el lookup. OJO: BASE_PRODUCTOS **tiene 11
  productos, no ~1019** (el "1019" es max_row-4; 1009 filas vacias). Los 11 items estan remapeados a lo
  que el simulador espera: `Price`=COSTO SIN IVA, campo `costo_con_iva`=COSTO, `exento_iva`="0"/"1"
  (Text), `proveedor`, marca (FK Brand) y stock en Bodega Central. Basura E2E limpia (0/0).
  (El catalogo es de 11 productos reales, no 1019: las demas filas del Excel estan vacias. Esta completo.)
- **Simulador COT `59a91ffe` TERMINADO por diseno (2026-07-27):** formulas objetivo activas (p_unitario
  con REDONDEAR.SUPERIOR, iva con SI + {#iva_pct}, aggWhen {sin_stock}=0 en los 5 totales); columna
  `codigo` con lookup a Item ya CONFIGURADO (valueField=sku, displayField=name; autofill:
  name->producto, description->detalle, price->costo, stock->stock, proveedor->proveedor,
  exento_iva->exento_iva); defaults cantidad=1/margen=20/desc=0; `card_layout`=Completo (~1600px).
  Verificado en BD. PENDIENTE MENOR: la columna `marca` NO se autollena porque
  `ItemLookupSource.DescribeFieldsAsync` no expone la Brand (FK nativa); publica id/sku/name/description/
  price/active/stock + dinamicos. Para autollenar marca: exponer `brand` en el adaptador [codigo] o
  guardarla como campo dinamico [datos]. No es calculo (solo se muestra); no bloquea.

---

### Extension del lookup de COLUMNA: marca + contenedor + presentacion (2026-07-27, sesion de CODIGO - HECHO)
Desplegado a prod (`fase-0/clon-backbone` @ `064ec36`, backup `ecorex-2026-07-27-1519.sql.gz`, sin migracion).

- **P2 Marca HECHO:** `ItemLookupSource` resuelve `Item.Brand.Name` y publica el campo **`brand`**
  ("Marca") en `DescribeFieldsAsync`, `ToItem`, la query y el `Row`. Con esto el simulador YA puede
  mapear `brand -> marca` en el autollenado (resuelve el "pendiente menor" de arriba).
- **P1 Lookup desde CONTENEDOR y las 3 fuentes HECHO:** el editor de columna del GridDetail
  (`FormDesigner`) ahora tiene el tipo "Lookup" con Fuente (Inventario/Directorio/Contenedor); al elegir
  Contenedor se escoge CUAL contenedor (sourceRef) y sus campos salen de `DescribeFieldsAsync`. Reusa
  `IFormLookupService` (Item/Tercero/DataContainer), sin logica duplicada. Campo clave, campo a mostrar
  y el mapa de autollenado se pueblan con los campos descritos.
- **P1 Presentacion lista/auto/modal HECHO:** `FormGridLookupConfig.Presentation` (JSON
  `"presentation":"list"|"autocomplete"|"modal"`, default autocomplete). `RenderGridLookupCell` pinta
  `<select>` con el catalogo en modo Lista y el type-ahead en Autocompletar. Fuente inaccesible o
  catalogo vacio -> cae a texto plano, no rompe. Selector de presentacion en el editor de columna.
- **P1 Editable HECHO:** la celda y las columnas autollenadas quedan editables (snapshot) en las 3
  fuentes y en modo Lista; la configuracion completa se edita desde el disenador sin tocar codigo.
- **Bug corregido:** `SaveGridColumnsAsync` del disenador ahora es LOSSLESS (preservaba solo los campos
  de `FormGridColumn`, borraria `lookup`/`default`/`stockCheck` al guardar una columna).
- **Tests:** `FormGridColumnLookupTests` (presentacion list/auto/modal, fuente Contenedor + sourceRef,
  mapeo `brand->marca`, autollenado por fila, default por columna) + `FormGridXlsxSmokeTests`. 15/15 verdes.

---

## 2026-07-27 - Alineacion de cajas en una fila (polish de render)

**DA1 [ ] P3 - Reservar la linea del SUBTITULO/caption aunque este vacia.**
Sintoma: en una fila con varios campos, si unos tienen `caption` (subtitulo) y otros no, las etiquetas
de los que SI lo tienen ocupan una linea mas y sus `<input>` quedan mas abajo -> las cajas de la fila no
se alinean (visto por el usuario en el SIMULADOR, seccion "Parametros del cliente").
- Workaround de DISENO aplicado (sin codigo): darle un caption corto a todos los campos de la fila para
  que todos midan igual. Es fragil (si el usuario quita un caption vuelve a desalinear) y manual por form.
- Arreglo REAL (renderer, `DynamicFormRenderer.razor` + su CSS): que el bloque etiqueta+subtitulo tenga
  altura consistente aunque el caption sea null (reservar la 2a linea, o alinear los inputs al fondo de la
  celda con `align-items:end` en la fila). Asi TODOS los formularios se alinean solos sin importar que
  captions ponga el usuario -> respeta la norma "el usuario ajusta el diseno" sin que se rompa la
  alineacion. Aplica a la vista de llenar (/f, /m y vista previa).

### Formato numerico por columna + nombre/clave (SKU) en resultados del lookup (2026-07-27, sesion de CODIGO - HECHO)
Desplegado a prod (`fase-0/clon-backbone` @ `67e4d3d`, backup `ecorex-2026-07-27-1902.sql.gz`, sin migracion).

- **Formato numerico por columna del GridDetail (configurable):** `FormGridColumn.Format`
  (currency/integer/decimal/percent) en `options_json`, reusando `FormatValue`. Selector "Formato
  numerico" en el editor de columna. Se aplica a celdas calculadas, editables (muestra formateado y al
  salir guarda el numero CRUDO, asi el calculo no se rompe) y la fila de totales. `format` entra en
  `GridCoreKeys` para el guardado lossless. Estilo InvariantCulture ($ 1,234), como el resto de la app.
- **Nombre + clave (SKU) en los resultados del lookup (campo y columna, 3 fuentes):** cada resultado
  muestra el Display + la clave atenuada (monospace) a la derecha, ej. "IMPRESORA HP · IMP1". No se
  repite si esta vacia, si es el id crudo o si coincide con el Display. La clave sale del valueField
  (`KeyOf` en columna); se asegura que el valueField viaje siempre en `Fields` (LookupFields del campo
  y `FormGridLookupConfig.Fields`). Nuevo `subLabel` opcional por columna para elegir el campo
  secundario (default: la clave), editable en el disenador. No se toco la busqueda (`ItemLookupSource`).
- **Fix previo (mismo dia):** el panel de autocompletado de la celda-lookup se portaba mal dentro del
  modal (position:fixed anclado a un ancestro con backdrop-filter/transform) y lo recortaba el scroller;
  se PORTA a <body> y se ancla al input por id (data-lkanchor). Commit `4b0b4fc`.
- Tests: formato por columna (`FormGridCalculatorTests`) y `subLabel` (`FormGridColumnLookupTests`). 32 en total, verdes.
