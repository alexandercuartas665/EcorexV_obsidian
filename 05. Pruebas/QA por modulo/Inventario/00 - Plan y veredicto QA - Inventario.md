---
tipo: plan-qa
modulo: Inventario (Items / Catalogos) (Capa 6)
proposito: Alcance de prueba, mapeo criterio->test, veredicto QA y riesgos/retos del inventario de items y catalogos normalizados.
estado: PENDIENTE DE AUDITORIA QA (2026-07-13) - construido, calidad no verificada por QA
fecha: 2026-07-13
---

# Plan y veredicto QA - Inventario (Items)

> Aplica la [[00 - Politicas QA y Definition of Done]]. Cubre items/productos, catalogos
> normalizados y campos de item; fuente de lookups de Formularios (F1). Historias:
> [[Historias de prueba - Inventario]]. Codigo: `ItemService`, `InventoryCatalogService`,
> `ItemFieldService`.

## 1. Alcance de prueba

- **CRUD de items** + catalogos normalizados (categorias/unidades).
- **Campos de item** (ItemFieldService) por tenant.
- **Consumo como lookup** desde Formularios (busqueda + autollenado de precio/unidad).

## 2. Mapeo criterio de aceptacion -> test requerido

| Criterio | Test requerido | Estado |
|---|---|---|
| Crear/editar item + catalogo | Integracion CRUD | Por confirmar |
| Autollenado de precio/unidad como lookup | Integracion lookup (F1) | Por confirmar |
| Aislamiento cross-tenant de items | Test dedicado cross-tenant (dual) | Por confirmar |
| Catalogos normalizados sin duplicar | Integracion de normalizacion | Por confirmar |

## 3. VEREDICTO QA (2026-07-13): PENDIENTE DE AUDITORIA

Construido; **sin gate QA**. Relevante por alimentar montos (precio) en cotizaciones (F2).

## 4. Riesgos y retos candidatos

| Sev | Riesgo / reto | Por que importa |
|---|---|---|
| **Critico** | Aislamiento cross-tenant (fuente de lookups) | Fuga de items/precios entre tenants |
| **Alto** | Precio del item alimenta calculos (F2) | Un precio erroneo propaga a totales |
| **Alto** | DAL dual (SQL Server) | Condicion de merge |
| **Medio** | Normalizacion de catalogos (duplicados) | Datos sucios en reportes |

## 5. Para pasar a HECHO

1. Test de aislamiento cross-tenant (CRUD + lookup).
2. Integracion del autollenado precio/unidad (encadena con F1/F2).
3. Integracion dual de catalogos.
4. Corrida en [[00 - Registro de corridas]].

Relacionado: [[Historias de prueba - Inventario]],
[[00 - Plan y veredicto QA - Formularios]], [[Estrategia de Testing (.NET 10)]].
