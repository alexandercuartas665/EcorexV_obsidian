---
tipo: historias-qa
modulo: Tareas y Proyectos (Capa 2)
proposito: Historias de prueba Dado/Cuando/Entonces, AJUSTABLES por el usuario, fuente de los tests.
estado: borrador ajustable
fecha: 2026-07-13
---

# Historias de prueba - Tareas y Proyectos

> Ajustalas libremente. Politica y DoD: [[00 - Politicas QA y Definition of Done]].
> Veredicto: [[00 - Plan y veredicto QA - Tareas y Proyectos]].

## Creacion de actividad

### H1 - Actividad-proceso arranca el flujo
- **Dado** un concepto (subcategoria) con flujo asignado
- **Cuando** el usuario crea la actividad desde Mis Procesos
- **Entonces** se crea la tarea, arranca la WorkflowInstance y el primer paso queda asignado
  al cargo del nodo
- Severidad: Alto | Cubierta: por confirmar

### H2 - Actividad simple desde tablero
- **Dado** un concepto sin flujo
- **Cuando** el usuario la crea desde el tablero (Empresa/Area -> Tipo -> Actividad -> Encargado)
- **Entonces** se crea la tarea sin instancia de flujo y aparece en el tablero
- Severidad: Alto | Cubierta: por confirmar

### H3 - Form-first (IniciaModulo)
- **Dado** una subcategoria con IniciaModulo y formulario
- **Cuando** se crea la actividad
- **Entonces** el wizard abre el formulario y al enviarlo se completa el paso vinculado
- Severidad: Alto | Cubierta: por confirmar

### H4 - Transaccion todo-o-nada (adversarial)
- **Dado** el alta que crea tarea + flujo + form + notificacion
- **Cuando** falla un paso intermedio (ej. la notificacion)
- **Entonces** se revierte todo; no queda una tarea a medias sin flujo
- Severidad: Alto | Cubierta: por confirmar

## Menu y tableros

### H5 - Mis Procesos refleja Conceptos
- **Dado** que se agrega/quita un concepto-proceso
- **Cuando** el usuario abre el menu Mis Procesos
- **Entonces** ve la categoria->subcategoria actualizada sin re-seed
- Severidad: Medio | Cubierta: por confirmar

## Multi-tenant

### H6 - Aislamiento cross-tenant (CRITICO)
- **Dado** tareas del tenant A y del tenant B
- **Cuando** un usuario del tenant B abre sus tableros/bandejas
- **Entonces** nunca ve tareas del tenant A
- Severidad: **Critico** | Cubierta: por confirmar

## Proyectos

### H7 - Hito y enlace de actividad
- **Dado** un proyecto con un hito (P1)
- **Cuando** se enlaza una actividad al hito (P3)
- **Entonces** la actividad queda asociada y el avance del hito lo refleja
- Severidad: Medio | Cubierta: por confirmar

> Agrega/ajusta historias; marca severidad y datos (tenant, concepto) exactos.
