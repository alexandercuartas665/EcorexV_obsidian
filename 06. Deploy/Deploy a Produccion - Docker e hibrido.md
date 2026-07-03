---
tipo: nota-deploy
capa: transversal
proposito: Guia de despliegue del sistema destino ECOREX Tareas (.NET 10) - Docker + hibrido VPS/Railway
estado: planeado (destino de migracion)
---

# Deploy a Produccion - Docker e hibrido

> Guia operativa de despliegue del nuevo **ECOREX - Sistema de Tareas** (.NET 10).
> Aterriza la estrategia de DevOps de la [[Visión y entorno|vision general]] (seccion 15).
> El deploy del sistema ORIGEN (WebForms a IIS/Contabo) se conserva como referencia
> en [[Deploy IIS y perfiles de publicacion]]. (Estado: planeado; el destino aun no
> esta en produccion.)

## 1. Estrategia general

Desacoplar el software de la infraestructura desde el inicio. Todo corre en
contenedores Docker, configurado por variables de entorno, para mover servicios
entre VPS (Contabo, ya en uso por el origen), Railway o Kubernetes sin reescribir.

```txt
Fase 1 - VPS centralizado (MVP): todo en un VPS Contabo con Docker Compose. Bajo costo, validar la migracion.
Fase 2 - Hibrido: Railway (Web/Api/SignalR) + VPS (BD, Redis, RabbitMQ, Workers, Evolution).
Fase 3 - Enterprise: Kubernetes, BD administrada, Redis cluster, colas administradas.
```

## 2. Reparto de servicios (Fase 2 recomendada)

| En Railway | En VPS propio (Contabo) |
|------------|---------------|
| `Ecorex.Web` (app Blazor del tenant) | PostgreSQL / SQL Server (segun tenant) |
| `Ecorex.Web.Platform` (consola PlatformAdmin) | Redis |
| `Ecorex.Api` (REST + webhooks) | RabbitMQ |
| SignalR (tableros y notificaciones vivas) | `Ecorex.Workers` (escalamiento de flujos, recordatorios, integraciones) |
| | Evolution API (sesiones WhatsApp persistentes) |

Razon: los workers de flujo y las integraciones salientes necesitan procesos
permanentes y persistencia; encajan en un VPS controlado. La Web, la Api y SignalR
se escalan y despliegan facil en Railway.

## 3. DAL dual en el deploy

Cada tenant vive en el motor que su contrato exija (ver
[[00 - Visión MotherData]]). El deploy debe soportar ambos:

- **PostgreSQL** (default SaaS gestionado): Railway/Supabase/Neon o VPS.
- **SQL Server** (enterprise on-prem): el `db3dev`/`db3prod` actual del origen sirve
  como destino natural para clientes que ya pagan licencia SQL.

La imagen es una sola; el proveedor se elige por `Database__Provider` (`Postgres` o
`SqlServer`) y la cadena de conexion resuelta por tenant. Las migraciones se aplican
por contexto: `PostgresDbContext` y/o `SqlServerDbContext`.

## 4. Variables de entorno clave

```txt
Database__Provider=Postgres            # o SqlServer, por despliegue/tenant
ConnectionStrings__Default=Host=...;Database=ecorex;Username=...;Password=...
Redis__Connection=...
RabbitMq__Host=...
Evolution__BaseUrl=...                 # ApiKey cifrada en BD, no en env
SendGrid__FromAddress=...
DataProtection__KeysInDb=true
Jwt__Issuer=...  Jwt__SigningKeyRef=keyvault://...
```

Secretos (Evolution, SendGrid, Slack, llaves IA) cifrados con DataProtection; las
llaves viven en la tabla `data_protection_keys` de la propia BD, asi que migrar la
base preserva los secretos. En produccion, referencias a Key Vault (nunca el valor
en env). Ver [[Deploy IIS y perfiles de publicacion]] para el contraste con el
`Config.xml` cifrado del origen (que tenia la key embebida en codigo — error a no repetir).

## 5. Workers programados

Reemplazan los jobs implicitos del origen; corren como `BackgroundService` /
Hangfire en `Ecorex.Workers`:

- `WorkflowEscalationWorker`: detecta flujos trabados o tareas vencidas y escala
  segun las reglas del nodo (alimenta el KPI `workflow_stuck_rate`).
- `ReminderWorker`: recuerda tareas por vencer a responsables, respetando la TZ del tenant.
- `NotificationWorker`: entrega notificaciones diferidas (SignalR + email).
- `IntegrationWorker`: envios salientes a Slack / SendGrid / Evolution (idempotentes, con reintentos).
- `SubscriptionWorker`: revisa planes, limites y estados de tenant (ver [[Gestion de Empresas - Admin multi-tenant]]).

## 6. Salud y observabilidad

Health checks de la BD (ambos motores), Redis, RabbitMQ, Evolution Connector y
servicios internos. Serilog estructurado con `CorrelationId` y `TenantId`. La consola
PlatformAdmin muestra errores recientes sin exponer datos sensibles. Detalle de
metricas de dominio (`workflow_stuck_rate`, `tasks_completed_total`) en la
[[Visión y entorno|vision]] seccion 16.

## 7. Checklist primer deploy piloto

El primer piloto (migrar un tenant real como SKY SYSTEM) requiere:

- [ ] PlatformAdmin + gestion de tenants operativa
- [ ] Un tenant provisionado (aislamiento probado: test cross-tenant que DEBE fallar)
- [ ] Modulos, usuarios y permisos (policies) del tenant migrados
- [ ] Tareas, Proyectos y Tableros funcionando con el aspecto del [[00 - Prototipo Final ECOREX|prototipo]]
- [ ] Al menos 1 flujo BPMN ejecutando (WorkflowEngine) end-to-end
- [ ] Al menos 1 formulario dinamico capturando datos (jsonb)
- [ ] Motor de reglas ejecutando un verbo Ensamblado
- [ ] Integraciones criticas del tenant (Evolution/SendGrid/Slack) conectadas
- [ ] Backup automatico corriendo (ver [[Backup y DR por proveedor]])
- [ ] Migraciones aplicadas en el motor del tenant sin dataloss

## 8. Enlaces

- [[CICD Pipeline]] — como se construye, prueba y despliega
- [[Backup y DR por proveedor]] — recuperacion ante desastre
- [[Deploy IIS y perfiles de publicacion]] — deploy del ORIGEN (referencia)
- [[Plan de validacion del sistema]] — que se valida antes de publicar
- [[Visión y entorno]] — estrategia DevOps en la vision

---

*Documento vivo. Registrar cada deploy real en una nota fechada dentro de `06. Deploy/`.*
