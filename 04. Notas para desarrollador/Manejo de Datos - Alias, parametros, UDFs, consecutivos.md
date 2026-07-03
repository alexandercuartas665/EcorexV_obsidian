---
tipo: ficha-arquitectura
capa: Acceso a Datos + Configuración global
stack_destino: .NET 10 / EF Core 10 (DAL dual)
---

# Manejo de Datos - alias, parametros, UDFs, consecutivos

> **Ficha del manejo de datos LEGACY = reglas de negocio a preservar en el ETL.**
> Documenta como el ORIGEN (WebForms + `MotherData.AdmDatos`) resuelve alias,
> parametros, consecutivos y UDFs. Cada mecanismo tiene su equivalente en el
> DESTINO (.NET 10 / EF Core 10) resumido en la seccion 0; el resto del documento
> conserva el detalle del origen. Vision maestra en [[Visión y entorno]]
> (seccion 13 persistencia); esquema destino en [[Modelo Entidad-Relacion logico]].

## 0. Traduccion ORIGEN -> DESTINO de los mecanismos de datos

| Mecanismo ORIGEN | Que hace hoy | DESTINO .NET 10 |
|---|---|---|
| Alias `[dbx.X]` concatenado en el SQL | `AdmDatos.PreparaCadena` sustituye el alias por el catalogo fisico antes de ejecutar | `IConnectionStringResolver` **por tenant**: NO se concatena en el query; la conexion (o el schema/contexto) la elige el tenant resuelto. Multi-base del origen -> multi-context DAL dual |
| `GEN_PARAMETROS` (SUCURSAL/MODULO/CODIGO) | Cada pagina lee su config con SQL concatenado | Entidad `ModuleSetting` (`TenantId, ModuleCode, Code`) con `VALOR` -> `jsonb`; acceso via config tipada / `IOptions` por tenant |
| `SUCURSAL_PAR` (params sin modulo) | Personaliza login/tenant | `TenantSetting` (`jsonb`) |
| `PARAMXML` (XML grande por modulo) | Plantillas y definiciones en XML | `ModuleXmlConfig` -> **`jsonb`** (Postgres) / `nvarchar(max)` (SQL Server) |
| Consecutivos `Funciones.tipdoc.Consecutivo` (`T05`,`0D7`,`03D`,`REG1`...) | `MAX(CODIGO)+1` por (SUCURSAL, tipo) - race condition | **`ISequenceService` por tenant** respaldado por sequences Postgres / SQL Server (concurrente-safe) |
| UDFs (`fn_ConsultaCargo`, `Getdate3dev`) | Logica en la BD | Servicios de dominio en `Application`; fechas en UTC + TZ del tenant |
| SQL concatenado (`sql_t = "..." & txt.Text`) | Inyeccion sistemica (error #1) | EF Core parametrizado / `FromSqlInterpolated`; `Session("Empresa")` -> `HasQueryFilter` global |
| `USUARIO.FLAG_*` (10+ flags rol) | Roles fijos hardcodeados | `UserRole` con roles dinamicos (`IRolService`) |
| `SUCURSAL varchar` como tenant | Aislamiento por columna + disciplina | `TenantId uuid` + `HasQueryFilter` + RLS (multi-tenant real) |

> Los consecutivos, el alias y PARAMXML son los tres puntos que el prompt de
> migracion marca como criticos: **secuencia por tenant, conexion por tenant,
> configuracion en jsonb**. El resto de esta ficha es el detalle del origen que
> justifica cada decision.

---

## 1. Sistema de ALIAS `[dbx.X]`

Toda query SQL del sistema usa el patrón:

```vb
sql_t = "SELECT * FROM [dbx.GENE].dbo.USUARIO WHERE FLAG_3DEV = 1"
```

### Por qué

- **Portabilidad entre entornos**: el alias `[dbx.GENE]` se reemplaza por el catálogo físico en `MotherData.AdmDatos.PreparaCadena` antes de ejecutar.
- **Multi-base**: una misma página puede consultar varias bases (`[dbx.GENE]`, `[dbx.TURNOS]`, `[dbx.TRABAJO]`, etc.) sin saber el nombre físico.
- **Cambio de entorno = 1 archivo**: solo se actualiza `App\Datos\alias.xml`.

### Alias detectados en uso (en el proyecto Tareas)

| Alias | Catálogo físico (en `db3dev`) | Para qué |
|---|---|---|
| `[dbx.GENE]` | `db3dev` (todo) | **Default** — gestión general, usuarios, formularios, reglas, flujos, tareas |
| `[dbx.TRABAJO]` | (apunta a otra BD) | Catálogo de módulos web (`PRO_MODULOS`) |
| `[dbx.TURNOS]` | **NO existe en `db3dev`** | Solo referenciado en `cl_manejador_Reglas` (roto — ver [[Reglas - Motor y discrepancia]]) |

### Cómo se carga

`MotherData.AdmDatos.New()` ejecuta `PrepararBases()` que carga 2 XML al arrancar el AppDomain:

| Archivo | Contenido |
|---|---|
| `App\Datos\Config.xml` (cifrado TripleDES) | Lista de catálogos físicos (BASE, CATALOGO, SERVIDOR, USUARIO, PW, TIPO) |
| `App\Datos\alias.xml` (texto) | Tabla `nombre [dbx.X] → catalogo físico` |

→ los arrays globales `Conexion()` y `AliasCon()` quedan poblados a nivel AppDomain. Cada `tbrec.FindDataset` corre `PreparaCadena` que sustituye los `[dbx.X]`.

> Para detalle ver [[VariablesGlobales - Conexiones y Empresa]].

---

## 2. Configuración por SUCURSAL/MÓDULO/CÓDIGO

Hay **3 tablas paralelas** para configuraciones (con propósitos solapados):

### 2.1 `GEN_PARAMETROS` (7 cols) — la más usada

| Columna | Tipo | Notas |
|---|---|---|
| `REG` | int | PK |
| `CODIGO` | varchar | Clave |
| `NOMBRE` | varchar | Etiqueta human-readable |
| `VALOR` | varchar | El valor (string) |
| `MODULO` | varchar | Código de módulo (`000636`, etc.) |
| `SUCURSAL` | varchar | Tenant |
| `FLAG_UNICO` | tinyint | Si el código debe ser único por (SUCURSAL, MODULO) |

**Patrón típico de uso** (visto en `NEWFRONT_admtareas.aspx.vb:225`):

```vb
XMLTareaNueva = tbrec.FindReader(
   "SELECT VALOR FROM GEN_PARAMETROS WHERE SUCURSAL = '" & Session("Empresa") &
   "' AND MODULO = '000636' AND CODIGO = 'XML_NUEVA_TAREA'", BASE_SISTEMA)
```

> Cada módulo carga sus configuraciones aquí: plantillas XML (`XML_NUEVA_TAREA`, `XML_NOTIFICACION_TAREA`), tokens (`FORMULARIO_VIRTUAL`), flags de comportamiento, etc.

### 2.2 `SUCURSAL_PAR` (6 cols) — parámetros a nivel sucursal (sin módulo)

| Columna | Tipo |
|---|---|
| `REG` | int |
| `SUCURSAL` | varchar |
| `CODIGO` | varchar |
| `NOMBRE` | varchar |
| `VALOR` | ntext |
| `FLAG_EMAIL_MASIVO` | tinyint |

Usado por `MotherData.VariablesGlobales.carga_empresaii()` para personalizar el `base_login` por tenant. También para configuraciones generales del tenant que no son por módulo.

### 2.3 `PARAMXML` (3 cols)

Definiciones XML grandes por (`SUCURSAL`, `MODULO`). Módulo 000057 lo administra.

Tabla simple — la clave es que el `VALOR` (cargado en el módulo, no podido inspeccionar por limite de sqlcmd) contiene XMLs estructurales: plantillas de panel, definiciones de catálogos, configs de reportes, etc.

---

## 3. Consecutivos numéricos (`Funciones.tipdoc.Consecutivo`)

Cada tipo de documento del sistema tiene un código de 3 chars (ej. `T05` para tareas, `0D7` para procesos BPMN, `03D` para formularios, `REG1` para reglas, `A06` para...).

```vb
Dim mantipo As New Funciones.tipdoc
Dim nuevo_codigo As String = mantipo.Consecutivo(tipo_doc, "", Session("Empresa"))
```

### Por qué importa
- **Cada módulo declara su consecutivo** (vimos `manejo_consecutivo As String = "T05"`, `"0D7"`, `"03D"`, `"REG1"` en distintas páginas)
- El método mantiene un contador atómico por (`SUCURSAL`, `tipo_doc`) → genera el siguiente número
- Garantiza códigos únicos predecibles (formato corto, secuencial)

### Riesgo
- **No concurrente-safe** sin verificar implementación: típico `MAX(CODIGO)+1` con race condition. Para migración → reemplazar por sequences PostgreSQL o secuencias SQL Server.

---

## 4. UDFs del sistema (críticas)

El sistema tiene **~140 funciones SQL** (`INFORMATION_SCHEMA.ROUTINES`). Las relevantes para flujos/formularios/reglas:

### Funciones críticas (usadas por AdmWorkflow y formularios)

| UDF | Tipo de retorno | Uso |
|---|---|---|
| **`fn_ConsultaCargo(@cargo)`** | TABLE | **Expande un cargo a la lista de usuarios que lo tienen.** Cross-apply en `AdmWorkflow` para resolver `ENCARGADO` de cada nodo BPMN |
| **`fn_GetProceso(...)`** | TABLE | Resolución de proceso (uso pendiente verificar) |
| **`dbo.Getdate3dev()`** | datetime | Fecha local del servidor con timezone correcto (Colombia) — usado en `AdmWorkflow` en lugar de `GETDATE()` para evitar problemas de TZ |
| `fn_lastday`, `fn_next_Friday`, `fn_calcular_edad` | varias | Cálculos de fechas |
| `fn_quitar_acentuados`, `fn_JustifyText`, `Capitalize` | varchar/nvarchar | Manipulación strings |
| `fn_ObtenerDigitoVerificador` | varchar | Dígito verificación NIT colombiano |
| `fn_redondeo` | varchar | Redondeo personalizado |

### Funciones de reporte/datawarehouse (`fn_table_*`)

| UDF | Para qué |
|---|---|
| `fn_table_bodegas` | Tabla de bodegas |
| `fn_table_calculo_nomina` | Cálculos de nómina |
| `fn_table_calificarcurso` | Calificación cursos |
| `fn_table_costos_procedencia` (+II) | Costos por procedencia |
| `fn_table_cruceregistro` | Cruce de registros |
| `fn_table_itinerario` | Itinerarios (módulo CUBOT.travels) |
| `fn_table_liquida_cotiza` | Liquidación cotización |
| `fn_table_pronconsumo` (+II) | Pronóstico consumo |
| `fn_table_pry_cotizaciones_automaticas` | Cotizaciones automáticas |
| `fn_table_qienergy_facturacion` | Facturación QiEnergy |
| `fn_table_tipocontenedor` | Tipos contenedor |

→ son funciones de soporte para reportes y dashboards. Cliente-específicas en algunos casos.

### Stored Procedures

| SP | Tipo |
|---|---|
| `AddNewItem` | Mantenimiento de items |
| `android_sp_gestion` | API móvil Android |
| `fac_sp_calcula_imptos_factura` | Impuestos en facturas |
| `CONVEVRTIRPEDIDO` | Convertir pedido (?) |

---

## 5. Tabla de USUARIO (la central)

`USUARIO` (catálogo en `[dbx.GENE].dbo.USUARIO`) — campos clave:

| Columna | Tipo | Notas |
|---|---|---|
| `NOMBRE` | varchar | **Identificador principal** (no es ID numérico, es el username) |
| `ID_USUARIO` | varchar | Identificador alternativo |
| `CARGO` | varchar | Cargo organizacional (apunta a `fn_ConsultaCargo`) |
| `SUCURSAL` | varchar | Tenant principal |
| `SUCURSAL_ACT` | varchar | Tenant activo en sesión |
| `EMAIL` | varchar | |
| `TELS` | varchar | Teléfonos |
| `CONCESION` | varchar | Concesión asignada |
| `CONTRATO` | varchar | Contrato |
| `F_FIRMA` | image | **Firma escaneada en binario** |
| `DOC_FIRMA` | text | Documento de firma adicional |
| `NOMBR_PER` | varchar | Nombre personalizado |
| `GRUPO_T` | varchar | Grupo de trabajo |
| `FLAG_ADM`, `FLAG_ADTEC`, `FLAG_COMER`, `FLAG_FDEF`, `FLAG_FTEC`, `FLAG_JMON`, `FLAG_MON`, `FLAG_OPER`, `FLAG_TEC`, `FLAG_THUM` | tinyint | **10+ flags binarios** que activan funcionalidades específicas por rol |
| `MUL_SQL_UNICO` | int | (¿identificador único?) |
| `PERIODO` | tinyint | Periodo |
| `AÑO` | smallint | Año |
| `BGLOBAL`, `BLOCAL` | varchar | (¿bases globales/locales?) |

> **Observación crítica**: el sistema tiene **10+ FLAG_* en USUARIO** (FLAG_ADM, ADTEC, COMER, FDEF, FTEC, JMON, MON, OPER, TEC, THUM) que actúan como roles fijos. Esto es **un anti-pattern** para sistemas modernos — al migrar conviene unificar a `IRolService` con roles dinámicos.

> **`Funciones.PermissionsManager.ValidaPermiso(usuario, empresa, permiso, modulo)`** consulta probablemente `PERMISO_CARGO` + `USUARIO` + estos flags. Pendiente leer la implementación.

---

## 6. Tabla `SUCURSAL` y multi-tenant

Toda tabla del sistema tiene `SUCURSAL varchar(5..15)` que actúa como **tenant_id literal**. No hay tabla central `TENANT` con UUID — el tenant es un código corto (`'01'`, `'00125'`, `'00046'`, etc.).

`MasterFormsII` resuelve el nombre del tenant para el título:
```vb
SELECT NOMBRE FROM [dbx.GENE].dbo.SUCURSAL WHERE CODIGO = Session("Empresa")
```

Tenant default si no hay sucursal: **"ECOREX"** (línea 32 del Master).

### Tenants observados en producción

Vimos en `GEN_TOKEN`:
- `01` (default, soporte 3DEV)
- `00046`, `00125` (clientes reales)

Y los tenants con datos específicos:
- `METROCALI` (tablas `DOCU_METROCALI_*`)
- `SUPERGIROS` (tablas `DOCU_SUPERGIROS_*`)
- `SKY SYSTEM` (vimos en master title del usuario demo)

---

## 7. Persistencia de blobs (archivos, imágenes, firmas)

Tablas con binarios:
- `USUARIO.F_FIRMA image` — firma escaneada del usuario
- `GEN_ARCHIVOS` (14 cols) — archivos genéricos
- `DOC_IMAGENES`, `DOC_DIGITALES` — imágenes y docs digitalizados
- Probablemente `ENCUESTAS_DOCU_ARCHIVOS` para attachments de formularios

API en `MotherData.AdmDatos`:
```vb
tbrec.SaveDataStrem(byteArray, "F_FIRMA", "WHERE NOMBRE = '" & user & "'", BASE_SISTEMA)
```

---

## 8. Cómo se modela el organigrama

Tabla `DOC_ENTREVISTAS_ORG` (vista en `crtFlujoProcesos`):
```sql
SELECT PERMISO_CARGO.CARGO AS CODE,
       DOC_ENTREVISTAS_ORG.NOMBRE_CARGO AS NAME
FROM PERMISO_CARGO
LEFT JOIN DOC_ENTREVISTAS_ORG
  ON PERMISO_CARGO.SUCURSAL = DOC_ENTREVISTAS_ORG.SUCURSAL
 AND PERMISO_CARGO.CARGO = DOC_ENTREVISTAS_ORG.REG
WHERE PERMISO_CARGO.MODULO = 'PROCESOS_USUARIOS'
```

`DOC_ENTREVISTAS_ORG` mapea ID de cargo → nombre del cargo. **Es la tabla de organigrama** — sus cargos se asignan a nodos BPMN via `PERMISO_CARGO`.

---

## 9. Patrón general que se repite EN CADA `.aspx.vb`

```vb
' Imports
Imports MotherData

Public Class MiPage
    Inherits System.Web.UI.Page

    ' Variables de manejo de datos
    Dim tbrec As New MotherData.AdmDatos          ' DAL singleton local
    Dim mdata As DataSet                          ' buffer
    Dim sql_t As String = ""                      ' query buffer (concat)
    Dim BASE_SISTEMA As String = "SERVERI_MAR"    ' alias canonico
    Dim PAG_LOGIN As String = "index.aspx"        ' redirect si no hay sesion

    Sub New()
        Call carga_empresa()                       ' carga mbase_empresa global
        BASE_SISTEMA = mbase_empresa.base_pricipal ' usa el principal del tenant
        PAG_LOGIN = mbase_empresa.base_login
    End Sub

    Dim manejo_tabla As String = "MI_TABLA"       ' tabla principal del modulo
    Dim manejo_consecutivo As String = "X05"      ' tipo doc para Consecutivo()
    Dim manejo_dbx As String = "GENE"             ' alias por defecto

    Dim PermManager As New PermissionsManager     ' ACL
    Dim Optimize As New Optimizer                 ' UI helpers
    Dim MValidaError As New Funciones.ValidaError ' validacion

    Protected Sub Page_Load(...)
        topBar.Modulo = "000xxx"
        If PermManager.ValidaPermiso(Session("Usuario"), Session("Empresa"), "PERMISO_X", topBar.Modulo) Then ...
        ' Carga combos via Optimize.ComboFill(sql, default, combo, BASE_SISTEMA)
    End Sub
End Class
```

Para migrar: cada uno de estos patrones se reemplaza por su análogo de .NET Core (DI, EF Core, `[Authorize]`, etc.).

---

## 10. TODO

- [ ] Documentar `Funciones.PermissionsManager.ValidaPermiso` (cómo combina USUARIO.FLAG_*, PERMISO_CARGO, GEN_PARAMETROS)
- [ ] Catalogar todos los `tipo_doc` (consecutivos) usados en el sistema (grep en `.vb` de `manejo_consecutivo As String = "..."`)
- [ ] Detallar el contenido típico de `XML_NUEVA_TAREA` y `XML_NOTIFICACION_TAREA` (consulta a `GEN_PARAMETROS` cuando se levante limite sqlcmd)
- [ ] Documentar `cl_FormCreator` (motor del constructor de formularios)
- [ ] Documentar `cl_gestion_reglas` (motor real que ejecuta reglas — reemplazo de `cl_manejador_Reglas`)
- [ ] Identificar `crtCargaEncuestaII` (renderer de formularios)
