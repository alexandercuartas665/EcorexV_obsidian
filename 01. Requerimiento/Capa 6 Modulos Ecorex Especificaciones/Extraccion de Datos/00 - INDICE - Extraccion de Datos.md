---
tipo: indice-proyecto
proyecto: Extraccion de Datos (automatizacion de navegador)
modulo_web: extraccion-datos (Sistema . General)
modulo_legacy: 000730 - NEWFRONT_web_scraping
estado: CONSTRUIDO (Olas 1-5) + E2E del runtime determinista. El modulo /extraccion-datos evoluciono de scraper HTTP simple (ADR-0025) a configurador de flujos ejecutados por el sub-agente Navegador; config + runtime determinista + paso de IA + programacion/paginacion/advertencias hechos y probados hasta donde el entorno de dev permite (sin colmena WebView2 ni proveedor de IA con llave). Ver [[05 - Estado de construccion y E2E]].
fecha: 2026-07-17 (actualizado 2026-07-18)
autor: documentado por agente IA a partir de decisiones del usuario
---

# Extraccion de Datos - Configurador de automatizaciones del navegador

> Especifica el modulo web **"Extraccion de datos"** (000730, antes `NEWFRONT_web_scraping`) como el
> **configurador** de flujos de automatizacion de navegador -un scraper avanzado-, cuyo **runtime es el
> sub-agente Navegador de la colmena** (ver capitulo [[00 - INDICE - Agente Conector On-Prem|Agente
> Conector On-Prem]]). Los datos que se extraen **aterrizan en un Contenedor de datos**, y los flujos
> se programan reusando la misma infraestructura de horarios y bitacora que ya construimos alli.

## 1. En una frase

Desde este modulo se **parametrizan las acciones del navegador** de un agente de la colmena: un flujo
es una lista ordenada de **pasos** que le dicen al navegador a que pagina ir, que JavaScript inyectar
para sacar datos, y -cuando la pagina es dificil- que le pida a un **agente de IA** que la resuelva
conversando con ese mismo navegador. El resultado se ingiere en una tabla de un contenedor. El sistema
web es el cerebro (guarda el flujo, firma el JS, programa y ingiere); la colmena es el brazo que
maneja el navegador dentro de la red del cliente.

## 2. Que hay HOY (linea base honesta)

**El modulo ya existe y funciona**, pero es una version simple. No se parte de cero:

- Pagina real `Ecorex.SuperAdmin/Components/Pages/ExtraccionDatos.razor` (`@page "/extraccion-datos"`,
  policy `ExtraccionDatos.Editar`), en el grupo de menu **Sistema . General**.
- Entidades `ScrapeSource` / `ScrapeRun` (ADR-0025) + `IScrapeService`.
- **Ejecutor server-side ACOTADO**: solo `GET` a URLs publicas http(s), guard anti-SSRF (rechaza IP
  privadas/loopback/puertos no estandar), timeout 15s, respuesta max 2 MB. Extrae HTML por selector
  CSS o cuenta/previsualiza JSON. **NO ejecuta ningun script.** La programacion automatica quedo
  pendiente en el propio ADR-0025.

En corto: hoy hay un **scraper HTTP de un solo paso, sin navegador real y sin JS**. Este capitulo lo
lleva a un **configurador de flujos multi-paso** cuyo motor es el navegador de verdad (WebView2) del
sub-agente. El scraper actual pasa a ser un caso particular (un flujo de un paso, sin JS).

## 3. Decisiones tomadas con el usuario (2026-07-17)

| # | Decision | Elegido |
|---|----------|---------|
| E1 | Donde aterrizan los datos extraidos | **En un Contenedor de datos** (el navegador es otra FUENTE que alimenta una tabla, como el Gateway y el API REST). Reusa `IRowIngestService`. |
| E2 | Programacion y bitacora de los flujos | **Reusar lo del Contenedor de datos**: `ImportProcess`/`ImportRun` con Cronos, reintento offline, timeout. Un flujo se dispara igual que un refresco: horario o manual, y deja su corrida en una bitacora identica. |
| E3 | Que configura un **paso de IA** | Instruccion en lenguaje natural + **contenedor/variable destino** + **allow-list de tools MCP** (que `browser.*` puede usar) + **tope de pasos/tiempo** + **proveedor/modelo de IA** de entre los que **el Super Admin habilite** (reusa el AI Provider Gateway y sus cupos por plan). |
| E4 | El runtime | **El sub-agente Navegador de la colmena** (WebView2 + acciones tipadas + JS FIRMADO por el servidor + allow-list de dominios + MCP), NO el runtime WPF "Doom" del legacy. |

## 4. Los documentos de este capitulo

| Doc | Contenido |
|-----|-----------|
| [[01 - Vision, arquitectura y decisiones (Extraccion)]] | Problema, actores, como encaja con la colmena y el Contenedor de datos, que hay hoy vs objetivo, principios de seguridad heredados |
| [[02 - Modelo de configuracion (flujo, pasos, variables)]] | El modelo de dominio a construir: `ScrapeFlow` / `ScrapeStep` / `ScrapeVariable`, tipos de paso, mapeo a `BrowserAction`, que se reusa vs que es nuevo |
| [[03 - Runtime - el sub-agente Navegador ejecuta el flujo]] | Contrato de ejecucion (DIFERIDO; el alcance ahora es la config): como un flujo se compila a una secuencia de acciones + el paso de IA por MCP; gobierno (JS firmado, allow-list, cifrado, bitacora) |
| [[04 - Plan de trabajo por olas (Extraccion)]] | Backlog en olas con criterios de aceptacion (Olas 1-5 marcadas HECHAS) |
| [[05 - Estado de construccion y E2E]] | **Estado REAL**: que se construyo ola por ola, el E2E del runtime determinista (agente stand-in por SignalR: auth + firma + ingesta + bitacora), pruebas, y lo que falta (colmena WebView2 + proveedor IA) |
| [[NEWFRONT_web_scraping - Spec para reconstruir]] | **Prior-art**: la spec del modulo legacy `NEWFRONT_web_scraping` (000730) tal cual estaba en WebForms + runtime WPF "Doom". Referencia del modelo de datos y los tipos de paso originales |

## 5. Relacion con el resto del ecosistema

- **[[00 - INDICE - Agente Conector On-Prem|Agente Conector On-Prem]]**: aporta el RUNTIME (el
  sub-agente Navegador, con sus 7 acciones tipadas, el JS firmado, la allow-list de dominios y el
  servidor MCP local). Este modulo es el CONFIGURADOR de ese runtime, del mismo modo que el Contenedor
  de datos configura el Gateway.
- **Contenedor de datos**: aporta el DESTINO (tablas) y el nucleo de ingesta, y el patron de
  **programaciones + bitacora** que este modulo reusa tal cual.
- **AI Provider Gateway** (backbone): aporta los proveedores/modelos de IA para el paso de IA, con
  cupos por plan; el operador elige entre los que el Super Admin habilite.

## 6. Alcance v1 vs backlog

- **v1 (config)**: modelar el flujo (maestro + pasos + variables cifradas), la UI del configurador
  milimetrica al prototipo `proto_web_scraping.html`, el destino en un contenedor y la programacion
  reusada. Pasos deterministas (navegar + inyectar JS + extraer) y el esqueleto del paso de IA.
- **backlog**: cablear el runtime completo (compilar flujo -> secuencia de acciones al Navegador),
  la orquestacion real del paso de IA (bucle agente <-> navegador por MCP), paginacion controlada,
  advertencias (etiqueta -> notificar/detener), historico rico y panel de salud.
