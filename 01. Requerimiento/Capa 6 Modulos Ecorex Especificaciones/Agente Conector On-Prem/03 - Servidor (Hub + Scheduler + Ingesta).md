---
tipo: spec-servidor
proyecto: Agente Conector On-Prem
proposito: Que construir del lado servidor (ECOREX) para soportar el agente: Hub SignalR, endpoint de token, scheduler en Ecorex.Workers, conector tipo Agente e ingesta reusando el motor existente.
---

# 03 - Servidor (Hub + Scheduler + Ingesta)

> Referencias al repo `EcorexV` (rama `fase-0/clon-backbone`). Todo lo del Contenedor de
> datos ya existe; aqui se listan las PIEZAS NUEVAS y donde encajan.

## 1. Proyectos tocados

- `Ecorex.Api` **o** `Ecorex.SuperAdmin`: mapear el Hub SignalR (`/hubs/agente`) y el endpoint
  `/api/agente/token`. (Definir en cual host vive el hub; recomendado el mismo que ya sirve la
  app y tiene el circuito autenticado.)
- `Ecorex.Application/DataContainers`: contratos del canal + servicio de orquestacion
  (`IAgentImportService`) y reuso de la ingesta.
- `Ecorex.Infrastructure`: registro DI, SignalR, y (si aplica) el store de conexiones/pendientes.
- `Ecorex.Workers`: **scheduler** (BackgroundService/Quartz) que lee `ImportProcess` y dispara.
- `Ecorex.Domain`: extender enums (nuevo `ConnectorKind.Agent` o reusar `Database` con un flag
  "via agente") y, si se persiste estado de agente, entidad `AgentConnection`/logs.

## 2. Hub SignalR

`AgenteHub : Hub` (en el host elegido):
- `[Authorize]` con JWT de agente (claims `client_id`, `tenant_id`).
- `OnConnectedAsync`: valida claims, agrega la conexion a los grupos `client:{clientId}` y
  `tenant:{tenantId}`, registra "agente en linea" (en memoria via `IAgentRegistry` y/o BD).
- `OnDisconnectedAsync`: marca "fuera de linea".
- Metodos servidor (invocados por el agente): `FetchResult`, `FetchFailed`, `AgentHello`,
  `Heartbeat` (ver doc 02). Cada uno resuelve el "pending fetch" por `correlationId`.
- Config: `MaximumReceiveMessageSize` amplio para chunks; WebSockets habilitado.

`IAgentRegistry` (singleton): sabe que `clientId` esta conectado (para responder "hay agente?"
antes de disparar). Puede ser en memoria (v1, 1 instancia) o Redis backplane (multi-instancia,
backlog) usando `IHubContext<AgenteHub>` para empujar.

Empujar una orden:
```
await _hub.Clients.Group($"client:{clientId}").SendAsync("FetchRequest", fetchRequest, ct);
```

## 3. Endpoint de token (handshake opcion A)

`POST /api/agente/token`  (anonimo, rate-limited):
- Body: `{ clientId, ts, nonce, hmac }`.
- Valida: `DataClient` activo por `clientId`; `ts` dentro de +/-120s; `nonce` no repetido
  (cache breve); `hmac == HMACSHA256(secret, clientId|ts|nonce)` usando el secreto descifrado
  con `ISecretProtector`.
- Devuelve un JWT corto (claims `client_id`, `tenant_id`, exp 15m). Reusar la infra JWT del
  backbone.

## 4. Conector tipo Agente

Decision D3: la fuente es "la que el modelo tenga como origen". Implementacion:
- Reusar `DataConnector` con `Kind = Database` (ya tiene DbEngine/Host/Port/Database/Username/
  credenciales cifradas) + un indicador de que se ejecuta **via agente** (opciones):
  - (a) nuevo `ConnectorKind.Agent` que envuelve la config de BD, o
  - (b) campo/booleano `RunsViaAgent` + `DataClientId` en `DataConnector` (a que agente se le
    pide). **Recomendado (b)**: menos ruptura, y deja claro QUE agente ejecuta el conector.
- Guardar en el conector (o en el `ImportProcess`) la **consulta** (SELECT + parametros) y el
  mapeo campo->columna (ya existe el patron de mapeo del import REST; reusarlo).
- Migracion EF pequena para el/los campos nuevos (seguir el patron squash del repo).

## 5. Scheduler (Ecorex.Workers)

`ImportSchedulerService : BackgroundService`:
- Cada minuto (o con Quartz por cron real), consulta `ImportProcess` activos cuyo horario
  vence (Intervalo: `LastRunAt + IntervalMinutes <= now`; Cron: evaluar expresion; Manual: no
  auto-dispara).
- Para cada proceso que vence y cuyo conector es "via agente":
  1. Resolver el `DataClientId` -> preguntar a `IAgentRegistry` si esta conectado.
  2. Si conectado: armar `FetchRequest` (config del conector con credencial descifrada +
     consulta + paging) y empujarlo por el hub; registrar "pending fetch" con timeout.
  3. Si NO conectado: marcar `ImportProcess` estado `PENDIENTE_AGENTE_OFFLINE`, `LastRunAt=now`,
     encolar reintento (o esperar reconexion, ver doc 02 seccion 8).
- Respetar tenant scoping y `HasQueryFilter`; el worker corre con contexto de plataforma o por
  tenant segun como se resuelva (definir en construccion; probablemente iterar por tenant).
- Zona horaria del tenant + UTC (regla del proyecto) para evaluar Cron.

Multi-instancia (backlog): tomar un lock (Redis) por proceso para que solo una instancia lo
dispare.

## 6. Ingesta (reuso del motor existente)

Cuando llegan los `FetchResult`, el servidor debe insertar/actualizar filas EXACTAMENTE como
ya hace `ApiImportService.ImportAsync` (Append/Replace/Upsert, celdas por columna, dedup por
clave). Refactor recomendado:
- Extraer de `ApiImportService` un **nucleo de ingesta** reutilizable:
  `IRowIngestService.IngestAsync(targetContainerId, mapping, IEnumerable<IReadOnlyDictionary<string,string?>> rows, ApiImportMode mode, keyColumnId, ...)`
  que haga el Replace (vaciar), Upsert (precargar clave->fila) e insert, devolviendo
  `ApiImportOutcome`.
- `ApiImportService` (REST) y el nuevo `AgentImportService` (agente) comparten ese nucleo.
  El agente aporta las filas; el mapeo/modo/clave los define el proceso.
- Ingerir por chunk (SaveChanges por chunk), respetando `maxRows`.

`IAgentImportService` (orquestador):
- `DispatchFetchAsync(importProcessId)`: arma y empuja el `FetchRequest` (usado por el
  scheduler y por "refrescar ahora").
- `OnFetchResultAsync(FetchResult)`: acumula/ingesta chunks via `IRowIngestService`.
- `OnFetchFailedAsync(FetchError)`: marca fallo + log.

## 7. UI web (ajustes al modulo contenedor-datos)

- En el conector: opcion "Ejecutar via agente on-prem" + selector del `DataClient` (agente) +
  campo de la consulta (SELECT) + mapeo (reusar el panel de mapeo del import REST).
- En la seccion "Clientes remotos": mostrar estado **en linea/offline** de cada agente (del
  `IAgentRegistry`), ultima conexion, version.
- Boton **"Refrescar ahora"** en el contenedor/proceso -> `IAgentImportService.DispatchFetchAsync`.
- En "Procesos": ver `LastRunAt`, estado (OK / offline / error) y ultimo resultado.

## 8. Auditoria y limites

- Log por orden: `AgentFetchLog` (tenant, clientId, processId, correlationId, filas, ms,
  resultado). Reusar el patron de auditoria del backbone.
- Limites por plan (backlog): frecuencia minima de horario, tope de filas por corrida (ya hay
  `MaxImportRows=5000` en el import REST; parametrizar por plan).

## 9. Checklist de construccion (servidor)

- [ ] Hub `/hubs/agente` con `[Authorize]` + grupos por client/tenant.
- [ ] `IAgentRegistry` (en linea/offline) + `IHubContext` para empujar.
- [ ] `POST /api/agente/token` (HMAC -> JWT corto).
- [ ] `DataConnector.RunsViaAgent` + `DataClientId` + consulta + migracion.
- [ ] `IRowIngestService` extraido de `ApiImportService` (compartido).
- [ ] `IAgentImportService` (dispatch + on-result + on-failed).
- [ ] `ImportSchedulerService` en `Ecorex.Workers` (Intervalo/Cron + offline handling).
- [ ] UI: opcion "via agente", estado en linea, "Refrescar ahora".
- [ ] Tests: unit (ingesta/upsert), integracion dual PG/SQLServer, y un test de canal con un
      agente fake (cliente SignalR de prueba).
