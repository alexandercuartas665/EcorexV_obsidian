---
tipo: historias-qa
modulo: Directorio (Terceros) (Capa 6)
proposito: Historias de prueba Dado/Cuando/Entonces, AJUSTABLES por el usuario, fuente de los tests.
estado: borrador ajustable
fecha: 2026-07-13
---

# Historias de prueba - Directorio (Terceros)

> Ajustalas libremente. Politica y DoD: [[00 - Politicas QA y Definition of Done]].
> Veredicto: [[00 - Plan y veredicto QA - Directorio]].

### H1 - Crear tercero por tipo
- **Dado** el modulo Directorio
- **Cuando** se crea un cliente con sus datos y campos dinamicos
- **Entonces** queda guardado con su tipo y sus campos, consultable
- Severidad: Alto | Cubierta: por confirmar

### H2 - Sub-permiso por tipo (adversarial)
- **Dado** un rol con permiso para crear "cliente" pero no "empresa"
- **Cuando** intenta crear una empresa
- **Entonces** la accion se rechaza (sub-permiso)
- Severidad: Alto | Cubierta: por confirmar

### H3 - Aislamiento cross-tenant (CRITICO)
- **Dado** terceros del tenant A y del tenant B
- **Cuando** un usuario del tenant B lista o busca terceros
- **Entonces** nunca ve terceros del tenant A
- Severidad: **Critico** | Cubierta: por confirmar

### H4 - Campos dinamicos
- **Dado** un tenant que define un campo dinamico "segmento"
- **Cuando** captura un tercero con ese campo
- **Entonces** el valor se guarda y se lee correctamente
- Severidad: Medio | Cubierta: por confirmar

### H5 - Busqueda paginada (lookup)
- **Dado** 50.000 terceros
- **Cuando** un formulario usa el lookup de cliente
- **Entonces** la busqueda devuelve solo una pagina acotada, server-side
- Severidad: Alto | Cubierta: por confirmar

> Agrega/ajusta historias; marca severidad y datos exactos (tipo, tenant).
