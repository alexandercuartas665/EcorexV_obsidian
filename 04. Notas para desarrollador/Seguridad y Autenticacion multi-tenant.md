---
tipo: nota-arquitectura
capa: transversal
proposito: Modelo de seguridad y autenticacion del sistema DESTINO ECOREX Tareas (.NET 10) - JWT, refresh, MFA, policies, RLS, secretos
estado: diseno objetivo de migracion
---

# Seguridad y Autenticacion multi-tenant

> Consolida la seguridad del sistema DESTINO, hoy dispersa entre la
> [[HOJA DE RUTA DESARROLLO]] (seccion 5), [[Puntos ciegos y motores transversales]]
> (Login, PermissionsManager) y [[Gestion de Empresas - Admin multi-tenant]]
> (aislamiento). El ORIGEN autentica con `LoginBitcode.aspx` + `Session(...)` y cifra
> config con una key embebida en `CargaConfig.vb:7` — errores que el destino corrige.

## 1. Autenticacion - flujo de login

```
POST /auth/login  { email, password, tenant_hint? }
   -> Resolver tenant (JWT claim / hint / subdominio); si el usuario tiene N tenants, exige eleccion
   -> Validar credenciales (Argon2id o bcrypt cost 12) + rate limit por IP (5/min)
   -> Emitir: access_token (JWT, 15 min) + refresh_token (opaco, 30 dias, hasheado) + CSRF cookie
```

Reemplaza el `Session("Nombre")`/`Session("Empresa")` del origen (estado de servidor
fragil, base del bug `Session("Emmpresa")` documentado en la spec de web_scraping).

## 2. Claims del JWT

```json
{ "sub": "user-guid-v7", "tenant_id": "tenant-guid-v7", "tenant_role": "Owner|Admin|Operator|...",
  "platform_role": "PlatformAdmin|null", "email": "...", "iat": 0, "exp": 0, "jti": "..." }
```

**Regla dura**: el `tenant_id` del JWT alimenta el `ITenantContext`, que alimenta el
`HasQueryFilter` global. Ningun usuario ve datos de otro tenant aunque manipule el
request. Ver [[Gestion de Empresas - Admin multi-tenant]] seccion 3.

## 3. Refresh tokens

Tabla `refresh_token` (id, user_id, token_hash SHA-256, device, ip, issued_at,
expires_at, revoked_at, revoked_reason). **Rotacion obligatoria**: cada refresh emite
uno nuevo y revoca el anterior; si aparece un token ya revocado -> revocar toda la
familia + alerta (posible robo). Revocacion en cambio de contrasena, cambio de rol y
logout.

## 4. Autorizacion por policies (evolucion de PermissionsManager)

Los permisos del origen (`PermissionsManager` + menu `000109` + `SUCURSAL_GRUPOS*`)
se derivan a **policies .NET** dinamicas:

```csharp
[Authorize(Policy = "Flujos.Editar")]      // en API/Blazor
public async Task<IResult> UpdateFlow(...) { ... }
```

- `PlatformAdmin`: rol de plataforma, consola separada, MFA obligatorio, acceso
  cross-tenant SOLO por `IPlatformDbContext` auditado.
- Roles de tenant (`Owner/Admin/Operator/...`): se mapean desde los grupos migrados.
- Las policies se registran leyendo el registro de modulos (`Module Registry` 000109).

## 5. MFA para PlatformAdmin

La consola de plataforma exige segundo factor (TOTP/WebAuthn), timeout de sesion
corto y re-autenticacion para acciones criticas (crear/suspender tenant, cambiar
plan). Ninguna accion sensible se ejecuta sin quedar en `AdminAuditLog`.

## 6. Aislamiento en profundidad

1. `TenantResolutionMiddleware` resuelve el tenant o rechaza (400) — nunca default silencioso.
2. `HasQueryFilter` global en EF Core (defensa principal).
3. **RLS** en la BD (Postgres `CREATE POLICY`, SQL Server `SECURITY POLICY`) por si un
   reporte/job salta EF Core.
4. Object Storage y Redis con prefijo por `tenant_id`.

## 7. Secretos (corrige el error del origen)

- Integraciones (Evolution, SendGrid, Slack, llaves IA) cifradas con **ASP.NET Data
  Protection**; referencias en **Key Vault**; nunca el valor en `appsettings` ni en logs.
- Reemplaza el `Config.xml` con TripleDES y **key embebida en `CargaConfig.vb:7`**
  (ver [[CargaConfig - Decode Config.xml]]). El secreto ya no vive en el codigo.
- Rotacion: API keys c/6 meses, passwords de BD c/3 meses.

## 8. Rate limiting

Por plan (ver [[Gestion de Empresas - Admin multi-tenant]]): Basic/Pro/Enterprise con
distinto `TokenBucket` por tenant. Endpoints publicos (login, webhooks) con limite por
IP. Cupo de tokens de IA aparte (ver [[Agentes de IA - Arquitectura y Operacion]]).

## 9. Contra-medidas

| Ataque | Defensa |
|---|---|
| SQL Injection | EF Core parametrizado (elimina el SQL concatenado sistemico del origen) |
| XSS | Blazor escapa por default + CSP estricto |
| CSRF | Antiforgery token (double-submit) |
| Fuga cross-tenant | Filtro global + RLS + test de aislamiento que DEBE fallar |
| Robo de refresh token | Rotacion + deteccion de re-uso |
| Enumeracion en login | Mensaje generico + comparacion constant-time |
| Secreto filtrado | Key Vault + DataProtection, nunca en codigo/logs |

## 10. Auditoria y compliance

`AdminAuditLog` inmutable (append-only) para toda accion de PlatformAdmin; retencion
2 anos (Ley 1581 Colombia). Derecho a eliminacion y portabilidad de datos por tenant.

## 11. Testing de seguridad

Ver [[Estrategia de Testing (.NET 10)]]: suite de aislamiento cross-tenant en ambos
motores, tests de auth flow, `dotnet list package --vulnerable` + CodeQL por PR, OWASP
ZAP semanal contra staging.

## 12. Enlaces

- [[Gestion de Empresas - Admin multi-tenant]] — aislamiento e invariante
- [[Puntos ciegos y motores transversales]] — Login y PermissionsManager (origen)
- [[CargaConfig - Decode Config.xml]] — el secreto embebido a corregir
- [[Agentes de IA - Arquitectura y Operacion]] — cupos y guardrails de IA
- [[Estrategia de Testing (.NET 10)]] — pruebas de seguridad
