---
tipo: historial-pruebas
proposito: Registrar aqui cada corrida del plan de validacion (fecha, alcance, resultado)
---

# Historial de pruebas

| Fecha | Alcance | Resultado | Notas |
|-------|---------|-----------|-------|
| 2026-07-03 | FASE 0+1: build solucion renombrada (Ecorex.sln), arranque Super Admin contra pila dedicada (Postgres 5442), seed SKY SYSTEM, DAL dual (migracion inicial SQL Server 1443 + seed), test de aislamiento cross-tenant en matriz dual | PASS | Build 0 errores. TenantIsolation 6/6 (3 casos x Postgres/SQL Server, Testcontainers). Canario verificado: con filtro roto el test falla. Unit tests 2/2. Commits a482b47, 0d09e16, dc323d5 en rama fase-0/clon-backbone de EcorexV |
| 2026-07-03 | Post limpieza belleza (ADR-0011) + menu Prototipo Final + migracion .NET 10/EF Core 10 (ADR-0012): build, unit tests, aislamiento dual, arranque en ambos motores | PASS | Build 0 errores en net10.0. TenantIsolation 6/6 dual. SuperAdmin /login 200 contra Postgres 5442 Y SQL Server 1443. 55 tablas identicas por motor. 12 rutas del menu sin 404/500. Commits e4907b7, cac187c, 0c1c3b0 |
| (pendiente) | Plan completo | - | Primera corrida del plan completo al cerrar Fase 2 (menu + auth) |

Ver el checklist en [[Plan de validacion del sistema]].
