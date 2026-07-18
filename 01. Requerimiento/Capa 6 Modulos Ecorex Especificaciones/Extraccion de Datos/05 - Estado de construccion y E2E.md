---
tipo: estado-construccion
proyecto: Extraccion de Datos
proposito: Que se construyo (Olas 1-5), como se probo, el E2E del runtime, y que falta. Fuente de verdad del avance real del modulo 000730.
actualizado: 2026-07-18
---

# 05 - Estado de construccion y E2E

> Resumen del avance REAL del capitulo. Las specs (docs 01-04) dicen COMO debe ser; este dice QUE hay
> hecho y probado a hoy. Repo: `feat/agente-colmena-gui` (EcorexV.git).

## 1. Que se construyo, ola por ola

| Ola | Que | Estado |
|-----|-----|--------|
| 1 | Dominio del flujo: `ScrapeFlow`/`ScrapeStep`/`ScrapeVariable` (+ enum `ScrapeStepKind`), FKs a `DataClient` y `DataContainer`, variables cifradas, CRUD `IScrapeFlowService`. Migracion DUAL. | HECHO |
| 2 | UI configurador (`ExtraccionDatos.razor`) milimetrica al prototipo: lista de flujos, hero, pasos por-tipo (los 6 deterministas + IA), cliente+variables, tabla destino. | HECHO |
| 3 | Runtime DETERMINISTA: compilar flujo -> `BrowserAction[]` (firma el JS), despachar al Navegador, ingerir los `Extract` con `IRowIngestService`, bitacora `ScrapeFlowRun`. UI "Ejecutar ahora" + historial. | HECHO + **E2E** (ver s2) |
| 4 | Paso de IA: orquestacion server-side sobre el AI Provider Gateway (tool-calling), tools de navegador acotadas por allow-list + topes, JS firmado, ingesta. Runtime unificado a ejecucion SECUENCIAL (canal await). | HECHO (tests con fakes; E2E con LLM real pendiente de llaves) |
| 5 | Programacion (reusa `ImportProcess`/recurrencia; el dispatcher ramifica a `RunFlowNowAsync`), paginacion (`{{PAGINA}}` + rango), advertencias (etiqueta -> Notify/Stop), y coexistir con `ScrapeSource`. | HECHO (programacion verificada en Chrome + BD) |

### Piezas clave (nombres para buscar en el repo)
- **Compilador**: `ScrapeFlowCompiler` (flujo -> acciones, sustituye `{{VAR}}`, FIRMA el JS con `AgentSign`).
- **Runtime**: `IBrowserRunService`/`BrowserRunService` (secuencial, en segundo plano) + `IBrowserActionChannel` (request/response por correlationId sobre el hub) + `IScrapeFlowRunLog` (bitacora).
- **Paso de IA**: `IAiStepOrchestrator` (bucle de tools sobre `IAiProviderClient.CompleteWithToolsAsync`).
- **Ingesta**: `ScrapeRowIngest` (reusa `IRowIngestService`, el mismo nucleo del Contenedor).
- **Programacion**: `ImportProcess.FlowId` + `ImportScheduleDispatcher` (rama de flujo) + `IScrapeFlowService.Save/GetSchedule`.

### ADRs del repo
- **ADR-0042**: bitacora DEDICADA `ScrapeFlowRun` (no reusar `ImportRun`, que cuelga de `ImportProcess`).
- **ADR-0043**: orquestacion del paso de IA server-side; el servidor FIRMA el JS que genera la IA (el
  agente lo rechaza sin firma). Contencion: allow-list de tools + topes + allow-list de dominios.
- **ADR-0044**: coexistir con `ScrapeSource` (no absorber hasta que el runtime este probado E2E).

## 2. E2E del runtime determinista (2026-07-18)

Se cerro el lazo COMPLETO del runtime determinista contra el servidor + la BD REALES, usando un
**agente stand-in** por SignalR (una consola .NET que referencia `Ecorex.Contracts.Agent`). El stand-in
NO es la colmena WebView2: se autentica como el bot, VERIFICA la firma del JS que manda el servidor, y
DEVUELVE resultados fabricados (los `Extract` responden filas). Asi se prueba todo el pipeline server
sin depender del navegador on-prem.

**Pasos del E2E:**
1. **Handshake**: `POST /api/agente/token` con `HMAC(secret, "clientId|ts|nonce")` -> JWT. OK.
2. **Conexion**: SignalR al `AgenteHub` real (`/hubs/agente`) con el JWT; `AgentHello` -> ONLINE.
3. **Disparo**: "Ejecutar ahora" desde Chrome sobre un flujo real (Navigate + Extract) con contenedor
   destino "Productos E2E" (columnas SKU/NOMBRE/PRECIO) y el bot asignado.
4. **Compila + firma + despacha**: el servidor emitio `BrowserRequest` con el `Extract` Eval FIRMADO.
5. **El agente verifica**: `firma = VALIDA` (el contrato de firma server<->agente funciona).
6. **Ingesta**: el servidor correlaciono el `BrowserResult`, parseo las 3 filas y las INGIRIO en el
   contenedor via `IRowIngestService`.
7. **Bitacora**: `ScrapeFlowRun` cerro **Ok, 3 filas** (visto en la UI: `Manual / OK / 0.3 s / 3` y en BD).

**Camino negativo (seguridad)**: se disparo el endpoint dev `/api/agente/dev/browse/...?nosign=true`
(JS SIN firma); el agente lo marco `firma = INVALIDA` y rechazo la accion (fail-closed). Queda probado el
contrato en ambos sentidos: JS firmado se ejecuta, JS sin firma se rechaza.

**Evidencia**: contenedor "Productos E2E" con 3 filas reales (Taladro Bosch/Martillo Stanley/
Destornillador Truper) en `data_container_cells`; `scrape_flow_runs` con `result=Ok, inserted=3`.

**Que cubre y que no**: cubre auth + transporte SignalR + compilacion + FIRMA + despacho + correlacion +
ingesta + bitacora, todo REAL. Lo unico fabricado por el stand-in es la EJECUCION del navegador (que en
produccion hace la colmena WebView2, ya probada en el capitulo del Agente Conector). El paso de IA no
entra en este E2E (necesita un proveedor de IA con llave habilitado en el Super Admin).

## 3. Pruebas automatizadas

- **SuperAdmin.Tests** (52 verdes): compilador (firma cubre el JS sustituido y ligado al corr; Extract
  ancla el indice; Wait/Click firmados; sin-secreto rechaza JS pero permite solo-Navigate; Ai rechazado),
  `ParseRows` (arreglo/objeto/doble-codificado/basura), y el orquestador de IA con fakes (bucle
  navegar->guardar ingiere; sin proveedor -> claro; tope de pasos -> corta; allow-list no ofrece
  eval/clic; firma el JS de un evaluar_js).
- **Application.Tests** (457 verdes). Build de la solucion 0 errores; migraciones DUALES con snapshot
  limpio en PG y SQL Server.

## 4. Lo que falta (honesto)

- **E2E del paso de IA**: un LLM real manejando el navegador y sus filas aterrizando. Necesita (a) un
  proveedor de IA habilitado con llave en el Super Admin, y (b) la colmena conectada. El bucle, la firma,
  la allow-list, los topes y la ingesta ya estan probados con fakes.
- **E2E con la colmena WebView2 real** contra un sitio real (en vez del stand-in). El stand-in ya probo
  todo el lado servidor; falta cambiar el stand-in por la colmena para validar la ejecucion del navegador
  de punta a punta (esa ejecucion ya se valido aparte en el capitulo del Agente / sub-agente Navegador).
- **Absorcion de `ScrapeSource`**: decidida a favor de COEXISTIR por ahora (ADR-0044); reevaluar cuando lo
  anterior este probado en vivo.

## 5. Como reproducir el E2E (para el proximo que lo toque)

1. Levantar el server dev del worktree (`start-agente-dev.ps1`, puerto 5262, BD `ecorex_agente`).
2. `POST /api/agente/dev/seed-client` -> devuelve `cli_dev_agent` / `dev-secret-ola-b`.
3. Un contenedor con columnas escalares + un flujo Navigate+Extract asignado al bot (via UI o SQL).
4. Correr un agente stand-in (consola SignalR que referencia `Ecorex.Contracts.Agent`): handshake al
   `/api/agente/token`, conectar al hub, responder `BrowserResult` a los `BrowserRequest` verificando la
   firma. (El del E2E de hoy quedo fuera del repo, en un scratchpad.)
5. "Ejecutar ahora" y verificar `scrape_flow_runs.result=Ok` + filas en `data_container_cells`.
