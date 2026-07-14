---
tipo: plan-qa
modulo: Menu, Roles y Permisos (Capa 2)
proposito: Alcance de prueba, mapeo criterio->test, veredicto QA y riesgos/retos del menu configurable por perfil, roles dinamicos y enforcement (menu filtrado + acciones gateadas).
estado: PENDIENTE DE AUDITORIA QA (2026-07-13) - construido, calidad no verificada por QA
fecha: 2026-07-13
---

# Plan y veredicto QA - Menu, Roles y Permisos

> Aplica la [[00 - Politicas QA y Definition of Done]]. Cubre el menu data-driven por tenant,
> los roles dinamicos con matriz de permisos (Modulo x Ver/Crear/Editar/Eliminar) y el
> enforcement real (menu podado por Ver + acciones gateadas por policy). Historias:
> [[Historias de prueba - Menu-Roles-Permisos]]. Codigo: `MenuConfigService`,
> `MenuTreeBuilder`, `MenuPermissionFilter`.

## 1. Alcance de prueba

- **Menu configurable** por perfil/vista, editor, drag-and-drop.
- **Roles dinamicos**: matriz Modulo x (Ver/Crear/Editar/Eliminar).
- **Enforcement**: menu filtrado por Ver + acciones gateadas por policy; regla opt-in que no
  bloquea a Owner/Admin ni a usuario sin rol.
- **Grupo dinamico** (Mis Procesos) integrado con Conceptos.

## 2. Mapeo criterio de aceptacion -> test requerido

| Criterio | Test requerido | Estado |
|---|---|---|
| Menu se poda por permiso Ver | Integracion MenuPermissionFilter | Por confirmar |
| Accion sin permiso se rechaza (policy) | Integracion de policy por accion | Por confirmar |
| Owner/Admin no se bloquean | Integracion de la regla opt-in | Por confirmar |
| Aislamiento cross-tenant de menus/roles | Test dedicado cross-tenant | Por confirmar |
| Grupo dinamico refleja Conceptos | Integracion (con Tareas) | Por confirmar |

## 3. VEREDICTO QA (2026-07-13): PENDIENTE DE AUDITORIA

Construido; **sin gate QA**. Severidad alta: define quien ve/hace que.

## 4. Riesgos y retos candidatos

| Sev | Riesgo / reto | Por que importa |
|---|---|---|
| **Critico** | Accion gateada solo en el menu, no en el backend | Ocultar el boton no es seguridad; hay que gatear la accion |
| **Alto** | Escalada por rol mal compuesto | Un rol ve/hace mas de lo debido |
| **Alto** | Vistas de menu faltantes para usuarios reales (prod) | Usuarios sin vista o con vista incorrecta |
| **Alto** | Aislamiento cross-tenant de roles/vistas | Roles de un tenant aplicados en otro |
| **Medio** | Regla opt-in (Owner/Admin/sin rol) casos borde | Bloqueos o permisos indebidos |

## 5. Para pasar a HECHO

1. Test de enforcement en el BACKEND (no solo poda de menu): accion sin permiso -> 403.
2. Test de composicion de roles (matriz) + regla opt-in.
3. Test de aislamiento cross-tenant de menus/roles.
4. Integracion del grupo dinamico con Conceptos.
5. Corrida en [[00 - Registro de corridas]].

Relacionado: [[Historias de prueba - Menu-Roles-Permisos]],
[[Seguridad y Autenticacion multi-tenant]], [[Estrategia de Testing (.NET 10)]].
