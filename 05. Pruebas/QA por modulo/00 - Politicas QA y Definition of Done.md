---
tipo: politica-qa
capa: transversal
proposito: Politica de QA estricta + persona/brief del agente QA + Definition of Done con dientes, aplicable a TODO modulo desarrollado. No bloquea el merge; informa riesgos y retos y el usuario decide.
estado: activo
fecha: 2026-07-13
---

# Politicas QA y Definition of Done (transversal)

> Capa de **enforcement de calidad** encima de la [[Estrategia de Testing (.NET 10)]]
> (esa define COMO se prueba el destino; esta define QUE se exige antes de decir HECHO,
> y quien lo verifica). Se aplica a **todos los grandes modulos** del sistema; cada uno
> tiene su carpeta con plan/veredicto e historias de prueba ajustables por el usuario.
> **El QA NO bloquea el merge**: reporta riesgos y retos con severidad, y el usuario
> decide. Complementa (no reemplaza) la [[Plan de validacion del sistema|validacion del origen]].

## 1. La persona QA (brief del agente)

Cuando se lanza una sesion QA, actua con esta identidad:

- **Rol**: ingeniero QA independiente y adversarial. NO eres quien codeo la feature;
  tu trabajo es **intentar romperla** y **rechazar el "HECHO" sin evidencia**, no aplaudirla.
- **Sesgo por defecto**: esceptico. Un criterio de aceptacion no se da por cumplido hasta
  que existe un **test automatizado y repetible** que lo prueba. "Verificado en navegador"
  (manual) NO cuenta como prueba de aceptacion: cuenta como humo hasta que hay test.
- **Salida**: un **reporte de riesgos y retos** (seccion 4), no un parche. El QA no arregla;
  el QA encuentra, prioriza y explica el fallo concreto (input -> resultado erroneo).
- **Independencia**: idealmente el QA corre en sesion/worktree aparte del build, para no
  heredar sus supuestos.

## 2. Definition of Done (DoD) - lo que separa "construido" de "HECHO"

Una feature es HECHO solo si cumple TODO esto (si algo falta, es "construido, no HECHO"):

1. **Cada criterio de aceptacion mapeado a un test con nombre** (no prosa: un test que corre).
2. **Test automatizado verde** que cubre el camino feliz Y al menos un camino de error.
3. **Matriz DAL DUAL**: los tests de integracion corren en **PostgreSQL Y SQL Server**
   (Testcontainers), no solo en el PG local. La paridad dual es condicion de DoD.
4. **Aislamiento cross-tenant probado** con un test dedicado siempre que la feature lea o
   escriba datos del tenant (imposible ver/tocar datos de otro tenant).
5. **Migracion reversible**: `up` y `down` probados; la migracion aplica limpio sobre una
   copia del esquema real (no solo sobre una BD vacia).
6. **El servidor es la fuente de verdad**: si hay montos/calculos/identidad, existe un test
   con **input malicioso del cliente** que demuestra que el servidor recalcula/valida y
   descarta lo que mando el cliente.
7. **Cobertura de TODAS las variantes declaradas**, no una: si la feature dice "3 fuentes"
   (Directorio/Inventario/Contenedor), las 3 se prueban; si dice "2 modos", los 2.
8. **Corre en CI** (no solo en la maquina del que codeo) y queda registrado en
   [[00 - Registro de corridas]].
9. **Sin regresiones**: la suite existente sigue verde.

## 3. Reglas de rechazo (anti-patrones que invalidan un "HECHO")

El QA marca NO-HECHO (con severidad) si ve:

- Claim sin evidencia ("ya quedo", "funciona") sin test ni pasos reproducibles.
- Verificacion **solo manual** en navegador como unica prueba de un criterio.
- **PG-only**: migracion/tests que solo corrieron en PostgreSQL, SQL Server sin ejecutar.
- **Una sola variante** probada cuando la spec declara varias (un adaptador de tres, etc.).
- Falta el **test de aislamiento cross-tenant** en algo que lee datos del tenant.
- Montos/identidad que dependen del cliente sin revalidacion servidor probada.
- "Funciona en mi maquina": no corre en CI / no reproducible.
- Migracion sin `down` probado o que asume BD vacia.

## 4. Severidad y reporte (el QA informa; NO bloquea)

Cada hallazgo se reporta con severidad y con el fallo CONCRETO (input -> salida erronea):

| Severidad | Significado | Ejemplo |
|---|---|---|
| **Critico** | Rompe seguridad/datos o incumple una regla transversal inviolable | Un lookup deja ver terceros de otro tenant |
| **Alto** | Criterio de aceptacion sin cubrir; probable bug en produccion | Solo se probo 1 de 3 adaptadores; SQL Server sin ejecutar |
| **Medio** | Riesgo real pero acotado o mitigable | Falta test de reversibilidad de migracion |
| **Bajo** | Deuda menor / cosmetico | Nombre de test poco descriptivo |

El reporte lista **riesgos y retos** priorizados. **No bloquea el merge**: el usuario lee,
decide que se atiende ahora y que queda como deuda, y puede dar override explicito. El
veredicto vive en el `00 - Plan y veredicto QA` del modulo.

## 5. Como engancha al flujo de trabajo (olas)

```
build (ola F/Ola N)  ->  QA gate (sesion QA independiente)  ->  reporte de riesgos y retos
                                                             ->  el usuario decide (no bloquea)
                                                             ->  se actualiza el veredicto del modulo
```

El QA se puede lanzar como agente con el brief de la seccion 1 + el
`00 - Plan y veredicto QA` del modulo + sus `Historias de prueba`.

## 6. Estructura por modulo y catalogo

Cada gran modulo tiene su carpeta bajo `05. Pruebas/QA por modulo/<Modulo>/` con:

- **`00 - Plan y veredicto QA - <Modulo>.md`**: alcance, mapeo criterio-de-aceptacion ->
  test requerido, **veredicto QA actual** (HECHO / construido-no-HECHO) y **riesgos y retos**.
- **`Historias de prueba - <Modulo>.md`**: historias de usuario en formato Dado/Cuando/
  Entonces, **ajustables por el usuario**; son la fuente de la que salen los tests.

Catalogo de modulos (a confirmar/podar con el usuario):

| Modulo | Capa | Estado de desarrollo | Carpeta QA |
|---|---|---|---|
| Formularios | 4 | F1/F2/F3 construidos | Formularios/ (plantilla lista) |
| Tareas y Proyectos | 2 | Olas 1-7 + P1/P3 en prod | Tareas y Proyectos/ |
| Flujos de Proceso (BPMN) | 3 | WorkflowEngine construido | Flujos de Proceso/ |
| Reglas | 3 | RulesEngine construido | Reglas/ |
| Directorio (Terceros) | 6 | construido | Directorio/ |
| Inventario (Items) | 6 | construido | Inventario/ |
| Contenedor de datos | 6 | construido | Contenedor de datos/ |
| Gestion de tenant (multi-tenant) | 1 | backbone | Gestion de tenant/ |
| Menu, Roles y Permisos | 2 | construido | Menu-Roles-Permisos/ |
| Agentes de IA | 7 | parcial/propuesta | Agentes de IA/ |

## 7. Reglas transversales que el QA verifica en TODO modulo

Independientes de la feature; su incumplimiento es Critico/Alto:

- **Multi-tenant real** (HasQueryFilter + RLS): cero fuga cross-tenant.
- **DAL dual** (PG y SQL Server): paridad probada.
- **Cero SQL concatenado**: consultas parametrizadas / expresiones tipadas.
- **Transaccion** en operaciones multi-tabla (todo o nada).
- **Auditoria + soft-delete + concurrencia optimista** donde aplique.
- **Sandbox tipado** para formulas/acciones (sin reflexion abierta ni SQL crudo: la deuda
  del legacy que NO se hereda).

Relacionado: [[Estrategia de Testing (.NET 10)]], [[00 - Registro de corridas]],
[[00 - Indice y personas del sistema]] (personas del negocio; estas de QA son de prueba).
