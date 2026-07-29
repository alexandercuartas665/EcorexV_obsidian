---
tipo: spec-modulo
proyecto: Motor de Reportes y BI para ECOREX Tareas
doc: 02 - Arquitectura (capa de definicion, catalogo semantico, datasource tenant-safe)
fecha: 2026-07-29
autor: documentado por agente IA a partir de decisiones del usuario
---

# 02 - Arquitectura: capa de definicion, catalogo semantico y datasource tenant-safe

> El corazon del proyecto NO es la libreria que pinta, sino tres piezas propias: la **capa de
> definicion declarativa**, el **catalogo semantico** (que es reportable) y el **contrato de
> datasource tenant-safe**. Este documento las especifica y muestra como enchufan el editor/visor
> Bold y los dashboards ECharts. Las firmas son ilustrativas (el sub-agente ajusta al codigo real).

## 1. Vista de conjunto

```
   Usuario (drag-drop)  ---\                       /---  Editor/Visor Bold (RDL)  --> PDF/Excel/print
                            >--  Definicion  ------<
   IA (instruccion)     ---/    (RDL / JSON-spec)   \---  Dashboards ECharts (option JSON) --> pantalla
                                     |
                                     v
                        Catalogo semantico (que es reportable)
                                     |
                                     v
                   Datasource tenant-safe (JSON/Web)  <-- UNICO camino a los datos
                                     |
                    +----------------+-----------------+
                    v                                  v
        Entidades nativas (EF Core,          Contenedores (DataModel /
        HasQueryFilter + RLS)                DataContainer, filas EAV)
```

Regla de oro: **el editor/visor y los charts JAMAS tocan la BD**. Solo consumen el datasource
tenant-safe, que devuelve filas ya filtradas por tenant.

## 2. Capa de definicion declarativa

Un reporte es un **dato**, no codigo. Dos formatos conviven:

- **RDL** (el nativo de Bold Reports): lo que el editor visual produce y consume. Es el artefacto
  de los imprimibles y de los reportes de usuario final.
- **JSON-spec propio** (opcional, para T3): una definicion mas simple que la IA genera con comodidad
  y que un **convertidor** transforma a RDL (imprimible) o a "option" de ECharts (dashboard). Sirve
  para que la IA no tenga que escribir RDL crudo cuando la plantilla lo permite.

Esbozo del JSON-spec (ilustrativo):

```json
{
  "titulo": "Ventas del mes por vendedor",
  "fuente": { "tipo": "nativa", "entidad": "TaskItem" },
  "campos": ["Vendedor", "Total"],
  "filtros": [{ "campo": "Fecha", "op": "entre", "valor": ["2026-07-01", "2026-07-31"] }],
  "agregados": [{ "campo": "Total", "func": "suma", "por": "Vendedor" }],
  "orden": [{ "campo": "Total", "dir": "desc" }],
  "presentacion": { "tipo": "tabla+barras", "salida": "dashboard" }
}
```

El sub-agente decide el esquema final; lo importante es: **sin SQL crudo** (lo arma el query-builder),
y **campos referenciados por nombre logico del catalogo** (no columnas fisicas).

## 3. Catalogo semantico (que es reportable)

Para que un usuario (o la IA) elija campos, hace falta un **catalogo** que exponga las fuentes
reportables con nombres de negocio, tipos y relaciones - sin filtrar detalles fisicos:

- **Entidades nativas**: un registro curado por el dev (no "toda la BD"). Cada entidad reportable
  declara sus campos logicos, tipos, y que agregaciones/filtros admite. Ej.: `TaskItem` expone
  `Titulo`, `Estado`, `Vendedor`, `Total`, `Fecha`.
- **Contenedores**: se derivan solos del `DataModel`/`DataContainer` (el usuario ya definio tablas y
  columnas). El catalogo los publica como fuentes reportables automaticamente.

El catalogo es tambien **el limite de seguridad**: si algo no esta en el catalogo, no es reportable.
Esto evita que la IA o el usuario alcancen tablas/campos que no deben.

## 4. Datasource tenant-safe (el contrato central)

Un unico componente traduce una definicion + el tenant del usuario en **filas ya filtradas**:

```csharp
public interface IReportDataSource
{
    Task<ReportDataSet> QueryAsync(ReportQuerySpec spec, ReportContext ctx, CancellationToken ct);
    // ctx lleva TenantId, usuario y policies. spec referencia SOLO campos del catalogo.
}
```

- Traduce el `spec` a **EF Core** para entidades nativas: el `HasQueryFilter` global inyecta el
  `TenantId` (imposible pedir cross-tenant por construccion) y la consulta es **parametrizada**
  (regla #3, cero concatenacion).
- Para contenedores, consulta `DataContainerRow`/`Cell` acotado al tenant y al DataModel.
- Devuelve un `ReportDataSet` neutro (columnas + filas) que alimenta por igual al visor Bold (como
  **JSON/Web data source** o business object) y a ECharts (como series del "option").
- El editor/visor Bold se configura para pedir sus datos a **este** endpoint, nunca a una cadena de
  conexion. Es el punto que cierra el gate #3.

## 5. Enchufe de Bold Reports (imprimibles + editor de usuario)

- **Editor**: el Report Designer web de Bold, embebido en una pagina Blazor tras policy. Guarda RDL
  por tenant (tabla nueva `ReportDefinition` con `TenantId`, nombre, RDL, tipo, auditoria).
- **Visor**: renderiza el RDL pidiendo datos al `IReportDataSource`. Export PDF/Excel nativo.
- **Datos del RDL**: se enlazan a un **JSON/Web data source** que apunta a la API tenant-safe (con la
  cookie/auth del usuario), pasando el nombre de la fuente + parametros. Nunca connection string.

## 6. Enchufe de ECharts (dashboards a medida)

- Un componente Blazor `<EChart Option="..."/>` que recibe el "option" JSON y lo pinta por interop
  (`.js` estatico de ECharts + un pequenio `echart-interop.js`).
- El "option" se arma desde el `ReportDataSet` (server) para T2, o lo genera la IA directamente para
  T3. Interactividad (tooltip, zoom, filtros) del lado de ECharts.
- Los tiles KPI, tablas y feed de la imagen de referencia son componentes Blazor normales sobre el
  mismo `ReportDataSet`.

## 7. Autoria por IA (T3), paso a paso

1. El usuario da la instruccion en lenguaje natural.
2. La IA consulta el **catalogo semantico** (que fuentes/campos existen).
3. La IA emite el **JSON-spec** (o RDL directo desde plantilla).
4. El convertidor produce RDL (imprimible) u "option" ECharts (dashboard).
5. Se guarda como `ReportDefinition` del tenant; el usuario puede abrirlo en el editor y ajustarlo.

La IA **no** escribe SQL ni cadenas de conexion: solo referencia el catalogo. La seguridad la
garantizan el catalogo + el datasource tenant-safe.

## 8. Persistencia y multi-tenant

- `ReportDefinition` (nuevo): `Id`, `TenantId` (ITenantScoped), `Nombre`, `Tipo` (imprimible/dashboard),
  `Rdl`/`SpecJson`, `CatalogoRef`, auditoria, soft-delete, concurrencia optimista (reglas #5, #8).
- Todo bajo `HasQueryFilter` + RLS. Un reporte de un tenant es invisible para otro por construccion.
- Migraciones **duales** (PG + SQL Server), como todo el sistema.

## 9. Que NO hace este motor

- No expone la BD cruda a la herramienta (sin connection string).
- No es un BI self-service tipo Superset/Metabase para explorar "toda la base" (eso viola el
  catalogo y el multi-tenant). El catalogo acota deliberadamente lo reportable.
- No corre como app aparte: vive dentro de `Ecorex.SuperAdmin` (Blazor), tras policy.

Sigue en [[03 - Plan de trabajo por olas (para sub-agente)]].
