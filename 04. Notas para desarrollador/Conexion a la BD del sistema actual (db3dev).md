---
tipo: nota-referencia
capa: transversal
proposito: Referencia de la BD de desarrollo del sistema actual (db3dev) - fuente del ETL y del descubrimiento de schema/modulos/usuarios
estado: activo
aviso_seguridad: las credenciales NO estan en este repo (publico); ver seccion 2
---

# Conexion a la BD del sistema actual (db3dev)

> Nota de referencia para **descubrir el sistema legacy en vivo** (schema, modulos,
> usuarios, datos) y como **fuente del ETL** hacia el destino .NET 10. La BD es una
> **copia de produccion en ambiente de desarrollo**: se puede consultar y probar
> cambios sin afectar produccion. Aun asi, aplican los guardrails de la seccion 5.

## 1. Que es

- **Motor**: SQL Server 2022.
- **Base**: `db3dev` — copia de produccion, ambiente de desarrollo (safe).
- **Volumen (verificado)**: ~1004 tablas de usuario, 105 sucursales (tenants), 532 modulos.
- Es la MISMA plataforma `GestionMovil` que el sistema en produccion; sirve para
  entender el origen antes de migrar (ver [[Visión y entorno]] seccion 18).

## 2. Credenciales - NO en este repo (importante)

> Este repositorio es **publico**. La cadena de conexion (host, puerto, usuario y
> **password**) **no se versiona aqui**. Vive fuera del repo:
>
> - En un archivo local `**.env**` del entorno de desarrollo (no versionado), y
> - En la **memoria privada del agente** (`.claude/.../memory/reference_bd_desarrollo.md`),
>   que persiste entre sesiones y NO se sube a GitHub.
>
> Para usar la conexion, cargar la variable de entorno local, por ejemplo:
> `DB3DEV_CONNECTION="Data Source=<host,puerto>;Initial Catalog=db3dev;User Id=<user>;Password=<secret>;"`.
> Si trabajas con un agente de IA en este proyecto, el agente ya tiene la cadena en
> su memoria privada; **no la pegues en ninguna nota del vault**.

Regla dura: si alguna vez aparece una cadena de conexion real (con password) dentro
de una nota del vault, **redactarla antes de commitear** (reemplazar el password por
`<secret>` o `REDACTED`).

## 3. Tenant principal a migrar

El tenant que corresponde al sistema que vamos a migrar es el de **codigo de
sucursal = `01`**, que en `SUCURSAL` es **`01 = "BITCODE"`**. Toda consulta de
descubrimiento orientada a la migracion debe filtrar por ese tenant (o compararlo
contra otros de las 105 sucursales para separar lo global de lo por-tenant).

```sql
SELECT CODIGO, NOMBRE FROM SUCURSAL WHERE CODIGO = '01';   -- 01 | BITCODE
```

## 4. Para que sirve (uso previsto)

Consultar (read-only por defecto) para:

- **Descubrir tablas**: nombres, columnas, FKs del sistema actual — insumo del
  [[Modelo Entidad-Relacion logico]] y del mapeo del ETL en [[HOJA DE RUTA DESARROLLO]] seccion 9.
- **Descubrir modulos**: `PRO_MODULOS` (CODIGO, NOMBRE) — 532 modulos; cuadra con el
  [[INVENTARIO GENERAL]] (000038 Crear actividad, 000042 Proyectos, 000291 Flujos, etc.).
- **Usuarios y permisos**: tablas `USUARIO`, `SUCURSAL_GRUPOS*`, `PRO_MODULOS_PERMISO`.
- **Relaciones y volumenes**: para estimar esfuerzo y detectar tablas pesadas.
- **Muestras de datos**: para entender formatos y casos borde antes de reconstruir.

Tablas ancla ya identificadas: `SUCURSAL` (tenants), `PRO_MODULOS` (modulos),
`USUARIO`, `DOC_PROCESOS*`/`TAR_WORKFLOW_*` (flujos), `ENCUESTAS_MOV*`/`FORX_DATA`
(formularios), `CONTROL_REGLAS*` (reglas).

## 5. Guardrails

**Politica del proyecto (decision del usuario, 2026-07-03): SOLO LECTURA (`SELECT`).**
No escribir en db3dev — ni sandbox — salvo que el usuario lo autorice explicitamente
para un caso puntual.

Ademas, aunque sea DEV:

- **Solo `SELECT`**. Nada de `INSERT`/`UPDATE`/`DELETE`/`DROP`/`TRUNCATE`.
- Si hay que modificar, respaldar el row afectado primero y hacerlo en tablas de sandbox.
- **Evitar `SELECT *`** en tablas grandes; usar `TOP`/`COUNT` para sondear volumen.
- **Nunca** respaldar esta base a lugares publicos (contiene datos reales copiados de produccion).
- **Nunca** commitear la cadena de conexion al repo.

## 6. Como conectar

- **Dentro del proyecto legacy** (`C:\Desarrollo\core`): patron `MotherData.AdmDatos` / `tbrec`
  (ver [[00 - Visión MotherData]]).
- **Externo** (descubrimiento): `sqlcmd`, DBeaver, Azure Data Studio o SSMS, con la
  cadena cargada desde el `.env` local.

```bash
# Ejemplo sqlcmd (la cadena/credenciales vienen del entorno, no del repo)
sqlcmd -S "$DB3DEV_HOST" -U "$DB3DEV_USER" -P "$DB3DEV_PASS" -d db3dev -Q "SELECT COUNT(*) FROM PRO_MODULOS;"
```

## 7. Cuando el agente/dev deberia proponer usar esta conexion

Sugerir consultar `db3dev` cuando:
- Se necesita el schema real (nombres/columnas/FKs) y no hay doc escrita.
- Se va a definir el mapeo ETL de una familia de tablas.
- Se necesita la lista real de modulos activos y su codificacion.
- Se reconstruye logica de negocio a partir de datos reales.
- Se estima esfuerzo/volumen antes de una fase.

Preguntas utiles antes de queries no triviales:
1. Cual es la columna que identifica el tenant/sucursal en la tabla objetivo?
2. Hay un campo `activo`/`estado` estandar a filtrar?
3. Hay tablas pesadas (millones de filas) a evitar?
4. Cuales son las tablas puente entre modulos?

## 8. Enlaces

- [[HOJA DE RUTA DESARROLLO]] — el ETL (seccion 9) consume esta BD
- [[Modelo Entidad-Relacion logico]] — schema derivado de esta BD
- [[INVENTARIO GENERAL]] — modulos que coinciden con `PRO_MODULOS`
- [[00 - Visión MotherData]] — patron de acceso a datos del legacy
- [[Seguridad y Autenticacion multi-tenant]] — por que las credenciales no van al repo
