---
tipo: plan-qa
modulo: Gestion de tenant (Multi-tenant) (Capa 1)
proposito: Alcance de prueba, mapeo criterio->test, veredicto QA y riesgos/retos del backbone multi-tenant: admin de empresas, aislamiento (HasQueryFilter + RLS) y Super Admin.
estado: PENDIENTE DE AUDITORIA QA (2026-07-13) - backbone construido, calidad no verificada por QA
fecha: 2026-07-13
---

# Plan y veredicto QA - Gestion de tenant (Multi-tenant)

> Aplica la [[00 - Politicas QA y Definition of Done]]. Es el backbone que TODOS los demas
> modulos dependen: aislamiento por tenant, admin de empresas, Super Admin, configuracion de
> la entidad. El aislamiento cross-tenant probado aqui es la base de confianza del sistema.
> Historias: [[Historias de prueba - Gestion de tenant]]. Spec:
> [[Gestion de Empresas - Admin multi-tenant]], [[Seguridad y Autenticacion multi-tenant]].

## 1. Alcance de prueba

- **Aislamiento**: `HasQueryFilter` por TenantId + RLS en BD; imposible ver/tocar cross-tenant.
- **Admin de empresas / tenants**: crear tenant, usuarios por tenant.
- **Super Admin**: operaciones de plataforma (MFA), sin fugar entre tenants.
- **Configuracion de la entidad** (Sede/Area) que consumen otros modulos.
- **Ambient tenant** (visor por token fija el tenant correcto).

## 2. Mapeo criterio de aceptacion -> test requerido

| Criterio | Test requerido | Estado |
|---|---|---|
| Un tenant nunca ve datos de otro (global) | Bateria de aislamiento por entidad | Por confirmar |
| RLS activo en BD (defensa en profundidad) | Test de RLS a nivel BD (dual) | Por confirmar |
| Crear tenant + usuario | Integracion | Por confirmar |
| Super Admin no fuga entre tenants | Integracion de plataforma | Por confirmar |
| Ambient del visor por token | Integracion del visor anonimo | Por confirmar |

## 3. VEREDICTO QA (2026-07-13): PENDIENTE DE AUDITORIA

Backbone construido; **sin gate QA**. Es el modulo de mayor severidad: un fallo aqui
compromete TODO el sistema. Prioridad alta de auditoria.

## 4. Riesgos y retos candidatos

| Sev | Riesgo / reto | Por que importa |
|---|---|---|
| **Critico** | Cualquier consulta sin filtro global (bypass del aislamiento) | Fuga masiva cross-tenant |
| **Critico** | RLS ausente o mal configurado en BD | Sin defensa en profundidad si falla el filtro EF |
| **Alto** | IgnoreQueryFilters usado fuera del visor por token | Punto unico legitimo; cualquier otro uso es riesgo |
| **Alto** | DAL dual: RLS y filtros en PG y SQL Server | Paridad de aislamiento |
| **Medio** | Super Admin / MFA | Escalada de privilegios |

## 5. Para pasar a HECHO

1. Bateria sistematica de aislamiento cross-tenant por CADA entidad (dual).
2. Test de RLS a nivel BD (defensa en profundidad).
3. Auditoria de todos los `IgnoreQueryFilters` (solo el visor por token es legitimo).
4. Integracion de creacion de tenant/usuario + Super Admin.
5. Corrida en [[00 - Registro de corridas]].

Relacionado: [[Historias de prueba - Gestion de tenant]],
[[Seguridad y Autenticacion multi-tenant]], [[Estrategia de Testing (.NET 10)]].
