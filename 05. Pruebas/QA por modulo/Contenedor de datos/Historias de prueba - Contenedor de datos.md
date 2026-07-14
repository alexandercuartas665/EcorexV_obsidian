---
tipo: historias-qa
modulo: Contenedor de datos (Capa 6)
proposito: Historias de prueba Dado/Cuando/Entonces, AJUSTABLES por el usuario, fuente de los tests.
estado: borrador ajustable
fecha: 2026-07-13
---

# Historias de prueba - Contenedor de datos

> Ajustalas libremente. Politica y DoD: [[00 - Politicas QA y Definition of Done]].
> Veredicto: [[00 - Plan y veredicto QA - Contenedor de datos]].

### H1 - Import paginado completo
- **Dado** una API REST con 3 paginas de datos
- **Cuando** se importa al contenedor
- **Entonces** se traen las 3 paginas completas (no solo la primera)
- Severidad: Alto | Cubierta: por confirmar

### H2 - Upsert por clave (idempotencia)
- **Dado** un contenedor con datos ya importados
- **Cuando** se reimporta con modo upsert por clave
- **Entonces** actualiza los existentes y no crea duplicados
- Severidad: Alto | Cubierta: por confirmar

### H3 - API externa caida (adversarial)
- **Dado** un import en curso
- **Cuando** la API externa no responde o corta a mitad
- **Entonces** hay timeout/manejo de error y el contenedor no queda en estado corrupto
- Severidad: Alto | Cubierta: por confirmar

### H4 - Aislamiento cross-tenant (CRITICO)
- **Dado** contenedores del tenant A y del tenant B
- **Cuando** un usuario del tenant B consulta/usa el contenedor como lookup
- **Entonces** nunca ve datos del tenant A
- Severidad: **Critico** | Cubierta: por confirmar

> Agrega/ajusta historias; marca severidad y datos exactos (fuente, modo, tenant).
