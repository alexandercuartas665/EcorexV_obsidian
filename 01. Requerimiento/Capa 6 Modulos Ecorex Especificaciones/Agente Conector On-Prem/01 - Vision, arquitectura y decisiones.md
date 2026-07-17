---
tipo: spec-arquitectura
proyecto: Agente Conector On-Prem
proposito: Vision, actores, diagrama, prior-art, seguridad y casos de uso del agente on-prem que alimenta el Contenedor de datos por SignalR.
---

# 01 - Vision, arquitectura y decisiones

> [!note] Alcance ampliado 2026-07-15
> Este doc describe el **sub-agente Gateway de datos**, que sigue vigente. El proyecto se
> expandio a una **colmena de sub-agentes** (gateway + navegador + archivos) con orquestador
> local; ver [[06 - Decision de stack y expansion a colmena multi-agente]]. Este Gateway es una
> de las capacidades de esa colmena.

> [!success] CONSTRUIDO (actualizado 2026-07-16)
> La vision de este doc **se cumplio**: el agente conecta saliente, el servidor le empuja ordenes y las
> filas aterrizan en el contenedor. Verificado E2E contra SQL Server real de la LAN.
>
> El texto de abajo ya quedo corregido en lo que habia envejecido (decia "VB.NET + WPF"; el stack real
> es **C# + .NET 10**, D7). Lo unico que el diagrama de la s3 sigue simplificando: el agente son **dos
> piezas** (ADR-0039), un **Servicio** dueno de identidad/canal/boveda que corre aunque nadie inicie
> sesion, y la **colmena WPF que es su CLIENTE** y le presta el escritorio al Navegador. La colmena ya
> no configura por su cuenta: se lo pide al servicio, que exige administrador para aceptarlo. Detalle
> en [[04 - Cliente (Servicio Windows + colmena WPF)]].
>
> De la **s7 (politica de credencial)**: primero se implemento la **opcion (b)** (la credencial vive
> solo en el agente), pero con la Ola 4 se volvio a la **opcion (a)** (**ADR-0040, 2026-07-17**): la
> credencial de la fuente se configura y se cifra EN LA WEB y **viaja** en el `FetchRequest`. El motivo
> fue concreto: el usuario pidio administrar la conexion (host/usuario/clave) desde el modulo web, y
> con (b) esos campos quedaban muertos. La eleccion es **por conector** -si un conector no guarda
> credencial en la web, el agente cae a su cadena local (b)-, no global. **Consecuencia directa: el TLS
> estricto pasa de deseable a BLOQUEANTE** para produccion, porque ahora por el canal viaja la
> contrasena de la BD del cliente. `QueryGuard` (solo-SELECT) sigue siendo la defensa que impide
> escribir en la fuente, venga la credencial de donde venga.

## 1. Problema

El "Contenedor de datos" ya sabe importar desde Excel y desde APIs REST publicas
(`IApiImportService`, con paginacion y modos Append/Replace/Upsert). Pero muchos clientes
tienen sus datos en una **BD interna** (SQL Server u otro motor) o un servicio **detras de
su firewall**: el servidor de ECOREX no puede alcanzarlos por HTTP entrante. Se necesita un
**agente instalado en la red del cliente** que haga de puente, sin abrir puertos entrantes
en el cliente y sin exponer su BD a internet.

## 2. Actores

- **Servidor ECOREX (web)**: dueno de la configuracion (contenedores, conectores, horarios,
  credenciales cifradas) y de la ingesta. Corre el **Hub SignalR** y el **scheduler**.
- **Agente On-Prem (cliente)**: Windows, **C# / .NET 10**, instalado en la red del cliente. Son DOS
  piezas (ADR-0039): un **Servicio** headless (LocalSystem) que sostiene el canal aunque nadie inicie
  sesion, y la **colmena WPF** que lo muestra y le presta el escritorio al Navegador.
  Marca una conexion SignalR **saliente** al servidor y queda a la espera de ordenes. Es
  **tonto**: no sabe de horarios; solo ejecuta la consulta que le mandan contra la fuente
  local y devuelve filas.
- **Fuente local**: la BD/API dentro de la red del cliente (lo que el modelo tenga
  configurado como origen del conector).
- **Usuario/operador**: configura el contenedor, el conector y el horario en el web; puede
  pedir "refrescar ahora".

## 3. Diagrama (ASCII)

```
        RED DEL CLIENTE (detras de NAT/firewall)         |        NUBE (ECOREX)
                                                          |
  +------------------+        +---------------------+     |    +---------------------+
  |  Fuente local    |  SQL   |  AGENTE ON-PREM      |     |    |  Servidor ECOREX    |
  |  (SQL Server u   |<-------|  Servicio Windows    |==============  Hub SignalR     |
  |   otro motor/API)|  local |  (conexion saliente) |  WS  |  |  (/hubs/agente)     |
  +------------------+        |  + WPF de config     |     |    |                     |
                              +---------------------+     |    |  Scheduler          |
                                                          |    |  (Ecorex.Workers)   |
                                                          |    |  lee ImportProcess  |
                                                          |    |                     |
                                                          |    |  Ingesta -> tablas  |
                                                          |    |  del Contenedor     |
                                                          |    +---------------------+
   El agente ABRE la conexion hacia la nube (outbound 443).
   El servidor EMPUJA ordenes "FetchNow" por esa conexion ya abierta.
   NUNCA hay conexion entrante hacia el cliente.
```

## 4. Prior-art propio (no partimos de cero conceptualmente)

El legacy GestionMovil ya tiene este patron: el modulo web `NEWFRONT_web_scraping` (000730)
es un **configurador** que persiste en SQL un flujo, y un **runtime WPF (proyecto "Doom")**
lo ejecuta del lado cliente pesado (abre navegador, corre scripts, guarda en el contenedor
`GEN_DATAWARE`). Ver `[[NEWFRONT_web_scraping - Spec para reconstruir]]`.

Diferencias del nuevo diseno (moderniza el patron):
- Antes: el runtime WPF **hacia polling** / se lanzaba manual. Ahora: **push por SignalR**
  (el servidor ordena en tiempo real; soporta refresco inmediato).
- Antes: configuracion en tablas propias. Ahora: reusa `DataConnector`/`ImportProcess`/
  `DataClient` del Contenedor de datos ya construido.
- Antes: .NET Framework 4.8.1. Ahora: **.NET 10 + C#** (D7 confirmada; ver doc 04).

## 5. Modelo de "cliente tonto" (clave del diseno)

El agente NO contiene:
- Horarios (los tiene el scheduler del web).
- La consulta SQL / definicion del dataset (la manda el servidor en cada orden).
- Reglas de mapeo campo->columna (eso lo aplica el servidor al ingerir).

El agente SI contiene (minimo local):
- Su **identidad** (ClientId + secreto del `DataClient`, guardados cifrados con DPAPI local).
- La **URL del hub** del servidor.
- Opcional: un "permiso" local por seguridad (ver seccion 7): a que fuentes locales se le
  permite conectarse (whitelist de host/BD), para que un servidor comprometido no pueda
  pedirle conectarse a cualquier cosa de la LAN.

## 6. Casos de uso

- **UC1 - Carga por horario**: el scheduler dispara un `ImportProcess` (Intervalo/Cron) ->
  el servidor arma la orden `FetchRequest` (config de conexion + consulta) -> la empuja al
  agente conectado del tenant -> el agente ejecuta y responde `FetchResult(rows)` -> el
  servidor ingesta (Append/Replace/Upsert) en la tabla destino.
- **UC2 - Refresco inmediato**: el usuario (o una regla del sistema) pulsa "Refrescar ahora"
  en el contenedor -> misma orden `FetchRequest`, disparada a mano por el canal ya abierto.
- **UC3 - Agente offline**: si al disparar el horario el agente no esta conectado, el
  servidor marca el `ImportProcess.LastRunAt`/estado como "pendiente/fallido - agente
  offline" y reintenta segun politica; al reconectar el agente, puede ejecutarse el pendiente.
- **UC4 - Alta de agente**: se crea un `DataClient` en el web (genera ClientId + secreto una
  sola vez) -> se instala el agente y se le pega ese ClientId+secreto -> el agente conecta y
  el web lo muestra "en linea".

## 7. Seguridad (requisitos)

- **Handshake autenticado**: el agente presenta ClientId + prueba del secreto (HMAC de un
  nonce del servidor). El servidor valida contra `DataClient` (secreto cifrado). Sin auth
  valida, se rechaza la conexion. (Ver doc 02.)
- **Tenant scoping**: cada `DataClient` pertenece a un tenant; el agente solo puede recibir
  ordenes y devolver datos de SU tenant. El servidor NUNCA mezcla.
- **Credenciales de la fuente local**: viajan del servidor al agente SOLO cuando se dispara
  una orden, sobre el canal ya cifrado (WSS/TLS), y en el servidor estan cifradas
  (`ISecretProtector`). Alternativa mas segura (backlog): que la credencial de la fuente
  local viva SOLO en el agente y el servidor mande unicamente la consulta (evita enviar
  secretos hacia afuera).
- **Whitelist local en el agente**: el agente solo se conecta a host/puertos/BD que su config
  local permita, para que una orden maliciosa no lo convierta en un escaner de la LAN.
- **Consulta solo-lectura**: el agente debe rechazar sentencias que no sean SELECT (o usar
  un usuario de BD de solo lectura). El servidor tambien valida/parametriza.
- **Auditoria**: cada orden y resultado se registra (quien, cuando, cuantas filas, errores).
- **Transporte**: WSS (TLS) siempre; nunca ws:// en produccion.

## 8. No-objetivos (v1)

- No es un ETL bidireccional: el agente solo LEE de la fuente local y ENVIA al servidor (no
  escribe en la fuente del cliente).
- No ejecuta transformaciones complejas en el cliente (el mapeo/normalizacion es del servidor).
- No reemplaza el import por Excel ni el import por API REST publica (conviven).

## 9. Enlaces

- Protocolo detallado: [[02 - Protocolo SignalR (mensajes, handshake, secuencias)]]
- Servidor: [[03 - Servidor (Hub + Scheduler + Ingesta)]]
- Cliente: [[04 - Cliente (Servicio Windows + colmena WPF)]]
- Plan de construccion: [[05 - Plan de trabajo por olas (para sub-agente)]]
