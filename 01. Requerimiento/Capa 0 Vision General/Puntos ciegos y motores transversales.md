---
tipo: mapa-superficie
proposito: las 11 areas transversales del sistema, con su plano ORIGEN y su resolucion DESTINO (.NET 10)
audiencia: quien migra - saber que se enfrenta y como se resuelve en el destino
estado: vision destino (.NET 10) + referencia del origen
stack_destino: .NET 10 / ASP.NET Core / EF Core 10 / SignalR / Key Vault
---

# Puntos ciegos y motores transversales

> Las **areas transversales** que atraviesan todo el sistema (seguridad, tiempo
> real, IA, correo, almacenamiento, errores, auditoria, reportes, deploy). En el
> origen quedaron poco documentadas y arrastran errores heredados. Aqui, para cada
> una: el plano ORIGEN (donde vive, que hace, riesgo) y su **resolucion DESTINO en
> .NET 10**. Es el complemento del [[Shell del sistema - Master + Controles compartidos]]
> y baja al detalle transversal de [[Visión y entorno]] (secciones 4, 12, 14).
>
> Son 11 areas; para cada una el destino cierra el punto ciego, no solo lo describe.

---

## 1. Autenticacion y Login

### Origen
Pagina `Bootstrap\Formularios\Modulos\Login\LoginBitcode.aspx` (+ recovery
`recuperaPass.aspx`). Recibe usuario+clave, valida contra `[dbx.GENE].dbo.USUARIO`,
setea `Session("Usuario")`, `Session("Empresa")`, `Session("Nombre")` y redirige a
`das_module.aspx`. Al primer login con clave temporal (`12345`) fuerza cambio por
modal. Riesgos: **sin MFA**, sin rate limiting documentado, hash de clave sin
auditar (posible `MD5.vb`, deprecated).

### Destino .NET 10
Autenticacion por **JWT** (access + refresh token) con `TenantId`, `UserId` y roles
en los claims, emitida por un endpoint de identidad de ASP.NET Core. **MFA
obligatorio para PlatformAdmin** y opcional por politica de tenant. Hash de claves
con **Argon2id/PBKDF2** (nunca MD5). **Rate limiting** nativo
(`AddRateLimiter`) y bloqueo por intentos. La sesion deja de ser estado de servidor
mutable: el contexto de tenant/usuario viaja en el token firmado, resuelto por
middleware. El cambio de clave temporal se modela como flag `MustChangePassword` en
el usuario. Recovery por token de un solo uso con expiracion. Todo el flujo queda
en `AdminAuditLog` (login ok/fallido, cambio de clave, MFA).

---

## 2. PermissionsManager -> policies .NET

### Origen
`Funciones\Seguridad\PermissionsManager.vb`; cada `Page_Load` llama
`PermManager.ValidaPermiso(usuario, empresa, permiso, modulo)`. Combina 3 fuentes:
(1) **10 flags binarios fijos** en `USUARIO` (`FLAG_ADM`, `FLAG_ADTEC`, `FLAG_COMER`,
`FLAG_FDEF`, `FLAG_FTEC`, `FLAG_JMON`, `FLAG_MON`, `FLAG_OPER`, `FLAG_TEC`,
`FLAG_THUM`); (2) `PERMISO_CARGO` (matriz cargo-modulo-permiso, expandida por UDF
`fn_ConsultaCargo`); (3) `GEN_PARAMETROS` por (sucursal, modulo). Permisos observados:
`PERMITE_ASIGNAR`, `PERMITE_VER_TODAS_LAS_ACTIVIDADES`, `PERMITE_VER_MIS_PROYECTOS`,
`PERMITE_VER_PROYECTOS_TODOS`, `SOLO_PROYECTOS_DE_ORGANIZACION`,
`VER_FILTRO_PROYECTOS`, `PROCESOS_USUARIOS` (ACL por nodo BPMN), `PERMISOS`.
Riesgos: 10 flags fijos son anti-patron para roles dinamicos; la UDF es opaca al
codigo .NET.

### Destino .NET 10
`PermissionsManager` se porta a **policies de autorizacion .NET** con
`IAuthorizationPolicyProvider` dinamico: cada permiso (`PERMITE_ASIGNAR`,
`PERMITE_VER_PROYECTOS_TODOS`, ...) es una **policy** evaluada por un
`AuthorizationHandler` contra los claims del usuario y el tenant activo. Los 10
flags fijos se sustituyen por **roles/permisos dinamicos** (RBAC persistido en el
Module Registry), y `PERMISO_CARGO` por asignacion rol->permiso; la UDF
`fn_ConsultaCargo` se porta a una consulta LINQ/vista sobre `IEcorexDbContext`. Las
pantallas Blazor usan `[Authorize(Policy="...")]` y los endpoints repiten la
verificacion (defensa de servidor, no solo ocultar en UI). Los permisos efectivos se
cachean por (tenant, usuario) en Redis, invalidados por evento al cambiar roles. La
ACL por nodo BPMN (`PROCESOS_USUARIOS`) se integra como policy contextual del
WorkflowEngine.

---

## 3. SignalR - tiempo real

### Origen
`NEWFRONT_admtareas.aspx.vb:1` importa `Microsoft.AspNet.SignalR`; modulo
Notificaciones `000288`. Refresca grids al asignar tareas, muestra badges "Tienes N
tareas", emite alertas y posible chat interno. Hub probablemente en
`App_Start\Startup.vb` o `Global.asax` (no leidos). Riesgos: **single-instance sin
Redis backplane** (no escala horizontal), sin autenticacion de canales documentada,
sin aislamiento multi-tenant confirmado (un usuario podria escuchar eventos de otro
tenant).

### Destino .NET 10
**Hub nativo de ASP.NET Core** (`NotificationHub`) con autenticacion **JWT en el
handshake**. Cada conexion se une a un **grupo por tenant** (`tenant:{TenantId}`) y a
su grupo de usuario (`user:{UserId}`), garantizando que nunca se filtren eventos
entre empresas. **Redis backplane** para escalado horizontal. Los eventos de dominio
(`task.assigned`, `task.state.changed`, `announcement.created`) se publican por
RabbitMQ/MassTransit y un consumidor los reenvia al grupo de tenant; los tableros
Kanban y las campanas se actualizan sin recargar. Reconexion automatica con
heartbeat del cliente Blazor. Ver seccion 5 del [[Shell del sistema - Master + Controles compartidos]].

---

## 4. Integracion OpenAI / ChatGPT

### Origen
`Funciones\ChatGPT\clChatGPT.vb` (+ `clGPTPerTasks.vb`). `Consultar(prompt) As
String` invoca OpenAI Completions; la instancia se guarda en `Session("CHATGPT")`
para contexto. Usado por `cl_ia_reglas_formularios.vb` (verbo `GENERAR_TABLAS_IA`),
por Flujos y por agentes IA (`000867`). Riesgos: **API key en `Datos\Config.xml`
cifrado con TripleDES y clave hardcodeada** (`CargaConfig.vb:7`); sin cuota ni rate
limiting; sin fallback si OpenAI cae; `topBarBotGPT` deshabilitado.

### Destino .NET 10
Servicio `IAiAssistantService` inyectable, con **secretos en Key Vault /
DataProtection** (nunca clave hardcodeada). Cliente tipado con **Polly** (reintentos,
circuit breaker, fallback), **cuota y rate limiting por tenant** y registro de
consumo/costo. El contexto de conversacion deja `Session` y pasa a un store por
usuario/hilo. Los verbos IA del RulesEngine (`GENERAR_TABLAS_IA`, LLENAR DATOS CON
IA) invocan este servicio de forma tipada. El asistente de topbar (`topBarBotGPT`) se
retoma como `<AssistantWidget>` con el contexto del modulo activo. Prompts
versionados y auditables, no hardcodeados.

---

## 5. Emails / SendGrid

### Origen
Proyectos C# `MailSendGrid\` y `SendGridMail\`; wrapper VB
`Funciones\Mail\MailService.vb`; notificaciones `Funciones\Notificaciones\mail_notificaciones.vb`.
Envia mails al asignar tareas, copias de formularios respondidos y avisos de
aprobacion/rechazo. Riesgos: API key en config cifrado; sin bounces/DKIM
documentado; cuerpos de mail hardcodeados o en `GEN_PLANTILLAS_IMPRESION`, sin
versionar.

### Destino .NET 10
Servicio `IEmailService` con proveedor SendGrid (u otro) detras de la interfaz,
**API key en Key Vault**. Envio **asincrono por RabbitMQ/MassTransit** (encolar y
que un worker despache), con reintentos y dead-letter. **Plantillas versionadas**
(Razor/Liquid) por tenant, no cuerpos hardcodeados. Manejo de **bounces y webhooks**
de SendGrid (verificados por firma), DKIM/SPF por dominio. Cada envio queda trazado
con `CorrelationId` y auditado.

---

## 6. Azure Blob Storage / Computer Vision / OCR

### Origen
`Funciones\AzureBlobStorage\` (Blob + DTO + Database), `Funciones\AzureService\`
(`OCRAzure.vb`, `ComputerVision.vb`), `Funciones\OCRLocal\OCRLocal.vb`,
`Funciones\TesseractEngine\TesseractEngineHelp.vb`,
`Funciones\AmazonWebServices\AWSTextract.vb`. Suben firma/foto/GPS/imagen de
formulario a Blob; OCR de facturas; reconocimiento facial. Riesgos: credenciales de
3 proveedores sin rotacion; sin fallback; costos sin monitoreo; **firma escaneada
como `image` en `USUARIO.F_FIRMA`** (deprecated).

### Destino .NET 10
Abstraccion `IObjectStorage` (Azure Blob / S3 / MinIO detras de la interfaz), con
**credenciales en Key Vault y rotacion**. Los binarios (firmas, fotos, adjuntos de
formularios) salen de la BD SQL y viven en object storage con URLs firmadas de corta
vida; `USUARIO.F_FIRMA` migra a storage externo referenciado por clave. OCR/Vision
tras `IDocumentIntelligenceService` con proveedor configurable y fallback. Monitoreo
de costo/uso por tenant. Subidas validadas (tipo, tamano, antivirus) antes de
persistir.

---

## 7. Global.asax y manejo de errores global

### Origen
`Bootstrap\Global.asax` (no leido). Probable `Application_Error` que loggea a
`GEN_LOG` (13 cols), muestra pagina de error y redirige a login si la sesion expiro.
Riesgo: un error no manejado puede exponer stack trace; sin logging estructurado, la
auditoria depende de leer `GEN_LOG` a mano.

### Destino .NET 10
`Global.asax` desaparece: su rol lo asumen **middleware de ASP.NET Core**. Un
`ExceptionHandlingMiddleware` (o `UseExceptionHandler` + `ProblemDetails`) captura
todo, devuelve una respuesta controlada (nunca stack trace al usuario) y registra con
**Serilog** estructurado + **OpenTelemetry** (trazas, `CorrelationId`). Health checks
de BD (ambos motores), Redis y RabbitMQ. La expiracion de sesion la maneja la
validacion del JWT, no un redirect manual.

---

## 8. `FORX_DATA_FLUJO` - vinculo Formularios <-> Flujos

### Origen
Tabla SQL (~8 columnas) sin ficha propia. Cuando un formulario se responde como
parte de un paso BPMN, ademas de `FORX_DATA` se registra en `FORX_DATA_FLUJO`:
`PROCESO` (codigo del flujo), `ID_ELEMENTO` (nodo BPMN), `ID_CASO` (caso), estado del
formulario en el paso. Es la **union EAV + BPMN**, clave para reportes de "cuanto
tarda el paso X" o "que respuestas se dieron en el nodo Y".

### Destino .NET 10
Relacion N-N modelada con **FKs declaradas** en EF Core entre `FormSubmission`,
`WorkflowInstance` y `WorkflowNode`, todas `ITenantScoped`. Las respuestas EAV de
`FORX_DATA` migran a columnas **`jsonb` (Postgres) / `nvarchar(max)` (SQL Server)**
via el DAL dual; el vinculo formulario-nodo-caso queda como entidad tipada con
indices por (tenant, proceso, nodo). El ETL valida campo por campo con tests de
round-trip (riesgo declarado en [[Visión y entorno]] seccion 17). Ver
[[00 - Visión Formularios]] y [[00 - Visión Flujos]].

---

## 9. Reportes / exportaciones

### Origen
Modulo `000119` (`newfront_reportes_admin.aspx`); `Funciones\Excel\epplusExport.vb`
+ `FastExportingMethod.vb`; `Funciones\Pdf\cHtmltoPDF.vb` + `PdftoHtml.vb`;
`HtmlAgilityPack`. Genera reportes SSRS, exporta a Excel (EPPlus) y PDFs. Riesgos:
SSRS opaco al codigo (se edita en Report Builder externo); **EPPlus con licencia
comercial** por verificar.

### Destino .NET 10
Servicio `IReportService` con generacion server-side: Excel via **ClosedXML o EPPlus
con licencia validada**, PDF via **QuestPDF** (u otro con licencia clara). Reportes
como consultas parametrizadas sobre `IEcorexDbContext` (nunca SQL concatenado),
respetando el filtro de tenant. Exportaciones pesadas se encolan como **jobs
asincronos** (RabbitMQ/Hangfire) que dejan el archivo en object storage con URL
firmada. Se sustituye SSRS por plantillas versionadas en codigo.

---

## 10. Deployment / IIS / entornos

### Origen
`Web.config` (NO tocar - regla del proyecto), `App\Datos\Config.xml` (TripleDES con
clave hardcodeada en `CargaConfig.vb:7`), `alias.xml` (texto plano), `Empresa.xml`.
Despliegue por Web Deploy a IIS/Contabo (ver perfiles `.pubxml`). Riesgos: cambiar de
entorno = editar y republicar 3 XML; clave TripleDES hardcodeada; sin CI/CD
documentado.

### Destino .NET 10
Configuracion por **`appsettings.{Environment}.json` + variables de entorno**;
**secretos en Key Vault / DataProtection**, nunca cifrado con clave hardcodeada ni
literales de entorno en codigo. `Database:Provider` elige Postgres o SQL Server sin
recompilar (DAL dual). Despliegue **dockerizado** (compose: api, postgres/sqlserver,
redis, rabbitmq), estrategia hibrida VPS + Railway, listo para Kubernetes. **CI/CD**
que corre tests en AMBOS motores por cada PR (ver [[Visión y entorno]] seccion 15 y
[[Deploy IIS y perfiles de publicacion]] para el estado actual).

---

## 11. Auditoria consolidada -> `AdminAuditLog` inmutable

### Origen
No existe vista unificada. Los eventos viven dispersos: `GEN_LOG` (13 cols, acciones
de usuario), `CONTROL_REGLAS_H` (9 cols, ejecucion de reglas), `TAR_SEGUIMIENTO_PROCESO`
(31 cols, avance de casos BPMN), `REGLAS_HIS` (6 cols, historial legacy obsoleto).
Sin correlacion entre ellos; auditar exige cruzar 4 tablas a mano.

### Destino .NET 10
**`AdminAuditLog` unico e inmutable** (append-only) que consolida todo evento
sensible: login/logout, cambios de permisos, acciones de pagina, avance de flujos,
ejecucion de reglas, cambios de configuracion. Cada registro lleva `TenantId`,
`UserId`, `CorrelationId`, timestamp UTC, accion, entidad y snapshot del cambio.
**Particionado por fecha**, protegido contra edicion/borrado (solo insert; RLS y
permisos de BD lo blindan). Escritura desde un `IAuditService` transversal (behavior
de MediatR) para que ningun comando de escritura escape a la auditoria. Las tablas
legacy alimentan el ETL historico; el sistema vivo escribe solo a `AdminAuditLog`.

---

## Sintesis: como el destino cierra cada punto ciego

| # | Area transversal | Origen (riesgo) | Destino .NET 10 |
|---|---|---|---|
| 1 | Login | sin MFA, hash sin auditar | JWT + MFA + Argon2 + rate limiting |
| 2 | PermissionsManager | 10 flags fijos, UDF opaca | Policies .NET dinamicas + RBAC + cache Redis |
| 3 | SignalR | single-instance, sin aislamiento | Hub nativo + grupos por tenant + Redis backplane |
| 4 | ChatGPT | key hardcodeada, sin cuota | Key Vault + Polly + cuota por tenant |
| 5 | SendGrid | plantillas hardcodeadas | IEmailService async + plantillas versionadas + bounces |
| 6 | Azure Blob/OCR | firma como image en BD, creds fijas | IObjectStorage + Key Vault + URLs firmadas |
| 7 | Global.asax | stack trace expuesto | Middleware + Serilog + OpenTelemetry + ProblemDetails |
| 8 | FORX_DATA_FLUJO | N-N sin FKs | FKs EF Core + jsonb/nvarchar + ETL validado |
| 9 | Reportes | SSRS opaco, licencia EPPlus | IReportService + QuestPDF + jobs async |
| 10 | Deploy | 3 XML + clave TripleDES fija | appsettings + Key Vault + Docker + CI dual |
| 11 | Auditoria | 4 tablas dispersas | AdminAuditLog inmutable particionado |

---

## Relacion con otras notas

- [[Visión y entorno]] - arquitectura, seguridad (14), deploy (15), riesgos (17)
- [[Shell del sistema - Master + Controles compartidos]] - login, permisos, SignalR y auditoria en el shell
- [[Visión y entorno|Prototipo Final ECOREX]] - aspecto destino
- [[Gestion de Empresas - Admin multi-tenant]] - multi-tenant real y correccion de errores heredados
- [[00 - Visión Formularios]] / [[00 - Visión Flujos]] - EAV y BPMN (union `FORX_DATA_FLUJO`)

---

## Este documento sigue siendo un mapa, no un TODO

Cada seccion tiene el plano del origen (que preservar en el ETL) y la resolucion
destino (como se implementa en .NET 10). La lista fina de tareas vive en cada nota
especifica. Ver [[00 - INDICE|INDICE maestro]] para el contexto completo.
