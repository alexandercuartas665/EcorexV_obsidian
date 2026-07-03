---
tipo: ficha-clase
clase: AdmDatos
archivo: C:\Desarrollo\core\MotherData\Datos\AdmDatos.vb
dependencias: System.Data.SqlClient, System.Data.OleDb, System.Xml.Linq
target_origen: .NET Framework 4.8.1
reemplazo_destino: SqlServerDbContext (implementacion SQL Server de IEcorexDbContext)
estado: adaptado a vision destino
---

# AdmDatos -- Motor SQL Server -> `SqlServerDbContext`

Clase central del DAL legacy. **TODO `.aspx.vb` del sistema crea una instancia
local `Dim tbrec As New MotherData.AdmDatos`** y opera contra ella: es el "padre
de todos los repositorios" en la arquitectura clasica de la plataforma. En el
destino .NET 10 su rol lo asume `SqlServerDbContext`, la implementacion SQL Server
de la abstraccion dual `IEcorexDbContext` (ver [[00 - Visión MotherData|00 - Vision MotherData]] y
[[Visión y entorno|Vision y entorno]] seccion 3.1). Su hermano PostgreSQL es
[[AdmNpgsql - Motor PostgreSQL]] -> `PostgresDbContext`; juntos son el ancestro
directo del DAL dual.

## 1. Que hacia (API publica del origen)

| Miembro | Tipo | Descripcion |
|---|---|---|
| `RSql` | Property String | Ultima SQL ejecutada (tras `PreparaCadena`). Para log/debug. |
| `ErrorSql` | Property String | Error de la ultima operacion. `""` si OK. Patron: `If tbrec.ErrorSql = "" Then...`. |
| `TimeExecute` | Property String | Tiempo de ejecucion (instrumentacion opcional). |
| `PreparaCadena(sql, base)` | Function String | Reemplaza alias `[dbx.X]` por catalogo real via `AliasCon()`. Setea `RSql`. |
| `AlizaQuery(sql)` | Function String | Limpia `vbCr/vbCrLf/vbLf/vbTab` y colapsa espacios. |
| `LimpiaEspacios(s)` | Function String | Quita dobles espacios iterativamente. |
| `CountCharacter(value, ch)` | Shared Function Integer | Contador. |
| `FindDataset(sql, base, tabla, tipo=0)` | Function DataSet | Carga principal: `DataSet` con una `DataTable` `tabla`. SqlClient u OleDb segun `LocalConexion.Tipo`. |
| `FindReader(sql, base)` | Function String | Primer campo de la primera fila como string. Atajo para `COUNT(*)` y lookups. |
| `Execute(sql, base)` | Function Integer | INSERT/UPDATE/DELETE/DDL. Filas afectadas. Setea `ErrorSql`. |
| `SaveDataStrem(byteArr, campo, where, base)` | Sub | Persiste `Byte()` en campo `varbinary` (imagenes, PDFs). |
| `New()` | Sub | Constructor -> `PrepararBases()` -> `carga_base() + carga_alias()` desde `VariablesGlobales`. |

## 2. Flujo interno de FindDataset / Execute (origen)

1. Validar `sql <> ""` y `base <> ""`.
2. `PreparaCadena(sql, base)` -- sustituye `[dbx.X]` por catalogo de `AliasCon`.
3. Lookup en `Conexion()` (array global) la `LocalConexion` que matchea `Base`.
4. Construir cadena de conexion segun `Tipo` (`SQL`, `OLE`, ...).
5. Abrir conexion, ejecutar (`SqlDataAdapter.Fill` o `SqlCommand.ExecuteNonQuery`).
6. Cerrar conexion (`Finally`).
7. En `Execute`, capturar excepcion -> `Me.ErrorSql = ex.Message`.

## 3. Estructura `LocalConexion` (en `VariablesGlobales.vb`)

```vb
Public Structure LocalConexion
    Public Base       As String   ' alias logico (ej. "SERVERI_MAR")
    Public Catalogo   As String   ' catalogo fisico (ej. "db3dev")
    Public Servidor   As String   ' host
    Public RutaDb     As String   ' Para Access/OleDb
    Public Usuario    As String
    Public Pw         As String
    Public Tipo       As String   ' "SQL" | "OLE" | ...
    Public Objeto     As String
    Public Puerto     As String
    ' Reporting
    Public servidor_rep As String
    Public servidor_dom As String
    Public servidor_usu As String
    Public servidor_cla As String
End Structure
```

## 4. Patron "tbrec" legacy (SQL concatenado = error a corregir)

```vb
Dim sql_t As String = ""
sql_t = "SELECT NOMBRE + ' (' + CODIGO + ')' AS NAME, CODIGO AS CODE " &
        " FROM [dbx." & manejo_dbx & "].dbo." & manejo_tabla &
        " WHERE SUCURSAL ='" & Session("Empresa") & "' ORDER BY NOMBRE"

Dim count = tbrec.FindReader("SELECT COUNT(*) FROM [dbx.GENE].dbo.X WHERE ID='" & id & "'", BASE_SISTEMA)

sql_t = "UPDATE [dbx." & manejo_dbx & "].dbo." & manejo_tabla &
        " SET NOMBRE='" & txt.Text & "'" &
        " WHERE CODIGO='" & p_factura & "' AND SUCURSAL='" & Session("Empresa") & "'"
tbrec.Execute(sql_t, BASE_SISTEMA)
If tbrec.ErrorSql = "" Then ... ' exito
```

Cada `& valor &` concatenado con input de usuario es un punto de inyeccion SQL.
Ademas el `WHERE SUCURSAL = ...` es el aislamiento multi-tenant "a mano" del
origen: depende de la disciplina del desarrollador, no del sistema.

## 5. Como se reemplaza en .NET 10 (`SqlServerDbContext` + repositorios)

- **`FindDataset` / `FindReader` / `Execute`** -> metodos de repositorio sobre
  `DbSet<T>` con LINQ o SQL parametrizado (`FromSqlInterpolated`). Nada de
  concatenacion.
- **Alias `[dbx.X]`** -> ya no viaja en la consulta; se resuelve en
  infraestructura por `tenant_id` (`IConnectionStringResolver`).
- **`WHERE SUCURSAL = ...`** -> desaparece del SQL de aplicacion: lo impone
  `HasQueryFilter(x => x.TenantId == _tenant.CurrentTenantId)` + RLS en BD.
- **`ErrorSql` (property de error)** -> excepciones tipadas + `Result`/manejo por
  middleware; nada de estado de error compartido.
- **`SaveDataStrem` (blob)** -> columna binaria via EF Core o Object Storage
  segun tamano.
- **Sincrono** -> API async de EF Core (`ToListAsync`, `SaveChangesAsync`).
- **`SELECT COUNT(*)` string** -> `await set.CountAsync(predicate)`.

Boceto destino:

```csharp
// Antes: tbrec.FindReader("SELECT COUNT(*) ... WHERE ID='" & id & "'", base)
var exists = await _ctx.Items.AnyAsync(x => x.Id == id);   // filtro de tenant + RLS implicitos
```

## 6. Lo bueno (que se conserva como concepto)

- API uniforme y predecible (3 verbos). En destino se conserva la simplicidad via
  repositorios de intencion clara.
- Alias `[dbx.X]` como palanca de portabilidad -> resolucion de conexion por tenant.
- Persistencia de blobs -> soportada por EF Core / Object Storage.

## 7. Lo malo (que NO se porta)

- **Cero parametrizacion**: inyeccion SQL sistemica (error #1 heredado).
- Estado global compartido (`Conexion()`, `AliasCon()`).
- No async; `FindDataset` carga todo en memoria (sin paging/streaming).
- Mezcla SqlClient + OleDb por `Tipo` en runtime.

Ver tambien [[AdmNpgsql - Motor PostgreSQL]] (motor hermano -> `PostgresDbContext`)
y [[00 - Visión MotherData|00 - Vision MotherData]].
