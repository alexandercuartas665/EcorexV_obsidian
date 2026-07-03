---
tipo: ficha-clase
clase: AdmNpgsql
archivo: C:\Desarrollo\core\MotherData\Datos\AdmNpgsql.vb
dependencias: Npgsql
target_origen: .NET Framework 4.8.1
reemplazo_destino: PostgresDbContext (implementacion PostgreSQL de IEcorexDbContext)
estado: adaptado a vision destino
---

# AdmNpgsql -- Motor PostgreSQL -> `PostgresDbContext`

**Espejo conceptual de `AdmDatos` pero contra PostgreSQL via `Npgsql`.** Misma API
(`PreparaCadena`, `FindDataset`, `Execute`, manejo de `RSql`/`ErrorSql`) y el mismo
modulo `VariablesGlobales`. Es la prueba de que MotherData fue **dual desde el
origen**: junto a [[AdmDatos - Motor SQL Server]] forma el ancestro directo del DAL
dual del destino. En .NET 10 su rol lo asume `PostgresDbContext`, la
implementacion PostgreSQL de la abstraccion `IEcorexDbContext` (ver
[[00 - Visión MotherData|00 - Vision MotherData]] y [[Visión y entorno|Vision y entorno]] seccion 3.1).

## 1. Diferencias vs [[AdmDatos - Motor SQL Server]] (origen)

| Aspecto | `AdmDatos` (SQL Server) | `AdmNpgsql` (PostgreSQL) |
|---|---|---|
| Cliente | `System.Data.SqlClient` (+ OleDb) | `Npgsql` |
| Sintaxis SQL | T-SQL | PG SQL (snake_case, JSONB, etc.) |
| Alias `[dbx.X]` | OK (se reemplaza por catalogo) | OK (mismo `PreparaCadena`) |
| Logs | `Temp\MyLogData*.txt` | `Temp\MyPostgresLogData*.txt` |
| Constructor | Llama `PrepararBases()` | **No** lo llama -- asume que `AdmDatos` ya lo hizo |
| `SaveDataStrem` | Si | No visible aun (revisar archivo completo) |

## 2. Que motor decide cual atiende la consulta

En el origen la capa que decide es **`LocalConexion.Tipo`**: si el `Config.xml` de
una sucursal trae `TIPO=PG`, `AdmNpgsql` toma el relevo; si trae `TIPO=SQL`,
responde `AdmDatos`. Ese switch en runtime es exactamente lo que en el destino se
vuelve explicito y tipado: la config `Database:Provider` de `appsettings.json`
selecciona `PostgresDbContext` o `SqlServerDbContext` sin bifurcar codigo de
aplicacion.

| Concepto | Origen | Destino (.NET 10) |
|---|---|---|
| Seleccion de motor | `LocalConexion.Tipo` (`PG` / `SQL`) en runtime | `Database:Provider` en config |
| Motor PostgreSQL | `AdmNpgsql` (Npgsql) | `PostgresDbContext` (Npgsql, EF Core 10) |
| API de consulta | `FindDataset` / `Execute` (SQL string) | repositorios + `DbSet` LINQ parametrizado |
| Campos dinamicos | texto/`FORX_DATA` | columnas `jsonb` nativas de Postgres |

## 3. Uso en el origen (idem AdmDatos cuando aplica)

```vb
Dim tbrec As New MotherData.AdmNpgsql
Dim ds = tbrec.FindDataset("SELECT * FROM tareas WHERE sucursal=@p1", "SERVERI_MAR_PG", "TAREAS")
'  ^ la parametrizacion sigue siendo manual por concatenacion hasta que se migre
```

- La clase existe pero **el proyecto Tareas no la usa** en las paginas
  exploradas: todo va contra SQL Server (`SERVERI_MAR`). Su razon de ser es la
  migracion progresiva a PostgreSQL en el ecosistema Bitcode (Visal, DokTrino).

## 4. Como se reemplaza en .NET 10 (`PostgresDbContext`)

- Misma abstraccion `IEcorexDbContext` que el motor SQL Server: los repositorios
  no saben contra que motor corren.
- **PostgreSQL es el motor por defecto del SaaS gestionado**; SQL Server queda
  para enterprise con infraestructura Windows (el `db3dev` legacy).
- Aprovecha nativas de Postgres: `jsonb` para el EAV de formularios (`FORX_DATA`
  migra a `jsonb`), `xmin` para concurrencia optimista, `CREATE POLICY` para RLS.
- Parametrizacion nativa de Npgsql via EF Core -- se elimina la concatenacion.
- CI corre los tests en **ambos** motores por cada PR (ver [[Visión y entorno|Vision y entorno]]
  seccion 15).

## 5. Ventaja arquitectonica que se hereda

El sistema **fue pensado para soportar ambos motores desde el inicio**, lo que
habilita una migracion por modulo (mover una BD a la vez). El unico obstaculo del
origen -- `Dim tbrec As New MotherData.AdmDatos` hard-codeado en cada `.aspx.vb` --
desaparece en el destino porque el `IEcorexDbContext` correcto se **inyecta** por
DI segun el proveedor configurado; ninguna pagina instancia el motor a mano.

## 6. TODO (residual, ya cubierto por el destino)

1. Inyectar `IEcorexDbContext` resuelto por proveedor en lugar de hard-codear
   `AdmDatos`/`AdmNpgsql`. -> resuelto por DI + `Database:Provider`.
2. Unificar la API en una interfaz comun. -> `IEcorexDbContext` + repositorios.
3. Pasar a Npgsql parametrizado. -> EF Core parametrizado.

Ver tambien [[AdmDatos - Motor SQL Server]] (motor hermano -> `SqlServerDbContext`)
y [[00 - Visión MotherData|00 - Vision MotherData]].
