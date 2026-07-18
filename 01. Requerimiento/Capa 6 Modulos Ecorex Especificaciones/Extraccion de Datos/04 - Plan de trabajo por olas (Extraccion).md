---
tipo: plan-construccion
proyecto: Extraccion de Datos
proposito: Backlog por olas con criterios de aceptacion. La v1 llega hasta la CONFIGURACION; el runtime va despues.
---

# 04 - Plan de trabajo por olas

> Contrato de trabajo. Respetar las reglas del repo (CLAUDE.md): multi-tenant real, SQL parametrizado,
> secretos cifrados, migracion DUAL PG/SQL Server, ASCII en archivos nuevos, PROGRESO.md por sesion,
> fidelidad milimetrica al prototipo (`proto_web_scraping.html`).

## Ola 0 - Descubrimiento (HECHO)

- Leer este capitulo + la spec legacy + el estado actual del modulo (`ExtraccionDatos.razor`,
  `ScrapeSource`/`ScrapeRun`, `IScrapeService`) + el sub-agente Navegador (acciones, MCP, JS firmado) +
  el Contenedor de datos (ingesta, `ImportProcess`/`ImportRun`).
- **Aceptacion**: mapa de piezas existentes vs nuevas y decisiones E1-E4 tomadas con el usuario. HECHO
  (2026-07-17).

## Ola 1 - Dominio del flujo + migracion dual

- Entidades `ScrapeFlow`, `ScrapeStep`, `ScrapeVariable` (+ enum `ScrapeStepKind`), con FKs a
  `DataClient` y `DataContainer`. Config EF + migracion DUAL (PG y SQL Server).
- Decidir el puente de programacion: `ImportProcess` gana un objetivo "flujo" (campo `FlowId`) o se
  generaliza. Reusar `ImportRun` como bitacora.
- Servicio CRUD (`IScrapeFlowService`) + DTOs. Variables cifradas con `ISecretProtector`.
- **Aceptacion**: se puede crear/editar/borrar un flujo con pasos ordenados y variables cifradas;
  build verde; migracion aplica en ambos motores; el snapshot queda consistente
  (`has-pending-model-changes` limpio en los dos contextos).

## Ola 2 - UI del configurador (milimetrica al prototipo)

- Reescribir `ExtraccionDatos.razor` (o una pagina nueva que la reemplace) segun
  `proto_web_scraping.html`: acento morado, sidebar de flujos con estado y numero de pasos, hero con
  KPIs, tarjeta de pasos ordenados con editor de JS inline y reordenar, panel de cliente+variables
  cifradas, destino (contenedor), programacion, e historico de corridas.
- Cada tipo de paso muestra sus campos (Navigate=URL; InjectScript/Extract=JS+mapeo; Wait=condicion;
  AI=instruccion+destino+allow-list de tools+topes+modelo).
- **Aceptacion**: el operador disena un flujo completo desde la web, con los dos tipos de paso, y queda
  guardado. Verificado en Chrome contra el tenant demo.

## Ola 3 - Runtime determinista (cablear al Navegador)

- Compilar un flujo a `BrowserAction[]`, sustituir variables, FIRMAR el JS, empujar `BrowserRequest`
  al agente, recibir `BrowserResult`, ingerir los pasos `Extract` con `IRowIngestService`, cerrar la
  corrida en `ImportRun`. Manejo offline reusado.
- **Aceptacion**: un flujo real (login + navegar + extraer) corre en un agente de la colmena y las filas
  aterrizan en el contenedor; la bitacora lo registra; el JS sin firma se rechaza; un dominio fuera de
  la allow-list se bloquea. E2E.

## Ola 4 - Paso de IA

- Orquestacion agente<->navegador por el MCP local, con el modelo del AI Provider Gateway (entre los
  habilitados por el Super Admin), acotada por allow-list de tools + tope de pasos/tiempo. Ingesta del
  resultado.
- **Aceptacion**: un paso de IA resuelve una pagina que un selector fijo no aguantaria, respeta los
  topes, y su resultado aterriza en el contenedor. Cupo por plan verificado.

## Ola 5 - Programacion + endurecimiento

- Programacion reusada (horario/manual) probada en vivo, como se hizo con el Contenedor de datos.
- Paginacion controlada, advertencias (etiqueta -> notificar/detener), historico rico, panel de salud.
- Decidir absorcion vs coexistencia con `ScrapeSource`/`ScrapeRun` (ADR-0025).
- **Aceptacion**: un flujo programado corre solo y deja bitacora; las advertencias detienen/notifican.

## Criterios transversales (toda ola)

- `dotnet build` verde; tests de lo tocado en verde (integracion dual donde aplique).
- Multi-tenant: imposible que un flujo de un tenant lea/escriba datos de otro.
- Secretos siempre cifrados (variables con `ISecretProtector`); JS del servidor FIRMADO; allow-list
  fail-closed; nunca secretos en logs.
- SQL parametrizado; ingesta por el nucleo `IRowIngestService`.
- PROGRESO.md actualizado; ADR si hay decision arquitectonica nueva; archivos nuevos en ASCII.
