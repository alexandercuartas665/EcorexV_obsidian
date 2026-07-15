---
tipo: plan-construccion
proyecto: Agente Conector On-Prem
proposito: Backlog por olas con criterios de aceptacion, para que un sub-agente construya el proyecto de forma incremental y verificable.
---

# 05 - Plan de trabajo por olas (para sub-agente)

> Contrato de trabajo. Cada ola es entregable y verificable por si sola. Respetar las reglas
> del proyecto (CLAUDE.md del repo `EcorexV`): multi-tenant real, SQL parametrizado, secretos
> cifrados, transacciones, ASCII en archivos nuevos, PROGRESO.md por sesion, tests dual PG/SQL.

## Ola 0 - Descubrimiento (leer antes de tocar)

- Leer este capitulo completo (docs 00-04) + el modulo ya construido: `ContenedorDatos.razor`,
  `IApiImportService`/`ApiImportService`, `DataConnector`/`DataClient`/`ImportProcess`,
  entidades `DataContainerRow/Cell/Link`, y `DataImportConfigService` (repo `EcorexV`).
- Confirmar en cual host va el hub (`Ecorex.Api` vs `Ecorex.SuperAdmin`) y como corre hoy
  `Ecorex.Workers`.
- **Aceptacion**: nota corta con el mapa de piezas existentes vs nuevas y decisiones abiertas
  (politica de credencial a/b; ConnectorKind.Agent vs flag RunsViaAgent) resueltas con el usuario.

## Ola 1 - Canal SignalR minimo (servidor + agente fake)

Servidor:
- `AgenteHub` con `[Authorize]` (JWT agente), grupos `client:{id}`/`tenant:{id}`, metodos
  `AgentHello`/`FetchResult`/`FetchFailed`.
- `POST /api/agente/token` (HMAC del secreto de `DataClient` -> JWT corto).
- `IAgentRegistry` (en linea/offline) + `IHubContext` para empujar `FetchRequest`.

Agente (o un cliente de prueba temporal en .NET):
- Conecta con token, manda `AgentHello`, responde a un `FetchRequest` de prueba con
  `FetchResult` fijo (sin BD real todavia).

- **Aceptacion**: desde un test/endpoint el servidor empuja un `FetchRequest` a un agente
  conectado y recibe de vuelta `FetchResult`; en el web se ve el agente "en linea". Handshake
  rechaza credenciales invalidas.

> **[AVANCE 2026-07-15] Lado AGENTE construido y probado E2E** (rama `feat/agente-colmena-gui`,
> mapeado a la "Ola B" de la colmena, doc 06). `RealHiveConnection` (SignalR client) + protocolo
> compartido en `Ecorex.Contracts.Agent`. Probado contra un **simulador** `tools/Ecorex.Agent.HubSim`
> (NO es apps/backend): AgentHello + round-trip `FetchRequest`/`FetchResult` reales, reconexion con
> backoff. **Pendiente lado SERVIDOR** (esta ola, en `apps/backend`): `AgenteHub` con `[Authorize]`,
> `POST /api/agente/token` (HMAC->JWT), `IAgentRegistry` + `IHubContext`. Auth aun en anonimo en el
> simulador; el hook `AccessTokenProvider` (opcion A de doc 02) queda listo en el cliente.

## Ola 2 - Ejecucion real contra BD local (agente)

- `EcorexAgent.Core` (VB.NET): ejecuta `FetchRequest.query` parametrizado contra la fuente
  segun `dbEngine` (empezar por SQL Server), lectura por lotes, chunking, whitelist + solo-lectura.
- Devuelve filas como `Dictionary(Of String,String)`.
- **Aceptacion**: con una BD SQL Server de prueba, el agente recibe una consulta, la ejecuta y
  responde las filas en chunks; una consulta no-SELECT o fuera de whitelist es rechazada.

> **[CONSTRUIDO 2026-07-15] Ejecucion real del Gateway** (mapeado a la "Ola C" de la colmena; C#, no
> VB.NET). `Services/SqlServerGatewayExecutor` (Microsoft.Data.SqlClient) + `QueryGuard` (whitelist
> solo-SELECT, un solo statement, verbos de escritura/DDL/exec bloqueados) + `GatewaySourceStore`
> (cadena de conexion cifrada con DPAPI LOCAL en el agente -opcion b: la credencial de la LAN NUNCA
> viaja por el canal ni se versiona). En `RealHiveConnection`, un `FetchRequest` Database ejecuta la
> consulta parametrizada, lee por lotes y devuelve `FetchResult` en chunks (fields en chunk 0).
> Verificado E2E contra una BD SQL Server REAL de la LAN (`M700_GEN`): `SELECT TOP 20 * FROM ciudades`
> -> 20 filas reales con sus columnas (DPTO, NOMBRE, PAIS, CODIGO_DIAN, DANE_DEP...); `DELETE` y
> `SELECT ... INTO` -> `FetchFailed QUERY_REJECTED`. **Pendiente**: chunking probado con dataset grande,
> mas motores (MySql/PostgreSql), y la ingesta en el servidor (doc 03 s6, Ola 3).

## Ola 3 - Ingesta reutilizable + conector via agente (servidor)

- Extraer `IRowIngestService` de `ApiImportService` (Append/Replace/Upsert por clave) y hacer
  que REST y Agente lo compartan.
- `DataConnector`: `RunsViaAgent` + `DataClientId` + consulta; migracion EF.
- `IAgentImportService.DispatchFetchAsync` + `OnFetchResultAsync` (ingesta por chunk).
- **Aceptacion**: un `FetchResult` del agente termina como filas en la tabla del contenedor,
  con Append/Replace/Upsert funcionando igual que el import REST (mismos outcomes).

> **[CONSTRUIDO 2026-07-15] Ingesta via agente**: `IRowIngestService` (nucleo EAV reutilizable) +
> `IAgentImportService` (pending-fetch + dispatch + on-result/on-failed) cableado en `AgenteHub`.
> Verificado E2E con `ciudades` (SQL Server real): Replace `ins=20`, 2o Replace `del/ins=20`, Upsert
> `upd=20` sin duplicar. **Pendiente de esta ola**: migrar `ApiImportService` (REST) al nucleo
> compartido (follow-up mecanico) y `DataConnector.RunsViaAgent`+consulta+migracion (se puede hacer
> junto al scheduler de la Ola 4).

## Ola 4 - Scheduler + refresco inmediato

- `ImportSchedulerService` en `Ecorex.Workers`: dispara `ImportProcess` (Intervalo/Cron) que
  usan conector via agente; maneja agente offline (marca estado + reintento).
- UI: opcion "Ejecutar via agente", selector de agente, campo de consulta + mapeo; estado
  en linea/offline en "Clientes remotos"; boton "Refrescar ahora".
- **Aceptacion**: un proceso por Intervalo dispara solo y carga datos via agente; "Refrescar
  ahora" carga al instante; con el agente apagado, el proceso queda "pendiente offline" y se
  ejecuta al reconectar.

## Ola 5 - Empaque del agente (Servicio Windows + WPF + instalador)

- `EcorexAgent.Service` (Worker Service) que mantiene el Core 24/7.
- `EcorexAgent.Config` (WPF): identidad, whitelist, estado, log, "probar conexion".
- Instalador (WiX/Inno) que registra el servicio.
- **Aceptacion**: instalable en una maquina Windows limpia; el servicio reconecta tras
  reinicio; la WPF configura identidad/whitelist y muestra el estado real.

## Ola 6 - Endurecimiento (seguridad + robustez)

- Politica de credencial gestionada por el agente (opcion b), TLS estricto, dedup de chunks,
  timeouts/cancelacion, auditoria `AgentFetchLog`, limites por plan.
- Multi-instancia del hub (backplane Redis) si aplica.
- **Aceptacion**: pruebas de seguridad (rechazo de no-SELECT, fuera de whitelist, cert
  invalido); pruebas de resiliencia (reconexion, chunk duplicado, timeout).

## Backlog (post v1)

- Streaming binario para datasets muy grandes; compresion.
- Fuentes locales de archivos (Excel/CSV en carpeta) y API local.
- Panel de salud de agentes (latencia, ultima corrida, errores) en el web.
- Auto-update del agente; firma de la orden (el agente verifica que la orden venga del server).
- Mapeo de campos anidados (ej. `category.name`) reutilizando lo del import REST.

## Criterios transversales de aceptacion (toda ola)

- `dotnet build` verde; tests de lo tocado en verde (integracion dual PG/SQL Server donde aplique).
- Multi-tenant: imposible que un agente reciba/entregue datos de otro tenant.
- Secretos siempre cifrados (server: `ISecretProtector`; agente: DPAPI); nunca en logs.
- SQL parametrizado; consulta del agente solo-lectura.
- PROGRESO.md actualizado; ADR si hay decision arquitectonica nueva.
- Archivos nuevos en ASCII.
