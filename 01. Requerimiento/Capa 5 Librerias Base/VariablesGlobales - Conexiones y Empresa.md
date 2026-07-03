---
tipo: ficha-modulo
modulo_vb: VariablesGlobales
archivo: C:\Desarrollo\core\MotherData\Datos\VariablesGlobales.vb
target_origen: .NET Framework 4.8.1
reemplazo_destino: ITenantContext (scoped) + ITenantConfigRepository
estado: adaptado a vision destino
---

# VariablesGlobales -- estado global de MotherData -> `ITenantContext` scoped

VB Module (estado estatico a nivel de AppDomain) que mantiene **las conexiones,
los alias de base de datos y los datos del tenant activo**. Todo el sistema
lee/escribe contra estas estructuras. Es el mayor obstaculo multi-tenant del
origen: el "tenant activo" es global al proceso. En el destino .NET 10 su rol se
divide en un **`ITenantContext` inyectado por request** (scoped) mas un
**`ITenantConfigRepository`** (config desde BD, no XML). Ver
[[00 - Visión MotherData|00 - Vision MotherData]] y [[Visión y entorno|Vision y entorno]] seccion 4.

## 1. Que mantenia (estado global del origen)

| Variable | Tipo | Contenido |
|---|---|---|
| `Conexion(0)` | `LocalConexion()` | Conexiones disponibles. Crece al cargar `Config.xml`. |
| `AliasCon(0)` | `aliasConexion()` | Tabla de alias `[dbx.X] -> catalogo_fisico`. Crece al cargar `alias.xml`. |
| `mbase_empresa` | `empresa` | Tenant activo: NOMBRE, LOGO, BASE, LOGIN, MAIL, SUCURSAL. |
| `Usuario` | String | Usuario actual (tambien en `Session("Usuario")`). |
| `Director` / `NDirector` / `Concesion` / `Bodega` | String | Variables legacy. |
| `Table` | DataTable | Buffer multiproposito (anti-patron). |
| `Desarrollo` | Integer | Flag de modo dev. |
| `manejoLogBase` / `manejoLogFiles` | Integer | Flags de logs en `Temp\My*.txt`. |
| `valor_servicio` | String | Buffer compartido legacy. |

## 2. Estructuras (origen)

```vb
Public Structure LocalConexion
    Public Base As String        ' alias logico
    Public Catalogo As String    ' catalogo fisico
    Public Servidor, RutaDb, Usuario, Pw, Tipo, Objeto, Puerto As String
    Public servidor_rep, servidor_dom, servidor_usu, servidor_cla As String  ' reporting
End Structure

Public Structure aliasConexion
    Public base As String
    Public nombre As String        ' literal [dbx.NOMBRE] que aparece en queries
    Public catalogo As String      ' nombre real de la BD
End Structure

Public Structure empresa
    Public nombre, logo, pre, pos As String
    Public base_pricipal As String      ' <-- usado como BASE_SISTEMA en cada pagina
    Public base_login, base_loginnew As String
    Public logo_file, cuenta_mail, sucursal As String
End Structure
```

## 3. Subs de carga (origen)

| Sub | Lee | Llena |
|---|---|---|
| `PrepararBases()` | (compuesto) | `carga_base()` + `carga_alias()`. Invocado por el constructor de `AdmDatos`. |
| `carga_base()` | `Datos\Config.xml` (cifrado) | `Conexion()` |
| `carga_alias()` | `Datos\alias.xml` (texto) | `AliasCon()` |
| `carga_empresa()` | `Datos\Empresa.xml` (texto) | `mbase_empresa` |
| `carga_empresaii(empresa)` | `Empresa.xml` + lookup en `[dbx.GENE].dbo.SUCURSAL_PAR` | `mbase_empresa` con `base_login` per-tenant |

Inicializacion tipica al arrancar un `.aspx.vb`:

```
Page.New() {
    Call carga_empresa()                ' Empresa.xml -> mbase_empresa.base_pricipal
    BASE_SISTEMA = mbase_empresa.base_pricipal
}
Dim tbrec As New MotherData.AdmDatos    ' su New() llama PrepararBases() la 1a vez
'   (las cargas son idempotentes: solo corren si los arrays estan vacios)
```

## 4. Riesgos (por que este patron NO se porta)

| Riesgo | Detalle |
|---|---|
| **Estado compartido entre tenants** | `Conexion()` y `AliasCon()` son unicos por proceso IIS. |
| **`mbase_empresa` es por AppDomain** | El tenant activo es global: `carga_empresa()` no recarga si `nombre IsNot Nothing`, asi que un request concurrente puede leer el tenant del request anterior. Mitigado en la practica por 1 sitio/pool IIS por tenant -- fragil. |
| **`Table` / `valor_servicio`** | Buffers globales reutilizados; no tocar en codigo nuevo. |
| **`carga_empresaii` ejecuta SQL al arrancar** | Lookup en `SUCURSAL_PAR`; si la BD esta caida en boot, la app falla feo. |

Este es exactamente el aislamiento "por disciplina" que [[Visión y entorno|Vision y entorno]]
seccion 4 convierte en invariante del sistema.

## 5. Como se reemplaza en .NET 10

| Origen | Destino (.NET 10) |
|---|---|
| Module + arrays globales (`Conexion`, `AliasCon`) | `ITenantContext` **scoped por request** (DI); sin estado compartido |
| `mbase_empresa` (tenant activo global) | `tenant_id` en el JWT + middleware de resolucion de tenant + claims |
| `carga_base()` desde `Config.xml` cifrado | `ITenantConfigRepository` sobre tabla `tenant_database_config` (EF Core) |
| `carga_alias()` desde `alias.xml` | `IConnectionStringResolver` (alias `[dbx.X]` -> proveedor + catalogo) |
| `carga_empresa()` / `carga_empresaii()` | carga del tenant por request, cacheada en Redis |
| `WHERE SUCURSAL = ...` implicito | `HasQueryFilter(x => x.TenantId == _tenant.CurrentTenantId)` + RLS en BD |

- El aislamiento deja de depender del desarrollador: filtro global EF Core + RLS
  como defensa en profundidad (Postgres `CREATE POLICY`, SQL Server `SECURITY POLICY`).
- Mantener `[dbx.X]` solo como convencion de resolucion en infraestructura, sin
  que viaje en el SQL de aplicacion.

## 6. Relacionado

- [[00 - Visión MotherData|00 - Vision MotherData]] -- vision de la capa.
- [[CargaConfig - Decode Config.xml]] -- provee el XML que `carga_base` consumia.
- [[AdmDatos - Motor SQL Server]] -- consumidor de `Conexion()` / `AliasCon()`.
