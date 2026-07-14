---
tipo: plan-qa
modulo: Formularios (Capa 4)
proposito: Alcance de prueba, mapeo criterio-de-aceptacion -> test requerido, veredicto QA actual y riesgos/retos del constructor + ejecutor de formularios y sus olas avanzadas F1/F2/F3.
estado: veredicto inicial 2026-07-13 - CONSTRUIDO, NO-HECHO (faltan pruebas)
fecha: 2026-07-13
---

# Plan y veredicto QA - Formularios

> Aplica la [[00 - Politicas QA y Definition of Done]] al modulo de Formularios. Cubre el
> motor base (constructor + ejecutor) y las olas avanzadas F1 (lookups), F2 (calculo) y
> F3 (transaccionalidad). Fuente de las historias: [[Historias de prueba - Formularios]].
> Spec del modulo: [[00 - INDICE y objetivo (Formularios avanzados)]] y su
> [[03 - Plan por olas y preguntas abiertas (Formularios avanzados)|plan por olas]].

## 1. Alcance de prueba

- **Motor base**: definir (arbol contenedores/preguntas), renderizar (controles Tier 1 +
  GridDetail), guardar respuesta (JSON), validar en servidor, publicar por token, union con
  flujo BPMN, reglas por campo.
- **F1 - Lookups/autollenado**: origen de datos (Directorio / Inventario / Contenedor),
  busqueda server-side paginada, autollenado por copia, aislamiento cross-tenant.
- **F2 - Calculo**: campo calculado (sandbox tipado), GridDetail (columna por fila, totales,
  roll-up), recomputo servidor.
- **F3 - Transaccionalidad**: identidad (consecutivo / clave natural), estado
  (Draft/Confirmed/Voided), fecha, anular, unicidad.

## 2. Mapeo criterio de aceptacion -> test requerido (DoD)

| Criterio (doc 03) | Test requerido | Estado |
|---|---|---|
| F1: autocompletar Directorio trae campos por copia | Integracion: elegir tercero -> data guarda id + copia NIT/ciudad | Manual OK / **falta test** |
| F1: adaptadores Item y Contenedor | Integracion por cada adaptador | **Falta** (solo Directorio) |
| F1: imposible ver datos de otro tenant | **Test aislamiento cross-tenant** (dual) | **Falta (PENDIENTE en doc 03)** |
| F1: no traer catalogo completo al cliente | Test: busqueda paginada, tope de filas | **Falta** |
| F2: subtotal/roll-up correctos | Unit del calculador (existe, 9 tests) + integracion en response | Unit OK / integracion parcial |
| F2: servidor corrige monto manipulado | Integracion: cliente manda total falso -> server recalcula | **Falta explicito** |
| F2: allow-list rechaza expresion peligrosa | Unit: expresion fuera de allow-list -> rechazada | Unit OK (16 tests) - confirmar cobertura |
| F3: consecutivo consume y no duplica | Integracion: 2 envios concurrentes -> numeros distintos | **Falta (concurrencia)** |
| F3: anular no libera numero | Integracion: void -> numero sigue ocupado | **Falta** |
| F3: clave natural unica rechaza duplicado | Integracion: 2 registros misma clave -> rechazo | **Falta** |
| Todos: matriz DAL dual (PG + SQL Server) | Testcontainers dual | **Falta (solo PG local)** |

## 3. VEREDICTO QA (2026-07-13): CONSTRUIDO, NO-HECHO

F1/F2/F3 estan marcados "HECHO / verificado en navegador" en el capitulo, pero por la
[[00 - Politicas QA y Definition of Done|DoD]] **aun no son HECHO**: la evidencia es
mayormente **manual**, falta la **matriz dual**, falta el **aislamiento cross-tenant** (que
es criterio de aceptacion Y regla transversal), y varias variantes no se probaron. Es buen
avance de construccion; el gate de calidad todavia no pasa.

## 4. Riesgos y retos (priorizados)

| Sev | Riesgo / reto | Por que importa | Reto para cerrarlo |
|---|---|---|---|
| **Critico** | Aislamiento cross-tenant del lookup **sin test** | Un lookup mal filtrado expondria terceros/items de otro cliente | Test dedicado cross-tenant (dual) por cada adaptador |
| **Alto** | **SQL Server sin ejecutar** (solo PG local) | La matriz dual es condicion de merge; el lado SQL Server podria fallar en migracion o consulta | Correr Testcontainers dual de F1/F2/F3 |
| **Alto** | Solo el adaptador **Directorio** verificado | Item y Contenedor pueden tener bugs no vistos | Probar los 3 adaptadores (auto + manual) |
| **Alto** | Recomputo servidor de montos **sin test de input malicioso** | Si el server no corrige, se pueden falsear totales | Test: cliente manda total falso -> server recalcula |
| **Medio** | Consecutivo bajo **concurrencia** sin test | Dos envios simultaneos podrian repetir numero | Test concurrente sobre `ISequenceService` |
| **Medio** | Migraciones **sin `down` probado** y aplicadas solo a BD local | Replicar a prod sin reversibilidad probada es fragil | Probar up/down sobre copia del esquema real |
| **Medio** | Dos campos de estado (`status` vs `record_status`) | Riesgo de confusion semantica (dos "Draft") | Test que fija la semantica de cada uno + decision documentada |
| **Bajo** | Deriva nombres doc vs codigo (enums `Form*`) | No rompe, pero desalinea la spec | Refrescar doc 01/02 (fuera del QA) |

## 5. Que se necesita para pasar a HECHO

1. Suite de integracion de F1/F2/F3 en **Testcontainers dual** (PG + SQL Server).
2. **Test de aislamiento cross-tenant** por adaptador de lookup.
3. Test de **recomputo servidor** con input manipulado (F2) y de **concurrencia** del
   consecutivo (F3).
4. Cobertura de los **3 adaptadores** y de los **2 modos de identidad**.
5. Reversibilidad de migracion probada; corrida registrada en [[00 - Registro de corridas]].

Relacionado: [[Historias de prueba - Formularios]],
[[03 - Plan por olas y preguntas abiertas (Formularios avanzados)]],
[[04 - Registro de tablas y cambios de esquema (Formularios avanzados)]],
[[Estrategia de Testing (.NET 10)]].
