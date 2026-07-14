---
tipo: historias-qa
modulo: Menu, Roles y Permisos (Capa 2)
proposito: Historias de prueba Dado/Cuando/Entonces, AJUSTABLES por el usuario, fuente de los tests.
estado: borrador ajustable
fecha: 2026-07-13
---

# Historias de prueba - Menu, Roles y Permisos

> Ajustalas libremente. Politica y DoD: [[00 - Politicas QA y Definition of Done]].
> Veredicto: [[00 - Plan y veredicto QA - Menu-Roles-Permisos]].

### H1 - Menu podado por permiso Ver
- **Dado** un rol sin permiso Ver sobre un modulo
- **Cuando** el usuario abre el menu
- **Entonces** ese modulo no aparece
- Severidad: Alto | Cubierta: por confirmar

### H2 - Accion gateada en el backend (adversarial, CRITICO)
- **Dado** un usuario sin permiso Crear que oculta el boton pero llama la API directo
- **Cuando** invoca la accion de crear
- **Entonces** el backend la rechaza (403), no solo el menu
- Severidad: **Critico** | Cubierta: por confirmar

### H3 - Owner/Admin no se bloquean
- **Dado** la regla opt-in de permisos
- **Cuando** un Owner/Admin (o usuario sin rol) usa el sistema
- **Entonces** no queda bloqueado por la regla
- Severidad: Alto | Cubierta: por confirmar

### H4 - Aislamiento cross-tenant de roles
- **Dado** roles del tenant A y del tenant B
- **Cuando** se aplica el rol de un usuario del tenant B
- **Entonces** nunca hereda permisos/vistas del tenant A
- Severidad: **Critico** | Cubierta: por confirmar

### H5 - Grupo dinamico Mis Procesos
- **Dado** conceptos-proceso configurados
- **Cuando** el usuario abre el grupo dinamico del menu
- **Entonces** ve categoria->subcategoria segun permiso
- Severidad: Medio | Cubierta: por confirmar

> Agrega/ajusta historias; marca severidad y datos exactos (rol, modulo, tenant).
