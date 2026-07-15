---
tipo: indice-proyecto
proyecto: Agente Conector On-Prem para Contenedor de datos
modulo_web: contenedor-datos (Sistema . Datos personalizados)
estado: especificacion (por construir). ALCANCE EN EXPANSION 2026-07-15 - de agente unico a colmena multi-agente (ver doc 06)
fecha: 2026-07-11 (actualizado 2026-07-15)
autor: documentado por agente IA a partir de decisiones del usuario
---

> [!important] ACTUALIZACION 2026-07-15 - lee primero el doc 06
> El usuario reabrio el **stack** (.NET 10 / C# / WPF / Windows-first) y **expandio el alcance**:
> de este agente unico (gateway de datos) a una **COLMENA de sub-agentes** (gateway + navegador
> web con JS inyectado + MCP + agente de archivos), orquestada por un servicio local. La duda
> "navegador en Linux -> Windows-only?" queda resuelta en
> [[06 - Decision de stack y expansion a colmena multi-agente]]. Los docs 01-05 de abajo siguen
> validos como la spec del **sub-agente Gateway de datos**.

# Agente Conector On-Prem - Contenedor de datos

> Capitulo nuevo. Especifica un **agente de escritorio (VB.NET + WPF, Windows)** que se
> instala en la red del cliente y **alimenta con datos** el modulo "Contenedor de datos"
> de ECOREX Tareas, disparado por los **horarios que se configuran en el sistema web**
> o por **refresco inmediato bajo demanda**. La comunicacion es por **WebSocket (SignalR)**.
> Documentacion auto-contenida para que un sub-agente pueda construirlo sin mas contexto.

## 1. En una frase

El sistema web (ECOREX) es el **cerebro** (guarda que traer, de donde y cada cuanto); el
agente on-prem es un **brazo tonto** que vive dentro de la red del cliente, mantiene una
conexion WebSocket saliente y, cuando el servidor se lo ordena (por horario o a mano),
ejecuta contra la fuente local (BD/API que el modelo tenga configurada como origen) la
consulta que el servidor le envia y **devuelve las filas**, que el servidor ingesta en la
tabla del contenedor.

## 2. Que YA existe (punto de partida real, en produccion)

El modulo web "Contenedor de datos" ya esta construido y desplegado (ver repo `EcorexV`,
rama `fase-0/clon-backbone`; PROGRESO.md Addenda 3-5). Lo que el agente REUSA:

- **`DataModel` (Contenedor)** con varias **`DataContainer` (tablas)** relacionadas.
- **`DataConnector`** por contenedor, con `Kind` = Excel / RestApi / **Database** (motor,
  host, puerto, BD, usuario, credenciales CIFRADAS con `ISecretProtector`).
- **`DataClient`** (ClientId publico + secreto cifrado): identidad de un agente/cliente
  remoto. **Se reusa para autenticar el WebSocket.**
- **`ImportProcess`** (horarios: Manual / Intervalo / Cron) por contenedor. **Es el job.**
- **`IApiImportService`** (motor de ingesta desde REST con paginacion + modos
  Append/Replace/Upsert). El agente reutiliza la MISMA logica de ingesta/upsert.
- Tablas de datos: `DataContainerRow` / `DataContainerCell` / `DataContainerLink`.

Lo NUEVO que agrega este proyecto: el **canal SignalR**, el **scheduler** que dispara los
procesos, el **conector de tipo "Agente" (pull desde BD local via agente)** y el **agente
WPF** en si.

## 3. Decisiones ya tomadas (con el usuario, 2026-07-11)

| # | Decision | Elegido |
|---|----------|---------|
| D1 | Transporte del canal | **SignalR sobre WebSocket** (reconexion/auth/framing gratis) |
| D2 | Donde vive la definicion de "que traer" | **En el servidor**; el cliente ejecuta lo que le mandan |
| D3 | Fuente de datos del agente | **La que el modelo tenga como origen** (conector Database/API del contenedor, ejecutado en la LAN del cliente) |
| D4 | Empaque del agente | **Servicio Windows headless** (conexion 24/7) **+ app WPF** de config/monitoreo |
| D5 | Auth del canal | **Reusa `DataClient`** (ClientId + secreto), handshake al conectar |
| D6 | Scheduling | **Job en el web** (`Ecorex.Workers`), el cliente es tonto respecto al horario |

## 4. Los documentos de este capitulo

| Doc | Contenido |
|-----|-----------|
| [[01 - Vision, arquitectura y decisiones]] | Actores, diagrama, prior-art (Doom WPF legacy), modelo de seguridad, casos de uso |
| [[02 - Protocolo SignalR (mensajes, handshake, secuencias)]] | Contrato del canal: metodos del hub, mensajes, handshake, secuencias (horario / refresco / offline / chunking) |
| [[03 - Servidor (Hub + Scheduler + Ingesta)]] | Implementacion servidor: Hub, `Ecorex.Workers` scheduler, conector tipo Agente, ingesta reusando el motor existente |
| [[04 - Cliente VB.NET WPF (Servicio Windows + config)]] | Arquitectura del agente: Servicio Windows + WPF, cliente SignalR .NET, ejecucion contra fuente local, empaque/instalador |
| [[05 - Plan de trabajo por olas (para sub-agente)]] | Backlog en olas con criterios de aceptacion; contrato de trabajo para el sub-agente |
| [[06 - Decision de stack y expansion a colmena multi-agente]] | **(2026-07-15)** Resuelve la duda de stack (.NET 10 / C# / WPF / navegador en Linux -> Windows-first, **D7 CONFIRMADA**) y encuadra la expansion a colmena: orquestador + sub-agentes (gateway / navegador+JS+MCP / archivos), GUI panal, identidad por ClientId (WhatsApp), seguridad de la superficie ampliada |
| [[07 - Prior-art Doom (orquestador drone + sub-agente navegador WebView2 + MCP)]] | **(2026-07-15)** Ingenieria inversa del sistema Doom (VB.NET 4.8) que el usuario ya construyo: orquestador `ucClienteDrone` que abre sub-agentes, transporte `DroneClientService` (SignalR), y sub-agente navegador `consumoweb_webserver` (WebView2 + inyeccion de JS + servidor MCP con 7 herramientas `browser.*`). Mapeo ORIGEN -> DESTINO |

## 5. Opinion de arquitectura (resumen)

El patron es estandar y solido: es un **agente on-prem / gateway hibrido** (mismo modelo
que Power BI Gateway o los conectores on-prem de Fivetran). El WebSocket saliente resuelve
el NAT/firewall (al cliente NO se le puede entrar por HTTP; el marca hacia afuera y el
servidor le empuja ordenes). El cliente tonto + job centralizado da una sola fuente de
verdad para los horarios. Ademas hay **prior-art propio**: el legacy ya tiene un runtime
WPF ("Doom") que ejecuta configuracion definida en la web (ver `NEWFRONT_web_scraping`);
este proyecto moderniza ese patron con push por SignalR y lo enchufa al Contenedor de datos.

## 6. Alcance v1 vs backlog

- **v1**: canal SignalR autenticado; conector tipo Agente sobre BD local (motor configurable
  el que soporte el agente); refresco inmediato bajo demanda; horario por Intervalo/Cron;
  ingesta con Append/Replace/Upsert; paginacion por consulta; agente como Servicio Windows +
  WPF de config; manejo de agente offline (marcar el proceso, reintentar).
- **Backlog**: streaming/chunking de datasets grandes; multiples fuentes por agente; fuentes
  de archivos (Excel/CSV en carpeta) y API local; panel de salud de agentes en el web;
  firma/verificacion de la consulta enviada; limites por plan (cupos de filas/frecuencia).

Relacionado en el repo: `docs/contenedor-datos-cliente-remoto.md` (contrato previo del cliente
remoto por webhook HMAC; este capitulo lo SUPERA con el canal SignalR bidireccional).
