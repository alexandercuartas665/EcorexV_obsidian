---
tipo: ingenieria-inversa
proyecto: Agente Conector On-Prem
proposito: Ingenieria inversa del prior-art real (sistema Doom, VB.NET 4.8 / WPF) que el usuario ya construyo: el orquestador "drone", su capa de comunicacion SignalR y el sub-agente navegador (WebView2 + servidor MCP). Mapea ORIGEN -> DESTINO (.NET 10 C#). Solo documentacion; no se codea aqui.
fecha: 2026-07-15
origen: C:/Desarrollo/core/Doom/Modulos (proyecto Doom, .NET Framework 4.8, VB.NET + WPF)
estado: prior-art del orquestador + navegador minado; pendiente el del sub-agente Archivos
---

# 07 - Prior-art Doom: orquestador drone + sub-agente navegador (WebView2 + MCP)

> El usuario ya construyo un sistema muy parecido a la colmena en el proyecto **Doom**
> (VB.NET 4.8 / WPF). Este doc documenta ese codigo real como **referencia de migracion** hacia
> el destino .NET 10 C# (decidido en [[06 - Decision de stack y expansion a colmena multi-agente]]).
> Hallazgo clave: **el patron ya existe casi completo** - orquestador que abre sub-agentes,
> transporte SignalR y un sub-agente navegador con inyeccion de JS y un servidor MCP con 7
> herramientas. La metafora "drone" del legacy es la colmena.

## O1. Los 3 componentes que ya existen (mapa)

| Componente Doom (VB.NET 4.8) | Rol real | Sucesor en el destino (.NET 10 C#) |
|---|---|---|
| `Modulos/Cliente/ucClienteDrone.xaml(.vb)` | UI + logica del **orquestador**: conecta, recibe instrucciones y **abre sub-agentes** | Orquestador (colmena) / Worker Service + GUI panal |
| `Modulos/Cliente/DroneClientService.vb` | **Transporte**: cliente **SignalR** (HubConnection), eventos, protocolo de instruccion/respuesta | Cliente SignalR del orquestador |
| `Modulos/Servicios/consumoweb_webserver.xaml(.vb)` | **Sub-agente Navegador**: WebView2 + inyeccion de JS + **servidor MCP** + motor de pasos/acciones | Sub-agente Navegador (capacidad de la colmena) |

## O2. Transporte: `DroneClientService` (ya es SignalR)

Confirma la decision D1 (SignalR/WebSocket): el legacy **ya usa SignalR**.

- `Imports Microsoft.AspNet.SignalR.Client`; `Private _connection As HubConnection`,
  `_hubProxy`. Conexion **saliente** a `serverUrl` con un `droneId`.
- Metodos que el servidor EMPUJA al cliente (server -> drone), via `_hubProxy.On(...)`:
  - `recibirInstruccion(droneId, instruccion, parametros, correlationId, timestamp)`
  - `registroConfirmado(droneId, connectionId, timestamp)`
  - `actualizarEstado(...)`, `droneConectado(...)`, `droneDesconectado(...)`
- Metodos que el cliente INVOCA (drone -> server): `ConectarAsync(serverUrl, droneId)`,
  `EnviarEstadoAsync(statusJson)`, `ResponderInstruccionAsync(correlationId, respuestaJson)`.
- **Protocolo de correlacion** (clave): `correlationId` vacio = instruccion
  **fire-and-forget** (no requiere respuesta); `correlationId` con valor = el drone **debe
  responder** con `ResponderInstruccionAsync`. Es un request/response sobre el canal push.
- Reconexion automatica (reintenta `ConectarAsync`).

Migracion: el destino usa `Microsoft.AspNetCore.SignalR.Client` (misma idea, API moderna). El
handshake autenticado por ClientId+secreto (doc 01 s7) se agrega encima; el legacy conecta por
`droneId` sin el HMAC (deuda a corregir, ver O6).

## O3. Orquestador: `ucClienteDrone` (abre sub-agentes por instruccion)

Este es el **modelo colmena** ya implementado:

- Usa `DroneClientService.GetInstance()` (singleton) y se suscribe a sus eventos
  (`InstruccionRecibida`, `RegistroConfirmado`, `EstadoDroneActualizado`, `DroneConectado`...).
- **Mantiene los sub-agentes abiertos** en `ventanasAbiertas As Dictionary(Of String,
  consumoweb_webserver)` (clave = tramite+token).
- **`AbrirVentana(tramiteId, token, parametros, url, paso, baseSistema, tokenUsuario)`**: al
  recibir una instruccion, **abre (o reutiliza) un `consumoweb_webserver`** para atenderla ->
  es el "instanciar un sub-agente efimero por solicitud" de la colmena. Varias instrucciones =
  varias ventanas = atencion en paralelo sin cola.
- El orquestador guarda el `correlationId` pendiente y responde cuando el sub-agente termina.

Migracion: el destino formaliza esto como **orquestador (Worker Service)** que instancia
sub-agentes por capacidad; el `Dictionary(clave -> ventana)` pasa a un **registro de
sub-agentes activos** (los poligonos que se llenan en la GUI panal). La GUI Windows/WPF hereda
directo de este UserControl.

## O4. Sub-agente Navegador: `consumoweb_webserver` (WebView2 + JS + MCP)

Es el ancestro directo del **sub-agente Navegador** del destino. Capacidades reales:

- **Motor de navegador**: `Microsoft.Web.WebView2.Core` (Edge/Chromium embebido). `Navigate`,
  `NavigationStarting/Completed`, manejo de descargas (`DescargaArchivos` ->
  `Directory.CreateDirectory` en temp).
- **Inyeccion / ejecucion de JS**: `wBrowser.CoreWebView2.ExecuteScriptAsync(script)` devuelve
  el resultado. Helper `WebView2Helper.ExecuteScriptAwaitGlobal(...)` para esperar una variable
  global. Esto es exactamente "JS que viene del servidor y se inyecta en la pagina".
- **Servidor MCP embebido** (`#Region "MCP navegador"`): un `HttpListener` que responde JSON
  MCP. **Seguridad ya presente**: solo acepta **localhost** (`"Acceso MCP denegado: solo se
  acepta localhost."`, HTTP 403). `GetMcpTools()` publica el catalogo; `ExecuteMcpToolAsync`
  despacha; `browser.status` reporta url/titulo/descargas.
- **Catalogo de 7 herramientas MCP ya definidas** (insumo directo del destino):

  | Herramienta | Que hace | Input |
  |---|---|---|
  | `browser.navigate` | Navega a una URL http/https | url, timeoutMs, screenshot |
  | `browser.eval` | Ejecuta JavaScript en la pagina y devuelve el resultado | script, waitMs, screenshot |
  | `browser.wait` | Espera ms o hasta que una condicion JS sea truthy | waitMs, conditionScript, screenshot |
  | `browser.screenshot` | Captura el navegador (PNG base64) | - |
  | `browser.downloads` | Historial reciente de descargas | - |
  | `browser.mouse` | Ejecuta un script JSON del "MouseBot" | scriptJson, screenshot |
  | `browser.html` | HTML de la pagina o de un selector CSS | selector, screenshot |

- **Motor de pasos/acciones** guiado por SQL: `IraPaso(paso)`, `IraAcciones(tabla, ..., numeroAccion, ...)`
  leen configuracion de las tablas `WEB_SCRAPING` / `WEB_SCRAPING_CLI` (ciclo, nombre, lider,
  cuenta Slack) y ejecutan la secuencia. Es el mismo patron "config en la web, runtime en el
  cliente" de [[NEWFRONT_web_scraping - Spec para reconstruir]].
- **Canales**: reporta a Slack (`PRO_SLACK`), log con colores (`InsertarLog`).

Migracion (destino): las **7 herramientas `browser.*` son el catalogo de partida** del
sub-agente Navegador; el servidor MCP localhost-only se conserva (buena practica) y se
**endurece** con las reglas del doc 06 s4 (allow-list de dominios, JS firmado/acotado,
consentimiento local). El motor de pasos por SQL (`WEB_SCRAPING*`) se reencuadra a la
configuracion en la nube que dispara acciones via el canal SignalR.

Decision de motor del navegador (refina doc 06 s2): el prior-art usa **WebView2** (navegador
**visible** en Windows, con MouseBot + MCP ya hechos). Para el destino Windows-first y navegador
visible, **WebView2 es la continuacion natural** (maximo reuso). **Playwright** queda como
alternativa si mas adelante se necesita **headless/portable** (ver doc 06 s1).

## O5. La metafora "drone" == colmena (ya existia)

El legacy llama "drone" a cada sub-agente y el orquestador (`ucClienteDrone`) los enciende bajo
demanda. Es literalmente la colmena que describio el usuario: cada drone es un poligono; el
orquestador abre uno por necesidad y lo cierra al terminar.

## O6. Deudas del origen a NO heredar

- **Handshake debil**: el drone conecta por `droneId` sin prueba de secreto (sin HMAC). El
  destino exige handshake autenticado (ClientId + HMAC del secreto, doc 01 s7).
- **SQL por concatenacion** en el motor de pasos (`WEB_SCRAPING*`) -> EF/consulta parametrizada
  o config servida por la nube (no SQL crudo en el cliente).
- **JS sin allow-list**: `ExecuteScriptAsync` corre lo que llegue; el destino acota dominios y
  (ideal) firma/veriona el JS del servidor.
- **Acoplado a Windows** (WebView2 + WPF): consistente con Windows-first (D7), pero se aisla la
  capa de navegador para poder cambiar a Playwright si se necesita headless/Linux.

## O7. Mapeo resumido ORIGEN -> DESTINO

| ORIGEN (Doom 4.8 VB.NET) | DESTINO (.NET 10 C#) |
|---|---|
| `DroneClientService` (SignalR client) | Cliente SignalR del orquestador + handshake HMAC |
| `ucClienteDrone` + `ventanasAbiertas` | Orquestador (Worker Service) + registro de sub-agentes + GUI panal (WPF) |
| `consumoweb_webserver` (WebView2 + MCP) | Sub-agente Navegador (WebView2, 7 tools MCP, endurecido) |
| Motor de pasos por `WEB_SCRAPING*` (SQL) | Acciones configuradas en la nube, disparadas por SignalR |
| Reporte a Slack + log | Notification Service + auditoria del destino |

## O8. Pendiente

- **Sub-agente Archivos**: falta el dir de codigo del prior-art (si existe) para minarlo igual.
- **Sub-agente Gateway de datos**: su prior-art ya esta encuadrado en los docs 01-05 (patron
  Doom + `NEWFRONT_web_scraping`); este doc no lo repite.

Relacionado: [[06 - Decision de stack y expansion a colmena multi-agente]],
[[01 - Vision, arquitectura y decisiones]], [[02 - Protocolo SignalR (mensajes, handshake, secuencias)]],
[[04 - Cliente (Servicio Windows + colmena WPF)]],
[[NEWFRONT_web_scraping - Spec para reconstruir]].
