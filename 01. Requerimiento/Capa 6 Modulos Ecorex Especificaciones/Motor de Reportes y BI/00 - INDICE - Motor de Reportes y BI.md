---
tipo: indice-proyecto
proyecto: Motor de Reportes y BI para ECOREX Tareas
modulo_web: reportes (Sistema . Reportes y tableros BI)
estado: DECIDIDO (stack + arquitectura), NO construido. Se construye en una sesion nueva en worktree. Gate #1 antes de escribir codigo - validar elegibilidad de la Community License (Syncfusion / Bold Reports).
fecha: 2026-07-29
autor: documentado por agente IA a partir de decisiones del usuario
---

> [!important] Que es y por que existe
> ECOREX necesita un **motor de reportes y BI dentro de la aplicacion** que cubra tres
> trabajos distintos: (1) **documentos imprimibles** (facturas, ordenes, PDF con membrete),
> (2) **dashboards interactivos** (estilo tablero de KPIs: tiles, area, dona, tablas, feed) y
> (3) **reportes ad-hoc que la IA crea por instruccion**, todo sobre las **tablas nativas del
> sistema Y los Contenedores de datos (DataModels)**. La meta del usuario: "te doy
> instrucciones, aprovechas el motor y creas los reportes con facilidad" - y ademas que
> **usuarios finales de negocio tambien los disenien** (drag-drop), no solo la IA o los devs.

> [!success] Decisiones tomadas con el usuario (2026-07-29)
> - **Autoria**: usuarios finales TAMBIEN disenian reportes (drag-drop), no solo IA/devs.
>   -> obliga a un **editor visual de reportes embebido**, la pieza mas cara del proyecto.
> - **Graficos**: se ACEPTA **Apache ECharts via JS interop** (un solo `.js` estatico, sin
>   Node/npm build) para los dashboards a medida.
> - **Camino elegido**: **Syncfusion Essential Studio (Blazor) + Bold Reports** para el editor
>   visual + visor + export, bajo **Community License** si la compania califica. La sesion del
>   worktree valida la elegibilidad ANTES de construir.

> [!warning] Como se trabaja (aislamiento de la sesion)
> - Se construye en un **git worktree dedicado llamado `informes`** (no en el arbol de la sesion
>   principal).
> - Corre en **puertos que NO choquen** con el dev actual (**5234**) ni con los de `launch.json`
>   (5236/5253/5256): bloque propio, p.ej. **5260**.
> - **NO se hace kill de procesos.** Jamas matar `Ecorex.SuperAdmin` (tumbaria el dev de la sesion
>   principal); parar solo lo propio, por PID/puerto verificado. Detalle en el doc 03.

# Motor de Reportes y BI - ECOREX Tareas

> Especifica un **motor de reportes y BI 100% dentro del producto (.NET 10 / Blazor Server)**
> que permite crear reportes imprimibles y dashboards sobre datos nativos y de contenedores,
> con un **editor visual para usuarios finales** y **autoria asistida por IA** sobre el mismo
> artefacto de definicion. Documentacion auto-contenida para que un sub-agente lo construya en
> un worktree sin mas contexto.

## 1. En una frase

El sistema ya sabe **guardar datos** (tablas nativas multi-tenant + Contenedores); lo que falta
es una capa que sepa **sacarlos formateados**: un **editor/visor de reportes** (imprimibles) y
un **motor de dashboards** (interactivos) que leen una **definicion declarativa** - la misma que
un usuario arma arrastrando campos y que la IA genera por instruccion - y que **siempre** consulta
los datos a traves de los guardarraeles de tenant del sistema, nunca por su cuenta.

## 2. Que YA existe (punto de partida real)

- **Tablas nativas multi-tenant**: toda entidad de negocio es `ITenantScoped` con
  `HasQueryFilter` global + RLS en BD (regla inviolable #1). El motor de reportes NO puede
  saltarse esto.
- **Contenedores de datos (`DataModel` / `DataContainer`)**: modelos ER definidos por el usuario,
  alimentados por Excel / REST / BD / agente on-prem (ver [[00 - INDICE - Agente Conector On-Prem]]).
  Son una fuente reportable de primera clase.
- **DAL dual** (PostgreSQL + SQL Server) tras `IEcorexDbContext`.
- **Menu dinamico + policies**: el modulo de reportes entra como un item de menu tras policy.
- **Autoria por IA ya probada** en otros modulos (la IA edita artefactos declarativos del sistema).

Lo NUEVO que agrega este proyecto: el **editor/visor de reportes**, el **motor de dashboards**,
la **capa de definicion declarativa**, el **catalogo semantico** (que es reportable) y el
**contrato de datasource tenant-safe**.

## 3. Los tres trabajos (no es una sola herramienta)

| # | Trabajo | Ejemplo | Salida |
|---|---------|---------|--------|
| T1 | Documentos imprimibles / pixel-perfect | Factura, orden, reporte PDF con membrete | PDF / impresion (bandas, totales, salto de pagina) |
| T2 | Dashboards interactivos | El tablero de KPIs de la imagen de referencia | Pantalla (charts con hover/filtros) |
| T3 | Reportes ad-hoc autoria IA | "dame ventas del mes por vendedor en tabla + barras" | T1 o T2 segun se pida |

## 4. Decisiones ya tomadas (con el usuario, 2026-07-29)

| #  | Decision | Elegido |
|----|----------|---------|
| D1 | Quien disenia los reportes | **Usuarios finales (drag-drop) + IA + devs** -> editor visual embebido REQUERIDO |
| D2 | Suite de reportes + editor visual | **Syncfusion Essential Studio (Blazor) + Bold Reports Report Designer** (formato **RDL**) |
| D3 | Licencia | **Community License si califican** (< 1M USD ingresos brutos/anio Y <= 5 devs). Si no, evaluar Stimulsoft/DevExpress o construir a medida |
| D4 | Dashboards interactivos a medida | **Apache ECharts** via JS interop (config "option" JSON, ideal para autoria por IA) |
| D5 | Acceso a datos | **SIEMPRE via objetos/endpoints ya filtrados por tenant** (JSON/Web data source apuntando a la API propia). NUNCA cadena de conexion directa a la BD |
| D6 | Autoria por IA | La IA **genera/parchea el RDL** (o un JSON-spec propio que un convertidor pasa a RDL): el MISMO artefacto que edita el usuario |
| D7 | Export | El del visor Syncfusion/Bold (PDF/Excel). **QuestPDF** queda solo como comodin para documentos hiper-custom si hiciera falta |
| D8 | Fuentes reportables | Tablas **nativas** (via catalogo semantico) + **Contenedores** (DataModel/DataContainer) con el mismo patron |
| D9 | Descartados como MOTOR | **Superset/Metabase** (apps aparte, riesgo cross-tenant, Metabase OSS es AGPL) -> solo referencia de features. **FastReport OSS** (sin editor de usuario final) |

## 5. El requisito decisivo (que ata todo)

Cualquier herramienta - comprada o propia - debe consumir **datos ya filtrados por tenant**
(business objects / view models / endpoints JSON tenant-safe), **nunca** una cadena de conexion.
Si se le da acceso directo a la BD, se pierde `HasQueryFilter`/RLS y se reabre la fuga cross-tenant
(el error heredado #1). Este es el contrato central del proyecto, detallado en el doc 02.

Corolario: **el valor esta en la capa de definicion + catalogo semantico + query-builder seguro
por tenant.** Las librerias que pintan (Syncfusion/Bold, ECharts, QuestPDF) son commodities
intercambiables detras de esa capa. Ahi va el esfuerzo.

## 6. Los documentos de este capitulo

| Doc | Contenido |
|-----|-----------|
| [[01 - Vision, decision de stack y gates de licencia]] | Los 3 trabajos, evaluacion de TODAS las opciones (FastReport OSS, QuestPDF, ECharts, Superset/Metabase, suites comerciales), por que gana Syncfusion/Bold + ECharts, y los 4 gates que la sesion valida ANTES de construir |
| [[02 - Arquitectura - capa de definicion, catalogo semantico y datasource tenant-safe]] | El corazon: modelo de definicion declarativa, catalogo semantico (nativas + contenedores), contrato de datasource tenant-safe, RDL como artefacto compartido con la IA, y como enchufan el editor/visor Bold + los dashboards ECharts |
| [[03 - Plan de trabajo por olas (para sub-agente)]] | Backlog en olas con criterios de aceptacion; PoC minima; contrato de trabajo para el sub-agente del worktree; que NO tocar (prod) |

## 7. Opinion de arquitectura (resumen)

El patron es estandar: un **motor de reportes embebido + una capa semantica propia**. Comprar la
suite (Syncfusion/Bold) resuelve lo caro y comoditizado - el **editor visual drag-drop**, el visor
y el export - que construir desde cero cuesta meses. Pero la suite NUNCA toca la BD: se le entrega
datos ya saneados por tenant. Los dashboards muy a medida (como la imagen de referencia) se hacen
con **ECharts**, que se configura por JSON y por eso encaja con la autoria por IA. Y como la
definicion vive como dato (RDL / JSON), **la IA y el usuario editan el mismo artefacto** - esa es
la sinergia que pidio el usuario.

## 8. Alcance v1 vs backlog

- **v1**: catalogo semantico de N entidades nativas + los contenedores; contrato de datasource
  tenant-safe; editor/visor Bold Reports embebido tras policy; 1 reporte RDL imprimible real (con
  filtro de tenant) + 1 dashboard ECharts; export PDF/Excel; autoria por IA generando/parcheando
  RDL sobre una plantilla.
- **Backlog**: diseniador de dashboards para usuarios (no solo IA); programacion/envio de reportes
  por correo (reusar el scheduler del Agente/`ImportProcess`); permisos por reporte; parametros
  interactivos; cache de datasets; limites por plan (cupos de filas/exportaciones); Report Server de
  Bold como repositorio central si el volumen lo pide.

## 9. Los 4 gates (riesgos que se validan PRIMERO)

1. **Elegibilidad Community License**: confirmar < 1M USD ingresos/anio y <= 5 devs (Syncfusion) Y
   **como licencia Bold Reports hoy** (se separo de Syncfusion; su editor/server puede tener terminos
   propios). **Riesgo #1.**
2. **Editor web embebe en Blazor Server** con la auth/cookies del sistema.
3. **Datasource tenant-safe** (JSON/Web) probado contra 1 entidad nativa + 1 contenedor.
4. **Redistribucion SaaS**: que la licencia permita exponer el editor a los tenants.

Relacionado: [[00 - INDICE - Agente Conector On-Prem]] (fuente de datos de contenedores),
[[00 - INDICE - Extraccion de Datos]] (otra fuente de contenedores). En el repo: memoria de proyecto
`motor-reportes-decision` y (a crear) `ADR-00XX - Motor de reportes y BI`.
