---
tipo: plan-qa
modulo: Tareas y Proyectos (Capa 2)
proposito: Alcance de prueba, mapeo criterio->test, veredicto QA y riesgos/retos del modulo de tareas/actividades y proyectos (Olas 1-7 + P1/P3, en prod).
estado: PENDIENTE DE AUDITORIA QA (2026-07-13) - construido y en prod, calidad no verificada por QA
fecha: 2026-07-13
---

# Plan y veredicto QA - Tareas y Proyectos

> Aplica la [[00 - Politicas QA y Definition of Done]]. Cubre la creacion y ejecucion de
> actividades (que consumen el concepto), la asignacion por cargo, tableros, el menu Mis
> Procesos dinamico, notificaciones y Proyectos (hitos). Historias:
> [[Historias de prueba - Tareas y Proyectos]]. Spec:
> [[Tareas y Proyectos - paginas basicas]], [[ctrTareasII - Spec para reconstruir en Claude Design]].

## 1. Alcance de prueba

- **Alta que consume el concepto**: wizard 4 pasos, flujo + auto-titulo/detalle, asignacion
  por cargo, form-first (IniciaModulo), herencia de tablero.
- **Dos caminos**: actividad-proceso (Mis Procesos) y actividad simple (desde tablero).
- **Menu Mis Procesos dinamico**: categoria -> subcategoria (concepto con flujo).
- **Tableros**: deep-link por subcategoria, crear-desde-tablero.
- **Notificaciones**: al asignar (in-app), entrega real.
- **Proyectos**: hitos (P1) y enlace actividad->proyecto/hito (P3).

## 2. Mapeo criterio de aceptacion -> test requerido

| Criterio | Test requerido | Estado |
|---|---|---|
| El alta arranca el flujo del concepto | Integracion: crear actividad con subcategoria con flujo -> WorkflowInstance + primer paso | Por confirmar |
| Asignacion por cargo (encargado por defecto) | Integracion: cargo/responsable -> usuarios resueltos | Por confirmar |
| Form-first (IniciaModulo) abre formulario | Integracion + E2E del wizard | Por confirmar |
| Menu Mis Procesos refleja Conceptos | Integracion: alta/baja de concepto-proceso -> menu | Por confirmar |
| Aislamiento cross-tenant de tareas/tableros | Test dedicado cross-tenant (dual) | Por confirmar |
| Notificacion al asignar | Integracion: asignar -> notificacion in-app | Por confirmar |
| Proyectos: hito y enlace actividad | Integracion P1/P3 | Por confirmar |

## 3. VEREDICTO QA (2026-07-13): PENDIENTE DE AUDITORIA

El modulo esta construido y desplegado a prod, pero **no ha pasado por el gate QA**. No hay
veredicto de calidad hasta ejecutar la auditoria contra las historias y el DoD.

## 4. Riesgos y retos candidatos

| Sev | Riesgo / reto | Por que importa |
|---|---|---|
| **Alto** | DAL dual: SQL Server rezagado (migraciones solo PG) | Condicion de merge; el lado SQL Server podria no aplicar |
| **Alto** | Aislamiento cross-tenant de tareas/tableros sin test explicito | Fuga de tareas entre tenants |
| **Alto** | Transaccionalidad del alta (tarea + flujo + form + notif) | Debe ser todo-o-nada; parcial deja datos inconsistentes |
| **Medio** | Vistas de menu de usuarios reales incompletas (prod) | Usuarios sin vista no ven lo que deben |
| **Medio** | Config de Conceptos en prod via editor (no seed) | Si falta el cableado, el alta no arranca flujos/forms |
| **Medio** | Concurrencia del consecutivo de tarea (Number) | Dos altas simultaneas no deben repetir numero |

## 5. Para pasar a HECHO

1. Suite de integracion dual (PG + SQL Server) del alta que consume el concepto.
2. Test de aislamiento cross-tenant (tareas, tableros, bandejas).
3. Test de la transaccion del alta (rollback si falla un paso).
4. E2E del wizard 4 pasos + Mis Procesos + form-first.
5. Corrida registrada en [[00 - Registro de corridas]].

Relacionado: [[Historias de prueba - Tareas y Proyectos]], [[AdmWorkflow - Motor de flujo de la tarea]],
[[Estrategia de Testing (.NET 10)]].
