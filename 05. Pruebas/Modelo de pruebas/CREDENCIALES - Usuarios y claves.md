---
tipo: nota-credenciales
capa: transversal
proposito: Plan de credenciales del entorno Development de ECOREX Tareas (.NET 10) - cuentas demo sembradas, infra local y rutas de login
estado: plan (el sistema destino aun no existe; define lo que los seeders crearan)
aviso_seguridad: REPO PUBLICO - aqui NO van secretos reales; solo defaults demo throwaway + punteros a .env
---

# CREDENCIALES - Usuarios y claves (Development)

> **REPO PUBLICO.** Este documento define el **plan** de credenciales del entorno
> **Development** de ECOREX Tareas destino (.NET 10). El sistema destino aun no
> existe: aqui se especifica lo que los seeders crearan en una BD local vacia.
>
> **Reglas de seguridad de esta nota:**
> - Las cuentas demo usan contrasenas **throwaway** de Development (cambiar en primer
>   acceso). NO son secretos de produccion.
> - Los secretos reales de infra (passwords de Postgres/SQL Server/RabbitMQ, llaves,
>   JWT) viven en `deploy/docker/.env` (no versionado) y en Key Vault — **nunca aqui**.
> - **Cero datos reales de usuarios** del sistema actual (db3dev tiene 105 tenants con
>   PII; esos NO se documentan en un repo publico).
> - En produccion: el PlatformAdmin lo crea el DBA manualmente; los usuarios de tenant
>   entran por onboarding/invitacion. Rotar todas las claves.

---

## 1. PlatformAdmin (consola de plataforma - operador del SaaS)

Rol maximo, fuera de cualquier tenant. Se siembra en Development (`SeedIfEmptyAsync`,
ver [[HOJA DE RUTA DESARROLLO]] seccion 3.3). Consola separada con MFA (ver
[[Seguridad y Autenticacion multi-tenant]]).

| Campo | Valor |
|---|---|
| **Consola** | `http://localhost:<puerto Web.Platform>` -> `/login` |
| **Email** | `admin@ecorex.local` |
| **Clave** | `Admin123*` (demo Development; MFA obligatorio en real) |
| Rol | `PlatformAdmin` (platform_role) |
| Tenant | ninguno (cross-tenant auditado) |

## 2. Administrador del tenant demo

Owner del tenant demo sembrado. El tenant demo replica al **`01` = BITCODE** del
sistema actual (ver [[Conexion a la BD del sistema actual (db3dev)]]).

| Campo     | Valor                                       |
| --------- | ------------------------------------------- |
| **App**   | `http://localhost:<puerto Web>` -> `/login` |
| **Email** | `owner@sky-system.local`                    |
| **Clave** | `Demo123*` (demo Development)               |
| Rol       | `Owner` (tenant_role)                       |
| Tenant    | SKY SYSTEM (demo)                           |

## 3. Usuarios demo por rol (sembrados)

Un usuario por rol para probar policies y aislamiento. Todos con clave demo
`Demo123*`, en el tenant SKY SYSTEM.

| Rol (tenant_role) | Email login | Ve |
|---|---|---|
| `Owner` | `owner@sky-system.local` | Todo el tenant |
| `Admin` | `admin@sky-system.local` | Casi todo (sin borrar ni roles) |
| `Operator` | `operator@sky-system.local` | Operacion diaria (tareas, tableros) |
| `Viewer` | `viewer@sky-system.local` | Solo lectura |
| `IA` | (interno, sin login humano) | Rol que usan los agentes de IA |

> Los roles se derivan de los grupos del legacy (`SUCURSAL_GRUPOS*` +
> `PermissionsManager`) a **policies .NET**. Ver [[Gestion de Empresas - Admin multi-tenant]].

## 4. Acceso al sistema ORIGEN (legacy, referencia)

Para descubrir/validar el sistema actual (no es el destino):

- **App legacy**: `http://localhost:47640/GestionMovil/.../LoginBitcode.aspx`.
- **Usuario demo**: `1048064705` (interno `GUADALUPE.LA`), clave temporal `12345`
  (fuerza cambio al login). Documentado en [[Visión y entorno]] seccion 18.
- **BD del sistema actual** (solo lectura, credenciales fuera del repo):
  [[Conexion a la BD del sistema actual (db3dev)]].

## 5. Infraestructura local (Docker) - passwords en `.env`

Bloque de puertos DEDICADO de ECOREX Tareas (ver [[HOJA DE RUTA DESARROLLO]] seccion 3).
Los usuarios/claves reales van en `deploy/docker/.env`, **no aqui**.

| Servicio | URL / Puerto | Usuario | Clave |
|---|---|---|---|
| **PostgreSQL** | `localhost:5442` (db `ecorex_dev`) | `<en .env>` | `<en .env>` |
| **SQL Server** | `localhost:1443` (db `ecorex_dev`) | `sa` | `<en .env>` |
| **Redis** | `localhost:6389` | - | - |
| **RabbitMQ** | `localhost:5682` / UI `http://localhost:15682` | `<en .env>` | `<en .env>` |
| **Adminer** | `http://localhost:8092` | (segun motor) | `<en .env>` |

> `Database:Provider` (`Postgres`/`SqlServer`) elige el motor activo. Ambos pueden
> estar arriba en local para la matriz de tests dual (ver [[Estrategia de Testing (.NET 10)]]).

## 6. Puertos de las apps (Development)

| App | Puerto | Notas |
|---|---|---|
| **Ecorex.Web** (app del tenant) | `http://localhost:5140` | Login `/login`, tableros `/tareas` |
| **Ecorex.Web.Platform** (PlatformAdmin) | `http://localhost:5141` | Consola separada + MFA |
| **Ecorex.Api** | `http://localhost:5142` | REST + webhooks + JWT |

> Puertos de app sugeridos (ajustar en `launchSettings.json`); no chocan con el bloque de infra.

## 7. Compilar y correr en local

```powershell
# stack docker (si esta caido)
cd C:\DesarrolloIA\ECOREX.tareas\deploy\docker
.\preflight.ps1            # valida puertos libres + docker + sin colision de nombres
docker compose up -d

# build + run
cd C:\DesarrolloIA\ECOREX.tareas
dotnet build
dotnet run --project src/Ecorex.Web

# tests en AMBOS motores (Postgres + SQL Server)
dotnet test
```

## 8. Como regenerar los datos demo

Los seeders solo corren en **Development** y son **idempotentes** (ver
[[HOJA DE RUTA DESARROLLO]] seccion 3.3):

- `SeedIfEmptyAsync` -> PlatformAdmin + tenant demo SKY SYSTEM + planes (solo si vacio).
- Roles/policies base derivados del registro de modulos.
- Usuarios demo por rol (seccion 3).

```powershell
# reset limpio (borra el volumen de datos)
cd C:\DesarrolloIA\ECOREX.tareas\deploy\docker
docker compose down -v
docker compose up -d
cd C:\DesarrolloIA\ECOREX.tareas
dotnet run --project src/Ecorex.Web    # los seeders recrean todo
```

## 9. Politica de claves

- Minimo 12 caracteres en produccion; demo Development >= 6.
- Hash **Argon2id** (o bcrypt cost 12), salt por usuario (ver [[Seguridad y Autenticacion multi-tenant]]).
- **MFA obligatorio** para PlatformAdmin.
- Refresh tokens con rotacion; forzar cambio en primer acceso de cuentas sembradas.
- Secretos de integracion (Evolution/SendGrid/Slack/IA) en Key Vault, nunca en claro.

## 10. Login: rutas de acceso

El login resuelve el tenant y aplica policies (ver [[Gestion de Empresas - Admin multi-tenant]] seccion 3):

| Perfil | Destino post-login |
|---|---|
| `PlatformAdmin` | consola de plataforma (`Web.Platform`) |
| `Owner`/`Admin`/`Operator`/`Viewer` | app del tenant (`Web`), vista segun policies |

Campo de tenant: se resuelve por claim JWT / subdominio; si el usuario pertenece a
varios tenants, se elige al iniciar sesion (`UserTenant`, multi-tenant real).

---

## 11. Tenants cliente en PRODUCCION (onboarding desde db3dev, 2026-07-09)

Mapeo: SUCURSAL `01` -> **BITCODE**, `00136` -> **SKY SYSTEM**; tercer tenant
**agrometalicas** creado a mano. Todos rol **Owner**.
**Regla de acceso: login = correo corporativo, clave = cedula del usuario.**

App de prod: `http://10.0.0.3:5480/login`

Un usuario **validado** por tenant (los demas del tenant siguen la misma regla):

| Tenant | Email (login) | Clave (cedula) | Rol |
|---|---|---|---|
| BITCODE | `acuartas@bitcode.com.co` | `80001976` | Owner |
| SKY SYSTEM | `adriana.borrero@skysystem.com.co` | `51888215` | Owner |
| agrometalicas | `calidad@agrometalicas.com` | `1116243150` | Owner |

Super Admin de plataforma (prod): `admin@ecorex.local` (la clave es el secreto
`ECOREX_SEED_ADMIN_PASSWORD` del `.env` del server, **NO** se documenta aqui).

La lista completa de usuarios por tenant se deriva de db3dev (sucursal 01 / 00136);
no se transcribe aqui para no multiplicar PII. Ver [[Conexion a la BD del sistema actual (db3dev)]].

---

> Documento de credenciales de PRUEBAS (Development). **Mantener los secretos reales
> fuera de este repo publico** (`.env` + Key Vault). En produccion: rotar todas las
> claves, MFA en PlatformAdmin, y crear los usuarios reales por onboarding/invitacion.

## Enlaces

- [[HOJA DE RUTA DESARROLLO]] — setup, puertos, seeders
- [[Seguridad y Autenticacion multi-tenant]] — auth, MFA, politica de claves
- [[Conexion a la BD del sistema actual (db3dev)]] — acceso al legacy (solo lectura)
- [[Estrategia de Testing (.NET 10)]] — matriz dual que usa este entorno
- [[Gestion de Empresas - Admin multi-tenant]] — roles y policies
