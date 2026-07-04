---
tipo: historial-pruebas
proposito: Registrar aqui cada corrida del plan de validacion (fecha, alcance, resultado)
---

# Historial de pruebas

| Fecha | Alcance | Resultado | Notas |
|-------|---------|-----------|-------|
| 2026-07-03 | FASE 0+1: build solucion renombrada (Ecorex.sln), arranque Super Admin contra pila dedicada (Postgres 5442), seed SKY SYSTEM, DAL dual (migracion inicial SQL Server 1443 + seed), test de aislamiento cross-tenant en matriz dual | PASS | Build 0 errores. TenantIsolation 6/6 (3 casos x Postgres/SQL Server, Testcontainers). Canario verificado: con filtro roto el test falla. Unit tests 2/2. Commits a482b47, 0d09e16, dc323d5 en rama fase-0/clon-backbone de EcorexV |
| 2026-07-03 | Post limpieza belleza (ADR-0011) + menu Prototipo Final + migracion .NET 10/EF Core 10 (ADR-0012): build, unit tests, aislamiento dual, arranque en ambos motores | PASS | Build 0 errores en net10.0. TenantIsolation 6/6 dual. SuperAdmin /login 200 contra Postgres 5442 Y SQL Server 1443. 55 tablas identicas por motor. 12 rutas del menu sin 404/500. Commits e4907b7, cac187c, 0c1c3b0 |
| 2026-07-03 | FASE 3 nucleo tareas/proyectos: dominio (10 entidades, maquina de estados, consecutivos concurrente-safe, concurrencia optimista) + UI (kanban por estado, wizard 3 pasos, detalle con worklog timer, proyectos, SignalR) | PASS | Build 0 errores. Domain 35/35, Integration 41/41 dual (TenantIsolation 6 + TaskCore 8 + Auth 27 tras fix IAgentAssetReader). E2E real en navegador: alta T00006, worklog timer/manual, drag valido/invalido con validacion de transiciones, kanban filtrado por proyecto. Commits 240c088, 50b1a27, 2add5b0 |
| 2026-07-03 | FASE 4 COMPLETA (3 motores): WorkflowEngine BPMN (ADR-0014), DynamicFormRenderer con visor por token (ADR-0015), RulesEngine con verbos tipados (ADR-0016) + alineacion visual milimetrica al prototipo ECOREX.dc.html | PASS | Suite completa: Domain 35, Application 83, Integration 77/77 en matriz dual (Postgres 16 + SQL Server 2022, Testcontainers). Probado en vivo: flujo COT-COM sembrado, formulario FRM-001 vinculado a nodo, regla ASIGNAR_CONSECUTIVO ejecutada con historial TTL. UI verificada con computed styles contra el prototipo (tabla de fidelidad en PROGRESO.md). Commits 93995d6, 9016828, 96e196c, 7cbf17b, d58e690, cb6ac63 |
| 2026-07-03 | FASE 5: Dependencias/organigrama (000850) + Modulos web/registry (000109) + gaps estructurales del prototipo (acordeones, seccion Modulos, badges KPI, colapso sidebar) | PASS | Suite: Domain 35, Application 91, Integration 85/85 dual. Verificado en navegador: arbol de unidades con ciclos rechazados, toggle de modulos persistido, settings jsonb. Commits 6b9a5c8, 1a22907 |
| 2026-07-03 | FASE 7: primer run del CI pr-check en GitHub Actions (gitleaks + build Release + dotnet format bloqueante + 126 unit + 85 integracion matriz dual con Testcontainers) | PASS | Run #1 conclusion=success en el runner ubuntu-latest. Commit 9933a89. Gates de merge activos |
| (pendiente) | Plan completo | - | Primera corrida del plan completo al cerrar el editor bpmn-js y arrancar FASE 6 (ETL) |

Ver el checklist en [[Plan de validacion del sistema]].
