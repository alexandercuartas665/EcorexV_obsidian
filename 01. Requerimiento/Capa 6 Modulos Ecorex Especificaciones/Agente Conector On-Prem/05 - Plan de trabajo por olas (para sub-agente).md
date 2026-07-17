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

- `Ecorex.Agent.Core` (C# / .NET 10): ejecuta `FetchRequest.query` parametrizado contra la fuente
  segun `dbEngine` (empezar por SQL Server), lectura por lotes, chunking, whitelist + solo-lectura.
- Devuelve filas como `Dictionary<string,string>`.
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
> `upd=20` sin duplicar.
>
> **[CONSTRUIDO 2026-07-16] REST migrado al nucleo**: `ApiImportService` ya no escribe filas por su
> cuenta (se borraron sus `InsertRow`/`DeleteAllRowsAsync`): abre una sesion del nucleo, `PrepareAsync`
> y manda cada pagina como chunk, conservando un SaveChanges por pagina y el tope de 5000 filas. Con
> esto se cumple la aceptacion "mismos outcomes que el import REST" por construccion: es el MISMO
> codigo. Cubierto por 5 tests unit del nucleo (`RowIngestServiceTests`, EF InMemory).
>
> **Pendiente de esta ola**: `DataConnector.RunsViaAgent`+consulta+migracion (se puede hacer junto al
> scheduler de la Ola 4).

## Ola 4 - Scheduler + refresco inmediato

- `ImportSchedulerService` en `Ecorex.Workers`: dispara `ImportProcess` (Intervalo/Cron) que
  usan conector via agente; maneja agente offline (marca estado + reintento).
- UI: opcion "Ejecutar via agente", selector de agente, campo de consulta + mapeo; estado
  en linea/offline en "Clientes remotos"; boton "Refrescar ahora".
- **Aceptacion**: un proceso por Intervalo dispara solo y carga datos via agente; "Refrescar
  ahora" carga al instante; con el agente apagado, el proceso queda "pendiente offline" y se
  ejecuta al reconectar.

> **[CONSTRUIDO 2026-07-17] Ola 4 COMPLETA y verificada en vivo.** El horario dispara solo y cada
> corrida deja bitacora. Difiere del plan de arriba en tres puntos, todos a mejor:
>
> - **Vive en `Ecorex.SuperAdmin`, no en `Ecorex.Workers`** (`RealTime/ImportSchedulerWorker.cs`),
>   por el mismo motivo que el worker de 000889: el compose de prod solo levanta `ecorex-app`, asi que
>   un hosted service en `Ecorex.Workers` NUNCA correria en prod. Copia el patron probado de
>   `ScheduledJobWorker`: `PeriodicTimer` 1 min, barrido cross-tenant que devuelve solo `TenantId`s,
>   luego scope propio + `AmbientTenantContext.Begin`, `Take(200)`.
> - **NO se uso `DataConnector.RunsViaAgent` + `DataClientId`**. El modelo ya lo resolvia sin campo
>   nuevo: **"via agente" = un `ImportProcess` que tiene cliente** (`ClientId`). Lo unico que faltaba
>   en el conector era la **consulta**: se agrego `DataConnector.Query` (ni el DTO ni el servicio ni la
>   UI la transportaban, aunque la columna existia; el boton era inejecutable y solo se via al usarlo).
> - **El boton se llama "Actualizar datos"** (no "Refrescar ahora") y el horario y el boton llaman al
>   **MISMO `IProcessRunner`**: si el horario tuviera su propia version, las dos acabarian
>   comportandose distinto.
>
> **Recurrencia (ADR-0041)**: `ImportRecurrence` calcula la proxima ventana para Intervalo (a mano) y
> Cron (**Cronos**, paquete MIT nuevo). NO reusa el motor de 000889: aquel recibe un `ScheduledJobRule`
> concreto y razona en frecuencias de calendario (Daily/Weekly/Monthly), que no contemplan ni "cada N
> minutos" ni cron. Se reusa lo comun: `ResolveTimeZone` y los patrones de bitacora y worker. El cron
> se interpreta **en hora del tenant**; tras una caida larga **NO se dispara la rafaga atrasada** (se
> salta al futuro); un horario invalido **desactiva la programacion con el motivo a la vista** en vez
> de dejarla "activa" sin disparar nunca.
>
> **Bitacora `ImportRun`** (esto es lo que el usuario pidio como "log de trabajos"): copia el patron de
> `ScheduledJobRun`, **incluido el indice unico `(TenantId, ProcessId, FiredAt)`** que da idempotencia
> -dos workers en la misma ventana chocan al guardar en vez de pedir el mismo refresco dos veces; por
> eso `FiredAt` es la ventana, no "ahora"-. La corrida nace en `Running` y la cierra `IImportRunLog`
> contra el `correlationId`, que ahora genera el **runner** (no el canal): si lo generara el canal, un
> agente rapido podria responder antes de que la corrida supiera su propio id.
>
> **Verificado E2E, dos veces, en los DOS motores**:
> - Intervalo "cada 1 minuto" sobre SQL Server (`prueba_cargas.dbo.PRODUCTOS`): disparo dos veces
>   seguidas sin tocar nada, `trigger=Scheduled`, `result=Ok`, separadas un minuto exacto.
> - Cron `9 7 * * *` sobre PostgreSQL: **disparo a las 12:09:00 UTC exactas** (07:09 Bogota), cargo 2
>   filas en una tabla vaciada a proposito, reprogramo sola para el dia siguiente.
>
> **Bug real, imposible de ver sin correr**: guardar un cron reventaba (`Cannot write DateTimeOffset
> with Offset=-05:00 to PostgreSQL 'timestamp with time zone'`): Cronos devuelve la hora con el desfase
> de la zona y Npgsql exige desfase 0. Las 12 pruebas del motor NO lo veian porque `DateTimeOffset`
> compara instantes (`08:00Z == 03:00-05:00`). Arreglado + 2 pruebas que afirman sobre `.Offset`.
> Leccion anotada: para fechas con destino Postgres, un `Assert.Equal` de instantes no basta.
>
> **Manejo de agente offline (UC3): HECHO y verificado en vivo (2026-07-17).** El runner comprueba
> `IsOnline` DESPUES de abrir la corrida y, si el agente no esta, la marca `PendingOffline` (NO `Error`:
> es un "no llegue a intentarlo") y **parquea la programacion** con `ImportProcess.PendingSince`. El
> worker tiene un pase de reintento -gateado por `IsOnline`- que reintenta las parqueadas SOLO cuando su
> agente vuelve; el gate evita generar una corrida cada minuto mientras sigue caido. Verificado con el
> reintento AISLADO del horario (se empujo `next_run_at` a +1h para que solo el reintento pudiera
> dispararlo): agente apagado -> `PendingOffline` + parqueo; agente encendido -> a los <70s el worker lo
> puso al dia solo (log *"agente reconecto, carga pendiente reintentada"*), bitacora `Ok`, `PendingSince`
> limpio. Migracion dual `AddImportPendingOffline`; la UI muestra "Esperando al agente desde X".

## Ola 5 - Empaque del agente (Servicio Windows + colmena WPF + instalador)

> **REVISADA 2026-07-16 por ADR-0039** (D8: el despliegue real es AMBOS escenarios). El plan de
> abajo se escribio pre-colmena y asumia dos piezas sueltas. Choca con dos hechos: el store cifra
> con DPAPI de USUARIO en `%APPDATA%` (un servicio corre con otro perfil/llave y **no puede leerlo**),
> y WebView2 no vive en la sesion 0 de un servicio. Modelo vigente: **el Servicio es dueno de
> identidad, canal y store; la colmena WPF es su CLIENTE por named pipe y le presta el escritorio al
> Navegador**. Sub-olas 5a-5d abajo.

- `Ecorex.Agent.Service` (Worker Service) que mantiene el Core 24/7 (canal + Gateway + Archivos).
- `Ecorex.Agent.Gui` (colmena WPF): identidad, allow-lists, consentimiento, estado, y **ejecuta el
  Navegador** por delegacion del servicio.
- Instalador (WiX/Inno) que registra el servicio, crea `ProgramData` con su ACL y deja la colmena
  con auto-arranque al inicio de sesion.
- **Aceptacion**: instalable en una maquina Windows limpia; el servicio reconecta tras
  reinicio; la WPF configura identidad/allow-lists y muestra el estado real; en un servidor SIN
  sesion, Gateway y Archivos atienden y el Navegador falla con motivo claro (no cuelga).

> **[CONSTRUIDO 2026-07-16] Ola 5a - seam del Navegador + nucleo sin WPF**: `IBrowserSubAgent` en
> Contracts (el marshalling al Dispatcher se escondio DENTRO de la impl WebView2, asi que el canal y
> el MCP ya no dependen de WPF) + proyecto **`Ecorex.Agent.Core`** (net10.0-windows, `UseWPF=false`)
> con 11 de los 12 servicios (~1400 de 1850 lineas): canal, Gateway, Archivos, stores DPAPI,
> allow-lists, consentimiento, QueryGuard y MCP. La Gui queda con WebView2 + UI. Que el Core compile
> con `UseWPF=false` ES la prueba del desacople. Verificado en runtime: 14 tools MCP,
> `browser.navigate` OK atravesando MCP(Core) -> seam -> WebView2 en hilo de UI; Archivos lee dentro
> de su raiz y sigue rechazando fuera. **Sin cambio de comportamiento.**
>
> **[PARCIAL 2026-07-16] Ola 5b - boveda de maquina + Worker Service**. Cuenta del servicio:
> **LocalSystem** (D9, decidida por el usuario). Hecho:
> - **`AgentVault`**: el P/Invoke a DPAPI y la ruta estaban duplicados en los **5** stores; ahora hay
>   un solo lugar. Boveda = `%ProgramData%\Ecorex\Agent` + DPAPI de **maquina** + **ACL** (rompe
>   herencia, solo SYSTEM+Administradores). Comprobado en maquina real con `Get-Acl`.
> - **`Ecorex.Agent.Service`** (Worker, `UseWindowsService`): hospeda el Core headless; el mismo
>   binario corre como servicio (log al Visor de eventos) o como consola. El Navegador responde NO con
>   motivo explicito (`UnavailableBrowserSubAgent`) en vez de colgar la peticion.
> - **`--save-config` es del SERVICIO**, no de la colmena (el dueno del store es el servicio), y
>   normaliza la URL al hub completo.
> - **Bug real corregido**: el canal se tragaba el motivo del fallo de conexion; headless eso es
>   indepurable. Ahora `LastError` explica el handshake (secreto, ClientId, reloj >120s).
>
> **Confirmado por la prueba (importante)**: al mudar la boveda, la colmena (proceso sin elevar) deja
> de poder leerla o escribirla -que es lo que el ADR quiere- y por tanto **no hay forma de configurar
> el agente hasta que exista el pipe (5c)**. 5b y 5c estan acoplados; entre medias se configura con
> `--save-config` desde consola de administrador (lo mismo que hara el instalador).
>
> **Cerrado el 2026-07-16**: el servicio CONECTA al hub de punta a punta leyendo su identidad de la
> boveda de maquina (`Conectado a .../hubs/agente como cli_dev_agent`).

> **[CONSTRUIDO 2026-07-16] Ola 5c - canal local (named pipe)**. Confirmo lo previsto: sin este canal
> NO se puede configurar el agente, porque la colmena ya no abre la boveda. No fue cirugia: los dos
> seams ya existian, asi que son dos implementaciones nuevas (`PipeHiveConnection : IHiveConnection`
> -> **el ViewModel y el panal no cambiaron**; `DelegatedBrowserSubAgent : IBrowserSubAgent` -> el
> canal y el MCP no se enteran de que el navegador vive en otro proceso).
> - **Seguridad**: el servicio es LocalSystem, asi que el pipe es superficie privilegiada (ensanchar
>   la allow-list de Archivos = abrirle a la nube el disco como SYSTEM). Leer estado y prestar
>   escritorio = cualquier usuario interactivo; **MUTAR = solo Administradores**, comprobado
>   impersonando al cliente del pipe. El **secreto no viaja al cliente**: se escribe, no se lee.
> - **`BrowserPolicy`**: el consentimiento y la allow-list los EMPUJA el servicio con el estado; la
>   colmena tiene escritorio pero no boveda, asi que sola fallaria cerrada siempre.
> - **Tres bugs reales** hallados probando (todos tapados por `catch` mudos, ya corregidos): ACL sin
>   `Synchronize` (se disfraza de **timeout**, no de acceso denegado, porque `Connect` reintenta);
>   **buffers del pipe en 0** (cada escritura espera al lector -> saludo abrazado a si mismo:
>   conexion viva y canal mudo); y falta de `CreateNewInstance` (anadir instancias a un pipe
>   existente lo exige sobre el DACL ya puesto -> servia a UNA colmena y la siguiente quedaba fuera).
> - **Verificado E2E**: colmena sin elevar conecta e identifica bien al no-admin; estado publicado;
>   **navegacion exitosa** (`OK 3 acciones`) recorriendo hub -> servicio -> pipe -> colmena ->
>   WebView2 -> vuelta; sin colmena, falla con motivo y no se cuelga; `set-consent` de un no-admin
>   RECHAZADO con su motivo.

> **[CONSTRUIDO 2026-07-16] Ola 5d - instalador**. En `deploy/agent/`: `publish.ps1` (autocontenido,
> **271 MB**: la aceptacion dice "maquina limpia" y exigir el runtime de .NET no es eso),
> `install.ps1` (**crea la boveda el**, owner=Administradores + ACL cerrada, por el hallazgo de
> propiedad del directorio de ADR-0039; registra `EcorexAgent` LocalSystem automatico con reintentos
> escalonados; colmena en auto-arranque de sesion), `uninstall.ps1` (**conserva la boveda** salvo
> `-RemoveVault`) y `README.md`.
>
> **No hay Inno ni WiX en la maquina**: un `.iss` seria codigo imposible de compilar o probar, asi que
> el instalador PowerShell ES el entregable (y es lo que un `.iss` acabaria invocando).
>
> **ACEPTACION CERRADA (2026-07-16, instalado y desinstalado de verdad)**: servicio **Running**, Auto,
> LocalSystem y **sin consola**; el hub le **despacha ordenes** (solo despacha a agentes en linea); la
> colmena instalada muestra **"En linea"** con el estado que le publica el servicio por el pipe; el
> desinstalador no deja rastro y **conserva la boveda**.
>
> **Dos bugs reales corregidos** (rompian promesas del propio README):
> - **El Visor de eventos estaba MUDO**: el origen no existia (crear uno exige privilegio y el
>   proveedor de EventLog no avisa, solo deja de escribir) **y** el proveedor filtra en `Warning` por
>   defecto, asi que "Conectado a X como Y" se descartaba. Ahora el instalador registra el origen y el
>   servicio baja el filtro a Information.
> - El desinstalador dejaba `C:\Program Files\ECOREX` huerfana.
>
> **NO verificado**: el arranque tras REINICIO real del equipo. Consta `StartMode=Auto` (el mecanismo)
> y que el servicio arranca y conecta solo.
>
> **Pendiente**: `.iss` + firma del ejecutable. El peso (271 MB) **se queda asi** por decision del
> usuario (2026-07-16): esta bien a cambio de no exigir el runtime en la maquina del cliente.

## Ola 6 - Endurecimiento (seguridad + robustez)

- Politica de credencial gestionada por el agente (opcion b), TLS estricto, dedup de chunks,
  timeouts/cancelacion, auditoria `AgentFetchLog`, limites por plan.
- Multi-instancia del hub (backplane Redis) si aplica.
- **Aceptacion**: pruebas de seguridad (rechazo de no-SELECT, fuera de whitelist, cert
  invalido); pruebas de resiliencia (reconexion, chunk duplicado, timeout).

> **ESTADO REAL (auditado 2026-07-16 contra el codigo, no de memoria).** La ola esta **a medias sin
> que nadie lo hubiera anotado**: la seguridad se fue construyendo por el camino, pero la
> **robustez/auditoria sigue intacta**. Punto por punto:
>
> | Punto de la Ola 6 | Estado |
> |---|---|
> | Credencial gestionada por el agente (opcion b) | **HECHO** (Ola C): vive en la boveda, no viaja |
> | Rechazo de no-SELECT | **HECHO** (`QueryGuard`), verificado |
> | Rechazo fuera de allow-list (dominios y rutas) | **HECHO**, fail-closed, verificado |
> | Reconexion | **HECHO** (backoff 0/2/5/10/30/60s), verificada |
> | *(extra, no estaba en la lista)* JS del servidor **firmado** | **HECHO** (doc 06 s4) |
> | *(extra)* Consentimiento local por capacidad | **HECHO** |
> | *(extra)* Boveda con DPAPI de **maquina** + ACL SYSTEM/Admins | **HECHO** (Ola 5b) |
> | *(extra)* **Mutar config exige Administrador** (impersonacion en el pipe) | **HECHO** (Ola 5c) |
> | **TLS estricto** | **PENDIENTE - guardrail, NO bloqueante (encuadre corregido 2026-07-17)**. El cifrado del canal lo da el **despliegue detras de HTTPS**: el agente conecta por `wss://` y la credencial va cifrada en transito, sin tocar el agente. La validacion de certificados de .NET ademas esta activa (un cert invalido YA se rechaza). Lo que falta es solo que el agente **RECHACE** una URL no-TLS (hoy acepta `http://`), como defensa contra config erronea/downgrade (salvo `localhost` en dev) + prueba con cert invalido. Barato; no es riesgo activo si prod es HTTPS. |
> | **Dedup de chunks** por (`correlationId`,`chunkIndex`) | **HECHO (2026-07-17)**. `Pending.SeenChunks` (HashSet por `ChunkIndex` bajo lock); un chunk repetido se descarta con log en vez de duplicar filas. |
> | **Timeout del pending fetch** | **HECHO (2026-07-17)**. `PendingTtl` 10 min + `AgentImportService.SweepAsync` en cada ciclo del worker: expira la peticion, libera sus filas y **cierra la corrida en la bitacora** aunque nadie tenga la pagina abierta. `_outcomes` tambien caduca (30 min); antes ninguno de los dos se limpiaba. |
> | **Cancelacion** | **HECHO (2026-07-17)**. `CancelMsg` tipado; el agente guarda un CTS por correlationId y `On(Cancel)` lo cancela, con el token llegando hasta el `GatewayExecutor` (aborta la consulta en la BD). Al cancelar manda `FetchFailed CANCELLED`. El servidor lo empuja desde `CancelAsync` y desde el timeout sweep. Verificado: Npgsql aborta un `pg_sleep(25)` a los 3s al cancelar el token. |
> | **Auditoria `AgentFetchLog`** | **CUBIERTO en parte por `ImportRun` (2026-07-17)**, con matices. La bitacora de la Ola 4 registra por corrida: trigger, resultado, filas ins/upd/del, `correlationId`, motivo del fallo, y sobrevive al proceso. Lo que `ImportRun` NO cubre y `AgentFetchLog` si preveia: corridas que ni llegan a ser `ImportProcess` (el endpoint dev de la Ola 3) y metricas de latencia (ms). Para el caso de negocio -horarios- ya hay rastro persistente. |
> | **Limites por plan** (cupos de filas/frecuencia) | **PENDIENTE** |
> | **Backplane Redis** (multi-instancia del hub) | **PENDIENTE**. Hoy `InMemoryAgentRegistry` -> con 2+ instancias del servidor, una no sabe de los agentes de la otra. Solo importa si se escala horizontal. |
>
> **Lectura (corregida 2026-07-17)**: lo que protege de un **servidor comprometido** esta hecho, y ya
> se cerraron **los dos que muerden en produccion** (dedup y timeout). Lo que queda, por urgencia:
> **(1) `Cancel`** -el contrato miente hasta que se implemente o se quite del protocolo-; **(2) TLS
> estricto** -guardrail del cliente, NO bloqueante: el cifrado lo da el despliegue HTTPS, esto solo
> impide que un agente mal configurado use `http://`-; **(3)** escala/plan (limites, Redis) que solo
> importan al crecer.

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
