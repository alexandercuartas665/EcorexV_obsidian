---
tipo: nota-adr
capa: transversal
proposito: Bitacora de decisiones de arquitectura (ADRs) del sistema destino ECOREX Tareas (.NET 10) - el "por que" detras de la hoja de ruta
estado: vivo
---

# ADRs - Decisiones de arquitectura

> Registro de las decisiones arquitectonicas ya tomadas, con su contexto y
> consecuencias, para que el "por que" no se pierda. Cada decision ya esta reflejada
> en la [[HOJA DE RUTA DESARROLLO]] y las notas de capa; aqui se consolida el
> razonamiento. Formato ADR ligero. En el repo destino se replican en `docs/decisiones/`.

## Indice

| ADR | Decision | Estado |
|---|---|---|
| 001 | DAL dual PostgreSQL + SQL Server | Aceptada |
| 002 | Multi-tenant real (TenantId + HasQueryFilter + RLS) | Aceptada |
| 003 | Blazor Server para la UI | Aceptada |
| 004 | Guid v7 como PK | Aceptada |
| 005 | EAV -> jsonb / nvarchar(max) para formularios | Aceptada |
| 006 | RabbitMQ + MassTransit para eventos | Aceptada |
| 007 | bpmn-js para el editor de flujos | Aceptada |
| 008 | Consola PlatformAdmin separada de la app del tenant | Aceptada |
| 009 | AI Provider Gateway multi-proveedor | Aceptada |
| 010 | Migracion por ETL (no big-bang), origen como referencia | Aceptada |

---

## ADR-001 - DAL dual PostgreSQL + SQL Server

**Contexto**: el origen corre en SQL Server (`db3dev`); hay clientes enterprise con
infraestructura Windows + licencia SQL, y el SaaS gestionado es mas barato en
PostgreSQL. El ecosistema hermano (CUBOT) usa Postgres. El origen YA era dual
(`MotherData.AdmDatos` + `AdmNpgsql`).

**Decision**: abstraccion `IEcorexDbContext` con `PostgresDbContext` y
`SqlServerDbContext`, elegida por `Database:Provider`. Nunca SQL crudo fuera de
repositorios por proveedor. Tests corren en ambos motores.

**Consecuencias**: (+) cada tenant en el motor que su contrato exija; reutiliza el
SQL Server legacy como destino enterprise. (-) doble mantenimiento de migraciones y
matriz de tests dual. Ver [[00 - Visión MotherData]].

## ADR-002 - Multi-tenant real (TenantId + HasQueryFilter + RLS)

**Contexto**: el origen aislaba por columna `SUCURSAL` + `WHERE` manual; un olvido
filtraba datos entre empresas (bug real `Session("Emmpresa")`).

**Decision**: `TenantId` (Guid) en toda entidad `ITenantScoped`, `HasQueryFilter`
global aplicado por reflexion, + RLS en la BD como defensa en profundidad. El
aislamiento es invariante del sistema, no responsabilidad del dev.

**Consecuencias**: (+) imposible olvidar el filtro; test de aislamiento que debe
fallar si se rompe. (-) `PlatformAdmin` necesita un `IPlatformDbContext` sin filtro,
auditado. Ver [[Gestion de Empresas - Admin multi-tenant]].

## ADR-003 - Blazor Server para la UI

**Contexto**: se quiere stack 100% .NET, sin Node/React, compartiendo DTOs y
validaciones. El prototipo final define el aspecto (Linear-style).

**Decision**: Blazor Server interactivo. SignalR (que Blazor Server ya usa) sirve
tambien para tableros y notificaciones vivas.

**Consecuencias**: (+) una sola solucion, sin toolchain JS, tiempo real gratis. (-)
dependencia de conexion viva (circuit); mitigado con reconexion automatica. Alterna
evaluada: Blazor WebAssembly (descartada por peor arranque y no compartir el circuito).

## ADR-004 - Guid v7 como PK

**Decision**: PKs con Guid v7 (ordenable por tiempo). Reemplaza los `int REG` /
`CODIGO` string del origen. El `LegacyId` conserva el id viejo para el ETL.

**Consecuencias**: (+) ordenable, sin colisiones, cross-motor, sin secuencia central.
(-) 16 bytes vs 4; indices un poco mas grandes (aceptable).

## ADR-005 - EAV -> jsonb / nvarchar(max) para formularios

**Contexto**: el origen guarda respuestas de formulario en EAV (`FORX_DATA`), con
lecturas O(n) por fila-atributo.

**Decision**: `FormResponse.data` como `jsonb` (Postgres, indexable con GIN) /
`nvarchar(max)` con `ISJSON` (SQL Server). La definicion del formulario sigue
relacional.

**Consecuencias**: (+) lectura O(1) del documento, indexable. (-) el ETL debe
consolidar el EAV; se prueba con round-trip. Ver [[Constructor - Patron EAV y motor visual]].

## ADR-006 - RabbitMQ + MassTransit para eventos

**Decision**: eventos de negocio (`task.state.changed`, `workflow.completed`,
`workflow.stuck`) via RabbitMQ + MassTransit; workers en `Ecorex.Workers`.

**Consecuencias**: (+) desacople, reintentos, escalamiento de flujos asincrono. (-)
infra extra en local (compose). Alterna: eventos in-process (descartada por no
sobrevivir reinicios ni escalar).

## ADR-007 - bpmn-js para el editor de flujos

**Contexto**: el origen ya usa bpmn-js y su XML es BPMN 2.0 estandar OMG (portable,
validado en bpmn.io). Ver [[Portabilidad BPMN - prueba en bpmn.io]].

**Decision**: reutilizar bpmn-js embebido en Blazor via interop. El motor de
ejecucion es propio (`WorkflowEngine`), no Camunda.

**Consecuencias**: (+) sin lock-in, editor maduro, XML portable. (-) un poco de
interop JS en un stack .NET puro (acotado a ese componente).

## ADR-008 - Consola PlatformAdmin separada

**Decision**: `Ecorex.Web.Platform` separada de `Ecorex.Web` (app del tenant). Se
separan por host y policy, con MFA en la de plataforma.

**Consecuencias**: (+) superficie de ataque acotada, sin mezclar gobierno con
operacion. (-) dos hosts a desplegar. Ver [[Seguridad y Autenticacion multi-tenant]].

## ADR-009 - AI Provider Gateway multi-proveedor

**Contexto**: el origen usa `Funciones.clChatGPT` (un solo proveedor cableado, sin
cupos ni trazabilidad).

**Decision**: `IAiProvider` con adaptadores (OpenAI, Claude, Gemini, Ollama...),
`AiUsageLog`, cupos por plan, herramientas gobernadas y guardrails.

**Consecuencias**: (+) sin lock-in, costos controlados, IA gobernada. (-) capa extra.
Ver [[Agentes de IA - Arquitectura y Operacion]].

## ADR-010 - Migracion por ETL, origen como referencia

**Decision**: no reescribir a ciegas ni big-bang. El legacy (`C:\Desarrollo\core`) es
la referencia de reglas de negocio; los datos se migran por ETL idempotente desde
`db3dev`, tenant por tenant, con round-trip y verificacion de conteos.

**Consecuencias**: (+) preserva 15 anos de logica de negocio y los datos reales; se
puede correr en paralelo. (-) doble esfuerzo mientras conviven. Ver
[[HOJA DE RUTA DESARROLLO]] seccion 9.

---

## Como agregar un ADR

1. Numero siguiente + titulo corto.
2. Contexto (por que se decide ahora), Decision, Consecuencias (+/-), alternativas.
3. Agregar fila al indice y, si aplica, enlazar la nota de capa afectada.
4. Replicar en `docs/decisiones/` del repo destino [EcorexV](https://github.com/alexandercuartas665/EcorexV.git).

## Enlaces

- [[HOJA DE RUTA DESARROLLO]] — donde estas decisiones se aplican
- [[Visión y entorno]] — vision que las enmarca
- [[00 - Visión MotherData]] / [[Gestion de Empresas - Admin multi-tenant]] / [[Seguridad y Autenticacion multi-tenant]]
