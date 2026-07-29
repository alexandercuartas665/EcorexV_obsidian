---
tipo: plan-olas
proyecto: Motor de Reportes y BI para ECOREX Tareas
doc: 03 - Plan de trabajo por olas (para sub-agente)
fecha: 2026-07-29
autor: documentado por agente IA a partir de decisiones del usuario
---

# 03 - Plan de trabajo por olas (para sub-agente)

> Backlog en olas con criterios de aceptacion, pensado para que un sub-agente lo ejecute en un
> **worktree** dedicado. Cada ola cierra con algo verificable. **Ola 0 es un gate**: si la licencia
> no califica, se decide el camino antes de construir.

## Reglas del worktree (contrato de trabajo)

> [!warning] Aislamiento de la sesion (obligatorio)
> Esta construccion corre EN PARALELO a la sesion principal, que tiene el dev en el puerto **5234**
> corriendo contra la BD de prod. Por eso:
> - **Worktree dedicado llamado `informes`.** Todo el trabajo vive en ese git worktree, no en el
>   arbol principal. No se mezcla con la rama/checkout de la sesion principal.
> - **Puertos que NO choquen** con el dev actual (5234) ni con los de `launch.json`
>   (5234 / 5236 / 5253 / 5256). Usar un bloque propio, p.ej. **5260** (y 5261+ si hace falta otro).
>   Agregar una config nueva en `launch.json` para el worktree en vez de reusar las existentes.
> - **NO hacer kill de procesos.** Nunca matar `Ecorex.SuperAdmin` (ni por nombre de imagen ni de
>   otra forma): mataria el dev de la sesion principal. Si hace falta parar algo del worktree, se
>   para SOLO por PID/puerto propio y verificando la ruta del proceso antes. El dev de la sesion
>   principal (5234) queda intacto durante toda la construccion.

- **No tocar produccion.** Deploy a prod solo con confirmacion explicita del usuario y con
  `./backup.sh` antes. Dev conectado a la BD de prod por tunel es SOLO lectura para validar.
- **Solo ASCII** en archivos nuevos. Repo publico: cero secretos/credenciales (ni claves de licencia
  versionadas).
- **Multi-tenant inviolable**: todo dato via el datasource tenant-safe; nada de connection string a
  la herramienta. Migraciones **duales** (PG + SQL Server).
- **UI 100% Blazor**; ECharts entra por interop con `.js` estatico (sin Node/npm build).
- Registrar avance en `PROGRESO.md` y un **ADR** con la decision de stack.

## Ola 0 - Gate de licencia y viabilidad (SIN codigo de producto)

**Objetivo**: despejar los 4 gates del doc 01 antes de construir.

- Confirmar elegibilidad **Community License** de Syncfusion (< 1M USD/anio, <= 5 devs) y **los
  terminos actuales de Bold Reports** (post separacion) + permiso de **redistribucion SaaS**.
- Prueba de humo: montar el **Report Designer web** de Bold en una pagina Blazor Server aislada y
  confirmar que embebe con la auth/cookies.
- Prueba de humo: el visor consumiendo un **JSON/Web data source** (datos dummy), sin cadena de
  conexion.

**Aceptacion**: documento corto (en el ADR) que responde SI/NO a cada gate. Si algun gate es NO ->
escalar al usuario con alternativa (Stimulsoft/DevExpress pagas, o construir editor a medida) ANTES
de seguir.

## Ola 1 - Catalogo semantico + datasource tenant-safe (el corazon)

**Objetivo**: la capa propia, independiente de la libreria.

- `IReportCatalog`: registra entidades nativas reportables (curadas por el dev) + deriva los
  contenedores (DataModel/DataContainer) automaticamente. Campos logicos, tipos, agregaciones/filtros
  admitidos.
- `IReportDataSource.QueryAsync(spec, ctx)`: traduce un `ReportQuerySpec` (referencia SOLO catalogo)
  a EF Core (nativas, con `HasQueryFilter` + parametrizado) y a consulta de contenedor. Devuelve
  `ReportDataSet` neutro.
- Endpoint web tenant-safe que expone `QueryAsync` como **JSON data source** (autenticado por cookie).

**Aceptacion**: test de integracion dual (PG + SQL Server) que pide el mismo spec como dos tenants y
demuestra **aislamiento** (cada uno ve solo lo suyo); un intento cross-tenant es imposible por
construccion. Cubre 1 entidad nativa + 1 contenedor.

## Ola 2 - Imprimibles: editor + visor + export (RDL)

**Objetivo**: el T1 con editor de usuario final.

- `ReportDefinition` (entidad nueva, ITenantScoped, soft-delete, concurrencia, migraciones duales).
- Pagina Blazor tras policy: **Report Designer** de Bold para crear/editar RDL; guardar por tenant.
- **Visor** que renderiza el RDL pidiendo datos al datasource tenant-safe; export **PDF/Excel**.
- 1 reporte real de ejemplo (p.ej. una orden/tarea) con filtro de tenant.

**Aceptacion**: un usuario crea un reporte en el editor, lo guarda, lo abre en el visor con datos
reales de SU tenant y exporta a PDF. Otro tenant no ve ese reporte ni esos datos.

## Ola 3 - Dashboards interactivos (ECharts)

**Objetivo**: el T2, milimetrico a la imagen de referencia.

- Componente `<EChart Option=.../>` (interop, `.js` estatico) + tiles KPI, tablas y feed en Blazor.
- Un dashboard de ejemplo (KPIs + area + dona + tabla) sobre `ReportDataSet` real, tras policy.
- Filtros/rango de fechas que re-consultan el datasource.

**Aceptacion**: el dashboard de ejemplo carga datos reales del tenant, con interactividad (tooltip,
filtro), y respeta el aislamiento. Comparacion visual contra la imagen de referencia.

## Ola 4 - Autoria por IA (T3)

**Objetivo**: "te doy una instruccion y creo el reporte".

- Esquema del **JSON-spec** + convertidor a RDL (imprimible) y a "option" ECharts (dashboard).
- Flujo: instruccion -> catalogo -> JSON-spec -> convertidor -> `ReportDefinition` guardado ->
  abrible en el editor.
- Guardrails: la IA solo referencia el catalogo; sin SQL ni connection string.

**Aceptacion**: con una instruccion en lenguaje natural, se genera un reporte valido (tabla+barras)
sobre una fuente real, guardado y editable. Un caso sobre nativa y otro sobre contenedor.

## Backlog (post v1)

- Diseniador de **dashboards** para usuarios (no solo IA).
- **Programacion/envio por correo** de reportes (reusar scheduler del Agente / `ImportProcess`).
- Permisos por reporte; parametros interactivos; cache de datasets.
- Limites por plan (cupos de filas/exportaciones).
- **Bold Report Server** como repositorio central si el volumen lo pide.

## Prompt de arranque para el sub-agente (resumen)

> Construye el Motor de Reportes y BI de ECOREX segun este capitulo. Trabaja en un **git worktree
> dedicado llamado `informes`**, en **puertos que NO choquen con 5234** (el dev de la sesion
> principal) ni con los de launch.json (5236/5253/5256) - usa un bloque propio (p.ej. 5260) y agrega
> tu propia config. **NO hagas kill de ningun proceso** (jamas mates `Ecorex.SuperAdmin`: tumbarias
> el dev de la sesion principal); si paras algo tuyo, hazlo solo por PID/puerto propio verificando la
> ruta. Empieza por la **Ola 0** (gate de licencia): NO escribas codigo de producto hasta responder
> los 4 gates del doc 01 y confirmarlos con el usuario. Luego Ola 1 (catalogo + datasource
> tenant-safe, con test de aislamiento dual) antes que cualquier UI. Respeta: multi-tenant via
> datasource (sin connection string), migraciones duales, solo ASCII, UI Blazor, ECharts por interop.
> No toques prod sin confirmacion + backup. Registra en PROGRESO.md + ADR.

Relacionado: [[00 - INDICE - Motor de Reportes y BI]],
[[01 - Vision, decision de stack y gates de licencia]],
[[02 - Arquitectura - capa de definicion, catalogo semantico y datasource tenant-safe]].
