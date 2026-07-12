---
tipo: indice-proyecto
proyecto: Propiedades avanzadas del Constructor de Formularios
capa: Capa 4 - Constructor de Formularios
modulos_web: /formularios (000131) + /contenedor-datos + /directorio + /inventario
estado: PROPUESTA / DISENIO (dictado del usuario 2026-07-11; motor base YA validado)
fecha: 2026-07-11
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

> [!note] ESTADO 2026-07-11: PROPUESTA. El motor base esta CONSTRUIDO; esto es el salto siguiente.
> El Constructor de Formularios ya define, renderiza, guarda (1 fila JSON), publica por
> token, se une al flujo BPMN y dispara reglas (validado sobre el codigo real, ver seccion 1).
> Lo de este capitulo es el **backlog de propiedades avanzadas** dictado por el usuario el
> 2026-07-11, aun SIN construir. El plan por olas y las decisiones a cerrar estan en
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

- **Maestro-detalle entre formularios** (cabecera/detalle: cotizacion -> items). D7.
- **Valores por defecto dinamicos** (usuario actual, fecha hoy, entidad del contexto).
- **Mascaras y formato** (moneda, telefono, documento, %).
- **Permisos a nivel de campo** (ver/editar por rol).
- **Impresion / PDF con plantilla** del registro (sucesor de las plantillas PDF->Blob legacy).
- **Webhooks / eventos al guardar** (integraciones: Siigo, WhatsApp, correo).
- **Adjuntos Tier 2** (foto, firma, GPS, archivo, codigo de barras: hoy placeholder).
- **Import de formulario desde imagen (IA)** (sucesor de FormCreatorMCP; encaja en Capa 7).

Detalle y recomendaciones en D8.

## 3. Brechas (que falta construir hoy)

| Brecha | Hoy | Objetivo |
|---|---|---|
| El formulario no es "registro" | 1 respuesta = 1 fila JSON sin identidad de negocio ni estado transaccional | Registro con numero, estado (Borrador/Confirmado/Anulado), fecha de transaccion |
| Sin identidad configurable | No hay modo de identidad en la definicion | `IdentityMode { None, NaturalKey, Sequence }` + config |
| Campos solo con opciones fijas | `OptionsJson` estatico en Select/Radio/MultiCheck | Origen de datos (Contenedor/Directorio/Inventario) + typeahead + autollenado |
| Sin calculo | No hay campo calculado ni totales de tabla | Formula tipada + agregados de columna + roll-up |
| Sin modulo/bandeja | Solo el gestor generico `/formularios` | Promover a modulo + bandeja con filtros + KPIs + export |
| Multimedia | Tier 2 son placeholders | Captura real + object storage |

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
