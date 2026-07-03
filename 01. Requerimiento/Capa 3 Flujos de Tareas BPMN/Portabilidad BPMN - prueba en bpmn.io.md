---
tipo: prueba-tecnica
fecha: 2026-06-28
hallazgo: 100% portable — XML del flujo cargó sin modificaciones en demo.bpmn.io
estado: prueba tecnica ORIGEN + decision de arquitectura DESTINO
destino_editor: bpmn-js embebido en Blazor (JS interop)
---

# Portabilidad del XML BPMN (prueba ORIGEN + decision DESTINO)

> Prueba tecnica que confirma que el XML BPMN del origen es estandar OMG puro y
> por tanto portable sin lock-in. Es un **punto a favor de la migracion** a
> .NET 10: el editor visual se reusa tal cual. Fija primero la decision DESTINO
> y conserva el detalle de la prueba. Enlaza con [[Visión y entorno]] (seccion 7)
> y [[00 - Prototipo Final ECOREX]].

## D. Decision de arquitectura DESTINO

- **Editor visual**: se reutiliza **bpmn-js** embebido en **Blazor** via JS
  interop (marca cero horas de reescritura del editor; el prototipo aun lo lista
  como pendiente — ver [[00 - Prototipo Final ECOREX]] "que falta").
- **Persistencia del diagrama**: columna `xml_bpmn` en `workflow_definition`
  (jsonb en Postgres / nvarchar(max) en SQL Server, via `IEcorexDbContext`), mas
  las tablas normalizadas `workflow_node` / `workflow_edge` / `workflow_node_*`
  que guardan la semantica ejecutable (no viaja en el XML).
- **Motor de ejecucion**: `WorkflowEngine` propietario portado (decision por
  defecto). La opcion Camunda/Flowable queda abierta escribiendo un exporter que
  traduzca las tablas a `extensionElements` cuando el volumen lo justifique.
- **Versionado**: el destino versiona `workflow_definition` (corrige la ausencia
  de versionado del origen); cada caso fija la version con la que arranco.

---

> A continuacion, el detalle de la PRUEBA sobre el sistema ORIGEN.

# Portabilidad del XML BPMN [ORIGEN] — prueba en `demo.bpmn.io`

> [!success] CONFIRMADO: 100% portable, cero modificaciones requeridas
> El XML BPMN extraído del editor de ECOREX (flujo `00001 COMERCIAL REQUERIMIENTO INFRAESTRUCTURA Y TECNOLOGIA`) se cargó **directamente, sin tocar un solo caracter**, en el modelador oficial `demo.bpmn.io` y renderizó los **47 elementos** idénticamente a la fuente.

---

## Procedimiento de la prueba

1. **Origen**: editor en `https://app.bitcode.com.co/Formularios/modulos/Documentos/NEWFRONT_doc_procesos.aspx`
2. **Captura**: `bpmnModeler.saveXML({format: true})` → 15,911 bytes XML (47 elementos lógicos)
3. **Guardado**: archivo `.bpmn` en este vault → [[ejemplo-bpmn-flujo-00001.bpmn]]
4. **Carga en destino**: navegar a `https://demo.bpmn.io/new`, ejecutar `bpmnio.openDiagram(xml)` con el XML literal
5. **Resultado**: ✅ sin errores, sin warnings, **47 nodos rendered** correctamente

```javascript
// Lo que se ejecutó en la consola de demo.bpmn.io
await bpmnio.openDiagram(xml);
// → ok: true
document.querySelectorAll('g.djs-element').length
// → 47 (mismo conteo que en la fuente)
document.querySelectorAll('svg').length
// → 8
```

---

## Implicaciones

### 🟢 Para migración (esto es el insight grande)

| Decisión | Justificación |
|---|---|
| **Reutilizar `bpmn-js` en el destino .NET 10 / Blazor** | No hay lock-in — el XML es estándar OMG puro |
| **No hace falta convertir/transformar nada** | El destino puede leer las definiciones tal cual y guardarlas tal cual |
| **Camunda Platform / Camunda 8 / Flowable son opciones reales** | Todos importan BPMN 2.0 sin extensiones, lo nuestro encaja |
| **Migración del editor visual = 0 horas de trabajo** | Solo reusar bpmn-js (es framework-agnostic, funciona dentro de Blazor con interop) |
| **Migración del motor de ejecución = N horas** | Aquí sí hay trabajo: portar `AdmWorkflow` + tablas `DOC_PROCESOS_R/_RULES/_PLUGINS/_GRUPOS` + ACL `PERMISO_CARGO` → o reemplazar todo con un motor BPMN ejecutable estándar (ver siguiente sección) |

### 🟡 La compensación

Como el XML tiene `isExecutable="false"` y **cero extensionElements**, **la semántica ejecutable NO está en el XML** — los datos críticos (asignación de usuarios, formularios por paso, reglas, plugins) viven afuera, en tablas relacionales. Esto significa:

- **Opción A — Mantener tablas propietarias** y portar `AdmWorkflow` tal cual a C# → menos cambio para el usuario, mismo modelo mental, motor custom mantenido
- **Opción B — Migrar a Camunda 7/8 con `extensionElements`** → convertir las tablas a atributos en el XML BPMN, ganar el motor estándar (timers, signals, message events, monitor web, REST API, etc.)
  - Ej. `<bpmn2:userTask camunda:assignee="${user}" camunda:formKey="cotizacion">` con `<bpmn2:extensionElements><camunda:formField id="precio" type="long"/>...`
  - Migración más costosa pero te lleva a un estándar de industria con tooling maduro

### 🔴 Lo que NO importó la prueba

- **Las extensionElements custom** no existen (cero datos custom) → no aplica
- **Los datos del panel propiedades** (asignación, formularios, reglas) NO viajaron porque no están en el XML → para una migración real, hay que **escribir un exporter** que genere `extensionElements` Camunda a partir de las tablas
- **Los layouts visuales** (`bpmndi:BPMNDiagram` con `dc:Bounds` y `di:waypoint`) **SÍ viajaron** — bpmn.io renderizó cada nodo y arista en sus coordenadas exactas

---

## Captura

El flujo se cargo en demo.bpmn.io con sus 47 elementos y renderizo sin errores (portabilidad BPMN confirmada).

---

## Replicar la prueba (manual)

1. Abrir https://demo.bpmn.io/new
2. Menú **Open file** (icono de carpeta arriba a la izquierda) → seleccionar `ejemplo-bpmn-flujo-00001.bpmn` de este vault
3. El diagrama renderiza inmediatamente con los 8 tasks, 3 gateways, 2 endevents, 1 startevent y 7 textannotations del flujo real de ECOREX

---

## Conclusión accionable

**Para el documento de arquitectura del sistema destino DokTrino (también afecta a Visal / cualquier proyecto BitCode que herede el motor de Flujos):**

- El **motor visual** se reutiliza tal cual (bpmn-js)
- El **modelo de datos del diagrama** se mantiene con campos `xml_bpmn` (text) + tablas asociadas (`flujo_nodo`, `flujo_regla`, `flujo_componente`, `flujo_permiso_cargo`)
- El **motor de ejecución** es la pieza que requiere decisión arquitectónica:
  - mantener semántica propietaria (mínimo cambio)
  - vs adoptar BPMN ejecutable estándar (Camunda/Flowable — máximo beneficio a largo plazo)

Esta nota cierra la duda sobre lock-in: **no lo hay en el frontend visual**.
