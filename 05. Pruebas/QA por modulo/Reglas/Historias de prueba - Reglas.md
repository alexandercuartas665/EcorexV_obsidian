---
tipo: historias-qa
modulo: Reglas (RulesEngine) (Capa 3)
proposito: Historias de prueba Dado/Cuando/Entonces, AJUSTABLES por el usuario, fuente de los tests.
estado: borrador ajustable
fecha: 2026-07-13
---

# Historias de prueba - Reglas

> Ajustalas libremente. Politica y DoD: [[00 - Politicas QA y Definition of Done]].
> Veredicto: [[00 - Plan y veredicto QA - Reglas]].

### H1 - Disparo de regla activa
- **Dado** un campo/evento con una regla activa
- **Cuando** ocurre el evento (cambio de valor, envio)
- **Entonces** el verbo se ejecuta y devuelve la accion/dato esperado
- Severidad: Alto | Cubierta: por confirmar

### H2 - Allow-list rechaza verbo peligroso (adversarial, CRITICO)
- **Dado** una regla que intenta invocar algo fuera de la allow-list (reflexion, SQL crudo)
- **Cuando** el motor la procesa
- **Entonces** la rechaza y no ejecuta (sin RCE)
- Severidad: **Critico** | Cubierta: por confirmar

### H3 - Corte de cadena ante error
- **Dado** una cadena de reglas
- **Cuando** un verbo falla con error marcado
- **Entonces** la cadena se detiene de forma segura y se informa; no deja estado a medias
- Severidad: Medio | Cubierta: por confirmar

### H4 - Regla de campo (show/hide/required)
- **Dado** un formulario con reglas de campo
- **Cuando** cambia el campo disparador
- **Entonces** se aplican show/hide/setValue/required y los campos ocultos no se validan como requeridos
- Severidad: Alto | Cubierta: parcial

### H5 - Integracion externa con timeout (adversarial)
- **Dado** un verbo que llama un servicio externo (Siigo/WhatsApp)
- **Cuando** el servicio no responde
- **Entonces** hay timeout y manejo de error; la prueba usa mock (sin efecto real)
- Severidad: Alto | Cubierta: por confirmar

### H6 - Aislamiento cross-tenant
- **Dado** reglas y datos de dos tenants
- **Cuando** una regla del tenant B se ejecuta
- **Entonces** solo lee/escribe datos del tenant B
- Severidad: **Critico** | Cubierta: por confirmar

> Agrega/ajusta historias; marca severidad y datos exactos (verbo, tenant).
