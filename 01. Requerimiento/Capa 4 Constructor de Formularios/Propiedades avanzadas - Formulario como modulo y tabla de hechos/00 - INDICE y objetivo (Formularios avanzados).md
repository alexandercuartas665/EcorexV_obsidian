---
tipo: indice-proyecto
proyecto: Propiedades avanzadas del Constructor de Formularios
capa: Capa 4 - Constructor de Formularios
modulos_web: /formularios (000131) + /contenedor-datos + /directorio + /inventario
estado: F1..F6 HECHOS y verificados (2026-07-13, BD local worktree; pendiente aplicar a prod). Diferido: PDF/plantilla, webhooks, object storage
fecha: 2026-07-11 (actualizado 2026-07-12)
autor: documentado por agente IA (validacion read-only del codigo real + dictado del usuario)
---

# Propiedades avanzadas - Formulario como modulo y tabla de hechos

> Capitulo de diseno. El objetivo es **elevar el formulario** de "captura suelta"
> (una respuesta que se guarda) a **dato transaccional gobernado**: un formulario
> que puede (1) convertirse en un **modulo del sistema** con su menu y permisos,
> (2) volverse una **tabla de hechos** consumiendo un consecutivo o una identidad
> externa en cada guardado, (3) llenar campos desde **tablas de datos con dominio
> del tenant** (autocompletado de clientes, items, listas), y (4) **calcular**
> (formulas, totales en tablas, filtros). Se apoya en [[00 - Visión Formularios]]
> (vision del motor) y reutiliza infraestructura que YA existe en el codigo.

> [!success] ESTADO 2026-07-12: EN CONSTRUCCION. El motor base ya estaba CONSTRUIDO; el salto avanza.
> - **F1 (lookups/autollenado): HECHO y verificado en navegador** (Directorio/Inventario/Contenedor;
>   commit `97d855d`). 
> - **F2 (calculo): HECHO** - campos calculados escalares + GridDetail (columna calculada por fila,
>   fila de totales, roll-up al encabezado); evaluador tipado + recomputo cliente/servidor. Verificado
>   en navegador (commits `1f98245` + GridDetail).
> - **F3 (transaccionalidad): HECHO** - confirmar/anular + identidad + panel de config + boton Anular;
>   verificado (FRM-021-000001).
> - **F4 (formulario-modulo): HECHO** - modulo con colocacion dinamica en el menu + bandeja con
>   columnas/filtros configurables, KPIs, export CSV+Excel, en vivo por SignalR, aplanado BI y policy por
>   menu; verificado. (Refinamiento opcional: policies tipadas.)
> - **F5 (maestro-detalle): HECHO** - campo Subform + FormRecordLink; el hijo se llena anidado y queda
>   enlazado; configurable en el designer; verificado.
> - **F6 (transversales): HECHO** - defaults dinamicos (Today/CurrentUser), formato (moneda/%/entero),
>   **permisos por campo** (ocultar/solo-lectura por rol), **mascaras de entrada** (phone/document) y
>   **captura Tier 2 real** (firma en canvas, GPS, archivo/foto->data-URI inline, tope 1 MB): TODO HECHO y
>   verificado E2E via Chrome (FRM-022, rol Advisor). **Diferido (integracion grande, NO construido)**:
>   PDF con plantilla, webhooks/botones-con-reglas (el usuario analiza la doc), object storage para
>   adjuntos grandes.
>
> Las migraciones de F1/F2/F3/F4 estan aplicadas en la BD LOCAL del worktree; **pendiente aplicarlas a
> prod** (protocolo y registro en [[04 - Registro de tablas y cambios de esquema (Formularios avanzados)]]).
> El plan por olas y las decisiones estan en
> [[03 - Plan por olas y preguntas abiertas (Formularios avanzados)]].

## 1. Estado actual del motor (validacion del codigo real, 2026-07-11)

Los cimientos estan y son solidos; lo que falta es la capa de **propiedades de negocio**
encima. Base ya construida:

| Capacidad del motor | Estado | Rutas (codigo real) |
|---|---|---|
| Definir formulario (arbol contenedores/preguntas, drag and drop) | Construido | `FormDefinitionService.cs`, `FormDesigner.razor` |
| Renderizar controles Tier 1 (Text, Select, Radio, Number, Date, Toggle, MultiCheck, TextArea, Heading, Literal) + GridDetail | Construido | `DynamicFormRenderer.razor`, `FormControlType.cs` |
| Guardar respuesta como 1 fila JSON (`jsonb` / `nvarchar(max)`), validacion servidor por campo | Construido | `FormResponseService.cs`, `FormFieldValidator.cs` |
| Publicar por token (hash SHA-256, expira, uso unico, revocable) + visor anonimo | Construido | `FormTokenService.cs`, `FormPublic.razor` |
| Union formulario <-> nodo BPMN (envio completa el paso en la misma transaccion) | Construido | `FormFlowLink`, `WorkflowNodeForm`, `FormResponseService.SaveAsync` |
| Reglas por campo (show/hide, setValue, required) via RulesEngine | Construido | `FormRuleDispatcher.cs` |
| Multi-tenant real + tests de aislamiento cross-tenant | Construido | filtro global + `DynamicFormsTests.cs` |

**Infraestructura reutilizable (clave para este capitulo) que YA existe:**

| Pieza | Que aporta a formularios avanzados | Ruta |
|---|---|---|
| **Consecutivos por tenant** | La identidad "por consecutivo" del formulario transaccional NO se inventa: se reusa | `TenantSequence.cs`, `ISequenceService`/`SequenceService.cs`, `AsignarConsecutivoVerb.cs` |
| **Contenedor de datos** (data source generico, import API REST) | Origen de los lookups/autocompletado con dominio del tenant | `DataContainer.cs`, `DataContainerConfig.cs`, `DataContainerService.cs`, `ContenedorDatos.razor` |
| **Directorio (terceros)** | Autocompletar clientes/proveedores + traer campos (NIT, direccion...) | `TerceroService.cs`, `TerceroFieldService.cs` |
| **Inventario (items/catalogos)** | Autocompletar items/productos + sus campos | `ItemService.cs`, `InventoryCatalogService.cs` |
| **RulesEngine tipado (verbos)** | Motor donde viven formulas y acciones sin SQL crudo ni RCE | `Ecorex.Application/Rules/` |

## 2. El salto que pide el usuario (5 ejes + extras)

1. **Formulario como MODULO del sistema.** Algunos formularios dejan de vivir solo en
   `/formularios` y se **promueven a modulo**: nodo de menu propio, ruta, permisos
   (Ver/Crear/Editar/Eliminar) e icono. Ver D1 en
   [[01 - Arquitectura, decisiones y datos (Formularios avanzados)]].
2. **Transaccionalidad -> tabla de hechos.** En las propiedades del formulario se activa
   "crear transacciones": cada envio pasa a ser un **registro/hecho** consultable. Dos
   modos de **identidad**:
   - (a) **Identidad externa**: el id sale de un dato (un SKU, el id de un tercero...).
   - (b) **Consecutivo del sistema**: en la config del formulario se entrega una secuencia;
     cada guardado **consume** el siguiente numero (reusa `ISequenceService`). Ver D2/D3.
3. **Campos con datos (lookup / autocompletado).** En las **propiedades de datos** de un
   campo se define que su contenido sale de una **tabla de datos con dominio del tenant**:
   autocompletar clientes, autocompletar items, o una lista para seleccionar. El campo
   guarda el id y puede **autollenar** campos dependientes. Ver D4.
4. **Calculo y agregacion.** Campos **calculados por formula** (ej. total = cantidad *
   precio), **totales en tablas** (sum/count/avg por columna del GridDetail), y **roll-up**
   del detalle al encabezado. Ver D5.
5. **Filtros / bandeja / indicadores.** El formulario-modulo tiene una **bandeja** (listado
   de registros) con **filtros dinamicos** por sus campos, KPIs (cantidad, % crecimiento) y
   export. Ver D6.

**Extras creativos propuestos (lo que ademas hace falta para "el mejor sistema"):**

- **Maestro-detalle entre formularios** (cabecera/detalle: cotizacion -> items). D7. **HECHO (F5)**.
- **Valores por defecto dinamicos** (usuario actual, fecha hoy, entidad del contexto). **HECHO (F6)**.
- **Mascaras y formato** (moneda, telefono, documento, %). **HECHO (F6)**.
- **Permisos a nivel de campo** (ver/editar por rol). **HECHO (F6)**.
- **Adjuntos Tier 2** (foto, firma, GPS, archivo). **HECHO (F6, inline)**; `barcode`/`audio` + object storage **diferidos**.
- **Impresion / PDF** del registro. **Basico HECHO (F6)**: print nativo del navegador (Imprimir / Guardar PDF) desde un documento limpio. **DIFERIDO**: PDF con plantilla de servidor + object storage (sucesor de las plantillas PDF->Blob legacy).
- **Webhooks / eventos al guardar** (integraciones: Siigo, WhatsApp, correo). **DIFERIDO** (via botones -> verbos tipados; usuario analiza doc).
- **Import de formulario desde imagen (IA)** (sucesor de FormCreatorMCP; encaja en Capa 7). **No iniciado** (fuera del alcance F1..F6).

Detalle y recomendaciones en D8.

## 3. Brechas (que falta construir hoy)

| Brecha | Hoy | Objetivo |
|---|---|---|
| ~~El formulario no es "registro"~~ **HECHO (F3)** | Registro con numero, estado (Borrador/Confirmado/Anulado) y fecha de transaccion | Construido y verificado (FRM-021-000001) |
| ~~Sin identidad configurable~~ **HECHO (F3)** | `IdentityMode { None, NaturalKey, Sequence }` + panel de config en el designer | Construido y verificado |
| ~~Campos solo con opciones fijas~~ **HECHO (F1)** | Origen de datos + typeahead + autollenado por copia (Contenedor/Directorio/Inventario) | Construido y verificado (commit `97d855d`) |
| ~~Sin calculo~~ **HECHO (F2)** | Campo calculado (evaluador tipado cliente+servidor) + GridDetail (columna calculada por fila, fila de totales, roll-up) | Construido y verificado |
| ~~Sin modulo/bandeja~~ **HECHO (F4)** | Modulo con colocacion en el menu + bandeja con columnas/filtros configurables, KPIs, export CSV+Excel, en vivo por SignalR | Construido y verificado |
| ~~Sin maestro-detalle~~ **HECHO (F5)** | Campo Subform + `FormRecordLink`; el hijo se llena anidado y queda enlazado | Construido y verificado |
| ~~Multimedia (Tier 2 placeholders)~~ **HECHO (F6)** | Firma/GPS/foto/archivo con captura real inline (data-URI, tope 1 MB) | Construido y verificado; **object storage / barcode / audio = diferidos** |
| ~~Impresion~~ **BASICA HECHA (F6)** | Print nativo del navegador (Imprimir / Guardar PDF) desde documento limpio | **DIFERIDO**: PDF con plantilla (object storage) + webhooks/botones -> verbos tipados (usuario analiza doc) |

## 4. Documentos del capitulo

- [[01 - Arquitectura, decisiones y datos (Formularios avanzados)]] - decisiones D1..D8,
  modelo de datos objetivo (nuevos campos/entidades) y la secuencia de consumo de un
  registro transaccional (identidad + consecutivo + reglas + auditoria).
- [[02 - UX y paneles de propiedades (Formularios avanzados)]] - como se ven las nuevas
  propiedades en el `FormDesigner` (propiedades del formulario + tab Datos/Calculo del
  campo), el GridDetail con totales, y la bandeja/listado con filtros.
- [[03 - Plan por olas y preguntas abiertas (Formularios avanzados)]] - backlog por olas
  con criterios de aceptacion + las decisiones a cerrar con el usuario.

## 5. Decisiones a cerrar (resumen)

El dictado deja varias decisiones de diseno abiertas (identidad por defecto, que fuentes de
datos priorizar, donde se evaluan las formulas, alcance de "formulario como modulo"). Estan
listadas como preguntas concretas en
[[03 - Plan por olas y preguntas abiertas (Formularios avanzados)]] seccion "Preguntas
abiertas". Cada decision de arquitectura del doc 01 lleva su recomendacion marcada.

## 6. Veredicto

Es un salto **grande pero muy apalancado**: 3 de los 5 ejes (consecutivos, contenedor de
datos, directorio/inventario) ya tienen su infraestructura construida, asi que buena parte
del trabajo es **cablear** propiedades nuevas sobre el motor existente, no construir de cero.
Se recomienda arrancar por los **lookups (F1)** y el **calculo (F2)** por su alto valor y bajo
riesgo, dejar **transaccionalidad (F3)** y **formulario-modulo (F4)** para cuando esos dos
esten firmes, y tratar cada eje como una ola con criterios de aceptacion (doc 03).

Relacionado: [[00 - Visión Formularios]], [[Constructor - Patron EAV y motor visual]],
[[Esquema completo - Tablas y tipos de control]],
[[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]],
[[Agentes de IA - Arquitectura y Operacion]].
