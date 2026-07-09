---
tipo: nota-desarrollador
proposito: Como conectar tu entorno de desarrollo a una base de datos (docker local o la BD de produccion) sin exponer credenciales
estado: vivo
fecha: 2026-07-08
---

# Conexion a la base de datos (dev y prod)

> Guia para levantar el entorno de ECOREX.tareas y elegir contra que BD trabaja
> tu maquina. **Regla de oro: ninguna credencial vive en el repo (es publico).**
> Las cadenas reales van SIEMPRE en archivos gitignored o variables de entorno;
> aqui solo hay placeholders `<...>`. Pide las credenciales al lider del equipo.
> Relacionado: [[Seguridad y Autenticacion multi-tenant]],
> [[Conexion a la BD del sistema actual (db3dev)]].

---

## 1. Como resuelve la app la cadena de conexion

`Ecorex.Infrastructure/DependencyInjection.cs` elige la cadena en este orden:

1. `ConnectionStrings:Default` de la configuracion (appsettings + overrides).
2. Si esta vacia, la variable de entorno `ECOREX_DB_CONNECTION`.
3. El proveedor (Postgres | SqlServer) sale de `Database:Provider` (default Postgres).

`appsettings.json` trae `ConnectionStrings:Default` **vacio** a proposito, para
que cada quien defina la suya sin tocar archivos versionados.

### Override local NO versionado

`Program.cs` carga, despues de los appsettings versionados:

```
builder.Configuration.AddJsonFile("appsettings.Development.local.json", optional: true, reloadOnChange: false);
```

Ese archivo esta en `.gitignore` (patron `appsettings.*.local.json`) y **tiene
prioridad** sobre `appsettings.json`. Es el lugar correcto para tu cadena local.
Si no existe, la app sigue leyendo `ECOREX_DB_CONNECTION` (lo que setea
`.claude/launch.json` para el docker local).

---

## 2. Opcion A - Postgres local en Docker (RECOMENDADO para ramas de feature)

Aislado, sin depender de la red ni pisar datos de nadie. Es lo mejor para una
rama como `formularios`.

```powershell
cd deploy\docker
.\preflight.ps1
docker compose up -d      # levanta ecorex-tareas-postgres en el puerto host 5442
```

Cadena (ya la setea `.claude/launch.json`, no necesitas archivo local):

```
Host=localhost;Port=5442;Database=ecorex_dev;Username=ecorex;Password=<PASSWORD_DOCKER_LOCAL>
```

En modo Development la app **siembra datos demo** al arrancar (tenant SKY SYSTEM,
usuarios `owner@sky-system.local` / `admin@sky-system.local` / etc., flujos
COT-COM, formularios). Ideal para desarrollar y probar sin tocar prod.

---

## 3. Opcion B - Conectar el dev a la BD de PRODUCCION (via tunel SSH)

La BD de prod corre en el server de produccion dentro de Docker y **no esta
publicada** al exterior. Se alcanza solo por un tunel SSH.

> AVISO: trabajar el dev contra prod significa que ves y modificas datos REALES,
> y que (en modo Development) una migracion nueva local se APLICA a prod al
> arrancar. Usalo solo si sabes lo que haces. Para features normales, usa la
> Opcion A.

### 3.1 Abrir el tunel

**El tunel debe estar arriba ANTES de arrancar el dev.** Mientras la ventana del
tunel siga abierta, tu `localhost:15433` apunta a la BD de prod; si la cierras, el
dev se queda sin BD.

**Forma recomendada: ejecutar el script `Script\abrir-tunel-prod.ps1`.** Detecta sola
la IP del contenedor de Postgres (que cambia si se recrea) y abre el tunel. Es un
archivo **LOCAL, gitignored** (no viene en el repo porque tiene el host del server):
creas el archivo con este contenido y reemplazas `<USUARIO>`, `<PROD_HOST>` y el
nombre de la llave por los reales (pidelos al lider):

```powershell
# Script\abrir-tunel-prod.ps1  -> USO:  pwsh -File .\Script\abrir-tunel-prod.ps1
$Server='<USUARIO>@<PROD_HOST>'; $KeyPath=Join-Path $HOME '.ssh\<TU_LLAVE_SSH>'
$LocalPort=15433; $PgContainer='ecorex-postgres-prod'; $PgPort=5432
if (-not (Test-Path $KeyPath)) { Write-Host "Falta la llave $KeyPath" -ForegroundColor Red; Read-Host; exit 1 }
if (Get-NetTCPConnection -LocalPort $LocalPort -State Listen -ErrorAction SilentlyContinue) { Write-Host "Puerto $LocalPort ya en uso (tunel abierto)"; Read-Host; exit 0 }
$tmpl='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
# -n desconecta stdin (si no, ssh se cuelga esperando teclado en consola interactiva)
$raw = & ssh -n -o BatchMode=yes -i $KeyPath -o StrictHostKeyChecking=accept-new -o ConnectTimeout=15 $Server "docker inspect -f '$tmpl' $PgContainer" 2>&1
$PgIp = ("$raw" -split "`r?`n" | Where-Object { $_ -match '^\s*\d{1,3}(\.\d{1,3}){3}\s*$' } | Select-Object -First 1)
if ([string]::IsNullOrWhiteSpace($PgIp)) { Write-Host "No obtuve la IP: $raw" -ForegroundColor Red; Read-Host; exit 1 }
$PgIp = $PgIp.Trim()
Write-Host "Tunel: localhost:$LocalPort -> $PgIp (deja ESTA ventana abierta)" -ForegroundColor Green
& ssh -n -i $KeyPath -N -o StrictHostKeyChecking=accept-new -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes -L "${LocalPort}:${PgIp}:${PgPort}" $Server
```

> [!warning] Antivirus (Kaspersky)
> Kaspersky puede poner en **cuarentena** este `.ps1` (lo lee como hacktool de tunel)
> y/o **colgar `ssh.exe`** al ejecutarse. Sintomas: el archivo desaparece, o el script
> se queda en "Conectando por SSH...". Solucion: agrega exclusiones en Kaspersky
> (Amenazas y exclusiones) para la carpeta `Script\` **y** para el ejecutable
> `C:\WINDOWS\System32\OpenSSH\ssh.exe` (marca "no analizar trafico de red"). Si tu
> Kaspersky lo administra IT central, pideselo a IT. Mientras tanto puedes pausar el AV.

**Forma manual (fallback, sin archivo):** un one-liner pegado en PowerShell (Kaspersky
molesta menos con un comando tecleado que con un archivo guardado). La IP se consulta
con `docker inspect ecorex-postgres-prod` en el server:

```powershell
ssh -n -i <RUTA_LLAVE_SSH> -N -L 15433:<IP_POSTGRES_PROD>:5432 <USUARIO>@<PROD_HOST>
```

### 3.2 Crear `appsettings.Development.local.json` (GITIGNORED)

En `apps/backend/src/Ecorex.SuperAdmin/appsettings.Development.local.json`:

```json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Port=15433;Database=<PROD_DB>;Username=<PROD_USER>;Password=<PROD_PASSWORD>"
  },
  "Database": { "Provider": "Postgres" },
  "Ecorex": { "SkipDemoSeed": true }
}
```

### 3.3 SkipDemoSeed - no contaminar prod con datos demo

En modo Development la app sembraria demo en cada arranque. El guard de
`Program.cs` lo evita cuando apuntas a una BD real:

- `Ecorex:SkipDemoSeed: true` (en el archivo de arriba), o
- variable de entorno `ECOREX_SKIP_DEMO_SEED=true`.

Con el guard activo solo corren migraciones + el tenant de plataforma
(idempotentes); NO se crea ningun dato demo.

### 3.4 Verificar

Arranca la app (o el preview 5234) y entra con el usuario que existe **solo en
prod** (`<SUPERADMIN_PROD_EMAIL>`). Si el dashboard muestra los datos de prod,
la conexion esta OK. Latencias de decenas/cientos de ms en las queries (vs pocos
ms en local) confirman que hablas con la BD remota.

### 3.5 Caveats de la Opcion B

- **El tunel debe estar arriba** siempre que desarrolles; si se cae, el dev se
  queda sin BD. Para dejarlo permanente: `autossh` o una tarea programada.
- **Migraciones fluyen a prod**: en Development, `MigrateAsync` corre al arrancar.
  Coordina los cambios de esquema con el equipo.
- **BD compartida**: si varios devs apuntan a la misma prod, se pisan datos. No
  es un entorno aislado.

---

## 4. Credenciales - de donde salen (NUNCA del repo)

| Dato | Donde vive |
|------|------------|
| Password del Postgres local (docker) | `.claude/launch.json` (password de docker, bajo riesgo) |
| Cadena/credencial de la BD de prod | te la pasa el lider del equipo; va en tu `appsettings.Development.local.json` (gitignored) |
| Host y llave SSH del server | te los pasa el lider; la llave va en tu `~/.ssh/`, nunca en el repo |
| Secretos del server (POSTGRES_PASSWORD, etc.) | `/opt/ecorex/.env` en el server (chmod 600) |

Si necesitas acceso a prod, pidelo; no busques credenciales en el codigo ni las
pegues en archivos versionados.

---

## 5. Trabajar en una rama nueva (ej. `formularios`)

```bash
git fetch origin
git checkout -b formularios origin/fase-0/clon-backbone   # partir del tip actual del backbone
```

- Para desarrollar formularios lo mas comodo y seguro es la **Opcion A** (docker
  local con datos demo): pruebas el constructor y el render sin riesgo.
- Si necesitas ver datos reales de prod, usa la **Opcion B** con `SkipDemoSeed`.
- Al terminar, `push origin formularios` y abre PR contra `fase-0/clon-backbone`
  (la rama viva; `main` en GitHub esta protegida).

---

## 6. Referencia rapida

- Rama viva del proyecto: `fase-0/clon-backbone` (de ahi se construye prod).
- Deploy de prod: build-from-git en el server (ver la bitacora `PROGRESO.md` del repo de codigo).
- Bloque de puertos del docker local: ver `CLAUDE.md` seccion 4 del repo de codigo.
