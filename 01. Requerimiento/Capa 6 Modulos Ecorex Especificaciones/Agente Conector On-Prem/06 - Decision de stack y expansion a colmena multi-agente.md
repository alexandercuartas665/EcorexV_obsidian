---
tipo: decision-arquitectura
proyecto: Agente Conector On-Prem
proposito: Resolver la duda de stack (.NET 10 / WPF / navegador en Linux -> Windows-only vs multiplataforma) y encuadrar la EXPANSION de un agente unico (gateway de datos) a una COLMENA de sub-agentes orquestados. Los docs 01-05 quedan como la base del sub-agente Gateway.
fecha: 2026-07-15
estado: D7 CONFIRMADA 2026-07-15 (.NET 10 + C#, Windows-first, Linux diferido). Orden de sub-agentes confirmado (Gateway -> Archivos -> Navegador). Prior-art del orquestador+navegador minado en doc 07; pendiente prior-art del sub-agente Archivos
---

# 06 - Decision de stack y expansion a colmena multi-agente

> El usuario reabre (2026-07-15) dos cosas: (a) el **stack** del cliente (.NET 10? WPF? Windows
> solo o tambien Linux?), y (b) el **alcance**: de un agente unico (gateway de datos, docs
> 01-05) a un **enjambre de sub-agentes** orquestados tipo "colmena". Este doc resuelve la duda
> tecnica y encuadra la expansion. Los docs 01-05 NO se tiran: pasan a ser la spec del
> **sub-agente Gateway de datos**, que es una de las capacidades de la colmena.

## 1. La duda a resolver primero: navegador en Linux -> Windows-only o multiplataforma?

**Respuesta corta:** un navegador automatizado en Linux **SI es robusto** (Playwright/CDP,
headless); Linux es tecnicamente viable. Pero la recomendacion pragmatica es **Windows-first**,
y NO por debilidad de Linux, sino por costo/beneficio.

Detalle tecnico:

- **El motor de navegador en Linux es solido.** Playwright para .NET maneja Chromium/Firefox/
  WebKit en Linux; es el estandar de la industria (flotas de CI enormes corren asi). La
  **inyeccion de JS** (`AddInitScript`/`Evaluate`) es identica en todo SO.
- **Matiz importante**: **headless** en Linux es rock-solid; un navegador **visible (headful)**
  en Linux necesita `Xvfb` (framebuffer virtual) y tiene mas aristas de fuentes/render. Si el
  sub-agente solo **inyecta JS y hace acciones** (headless), Linux va perfecto. Si necesitas un
  navegador que el usuario **ve y opera**, es mas natural en Windows.
- **WPF es Windows-only** (incluso en .NET 10). La "colmena" en WPF => cliente Windows. Para
  multiplataforma habria que usar **Avalonia UI** (XAML tipo WPF, multiplataforma): opcion real
  y madura, pero mas trabajo.
- **El "Servicio Windows" es de Windows**, pero un **.NET Worker Service** corre como servicio
  Windows (`UseWindowsService`) **o** como demonio `systemd` en Linux (`UseSystemd`) desde el
  MISMO codigo. La capa de servicio SI puede ser multiplataforma.

**Recomendacion (D7):** **Windows-first en .NET 10 (C#)** - colmena en WPF, orquestador como
Worker Service, navegador via Playwright. Se arquitecta con piezas portables (Worker Service,
Playwright, capa de archivos abstraida) para que **Linux quede como add-on barato despues** si
aparece un cliente real Linux; **no se construye Linux ahora**. Esto responde tu intuicion: nos
quedamos en Windows y no metemos tanta tecnologia, pero sobre .NET moderno para que soportar
Linux mas adelante NO sea un reescribir.

## 2. Stack recomendado

| Pieza | Recomendado | Nota |
|---|---|---|
| Runtime | **.NET 10** | tu legacy es Framework 4.8; .NET 10 corrige perf/soporte |
| Lenguaje | **C#** | VB.NET corre en .NET 10, pero Playwright/MCP y los ejemplos son C#-first |
| Orquestador ("cerebro" local) | **Worker Service** (Windows Service; systemd-capable) | mantiene el WebSocket y lanza sub-agentes |
| GUI colmena | **WPF** (Windows) | Avalonia si algun dia se quiere multiplataforma |
| Canal | **SignalR/WebSocket** saliente | ya decidido (D1); sin puertos entrantes |
| Navegador | **Playwright .NET** (Chromium) | headless por defecto; headful opcional en Windows |
| Archivos | System.IO + capa OS abstraida | portable |
| Gateway BD | proveedores por motor (SqlClient/Npgsql/MySqlConnector/Oracle) | ya en [[04 - Cliente VB.NET WPF (Servicio Windows + config)|doc 04]] |

> Cambio vs docs previos: el doc 04 decia "VB.NET + .NET 8". La recomendacion actualizada es
> **.NET 10 + C#**. Se deja como D7 a confirmar antes de tocar el doc 04 a fondo.

## 3. Expansion de alcance: de agente unico a COLMENA de sub-agentes

El diseno pasa de "un agente gateway" a un **orquestador local ("colmena") que administra
sub-agentes efimeros por capacidad**. El servidor web (cerebro remoto) decide QUE capacidades y
acciones necesita; el orquestador local instancia el sub-agente adecuado para cada solicitud.

### 3.1 El orquestador (colmena)

- Servicio Windows (Worker Service) que mantiene la conexion **WebSocket saliente autenticada**
  (ClientId + secreto, como hoy). Es el UNICO con identidad/credenciales.
- Por cada solicitud del servidor, **instancia un sub-agente efimero** de la capacidad pedida;
  el sub-agente ejecuta y **se cierra**. Si llegan N solicitudes (p.ej. varios gateways) a la
  vez, abre **N sub-agentes en paralelo** -> atencion concurrente **sin cola de espera**.
- Los sub-agentes reciben solo la **orden acotada** (no las credenciales maestras).

### 3.2 Tipos de sub-agente (capacidades)

| Sub-agente | Que hace | Estado de la doc |
|---|---|---|
| **Gateway de datos** | Ejecuta consulta (solo lectura) contra BD/API de la LAN y devuelve filas; alimenta el Contenedor de datos | **YA especificado** (docs 01-05) |
| **Navegador web** | Abre paginas, **ejecuta JS inyectado desde el servidor** en la pagina, y expone **herramientas MCP** administradas por la app web para acciones controladas | **NUEVO** (pendiente spec + dirs de codigo) |
| **Archivos/Directorios** | Lee/escribe archivos y carpetas del equipo (Windows/Linux) segun tareas administradas por el servidor | **NUEVO** (pendiente spec + dirs de codigo) |

Es extensible: capacidades nuevas = sub-agentes nuevos, sin tocar el orquestador.

> **[CONSTRUIDO 2026-07-15] Sub-agente Navegador (base) - Nav-1/2/3.** `WebView2BrowserSubAgent`
> (Microsoft.Web.WebView2) en el agente ejecuta acciones TIPADAS del catalogo browser.* (doc 07):
> `Navigate`, `Eval` (JS inyectado), `Wait`, `Screenshot`, `Html`. Contrato aditivo en
> `Ecorex.Contracts.Agent` (`BrowserRequestMsg`/`BrowserResultMsg`/`BrowserAction`). **Seguridad
> (doc 06 s4)**: `BrowserAllowList` LOCAL cifrada con DPAPI gobierna a que dominios se navega y en
> cuales se permite inyectar JS -fail-closed si esta vacia; nada fuera de la lista aunque la nube lo
> pida. Cableado en `RealHiveConnection` (marshala al hilo UI) + `AgenteHub.BrowserResult` + endpoint
> dev. Verificado E2E: el servidor ordena navegar a example.com -> el agente abre WebView2, `Eval`
> `document.title` -> "Example Domain", captura el PNG real; navegar a un dominio NO permitido ->
> `Navigate ok=False` (bloqueado).
>
> **[CONSTRUIDO 2026-07-15] Nav-4 - servidor MCP + las 7 tools.** `BrowserMcpServer` (TcpListener
> **solo 127.0.0.1**, JSON-RPC 2.0: `initialize`/`tools/list`/`tools/call`) expone el catalogo
> completo del doc 07: `browser.navigate`, `browser.eval`, `browser.wait`, `browser.screenshot`,
> `browser.html`, `browser.mouse` (MouseBot: guion JSON click/type por selector), `browser.downloads`
> (historial). Comparte la MISMA instancia WebView2 que el hub y marshala al hilo de UI; respeta la
> allow-list. Devuelve contenido MCP (texto + imagen PNG). Verificado por JSON-RPC: `tools/list` = 7
> tools; `tools/call` navigate+eval+screenshot ok; navegar a un dominio no permitido -> `isError:true`.
> **Pendiente**: JS firmado/versionado por el servidor, UI de la allow-list en la colmena.

> **[CONSTRUIDO 2026-07-15] Sub-agente Archivos - Files-1/2/3.** Tercera capacidad de la colmena.
> `FileSubAgent` ejecuta acciones TIPADAS: `List`, `Read` (tope 1 MB), `Write`, `Delete`, `Exists`,
> `MakeDir`. Contrato aditivo en `Ecorex.Contracts.Agent` (`FileRequestMsg`/`FileResultMsg`/
> `FileAction`/`FileEntry`). **Seguridad (doc 06 s4)**: `FileAllowList` LOCAL cifrada con DPAPI define
> las rutas RAIZ permitidas; toda ruta se canonicaliza (impide traversal `..`) y debe caer DENTRO de
> una raiz -fail-closed si esta vacia; no es un shell generico. Cableado en `RealHiveConnection` +
> `AgenteHub.FileResult` + endpoint dev; y expuesto por el **servidor MCP** como `file.*` (6 tools),
> junto a las `browser.*` (`AgentMcpServer` publica 13 tools). Verificado E2E por el hub y por MCP:
> write/list/read dentro del sandbox ok; leer `C:\Windows\win.ini` (fuera de la allow-list) -> bloqueado.
> **Pendiente**: lectura de binarios (base64), UI de allow-list, read-only vs read-write por raiz.

### 3.3 GUI colmena (aspecto de panal)

- Cada **poligono** es un sub-agente.
- **Un poligono siempre lleno = Configuracion**: pegar el **ClientId** para conectar con el
  servidor (la app web), ver estado.
- A medida que se activan sub-agentes, los poligonos **se llenan**. Cuando el servidor pide una
  conexion (p.ej. un gateway), se **enciende** un poligono/sub-agente para atenderla; al
  terminar, se **apaga**. Multiples necesidades simultaneas = multiples poligonos en paralelo.
- La colmena es **monitoreo + configuracion**; la ejecucion la hace el orquestador (servicio).

> **[CONSTRUIDO 2026-07-15] Ola A - cascara visual (WPF).** Rama `feat/agente-colmena-gui` en el
> monorepo (`apps/agent/Ecorex.Agent.Gui` + contratos `libs/Ecorex.Contracts.Agent`). Ventana sin
> borde, translucida, monocroma, arrastrable, con tray icon. Panal de hexagonos (`HexTile` +
> `HoneycombPanel`) con estados Vacio/Lleno/Atendiendo(pulso)/Error; celda **Configuracion** siempre
> llena que abre un flyout (ClientId/URL/Estado/"Probar conexion" stub + persistencia **DPAPI**). El
> panal **crece/decrece** con workers efimeros. Seam `IHiveConnection` (mock en Ola A) listo para
> enchufar el **SignalR real en la Ola B sin tocar la GUI**. Compila y corre; capturas de 3 estados
> (idle / config / atendiendo). SIN SignalR real ni ejecucion de sub-agentes (olas siguientes).

> **[CONSTRUIDO 2026-07-15] Ola B - canal SignalR real (lado agente).** `RealHiveConnection`
> (Microsoft.AspNetCore.SignalR.Client) implementa `IHiveConnection` -la GUI y el ViewModel NO
> cambian, solo se sustituye el mock-. Conecta saliente al hub (`wss://.../hubs/agente`, doc 02),
> saluda con `AgentHello`, reconecta con backoff (0/2/5/10/30/60s) y traduce cada `FetchRequest`
> empujado por el servidor en el encendido de la capacidad + un worker efimero, cerrando el canal
> con un acuse `FetchResult` (la EJECUCION real es Ola C). El protocolo vive en
> `Ecorex.Contracts.Agent` (`AgentProtocol`, `AgentHubMethods`, DTOs) como fuente de verdad
> compartida con el futuro hub. Verificado END-TO-END contra `tools/Ecorex.Agent.HubSim` (simulador
> del backend, NO toca apps/backend): en los logs se ven `AgentHello` + round-trip `FetchRequest`/
> `FetchResult`, y en la colmena "En linea" con Gateway/Navegador encendiendose por ordenes reales.
> Arranque headless `--save-config <clientId> <hubUrl>` (DPAPI) para despliegue/servicio.

### 3.4 Identidad y cliente (via WhatsApp)

- La **app administrativa** carga el **ClientId** del cliente. Ese cliente es el que se conecta
  por **WhatsApp** a la app web; el ClientId ata el equipo on-prem a ese cliente/tenant.
- **Un ClientId = un tenant**: el orquestador solo atiende ordenes de su tenant (tenant scoping,
  igual que hoy en el handshake del gateway).

### 3.5 Alimentacion de contenedores por programacion

- El servidor tiene **programaciones listas** que disparan cargas del Contenedor de datos via el
  sub-agente Gateway. Es el **mismo scheduler central** (el cliente sigue tonto respecto a los
  horarios). Ver [[Programar actividad - Motor de programaciones (000889)]] y el `ImportProcess`
  del Contenedor de datos.

## 4. Seguridad de la superficie expandida (CRITICO)

La expansion convierte al agente en una **superficie de administracion remota** potente: ejecuta
JS venido de la nube en paginas, toca archivos locales y abre gateways de BD, todo orquestado
remotamente. Debe tratarse como tal (esto es lo que MAS hay que cuidar):

- **Allow-list por capacidad**: que dominios puede navegar el sub-agente navegador; que
  carpetas/rutas puede tocar el de archivos; que hosts/BD el gateway (ya esta en doc 01 s7).
  Nada fuera de la lista local, aunque la nube lo pida.
- **Sin ejecucion de comandos/shell arbitrarios**: cada capacidad expone **acciones tipadas**
  (herramientas MCP acotadas), NO un canal generico "ejecuta lo que te diga".
- **JS inyectado con limites**: idealmente **firmado/versionado** por el servidor y acotado a
  dominios permitidos; nunca JS arbitrario en cualquier pagina (banca, correo, etc.).
- **Least privilege**: el servicio corre con el minimo de permisos; el sub-agente de archivos
  solo accede a las rutas permitidas.
- **Consentimiento local explicito**: activar capacidades sensibles (archivos / navegador)
  exige que el operador lo habilite en la colmena, no basta que la nube lo pida.
- **Base ya definida** (aplica a TODOS los sub-agentes): handshake autenticado (HMAC del
  secreto), tenant scoping, WSS/TLS siempre, auditoria total (ver [[01 - Vision, arquitectura y decisiones]] s7).

Es **dual-use**: conectividad hibrida legitima (tipo Power BI Gateway / RPA on-prem), pero el
combo JS + archivos + BD + control-remoto obliga a guardrails duros y a que el dueno del equipo
mantenga el control.

## 5. Prior-art (tu sistema previo VB.NET 4.8)

- Ya construiste algo similar (Framework 4.8, VB.NET). El runtime WPF **"Doom"** del legacy (ver
  [[NEWFRONT_web_scraping - Spec para reconstruir]]) es el **ancestro del sub-agente navegador**
  (ejecuta configuracion definida en la web, abre navegador, corre scripts).
- **Pendiente de ti**: entregar los **directorios con ejemplos de codigo** del sistema 4.8. Con
  ellos se hace la ingenieria inversa de cada sub-agente (navegador, archivos, gateway) y se
  llenan las specs nuevas (07+).

## 6. Que decidir / que sigue

Decisiones CONFIRMADAS por el usuario (2026-07-15):

1. **D7 stack: CONFIRMADO** - **.NET 10 + C#**, Windows-first, Linux diferido.
2. **Orden de sub-agentes v1: CONFIRMADO** - **Gateway** (casi listo, docs 01-05) ->
   **Archivos** (acotado) -> **Navegador** (el mas complejo y riesgoso, ultimo).
3. **Colmena en WPF (Windows)**: implicito en D7 (Windows-first).

**Avance de construccion:**

- [x] **Ola A - cascara visual (WPF)** (2026-07-15): panal + estados + config (DPAPI) + tray + mock.
      Ver recuadro en 3.3. Rama `feat/agente-colmena-gui`.
- [x] **Ola B - cliente SignalR real (lado agente)** (2026-07-15): `RealHiveConnection` detras de
      `IHiveConnection` (GUI/VM sin cambios); protocolo compartido en `Ecorex.Contracts.Agent`
      (`AgentProtocol`/`AgentHubMethods` + DTOs de doc 02); AgentHello, reconexion con backoff,
      `FetchRequest`->colmena->acuse `FetchResult`. Probado E2E contra un **simulador de hub**
      (`tools/Ecorex.Agent.HubSim`, stand-in del backend, NO toca apps/backend): handshake + round-trip
      reales; la colmena enciende Gateway/Navegador con las ordenes empujadas. Ver recuadro en 3.3.
      El hub de produccion se construyo a continuacion (ver siguiente item).
- [x] **Hub real de servidor** (2026-07-15): `AgenteHub` `[Authorize]` + `IAgentRegistry` +
      `POST /api/agente/token` (HMAC del secreto de `DataClient` -> JWT) en `Ecorex.SuperAdmin`
      (esquema bearer "Agent" no-default; no toca la auth de cookies). Verificado E2E contra la BD
      dev: handshake rechaza invalidos, el agente conecta autenticado y responde a los `FetchRequest`
      del servidor. Detalle en doc 03 s9. **Pendiente**: `RunsViaAgent`+ingesta+scheduler+UI (doc 03).
- [~] **Ola C (Gateway ejecuta real)** (2026-07-15): el sub-agente Gateway ejecuta el `FetchRequest`
      contra SQL Server de la LAN (solo-lectura + whitelist `QueryGuard`, chunking), credencial local
      cifrada con DPAPI (no viaja). Verificado E2E contra una BD real (`M700_GEN`, tabla `ciudades`):
      20 filas reales de vuelta; no-SELECT rechazado. Detalle en doc 05 Ola 2. **Pendiente Ola C+**:
      sub-agentes Archivos y Navegador, mas motores de BD, allow-list avanzada, instalador/servicio.
- [x] **Ingesta en el servidor (doc 05 Ola 3)** (2026-07-15): el `FetchResult` del agente termina como
      filas en un contenedor reusando el motor EAV (`IRowIngestService` + `IAgentImportService`).
      Verificado E2E con `ciudades` (Replace/Upsert). Detalle en doc 03 s6/s9. **Pendiente**: scheduler
      (Ola 4) + `RunsViaAgent`/UI + migrar el import REST al nucleo compartido.
- [~] **Sub-agente Navegador (Nav-1/2/3/4)** (2026-07-15): WebView2 + las 7 acciones tipadas del
      catalogo browser.* (navigate/eval/wait/screenshot/html/mouse/downloads) + allow-list de dominios
      local (DPAPI) + **servidor MCP localhost** (JSON-RPC, `tools/list`/`tools/call`) para clientes/IA
      locales. Verificado E2E por el hub y por MCP (example.com ok; dominio no permitido -> bloqueado).
      Ver recuadros en 3.2. **Pendiente**: JS firmado/versionado por el servidor, UI de allow-list.
- [~] **Sub-agente Archivos (Files-1/2/3)** (2026-07-15): acciones tipadas List/Read/Write/Delete/
      Exists/MakeDir acotadas a la allow-list de rutas local (DPAPI, sin traversal) + expuesto por MCP
      (`file.*`). Verificado E2E por hub y por MCP (sandbox ok; fuera de la allow-list bloqueado). Ver
      recuadro en 3.2. **Pendiente**: binarios base64, UI de allow-list, read-only vs read-write.

Prior-art minado (2026-07-15): el usuario entrego el codigo del orquestador y del sub-agente
navegador del sistema Doom (VB.NET 4.8) -> documentado en
[[07 - Prior-art Doom (orquestador drone + sub-agente navegador WebView2 + MCP)]]. **Pendiente**:
el prior-art del **sub-agente Archivos** (dir de codigo, si existe).

Relacionado: [[00 - INDICE - Agente Conector On-Prem]], [[01 - Vision, arquitectura y decisiones]],
[[04 - Cliente VB.NET WPF (Servicio Windows + config)]],
[[Programar actividad - Motor de programaciones (000889)]],
[[NEWFRONT_web_scraping - Spec para reconstruir]].
