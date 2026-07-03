---
tipo: historias-usuario
protagonista: Guadalupe (Coordinadora Operativa)
modulos_cubiertos: [Crear Actividad 000038, Administrar Tareas 000636, Mis Tareas 000620, Proyectos 000042]
---

# Historias del Operativo - Guadalupe

> **Guadalupe** coordina un equipo de 8 operadores en SKY SYSTEM (su tenant
> aislado). Es la persona que asigna trabajo, monitorea avance y cierra casos. NO
> configura procesos ni edita formularios - simplemente **usa** lo que Andres dejo
> configurado. En el sistema destino .NET 10 trabaja dentro del layout del
> [[00 - Prototipo Final ECOREX]]: sidebar SKY SYSTEM, Inicio con KPIs y tableros
> Kanban. La vision maestra esta en [[Visión y entorno]].

---

## Escenario 1 - Recibir un pedido y crear la tarea

### User Story
**Como** coordinadora operativa,
**quiero** capturar rapidamente un pedido de cliente que me llego por mail,
**para que** el operador correcto empiece a trabajarlo en menos de 10 minutos.

### Precondiciones
- Sesion iniciada como Guadalupe en el tenant SKY SYSTEM (JWT con `tenant_id`; todo
  lo que ve queda filtrado a su empresa por el filtro global de EF Core)
- El pedido incluye: cliente, tipo de requerimiento, urgencia, fecha limite

### Pasos concretos
1. Guadalupe entra a **Inicio / Resumen** (dashboard "Buenos dias, Guadalupe") y pulsa el boton **"Nueva actividad"** (o abre **Crear actividad** `000038` en el bloque MODULOS del sidebar).
2. Se abre el panel de creacion de actividad del tenant.
3. Selecciona **Categoria = "Cotizacion Comercial"** (la actividad `TIPO_TAR`, servida por el Task Service del tenant).
4. El sistema le muestra las subcategorias validas de esa categoria.
5. Elige **Subcategoria = "Requerimiento infraestructura"**.
6. Escribe titulo: `"Cotizar 20 laptops Lenovo ThinkPad para AlpesTechnology"`.
7. Asigna a **Andres (Lider Tecnico)** en el combo Encargado.
8. Marca prioridad **ALTA** y fecha entrega **manana**.
9. Click en **Crear tarea**.

### Que pasa por detras
El comando de creacion (CQRS, en una sola transaccion) llega al **WorkflowEngine**
(motor portado de `AdmWorkflow`) que:
- Resuelve el flujo BPMN asociado a esa (categoria, subcategoria) para el tenant actual
- Crea la instancia de flujo (`workflow_instance`) y precrea **todos los nodos** como pasos pendientes
- Marca el paso 1 (StartEvent) como el nodo activo
- Resuelve los usuarios validos por nodo via las policies del PermissionsManager (permisos por cargo)

Toda entidad lleva `tenant_id` (invariante `ITenantScoped`), asi que la tarea nace
aislada dentro de SKY SYSTEM. Guadalupe no ve nada de esto - solo recibe un toast
**"Tarea T0042 creada"**.

### Criterios de aceptacion
- La tarea aparece en el tablero, columna **"Por hacer"**, sin responsable si no eligio encargado
- Si asigno a Andres, aparece en su tablero ya con responsable
- Andres recibe una notificacion en tiempo real (SignalR) sin recargar
- La tarea se persiste con su `tenant_id` (nunca visible desde otro tenant)
- La instancia de flujo BPMN queda creada con todos sus pasos precreados

---

## Escenario 2 - Distribuir la carga del dia via Kanban

### User Story
**Como** coordinadora,
**quiero** ver de un vistazo el estado del equipo y arrastrar tareas entre estados,
**para que** ningun operador este sobrecargado o inactivo.

### Pasos concretos
1. Guadalupe abre **Gestor de tareas / Tableros** en el sidebar.
2. Entra al tablero del proyecto (ej. `PRY-0042 Comercial - Requerimiento Infraestructura`).
3. Ve 4 columnas: `Por hacer | En progreso | En revision | Completado`, con contador por columna.
4. Cada tarjeta muestra: titulo, responsable (avatar), prioridad, fecha entrega, etiquetas.
5. Arrastra la tarjeta **"Cotizar 20 laptops"** de "Por hacer" a "En progreso" y la asigna a Andres.
6. Repite con otras 5 tareas.
7. Usa la barra de **Filtros** (Usuario, Etiqueta, Categoria, Subcategoria, Fecha) para depurar la vista y exportar el reporte del dia.

### Criterios de aceptacion
- El drag&drop persiste el nuevo estado de inmediato (comando CQRS, concurrencia optimista)
- Los contadores de cada columna se actualizan y, via SignalR, tambien en las pantallas de los demas del equipo
- Al recargar la pagina, las tareas siguen en su nuevo lugar
- Si el usuario destino no tiene la policy para recibir asignaciones, la tarea rebota con un toast de error
- Nunca aparece una tarea de otro tenant en el tablero (filtro global + RLS en BD)

### Notas para el prototipo
En el prototipo final el tablero Kanban con columnas `Por hacer / En progreso / En
revision / Completado` ya esta definido - ver [[00 - Prototipo Final ECOREX]]
(pantalla "Proyecto - detalle + Tablero Kanban"). En destino el DnD se implementa
en Blazor con persistencia transaccional y actualizacion viva por SignalR.

---

## Escenario 3 - Cerrar mi jornada y ver historial

### User Story
**Como** coordinadora,
**quiero** ver un resumen visual de las tareas cerradas hoy,
**para que** al final del dia pueda demostrar la productividad del equipo.

### Pasos concretos
1. Abre **Gestor de tareas / Tableros** y consulta la columna **Completado** o la vista **Timeline** del proyecto.
2. Ve un timeline vertical con las tareas cerradas ordenadas por fecha.
3. Cada entrada muestra: quien la cerro, prioridad, cuanto tiempo tomo.
4. Puede filtrar por rango de fechas y exportar.

### Que hace click aqui
El **timeline** lista los pasos completados de la instancia de flujo (filtrados por
estado "Completado" y ordenados por fecha de resolucion). La duracion se calcula
entre fecha de inicio y fecha de resolucion, normalizadas a UTC con la zona horaria
del tenant. En el prototipo esta vista corresponde a la pestana `Timeline` del
proyecto.

---

## Escenario 4 - Detectar tareas atascadas

### User Story
**Como** coordinadora,
**quiero** que el sistema me alerte cuando una tarea lleva mas de 24h sin avanzar,
**para que** intervenga antes de que el cliente reclame.

### Como lo resuelve el sistema destino
Esto es de primera clase en .NET 10 y aparece en el dashboard: la tarjeta KPI
**"Alertas urgentes"** de Inicio y el KPI operativo `workflow_stuck_rate` (flujos
trabados / total) miden justo esto.

- Un **worker asincrono** (Hangfire/BackgroundService) recorre las instancias de
  flujo del tenant y detecta las que llevan el nodo activo sin avanzar mas de 24h.
- Por cada una, el **RulesEngine** dispara una accion de notificacion y emite el
  evento `task.stuck` al Event Bus.
- El **Notification Service (SignalR)** empuja la alerta a la campana de Guadalupe
  en vivo, y la tarjeta "Alertas urgentes" incrementa su contador.

### Configuracion
Andres modela la regla de escalamiento una sola vez en el RulesEngine (verbo de
notificacion) y la asocia al umbral de 24h; el worker la ejecuta periodicamente por
tenant. Las fechas se comparan en UTC con la zona horaria de SKY SYSTEM para no
disparar alertas falsas por husos horarios.

---

## Escenario 5 - Un cliente pide un cambio de ultimo momento

### User Story
**Como** coordinadora,
**quiero** poder modificar los datos de una tarea en curso sin perder su historial,
**para que** el equipo no repita trabajo cuando el cliente cambia de opinion.

### Pasos concretos
1. Guadalupe abre la tarea T0042 (click en la tarjeta del tablero).
2. Se abre el panel de detalle con **datos + Seguimiento del Proceso**.
3. Ve los 6 pasos del flujo BPMN, con el paso 3 marcado como "En progreso".
4. En la seccion de datos, edita la fecha limite y guarda.
5. El WorkflowEngine registra el cambio como una edicion de la tarea (con concurrencia optimista) sin tocar el historial de pasos.

### Criterios de aceptacion
- El proceso BPMN NO se reinicia
- El historial de pasos anteriores queda intacto
- El ejecutor registrado de cada paso persiste
- Si otra persona edito la misma tarea a la vez, la concurrencia optimista (rowversion / xmin) evita pisar el cambio

---

## Escenario 6 - Crear un proyecto que agrupa 15 tareas relacionadas

### User Story
**Como** coordinadora,
**quiero** agrupar tareas relacionadas bajo un proyecto,
**para que** pueda ver el avance conjunto y calcular presupuestos.

### Pasos concretos
1. Va a **Proyectos** (`000042`) en el bloque MODULOS del sidebar.
2. Click "+ Nuevo proyecto".
3. Llena: nombre, tipo (Implementacion Cliente), coordinador, fecha inicio/fin, presupuesto, etiquetas y equipo (avatares).
4. Click "Crear".
5. Al crear una tarea nueva, en el combo "Proyecto" puede asociar al proyecto recien creado. El proyecto abre su propia vista con cabecera (Estado, Asignados, Fecha limite, Etiquetas) y su tablero Kanban.

### Criterios de aceptacion
- El proyecto se persiste con su `tenant_id` y alimenta el KPI "Proyectos en curso" del dashboard
- El coordinador queda como responsable con permiso de edicion y de vista
- El progreso se calcula: `tareas Completado / tareas total * 100`
- Al eliminar el proyecto, el sistema valida via policy que el usuario tenga permiso de edicion sobre el

---

## Recap de los modulos que Guadalupe toca

```
Inicio / Resumen (dashboard "Buenos dias" + KPIs)
  |
  +-- Bloque PRINCIPAL
  |     +-- Inicio / Anuncios / Gestor de tareas
  |
  +-- Bloque MODULOS
      +-- Crear actividad            000038  <- para pedidos nuevos
      +-- Proyectos                  000042  <- para agrupar (+ Kanban por proyecto)
      +-- Gestor de tareas/Tableros          <- vista principal (Kanban `Por hacer/En progreso/En revision/Completado`)
      +-- Programar actividad        000889  <- para tareas recurrentes
```

Y en la barra superior consulta constantemente:
- Contador de tareas pendientes (badge junto al icono)
- Campana de notificaciones (alertas en vivo por SignalR)
- Buscador global con atajo Cmd+K (para buscar cliente por NIT o nombre)

---

## Lo que Guadalupe NO hace pero podria querer

- Configurar sus propios flujos de trabajo (lo hace Andres en Flujos del proceso)
- Cambiar el color de estados (lo hace Andres)
- Ver el historial de reglas ejecutadas (auditoria - lo hace Julia)
- Editar plantillas de mail (lo hace Julia)
- Gobernar el tenant, planes o limites (eso es del PlatformAdmin, fuera de SKY SYSTEM)

---

## Enlaces a documentacion tecnica

- Aspecto y navegacion destino: [[00 - Prototipo Final ECOREX]]
- Vision maestra del sistema: [[Visión y entorno]]
- Motor de flujos: [[00 - Visión Flujos]] y [[AdmWorkflow - Motor de ejecucion]]
- Ejecucion paso a paso: [[Ejecucion - SiguienteEstado y Reinicios]]
- Paginas Tareas y Proyectos: [[Tareas y Proyectos - paginas basicas]]
