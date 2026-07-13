---
tipo: plan-construccion
proyecto: Propiedades avanzadas del Constructor de Formularios
proposito: Backlog por olas con criterios de aceptacion + las decisiones a cerrar con el usuario antes de delegar a un sub-agente.
---

# 03 - Plan por olas y preguntas abiertas (Formularios avanzados)

## ESTADO GENERAL (actualizado 2026-07-13)

> [!success] EN CONSTRUCCION
> - **F1 (lookups/autollenado): HECHO** y verificado en navegador (commit `97d855d`).
> - **F2 (calculo): HECHO** - campos calculados + GridDetail (totales, roll-up); evaluador tipado
>   cliente+servidor. Verificado.
> - **F3 (transaccionalidad): HECHO** - confirmar/anular + identidad + panel de config + boton Anular;
>   verificado (FRM-021-000001). Pendiente fino: cierre por evento (firma), indices de dimension.
> - **F4 (formulario-modulo): HECHO** - promover a modulo con **colocacion dinamica en el menu** (vista +
>   grupo) + bandeja con **columnas/filtros configurables**, KPIs, **export CSV+Excel**, **en vivo por
>   SignalR**, aplanado BI y policy por visibilidad de menu. Verificado en navegador. (Refinamiento
>   opcional: policies tipadas `Form.{code}.*`.)
> - **F5 (maestro-detalle): HECHO** - campo Subform + `FormRecordLink`; el hijo se llena ANIDADO y queda
>   enlazado; **configurable en el designer** (paleta + selector de formulario hijo). Verificado.
> - **F6 (transversales): HECHO** - defaults dinamicos (Today/CurrentUser), formato (moneda/%/entero),
>   permisos por campo (ocultar/solo-lectura por rol), mascaras de entrada (phone/document) y captura
>   Tier 2 real (firma/GPS/archivo->data-URI inline) HECHOS y verificados E2E via Chrome (FRM-022).
>   Diferido (integracion grande, NO construido): PDF/plantilla, webhooks/botones-con-reglas, object storage.
>
> Migraciones F1..F5 aplicadas en la BD local del worktree; **pendiente aplicarlas a prod**
> (ver [[04 - Registro de tablas y cambios de esquema (Formularios avanzados)]]). F6 no anade esquema.

> [!todo] QUE FALTA DE VERDAD (backlog tras cerrar F1..F6) - inventario unico 2026-07-13
> El **nucleo** de las 6 olas esta HECHO y verificado. Lo que resta son piezas de integracion o
> refinamientos, ordenados por tipo:
>
> **A) Despliegue (no es codigo nuevo):**
> - [ ] Aplicar a **prod** las migraciones F1..F5 (todo el esquema vive en la BD local del worktree).
>       F6 (mascaras + Tier 2) NO lleva migracion: solo codigo.
>
> **B) En pausa por decision del usuario (NO construir hasta su senal):**
> - [ ] **Botones con reglas de accion + integraciones** (Button -> FormFieldRule -> verbo tipado del
>       RulesEngine -> Siigo/WhatsApp/correo). El usuario analiza la documentacion antes de delegarlo.
> - [ ] **Impresion / PDF con plantilla** (requiere renderer PDF + object storage).
>
> **C) Incrementos de infraestructura (grandes, sin bloquear el nucleo):**
> - [ ] **Object storage** para adjuntos grandes (hoy Tier 2 va inline en jsonb con tope de 1 MB).
> - [ ] Captura real de **codigo de barras** y **audio** (hoy placeholder "proximamente").
>
> **D) Refinamientos finos (mejoran, no bloquean):**
> - [ ] **KPIs configurables** por el usuario en la bandeja del modulo (hoy fijos: cantidad + % crecimiento).
> - [ ] **Cierre por evento** parametrizable (p.ej. agregar una firma confirma/cierra el registro) - pregunta 3.
> - [ ] **Indices de dimension** sobre las claves lookup mas consultadas (PG + SQL Server) para BI.
> - [ ] **Policies tipadas** `Form.{code}.*` (hoy el acceso a la bandeja es por visibilidad del nodo de menu).
> - [ ] **Test de aislamiento cross-tenant del lookup** en Integration.Tests (Testcontainers, dual) +
>       adaptadores Item/DataContainer ejercitados en navegador (F1 dejo esto pendiente).

## A. Plan por olas (orden recomendado por valor / riesgo)

Se recomienda este orden porque F1 y F2 son alto valor y **reusan** infra existente
(contenedor de datos, directorio, inventario, motor de reglas), mientras F3/F4 introducen el
concepto de registro/modulo y conviene montarlos sobre F1/F2 ya firmes.

### F1 - Lookups / autocompletado (datos con dominio del tenant)
Objetivo: un campo se llena desde una tabla de datos del tenant (cliente, item, lista) con
autollenado de campos dependientes.
- [x] `FormQuestion` gana `SourceKind/SourceRef/DisplayField/ValueField/FilterJson/AutofillMapJson/Presentation`. (migracion dual `AddFormLookupFields`; ver doc 04)
- [x] Servicio de busqueda server-side, parametrizado y paginado, sobre `DataContainerService`,
      `TerceroService` e `ItemService` (filtro por tenant garantizado). (`IFormLookupService` + 3 adaptadores en `Application/Forms/Lookups`)
- [x] `DynamicFormRenderer`: control de autocompletar/lista/buscador; al elegir, aplica el
      autollenado (incluye campos dinamicos de `TerceroFieldService`/`ItemFieldService`).
- [x] `FormDesigner`: bloque "Origen de datos" en el tab Datos (doc 02 seccion 2).
- Aceptacion: elegir un cliente autocompletando trae su NIT/direccion; el valor guardado es el
  id; imposible ver datos de otro tenant; test de aislamiento cross-tenant del lookup.
  > VERIFICADO 2026-07-12 en navegador (BD local `ecorex_forms`, copia de prod): campo Cliente
  > (Directorio, Autocompletar) -> escribir "a" trae 5 terceros reales -> elegir "ANDINA S.A.S"
  > autollena NIT=901.111.222 y Ciudad=Bogota. Designer muestra Origen/Presentacion/Mostrar
  > (con fichas dinamicas)/Filtro/mapa de autollenado. Unit del dispatcher verde (4/4). PENDIENTE:
  > test de aislamiento cross-tenant en Integration.Tests (Testcontainers, dual) + adaptadores Item/DataContainer en navegador.

### F2 - Calculo y agregacion (formulas, totales de tabla)
Objetivo: campos calculados, totales de columna en GridDetail y roll-up al encabezado.
- [x] `FormQuestion.CalcExpression` + `Aggregate`; evaluador de expresiones **tipado/sandbox**
      (`FormExpressionEvaluator`, allow-list; 16 unit tests). Cliente (renderer) + servidor. (migracion `AddFormCalcFields`, doc 04)
- [x] GridDetail: columna calculada por fila + fila de totales + roll-up a campo del encabezado.
      (helper compartido `FormGridCalculator` cliente+servidor; 9 unit tests; VERIFICADO en navegador:
      2x1500 + 3x1000 -> subtotales 3000/3000, fila de totales 6000, roll-up "Total general"=6000)
- [x] `FormFieldValidator`/`FormResponseService`: recalculo en servidor al guardar (no confiar
      en el cliente para montos). (recomputo en `FormResponseService.SaveAsync`, descarta el valor del cliente)
  > VERIFICADO 2026-07-12 en navegador (`ecorex_forms`): Subtotal = `{cantidad} * {precio_item}` ->
  > 3 x 540000 = 1620000 en vivo, solo lectura. Encadena F1->F2 (elegir item autollena precio -> subtotal).
- Aceptacion: cotizacion con lineas item x cantidad = subtotal, total = suma; si el cliente
  manipula el total, el servidor lo corrige; sin expresiones arbitrarias (allow-list).

### F3 - Transaccionalidad (registro: identidad, estado, fecha)
Objetivo: el envio confirmado se vuelve un registro/hecho con numero, estado y fecha.
- [x] `FormDefinition`: `IsTransactional`, `IdentityMode`, `IdentitySourceFieldCode`,
      `UniqueKeyFieldsJson`, `SequenceId`. (migracion `AddFormTransactional`, doc 04)
- [x] `FormResponse`: `RecordNumber`, `RecordStatus(Draft/Confirmed/Voided)` (APARTE del `Status`
      de envio, decision del usuario), `TransactionDate`, `VoidedAt/By/Reason`. Indice unico filtrado por `record_number`.
- [x] Confirmar: consume `ISequenceService` (Sequence) o valida unicidad (NaturalKey); identidad
      resuelta antes de la tx; anular (`VoidAsync`) no libera el numero; idempotente.
      VERIFICADO en navegador: FRM-021 (Sequence) -> Enviar -> **FRM-021-000001**, Confirmed, tx_date, secuencia consumida.
- [ ] Indices de dimension sobre las claves lookup mas consultadas (PG + SQL Server). (para BI, F4/pendiente)
- [x] UI de configuracion: panel "Propiedades del formulario" en el designer (boton Propiedades ->
      IsTransactional + modo de identidad + campo clave) + boton Anular en el renderer. VERIFICADO en
      navegador (lee Sequence, escribe NaturalKey campo=nit).
- Aceptacion: guardar una cotizacion consume COT-2026-000124; reintento no duplica; anular deja
  el registro Voided con motivo; unicidad de clave natural rechaza duplicados.

### F4 - Formulario como modulo (menu, permisos, bandeja, filtros, KPIs)
Objetivo: promover un formulario a modulo con su bandeja consultable.
- [x] `FormDefinition`: `IsModule/ModuleMenuNodeId/ModuleIcon/ListColumnsJson/FilterFieldsJson`. (migracion `AddFormModule`, doc 04)
- [x] Generacion de nodo de menu (reusa `MenuConfigService.CreateNodeAsync`, Kind=Item, Route=/m/{code})
      **EN LA VISTA + GRUPO que elige el usuario** (colocacion dinamica) + policy: la bandeja solo es
      accesible si el nodo esta VISIBLE en el menu del usuario (reusa permisos de menu). VERIFICADO.
      > Nota: policies tipadas `Form.{code}.*` con provider dinamico quedan como refinamiento (hoy el
      > control de acceso es la visibilidad del nodo de menu, que ya es por permiso).
- [x] Ruta `/m/{code}`: bandeja con grid + **columnas/filtros CONFIGURABLES** (checkboxes en el designer) +
      **KPIs** + **export CSV y Excel (.xlsx ClosedXML)** + **EN VIVO por SignalR**. VERIFICADO en navegador
      (columnas NIT/Ciudad con datos reales; filtro Ciudad "bog"->2; Excel exporta; bandeja 4->5 en vivo).
- [x] Vista aplanada (`data` + dimensiones) para BI/Power BI. (los registros exponen `Fields` y el export
      CSV/Excel incluye las columnas de datos configuradas = aplanado listo para BI).
- Aceptacion: un formulario marcado como modulo aparece en el menu segun permiso; su bandeja
  filtra por sus campos; los KPIs muestran cantidad y % de crecimiento; export a Excel funciona.
  > VERIFICADO 2026-07-13 (COMPLETO): FRM-021 promovido -> aparece en el menu bajo Automatizacion ->
  > `/m/FRM-021` lista el registro FRM-021-000001 (Confirmado); columnas/filtros configurables, KPIs,
  > export CSV+Excel y bandeja en vivo por SignalR verificados. **Refinamiento pendiente**: KPIs
  > CONFIGURABLES por el usuario (hoy fijos: cantidad + % crecimiento; ver pregunta abierta 11).

### F5 - Maestro-detalle entre formularios  **HECHO (nucleo)**
Objetivo: el detalle son registros de otro formulario (no solo GridDetail embebido).
- [x] Campo `Subform` (`FormControlType.Subform` + `FormQuestion.SubformDefinitionId`) + tabla
      `FormRecordLink` (padre-hijo). Servicio `ListChildren/AddChild/UnlinkChild`. Renderer: el campo
      lista los hijos + "Agregar" -> formulario hijo ANIDADO (DynamicFormRenderer recursivo) -> al
      enviar queda enlazado. Migracion dual `AddFormRecordLink` (doc 04).
- Aceptacion: una cotizacion referencia N lineas que son registros hijos reportables aparte.
  > VERIFICADO 2026-07-13 en navegador: FRM-021 (padre) con Subform -> hijo FRM-002; Agregar -> form
  > FRM-002 anidado (Bodega/Fecha) -> Enviar -> hijo FRM-002-000001 (Confirmado) enlazado; cada hijo es un
  > FormResponse propio (reportable aparte). Campo Subform CREABLE/CONFIGURABLE en el designer (paleta
  > "Subformulario" + selector de formulario hijo en el tab Datos). VERIFICADO 2026-07-13.

### F6 - Transversales (defaults dinamicos, mascaras, PDF, webhooks, permisos de campo, Tier 2)  **HECHO**
- [x] Defaults dinamicos (`Today`/`CurrentUser`) + **formato** (moneda/%/entero) + **permisos por campo**
      (`FieldVisibilityJson`: ocultar/solo-lectura por rol) HECHOS (schema + renderer + designer).
      VERIFICADO: Fecha Today -> hoy; subtotal currency -> "$ 1,620,000"; NIT hide=[Owner] no se pinta;
      Ciudad readonly=[Owner] queda deshabilitado.
- [x] **Mascaras de ENTRADA** (`DynamicFormRenderer.MaskInput`): en texto reformatea al perder foco.
      `phone`->(300) 123-4567; `document`->900,123,456. El valor GUARDADO queda crudo (mascara solo
      presentacion). Opciones agregadas al dropdown Formato del designer. VERIFICADO E2E Chrome (FRM-022).
- [x] **Captura real Tier 2** (firma en canvas->dataURL PNG, GPS via geolocation, archivo/foto->data-URI
      inline con tope 1 MB): `form-capture.js` + fragments en el renderer + estilos. VERIFICADO E2E Chrome
      (FRM-022, rol Advisor): firma (4358 chars), GPS (cableado OK, permiso denegado en sandbox), archivo
      PNG con preview; todo persiste en `form_responses.data`. **Object storage para adjuntos grandes = diferido.**
- [x] Captura Tier 2 (firma, GPS, foto/archivo): HECHO e inline (data-URI, tope 1 MB). VERIFICADO Chrome.
      **Diferido**: `barcode` y `audio` (hoy placeholder) + object storage para adjuntos grandes.
- [ ] **DIFERIDO** - Impresion/PDF con plantilla (object storage) + webhooks tipados al confirmar (allow-list,
      no reflexion) hacia integraciones (Siigo, WhatsApp HSM, correo).
      > DECISION/NOTA (usuario 2026-07-13): el usuario quiere BOTONES en el formulario con reglas de accion
      > configurables (integraciones). El patron .NET correcto NO es reflexion abierta (Activator sobre texto,
      > el legacy cayo en RCE por eso); es un REGISTRO DE VERBOS TIPADOS resuelto por DI ("keyed handlers" /
      > command dispatch), que YA existe en ECOREX: el RulesEngine (`Ecorex.Application/Rules/Verbs/`,
      > `IRuleVerb.Name`). MAPEO: boton (`FormControlType.Button`) -> `FormFieldRule` -> verbo tipado
      > (allow-list) -> integracion (Siigo/WhatsApp/correo). PENDIENTE de construir: el usuario analizara la
      > documentacion (esta nota + Clases de Reglas del modulo Documental) antes de delegarlo.
- Aceptacion: al confirmar se genera el PDF y se dispara el webhook configurado (DIFERIDO); una firma se
  captura y se guarda (HECHO); el precio de costo solo lo ve el rol autorizado (HECHO, permisos por campo).
  > Nota: el nucleo de campo (defaults, formato, mascaras, permisos, captura Tier 2 inline) YA esta. Solo
  > restan las piezas de INTEGRACION grandes (object storage, renderer PDF, cola de webhooks/botones) ->
  > incrementos siguientes, en pausa por decision del usuario.

## B. Preguntas abiertas (cerrar antes de delegar)

**Identidad y transaccionalidad**
1. Por defecto, un formulario transaccional, cual modo de identidad usa: consecutivo del
   sistema o clave natural? Hay formularios sin numero pero si transaccionales (solo estado)?
R/ Debe ser indicado una propiedad que indica como el formulario produce la identidad tambien existe el formulario solido que no tiene identidad.
2. El consecutivo debe poder ser **por entidad** (una numeracion por sede/area) desde el inicio,
   o una sola por tenant basta para v1?
R/Una sola por tenant y por id de formulario
3. Se permite **editar** un registro ya Confirmado (con auditoria) o solo se puede **anular** y
   crear uno nuevo?
R:/Algunos eventos pueden generar el cierre pero tambien debe ser parametrizable, por ejemplo si se agrega un componente firma eso produce el cierre
4. La fecha de transaccion la fija el sistema (hoy) o el usuario puede backdatearla (con permiso)?
R:/ por sistema pero es interna del sistema si existen campos fecha agregados al formulario dependen de lo que llene un usuaio

**Datos / lookups**
1. Que fuentes priorizar en F1: Directorio (clientes), Inventario (items), Contenedor de datos
   generico... en que orden?
R:/ Todas las tablas del sistema mas datos de cotenedores.
2. El autollenado copia el valor (foto del dato al momento) o guarda solo el id y resuelve en
   lectura? (Copiar preserva historia; referenciar refleja cambios.) Recomendacion: guardar id +
   copiar los campos que importan al hecho (ej. precio pactado).
R:/Mejor es copiar
3. El buscador debe permitir **crear al vuelo** un dato que no existe (ej. cliente nuevo desde el
   formulario) o solo seleccionar existentes?
R:/ Permite ir al modulo de creacion si falta el dato 
**Calculo**
8. Alcance de las formulas en v1: aritmetica + condicional + fechas basta, o se necesitan
   funciones de texto/busqueda desde el inicio?
R:/ Saca la artilleria pesada en estos calculos
9. Los totales de tabla y roll-ups, se recalculan tambien en reportes/BI, o solo en captura?
R:/si recalculo
**Formulario-modulo**
10. "Convertir en modulo" lo hace el configurador (Andres) por si mismo, o requiere aprobacion
    de un admin (porque crea menu + permisos globales)?
R:/ Convertir en modulo es una opcion de configuracin del formulario es opcional
11. Los KPIs de la bandeja son fijos (cantidad, suma, %) o configurables por el usuario?
r:/ configurables
**Gobierno / alcance**
12. Prioridad real del usuario: cual de F1..F6 resuelve el dolor mas inmediato (por lo dictado
    parece F1 lookups + F2 calculo para cotizaciones)? Confirmar para ordenar la primera entrega.
R:/ si este es uno de los conceptos que debemos llevar pero formularios no es rigido
## C. Riesgos

- **Formulas/expresiones**: tentacion de permitir codigo arbitrario (el legacy cayo en RCE por
  reflexion). Mitigacion: evaluador tipado con allow-list, sin acceso a SQL ni a tipos del host.
- **Lookups y rendimiento**: catalogos grandes (miles de items/terceros) exigen busqueda
  server-side paginada + indices; nunca traer el catalogo completo al cliente.
- **Consecutivos y concurrencia**: dos confirmaciones simultaneas no pueden repetir numero;
  `ISequenceService` debe consumir bajo bloqueo/transaccion (verificar que ya lo hace).
- **DAL dual**: cada campo/entidad nuevo y cada indice de dimension deben generarse en PG **y**
  SQL Server (condicion de merge del proyecto).
- **Alcance**: F1..F6 es grande; conviene entregar por olas verificables y NO mezclar
  transaccionalidad (F3) con formulario-modulo (F4) en la misma ola.

## D. Protocolo de trabajo agente-worktree con BD remota compartida (acordado 2026-07-12)

> [!important] ANCLA DE CONTEXTO PARA EL AGENTE
> Si el agente pierde contexto, este es el contrato de trabajo vigente. Volver aqui antes de
> tocar esquema o memoria. Decidido con el usuario el 2026-07-12.

**Contexto.** El desarrollo de formularios avanzados corre en un **git worktree dedicado**
(`funny-bell-3f8562`, rama `claude/briefing-worktree-formularios-f50017`), en **paralelo** con
la sesion principal. Ambas sesiones apuntan a la **MISMA BD remota (prod)**. Por eso el esquema
es un recurso compartido y su cambio se coordina.

**Loop de trabajo (por cada cambio de esquema):**
1. El agente **codea todo**: entidad + configuracion EF + **migracion DUAL** (PostgreSQL y
   SQL Server) + servicios + UI.
2. **Valida la migracion en un Postgres local efimero** (Testcontainers / contenedor desechable),
   NUNCA a ciegas contra la remota.
3. **Para** y entrega al usuario: **(a)** el script de la tabla/columnas (la migracion) y
   **(b)** para que la usa. El usuario le comenta a la sesion principal que ese cambio ocurre.
4. Con el **"dale"** del usuario, la migracion se aplica a la remota y el agente continua con
   lo que depende de ella.

**Dos reglas de oro (romper una rompe todo):**
- **Un solo dueno de la migracion = el agente del worktree.** La sesion principal NO genera su
  propia migracion para las mismas columnas; solo se entera (por eso se le avisa) y, si su rama
  las necesita, hace rebase / trae la del worktree. Dos migraciones para el mismo cambio =
  snapshots divergentes + conflicto de git.
- **La remota es compartida (prod): el apply es deliberado, no automatico.** En `Development` la
  app aplica migraciones al arrancar; mientras la migracion este pendiente, el agente mantiene su
  app apuntando al **local efimero**, no a la remota. Solo la apunta a la remota (y aplica)
  **despues** del "dale" del paso 3.

**Reporte de campos nuevos - F1 (borrador, pendiente de afinar tipos con doc 01 seccion D4):**

Tabla `form_question`, 7 columnas nuevas (todas ADITIVAS, no alteran nada existente):

| Columna | Tipo PG | Tipo SQL Server | Null | Default | Indice |
|---|---|---|---|---|---|
| `source_kind` | varchar(40) | nvarchar(40) | no | `'Options'` | no |
| `source_ref` | varchar(200) | nvarchar(200) | si | - | no |
| `display_field` | varchar(120) | nvarchar(120) | si | - | no |
| `value_field` | varchar(120) | nvarchar(120) | si | - | no |
| `filter_json` | jsonb | nvarchar(max) | si | - | no |
| `autofill_map_json` | jsonb | nvarchar(max) | si | - | no |
| `presentation` | varchar(40) | nvarchar(40) | no | `'Autocomplete'` | no |

Enums nuevos (persistidos como string, registrar en `ConfigureConventions` con
`HaveConversion<string>()` en `EcorexDbContext`):
- `SourceKind` = `Options | DataContainer | Tercero | Item`
- `Presentation` = `Autocomplete | Dropdown | Modal`

> Nota de reconciliacion con el prompt de arranque F1: el prompt planteaba que la sesion
> principal aplicara la migracion; aqui se acordo que **el agente del worktree la genera Y (tras
> el "dale") la aplica**, y la principal solo queda enterada. Todo lo demas del prompt (alcance,
> criterios de aceptacion, DAL dual, multi-tenant, lectura obligatoria) sigue vigente.

Relacionado: [[00 - INDICE y objetivo (Formularios avanzados)]],
[[01 - Arquitectura, decisiones y datos (Formularios avanzados)]],
[[02 - UX y paneles de propiedades (Formularios avanzados)]], [[00 - Visión Formularios]],
[[Agentes de IA - Arquitectura y Operacion]].
