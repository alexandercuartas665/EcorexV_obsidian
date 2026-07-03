---
tipo: nota-capa
capa: 7
proposito: Capa inteligente de ECOREX Tareas (.NET 10) - agentes que operan y configuran tareas, flujos, formularios y reglas, gobernados y monetizables
estado: diseno objetivo de migracion
stack_destino: .NET 10 / ASP.NET Core / EF Core 10 / Blazor
---

# Agentes de IA - Arquitectura y Operacion

> Documento de Capa 7 conectado con [[Visión y entorno]] (vision destino) y con los
> motores base ([[00 - Visión Flujos]], [[00 - Visión Formularios]],
> [[Reglas - Quien invoca realmente (cierre)]]). Define la capa inteligente:
> agentes, orquestacion, memoria, prompts, herramientas, limites y guardrails.
> El sistema ORIGEN ya usa IA de forma primitiva (`Funciones.clChatGPT`, un solo
> proveedor cableado); el destino lo generaliza a una capa gobernada — ver seccion 12.

## 1. Proposito

La IA en ECOREX Tareas es una capacidad **gobernada y monetizable**, no una funcion
decorativa. Su objetivo es **reducir la friccion operativa** (crear tareas, moverlas
por sus flujos, llenar formularios, saber que esta pendiente) y **acelerar la
configuracion** (modelar un flujo BPMN, armar un formulario o redactar una regla
desde lenguaje natural), sin reemplazar el criterio humano en decisiones sensibles.

Hay dos agentes estrella, uno por audiencia:

- **Asistente Operativo** (para el usuario del tenant): entiende peticiones como
  "crea una tarea de cobro para el cliente X con vencimiento el viernes", "que tengo
  pendiente hoy", "avanza mi flujo de aprobacion", y las ejecuta contra el motor real.
- **Copiloto de Configuracion** (para el administrador): ayuda a **construir** los
  artefactos configurables — proponer un flujo BPMN, generar un formulario dinamico o
  redactar una regla — que luego el humano revisa y publica. Este copiloto es el que
  mas apalanca el diferenciador de ECOREX (proceso + formulario + regla sin codigo).

Regla rectora de toda la capa: **la IA jamas inventa datos, jamas salta una regla o
un permiso, y jamas avanza un flujo mas alla de un nodo que requiere intervencion
humana sin confirmacion.** Solo actua sobre lo que el WorkflowEngine, el RulesEngine
y el modelo de permisos validan. Toda ejecucion respeta limites por plan,
trazabilidad de tokens, aislamiento por tenant y supervision humana.

## 2. AI Provider Gateway

Capa de abstraccion que desacopla la plataforma de un unico proveedor (OpenAI,
Claude, Gemini, Azure OpenAI, Ollama local, OpenRouter, DeepSeek). Normaliza
peticiones, respuestas, conteo de tokens, errores, streaming y fallback.

```csharp
public interface IAiProvider { Task<AiResponse> GenerateAsync(AiRequest request); }

builder.Services.AddKeyedScoped<IAiProvider, OpenAiProvider>("openai");
builder.Services.AddKeyedScoped<IAiProvider, ClaudeProvider>("claude");
builder.Services.AddKeyedScoped<IAiProvider, GeminiProvider>("gemini");
```

El `PlatformAdmin` habilita proveedores (`AiProviderConfig`, llave **cifrada en Key
Vault / DataProtection**) y cada tenant usa los habilitados segun su plan. Failover:
si el proveedor principal satura, se redirige al fallback. Trazabilidad completa de
consumo en `AiUsageLog`. Esto reemplaza el `clChatGPT` mono-proveedor del origen
(seccion 12).

## 3. Agent Orchestrator

Nucleo tecnico que convierte eventos de negocio (mensaje del usuario, tarea creada,
flujo trabado) en intervenciones de IA observables. Por cada solicitud: selecciona
el agente, **valida limites del plan y cupo de tokens**, resuelve el `TenantId`
activo, recupera contexto (tarea, estado del flujo, carga del usuario), arma el
prompt, consulta el proveedor, **ejecuta herramientas permitidas**, registra costos
y publica el resultado.

```txt
Evento (mensaje / task.created / workflow.stuck)
   -> Orchestrator -> [valida plan/tokens + tenant activo]
   -> recupera contexto (tarea, flujo, formulario, carga)
   -> arma prompt + herramientas segun rol/permisos
   -> proveedor IA (via Gateway)
   -> accion (crear tarea borrador / proponer avance / sugerir regla)
   -> registra consumo (AiUsageLog) + audita
```

## 4. Herramientas (Tools) del Agente

A diferencia de un chatbot generico, los agentes tienen herramientas con efecto
real, **todas validadas server-side** y sujetas a las policies del usuario:

| Herramienta | Que hace | Guardrail |
|-------------|----------|-----------|
| `buscar_tareas(filtros)` | Tareas del tenant segun criterios | Solo lectura; respeta permisos y filtro de tenant |
| `consultar_carga(usuario?)` | Carga de trabajo / pendientes del dia | Solo lectura |
| `crear_tarea_borrador(...)` | Crea `TaskItem` en estado borrador | No la asigna ni la publica sin confirmacion |
| `avanzar_flujo(instancia, nodo)` | Pide al WorkflowEngine avanzar | Solo nodos automaticos; nodo humano requiere confirmacion |
| `explicar_flujo(codigo)` | Resume un flujo BPMN en lenguaje natural | Solo lectura |
| `proponer_flujo(descripcion)` | Genera un borrador BPMN | El humano revisa y publica; nunca activa solo |
| `proponer_formulario(descripcion)` | Genera definicion de formulario | Idem: borrador para revision |
| `sugerir_regla(objetivo)` | Redacta una regla candidata | No se activa sin aprobacion |
| `escalar_a_humano()` | Deriva a un responsable | Ante duda, dato ambiguo, permiso o caso sensible |

Ninguna herramienta ejecuta SQL crudo ni salta el RulesEngine; todas pasan por los
servicios de Application con sus validaciones y transacciones.

## 5. Asistente Operativo (agente principal del tenant)

Atiende al usuario dentro de la app (o por un canal como WhatsApp/Evolution si el
tenant lo habilita). Detecta intencion, identifica la actividad (`TIPO_TAR`), el
proyecto y los datos, y actua.

```txt
Usuario: "crea una tarea de seguimiento para la cotizacion de Andina y asignamela"
```

Extraccion:

```json
{ "intent": "crear_tarea", "actividad": "Seguimiento", "referencia": "cotizacion Andina",
  "asignar_a": "self", "proyecto": null }
```

Flujo: `crear_tarea_borrador` -> "Cree la tarea 'Seguimiento cotizacion Andina',
asignada a ti, sin fecha limite. La publico?" -> el usuario confirma -> la tarea
entra a su tablero y, si la actividad tiene flujo, se instancia el WorkflowEngine.
**Si falta un dato obligatorio (ej. el flujo exige un formulario de intake), el
agente lo pide; nunca inventa el valor.**

## 6. Copiloto de Configuracion (agente del administrador)

El agente que materializa el "sin codigo". Ayuda al administrador a construir los
artefactos configurables a partir de una descripcion:

- **Flujo BPMN**: "necesito un flujo de aprobacion de compras con revision de jefe y
  visto bueno de finanzas" -> `proponer_flujo` genera un borrador BPMN (nodos,
  gateways, transiciones) que se abre en el editor bpmn-js para revision. Ver
  [[AdmWorkflow - Motor de ejecucion]].
- **Formulario**: "un formulario de requisicion con articulo, cantidad, centro de
  costo y justificacion" -> `proponer_formulario` genera la definicion (tipos de
  control, validaciones) para revisar. Ver [[00 - Visión Formularios]].
- **Regla**: "si el monto supera 5 millones, exigir aprobacion del gerente" ->
  `sugerir_regla` redacta la condicion/accion candidata. Ver
  [[Reglas - Quien invoca realmente (cierre)]].

**Guardrail clave**: el copiloto SIEMPRE deja un borrador para revision humana;
nunca activa un flujo, formulario o regla en produccion por su cuenta.

## 7. Agente Clasificador de Intencion

Detecta de cada mensaje: intencion (crear tarea, consultar, avanzar flujo, reportar
avance, configurar, queja), entidad mencionada (tarea/proyecto/flujo/formulario),
datos faltantes y urgencia. Alimenta el enrutamiento y permite que humano e IA
trabajen con informacion estructurada, no solo texto libre.

## 8. Agente de Escalamiento y Resumen de Flujos

Apoya la operacion de procesos: detecta **flujos trabados** (alimenta el KPI
`workflow_stuck_rate` de la [[Visión y entorno|vision]] seccion 16), resume el estado
de un proceso ("esta tarea lleva 3 dias en el nodo 'Revision de jefe', esperando a
Maria"), y sugiere la siguiente accion o escala al responsable. Se apoya en el
historial `workflow_step_history` del WorkflowEngine.

## 9. Memoria, Prompts y Guardrails

- **Memoria inmediata**: ultimos mensajes de la conversacion / contexto de la vista.
- **Resumen del usuario**: rol, carga actual, actividades frecuentes, proyectos.
- **Contexto del tenant**: modulos habilitados, actividades (`TIPO_TAR`), politicas,
  tono. Nunca contexto de otro tenant (aislamiento).
- **Prompts versionados** por agente (`AiAgentPrompt`), enrutados por reglas.
- **Recursos entregables** (`AiAgentResource`): el agente adjunta EXACTO un recurso
  cargado (instructivo, plantilla) con un marcador `[[enviar: Nombre]]`, sin
  improvisar contenido sensible.

Guardrails no negociables:

- No inventar tareas, datos de formulario, estados de flujo ni resultados de regla.
- No avanzar un flujo mas alla de un nodo humano sin confirmacion explicita.
- No activar en produccion un flujo, formulario o regla generado por IA sin revision.
- No saltar permisos ni el RulesEngine; toda accion pasa por Application.
- No exponer datos de otro tenant ni de otro usuario sin permiso.
- Escalar a humano ante ambiguedad, permiso insuficiente o caso sensible.

## 10. Consumo, Limites y Costos

Todo consumo pasa por `IAiUsageService.RecordAsync` (proveedor, modelo, tokens, costo
estimado, agente, fuente). `GetQuotaAsync` aplica el cupo mensual del plan
(`max_ai_tokens_monthly`, modo Hard bloquea) definido en
[[Gestion de Empresas - Admin multi-tenant]]. Ningun agente se ejecuta sin tenant
activo ni si el plan no habilita IA. El `PlatformAdmin` ve consumo global y por
tenant.

## 11. Reglas No Negociables para IA

- Ningun agente se ejecuta sin tenant activo.
- Ningun agente consume tokens si el plan no lo permite.
- Toda ejecucion registra proveedor, modelo, tokens, costo, agente y usuario.
- La IA no inventa datos, estados ni resultados.
- La primera version opera en **modo asistido** (propone; el humano confirma/publica).
- Las acciones 100% automaticas solo se permiten en escenarios de bajo riesgo
  configurados por el administrador del tenant (ej. crear una tarea rutinaria).

## 12. Sistema ORIGEN (referencia de migracion)

El legacy ya tiene IA, pero primitiva: la libreria `Funciones` incluye
`clChatGPT` (ver [[00 - Mapa Funciones]]), un cliente **mono-proveedor** (OpenAI)
con la llave gestionada de forma dispersa, integrado puntualmente — por ejemplo en el
editor de flujos `NEWFRONT_doc_procesos`. No hay gateway, ni trazabilidad de tokens,
ni limites por plan, ni herramientas gobernadas: es una llamada suelta a un modelo.

El destino conserva la capacidad pero la formaliza: `clChatGPT` -> **AI Provider
Gateway** multi-proveedor con `AiUsageLog`, cupos por plan, herramientas validadas y
guardrails. Este es el mismo salto que la Capa 3 de IA de CUBOT.nails. Ver tambien la
seccion de ChatGPT en [[Puntos ciegos y motores transversales]].

## 13. Orden de Implementacion de IA

1. AI Provider Gateway (servicio interno, sin UI avanzada).
2. Agent Orchestrator con trazabilidad y herramientas de solo lectura
   (`buscar_tareas`, `consultar_carga`, `explicar_flujo`).
3. Agente Clasificador de Intencion.
4. Asistente Operativo en modo sugerencia (propone, humano confirma).
5. Herramientas de escritura (`crear_tarea_borrador`, `avanzar_flujo`) con guardrails.
6. Copiloto de Configuracion (`proponer_flujo`/`proponer_formulario`/`sugerir_regla`).
7. Agente de Escalamiento y Resumen de Flujos.

## 14. Relacion con otras capas

- [[Visión y entorno]] — vision destino y KPIs (`workflow_stuck_rate`).
- [[Gestion de Empresas - Admin multi-tenant]] — planes, limites, cupo de tokens, PlatformAdmin.
- [[00 - Visión Flujos]] / [[AdmWorkflow - Motor de ejecucion]] — el motor que la IA opera y configura.
- [[00 - Visión Formularios]] — formularios que la IA llena y propone.
- [[Reglas - Quien invoca realmente (cierre)]] — reglas que la IA sugiere (sin saltarlas).
- [[Puntos ciegos y motores transversales]] — el ChatGPT legacy como punto de partida.
- [[HOJA DE RUTA DESARROLLO]] — fase donde entra la capa de IA (despues del nucleo).

---

*Documento vivo. Actualizar cuando se agregue un agente, una herramienta o cambie una politica de guardrails.*
