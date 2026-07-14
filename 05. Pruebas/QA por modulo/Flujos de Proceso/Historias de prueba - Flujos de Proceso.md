---
tipo: historias-qa
modulo: Flujos de Proceso (BPMN) (Capa 3)
proposito: Historias de prueba Dado/Cuando/Entonces, AJUSTABLES por el usuario, fuente de los tests.
estado: borrador ajustable
fecha: 2026-07-13
---

# Historias de prueba - Flujos de Proceso (BPMN)

> Ajustalas libremente. Politica y DoD: [[00 - Politicas QA y Definition of Done]].
> Veredicto: [[00 - Plan y veredicto QA - Flujos de Proceso]].

### H1 - Arranque de instancia
- **Dado** una definicion de flujo publicada
- **Cuando** se arranca una instancia
- **Entonces** existe la instancia y su primer paso queda current/pending, asignado al cargo del nodo
- Severidad: Alto | Cubierta: por confirmar

### H2 - Avanzar el flujo
- **Dado** un paso pendiente
- **Cuando** el usuario asignado lo atiende y avanza
- **Entonces** el paso se cierra y el siguiente queda current segun la definicion
- Severidad: Alto | Cubierta: por confirmar

### H3 - Gateway por decision (adversarial)
- **Dado** un nodo con gateway (aprobar/rechazar)
- **Cuando** la decision es Rechazado
- **Entonces** el caso enruta por la rama de rechazo (no por la de aprobado)
- Severidad: Alto | Cubierta: por confirmar

### H4 - Paso con formulario bloquea
- **Dado** un nodo que exige formulario
- **Cuando** el usuario intenta avanzar sin enviarlo
- **Entonces** el avance se bloquea; solo al enviar el formulario (misma tx) se completa el paso
- Severidad: Alto | Cubierta: por confirmar

### H5 - Completar paso dos veces (idempotencia)
- **Dado** un paso ya completado
- **Cuando** llega una segunda solicitud de completar el mismo paso
- **Entonces** no hay doble avance ni error de estado
- Severidad: Medio | Cubierta: por confirmar

### H6 - Aislamiento cross-tenant (CRITICO)
- **Dado** instancias del tenant A y del tenant B
- **Cuando** un usuario del tenant B abre su bandeja
- **Entonces** nunca ve pasos/instancias del tenant A
- Severidad: **Critico** | Cubierta: por confirmar

### H7 - Reinicio de flujo
- **Dado** una instancia en curso
- **Cuando** se reinicia segun la regla del proceso
- **Entonces** el estado vuelve al punto definido sin dejar datos inconsistentes
- Severidad: Medio | Cubierta: por confirmar

> Agrega/ajusta historias; marca severidad y datos exactos (definicion, nodo, tenant).
