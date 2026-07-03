---
tipo: vision-libreria
libreria: MotherData
proyecto_path: C:\Desarrollo\core\MotherData\
namespace_raiz: MotherData (sin namespace explicito en clases -- todo cuelga del default del .vbproj)
target_origen: .NET Framework 4.8.1
target_destino: .NET 10 / EF Core 10
estado: adaptado a vision destino
---

# MotherData -- del DAL legacy al DAL dual `IEcorexDbContext`

> [!success] Que era y en que se convierte
> **Origen:** libreria VB.NET (.NET Framework 4.8.1) que provee acceso a datos
> uniforme para SQL Server (`AdmDatos`) y PostgreSQL (`AdmNpgsql`), mas carga de
> configuracion cifrada (`CargaConfig` + `AdmCrypto`) y un modulo de variables
> globales que mantiene en memoria las conexiones, alias de base de datos y datos
> de empresa por tenant. La consumen Visal, DokTrino, ECOREX/Tareas, CUBOT y todo
> proyecto que hereda de `MasterFormsII`.
>
> **Destino (.NET 10):** MotherData es el **ancestro directo del DAL dual** de
> ECOREX. No se reescribe desde cero: su concepto -- un mismo API sobre dos
> motores -- se formaliza en la abstraccion `IEcorexDbContext` con
> implementaciones `PostgresDbContext` (Npgsql) y `SqlServerDbContext`
> (Microsoft.Data.SqlClient). Enlaza con [[Visión y entorno|Vision y entorno]] seccion 3.1.

Esta nota es la vision de la capa; las fichas de cada pieza cuelgan de aqui.

## 1. Continuidad natural: el origen YA era dual

El punto mas importante para la migracion es que MotherData **no inventa** la
dualidad de motor en el destino: la trae de fabrica. Desde su origen convivieron
dos motores hermanos con la MISMA API publica:

- `AdmDatos` -- motor SQL Server (`System.Data.SqlClient` + OleDb). Ver
  [[AdmDatos - Motor SQL Server]].
- `AdmNpgsql` -- espejo del anterior contra PostgreSQL (`Npgsql`). Ver
  [[AdmNpgsql - Motor PostgreSQL]].

Ambos comparten `PreparaCadena`, `FindDataset`, `Execute`, el manejo de
`RSql`/`ErrorSql` y el mismo modulo `VariablesGlobales`. La decision de que motor
atiende una consulta la toma en runtime `LocalConexion.Tipo` (`SQL` vs `PG`). Es,
en esencia, un `IEcorexDbContext` embrionario resuelto por configuracion. El
destino solo hace explicito y tipado lo que el origen ya hacia por convencion:

| Concepto | Origen (MotherData) | Destino (.NET 10) |
|---|---|---|
| Abstraccion de motor | API comun `AdmDatos` / `AdmNpgsql` | `IEcorexDbContext` |
| Backend SQL Server | `AdmDatos` (SqlClient) | `SqlServerDbContext` (Microsoft.Data.SqlClient) |
| Backend PostgreSQL | `AdmNpgsql` (Npgsql) | `PostgresDbContext` (Npgsql) |
| Eleccion de motor | `LocalConexion.Tipo` en runtime | config `Database:Provider` en `appsettings.json` |
| Ejecucion de consulta | `FindDataset` / `Execute` (SQL string) | repositorios + `DbSet` LINQ / EF Core parametrizado |

Por eso ECOREX puede alojar cada tenant en el motor que su contrato exija --
PostgreSQL para el SaaS gestionado, SQL Server para enterprise con IIS/Windows
(como el `db3dev` legacy) -- sin bifurcar codigo de aplicacion. El patron es
identico al del vault CUBOT.nails ("Motor SQL DAL Dual").

## 2. Mapa de archivos del origen (referencia)

```
C:\Desarrollo\core\MotherData\
├── Datos\
│   ├── AdmDatos.vb            <-- Motor SQL Server (SqlClient + OleDb)
│   ├── AdmNpgsql.vb           <-- Motor PostgreSQL (Npgsql)
│   └── VariablesGlobales.vb   <-- Module: Conexion(), AliasCon(), mbase_empresa, carga_*
├── Crypto\
│   ├── AdmCrypto.vb           <-- TripleDES + MD5 (Encrypt/Decrypt simetrico)
│   └── CargaConfig.vb         <-- Lee Datos\Config.xml cifrado y devuelve XML plano
└── Web References\ServiceData\Reference.vb   <-- Cliente SOAP (legacy)
```

## 3. Pieza por pieza: que hacia -> como se reemplaza

| Pieza (origen) | Responsabilidad legacy | Reemplazo .NET 10 |
|---|---|---|
| [[AdmDatos - Motor SQL Server]] | CRUD SQL Server: `FindDataset`, `Execute`, `FindReader`, `SaveDataStrem`; resuelve alias `[dbx.X]` via `PreparaCadena` | `SqlServerDbContext` implementando `IEcorexDbContext`; repositorios EF Core |
| [[AdmNpgsql - Motor PostgreSQL]] | Espejo de `AdmDatos` sobre `Npgsql`, misma API | `PostgresDbContext` implementando `IEcorexDbContext` |
| [[AdmCrypto - Cifrado simétrico|AdmCrypto - Cifrado simetrico]] | TripleDES + MD5 (ECB, sin IV) para cifrar/descifrar configs | `IDataProtectionProvider` (DataProtection); AES-256-GCM para valores nuevos |
| [[CargaConfig - Decode Config.xml]] | Descifra `Config.xml` con una **key embebida en el codigo** | Configuracion .NET (`appsettings` + user-secrets) + secretos en **Key Vault** |
| [[VariablesGlobales - Conexiones y Empresa]] | Module global: `Conexion()`, `AliasCon()`, `mbase_empresa`, `carga_*` | `ITenantContext` inyectado por request + `ITenantConfigRepository` |

## 4. Configuracion que cargaba el origen (XML) -> destino

| Archivo | Cifrado | Contenido | Cargado por (origen) | Destino |
|---|---|---|---|---|
| `App\Datos\Config.xml` | **Si** (TripleDES+MD5) | Conexiones (BASE, CATALOGO, SERVIDOR, RUTADB, USUARIO, PW, TIPO, REPORTADOR) | `carga_base()` | tabla `tenant_database_config` via EF Core; secretos en Key Vault |
| `App\Datos\alias.xml` | No | Alias `[dbx.NOMBRE]` -> catalogo fisico | `carga_alias()` | resolucion de tenant/conexion (`IConnectionStringResolver`) |
| `App\Datos\Empresa.xml` | No | Tenant: NOMBRE, LOGO, BASE, LOGIN, LOGINNEW, MAIL, SUCURSAL | `carga_empresa()` / `carga_empresaii(empresa)` | `ITenantContext` + claims JWT (`tenant_id`) |

## 5. El alias `[dbx.GENE]` -> resolucion de tenant/conexion

En el origen, toda consulta usa el token `[dbx.NOMBRE]` en lugar del catalogo
fisico, y `PreparaCadena` lo sustituye consultando `AliasCon()`. Era la palanca de
portabilidad entre entornos sin tocar SQL. En el destino esa idea se **conserva
como valida** pero se mueve a infraestructura: el alias se convierte en la clave
de resolucion de tenant y conexion (un `IConnectionStringResolver` que, dado el
`tenant_id` del request, entrega el proveedor y el catalogo). El SQL de aplicacion
ya no lleva el token porque las consultas pasan por `DbSet` LINQ contra el
`IEcorexDbContext` correcto del tenant.

## 6. Errores heredados que el destino corrige

> [!danger] Los dos defectos que NO se portan
> 1. **SQL concatenado (inyeccion sistemica).** `AdmDatos`/`AdmNpgsql` reciben la
>    SQL ya armada por concatenacion de strings (~700+ puntos en el codebase).
>    Es el error #1 heredado. **Destino:** EF Core parametrizado; el token de
>    input de usuario nunca toca la cadena de consulta.
> 2. **Key de cifrado embebida en el fuente.** `CargaConfig.vb` linea 7 lleva la
>    key de descifrado como literal VB (base64 ofuscado con `####`). Cualquiera con
>    acceso al repo descifra todos los `Config.xml`. **Destino:** DataProtection +
>    Key Vault; ninguna key vive en codigo.

Otros riesgos del origen y su mitigacion en destino:

- **Estado global compartido.** `VariablesGlobales` mantiene `Conexion()`,
  `AliasCon()` y `mbase_empresa` a nivel de AppDomain -- en multi-tenant sobre un
  mismo proceso puede haber fuga de tenant activo. Destino: contexto por request
  (DI scoped) + `HasQueryFilter` por `TenantId` + RLS en BD.
- **Todo sincrono, sin async, sin paging.** `FindDataset` carga todo en memoria.
  Destino: EF Core async, `IQueryable` con paginacion.
- **Catch silencioso** en descifrado (ver `AdmCrypto`/`CargaConfig`): arranca con
  conexiones vacias. Destino: fail-fast con mensaje claro.

## 7. Ruta de migracion (resumen)

1. Definir `IEcorexDbContext` + `PostgresDbContext` + `SqlServerDbContext`;
   seleccion por `Database:Provider`.
2. Portar `AdmDatos.Execute`/`FindDataset` a repositorios EF Core parametrizados;
   auditar los puntos de SQL concatenado por modulo mas expuesto primero.
3. Sacar la key del fuente -> DataProtection + Key Vault (ver
   [[CargaConfig - Decode Config.xml]]).
4. Reemplazar el Module global por `ITenantContext` scoped y
   `ITenantConfigRepository` (config desde tabla, no XML).
5. Mantener `[dbx.X]` solo como convencion de resolucion en infraestructura.
6. OleDb a Excel/Access: aislar en modulo opcional; no todo tenant lo necesita.

Ver tambien [[00 - Mapa Funciones]] (utilidades cross-cutting -> servicios .NET
inyectables) y [[Visión y entorno|Vision y entorno]].
