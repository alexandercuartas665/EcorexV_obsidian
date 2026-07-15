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

| Tenant        | Email (login)                      | Clave (cedula) | Rol   |
| ------------- | ---------------------------------- | -------------- | ----- |
| BITCODE       | `acuartas@bitcode.com.co`          | `80001976`     | Owner |
| SKY SYSTEM    | `adriana.borrero@skysystem.com.co` | `51888215`     | Owner |
| agrometalicas | `calidad@agrometalicas.com`        | `1116243150`   | Owner |

Super Admin de plataforma (prod): `admin@ecorex.local` (la clave es el secreto
`ECOREX_SEED_ADMIN_PASSWORD` del `.env` del server, **NO** se documenta aqui).

El **directorio completo de usuarios por tenant** (con el `tenant_id` de cada empresa)
esta en la **seccion 12**. Las cedulas individuales NO se transcriben (repo publico);
son la clave y se derivan de db3dev. Ver [[Conexion a la BD del sistema actual (db3dev)]].

---

## 12. Directorio de usuarios por tenant en PRODUCCION (2026-07-14)

> Estado real de `http://10.0.0.3:5480/login` a 2026-07-14, un renglon por tenant con su
> `tenant_id`. **REGLA DE ACCESO UNICA de los usuarios migrados/reales**: login = el correo
> de la lista; **clave = documento de identidad (cedula) del usuario** (consta en db3dev),
> que cada quien cambia en el primer ingreso.
>
> **Las cedulas NO se transcriben aqui: el repo es PUBLICO** (regla de la seccion 11 / de la
> nota de db3dev). Hay 3 ejemplos listos-para-usar en la seccion 11; el resto se deriva de
> db3dev. Excepcion: los usuarios demo `*@sky-system.local` usan la clave throwaway `Demo123*`.

Todos `Active`. Cada usuario quedo con la vista de menu **Completo** (todas las opciones),
salvo donde se indique.

### BITCODE - `019f478d-1964-7892-89de-36a619b28e4b` (Standard, sucursal 01) - 13 usuarios, todos Owner
`acuartas@bitcode.com.co`, `jmolina@bitcode.com.co`, `kelly@bitcode.com.co`,
`lloaiza@bitcode.com.co`, `programador1@bitcode.com.co`, `soporte2@bitcode.com.co`,
`soporte3@bitcode.com.co`, `bettyfbenavides@gml.com.co`, `jackelinemarin@flecto.com.co`,
`stefania@flecto.com.co`, `andresdesarrollador4@gmail.com`, `diasflac1213@gmail.com`,
`valeriauribecbo@gmail.com`

### SOLDARCO - `e3519cc4-150f-4f63-a0cd-21eb9d59f1fa` (Standard, sucursal 02) - 25 usuarios, todos Owner (migrados 2026-07-14)
`recepcion@soldarco.com`, `gerenciacomercial@soldarco.com`, `cartera@soldarco.com`,
`cartera2@soldarco.com`, `soporteventas1@soldarco.com`, `soporteventas2@soldarco.com`,
`comercial3@soldarco.com`, `comercial6@soldarco.com`, `comercial7@soldarco.com`,
`comercial9@soldarco.com`, `comercial14@soldarco.com`, `comercial18@soldarco.com`,
`contabilidad@soldarco.com`, `gestion@soldarco.com`, `auxiliaradmin@soldarco.com`,
`administracion@soldarco.com`, `almacen@soldarco.com`, `postventa@soldarco.com`,
`serviciotecnico1@soldarco.com`, `cristianddiaz@bitcode.com.co`, `freiderclondono@bitcode.com.co`
(el correo real lleva enie), `hectorfrivera@bitcode.com.co`, `lauraccamila@bitcode.com.co`,
`sharonfruiz@bitcode.com.co`, `gdiaz@autoliderazgoyliderazgodequipos.com`

### AGROMETALICAS - `019f478d-6428-7283-a5cd-b7e35f802ef3` (Standard) - 1 usuario, Owner
`calidad@agrometalicas.com`

### CHUZO DE IVAN - `72450050-a07d-40a1-9df0-f611cfcfa48b` (Standard) - 0 usuarios
Tenant recien creado (2026-07-14); aun sin usuarios (el cliente no los ha entregado).

### PLATAFORMA ECOREX - `019f2d5f-cc42-7492-9eba-0217996338b1` (Internal) - 1 usuario
`admin@ecorex.local` (Owner). Es tambien el Super Admin de plataforma; su clave es el secreto
`ECOREX_SEED_ADMIN_PASSWORD` del `.env` del server, **NO** una cedula.

### SKY SYSTEM - `019f2d5f-c4f3-737b-89dc-902eeebdb74b` (Demo/QA) - 18 usuarios
- **Reales** (clave = cedula): `adriana.borrero@skysystem.com.co`, `fabian.perez@skysystem.com.co`,
  `felipe.galeano@skysystem.com.co`, `josemiguel.buritica@skysystem.com.co`,
  `mauricio.borrero@skysystem.com.co`, `mercadeo@skysystem.com.co` (todos Owner).
- **Demo/seed** (clave `Demo123*`): `owner@` (Owner), `admin@` (Admin), `operator@`/`viewer@`/
  `completo@`/`qa.usuario@` (Advisor) - todos con menu Completo; `simple@sky-system.local`
  (Advisor) con vista **Simple** a proposito (para demostrar el filtrado de menus).
- **QA E2E** (throwaway): `e2e-14ccd408@`, `e2e-50420b70@`, `e2e-934001c0@`, `e2e-c0c2eaf5@`
  `@sky-system.local` (Supervisor). Mas `correo@correo.com` (Owner), dato de prueba.

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
