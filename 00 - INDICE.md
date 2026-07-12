---
tipo: indice-maestro
proyecto: 044. Tareas (ECOREX)
tenant_demo: SKY SYSTEM
usuario_demo: 1048064705 (GUADALUPE.LA)
url_local: http://localhost:47640/GestionMovil/Formularios/Modulos/Login/LoginBitcode.aspx
url_prod: https://app.bitcode.com.co/Formularios/Modulos/Login/LoginBitcode.aspx
base_datos_dev: db3dev (sql.bitcode.com.co,44566)
total_docs: 56
carpetas: 8
fecha_ultima_actualizacion: 2026-07-11
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
> El aspecto y la navegacion definitivos estan en [[Visión y entorno|Prototipo Final ECOREX]] y
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
- [[AdmWorkflow - Motor de flujo de la tarea]] — el proceso que acompaña a una tarea: ciclo de vida, reglas por actividad, reinicios, autorizacion, mover a flujo (AdmWorkflow.vb, 1613 lineas)
- [[crtFlujoProcesos - Graficador de flujo en el visor]] — herramienta que dibuja el flujo (bpmn-js) dentro del visor de tareas, con paneles de reglas/componentes/permisos por nodo

### Capa 3 - Flujos de Tareas BPMN (mod 000291)
- [[00 - Visión Flujos]]
- [[AdmWorkflow - Motor de ejecucion]]
- [[Ejecucion - SiguienteEstado y Reinicios]]
- [[Parametrizacion por nodo - panel Propiedades]]
- [[Portabilidad BPMN - prueba en bpmn.io]] (+ `ejemplo-bpmn-flujo-00001.bpmn`)
- [[Reglas - Motor y discrepancia]] / [[Reglas - Catalogo real y verbos Ensamblado]] / [[Reglas - Quien invoca realmente (cierre)]]
- [[gen_reglas - Spec para reconstruir en Claude Design]]
- [[Clases de Reglas del modulo Documental]] — las 11 clases `cl_*_reglas`: despacho por `Type.GetType`+`Activator`+`Invoke` de verbos `Ensamblado` (migrar a formulario, generar tareas, cotizador, Siigo, plantillas PDF->Blob, ChatGPT, WhatsApp). Riesgo: reflexion abierta = RCE de facto sin allow-list. (Movida desde Capa 4: es el motor de Reglas, no el constructor/ejecutor de formularios; su cara form-specific `cl_form_reglas_formularios` se referencia desde [[00 - Visión Formularios]])

### Capa 4 - Constructor de Formularios (mod 000131)
- [[00 - Visión Formularios]]
- [[Constructor - Patron EAV y motor visual]]
- [[Esquema completo - Tablas y tipos de control]]
- [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]
- [[Visor por token - docu_viewform]]
- **Ingenieria inversa del codigo real (`Documental/`)** — 61 archivos VB del motor vigente:
  - [[Clases del nucleo de formularios (Documental)]] — las 9 clases `cl_*` (definicion EAV en `ENCUESTAS_MOV_*` + propiedades en `FORX_DATA`, motor de calculos por grafo `REG_ORIGEN->REG_DESTINO`, reglas por **reflexion**). Riesgo: secreto Azure embebido en `cl_Transcripcion_audio.vb`, SQL concatenado
  - [[Controles del constructor y renderers (Documental)]] — los 38 `.ascx`: disenador `ctrFormCreator` (WYSIWYG) + renderer vigente `crtCargaEncuestaII` (dispatch oculto en `cl_gestion_formularios.AdornoDetalle`) + catalogo de controles por tipo de campo (Quill, firma canvas, audio+transcripcion, GPS, reCAPTCHA, grid detalle, tabla hija)
  - [[FormCreatorMCP - Servidor MCP del constructor]] — endpoint `.ashx` que expone **20 herramientas** para que un agente de IA construya formularios de punta a punta; usa **Gemini 2.0 Flash Vision** (tool estrella `import_form_from_image`: imagen -> blueprint -> formulario). RPC JSON propietario (no MCP estandar). Vive en Capa 4 (construye formularios); su evolucion destino es el Copiloto de Configuracion de [[Agentes de IA - Arquitectura y Operacion|Capa 7]]. Riesgo: sin auth, CORS `*`, SQL por concatenacion
- **Propiedades avanzadas (PROPUESTA, capitulo de 4 docs)** — el salto del motor a dato transaccional gobernado: [[00 - INDICE y objetivo (Formularios avanzados)]] (estado real + objetivo), [[01 - Arquitectura, decisiones y datos (Formularios avanzados)]] (formulario-modulo, transaccionalidad/consecutivo, lookups de datos con dominio del tenant, calculo/totales), [[02 - UX y paneles de propiedades (Formularios avanzados)]] y [[03 - Plan por olas y preguntas abiertas (Formularios avanzados)]]. Reutiliza infra existente (`TenantSequence`, `DataContainer`, Directorio, Inventario)

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
- **[[Visión y entorno|Prototipo Final ECOREX]]** — prototipo visual DEFINITIVO del sistema destino (.NET 10). Es el concepto de arranque del proyecto. Recorre cada pantalla con capturas embebidas
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
- [[PROMPT DE ARRANQUE - Sesion de desarrollo]] — prompt maestro auto-contenido para que una nueva sesion (multi-agente) inicie el desarrollo: rutas, lectura obligatoria, clonar backbone CUBOT.nails, Super Admin primero, registro de avance
- [[Pendientes y deudas tecnicas]] — backlog vivo: deudas por modulo, stubs sin construir, FASE 6 (ETL) y abiertos de proceso

## 04. Notas para desarrollador
- [[Manejo de Datos - Alias, parametros, UDFs, consecutivos]] — `[dbx.GENE]`, PARAMXML, UDFs, consecutivos
- [[Conexion a la BD del sistema actual (db3dev)]] — BD real del sistema actual (copia de produccion): tenant 01=BITCODE, 532 modulos, para descubrimiento y ETL. Credenciales FUERA del repo
- [[Seguridad y Autenticacion multi-tenant]] — JWT+refresh, MFA PlatformAdmin, policies (ex-PermissionsManager), RLS, secretos en Key Vault
- [[ADRs - Decisiones de arquitectura]] — 10 decisiones (DAL dual, multi-tenant real, Blazor Server, Guid v7, EAV->jsonb, bpmn-js, ETL...) con su por que

## 05. Pruebas
- Modelo de pruebas: [[Estrategia de Testing (.NET 10)]] — DESTINO: piramide, matriz Testcontainers dual, test de aislamiento cross-tenant, ETL, E2E Playwright
- Modelo de pruebas: [[Plan de validacion del sistema]] — ORIGEN: checklist manual del legacy por modulo
- Modelo de pruebas: [[CREDENCIALES - Usuarios y claves]] — plan de cuentas demo Development (sin secretos reales; passwords en .env)
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
1. [[Visión y entorno|Prototipo Final ECOREX]] → abrir `ECOREX - Prototipo Final.html` en el navegador

### Voy a reconstruir un modulo en Claude Design
1. [[00 - INDICE Modulos Ecorex]] → spec del modulo (Capa 6) → concepto HTML en `01. Requerimiento/Prototipo/conceptos-claude-design/`

### Quiero entender el negocio sin leer codigo
1. [[00 - Indice y personas del sistema]] → 2. [[05 - Un dia en SKY SYSTEM (narrativa E2E)]]

### Voy a planear la migracion
1. [[HOJA DE RUTA DESARROLLO]] → 2. vault CUBOT.nails (esquema destino: DAL dual PostgreSQL/SQL Server + modulos base portados)

## Novedades (2026-07-11)

- **Higiene de alcance de Capa 4 (formularios)**: se auditaron las notas de Capa 4 para dejarla
  acotada al **constructor** + **ejecutor** de formularios. `Clases de Reglas del modulo
  Documental` (los 11 verbos `Ensamblado`: Siigo, WhatsApp, PDF, IA, SQL crudo, migracion entre
  formularios) **se movio a Capa 3 (Reglas)** por ser el motor de Reglas, no el constructor.
  `FormCreatorMCP` **se queda en Capa 4** (construye formularios) con cross-link reforzado a
  [[Agentes de IA - Arquitectura y Operacion|Capa 7]] como su evolucion destino. Los wikilinks
  son por nombre, asi que el movimiento no rompe enlaces.
- **Capitulo nuevo en Capa 4 - "Propiedades avanzadas: Formulario como modulo y tabla de
  hechos" (PROPUESTA)**: a partir del dictado del usuario y de una validacion read-only del
  codigo real, se documento el salto del motor de formularios a **dato transaccional
  gobernado** en 4 docs (mismo esquema que el capitulo de Tareas): formulario promovido a
  **modulo** (menu+permisos+bandeja), **transaccionalidad** (registro con estado/fecha e
  identidad por **consecutivo** -reusa `TenantSequence`/`ISequenceService`- o **clave natural**
  tipo SKU/tercero), **campos con datos** (autocompletado desde `DataContainer`/Directorio/
  Inventario con autollenado), **calculo** (formulas, totales de tabla, roll-up) y **filtros/
  KPIs**. Entrada: [[00 - INDICE y objetivo (Formularios avanzados)]]. Nada construido aun;
  plan por olas F1..F6 + preguntas abiertas en el doc 03.
- **Ingenieria inversa profunda del motor de formularios (`Bootstrap/.../Documental/`)**:
  se documentaron las 4 carpetas del codigo VB real que sostiene el constructor
  (61 archivos, ~1.2 MB) en 4 notas nuevas de Capa 4, cada una con doble encuadre
  ORIGEN/DESTINO: [[Clases del nucleo de formularios (Documental)]] (9 clases `cl_*`),
  [[Controles del constructor y renderers (Documental)]] (38 `.ascx`),
  [[FormCreatorMCP - Servidor MCP del constructor]] (20 herramientas de IA, Gemini
  Vision) y [[Clases de Reglas del modulo Documental]] (11 clases de verbos `Ensamblado`).
- **Hallazgo clave**: `FormCreatorMCP` permite construir un formulario **a partir de una
  imagen** (foto -> blueprint -> formulario) via Gemini 2.0 Flash Vision; el motor de
  reglas ejecuta verbos por **reflexion** (`Type.GetType`+`Activator`+`Invoke`) sin
  allow-list. Ambos son insumo directo de las capas [[Agentes de IA - Arquitectura y Operacion]]
  y del `RulesEngine` tipado del destino.
- **Riesgos de seguridad del legacy anotados (no publicados verbatim)**: secreto Azure
  embebido en `cl_Transcripcion_audio.vb`, SQL por concatenacion generalizado, endpoint
  de IA sin auth con CORS `*`, reflexion abierta = RCE de facto. Refuerzan los 9 errores
  a NO heredar de la [[Gestion de Empresas - Admin multi-tenant]].

## Novedades (2026-07-07)

- **Hilo Usuarios -> Roles -> Enforcement** (ADR-0031/0032/0033, apoyo en el
  hermano Visal): modulo de usuarios del tenant, roles dinamicos con matriz de
  permisos (Modulo x Ver/Crear/Editar/Eliminar, catalogo derivado del menu) y
  enforcement real (menu filtrado por Ver + acciones gateadas, con regla opt-in
  que no bloquea a Owner/Admin ni a usuarios sin rol). Antes: **Menu
  configurable por perfil** (ADR-0030) con editor y drag-and-drop.
- **Nuevo backlog**: [[Pendientes y deudas tecnicas]] concentra todo lo
  pendiente (deudas por modulo, stubs, FASE 6 ETL, abiertos de proceso).
- Tracker DESTINO ([[INVENTARIO GENERAL]] 0.1) y [[00 - Registro de corridas]]
  al dia con estos modulos.

## Novedades (2026-07-05)

- **Tracker DESTINO reconciliado**: [[INVENTARIO GENERAL]] seccion 0.1 se actualizo con el
  estado real del codigo (commits `a482b47..39b066e`): FASE 5, los 5 modulos de Capa 6
  (conceptos, reglas, ficha de empresa, web scraping, cargador de contactos) y las 3
  integraciones traidas de la familia CUBOT — **Inventario con catalogos normalizados**
  (ADR-0027), **Plantillas WhatsApp HSM** (ADR-0029) e **Infraestructura IA** con su grupo
  de menu propio + desacople de `crear_lead` (ADR-0028).
- **Aclaracion de ADRs**: [[ADRs - Decisiones de arquitectura]] ahora explica que el repo
  `docs/decisiones/` lleva dos series (0001-0010 heredadas del backbone CUBOT + 0011-0029
  de implementacion de ECOREX Tareas), que NO mapean 1:1 con los 10 ADR estrategicos del vault.
- **Corridas al dia**: [[00 - Registro de corridas]] registra las corridas de las 3
  integraciones CUBOT (build + format + unit + integracion dual + E2E, todo verde).

## Novedades (2026-07-03)

- **Vault adaptado a la vision destino (.NET 10)**: se reescribio la vision maestra [[Visión y entorno]] al nivel del referente CUBOT.nails (18 secciones) y luego se adaptaron ~40 notas de todas las capas (0-6, Inventario, Notas dev, Historias) para reflejar el sistema DESTINO .NET 10, conservando el analisis legacy como referencia de ETL. Patron: cada nota separa ORIGEN vs DESTINO. Trabajo hecho con 7 subagentes en paralelo + auditoria de integridad (0 links rotos reales)
- **Capa 1 y vision como destino**: [[Gestion de Empresas - Admin multi-tenant]] y [[Visión y entorno]] son ahora specs del sistema destino (multi-tenant real, correccion de los 9 errores heredados)
- **Prototipo Final incorporado**: llego el prototipo visual definitivo del sistema destino (.NET 10) desde Claude Design. Vive en `01. Requerimiento/Prototipo/` con su nota [[Visión y entorno|Prototipo Final ECOREX]], SPA ejecutable, fuente `.dc` y 12 capturas por pantalla. **Es el concepto de arranque del proyecto**
- **Capturas viejas eliminadas**: se quitaron los 3 GIFs de produccion del sistema legacy (ya no representan el aspecto objetivo). Las referencias en Flujos/Formularios se reapuntaron al prototipo
- **Conceptos DC agrupados**: los 5 `proto_*.html` puntuales pasaron a `Prototipo/conceptos-claude-design/` (siguen como referencia por spec de Capa 6)
- README de capturas obsoleto eliminado

## Novedades (2026-07-02)

- **Vault espejado 1:1 con CUBOT.nails**: mismo arbol raiz (01-06 + 10), `01. Requerimiento` con capas + carpeta `Prototipo`, `05. Pruebas` con `Modelo de pruebas` + `Historial de pruebas`
- Las carpetas `07. Capturas` y `11. Modulos Ecorex Especificaciones` se absorbieron: assets → `01. Requerimiento/Prototipo/`, specs → `Capa 6`
- Todos los wikilinks son nombre-puro (inmunes a movimientos de carpetas)
- Nueva bitacora: [[00 - Registro de corridas]] en Historial de pruebas

## Estado

- **Total notas**: 56 md (+ prototipo final HTML/`.dc`, 12 capturas, 1 .bpmn, 5 conceptos DC)
- **Prototipo**: [[Visión y entorno|Prototipo Final ECOREX]] es la referencia visual oficial del proyecto
- **Cobertura**: ~90 por ciento — el resto son TODOs listados en la [[HOJA DE RUTA DESARROLLO]]
- **Convencion**: notas nuevas en ASCII; wikilinks por nombre de archivo (sin rutas)