---
tipo: vision-modulo
modulo_codigo: "000291"
modulo_nombre: Flujos del Proceso
pagina: Bootstrap\Formularios\Modulos\Documentos\NEWFRONT_doc_procesos.aspx
codebehind: NEWFRONT_doc_procesos.aspx.vb
consecutivo_documento: "0D7"
tabla_principal: DOC_PROCESOS
base_alias: GENE
estado: vision destino .NET 10 + UI viva capturada en produccion (origen)
url_prod: https://app.bitcode.com.co/Formularios/modulos/Documentos/NEWFRONT_doc_procesos.aspx
stack_destino: .NET 10 / ASP.NET Core / EF Core 10 / Blazor / bpmn-js / SignalR
---

# Vision del modulo Flujos de Proceso (BPMN)

> Vision del sistema DESTINO para el motor de flujos de ECOREX sobre .NET 10.
> Este documento describe QUE se construye (el `WorkflowEngine` y su editor
> visual) y conserva al final el analisis del sistema ORIGEN (`AdmWorkflow` +
> `DOC_PROCESOS*`) como plano del ETL. Enlaza con [[Visión y entorno]] (vision
> maestra) y [[Visión y entorno|Prototipo Final ECOREX]] (aspecto y navegacion).

## D0. Estado de implementacion destino (.NET 10) -- al 2026-07-15

> [!success] El motor de flujos esta CONSTRUIDO y DESPLEGADO A PRODUCCION (`0bf057d`, 10.0.0.3)
> Las tres piezas de D1 (editor visual, motor de ejecucion, motor de reglas) existen y funcionan
> punta a punta; el arranque de tareas-proceso y la bandeja tambien. Detalle por pieza abajo.

| Pieza | Estado | Materializacion en codigo (`apps/backend`) |
|---|---|---|
| **Editor visual (bpmn-js)** | ✅ Operativo | `Flujos.razor` (000291) + `WorkflowDesignService`; parser/writer/merger propios (`BpmnProcessParser`, `BpmnXmlWriter`, `BpmnXmlMerger`), auto-layout, paleta completa (14 entradas), context pad, color y nota post-it por nodo. **La semantica que no viaja en el XML (cargo, formulario, reglas, reinicio) se CONSERVA** emparejando por `BpmnElementId` al guardar. |
| **Motor de ejecucion (`WorkflowEngine`)** | ✅ Operativo | `WorkflowEngine.cs` = port de `AdmWorkflow`: `AdvanceAsync`/`ResolveOutgoing` en transaccion, gateways (aprobado/rechazado enrutan), rechazo que reactiva el paso previo, **reinicios por `RestartNodeId`** (ciclo con `CycleIndex`), concurrencia optimista. Instancia + historial: `WorkflowInstance` / `WorkflowStepHistory` (append-only). |
| **Motor de reglas (`RulesEngine`)** | ✅ Operativo | `RulesEngine.cs` + 5 verbos tipados (`AsignarConsecutivo`, `BloquearCampoPorCondicion`, `GenerarTareasDesdeTabla`, `Notificar`, `PasarCampos`), enganchado por nodo via `WorkflowNodeRule` + `WorkflowRuleHook`. |
| **Asignacion por nodo -> cargo -> usuarios** | ✅ Operativo | `WorkflowNodePolicy` (ACL por nodo) + `INodeAssigneeResolver` (organigrama 000850). El primer paso nace asignado y notificado. |
| **Bandeja / runtime dentro de la tarea** | ✅ Operativo | `WorkflowInboxService` (reclamar/atender/aprobar/rechazar/reasignar); el runtime va EMBEBIDO en el detalle de la tarea (ADR-0038), descubrimiento por el alcance "Pendientes mios" del tablero. |
| **Formulario por paso** | ✅ Operativo | `WorkflowNodeForm` (form por nodo, migracion `AddDynamicForms`) + runtime `GetTaskStepFormsAsync`; un paso con formulario bloquea el avance hasta enviarlo (misma tx). |
| **Arranque de tareas-proceso** | ✅ Operativo | `WorkflowStartService` (resuelve el 1er nodo/encargado en seco), wizard con encargado restringido, arranque **form-first** (admision) y guardas (avisa si el flujo no es publicable/utilizable). Capitulo [[03 - Plan por olas pequenas|Tareas de proceso - Arranque y encargado]] CERRADO. |
| **Versionado del flujo** | ✅ Corrige el origen | `WorkflowDefinition` versionada + `PublishAsync` (bloquea publicar un flujo sin nodo Task); cada caso fija su version. |
| **DAL dual + multi-tenant** | ✅ | Todas las entidades `workflow_*` con `TenantId` + `HasQueryFilter`; migraciones **duales** (PG + SQL Server) aplicadas en prod al arrancar. |

**Cobertura de pruebas (suites verdes en la ultima corrida, integracion = dual PG + SQL Server):**
`WorkflowEngineTests` (10), `WorkflowDesignServiceTests` (8), `WorkflowStartServiceTests` (10),
`WorkflowInboxTests` (3), `DynamicFormsTests` (8, union form+flujo), `RulesEngineTests` (7) +
unitarias `Bpmn*Tests`, `WorkflowConditionEvaluatorTests`, `WorkflowInboxProjectionTests`, `RuleVerbTests`.
Veredicto QA formal: ver [[00 - Plan y veredicto QA - Flujos de Proceso]].

> [!todo] Pendientes reales (no bloquean el uso)
> - **Auditoria QA formal** del motor contra historias + DoD (hoy hay suites verdes, falta el gate).
> - **Reasignacion de paso con UI** dentro de la tarea (el servicio ya existe, `WorkflowInboxService`).
> - **SLA / vencimiento por paso** (recordatorio/escalamiento si un paso lleva N dias).
> - **Integracion IA** para sugerir/generar flujos (`GPT_CODE` del origen) -- aun no portada.
> - **Refresco en vivo del badge** por SignalR y **canal EMAIL** de notificaciones (backlog transversal).

## D1. Que es en el destino

El modulo de Flujos es el nucleo diferenciador de ECOREX (seccion 7 de
[[Visión y entorno]]): cada actividad (`TIPO_TAR`) puede tener asociado un flujo
**BPMN 2.0** que la dirige estado por estado. En el destino se materializa como
tres piezas separadas y testeables:

1. **Editor visual** — `bpmn-js` embebido en un componente **Blazor** via JS
   interop. Reemplaza el canvas propietario `crtFlujoProcesos.ascx`. El XML que
   produce es estandar OMG puro (ver [[Portabilidad BPMN - prueba en bpmn.io]]),
   sin lock-in.
2. **Motor de ejecucion** — `WorkflowEngine` (.NET), el port de `AdmWorkflow`.
   Resuelve la transicion `AdvanceAsync(tarea)` combinando la definicion del
   flujo, el estado actual del caso y las reglas del nodo. Detalle en
   [[AdmWorkflow - Motor de ejecucion]] y [[Ejecucion - SiguienteEstado y Reinicios]].
3. **Motor de reglas por nodo** — `RulesEngine`, el port de `cl_gestion_reglas`,
   invocado desde cada nodo del flujo. Detalle en
   [[Reglas - Quien invoca realmente (cierre)]].

## D2. Modelo de datos destino (EF Core 10, DAL dual)

El grafo propietario del origen se normaliza a entidades con `Guid` v7 (UUID
ordenable), `TenantId` obligatorio y `HasQueryFilter` + RLS. Correspondencia
origen -> destino:

| Origen (SQL Server legacy) | Destino (`IEcorexDbContext`) | Rol |
|---|---|---|
| `DOC_PROCESOS` | `workflow_definition` | Cabecera del flujo + `xml_bpmn` (jsonb/nvarchar(max)) |
| `DOC_PROCESOS_R` | `workflow_node` | Nodos BPMN (`TIPO_BPMN`, `PASO`, `ID_REINICIO`) |
| `DOC_PROCESOS_RULES` | `workflow_edge` | Aristas / sequence flows |
| `TAR_SEGUIMIENTO_PROCESO` | `workflow_step_history` | Estado por caso (append-only, Guid v7) |
| `DOC_PROCESOS_PLUGINS` / `GEN_COMPONENTES_R` | `workflow_node_component` | Formularios/plugins por paso |
| `PERMISO_CARGO` (`MODULO='PROCESOS_USUARIOS'`) | `workflow_node_policy` | ACL por nodo -> policies .NET |

El `xml_bpmn` se persiste tal cual (es portable). La semantica ejecutable
(asignaciones, formularios, reglas) sigue viviendo en tablas relacionales
ligadas al `ID_ELEMENTO` del nodo, pero ahora con FK reales y `TenantId`, no con
la disciplina del desarrollador. El editor bpmn-js sincroniza `workflow_node`
contra el XML en cada guardado (equivalente destino de `RepararNodosFaltantes` /
`EliminarNodosSobrantes`).

## D3. Ejecucion y eventos

`WorkflowEngine.AdvanceAsync` reemplaza `SiguienteEstado`: mismo while-loop con
tope de iteraciones (anti-ciclo, heredado), pero envuelto en **transaccion**,
con concurrencia optimista (`xmin`/rowversion) contra la doble-edicion de un
caso. Cada transicion emite eventos de dominio (`task.state.changed`,
`workflow.completed`) al bus (RabbitMQ/MassTransit) que alimentan SignalR
(tablero vivo), metricas (`workflow_instances_active`, `workflow_stuck_rate`) y
workers asincronos (recordatorios, escalamientos). El ratio de flujos trabados
es KPI operativo (seccion 16 de [[Visión y entorno]]).

## D4. Errores del origen que el destino corrige

- **SQL concatenado** en INSERT/UPDATE/SELECT de cabecera -> EF Core parametrizado.
- **Multi-tenant por columna `SUCURSAL`** + disciplina -> `TenantId` + filtro
  global EF + RLS en BD. Nunca fuga entre tenants.
- **Sin versionado del flujo** (un guardado sobrescribe y deja casos en curso
  inconsistentes; `ValidarCambiosFlujo` solo repara) -> `workflow_definition`
  versionada; cada caso fija la version con la que arranco.
- **Operaciones multi-tabla sin transaccion** (siembra + avance) -> todo comando
  multi-entidad en transaccion + Unit of Work.
- **`fn_ConsultaCargo` (UDF opaca)** -> query/servicio de resolucion de cargos
  portado a EF Core.

## D5. Portabilidad como palanca de migracion

El XML BPMN del origen es estandar OMG (validado: cargo sin cambios en
demo.bpmn.io, ver [[Portabilidad BPMN - prueba en bpmn.io]]). Esto reduce a cero
la migracion del editor visual (se reusa bpmn-js) y deja abierta la opcion de
adoptar un motor ejecutable estandar (Camunda/Flowable) escribiendo un exporter
que traduzca las tablas propietarias a `extensionElements`. La decision por
defecto del destino es **mantener el motor propietario portado** (`WorkflowEngine`)
por fidelidad de comportamiento, con la puerta abierta a Camunda cuando el
volumen lo justifique.

---

> A partir de aqui se conserva el ANALISIS DEL ORIGEN (sistema legacy WebForms)
> como referencia de migracion / reglas de negocio a preservar. El aspecto
> DESTINO que reemplaza este shell esta en [[Visión y entorno|Prototipo Final ECOREX]].

# Módulo ESPECIAL: Flujos del Proceso (000291) [ORIGEN / legacy]

> [!success] UI viva capturada en producción
> Hallazgos confirmados visitando la pantalla en `app.bitcode.com.co` (el aspecto DESTINO esta en [[Visión y entorno|Prototipo Final ECOREX]]):
> - **Tabs principales**: `Diseño` (#tab-1) y `Detalle` (#tab-3)
> - **Botones topbar**: `Nuevo Flujo`, `Guardar Cambios` (con marcador "último guardado: Sesion actual")
> - **Editor BPMN**: 4 elementos `<svg>` con clase `.djs-container` — **es bpmn-js library** (confirmado por el prefijo `djs-` / `bjs-` típico de [bpmn-io.github.io/bpmn-js](https://github.com/bpmn-io/bpmn-js)). Mismo motor visual que DokTrino.
> - **6 modales** en la página (configurar elementos, asignar usuarios, recursos, etc.)
> - **Acordeones del panel Propiedades**: `collapseBasicProps`, `modcollapse0`, `collapse1m`–`collapse4m`, `collapsePropiedad2`
> - **Combo `Seleccione el Proceso`** carga procesos reales del tenant. Visto: `COMERCIAL REQUERIMIENTO INFRAESTRUCTURA Y TECNOLOGIA (00001)`. También hay combo secundario `Seleccione Flujo` (multi-nivel: un proceso puede tener varios subflujos).
> - **Flujo de ejemplo cargado**: "Gestión de Seguimiento v1" con secciones "DETALLE DE ACTIVIDAD", "Configuración Básica", "Asignar Usuarios", "Recursos y Componentes".
> - **Breadcrumb**: `Home / Procesos / Flujo de Proceso`

# Módulo ESPECIAL: Flujos del Proceso (000291) — detalles

> [!abstract] En una frase
> Editor visual de flujos tipo **BPMN** (cajas + flechas + roles + plugins) que define la cadena de actividades por la que pasa una "tarea" del módulo de Actividades. Se monta sobre un canvas custom (`crtFlujoProcesos`), persiste el grafo en `DOC_PROCESOS_RU` / `DOC_PROCESOS_RULES`, y dispara ejecución vía `AdmWorkflow`.

---

## 1. Ubicación

| Ruta menú | Sección | URL |
|---|---|---|
| AUTOMATIZACIÓN → Flujos Del Proceso | Cliente | `../documentos/NEWFRONT_doc_procesos.aspx` |
| Desarrollo → Flujos del Proceso (duplicado) | Plataforma | mismo path |

## 2. Page_Load (resumen)

```vb
topBar.Title = "FLUJOS DE PROCESO"
topBar.Modulo = "000291"
topBar.Ubicacion = "Home/Procesos/Flujo de Proceso"

If Not IsPostBack Then
    ChatGPT = New Funciones.clChatGPT     ' integra IA para generar/sugerir flujos
    Call ListaProceso()                   ' combo de procesos existentes
    Call DetalleServicio()
End If

crtFlujoProcesos.Sucursal = Session("Empresa")
crtFlujoProcesos.Usuario  = Session("Usuario")
```

**Eventos topbar manejados:** Save → `AddCatalogo()`, Delete → `DelCatalogo()`, Clear → `Limpiar(1)`.

## 3. Persistencia — Tablas SQL involucradas

Confirmado por consulta a `db3dev`. Familia `DOC_PROCESOS*`:

| Tabla | Cols | Rol |
|---|---|---|
| `DOC_PROCESOS` | (revisar) | Cabecera del proceso: `CODIGO`, `NOMBRE`, `DESCRIPCION`, `SUCURSAL`, `GPT_CODE` |
| `DOC_PROCESOS_G` | | Grupos del proceso |
| `DOC_PROCESOS_GRUPOS` | | Definición de grupos de usuarios participantes |
| `DOC_PROCESOS_GRUPOS_USUARIOS` | | Membresía usuario↔grupo |
| `DOC_PROCESOS_PLUGINS` | | Plugins enganchados a un nodo (ej. enviar mail, OCR, llamar GPT) |
| `DOC_PROCESOS_R` | | Relación (¿roles? — verificar) |
| **`DOC_PROCESOS_RU`** | | **Nodos del diagrama** (la "U" sugiere UI / unidades visuales) |
| **`DOC_PROCESOS_RULES`** | | **Reglas / aristas / transiciones** entre nodos |
| `DOC_PROCESOS_TRD` | | Vínculo con Tablas de Retención Documental |

Y tablas adicionales del workflow ya documentadas:

| Tabla | Cols | Rol |
|---|---|---|
| `TAR_WORKFLOW_HORARIOS` | 15 | Programación temporal de pasos |
| `TAR_WORKFLOW_PASOS` | 9 | Pasos individuales en ejecución |
| `TAR_WORKFLOW_PROYECTOS` | 5 | Asociación workflow ↔ proyecto |

## 4. Operaciones de negocio (vistas en `AddCatalogo`)

1. **Validar nombre** vía `Funciones.ValidaError.empty_field`.
2. Si el código **no existe** → generar nuevo con `Funciones.tipdoc.Consecutivo("0D7", "", Empresa)`, `INSERT INTO DOC_PROCESOS`, `crtFlujoProcesos.GuardarFlujo(p_factura)`.
3. Si **existe** → `UPDATE` cabecera, `GuardarFlujo`.
4. **Validar cambios** con `AdmWorkflow.ValidarCambiosFlujo(codigo, Empresa)` (notifica al motor cualquier cambio que afecte ejecuciones en curso).
5. **Reparar nodos** y **eliminar sobrantes** (`RepararNodosFaltantes`, `EliminarNodosSobrantes`).
6. Refrescar `ListaProceso()` y `DetalleServicio()` + `UpdatePanelDatos.Update()`.

## 5. Componentes UI clave

| Componente | Rol |
|---|---|
| `crtFlujoProcesos` (custom user control) | Canvas del editor BPMN. Métodos: `LimpiarDiagrama`, `DibujarDiagrama(codigo)`, `GuardarFlujo(codigo)`, `RepararNodosFaltantes`, `EliminarNodosSobrantes`. |
| `cmbcodigo` | Combo con lista de procesos existentes. |
| `GridDetalle` | Listado de servicios/configuraciones asociadas al proceso. |
| `topBar` (compartido) | Save/Delete/Clear |

## 6. Integración con IA (ChatGPT)

`ChatGPT = New Funciones.clChatGPT` (sesión persistida en `Session("CHATGPT")`).
La cabecera `DOC_PROCESOS.GPT_CODE` permite versionar prompts asociados al flujo (campo guardado vacío en el INSERT actual, pero diseñado para llenarse).

## 7. Clase `AdmWorkflow` (negocio del motor)

Vista en el código:
```vb
Dim workFlow As AdmWorkflow = New AdmWorkflow(Session("Empresa"), "", "", "", Session("Usuario"), "", "")
Call workFlow.ValidarCambiosFlujo(cmbcodigo.SelectedValue, Session("Empresa"))
```

> ⚠️ **AdmWorkflow** está en alguna parte del proyecto Bootstrap o de Funciones — pendiente localizarla y documentarla aparte (`AdmWorkflow - Motor de ejecución.md`).

## 8. Riesgos y notas técnicas

- **SQL injection** en `INSERT/UPDATE/SELECT` de cabecera (txtnombre y otros van concatenados).
- **`Session("Empresa")` en cada query** — multi-tenant débil (depende de que la sesión no se contamine).
- **Estado del editor en cliente**: el canvas mantiene el grafo en JS; al guardar se serializa al servidor. Si se pierde el postback, se pierde el draft.
- **No hay versionado del flujo** — un cambio sobrescribe el anterior, las ejecuciones en curso pueden quedar inconsistentes (mitigado por `ValidarCambiosFlujo` que repara, pero no es versionado real).
- **Sin motor BPMN estándar de terceros (Camunda/Activiti)** — la ejecución corre en `AdmWorkflow` propio y el grafo se persiste en tablas custom (`DOC_PROCESOS_RU` + `DOC_PROCESOS_RULES`). Matiz importante: el XML del diagrama SÍ es BPMN 2.0 estándar OMG y carga sin cambios en bpmn.io (ver [[Portabilidad BPMN - prueba en bpmn.io]]); lo que no es estándar es la *semántica ejecutable* (`isExecutable="false"`, sin `extensionElements`). El layout es portable; el motor de ejecución no.

## 9. TODO en este vault

- [ ] Capturar screenshot del editor en vivo (carpeta `01. Requerimiento/Prototipo`).
- [ ] Documentar estructura completa de `DOC_PROCESOS_RU` y `DOC_PROCESOS_RULES` (SQL `INFORMATION_SCHEMA.COLUMNS`).
- [ ] Encontrar y documentar `AdmWorkflow` (clase de ejecución).
- [ ] Documentar `crtFlujoProcesos.ascx` y su JS asociado.
- [ ] Listar plugins disponibles en `DOC_PROCESOS_PLUGINS`.
- [ ] Diagrama de secuencia: usuario edita → guarda → workflow valida → propaga a ejecuciones activas.
