---
tipo: spec-protocolo
proyecto: Agente Conector On-Prem
proposito: Contrato del canal SignalR entre servidor y agente: endpoint, handshake/auth, metodos del hub, mensajes (con forma JSON), y secuencias (horario, refresco, offline, chunking).
---

# 02 - Protocolo SignalR (mensajes, handshake, secuencias)

> Todo el canal es **SignalR sobre WebSocket seguro (WSS/TLS)**. El agente es el que
> **inicia** la conexion (saliente). El servidor **empuja** ordenes por esa conexion.

## 1. Endpoint

- Hub: `POST/WS  https://<host>/hubs/agente` (mapeado en el servidor, ver doc 03).
- Transporte forzado: WebSockets (fallback a Server-Sent Events solo si hace falta).
- Reconexion automatica: activada en el cliente SignalR (backoff: 0s, 2s, 5s, 10s, 30s, luego cada 60s).
- Keep-alive: el default de SignalR (KeepAlive 15s, ClientTimeout 30s) o ajustado.

## 2. Identidad y autenticacion (handshake)

El agente se autentica con las credenciales del `DataClient` (ClientId publico + secreto).
NO se manda el secreto en claro: se prueba con HMAC de un nonce.

Flujo de handshake (dos opciones; elegir A para v1 por simplicidad):

**Opcion A - token de acceso en la conexion (recomendada v1)**
1. El agente pide un token corto a un endpoint REST previo:
   `POST /api/agente/token` con `{ clientId, ts, nonce, hmac }` donde
   `hmac = HMACSHA256(secret, clientId + "|" + ts + "|" + nonce)`.
   El servidor valida (secreto de `DataClient`, ts dentro de +/-120s, nonce no repetido) y
   devuelve un **JWT** de vida corta (ej. 15 min) con claims `client_id`, `tenant_id`.
2. El agente conecta al hub pasando ese JWT:
   `HubConnectionBuilder().WithUrl(hubUrl, o => o.AccessTokenProvider = () => jwt)`.
3. El hub valida el JWT (Authorize) y asocia la conexion al `tenant_id` + `client_id`.

**Opcion B - handshake dentro del hub (backlog)**: conectar anonimo y como primer mensaje
`Hello(clientId, ts, nonce, hmac)`; el hub valida y sino `Abort()`. Mas codigo, menos estandar.

El servidor agrupa la conexion en un **grupo por cliente**: `Groups.Add(connId, $"client:{clientId}")`
y por tenant `$"tenant:{tenantId}"`, para poder empujar ordenes por cliente.

## 3. Metodos del hub (servidor) - los llama el AGENTE

```
// El agente -> servidor
Task AgentHello(AgentHello msg)          // opcional (opcion B); reporta version, host, capacidades
Task FetchResult(FetchResult msg)        // responde a una orden FetchRequest (o un chunk)
Task FetchFailed(FetchError msg)         // reporta que no pudo ejecutar la orden
Task Heartbeat()                         // opcional; el agente avisa que sigue vivo
```

## 4. Metodos del cliente (agente) - los invoca el SERVIDOR (push)

```
// Servidor -> agente (por el grupo client:{clientId})
On("FetchRequest", FetchRequest req)     // "traeme estos datos ya"
On("Ping")                               // sanity/keepalive de aplicacion
On("Cancel", { correlationId })          // cancelar una orden en curso (backlog)
```

## 5. Mensajes (forma JSON)

`FetchRequest` (servidor -> agente):
```json
{
  "correlationId": "uuid",          // para casar request/response
  "tenantId": "uuid",
  "connector": {
    "kind": "Database",             // Database | RestApi (fuente que corre el agente)
    "dbEngine": "SqlServer",        // SqlServer | MySql | PostgreSql | Oracle | ...
    "host": "10.0.0.20",            // host DENTRO de la LAN del cliente
    "port": 1433,
    "database": "db3dev",
    "username": "ecorex_ro",
    "secretRef": "opaque-or-inline" // ver seguridad: credencial cifrada/parametro
  },
  "query": {
    "text": "SELECT id, name, price FROM items WHERE updated_at > @since",
    "params": { "since": "2026-07-01T00:00:00Z" },
    "timeoutSeconds": 60
  },
  "paging": {                        // opcional; el servidor decide como paginar
    "mode": "Offset",               // Offset | Page | None
    "pageSize": 500,
    "maxRows": 100000
  }
}
```

`FetchResult` (agente -> servidor), posiblemente en varios chunks:
```json
{
  "correlationId": "uuid",
  "chunkIndex": 0,
  "isLast": false,
  "fields": ["id", "name", "price"],   // solo en chunkIndex 0
  "rows": [
    { "id": "351", "name": "PACK ORO", "price": "0" },
    { "id": "352", "name": "...",       "price": "12000" }
  ],
  "rowCount": 500
}
```

`FetchError` (agente -> servidor):
```json
{ "correlationId": "uuid", "code": "DB_CONN", "message": "No se pudo conectar a 10.0.0.20:1433", "retryable": true }
```

`AgentHello` (agente -> servidor, al conectar):
```json
{ "clientId": "cli_2846098d60c5", "agentVersion": "1.0.0", "host": "PC-BODEGA-01", "os": "Windows 10", "capabilities": ["Database:SqlServer", "Database:MySql"] }
```

## 6. Secuencia - carga por horario (UC1)

```
Scheduler(web)      Hub          Agente(on-prem)     FuenteLocal
   |  timer ImportProcess vence   |                    |
   |----- arma FetchRequest ----->|                    |
   |         (config+consulta)    |-- FetchRequest --->|                 (push)
   |                              |                    |-- SELECT ------>|
   |                              |                    |<-- filas -------|
   |                              |<-- FetchResult(ch0)|                 (stream)
   |                              |<-- FetchResult(chN,isLast=true)      |
   |<-- ingesta (Append/Replace/Upsert) en tabla destino                |
   |-- update ImportProcess.LastRunAt = ahora, estado = OK -------------|
```

## 7. Secuencia - refresco inmediato (UC2)

Igual que UC1 pero el disparo viene de un boton "Refrescar ahora" en la UI del contenedor
(o de una regla del sistema), no del timer. El canal ya esta abierto, asi que la latencia es
de milisegundos + lo que tarde la consulta.

## 8. Secuencia - agente offline (UC3)

```
Scheduler       Hub                      (agente NO conectado)
   |  timer vence                          |
   |-- hay agente en grupo client:{id}? -->| NO
   |<-- vacio ------------------------------|
   |-- marca ImportProcess: estado=PENDIENTE_AGENTE_OFFLINE, LastRunAt=ahora
   |-- encola reintento (politica: al reconectar o en N minutos)
   ...
   (mas tarde) Agente conecta -> AgentHello -> el servidor revisa pendientes de ese cliente
   y dispara los FetchRequest atrasados (o espera al proximo timer, segun politica).
```

## 9. Chunking / datasets grandes

- El agente lee por lotes (`pageSize`) y envia varios `FetchResult` con `chunkIndex`
  incremental; el ultimo lleva `isLast=true`.
- El servidor ingesta por chunk (igual que el import REST hace SaveChanges por pagina) para
  no acumular todo en memoria; respeta el tope `maxRows`.
- Tamano de chunk recomendado: 500-2000 filas. Mensajes SignalR grandes: subir
  `MaximumReceiveMessageSize` en el hub si hace falta (ver doc 03).

## 10. Correlacion, timeouts y cancelacion

- `correlationId` casa request<->results. El servidor guarda un "pending fetch" con timeout
  (ej. 5 min); si no llegan resultados, marca fallo y libera.
- Idempotencia: si el agente re-envia un chunk (por reconexion), el servidor debe deduplicar
  por (`correlationId`, `chunkIndex`).
- Cancelacion (backlog): `Cancel(correlationId)` para abortar una consulta larga.

## 11. Versionado del protocolo

- `AgentHello.agentVersion` + un `protocolVersion` constante. El servidor puede rechazar
  agentes por debajo de una version minima con un mensaje claro.
