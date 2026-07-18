---
tipo: spec-arquitectura
proyecto: Extraccion de Datos
proposito: Vision, actores, arquitectura y decisiones del configurador de flujos de automatizacion de navegador cuyo runtime es el sub-agente Navegador de la colmena.
---

# 01 - Vision, arquitectura y decisiones

## 1. Problema

El modulo "Contenedor de datos" ya sabe traer datos de tres fuentes: Excel, API REST publica y base de
datos (via el agente Gateway). Pero muchos datos solo existen **detras de una pagina web que hay que
operar como un humano**: iniciar sesion, esperar a que un grid cargue por JavaScript, paginar haciendo
clic, o -peor- resolver una pagina cuyo layout cambia y un selector fijo no aguanta.

El scraper actual del modulo (ADR-0025) NO alcanza esos casos: hace un `GET` y lee HTML/JSON, sin
navegador real y **sin ejecutar JavaScript**. Se necesita un motor que **maneje un navegador de
verdad** dentro de la red del cliente, y un **configurador** desde el cual el operador defina, sin
programar en el servidor, la secuencia de acciones a ejecutar.

Ese motor **ya existe**: es el sub-agente Navegador de la colmena (WebView2, acciones tipadas, JS
firmado por el servidor, allow-list de dominios, servidor MCP local). Lo que falta es el **puente**:
un modelo de "flujo" en el servidor y la UI para disenarlo. Este modulo es ese puente.

## 2. Que hay HOY vs a donde va

**Hoy** (ADR-0025, en produccion):

```
[Operador] --define--> ScrapeSource (URL + selector CSS)
                          |
                          v
                    IScrapeService (server-side)
                    GET publico + guard SSRF, 15s, 2MB, sin JS
                          |
                          v
                    ScrapeRun (Success/Failed)  +  preview
```

**Objetivo** (este capitulo):

```
[Operador] --disena--> ScrapeFlow (maestro)
                          |  pasos ordenados
                          v
              ScrapeStep[]  (Navegar / Inyectar JS / Extraer / Esperar / IA)
                          |  el servidor FIRMA el JS y arma la orden
                          v
                    Hub SignalR  --BrowserRequest-->  Sub-agente Navegador (colmena, en la LAN)
                                                          WebView2 + allow-list + JS firmado
                          |                                   |  (paso IA: agente <-> navegador por MCP)
                          |<----- BrowserResult / filas -------|
                          v
                    IRowIngestService --> tabla de un Contenedor de datos
                          + ImportRun (bitacora, reusada)
```

El scraper HTTP de hoy queda como un **caso degenerado** del modelo nuevo: un flujo de un paso, tipo
"Extraer", sin JS. La migracion (absorberlo o dejarlo coexistir) es una decision de construccion, no
de ahora.

## 3. Actores

- **Operador**: disena el flujo desde la web (pasos, scripts, variables, destino, horario). No toca
  codigo del servidor.
- **Servidor ECOREX (web)**: dueno del flujo. **Firma** cada script antes de mandarlo (HMAC del secreto
  del `DataClient`), programa las corridas, e **ingiere** el resultado en el contenedor. Tambien
  orquesta el paso de IA contra el AI Provider Gateway.
- **Sub-agente Navegador (colmena)**: el RUNTIME. Corre en la red del cliente, abre el navegador
  (WebView2), ejecuta la secuencia de acciones tipadas, respeta la allow-list de dominios local y
  verifica la firma del JS (fail-closed). Publica un servidor MCP local por el que un agente de IA
  puede manejarlo.
- **Contenedor de datos**: el DESTINO. Cada extraccion aterriza como filas de una de sus tablas.
- **DataClient**: la identidad del agente que ejecuta el flujo (el "bot asignado" del prototipo). Es
  la misma identidad que ya usa el Gateway.
- **AI Provider Gateway** (backbone): provee el modelo de IA del paso de IA, entre los que el Super
  Admin habilite, con cupos por plan.

## 4. Los dos tipos de paso (el corazon del diseno)

Un flujo mezcla dos clases de paso, y elegir bien entre ellas es la decision de diseno de cada flujo:

- **Paso DETERMINISTA** (barato, exacto): "ve a esta URL", "inyecta este JS y saca estos datos",
  "espera a que aparezca este selector". Se traduce 1:1 a las acciones tipadas del Navegador
  (`Navigate`, `Eval`, `Wait`, `Html`, `Screenshot`, `Mouse`, `Downloads`). El JS lo escribe el
  operador y el servidor lo FIRMA. Ideal cuando el sitio es estable y el selector aguanta.
- **Paso de IA** (flexible, cuesta tokens): "de esta pagina, sacame la tabla de precios y ponla en el
  contenedor X". Un agente de IA maneja el navegador por el MCP local (`browser.*`), razonando sobre
  lo que ve, acotado por una **allow-list de tools**, un **tope de pasos/tiempo** y el **modelo** que
  el operador eligio. Ideal cuando el layout cambia o hace falta "entender" la pagina.

Un mismo flujo puede empezar determinista (login con credenciales cifradas) y terminar con un paso de
IA (extraer lo que sea que haya cargado). Esa mezcla es la propuesta de valor.

## 5. Principios de seguridad (heredados, y por que importan)

El modulo legacy tenia un agujero grave: el JS almacenado se ejecutaba tal cual en el navegador del
bot, lo que equivalia a **RCE** para cualquier operador con acceso al configurador (ver la spec legacy,
s12). El diseno nuevo lo cierra por construccion, reusando lo del capitulo del Agente:

- **JS FIRMADO por el servidor**: cada script de un paso viaja con un HMAC (`correlationId|payload`) que
  el agente verifica en tiempo constante y **fail-closed**. Un flujo manipulado en transito, o un
  agente al que alguien le empuje JS sin firma, no ejecuta nada.
- **Allow-list de dominios local**: el agente solo navega a los dominios que su config local permite.
  Un flujo no puede convertir al agente en un escaner de la web ni de la LAN.
- **Credenciales cifradas**: las variables sensibles (usuario/clave/token) se cifran con
  `ISecretProtector` y se sustituyen en el script (`{{USER}}`) en el momento de armar la orden; no se
  guardan en claro ni viajan en el flujo.
- **SQL parametrizado / ingesta por el nucleo**: los datos entran por `IRowIngestService`, el mismo
  camino de escritura del Contenedor de datos; nada de concatenar SQL (otro pecado del legacy, s12).
- **Bitacora**: cada corrida deja registro (reusando `ImportRun`), cosa que el legacy no tenia.

## 6. Decisiones

Ver la tabla E1-E4 del [[00 - INDICE - Extraccion de Datos|indice]]. En resumen:

- **E1 - destino = Contenedor de datos**: el navegador es una fuente mas; unifica el "donde viven los
  datos" y reusa la ingesta.
- **E2 - programacion = la del Contenedor de datos**: `ImportProcess`/`ImportRun` con Cronos, reintento
  offline y timeout. No se reconstruye un scheduler propio.
- **E3 - paso de IA**: instruccion + contenedor/variable destino + allow-list de tools MCP + tope de
  pasos/tiempo + proveedor/modelo entre los que habilite el Super Admin.
- **E4 - runtime = sub-agente Navegador**, no el WPF "Doom" del legacy.

## 7. No-objetivos (v1)

- No se construye el runtime nuevo: el runtime **ya existe** (el sub-agente Navegador). v1 es el
  CONFIGURADOR y el modelo de datos.
- No se reemplaza aun el scraper HTTP simple (ADR-0025): convive; la absorcion se decide despues.
- No se ejecuta scraping desde el servidor: el navegador vive en la LAN del cliente (por eso hace falta
  el agente).

## 8. Prior-art

- **Legacy**: [[NEWFRONT_web_scraping - Spec para reconstruir]] - el mismo modulo en WebForms + runtime
  WPF "Doom" (WebBrowser/Trident). Aporta el modelo de datos original (10 tablas `WEB_SCRAPING_*`) y
  los tipos de paso/accion, utiles para no reinventar el vocabulario.
- **Runtime nuevo**: capitulo [[00 - INDICE - Agente Conector On-Prem|Agente Conector On-Prem]], docs
  del sub-agente Navegador (acciones tipadas, JS firmado, allow-list, MCP).
- **Prototipo visual**: `01. Requerimiento/Prototipo/conceptos-claude-design/proto_web_scraping.html`
  (acento morado; lista de configuraciones, hero con KPIs, pasos con editor de JS, variables cifradas,
  historico de corridas).
