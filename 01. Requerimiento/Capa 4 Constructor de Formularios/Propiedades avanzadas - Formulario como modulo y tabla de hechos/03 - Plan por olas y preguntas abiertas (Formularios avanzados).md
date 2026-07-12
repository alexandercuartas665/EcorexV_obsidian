---
tipo: plan-construccion
proyecto: Propiedades avanzadas del Constructor de Formularios
proposito: Backlog por olas con criterios de aceptacion + las decisiones a cerrar con el usuario antes de delegar a un sub-agente.
---

# 03 - Plan por olas y preguntas abiertas (Formularios avanzados)

## ESTADO GENERAL (2026-07-11)

> [!note] PROPUESTA - NO CONSTRUIDO
> El motor base esta construido y validado (ver
> [[00 - INDICE y objetivo (Formularios avanzados)]] seccion 1). Este capitulo es el backlog
> del **salto a propiedades avanzadas** dictado por el usuario el 2026-07-11. Nada de esto
> esta implementado aun; las olas F1..F6 son la ruta propuesta. Antes de delegar a un
> sub-agente hay que cerrar las **preguntas abiertas** de la seccion B.

## A. Plan por olas (orden recomendado por valor / riesgo)

Se recomienda este orden porque F1 y F2 son alto valor y **reusan** infra existente
(contenedor de datos, directorio, inventario, motor de reglas), mientras F3/F4 introducen el
concepto de registro/modulo y conviene montarlos sobre F1/F2 ya firmes.

### F1 - Lookups / autocompletado (datos con dominio del tenant)
Objetivo: un campo se llena desde una tabla de datos del tenant (cliente, item, lista) con
autollenado de campos dependientes.
- [ ] `FormQuestion` gana `SourceKind/SourceRef/DisplayField/ValueField/FilterJson/AutofillMapJson/Presentation`.
- [ ] Servicio de busqueda server-side, parametrizado y paginado, sobre `DataContainerService`,
      `TerceroService` e `ItemService` (filtro por tenant garantizado).
- [ ] `DynamicFormRenderer`: control de autocompletar/lista/buscador; al elegir, aplica el
      autollenado (incluye campos dinamicos de `TerceroFieldService`/`ItemFieldService`).
- [ ] `FormDesigner`: bloque "Origen de datos" en el tab Datos (doc 02 seccion 2).
- Aceptacion: elegir un cliente autocompletando trae su NIT/direccion; el valor guardado es el
  id; imposible ver datos de otro tenant; test de aislamiento cross-tenant del lookup.

### F2 - Calculo y agregacion (formulas, totales de tabla)
Objetivo: campos calculados, totales de columna en GridDetail y roll-up al encabezado.
- [ ] `FormQuestion.CalcExpression` + `Aggregate`; evaluador de expresiones **tipado/sandbox**
      compartido con el `RulesEngine` (cliente para UX + revalidacion en servidor).
- [ ] GridDetail: columna calculada por fila + fila de totales + roll-up a campo del encabezado.
- [ ] `FormFieldValidator`/`FormResponseService`: recalculo en servidor al guardar (no confiar
      en el cliente para montos).
- Aceptacion: cotizacion con lineas item x cantidad = subtotal, total = suma; si el cliente
  manipula el total, el servidor lo corrige; sin expresiones arbitrarias (allow-list).

### F3 - Transaccionalidad (registro: identidad, estado, fecha)
Objetivo: el envio confirmado se vuelve un registro/hecho con numero, estado y fecha.
- [ ] `FormDefinition`: `IsTransactional`, `IdentityMode`, `IdentitySourceFieldCode`,
      `UniqueKeyFieldCodes`, `SequenceId`.
- [ ] `FormResponse`: `RecordNumber`, `Status(Draft/Confirmed/Voided)`, `TransactionDate`,
      `VoidedAt/By/Reason`.
- [ ] Confirmar: consume `ISequenceService` (modo Sequence) o valida unicidad (modo NaturalKey),
      todo en la transaccion; anular no libera el numero; idempotente por `FormResponse.Id`.
- [ ] Indices de dimension sobre las claves lookup mas consultadas (PG + SQL Server).
- Aceptacion: guardar una cotizacion consume COT-2026-000124; reintento no duplica; anular deja
  el registro Voided con motivo; unicidad de clave natural rechaza duplicados.

### F4 - Formulario como modulo (menu, permisos, bandeja, filtros, KPIs)
Objetivo: promover un formulario a modulo con su bandeja consultable.
- [ ] `FormDefinition`: `IsModule/ModuleMenuNodeId/ModuleIcon/ListColumnsJson/FilterFieldsJson`.
- [ ] Generacion de nodo de menu (`Kind=FormModule`) + policies `Form.{code}.*` (reusa
      `MenuConfigService`/`MenuPermissionFilter`).
- [ ] Ruta `/m/{code}`: bandeja con grid de registros, filtros dinamicos, KPIs, export, "en vivo"
      por SignalR.
- [ ] Vista aplanada (`data` + dimensiones) para BI/Power BI.
- Aceptacion: un formulario marcado como modulo aparece en el menu segun permiso; su bandeja
  filtra por sus campos; los KPIs muestran cantidad y % de crecimiento; export a Excel funciona.

### F5 - Maestro-detalle entre formularios
Objetivo: el detalle son registros de otro formulario (no solo GridDetail embebido).
- [ ] Campo `Subform`/relacion + `FormRecordLink` (padre-hijo).
- Aceptacion: una cotizacion referencia N lineas que son registros hijos reportables aparte.
- Nota: si F1..F4 cubren los casos reales con GridDetail, F5 puede diferirse.

### F6 - Transversales (defaults dinamicos, mascaras, PDF, webhooks, permisos de campo, Tier 2)
- [ ] Defaults dinamicos (`CurrentUser/Today/CurrentEntidad`), mascaras/formato, permisos por
      campo.
- [ ] Impresion/PDF con plantilla (object storage) + webhooks tipados al confirmar (allow-list,
      no reflexion) hacia integraciones (Siigo, WhatsApp HSM, correo).
- [ ] Captura real Tier 2 (foto, firma, GPS, archivo, barcode) + object storage.
- Aceptacion: al confirmar se genera el PDF y se dispara el webhook configurado; una firma se
  captura y se guarda; el precio de costo solo lo ve el rol autorizado.

## B. Preguntas abiertas (cerrar antes de delegar)

**Identidad y transaccionalidad**
1. Por defecto, un formulario transaccional, cual modo de identidad usa: consecutivo del
   sistema o clave natural? Hay formularios sin numero pero si transaccionales (solo estado)?
2. El consecutivo debe poder ser **por entidad** (una numeracion por sede/area) desde el inicio,
   o una sola por tenant basta para v1?
3. Se permite **editar** un registro ya Confirmado (con auditoria) o solo se puede **anular** y
   crear uno nuevo?
4. La fecha de transaccion la fija el sistema (hoy) o el usuario puede backdatearla (con permiso)?

**Datos / lookups**
5. Que fuentes priorizar en F1: Directorio (clientes), Inventario (items), Contenedor de datos
   generico... en que orden?
6. El autollenado copia el valor (foto del dato al momento) o guarda solo el id y resuelve en
   lectura? (Copiar preserva historia; referenciar refleja cambios.) Recomendacion: guardar id +
   copiar los campos que importan al hecho (ej. precio pactado).
7. El buscador debe permitir **crear al vuelo** un dato que no existe (ej. cliente nuevo desde el
   formulario) o solo seleccionar existentes?

**Calculo**
8. Alcance de las formulas en v1: aritmetica + condicional + fechas basta, o se necesitan
   funciones de texto/busqueda desde el inicio?
9. Los totales de tabla y roll-ups, se recalculan tambien en reportes/BI, o solo en captura?

**Formulario-modulo**
10. "Convertir en modulo" lo hace el configurador (Andres) por si mismo, o requiere aprobacion
    de un admin (porque crea menu + permisos globales)?
11. Los KPIs de la bandeja son fijos (cantidad, suma, %) o configurables por el usuario?

**Gobierno / alcance**
12. Prioridad real del usuario: cual de F1..F6 resuelve el dolor mas inmediato (por lo dictado
    parece F1 lookups + F2 calculo para cotizaciones)? Confirmar para ordenar la primera entrega.

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

Relacionado: [[00 - INDICE y objetivo (Formularios avanzados)]],
[[01 - Arquitectura, decisiones y datos (Formularios avanzados)]],
[[02 - UX y paneles de propiedades (Formularios avanzados)]], [[00 - Visión Formularios]],
[[Agentes de IA - Arquitectura y Operacion]].
