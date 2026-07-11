---
tipo: spec-ux
proyecto: Modulo de Tareas
proposito: Fidelidad al prototipo del modal "Nueva actividad" (wizard 4 pasos), del menu dinamico "Mis Procesos" y de los tableros. La fuente de verdad visual es ECOREX.dc.html.
---

# 02 - UX y fidelidad (modal, menu, tableros)

> Regla del proyecto: **fidelidad milimetrica** al prototipo `01. Requerimiento/Prototipo/
> ECOREX.dc.html`. Extraer de ahi colores hex, tipografia, espaciados, radios y sombras (vars
> `--surface`, `--line`, `--ink*`, `--brand`, `--t-blue/violet/amber/green`, `--sh-sm/lg`...).
> Referencias de linea en el prototipo se dan como guia; ante duda gana el HTML.

## 1. Modal "Nueva actividad" - wizard de 4 pasos (proto ~4280-4438)

Modal grande (min(1100px,97vw) x 90vh), header "ECOREX . Actividades / Nueva actividad", cuerpo
en 2 columnas: izquierda el wizard, derecha la **columna de resumen** (aside 280px). Barra de
pasos arriba: **1 Informacion . 2 Contacto . 3 Formulario . 4 Documentos** (chips clicables).

### Paso 1 - Informacion (proto ~4303-4341)
Tres tarjetas:
- **Clasificacion y asignacion** ("Define a que proceso pertenece la actividad, quien la atiende
  y para cuando"): grid 3 col -> **Empresa / Area** | **Proceso / Tipo** | **Actividad**; grid 2
  col -> **Encargado** | **Fecha de entrega** (input date).
  - Empresa/Area <- OrgUnit (D2). Proceso/Tipo <- categoria del concepto. Actividad <-
    subcategoria (concepto). Encargado <- cargos/encargado (cascada, ver doc 01 s.4).
  - Cascada: al elegir Tipo(categoria) se cargan las Actividades(subcategorias) de esa categoria;
    al elegir Actividad se puede prefijar el encargado sugerido (cargo del concepto).
- **Descripcion de la actividad**: input **Titulo** + textarea **Descripcion / detalle**.
- Grid 2 col: **Prioridad** (chips Baja/Media/Alta -> `TaskPriority`) | **Etiquetas** (chips con
  Enter, quitar con x).

### Paso 2 - Contacto (proto ~4344-4360)
- Informacion del contacto/solicitante: Nombre completo, Identificacion, Email, Telefono, Numero
  de actividad (auto, ej. A06/T#####), **Proyecto** (select), **Hito del proyecto** (select).
- **Copiar a correos** (chips de email con Enter) -> `TaskItem.CcEmails`.

### Paso 3 - Formulario (proto ~4362-4380) - CONDICIONAL
- Si la subcategoria tiene formulario (`IniciaModulo`/`FormDefinitionId`): renderiza el
  **formulario dinamico** (campos del proceso) con `DynamicFormRenderer` (modo Fill). El proto
  muestra ejemplo (Presupuesto, Tipo de equipo, Observaciones) - en real, los campos vienen de la
  `FormDefinition`.
- Si NO tiene: estado vacio "Esta actividad no tiene formulario asociado".

### Paso 4 - Documentos (proto ~4381-4394)
- **Archivos adjuntos**: select "Concepto del archivo" (Cotizacion/Soporte/Evidencia) + dropzone
  (arrastrar/click; PDF, imagenes, hojas de calculo).

### Columna de resumen (aside, proto ~4410-4436)
Estado "Borrador / Sin guardar"; NUM. DE ACTIVIDAD (Sin asignar hasta guardar); campos vivos:
Empresa/Area, Proceso, Actividad, Encargado (avatar), Fecha de entrega, Prioridad (chip),
Etiquetas; bloque "ADJUNTOS Y SEGUIMIENTO": contadores Archivos y Copia a.

### Footer (proto ~4396-4408)
`<- Atras` | `Limpiar` | (spacer) | **Guardar y crear otra** | **Siguiente ->** (o **Guardar
actividad** en el ultimo paso). El boton "Guardar y crear otra" replica el patron ya construido
en el Contenedor de datos ("+ Guardar y nueva"): persiste y deja el formulario limpio para otra.

## 2. Menu "Mis Procesos" dinamico (proto ~4711 groupDefs; imagen adjunta)

En el prototipo, el grupo **Mis Procesos** (icono checklist, tono violeta `--t-violet`) contiene
subgrupos = **categorias** del concepto (Direccion Comercial, Gestion Calidad y Documentacion,
Gestion Comercial, Gestion de Seguimiento, Gestion Marketing, Software Desarrollo), cada uno
expandible a sus **subcategorias** (actividades tipo proceso). Al hacer clic en una subcategoria:
- Carga el **tablero especifico** de esa actividad (el `TaskBoardId` del concepto) y abre el
  **modal** de actividad ya con categoria/subcategoria fijadas (no vuelve a preguntar).

Implementacion (ver doc 01 D3): un grupo de menu marcado como "muestra actividades tipo proceso"
que `NavMenu` expande dinamicamente desde `IActividadCatalogoService` (categorias -> subcategorias
con `WorkflowDefinitionId != null`). El editor de menu (`ConfiguracionMenu.razor`) gana la opcion
para marcar ese grupo (pedido explicito del usuario: "arreglar el editor de menu para indicar que
grupo muestra las actividades tipo proceso").

Fidelidad del rail: sangria por nivel, punto/vinieta por subcategoria, chevron de expandir, tono
violeta del grupo; igual que la imagen `Pasted image 20260710171009.png` del vault.

## 3. Tableros y listas (proto: tablero kanban; Capa 2 destino)

- `/actividades` (000636, "Administrar actividades"): index de tableros + detalle. Vista
  principal del operativo. Tabs (del legacy/Capa 2): No Asignados / Asignados / Todos / Kanban,
  con conteos vivos por SignalR. Preseleccion por `?cat=&sub=` (usada por los links de Mis
  Procesos).
- Tablero kanban: columnas por estado; drag&drop entre columnas (ADR-0020, `TaskItem.BoardId/
  ColumnId/BoardSortOrder`). En la lista/modal de "crear actividad" desde un tablero solo se
  ofrecen las actividades (subcategorias) **sin proceso** (camino B).
- Detalle de la tarea: `TaskDetailModal.razor` ya embebe la tarjeta "Flujo" (reclamar/atender/
  aprobar) y el formulario del paso activo. Se conserva.

## 4. Consideraciones de fidelidad (checklist para el que construya)

- Extraer tokens exactos del `ECOREX.dc.html` (no inventar colores). Ver `<style>` y las vars CSS.
- Wizard: mismos radios (16-20px tarjetas, 11px inputs), sombras (`--sh-sm/lg`), tipografia
  (pesos 600-800, tamanos 11.5-16px), grid de 3 y 2 columnas del paso 1.
- Chips de prioridad/etiquetas/emails con el mismo estilo (soft brand, x para quitar).
- Columna de resumen con los campos vivos que reflejan el paso 1.
- Menu: tono violeta del grupo Mis Procesos, sangria, vinietas, chevrons.
- Reusar el patron "Guardar y crear otra" ya hecho en Contenedor de datos.
