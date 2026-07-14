---
tipo: historias-qa
modulo: Gestion de tenant (Multi-tenant) (Capa 1)
proposito: Historias de prueba Dado/Cuando/Entonces, AJUSTABLES por el usuario, fuente de los tests.
estado: borrador ajustable
fecha: 2026-07-13
---

# Historias de prueba - Gestion de tenant (Multi-tenant)

> Ajustalas libremente. Politica y DoD: [[00 - Politicas QA y Definition of Done]].
> Veredicto: [[00 - Plan y veredicto QA - Gestion de tenant]].

### H1 - Aislamiento total (CRITICO)
- **Dado** dos tenants con datos en cada entidad (tareas, terceros, formularios, etc.)
- **Cuando** un usuario del tenant B consulta cualquier entidad
- **Entonces** nunca ve, cuenta ni exporta datos del tenant A
- Severidad: **Critico** | Cubierta: por confirmar

### H2 - RLS como defensa en profundidad (adversarial)
- **Dado** una consulta que (por bug) omite el filtro EF
- **Cuando** llega a la BD
- **Entonces** RLS la corta igual (no devuelve filas de otro tenant)
- Severidad: **Critico** | Cubierta: por confirmar

### H3 - Crear tenant y usuario
- **Dado** el Super Admin
- **Cuando** crea un tenant nuevo y su primer usuario
- **Entonces** el tenant queda aislado y el usuario solo ve lo suyo
- Severidad: Alto | Cubierta: por confirmar

### H4 - IgnoreQueryFilters acotado
- **Dado** el codigo del sistema
- **Cuando** se auditan los usos de IgnoreQueryFilters
- **Entonces** el unico uso legitimo es el visor por token (que fija el ambient del tenant)
- Severidad: Alto | Cubierta: por confirmar

### H5 - Visor por token fija el tenant correcto
- **Dado** un token de un formulario del tenant A
- **Cuando** se abre el visor anonimo
- **Entonces** el ambient queda en el tenant A y todo lo demas se filtra por A
- Severidad: Alto | Cubierta: por confirmar

> Agrega/ajusta historias; marca severidad y datos exactos (tenants, entidad).
