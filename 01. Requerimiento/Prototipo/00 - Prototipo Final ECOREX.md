---
tipo: prototipo-final
proposito: Prototipo visual definitivo de ECOREX Sistema de Tareas (.NET 10) - concepto de arranque del proyecto
origen: Claude Design (ECOREX.dc.html)
estado: prototipo aprobado - base para desarrollo
---

# Prototipo Final - ECOREX Sistema de Tareas

> **Este es el prototipo con el que arranca el proyecto.** Reemplaza todas las
> capturas y bocetos anteriores. Define el aspecto y la interaccion definitivos
> del nuevo ECOREX sobre .NET 10. Los conceptos individuales que hicimos antes
> para Claude Design (carpeta `conceptos-claude-design/`) se conservan solo como
> referencia de pantallas puntuales; el estandar visual manda es este.

## Como abrirlo

- **Ejecutable (recomendado)**: abrir `ECOREX - Prototipo Final.html` en el
  navegador. Es una SPA React autocontenida — doble clic y funciona sin servidor.
- **Fuente Claude Design**: `ECOREX.dc.html` + `support.js` — para re-importar o
  editar el diseno en Claude Design.

## Sistema de diseno

| Elemento | Definicion |
|---|---|
| Layout | Doble panel: rail de iconos (extrema izquierda) + sidebar contextual `SKY SYSTEM` |
| Marca | `SKY SYSTEM` / `Plan Empresa - ECOREX`, avatar de inicial |
| Estilo | Claro, moderno, estilo Linear/Height. Tarjetas KPI con tile de icono en color suave |
| Acentos | Morado de marca en activo; botones primarios negros; badges de estado en color |
| Tipografia | Sans del sistema, titulos grandes con jerarquia marcada |
| Modo oscuro | Toggle disponible (icono luna, esquina inferior del sidebar) |
| Navegacion | Breadcrumbs arriba (`Equipos / SKY SYSTEM / <seccion>`), boton `Compartir`, campana de notificaciones, buscador global con atajo Cmd+K |
| Multi-tenant | Cada modulo muestra su codigo (`MODULO 000XXX`) y nota "variables por tenant" |

El sidebar organiza la navegacion en dos bloques: **PRINCIPAL** (Inicio, Anuncios,
Gestor de tareas, Configuracion) y **MODULOS** (procesos del tenant con su codigo:
Crear actividad `000038`, Proyectos `000042`, Administrar actividades `000636`,
Programar actividad `000889`, Comercial `000477`, etc.). Abajo, la ficha del
usuario (Guadalupe Lopez - Administradora).

## Recorrido por pantalla

### 1. Inicio / Resumen

Dashboard de bienvenida contextual: saludo por hora, resumen de pendientes
(`8 tareas y 3 alertas`), boton `Nueva actividad`, y 4 tarjetas KPI (Tareas
activas, Proyectos en curso, Flujos ejecutandose, Alertas urgentes) con deltas.
Debajo, "Mis tareas de hoy" y "Alertas del sistema".

![[01-inicio-resumen.png]]

> Variante de sidebar mas simple (bloque PRINCIPAL / CATALOGOS) capturada para
> comparacion — el estandar es la de arriba (rail + MODULOS):
>
> ![[01b-inicio-variante.png]]

### 2. Gestor de tareas / Tableros

Lista de tableros del tenant. KPIs (Tableros, Tareas totales, Completadas),
barra de **Filtros** (Usuario, Etiqueta, Categoria, Subcategoria, Fecha) y
tarjetas de tablero (ej. `PRY-0042 Comercial - Requerimiento Infraestructura`)
con su pipeline de columnas.

![[02-tableros-gestor-tareas.png]]

Estados progresivos de la misma vista (KPIs, fila de filtros, tarjetas):

![[02a-tableros-kpis.png]]
![[02b-tableros-filtros.png]]
![[02c-tableros-cards.png]]

### 3. Proyecto - detalle + Tablero Kanban

Vista de un proyecto/requerimiento con cabecera (Estado, Asignados con avatares,
Fecha limite con dias restantes, Etiquetas), pestanas `Vista Tablero / Timeline /
Calendario`, boton `+ Tarea`, y el Kanban con columnas `Por hacer / En progreso /
En revision / Completado` y contador por columna.

![[03-proyecto-detalle-tablero.png]]

### 4. Dependencias (Organigrama) - MODULO 000850

Estructura organizacional del tenant: KPIs (Dependencias, Usuarios, Areas),
boton `Nueva dependencia`, arbol de **Organigrama**
(`ECOREX S.A.S > Gerencia General > Comercial / Operaciones / Tecnologia`) con
conteo de usuarios por nodo, y panel de detalle de la dependencia seleccionada
(tipo Area, estado Activa, codigo `DEP-100`, descripcion).

![[04-dependencias-organigrama.png]]
![[04b-dependencias-detalle.png]]

### 5. Modulos web (Registro) - MODULO 000109

Registro de todos los modulos del sistema, con permisos gestionables y variables
por tenant. KPIs (Modulos registrados, Activos, Permisos definidos), boton
`Registrar modulo`, buscador y lista de modulos con codigo y estado
(`Crear actividad 000038 - Operaciones - Activo`).

![[05-modulos-web-registro.png]]

### 6. Anuncios

Tablon de anuncios del workspace (badge de no leidos). En el prototipo la vista
esta esbozada (contenido pendiente de completar).

![[06-anuncios.png]]
![[06b-anuncios-lista.png]]

## Estado del prototipo (que falta)

El prototipo cubre el aspecto y la navegacion definitivos, pero algunas pantallas
estan esbozadas o pendientes de completar. Lista de ajustes conocidos:

- [ ] **Anuncios**: vista con contenido real (hoy es placeholder)
- [ ] **Flujos del proceso**: falta el editor BPMN embebido (bpmn-js) — ver [[00 - Visión Flujos]]
- [ ] **Formularios**: falta el constructor EAV — ver [[00 - Visión Formularios]]
- [ ] **Entidad**: una de las vistas capturaba error React (`#62`) — revisar
- [ ] Unificar las dos variantes de sidebar en la definitiva (rail + MODULOS)
- [ ] Conectar KPIs y listas a datos reales (hoy son demo de SKY SYSTEM)

## Relacion con la documentacion

- El prototipo materializa el sistema DESTINO descrito por capas en
  [[00 - INDICE|el vault]] — especialmente la vision de
  [[Gestion de Empresas - Admin multi-tenant|multi-tenant real]] (los codigos
  `000XXX` y "variables por tenant" del prototipo).
- Los 5 conceptos en `conceptos-claude-design/` alimentaron pantallas puntuales
  (tar_conceptos, gen_reglas, adm_empresas, web_scraping, ContactLoader); este
  prototipo integra el estandar visual comun.
- Las capturas viven en `capturas/` y son la referencia visual oficial del
  proyecto.

---

*Prototipo aprobado. Actualizar las capturas cuando el prototipo evolucione y
marcar en la lista de "que falta" lo que se vaya completando.*
