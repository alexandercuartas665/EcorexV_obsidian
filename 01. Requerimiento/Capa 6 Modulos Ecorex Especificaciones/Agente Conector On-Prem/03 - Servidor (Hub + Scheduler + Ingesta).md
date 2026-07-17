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

> **[CONSTRUIDO 2026-07-15] Canal minimo del servidor (doc 05 Ola 1, lado servidor).** En
> `Ecorex.SuperAdmin` (host que ya tiene SignalR + auth), rama `feat/agente-colmena-gui`:
> `RealTime/AgenteHub.cs` (`[Authorize(AuthenticationSchemes="Agent")]`, grupos client/tenant,
> presencia) + `Agents/` (`IAgentRegistry`/`InMemoryAgentRegistry`, `AgentTokenIssuer`,
> `AgentNonceCache`, `AgentChannel` con el esquema bearer "Agent" NO-default -no toca la auth de
> cookies- y los endpoints). Identidad = entidad `DataClient` existente (ClientId + secreto cifrado
> con `ISecretProtector`). Verificado E2E contra la BD dev (Postgres 5442): handshake rechaza
> credencial invalida/ts fuera de rango (401); con un `DataClient` valido el agente obtiene el JWT,
> conecta autenticado ("[AGENTE] En linea" + AgentHello con tenant), y un push del servidor -> el
> agente responde `FetchResult` (round-trip). La colmena WPF marca "En linea" con Gateway encendido.

> **[CONSTRUIDO 2026-07-15] Ingesta via agente (s6, doc 05 Ola 3).** El `FetchResult` del agente
> termina como filas en un contenedor de datos reusando el motor EAV. Verificado E2E: `SELECT ... FROM
> ciudades` (SQL Server real) -> 20 filas -> contenedor "Ciudades (agente)": Replace `ins=20`, 2o
> Replace `del=20/ins=20`, Upsert por CODIGO_POSTAL `upd=20` sin duplicar (queda en 20). Falta el
> Scheduler (Ola 4) y el `RunsViaAgent`+UI para disparar desde un `ImportProcess` real (hoy con un
> endpoint dev). Nota de entorno: la BD dev tenia drift (faltaba fisicamente
> `data_container_columns.referenced_container_id` pese al historial); se corrigio con un ALTER puntual
> -no es bug del codigo; una BD limpia trae la columna via su migracion existente.

> **[CONSTRUIDO 2026-07-17] Scheduler + refresco + bitacora (Ola 4).** El scheduler vive en
> `Ecorex.SuperAdmin` (`RealTime/ImportSchedulerWorker.cs`), NO en `Ecorex.Workers` (prod solo levanta
> ese servicio). `RunsViaAgent` NO se implemento: "via agente" = el `ImportProcess` tiene cliente; lo
> que faltaba era la CONSULTA, agregada como `DataConnector.Query`. Recurrencia con **Cronos**
> (ADR-0041), no el motor de 000889. Bitacora `ImportRun` (indice unico de idempotencia como
> `ScheduledJobRun`) cierra la corrida contra el `correlationId`, que ahora genera el runner. Verificado
> en vivo en los dos motores (intervalo/SQL Server, cron/PostgreSQL). Detalle en el doc 05 Ola 4.

- [x] Hub `/hubs/agente` con `[Authorize]` + grupos por client/tenant.
- [x] `IAgentRegistry` (en linea/offline) + `IHubContext` para empujar.
- [x] `POST /api/agente/token` (HMAC -> JWT corto). Handshake opcion A completo (nonce + ts +/-120s).
- [x] ~~`DataConnector.RunsViaAgent` + `DataClientId`~~ **descartado**: "via agente" = el
      `ImportProcess` tiene cliente (no hizo falta campo nuevo). SI se agrego `DataConnector.Query`
      (la consulta) + migracion dual `AddConnectorQuery`.
- [x] `IRowIngestService` (nucleo de ingesta EAV Append/Replace/Upsert por sesion, en
      `Ecorex.Application/DataContainers/RowIngestService.cs`). **Un solo camino de escritura**: lo usan
      TANTO el importador via agente COMO `ApiImportService` (REST), migrado el 2026-07-16. El REST abre
      la sesion, `PrepareAsync` (Replace: vacia; Upsert: precarga la clave) y manda cada pagina como un
      chunk (un SaveChanges por pagina, igual que antes); los contadores salen de la sesion.
- [x] Tests unit del nucleo: `tests/Ecorex.Application.Tests/RowIngestServiceTests.cs` (5, EF InMemory):
      Append (fila+celdas, TenantId), Append en 2 chunks, Replace (vacia antes), Upsert (actualiza por
      clave sin duplicar) y Upsert con claves repetidas en la misma corrida (gana la ultima).
- [x] `IAgentImportService` (dispatch + on-result + on-failed) en `Ecorex.SuperAdmin/Agents/
      AgentImportService.cs`: pending-fetch por correlationId, acumula chunks y en el ultimo ingiere
      via `IRowIngestService` (scope propio con el tenant fijado). Cableado en `AgenteHub`.
- [x] `ImportSchedulerWorker` (Intervalo/Cron) **en `Ecorex.SuperAdmin`, no `Ecorex.Workers`** +
      `ImportScheduleDispatcher` + `ImportRecurrence` (Cronos) + bitacora `ImportRun`. Offline: la
      corrida queda `Error` con motivo (el reintento-al-reconectar del UC3 NO se implemento; simplif.).
- [x] UI: config de BD + consulta desde la web; estado en linea ("Probar conexion" por cliente);
      boton **"Actualizar datos"**; en Procesos, proxima ventana, motivo de desactivacion y
      **"Ultimas ejecuciones"** (la bitacora).
- [x] Dedup de chunks + timeout del pending fetch (Ola 6, 2026-07-17): `SeenChunks` + `SweepAsync`.
- [ ] TLS estricto (**BLOQUEANTE**, ADR-0040) + `Cancel` (declarado sin implementar) + `AgentFetchLog`
      completo (hoy `ImportRun` cubre el caso de negocio) + integracion dual PG/SQLServer del canal.
