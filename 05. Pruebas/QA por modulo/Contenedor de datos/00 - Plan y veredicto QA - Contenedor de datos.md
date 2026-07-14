---
tipo: plan-qa
modulo: Contenedor de datos (Capa 6)
proposito: Alcance de prueba, mapeo criterio->test, veredicto QA y riesgos/retos del contenedor de datos generico y su importacion desde API REST.
estado: PENDIENTE DE AUDITORIA QA (2026-07-13) - construido, calidad no verificada por QA
fecha: 2026-07-13
---

# Plan y veredicto QA - Contenedor de datos

> Aplica la [[00 - Politicas QA y Definition of Done]]. Cubre el data source generico por
> tenant y su import desde API REST (descubrir campos, mapeo, paginacion, modos de recarga).
> Fuente de lookups de Formularios (F1). Historias: [[Historias de prueba - Contenedor de datos]].
> Codigo: `DataContainerService`, `DataContainerConfig`, `ContenedorDatos.razor`.

## 1. Alcance de prueba

- **Definir contenedor** (campos) + datos por tenant.
- **Import API REST**: descubrir campos, mapeo, paginacion, modos (agregar / reemplazar /
  upsert por clave).
- **Consumo como lookup** desde Formularios.

## 2. Mapeo criterio de aceptacion -> test requerido

| Criterio | Test requerido | Estado |
|---|---|---|
| Import trae todas las paginas | Integracion de paginacion | Por confirmar |
| Upsert por clave no duplica | Integracion upsert | Por confirmar |
| Reemplazar limpia y recarga | Integracion modo reemplazo | Por confirmar |
| Aislamiento cross-tenant del contenedor | Test dedicado cross-tenant (dual) | Por confirmar |
| Lookup desde Formularios | Integracion (F1) | Por confirmar |

## 3. VEREDICTO QA (2026-07-13): PENDIENTE DE AUDITORIA

Construido; **sin gate QA**. Atencion a la robustez del import (fuentes externas).

## 4. Riesgos y retos candidatos

| Sev | Riesgo / reto | Por que importa |
|---|---|---|
| **Critico** | Aislamiento cross-tenant | Fuga de datos importados entre tenants |
| **Alto** | Import: API externa lenta/caida/paginacion rota | Import incompleto o cuelgue |
| **Alto** | Upsert por clave con claves duplicadas/nulas | Duplicados o perdida de datos |
| **Medio** | DAL dual (SQL Server) | Condicion de merge |
| **Medio** | Tipos de datos del import (fechas/numeros) | Datos mal tipados en el lookup |

## 5. Para pasar a HECHO

1. Integracion del import con mock de API (paginacion, timeout, error).
2. Test de upsert/reemplazo/agregar (idempotencia).
3. Test de aislamiento cross-tenant.
4. Integracion del lookup (encadena con F1).
5. Corrida en [[00 - Registro de corridas]].

Relacionado: [[Historias de prueba - Contenedor de datos]],
[[00 - Plan y veredicto QA - Formularios]], [[Estrategia de Testing (.NET 10)]].
