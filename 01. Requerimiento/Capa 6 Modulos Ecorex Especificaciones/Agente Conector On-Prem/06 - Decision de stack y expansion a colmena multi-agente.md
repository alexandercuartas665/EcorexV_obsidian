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
      Ver recuadro en 3.3. Rama `feat/agente-colmena-gui` (sin merge/push a compartida aun).
- [ ] **Ola B**: cliente SignalR real detras de `IHiveConnection` (sin tocar GUI/VM).
- [ ] **Ola C+**: ejecucion de sub-agentes (Gateway -> Archivos -> Navegador), allow-list de
      seguridad, instalador/servicio Windows.

Prior-art minado (2026-07-15): el usuario entrego el codigo del orquestador y del sub-agente
navegador del sistema Doom (VB.NET 4.8) -> documentado en
[[07 - Prior-art Doom (orquestador drone + sub-agente navegador WebView2 + MCP)]]. **Pendiente**:
el prior-art del **sub-agente Archivos** (dir de codigo, si existe).

Relacionado: [[00 - INDICE - Agente Conector On-Prem]], [[01 - Vision, arquitectura y decisiones]],
[[04 - Cliente VB.NET WPF (Servicio Windows + config)]],
[[Programar actividad - Motor de programaciones (000889)]],
[[NEWFRONT_web_scraping - Spec para reconstruir]].
