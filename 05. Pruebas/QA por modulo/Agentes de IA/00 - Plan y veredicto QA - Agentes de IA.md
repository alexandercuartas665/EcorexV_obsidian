---
tipo: plan-qa
modulo: Agentes de IA (Capa 7)
proposito: Alcance de prueba, mapeo criterio->test, veredicto QA y riesgos/retos de la capa de IA gobernada: gateway multi-proveedor, asistente operativo, copiloto de configuracion, cupos y guardrails.
estado: PENDIENTE DE AUDITORIA QA (2026-07-13) - parcial/propuesta, calidad no verificada por QA
fecha: 2026-07-13
---

# Plan y veredicto QA - Agentes de IA

> Aplica la [[00 - Politicas QA y Definition of Done]]. Cubre la capa inteligente: AI Provider
> Gateway multi-proveedor, Asistente Operativo (crea/mueve tareas), Copiloto de Configuracion
> (propone flujos/formularios/reglas), cupos por plan y guardrails. Historias:
> [[Historias de prueba - Agentes de IA]]. Spec: [[Agentes de IA - Arquitectura y Operacion]].

## 1. Alcance de prueba

- **Gateway multi-proveedor**: seleccion de proveedor, fallback, timeouts.
- **Asistente Operativo**: crear/mover tareas via IA, dentro de permisos del usuario.
- **Copiloto de Configuracion**: propone flujos/formularios/reglas (el humano aprueba).
- **Cupos por plan** y **guardrails** (prompt injection, acciones peligrosas, costos).

## 2. Mapeo criterio de aceptacion -> test requerido

| Criterio | Test requerido | Estado |
|---|---|---|
| El asistente respeta permisos del usuario | Integracion: accion IA fuera de permiso -> rechazada | Por confirmar |
| El copiloto propone, no aplica sin aprobacion | Integracion del flujo propuesta->aprobacion | Por confirmar |
| Cupos por plan se respetan | Integracion de cupo agotado | Por confirmar |
| Guardrail contra prompt injection | Test adversarial de inyeccion | Por confirmar |
| Aislamiento cross-tenant del contexto IA | Test dedicado cross-tenant | Por confirmar |
| Gateway: fallback/timeout de proveedor | Integracion con mock de proveedor | Por confirmar |

## 3. VEREDICTO QA (2026-07-13): PENDIENTE DE AUDITORIA

Parcial/propuesta; **sin gate QA**. Alta sensibilidad de seguridad (la IA actua sobre el
sistema): los guardrails y el respeto de permisos son criticos.

## 4. Riesgos y retos candidatos

| Sev | Riesgo / reto | Por que importa |
|---|---|---|
| **Critico** | La IA ejecuta acciones fuera de los permisos del usuario | Escalada via IA |
| **Critico** | Prompt injection desde datos/documentos | La IA obedece instrucciones inyectadas |
| **Critico** | Fuga cross-tenant en el contexto/memoria de la IA | Datos de un tenant en la respuesta a otro |
| **Alto** | Copiloto aplica cambios sin aprobacion humana | Config no revisada en produccion |
| **Alto** | Costos/cupos sin control | Gasto descontrolado por proveedor |
| **Medio** | Proveedor caido sin fallback/timeout | Cuelgues del asistente |

## 5. Para pasar a HECHO

1. Test de que la IA opera SOLO dentro de los permisos del usuario (sin escalada).
2. Test adversarial de prompt injection (datos como datos, no como ordenes).
3. Test de aislamiento cross-tenant del contexto IA.
4. Test del flujo propuesta->aprobacion del copiloto (nada se aplica sin humano).
5. Test de cupos y de fallback/timeout del gateway.
6. Corrida en [[00 - Registro de corridas]].

Relacionado: [[Historias de prueba - Agentes de IA]],
[[Agentes de IA - Arquitectura y Operacion]], [[Estrategia de Testing (.NET 10)]].
