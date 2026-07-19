---
tipo: especificacion-modulo
modulo: Agentes Colmena (Infraestructura IA)
adr: ADR-0045
estado: COMPLETO (Olas 1-5). Item de menu (Infraestructura IA) + diag log del agente + concurrencia VERIFICADA en vivo (2 navegador + 2 gateway a la vez).
fecha: 2026-07-19
autor: documentado por agente IA a partir de decisiones del usuario
---

# Modulo Agentes Colmena — el cliente/agente como recurso propio + bitacora

> Origen (decision del usuario, 2026-07-18): comparando **Contenedores de datos** y **Extraccion de
> datos** se noto que AMBOS necesitan un "cliente" para conectar con el agente colmena, y que ese
> alta estaba **duplicada** en las dos paginas. Se decidio **independizar** el cliente/agente en su
> propio modulo, para registrarlo UNA vez y reusarlo, y de paso tener **mejor gestion de log** de lo
> que hace cada agente. Ver [[00 - INDICE - Agente Conector On-Prem]].

## 1. En una frase

El **cliente/agente colmena** (identidad on-prem: `ClientId` publico + secreto cifrado) es un recurso
de infraestructura **transversal**, no algo que pertenezca a un modulo. Ahora vive en su propio modulo
**Agentes Colmena** (Infraestructura IA): se registra/rota/revoca ahi, y Contenedores + Extraccion solo
lo **seleccionan**. Ademas hay una **bitacora transversal** de lo que atendio cada agente.

## 2. Decisiones (con el usuario)

| # | Decision | Elegido |
|---|----------|---------|
| A | Migracion del CRUD de las 2 paginas | **Mover al modulo nuevo**; Contenedores/Extraccion solo SELECCIONAN. |
| B | Detalle del log | **Resumen por orden** (cliente, tipo, origen, resultado, duracion, detalle). |
| C | Ubicacion en el menu | **Infraestructura IA -> Agentes Colmena**. |
| D | Concurrencia del navegador | 1 instancia WebView2 **aislada por orden**, **sin tope**, sesion aislada. |

## 3. Que se construyo (Olas)

- **Ola 1 (servicio)**: `IAgentClientService` (namespace `Agents`) es el duenio del ciclo de vida de
  `data_clients`: List/Save/RotateSecret/**Revoke** (deshabilitar sin borrar)/Delete. Reusa la tabla
  (sin migrar datos). `DataImportConfigService` **delega** para no romper a los consumidores. Commit `00f0400`.
- **Ola 2 (bitacora)**: entidad `AgentActivityLog` (tenant) + enums `AgentActivityKind` (Browser/Fetch/
  File) y `AgentActivityResult`; migracion **dual** (PostgreSQL + SQL Server). El servidor escribe 1
  registro resumen por orden (best-effort: si falla, no tumba la orden), con el flujo como origen.
  Commit `e40108c`.
- **Ola 3 (UI)**: pagina `/agentes-colmena` — clientes (registrar/rotar/revocar, secreto una vez) +
  **presencia** (en linea / host / version / hace-N-min, via el registro del hub) + **feed de actividad**
  con filtro por agente. Commit `3d9b5ce`.
- **Ola 4 (consumidores)**: Contenedores y Extraccion pasan a **seleccionar**; se quita el "+ Agente"/
  CRUD duplicado y se agrega un enlace "Gestionar agentes". Commit `a3caa40`.

## 4. Concurrencia de la colmena (relacionado, misma tanda)

Se detecto que la colmena tenia **una sola** WebView2 compartida sin candado: 2 ordenes solapadas se
pisaban la pagina (datos cruzados). Se corrigio: **una instancia WebView2 efimera y aislada por orden**
(ventana + carpeta de datos propia), y el Servicio despacha **cada orden en su propia Task** (no bloquea
el bucle de SignalR) -> varias ordenes simultaneas abren varios navegadores en paralelo. El **panal** ya
estaba cableado para brotar un hexagono efimero por orden, asi que se veran varios a la vez. Commit `3b88e61`.

## 5. Ola 5 (cerrada 2026-07-19): menu + diag log + concurrencia verificada en vivo

- **Item de menu** bajo **Infraestructura IA** (`Agentes Colmena`, 000868). Se sembraba en
  `EnsureDefaultMenuAsync` pero los tenants YA existentes no lo recibian; se agrega
  `EnsureAgentesColmenaMenuItemAsync` (backfill idempotente sobre toda Section Route="ia") + cableado
  en el arranque tras ambas ramas (skip/demo). Verificado como usuario cliente tenant. Commit `21db1a2`.
- **Diag log del agente**: `FileLoggerProvider` en el Servicio deja SIEMPRE copia del ciclo de conexion
  en `%PUBLIC%\Documents\ecorex-agent-diag.log` + una linea con la config leida de la boveda
  (ClientId/Hub/Secreto). Fue lo que ubico al instante un ClientId viejo en la boveda. Commit `d3d26c3`.
- **Concurrencia VERIFICADA en vivo** (colmena real elevada, BD aislada `ecorex_agente`, puerto 5262):
  4 tareas con el MISMO `next_run_at` se pisaron -> 2 de Navegador (page/1 + page/2) abiertas EN
  PARALELO (mismo timestamp) + 2 de Gateway/DB headless a la vez. 20 filas ingestadas, feed poblado,
  agente En linea. Prueba: programar todo con la misma hora de ejecucion en `import_processes`.

**Pendiente menor (post-Ola 5):**
- Log de la ruta **fetch** (gateway) en el **feed de la UI** — hoy solo salen las ordenes de Navegador
  en el feed; las de Gateway salen en el diag del agente. La entidad/servicio ya son agnosticos al tipo.
- Un **permiso propio** del modulo (hoy reusa `ExtraccionDatos.Editar`).

## 6. Confirmacion: consulta editable del conector de BD (2026-07-19)

A peticion del dueno se confirmo (ya existia, no hubo que construir): el conector de **Base de datos**
se define por un **SELECT libre editable** en la UI del contenedor (textarea "Consulta: SELECT que trae
los datos"). Sirve para una tabla simple (`SELECT * FROM t`) o una consulta compleja (joins/where/
columnas calculadas). Editable al crear Y al editar (`_conQuery = c.Query`). Solo-lectura; el agente la
ejecuta tal cual contra la BD remota. `ContenedorDatos.razor:377`, `ProcessRunner`, `GatewayExecutor`.

## 7. Referencias de codigo (repo ecorex-agent-gui)

- Servicio: `apps/backend/src/Ecorex.Application/Agents/AgentClientService.cs`, `AgentActivityQuery.cs`.
- Bitacora: `Ecorex.Domain/Entities/AgentActivityLog.cs`, escritor en `Ecorex.SuperAdmin/Agents/AgentActivityLogWriter.cs`.
- UI: `Ecorex.SuperAdmin/Components/Pages/AgentesColmena.razor`.
- Concurrencia: `apps/agent/Ecorex.Agent.Gui/Services/WebView2BrowserSubAgent.cs`, `Ecorex.Agent.Core/Services/RealHiveConnection.cs`.
- Menu (Ola 5): `EnsureAgentesColmenaMenuItemAsync` en `Ecorex.Infrastructure/Persistence/DatabaseSeeder.cs` + cableado en `Ecorex.SuperAdmin/Program.cs`.
- Diag log (Ola 5): `apps/agent/Ecorex.Agent.Service/FileLoggerProvider.cs` (+ Program.cs, AgentWorker.cs) -> `%PUBLIC%\Documents\ecorex-agent-diag.log`.
- ADR: `docs/decisiones/0045-modulo-propio-para-el-cliente-agente-colmena.md`.
