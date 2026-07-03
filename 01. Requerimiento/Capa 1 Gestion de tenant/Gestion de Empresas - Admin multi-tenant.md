---
tipo: nota-capa
capa: 1
proposito: Especificacion del sistema DESTINO (.NET 10) para la gestion multi-tenant real de empresas - corrige los errores del ECOREX WebForms
estado: diseno objetivo de migracion
stack_destino: .NET 10 / ASP.NET Core / EF Core 10 / Blazor
---

# Gestion de Empresas - Admin multi-tenant (.NET 10)

> **Documento de diseno del sistema destino.** No describe como funciona ECOREX
> hoy, sino como DEBE funcionar tras la migracion a .NET 10. El comportamiento
> legacy (WebForms + `MotherData.AdmDatos`) queda documentado en la spec de
> origen [[NEWFRONT_adm_empresas - Spec para reconstruir]] (Capa 6) y sirve solo
> como referencia de reglas de negocio a preservar. Aqui la consigna es explicita:
> **multi-tenant de verdad, sin heredar los errores del pasado.**

## 1. Proposito y principio rector

Esta capa define el gobierno del ciclo de vida de las empresas (tenants) en el
nuevo Bitcode/ECOREX sobre .NET 10. Un tenant es una organizacion que opera su
propia agenda de tareas, flujos, formularios, reglas y usuarios, aislada de las
demas. El operador de la plataforma controla creacion, planes, limites, modulos,
integraciones, aprovisionamiento, auditoria y baja.

**Principio rector**: el aislamiento entre tenants es una **invariante del
sistema, no una responsabilidad del desarrollador**. En el origen, un query sin
`WHERE SUCURSAL=...` filtraba datos entre empresas; en el destino eso es
imposible por construccion. Todo lo demas (planes, modulos, integraciones) se
apoya sobre esa garantia.

## 2. Los 9 errores del origen y como se corrigen

Esta es la tabla de contrato de la migracion. Cada fila es una decision de diseno
obligatoria, no una recomendacion.

| # | Error heredado (ECOREX) | Correccion en .NET 10 |
|---|---|---|
| 1 | Aislamiento por disciplina (columna `SUCURSAL` + `WHERE` manual) | `TenantId` (Guid) + `HasQueryFilter` global en EF Core + RLS en BD. Imposible olvidar el filtro |
| 2 | Rol operador inexistente (`flag_super` sin uso, acceso solo por menu) | Rol `PlatformAdmin` formal, policy-based, consola separada, MFA obligatorio |
| 3 | Cero auditoria de acciones administrativas | `AdminAuditLog` inmutable: quien, cuando, IP, entidad, valor previo, valor nuevo |
| 4 | Operaciones multi-tabla sin transaccion | Todo caso de uso multi-entidad corre en `IDbContextTransaction` o `TransactionScope` |
| 5 | DELETE de empresa sin cascada (huerfanos) | Baja logica (`soft-delete`) + archivado; purga fisica en cascada controlada por worker |
| 6 | Secretos de integracion dispersos / en texto plano | Boveda unica: ASP.NET Data Protection + Key Vault; nunca se muestran tras guardar |
| 7 | SQL concatenado (inyeccion sistemica) | EF Core parametrizado; prohibido `FromSqlRaw` con interpolacion de usuario |
| 8 | `db3dev.dbo.*` y blacklists hardcodeadas | Cero literales de catalogo/entorno en codigo; todo por configuracion y convencion |
| 9 | Parametros sin enforcement (`SUCURSAL_PAR` decorativo) | Planes con limites duros/blandos aplicados por `ITenantLimitService` en cada operacion |

## 3. Arquitectura de aislamiento (la invariante)

El corazon del sistema. Se implementa en tres capas defensivas:

### 3.1 Resolucion del tenant (entrada)

Un `TenantResolutionMiddleware` determina el tenant de cada request antes de
tocar la BD, en este orden de precedencia:

1. Claim `tenant_id` del JWT (usuarios autenticados).
2. Subdominio o header `X-Tenant` (APIs publicas por token).
3. Rechazo con 400 si no se puede resolver (nunca un default silencioso).

El `TenantId` resuelto se guarda en un `ITenantContext` scoped, unica fuente de
verdad para el resto del request.

### 3.2 Filtro global EF Core (defensa principal)

Toda entidad de negocio implementa `ITenantScoped { Guid TenantId }`. En el
`DbContext` se aplica el filtro por reflexion a TODAS ellas de una vez:

```csharp
protected override void OnModelCreating(ModelBuilder mb) {
    foreach (var et in mb.Model.GetEntityTypes()
             .Where(e => typeof(ITenantScoped).IsAssignableFrom(e.ClrType))) {
        mb.Entity(et.ClrType).AddQueryFilter<ITenantScoped>(
            e => e.TenantId == _tenantContext.TenantId);
    }
}
```

Con esto, `db.Tasks.ToList()` devuelve SOLO las tareas del tenant actual — sin
que el desarrollador escriba un solo `Where`. Olvidarlo ya no es posible.

### 3.3 Row-Level Security en BD (defensa en profundidad)

Ademas del filtro EF, la propia base de datos aplica RLS por si un acceso
saltara EF Core (reportes, jobs, herramientas):

- **PostgreSQL**: `CREATE POLICY tenant_isolation USING (tenant_id = current_setting('app.tenant_id')::uuid)`.
- **SQL Server**: `CREATE SECURITY POLICY` con predicado por `SESSION_CONTEXT('tenant_id')`.

La sesion de BD recibe el `tenant_id` en la apertura de conexion (interceptor de
EF Core que setea `SET app.tenant_id` / `sp_set_session_context`).

> El operador `PlatformAdmin` es el UNICO que puede consultar cross-tenant, y lo
> hace por un `DbContext` sin filtro (`IPlatformDbContext`) explicitamente
> auditado en cada uso.

## 4. Rol PlatformAdmin y consola separada

En el origen no habia rol de operador. En el destino:

- **Rol formal `PlatformAdmin`**, fuera del modelo de roles de cualquier tenant.
  No consume cupos de usuario, no hereda permisos de tenant.
- **Consola separada** (`admin.ecorex.*` o area `/platform`), distinta de la app
  de operacion diaria, protegida por policy `RequirePlatformAdmin`.
- **MFA obligatorio** y sesion reforzada (timeout corto, re-auth para acciones
  criticas).
- **Toda accion sensible se audita** en `AdminAuditLog` (seccion 9). Sin
  auditoria, la accion no se ejecuta — el registro es parte de la transaccion.

## 5. Modelo de dominio del tenant

La entidad `Tenant` reemplaza la tabla plana `SUCURSAL`, con subentidades reales
en vez de columnas y tablas sueltas.

```csharp
public class Tenant {                    // reemplaza SUCURSAL
    public Guid Id { get; set; }         // Guid v7 (ordenable), reemplaza CODIGO string
    public string PublicSlug { get; set; } // identificador publico unico (subdominio)
    public string LegalName { get; set; }  // NOMBRE_REAL
    public string CommercialName { get; set; } // NOMBRE
    public string TaxId { get; set; }        // NIT (validado, unico)
    public TenantStatus Status { get; set; } // enum con consecuencias (seccion 8)
    public Guid PlanId { get; set; }
    public ContactInfo Contact { get; set; }        // owned entity
    public LegalRepresentatives Legal { get; set; } // contador + revisor (owned)
    public DateOnly? ContractStart { get; set; }
    public DateOnly? ContractEnd { get; set; }
    public bool IsArchived { get; set; }     // soft-delete
    public DateTimeOffset CreatedAt { get; set; }
    public Guid CreatedByAdminId { get; set; }
}
```

Subentidades (antes tablas `SUCURSAL_*` sueltas):
- `TenantModule` (modulos habilitados) — reemplaza `SUCURSAL_MOD`
- `TenantRoleGroup` + `TenantRolePermission` — reemplazan `SUCURSAL_GRUPOS*`
- `TenantParameter` — reemplaza `SUCURSAL_PAR`, ahora **con enforcement**
- `TenantIntegration` — reemplaza `SUCURSAL_INT`, con secretos en boveda
- `TenantActivity` — reemplaza `TIPO_TAR_EMPRESA`
- Usuarios: relacion `UserTenant` (un usuario PUEDE pertenecer a varios tenants,
  ver seccion 6.3) — reemplaza `USUARIO.SUCURSAL`

## 6. Ciclo de vida del tenant

### 6.1 Alta guiada (onboarding real)

El alta deja de ser "guardar una ficha vacia". Es un caso de uso transaccional
`ProvisionTenantCommand` que en UNA transaccion:

1. Valida (NIT unico, slug disponible, email admin no duplicado, plan existe,
   moneda compatible).
2. Crea el `Tenant` con estado `Trial` o `Active`.
3. Crea el usuario `Owner` inicial y le envia invitacion (reset de clave).
4. Aplica los limites y feature flags heredados del plan.
5. Siembra configuracion base (modulos del plan, actividades por defecto).
6. Escribe el `AdminAuditLog` del alta.

Si cualquier paso falla, **rollback total**: no quedan tenants a medias (corrige
error #4).

Dos rutas de origen, como en el destino CUBOT:
- **Onboarding asistido** (`IOnboardingService`): el `PlatformAdmin` crea el tenant.
- **Auto-registro** (`ISelfSignupService`, `/auth/register`): la empresa se
  registra sola, queda en `Trial`, elige plan.

### 6.2 Aprovisionamiento por plantilla (clonacion segura)

La clonacion de ECOREX (copiar `sys.tables` con columna SUCURSAL, copiar
formularios con 5 INSERT sueltos) se reemplaza por **plantillas versionadas**:

- `TenantTemplate` define un conjunto reproducible de modulos, actividades,
  formularios y reglas base.
- `ApplyTemplateCommand` clona la plantilla al tenant en una transaccion, con
  remapeo de IDs gestionado por el ORM (no por subqueries a `db3dev.dbo.*`).
- Sin blacklists hardcodeadas: la plantilla declara explicitamente que incluye.

### 6.3 Usuarios multi-tenant

En el origen un usuario pertenecia a UNA empresa (`UPDATE USUARIO.SUCURSAL`). En
el destino, `UserTenant` permite que un usuario (ej. un contador externo) opere
varios tenants con roles distintos en cada uno, cambiando de contexto sin
re-loguearse. El JWT lleva el `tenant_id` activo; cambiar de tenant reemite token.

## 7. Planes, limites y enforcement

Aqui se cierra el error #9. Los parametros dejan de ser decorativos.

- `Plan` versionado (nombre, precios, feature flags, nivel de soporte).
- `PlanLimit` clave/valor con `enforcement_mode` (`hard` bloquea, `soft` alerta):

```txt
max_users, max_activities, max_workflows, max_forms,
max_rules, max_integrations, max_storage_mb, max_api_calls_monthly
```

- `ITenantLimitService.CheckAsync(tenantId, limitKey, delta)` se invoca ANTES de
  toda operacion que consuma un recurso limitado. Un `hard` alcanzado lanza
  `LimitExceededException` (HTTP 402/409); un `soft` deja pasar y emite alerta.
- Los feature flags del plan (`IFeatureManager`) habilitan/ocultan modulos sin
  fragmentar el codigo ni desplegar binarios distintos.

## 8. Estados con consecuencias

En el origen `SUC_ESTADOS` era informativo. En el destino `TenantStatus` es una
maquina de estados que **condiciona el comportamiento**:

| Estado | Efecto funcional |
|---|---|
| `Trial` | Acceso completo, contador de dias, banner de conversion |
| `Active` | Operacion normal |
| `PastDue` | Aviso de pago; acceso completo durante periodo de gracia |
| `Suspended` | Login bloqueado excepto Owner (para regularizar pago) |
| `Archived` | Solo lectura para PlatformAdmin; invisible para el tenant |

Workers programados (`BackgroundService`) revisan vencimientos, aplican
transiciones y notifican. Las transiciones se auditan.

## 9. Auditoria administrativa (error #3)

Toda accion del `PlatformAdmin` y toda transicion de estado de tenant se registra
en `AdminAuditLog`, **inmutable** (append-only, sin UPDATE/DELETE):

```csharp
public class AdminAuditLog {
    public Guid Id { get; set; }
    public Guid ActorId { get; set; }         // quien
    public DateTimeOffset At { get; set; }     // cuando
    public string IpAddress { get; set; }
    public string Action { get; set; }         // TENANT_CREATED, PLAN_CHANGED...
    public Guid? TargetTenantId { get; set; }  // sobre que entidad
    public string? OldValue { get; set; }      // jsonb / nvarchar(max)
    public string? NewValue { get; set; }
    public string? Justification { get; set; }
}
```

El registro es parte de la misma transaccion del comando: si no se puede auditar,
la accion se revierte.

## 10. Integraciones con boveda de secretos (error #6)

Los 11 conectores del origen (Slack, WhatsApp, Evolution, Sigma, Mail, SendGrid,
Azure Blob, AWS, Google Console, Vision IA, API Manager) se conservan como
capacidad, pero:

- Cada integracion es un **plugin** con un schema de configuracion tipado.
- Los secretos (tokens, llaves) se cifran con **ASP.NET Data Protection** y se
  guardan referenciando **Key Vault**; nunca se devuelven en claro tras guardar
  (solo se muestra `****last4`).
- Cada plugin expone `TestConnectionAsync()` — se valida sin ejecutar acciones
  reales.
- Un tenant puede tener varias integraciones activas (se elimina el limite
  artificial `REG=0` del origen).

## 11. Eliminacion segura (error #5)

`Quitarempresa` (DELETE directo sin cascada) se reemplaza por:

1. **Baja logica**: `ArchiveTenantCommand` marca `IsArchived=true`, estado
   `Archived`. El tenant desaparece de la operacion pero los datos se conservan.
2. **Confirmacion fuerte**: la UI exige re-digitar el `PublicSlug` del tenant.
3. **Purga diferida**: un worker (`TenantPurgeService`), pasado el periodo de
   retencion legal, borra en cascada TODAS las entidades del tenant en una
   transaccion (posible gracias a las FK reales que el origen no tenia).
4. Todo el proceso se audita.

## 12. Stack destino y estructura de proyectos

```
src/
  Ecorex.Domain/                 (Tenant, Plan, entidades, sin EF)
  Ecorex.Application/            (comandos: ProvisionTenant, ApplyTemplate, ArchiveTenant...)
  Ecorex.Infrastructure/         (ITenantContext, ITenantLimitService, boveda secretos)
  Ecorex.Infrastructure.SqlServer/  (DbContext + migraciones, RLS SQL Server)
  Ecorex.Infrastructure.Postgres/   (DbContext + migraciones, RLS Postgres)
  Ecorex.Web/                    (Blazor: app de tenant)
  Ecorex.Web.Platform/           (Blazor: consola PlatformAdmin separada)
  Ecorex.Api/                    (REST publica, resolucion de tenant por token)
  Ecorex.Workers/                (vencimientos, purga, notificaciones)
```

- **.NET 10 / ASP.NET Core / EF Core 10 / Blazor**.
- **DAL dual PostgreSQL + SQL Server** por configuracion (`Database:Provider`),
  igual que el vault CUBOT.nails — cada tenant puede vivir en el motor que su
  contrato exija. Ver la nota de DAL dual en el vault CUBOT.nails para el detalle
  del patron `ICubotDbContext` / `IEcorexDbContext`.
- Autenticacion JWT + refresh, claim `tenant_id`, MFA para PlatformAdmin.

## 13. Mapeo de migracion (origen -> destino)

Para el ETL de datos legacy hacia el nuevo esquema:

| Origen (ECOREX) | Destino (.NET 10) | Nota de migracion |
|---|---|---|
| `SUCURSAL.CODIGO` (string) | `Tenant.Id` (Guid v7) + `PublicSlug` | Generar Guid; conservar CODIGO como slug si es unico |
| `SUCURSAL.*` columnas | `Tenant` + owned `ContactInfo`/`LegalRepresentatives` | Normalizar contador/revisor a owned entities |
| `SUCURSAL_MOD` | `TenantModule` | La columna `COSTO` se descarta (pasa a `Plan`) |
| `SUCURSAL_GRUPOS*` | `TenantRoleGroup` + `TenantRolePermission` | Reconstruir permisos como policies |
| `SUCURSAL_PAR` | `TenantParameter` + `PlanLimit` | Separar config libre de limites con enforcement |
| `SUCURSAL_INT` + secretos | `TenantIntegration` + Key Vault | Re-cifrar secretos al importar |
| `TIPO_TAR_EMPRESA` | `TenantActivity` | Directo |
| `USUARIO.SUCURSAL` | `UserTenant` | Permite N tenants por usuario |
| (columna `SUCURSAL` en cada tabla) | `TenantId` (Guid) + `HasQueryFilter` | Reindexar; el filtro deja de ser manual |
| (sin equivalente) | `AdminAuditLog` | Nace vacio; se llena desde la primera accion |

## 14. Checklist de "no heredar errores"

Antes de cerrar la migracion de esta capa, verificar:

- [ ] Ningun repositorio escribe `Where(x => x.TenantId == ...)` a mano (lo hace el filtro global)
- [ ] Ninguna query usa `FromSqlRaw` con interpolacion de input de usuario
- [ ] Todo comando multi-entidad esta envuelto en transaccion
- [ ] RLS activo y probado en ambos motores (test cross-tenant que DEBE fallar)
- [ ] Eliminar tenant es soft + confirmacion + purga diferida (nunca DELETE directo)
- [ ] Secretos de integracion nunca vuelven en claro por la API
- [ ] Toda accion de PlatformAdmin genera fila en `AdminAuditLog`
- [ ] Limites `hard` bloquean de verdad (test que exceda un limite)
- [ ] Cero literales `db3dev`, cero blacklists de tablas en codigo
- [ ] Un usuario puede operar dos tenants sin cruce de datos (test de aislamiento)

## 15. Relacion con otras capas

- [[Visión y entorno|Capa 0 - Vision General]]: contexto de la plataforma destino.
- [[Puntos ciegos y motores transversales|Capa 0 - Puntos ciegos]]: PermissionsManager evoluciona a policies .NET.
- [[00 - Visión Flujos|Capa 3 - Flujos]]: las actividades del tenant arrastran flujos BPMN (motor portado).
- [[00 - Visión Formularios|Capa 4 - Formularios]]: las plantillas de tenant incluyen formularios EAV migrados a jsonb.
- [[NEWFRONT_adm_empresas - Spec para reconstruir|Capa 6 - Spec de ORIGEN]]: comportamiento legacy a preservar como reglas de negocio.
- [[HOJA DE RUTA DESARROLLO|Hoja de Ruta]]: fase donde se implementa esta capa.

---

*Documento de diseno destino. Actualizar a medida que se implementen los comandos,
se definan los limites de plan definitivos y se cierre el ETL de migracion.*
