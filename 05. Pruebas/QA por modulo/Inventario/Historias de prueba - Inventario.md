---
tipo: historias-qa
modulo: Inventario (Items / Catalogos) (Capa 6)
proposito: Historias de prueba Dado/Cuando/Entonces, AJUSTABLES por el usuario, fuente de los tests.
estado: borrador ajustable
fecha: 2026-07-13
---

# Historias de prueba - Inventario (Items)

> Ajustalas libremente. Politica y DoD: [[00 - Politicas QA y Definition of Done]].
> Veredicto: [[00 - Plan y veredicto QA - Inventario]].

### H1 - Crear item con catalogo
- **Dado** el modulo Inventario
- **Cuando** se crea un item con categoria y unidad (catalogo normalizado)
- **Entonces** queda guardado y consultable, sin duplicar el catalogo
- Severidad: Alto | Cubierta: por confirmar

### H2 - Autollenado de precio como lookup
- **Dado** un formulario con campo "Item" (Origen=Inventario)
- **Cuando** el usuario elige un item
- **Entonces** se copian precio y unidad a sus campos (y F2 calcula el subtotal)
- Severidad: Alto | Cubierta: por confirmar

### H3 - Aislamiento cross-tenant (CRITICO)
- **Dado** items del tenant A y del tenant B
- **Cuando** un usuario del tenant B busca items
- **Entonces** nunca ve items del tenant A
- Severidad: **Critico** | Cubierta: por confirmar

### H4 - Precio erroneo no rompe el calculo (adversarial)
- **Dado** un item con precio 0 o nulo
- **Cuando** se usa en una linea de cotizacion
- **Entonces** el calculo lo maneja sin reventar (subtotal 0, no excepcion)
- Severidad: Medio | Cubierta: por confirmar

> Agrega/ajusta historias; marca severidad y datos exactos (item, tenant).
