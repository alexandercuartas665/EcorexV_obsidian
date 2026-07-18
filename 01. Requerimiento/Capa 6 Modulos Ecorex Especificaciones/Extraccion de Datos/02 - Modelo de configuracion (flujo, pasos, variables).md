---
tipo: spec-modelo
proyecto: Extraccion de Datos
proposito: El modelo de dominio a construir para configurar flujos de automatizacion de navegador. Alcance = la CONFIGURACION (no el runtime).
---

# 02 - Modelo de configuracion (flujo, pasos, variables)

> Este es el nucleo del alcance actual: **poder guardar la configuracion de un flujo**. El runtime
> (que lo ejecuta) es el sub-agente Navegador y se documenta aparte (doc 03), diferido.

## 1. Vista de conjunto

```
ScrapeFlow (maestro)  1---N  ScrapeStep (pasos ordenados)
     |                            |
     | 1---N                      | (paso IA) --> config IA
     v                            v
ScrapeVariable (cifradas)    accion tipada / instruccion

ScrapeFlow --FK--> DataClient (que agente lo ejecuta)
ScrapeFlow --FK--> DataContainer (tabla destino por defecto)
ScrapeFlow --reusa--> ImportProcess (horario) + ImportRun (bitacora)
```

Un `ScrapeFlow` es lo que el legacy llamaba `WEB_SCRAPING` (maestro), sus `ScrapeStep` son los
`WEB_SCRAPING_R` (pasos-URL) fusionados con `WEB_SCRAPING_RS` (acciones), y las `ScrapeVariable` son
`WEB_SCRAPING_CLI` + `WEB_SCRAPING_RAV` (cliente + variables cifradas). Se colapsan las 10 tablas del
legacy en un modelo mas plano, aprovechando que el destino, la identidad y el horario **ya existen** en
ECOREX y no hay que remodelarlos.

## 2. `ScrapeFlow` (el maestro)

Campos (mas los de `TenantEntity`: Id, TenantId, auditoria):

| Campo | Tipo | Nota |
|-------|------|------|
| Name | string | Nombre del flujo ("Precios competencia Homecenter"). |
| Description | string? | Para que sirve. |
| StartUrl | string | URL de arranque (la del hero del prototipo). |
| Status | enum | Active / Inactive / Error (reusa el patron de `ScrapeSourceStatus`). |
| ClientId | Guid? -> DataClient | Que agente de la colmena lo ejecuta (el "bot asignado"). |
| ContainerId | Guid? -> DataContainer | Tabla destino por defecto de las extracciones del flujo. |

Programacion: **no lleva campos propios de horario**. Un flujo se programa creando un `ImportProcess`
que lo apunte (o generalizando `ImportProcess` para que su "objetivo" pueda ser un flujo, no solo un
conector). Asi hereda Cronos, reintento offline, timeout y la bitacora `ImportRun` sin duplicar nada.
Esta es la decision E2; el detalle de si `ImportProcess` gana un campo `FlowId` o se generaliza es de
construccion (Ola 1).

## 3. `ScrapeStep` (los pasos, ordenados)

Un paso tiene un **tipo** que decide que campos aplican. El tipo mapea a las acciones tipadas del
Navegador (`BrowserActionKind`: Navigate, Eval, Wait, Screenshot, Html, Mouse, Downloads) o al paso
de IA.

Campos comunes:

| Campo | Tipo | Nota |
|-------|------|------|
| FlowId | Guid -> ScrapeFlow | Dueno. |
| Order | int | Orden de ejecucion (el operador reordena arrastrando). |
| Kind | enum ScrapeStepKind | Ver abajo. |
| Name | string | Titulo del paso ("Login con credenciales"). |
| WaitMs | int? | Espera tras el paso (el `TIEMPO`/`ESPERA` del legacy). |

`enum ScrapeStepKind` (propuesto):

- **Navigate** -> `BrowserAction.Navigate`. Campos: `Url` (puede llevar `{{VAR}}`).
- **InjectScript** -> `BrowserAction.Eval`. Campos: `Script` (JS; el servidor lo FIRMA al ejecutar).
  No extrae por si solo; deja el resultado en el navegador (p.ej. `window.__extractedData`).
- **Extract** -> `Eval` + ingesta. Campos: `Script` (que devuelve filas) + **mapeo** campo->columna de
  la tabla destino + modo (Append/Replace/Upsert, reusando `ApiImportMode`). El resultado va por
  `IRowIngestService` al contenedor.
- **Wait** -> `BrowserAction.Wait`. Campos: `WaitMs` y/o `ConditionScript` (espera a que un JS
  devuelva true; el servidor tambien lo firma).
- **Click / Mouse** -> `BrowserAction.Mouse`. Campos: `Selector` o coordenadas (`ScriptJson`).
- **Screenshot** -> `BrowserAction.Screenshot`. Para diagnostico/evidencia.
- **AI** -> el paso de IA (ver s5). No es una accion tipada: es una orquestacion.

Nota: `Extract` es `InjectScript` + "esto SI se ingiere". Se separan porque no todo JS produce filas
(un login no extrae nada). El legacy resolvia esto con el `TIPO` de la accion (Tabla/Variable/Api/...);
aqui se simplifica a "extrae o no, y a donde".

Opcional (del legacy, backlog): **advertencias** por paso -una etiqueta que si aparece en el DOM
dispara `Notificar` o `Detener`- como una lista `ScrapeStepWarning`.

## 4. `ScrapeVariable` (variables cifradas)

Sustituciones tipo `{{USER}}`, `{{PASS}}`, `{{TOKEN}}` que el servidor inyecta en los scripts al armar
la orden. Campos:

| Campo | Tipo | Nota |
|-------|------|------|
| FlowId | Guid -> ScrapeFlow | Alcance por flujo (o por cliente, ver nota). |
| Name | string | El nombre que se usa como `{{Name}}` en el script. |
| ValueEncrypted | byte[]/string | Valor cifrado con `ISecretProtector`. NUNCA se devuelve en claro. |
| IsSecret | bool | Si es sensible (se pinta con candado, no se puede consultar). |

La sustitucion ocurre en el SERVIDOR, justo antes de firmar el script: el JS que viaja ya lleva el
valor real, cifrado en transito por el canal (y, en produccion, por HTTPS/WSS). Reusa exactamente el
patron de secreto del `DataClient` (el secreto que se muestra una vez y se rota, no se consulta).

## 5. Config del paso de IA

Un paso `Kind = AI` guarda (decision E3):

| Campo | Tipo | Nota |
|-------|------|------|
| Instruction | string | La orden en lenguaje natural ("saca la tabla de precios y ponla en Productos"). |
| TargetContainerId | Guid? | Tabla destino del resultado (o una variable, si es un dato suelto). |
| ToolAllowList | string[] | Que tools `browser.*` puede usar el agente (subconjunto de navigate/eval/wait/screenshot/html/mouse/downloads). Acota lo que puede hacer. |
| MaxSteps | int? | Tope de iteraciones del agente en este paso. |
| MaxSeconds | int? | Tope de tiempo. |
| AiProviderId / Model | ... | Proveedor y modelo, **entre los que el Super Admin habilite** (AI Provider Gateway, cupos por plan). |

El operador NO escribe JS en un paso de IA: describe el objetivo. El agente decide como, usando el MCP
local del Navegador, dentro de la allow-list de tools y el tope. La orquestacion (el bucle
agente<->navegador) es del runtime (doc 03), diferida.

## 6. Que se REUSA vs que es NUEVO

| Pieza | Estado | De donde |
|-------|--------|----------|
| Acciones tipadas del navegador (`BrowserAction`, 7 kinds) | **EXISTE** | Sub-agente Navegador (Contracts.Agent) |
| JS firmado (HMAC) + verificacion fail-closed | **EXISTE** | Sub-agente Navegador |
| Allow-list de dominios local | **EXISTE** | Sub-agente Navegador |
| Servidor MCP local `browser.*` (para el paso de IA) | **EXISTE** | Sub-agente Navegador |
| Nucleo de ingesta `IRowIngestService` | **EXISTE** | Contenedor de datos |
| Identidad del agente `DataClient` | **EXISTE** | Contenedor de datos |
| Programacion + bitacora (`ImportProcess`/`ImportRun`, Cronos) | **EXISTE** | Contenedor de datos (lo construimos esta semana) |
| Cifrado de variables `ISecretProtector` | **EXISTE** | Backbone |
| Proveedores/modelos de IA | **EXISTE** | AI Provider Gateway |
| `ScrapeFlow` / `ScrapeStep` / `ScrapeVariable` + CRUD + UI | **NUEVO** | Este modulo |
| Compilar flujo -> secuencia de `BrowserAction` + ingesta | **NUEVO** (runtime, diferido) | Este modulo (doc 03) |
| Orquestacion del paso de IA (agente<->navegador por MCP) | **NUEVO** (runtime, diferido) | Este modulo (doc 03) |

La lectura importante: **casi todo el musculo ya existe**. Lo nuevo es el MODELO DE FLUJO y la UI, mas
-despues- el pegamento del runtime. Por eso el alcance de ahora (la config) es acotado.

## 7. Relacion con `ScrapeSource`/`ScrapeRun` (lo que hay hoy)

`ScrapeSource` (URL + selector CSS, sin JS) es, en el modelo nuevo, un `ScrapeFlow` de un paso tipo
`Extract` sin JS y con ejecutor server-side en vez de agente. Dos caminos posibles (a decidir en Ola 1):

- **Absorber**: migrar cada `ScrapeSource` a un `ScrapeFlow` degenerado y retirar `IScrapeService`.
- **Coexistir**: dejar el scraper HTTP simple para URLs publicas triviales (no necesita agente) y usar
  los flujos para lo que requiere navegador real. Es la opcion mas barata a corto plazo.

Se recomienda **coexistir** al principio (no romper lo que funciona) y evaluar la absorcion cuando el
runtime de flujos este cableado.
