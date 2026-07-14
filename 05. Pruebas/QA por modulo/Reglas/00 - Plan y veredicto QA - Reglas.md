---
tipo: plan-qa
modulo: Reglas (RulesEngine) (Capa 3)
proposito: Alcance de prueba, mapeo criterio->test, veredicto QA y riesgos/retos del motor de reglas y sus verbos tipados (Ensamblado del legacy -> RulesEngine con allow-list).
estado: PENDIENTE DE AUDITORIA QA (2026-07-13) - construido, calidad no verificada por QA
fecha: 2026-07-13
---

# Plan y veredicto QA - Reglas

> Aplica la [[00 - Politicas QA y Definition of Done]]. Cubre el motor de reglas: despacho de
> verbos tipados (sucesor de los `Ensamblado` por reflexion del legacy), acciones sobre
> formularios/tareas y la seguridad del sandbox. Historias: [[Historias de prueba - Reglas]].
> Spec: [[Reglas - Motor y discrepancia]], [[Reglas - Catalogo real y verbos Ensamblado]],
> [[Clases de Reglas del modulo Documental]].

## 1. Alcance de prueba

- **Despacho de reglas**: por evento/condicion, en orden, resultado de UI + datos.
- **Verbos tipados** (allow-list): migrar entre formularios, generar tareas, cotizador,
  consecutivo, integraciones (Siigo/WhatsApp/correo), plantillas PDF.
- **Seguridad**: NO reflexion abierta, NO SQL crudo (la deuda del legacy que no se hereda).
- **Reglas sobre formularios**: show/hide, setValue, required (FormRuleDispatcher).

## 2. Mapeo criterio de aceptacion -> test requerido

| Criterio | Test requerido | Estado |
|---|---|---|
| Una regla activa dispara su verbo en orden | Integracion del dispatcher | Por confirmar |
| Verbo fuera de la allow-list NO ejecuta | Unit/seguridad: rechazo | Por confirmar |
| Sin SQL crudo desde una regla | Revision + test negativo | Por confirmar |
| Reglas de campo (show/hide/setValue/required) | Integracion FormRuleDispatcher | Parcial (F1 usa parte) |
| Aislamiento cross-tenant de la ejecucion | Test dedicado cross-tenant | Por confirmar |
| Error en un verbo detiene la cadena de forma segura | Integracion de fallo | Por confirmar |

## 3. VEREDICTO QA (2026-07-13): PENDIENTE DE AUDITORIA

Motor construido; **sin gate QA**. Punto de atencion especial: **seguridad del sandbox** (el
legacy tenia RCE de facto por reflexion). Sin veredicto hasta auditar.

## 4. Riesgos y retos candidatos

| Sev | Riesgo / reto | Por que importa |
|---|---|---|
| **Critico** | Sandbox de verbos sin allow-list estricta | RCE / SQL crudo heredado del legacy |
| **Alto** | Aislamiento cross-tenant en verbos que tocan datos | Un verbo podria leer/escribir cross-tenant |
| **Alto** | Integraciones externas (Siigo/WhatsApp) sin mocks/timeouts | Cuelgues o efectos reales en pruebas |
| **Medio** | Orden y corte de cadena ante error | Reglas a medias dejan estado inconsistente |
| **Medio** | DAL dual en verbos que persisten | Paridad PG/SQL Server |

## 5. Para pasar a HECHO

1. Test de seguridad del sandbox (allow-list; rechaza verbo/expresion peligrosa).
2. Integracion del despacho (orden, corte por error, resultado).
3. Aislamiento cross-tenant de la ejecucion de reglas.
4. Mocks/timeouts para integraciones externas.
5. Corrida en [[00 - Registro de corridas]].

Relacionado: [[Historias de prueba - Reglas]], [[00 - Visión Flujos]],
[[Estrategia de Testing (.NET 10)]].
