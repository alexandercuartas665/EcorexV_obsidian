---
tipo: spec-modulo
proyecto: Motor de Reportes y BI para ECOREX Tareas
doc: 01 - Vision, decision de stack y gates de licencia
fecha: 2026-07-29
autor: documentado por agente IA a partir de decisiones del usuario
---

# 01 - Vision, decision de stack y gates de licencia

> Este documento explica el razonamiento completo: los tres trabajos, la evaluacion de TODAS las
> opciones consideradas (incluidas las que el usuario propuso), por que gana el stack elegido y los
> cuatro gates que la sesion del worktree valida antes de escribir una linea de codigo.

## 1. Actores y contexto

- **Usuario de negocio**: quiere ver dashboards y armar/ajustar reportes sin depender de un dev.
- **La IA (Claude)**: recibe una instruccion ("dame ventas del mes por vendedor en tabla + barras")
  y produce el reporte listo, aprovechando el motor.
- **El dev**: define el catalogo semantico (que es reportable) y las plantillas base.
- **El sistema**: multi-tenant real, DAL dual, repo publico (cero secretos), UI 100% Blazor.

## 2. Los tres trabajos escondidos en el pedido

No es una sola herramienta. Son tres necesidades con motores naturales distintos:

1. **T1 - Documentos imprimibles / pixel-perfect**: facturas, ordenes, PDF con membrete. Bandas,
   agrupaciones, totales, salto de pagina.
2. **T2 - Dashboards interactivos**: el tablero de KPIs de la imagen de referencia (tiles, area,
   dona, tablas, feed). Se ven en pantalla, con hover y filtros.
3. **T3 - Reportes ad-hoc que la IA crea por instruccion**, sobre tablas nativas Y contenedores.

El T3 es el diferenciador y condiciona todo: para que "te doy una instruccion y creo el reporte"
funcione, la definicion del reporte debe ser un **dato declarativo** que un motor renderiza, y debe
pasar por los guardarraeles de tenant. Ver doc 02.

## 3. El requisito que descarta media lista

Cualquier motor debe consumir **datos ya filtrados por tenant** (objetos/endpoints), **nunca** una
cadena de conexion directa. Un motor que consulta la BD por su cuenta rompe `HasQueryFilter`/RLS y
reabre la fuga cross-tenant (error heredado #1). Esto elimina de raiz las herramientas que quieren
"apuntar a tu base de datos" (Superset/Metabase directos), y obliga a que la suite que se compre use
su modo "business object / JSON data source".

## 4. Evaluacion de las opciones (las del usuario + las que faltaban)

| Herramienta | Rol | Encaje | Veredicto |
|-------------|-----|--------|-----------|
| **FastReport OSS** (.NET) | T1 | Nativo, bandas para facturas. PERO la edicion OSS recorta el editor visual y (segun version) el export PDF; formato `.frx` incomodo para IA. **Sin editor de usuario final.** | Descartado como motor. Solo si se quisiera imprimibles sin editor de usuario |
| **QuestPDF** (.NET) | T1 | MIT en codigo, API fluida en C#, PDF excelente. Ideal para autoria por IA/dev, pero **code-first, sin editor visual para usuarios**. Licencia Community gratis salvo > 1M USD/anio | **Comodin** para documentos hiper-custom. No resuelve el editor de usuario |
| **Apache ECharts** (JS) | T2 | El mejor para la imagen (area, dona, interactividad). Config **"option" JSON** -> perfecto para IA. Entra por interop con un `.js` estatico, sin Node/npm build. Apache-2.0 | **Elegido para dashboards a medida** |
| **Charts nativos Blazor** (Radzen/MudBlazor) | T2 | 100% .NET/SVG, cero JS. Menos potentes que ECharts | Fallback si el "solo .NET" fuera innegociable. El usuario ACEPTO ECharts, asi que no aplica |
| **Superset** (Python) | T2/BI | App aparte. Multi-tenant sobre la BD = riesgo cross-tenant. No conoce los filtros de tenant ni sirve a la autoria por IA. Apache-2.0 | **Solo referencia de features** |
| **Metabase** (Java) | T2/BI | App aparte. Igual riesgo multi-tenant. OSS es **AGPL** (delicado para SaaS) | **Solo referencia**. Embebido interno para analistas, quiza en el futuro, con datasource aislado |
| **Syncfusion Essential Studio (Blazor) + Bold Reports** | T1+T2 | **Editor visual web de reportes** (drag-drop, formato **RDL**) + visor + charts + export PDF/Excel, embebible en Blazor. Community License gratis si califican | **ELEGIDO** (editor de usuario final + imprimibles) |
| **Stimulsoft / DevExpress / Telerik** | T1+T2 | Suites con editor Blazor de reportes y dashboards muy buenas. **Sin tier gratis** | Alternativa paga si NO califican para Community |

## 5. Por que "usuarios finales tambien disenian" es la bisagra

Es el requisito mas caro. El combo OSS puro (FastReport OSS + QuestPDF + ECharts) **no** trae un
editor drag-drop de usuario final; habria que construirlo (meses). Por eso el mercado se parte en:

- **A) Comprar una suite** con editor Blazor listo (Syncfusion/Bold, Stimulsoft, DevExpress, Telerik).
- **B) Construir el editor a medida** sobre la capa de definicion propia.

El usuario eligio **A con Syncfusion/Bold bajo Community License si califican**, por ser el camino
mas rapido y barato con un editor real. Si la elegibilidad falla, el plan B/alternativa paga se
evalua en la Ola 0 (ver doc 03).

## 6. El stack elegido

- **Imprimibles + editor de usuario final -> Syncfusion Essential Studio (Blazor) + Bold Reports
  Report Designer** (formato RDL).
- **Dashboards a medida -> Apache ECharts** por interop (option JSON, autoria por IA).
- **Export -> el del visor Syncfusion/Bold** (PDF/Excel). QuestPDF solo comodin.
- **Datos -> SIEMPRE via datasource tenant-safe** (JSON/Web apuntando a la API propia).
- **Autoria por IA -> generar/parchear RDL** (o un JSON-spec propio -> convertidor -> RDL): mismo
  artefacto que el editor visual.

## 7. Sinergia con la autoria por IA

Como la definicion del reporte vive como dato (RDL / JSON), **la IA y el usuario editan el mismo
artefacto**. La IA parte de una plantilla, enlaza campos del catalogo semantico y produce el RDL;
el usuario lo abre en el editor y lo ajusta. No hay dos mundos separados.

## 8. Los 4 gates (se validan en la Ola 0, antes de construir)

1. **Elegibilidad Community License**: < 1M USD ingresos brutos/anio Y <= 5 devs (Syncfusion). Y
   **confirmar como licencia Bold Reports hoy** (se separo de Syncfusion; su editor/server puede
   tener terminos propios). **Riesgo #1: si falla, cambia el camino.**
2. **Editor web embebe en Blazor Server** con la auth/cookies del sistema (no como app aparte).
3. **Datasource tenant-safe** (JSON/Web): probar que el visor consume datos ya filtrados por tenant,
   sin cadena de conexion, contra 1 entidad nativa + 1 contenedor.
4. **Redistribucion SaaS**: que la licencia permita exponer el editor a los tenants (no solo uso
   interno del equipo).

## 9. Riesgos y notas

- **Licencia** (gate #1) es el riesgo principal. No escribir codigo de producto hasta despejarlo.
- **Acoplamiento a la suite**: mitigado porque el valor propio (capa de definicion + catalogo +
  datasource) es independiente de la libreria; si un dia se cambia de suite, esa capa se conserva.
- **AGPL Metabase / apps BI aparte**: quedan fuera del producto; a lo sumo herramienta interna.
- **ECharts es JS**: aceptado como interop con `.js` estatico (sin Node/npm build), coherente con la
  regla "UI del producto en Blazor" (que prohibe el build con Node, no cargar una lib JS por interop).

Sigue en [[02 - Arquitectura - capa de definicion, catalogo semantico y datasource tenant-safe]].
