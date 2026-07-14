---
tipo: plan-qa
modulo: Flujos de Proceso (BPMN) (Capa 3)
proposito: Alcance de prueba, mapeo criterio->test, veredicto QA y riesgos/retos del motor de flujos (WorkflowEngine) y su ejecucion por pasos, gateways y bandeja.
estado: PENDIENTE DE AUDITORIA QA (2026-07-13) - construido, calidad no verificada por QA
fecha: 2026-07-13
---

# Plan y veredicto QA - Flujos de Proceso (BPMN)

> Aplica la [[00 - Politicas QA y Definition of Done]]. Cubre el WorkflowEngine: arrancar
> instancia, avanzar/rechazar, gateways, asignacion por cargo, bandeja (reclamar/atender/
> aprobar) y la union con formularios. Historias: [[Historias de prueba - Flujos de Proceso]].
> Spec: [[00 - Visión Flujos]], [[AdmWorkflow - Motor de ejecucion]],
> [[Ejecucion - SiguienteEstado y Reinicios]].

## 1. Alcance de prueba

- **Arranque**: StartInstance crea instancia + primer paso (IsCurrent, Pending).
- **Ejecucion**: SiguienteEstado / Advance / Reject; reinicios.
- **Gateways**: decision (aprobado/rechazado) que enruta la transicion.
- **Asignacion**: nodo -> cargo -> usuarios (OrgAssigneeTree / INodeAssigneeResolver).
- **Bandeja**: reclamar / atender / aprobar; completar paso.
- **Union con formulario**: nodo que exige formulario bloquea el avance hasta enviarlo.
- **Portabilidad BPMN** (.bpmn) y parametrizacion por nodo.

## 2. Mapeo criterio de aceptacion -> test requerido

| Criterio | Test requerido | Estado |
|---|---|---|
| Arrancar instancia deja el primer paso current/pending | Integracion StartInstance | Por confirmar |
| Avanzar/rechazar mueve el estado correcto | Integracion Advance/Reject | Por confirmar |
| Gateway enruta segun decision | Integracion: aprobado vs rechazado | Por confirmar |
| Paso con formulario bloquea hasta enviar | Integracion form+flujo (misma tx) | Por confirmar |
| Asignacion por cargo resuelve usuarios | Integracion OrgAssigneeTree | Por confirmar |
| Aislamiento cross-tenant de instancias/bandeja | Test dedicado cross-tenant (dual) | Por confirmar |
| Reinicio de flujo | Integracion de reinicio | Por confirmar |

## 3. VEREDICTO QA (2026-07-13): PENDIENTE DE AUDITORIA

Motor construido y maduro segun la documentacion; **sin gate QA aun**. Sin veredicto de
calidad hasta correr la auditoria contra historias y DoD.

## 4. Riesgos y retos candidatos

| Sev | Riesgo / reto | Por que importa |
|---|---|---|
| **Alto** | Gateways: condicion como literal / casos borde | Un gateway mal resuelto estanca o desvia el caso |
| **Alto** | Concurrencia del DbContext en el paso con formulario | Ya hubo "second operation on this context"; regresion posible |
| **Alto** | Aislamiento cross-tenant de instancias y bandeja | Fuga de pasos/tareas entre tenants |
| **Alto** | DAL dual: SQL Server rezagado | Condicion de merge |
| **Medio** | Reinicios y reglas por actividad (casos borde) | Estados inconsistentes al reiniciar |
| **Medio** | Idempotencia al completar un paso dos veces | Doble avance |

## 5. Para pasar a HECHO

1. Suite de integracion dual del ciclo arrancar->avanzar->gateway->cerrar.
2. Test de union form+flujo en la misma transaccion (feliz + fallo).
3. Test de aislamiento cross-tenant de instancias/bandeja.
4. Test de concurrencia/idempotencia al completar paso.
5. Corrida en [[00 - Registro de corridas]].

Relacionado: [[Historias de prueba - Flujos de Proceso]], [[Reglas - Motor y discrepancia]],
[[Estrategia de Testing (.NET 10)]].
