---
tipo: indice-maestro
proyecto: 044. Tareas (ECOREX)
tenant_demo: SKY SYSTEM
usuario_demo: 1048064705 (GUADALUPE.LA)
url_local: http://localhost:47640/GestionMovil/Formularios/Modulos/Login/LoginBitcode.aspx
url_prod: https://app.bitcode.com.co/Formularios/Modulos/Login/LoginBitcode.aspx
base_datos_dev: db3dev (sql.bitcode.com.co,44566)
total_docs: 52
carpetas: 8
fecha_ultima_actualizacion: 2026-07-02
estado: adaptado a la vision del sistema destino .NET 10 (ORIGEN preservado como referencia ETL)
nivel_completitud: 90 por ciento
---

# INDICE MAESTRO - OBSIDIAN.tareas

> **Especificacion del nuevo ECOREX - Sistema de Tareas sobre .NET 10.** El vault
> describe el sistema DESTINO (ASP.NET Core 10 / EF Core 10 / Blazor / DAL dual
> PostgreSQL o SQL Server / multi-tenant real con `TenantId` + `HasQueryFilter` +
> RLS) y conserva el sistema ORIGEN (WebForms + VB.NET + `MotherData`, multi-tenant
> por columna `SUCURSAL`) como **referencia de migracion / plano del ETL**. Cada
> nota tecnica separa ORIGEN (que hacia el legacy) de DESTINO (como se reconstruye).
> El aspecto y la navegacion definitivos estan en [[00 - Prototipo Final ECOREX]] y
> la vision maestra en [[Visión y entorno]].
>
> **Organizacion del vault**: identica a CUBOT.nails — `01. Requerimiento`
> por capas + `Prototipo`, Inventario, Hoja de Ruta, Notas dev,
> Pruebas (Modelo + Historial), Deploy, Historias.

## Estructura del vault

```
OBSIDIAN.tareas/
├── 00 - INDICE.md                              (este archivo)
├── 01. Requerimiento/
│   ├── Capa 0 Vision General/                  (vision + shell + puntos ciegos)
│   ├── Capa 1 Gestion de tenant/               (multi-tenant SUCURSAL)
│   ├── Capa 2 Tareas y Proyectos/              (nucleo operativo + specs UI)
│   ├── Capa 3 Flujos de Tareas BPMN/           (AdmWorkflow + reglas + .bpmn)
│   ├── Capa 4 Constructor de Formularios/      (EAV + renderers + visor token)
│   ├── Capa 5 Librerias Base/                  (MotherData + Funciones)
│   ├── Capa 6 Modulos Ecorex Especificaciones/ (5 specs auditadas 10/10)
│   ├── Capa 7 Agentes de IA/                   (capa inteligente: asistente + copiloto de configuracion)
│   └── Prototipo/                              (PROTOTIPO FINAL .NET 10 + capturas + conceptos DC)
├── 02. Inventario de modulos/                  (inventario + mapa + ER + tablas SQL)
├── 03. Hoja de Ruta desarrollo/                (cierres doc + ruta migracion)
├── 04. Notas para desarrollador/               (manejo de datos, alias, consecutivos)
├── 05. Pruebas/
│   ├── Modelo de pruebas/                      (plan de validacion manual)
│   └── Historial de pruebas/                   (registro de corridas)
├── 06. Deploy/                                 (IIS + publish profiles Contabo)
└── 10. Historias de Usuario/                   (4 personas + E2E + auditoria)
```

## 01. Requerimiento — por capas

### Capa 0 - Vision General
- [[Visión y entorno]] — que es ECOREX Tareas, stack, acceso de desarrollo, shell
- [[Shell del sistema - Master + Controles compartidos]] — MasterFormsII + topBar + controles compartidos
- [[Puntos ciegos y motores transversales]] — 11 areas transversales (Login, PermissionsManager, SignalR, ChatGPT, SendGrid, Azure Blob, Global.asax, FORX_DATA_FLUJO, Reportes, Deploy, Auditoria)

### Capa 1 - Gestion de tenant
- [[Gestion de Empresas - Admin multi-tenant]] — puente a la spec del modulo `000072`

### Capa 2 - Tareas y Proyectos (nucleo operativo)
- [[Tareas y Proyectos - paginas basicas]]
- [[ctrTareasII - Spec para reconstruir en Claude Design]] — wizard crear tarea (5 pasos)
- [[ctrVertareasII - Spec para reconstruir en Claude Design]] — detalle + timer worklog + 4 tabs

### Capa 3 - Flujos de Tareas BPMN (mod 000291)
- [[00 - Visión Flujos]]
- [[AdmWorkflow - Motor de ejecucion]]
- [[Ejecucion - SiguienteEstado y Reinicios]]
- [[Parametrizacion por nodo - panel Propiedades]]
- [[Portabilidad BPMN - prueba en bpmn.io]] (+ `ejemplo-bpmn-flujo-00001.bpmn`)
- [[Reglas - Motor y discrepancia]] / [[Reglas - Catalogo real y verbos Ensamblado]] / [[Reglas - Quien invoca realmente (cierre)]]
- [[gen_reglas - Spec para reconstruir en Claude Design]]

### Capa 4 - Constructor de Formularios (mod 000131)
- [[00 - Visión Formularios]]
- [[Constructor - Patron EAV y motor visual]]
- [[Esquema completo - Tablas y tipos de control]]
- [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]
- [[Visor por token - docu_viewform]]

### Capa 5 - Librerias Base
- [[00 - Visión MotherData]] — DAL. Fichas: [[AdmDatos - Motor SQL Server]], [[AdmNpgsql - Motor PostgreSQL]], [[AdmCrypto - Cifrado simétrico]], [[CargaConfig - Decode Config.xml]], [[VariablesGlobales - Conexiones y Empresa]]
- [[00 - Mapa Funciones]] — libreria compartida (Azure, ChatGPT, Excel, Mail...)

### Capa 6 - Modulos Ecorex Especificaciones
- [[00 - INDICE Modulos Ecorex]] — 5 specs auditadas (fidelidad 10/10), cada una con prototipo HTML:
  - [[NEWFRONT_tar_conceptos - Spec para reconstruir]] (000270)
  - [[gen_reglas - Spec para reconstruir]] (000802)
  - [[NEWFRONT_adm_empresas - Spec para reconstruir]] (000072)
  - [[NEWFRONT_web_scraping - Spec para reconstruir]] (000730)
  - [[comer_ContactLoader - Spec para reconstruir]] (000873)

### Capa 7 - Agentes de IA
- [[Agentes de IA - Arquitectura y Operacion]] — capa inteligente gobernada: AI Provider Gateway multi-proveedor, Asistente Operativo (crea/mueve tareas), Copiloto de Configuracion (propone flujos/formularios/reglas), cupos por plan, guardrails. Generaliza el `clChatGPT` legacy.

### Prototipo
- **[[00 - Prototipo Final ECOREX]]** — prototipo visual DEFINITIVO del sistema destino (.NET 10). Es el concepto de arranque del proyecto. Recorre cada pantalla con capturas embebidas
- `ECOREX - Prototipo Final.html` — SPA React ejecutable (self-contained, doble clic)
- `ECOREX.dc.html` + `support.js` — fuente Claude Design (para re-importar/editar)
- `capturas/` — screenshots oficiales por pantalla (Inicio, Tableros, Proyecto, Dependencias, Modulos web, Anuncios)
- `conceptos-claude-design/` — los 5 `proto_*.html` de concepto puntual (referencia por spec de Capa 6)

## 02. Inventario de modulos
- [[INVENTARIO GENERAL]] — todos los modulos por carpeta
- [[Mapa de Modulos - arbol ASCII]] — menus y opciones indexadas tipo arbol
- [[Modelo Entidad-Relacion logico]] — ER de las tablas nucleo
- [[Tablas detectadas por módulo]] — referencia SQL (db3dev)

## 03. Hoja de Ruta desarrollo
- [[HOJA DE RUTA DESARROLLO]] — plan completo de construccion .NET 10: estrategia de arranque, solucion Clean Architecture con DAL dual, menu del prototipo, fundamentos multi-tenant, motores base, ETL de datos legacy, CI/CD, checklist y riesgos

## 04. Notas para desarrollador
- [[Manejo de Datos - Alias, parametros, UDFs, consecutivos]] — `[dbx.GENE]`, PARAMXML, UDFs, consecutivos
- [[Seguridad y Autenticacion multi-tenant]] — JWT+refresh, MFA PlatformAdmin, policies (ex-PermissionsManager), RLS, secretos en Key Vault
- [[ADRs - Decisiones de arquitectura]] — 10 decisiones (DAL dual, multi-tenant real, Blazor Server, Guid v7, EAV->jsonb, bpmn-js, ETL...) con su por que

## 05. Pruebas
- Modelo de pruebas: [[Estrategia de Testing (.NET 10)]] — DESTINO: piramide, matriz Testcontainers dual, test de aislamiento cross-tenant, ETL, E2E Playwright
- Modelo de pruebas: [[Plan de validacion del sistema]] — ORIGEN: checklist manual del legacy por modulo
- Historial de pruebas: [[00 - Registro de corridas]] — bitacora de corridas

## 06. Deploy
- [[Deploy a Produccion - Docker e hibrido]] — DESTINO: Docker + hibrido VPS/Railway, DAL dual, workers, checklist piloto
- [[CICD Pipeline]] — GitHub Actions: build + test dual (Postgres/SQL Server), blue/green, rollback <2 min
- [[Backup y DR por proveedor]] — RTO/RPO, procedimientos diferenciados PostgreSQL / SQL Server, playbooks DR
- [[Deploy IIS y perfiles de publicacion]] — ORIGEN (referencia): Web Deploy a Contabo (legacy)

## 10. Historias de Usuario
- [[00 - Indice y personas del sistema]] — 4 personas del sistema
- [[01 - Historias del Operativo]] (Guadalupe) / [[02 - Historias del Configurador]] (Andres) / [[03 - Historias del Administrador]] (Julia) / [[04 - Historias del Cliente y Respondedor]] (Carlos + Maria)
- [[05 - Un dia en SKY SYSTEM (narrativa E2E)]] — 17 modulos en 9 horas
- [[06 - Auditoria de la documentacion (post-revision)]] — auditoria 2026-07-01

## Como leer el vault (por rol)

### Soy desarrollador nuevo en el sistema
1. [[Visión y entorno]] → 2. [[Shell del sistema - Master + Controles compartidos]] → 3. [[Manejo de Datos - Alias, parametros, UDFs, consecutivos]] → 4. [[INVENTARIO GENERAL]]

### Voy a tocar Flujos o Reglas
1. [[00 - Visión Flujos]] → 2. [[AdmWorkflow - Motor de ejecucion]] → 3. [[Reglas - Quien invoca realmente (cierre)]]

### Voy a tocar Formularios
1. [[00 - Visión Formularios]] → 2. [[Constructor - Patron EAV y motor visual]] → 3. [[Esquema completo - Tablas y tipos de control]]

### Quiero ver como se vera el sistema nuevo
1. [[00 - Prototipo Final ECOREX]] → abrir `ECOREX - Prototipo Final.html` en el navegador

### Voy a reconstruir un modulo en Claude Design
1. [[00 - INDICE Modulos Ecorex]] → spec del modulo (Capa 6) → concepto HTML en `01. Requerimiento/Prototipo/conceptos-claude-design/`

### Quiero entender el negocio sin leer codigo
1. [[00 - Indice y personas del sistema]] → 2. [[05 - Un dia en SKY SYSTEM (narrativa E2E)]]

### Voy a planear la migracion
1. [[HOJA DE RUTA DESARROLLO]] → 2. vault CUBOT.nails (esquema destino: DAL dual PostgreSQL/SQL Server + modulos base portados)

## Novedades (2026-07-03)

- **Vault adaptado a la vision destino (.NET 10)**: se reescribio la vision maestra [[Visión y entorno]] al nivel del referente CUBOT.nails (18 secciones) y luego se adaptaron ~40 notas de todas las capas (0-6, Inventario, Notas dev, Historias) para reflejar el sistema DESTINO .NET 10, conservando el analisis legacy como referencia de ETL. Patron: cada nota separa ORIGEN vs DESTINO. Trabajo hecho con 7 subagentes en paralelo + auditoria de integridad (0 links rotos reales)
- **Capa 1 y vision como destino**: [[Gestion de Empresas - Admin multi-tenant]] y [[Visión y entorno]] son ahora specs del sistema destino (multi-tenant real, correccion de los 9 errores heredados)
- **Prototipo Final incorporado**: llego el prototipo visual definitivo del sistema destino (.NET 10) desde Claude Design. Vive en `01. Requerimiento/Prototipo/` con su nota [[00 - Prototipo Final ECOREX]], SPA ejecutable, fuente `.dc` y 12 capturas por pantalla. **Es el concepto de arranque del proyecto**
- **Capturas viejas eliminadas**: se quitaron los 3 GIFs de produccion del sistema legacy (ya no representan el aspecto objetivo). Las referencias en Flujos/Formularios se reapuntaron al prototipo
- **Conceptos DC agrupados**: los 5 `proto_*.html` puntuales pasaron a `Prototipo/conceptos-claude-design/` (siguen como referencia por spec de Capa 6)
- README de capturas obsoleto eliminado

## Novedades (2026-07-02)

- **Vault espejado 1:1 con CUBOT.nails**: mismo arbol raiz (01-06 + 10), `01. Requerimiento` con capas + carpeta `Prototipo`, `05. Pruebas` con `Modelo de pruebas` + `Historial de pruebas`
- Las carpetas `07. Capturas` y `11. Modulos Ecorex Especificaciones` se absorbieron: assets → `01. Requerimiento/Prototipo/`, specs → `Capa 6`
- Todos los wikilinks son nombre-puro (inmunes a movimientos de carpetas)
- Nueva bitacora: [[00 - Registro de corridas]] en Historial de pruebas

## Estado

- **Total notas**: 52 md (+ prototipo final HTML/`.dc`, 12 capturas, 1 .bpmn, 5 conceptos DC)
- **Prototipo**: [[00 - Prototipo Final ECOREX]] es la referencia visual oficial del proyecto
- **Cobertura**: ~90 por ciento — el resto son TODOs listados en la [[HOJA DE RUTA DESARROLLO]]
- **Convencion**: notas nuevas en ASCII; wikilinks por nombre de archivo (sin rutas)
