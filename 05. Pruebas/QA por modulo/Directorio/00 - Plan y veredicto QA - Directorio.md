---
tipo: plan-qa
modulo: Directorio (Terceros) (Capa 6)
proposito: Alcance de prueba, mapeo criterio->test, veredicto QA y riesgos/retos del directorio de terceros (empresas/clientes/sospechosos) con campos dinamicos y sub-permisos.
estado: PENDIENTE DE AUDITORIA QA (2026-07-13) - construido, calidad no verificada por QA
fecha: 2026-07-13
---

# Plan y veredicto QA - Directorio (Terceros)

> Aplica la [[00 - Politicas QA y Definition of Done]]. Cubre la creacion unificada de
> terceros (empresas, clientes, sospechosos), sus campos dinamicos por tenant y los
> sub-permisos por tipo. Es fuente de lookups de Formularios (F1). Historias:
> [[Historias de prueba - Directorio]]. Codigo: `TerceroService`, `TerceroFieldService`,
> `DirectorioSubPermisos`.

## 1. Alcance de prueba

- **CRUD de terceros** por tipo (empresa/cliente/sospechoso) con transicion entre tipos.
- **Campos dinamicos por tenant** (TerceroFieldService): definir y capturar.
- **Sub-permisos por tipo** (crear empresa vs cliente vs sospechoso).
- **Consumo como lookup** desde Formularios (busqueda + autollenado).

## 2. Mapeo criterio de aceptacion -> test requerido

| Criterio | Test requerido | Estado |
|---|---|---|
| Crear/editar tercero por tipo | Integracion CRUD | Por confirmar |
| Campos dinamicos se guardan/leen | Integracion campos dinamicos | Por confirmar |
| Sub-permisos gatean la accion por tipo | Integracion de policy por sub-permiso | Por confirmar |
| Aislamiento cross-tenant de terceros | Test dedicado cross-tenant (dual) | Por confirmar |
| Busqueda server-side paginada (lookup) | Integracion de busqueda | Por confirmar |

## 3. VEREDICTO QA (2026-07-13): PENDIENTE DE AUDITORIA

Construido; **sin gate QA**. Critico por ser fuente de datos de otros modulos (lookups).

## 4. Riesgos y retos candidatos

| Sev | Riesgo / reto | Por que importa |
|---|---|---|
| **Critico** | Aislamiento cross-tenant (base de lookups) | Fuga de clientes entre tenants via Formularios |
| **Alto** | Sub-permisos por tipo mal aplicados | Un rol crea tipos que no debe |
| **Alto** | DAL dual (SQL Server) | Condicion de merge |
| **Medio** | Campos dinamicos: tipos/validacion | Datos basura si no se validan |
| **Medio** | Busqueda no paginada con catalogo grande | Rendimiento/fuga masiva |

## 5. Para pasar a HECHO

1. Test de aislamiento cross-tenant (CRUD + lookup).
2. Integracion de sub-permisos por tipo.
3. Integracion dual de campos dinamicos.
4. Busqueda paginada probada con catalogo grande.
5. Corrida en [[00 - Registro de corridas]].

Relacionado: [[Historias de prueba - Directorio]],
[[00 - Plan y veredicto QA - Formularios]] (lookups), [[Estrategia de Testing (.NET 10)]].
