---
tipo: backlog
proposito: Registro vivo de lo PENDIENTE y las deudas tecnicas del sistema destino ECOREX Tareas (.NET 10), para no perder el hilo entre sesiones
estado: vivo
fecha_ultima_actualizacion: 2026-07-07
---

# Pendientes y deudas tecnicas

> Backlog transversal. Complementa la [[HOJA DE RUTA DESARROLLO]] (el plan) y el
> tracker de construccion de [[INVENTARIO GENERAL]] seccion 0.1 (que esta CONSTRUIDO).
> Las deudas por modulo tambien viven en su ADR en `docs/decisiones/` del repo.
> Rama de trabajo: `fase-0/clon-backbone`. Convencion del vault: solo ASCII.

## 1. Deudas de lo recien construido (accionable ya)

### Roles / Enforcement (ADR-0032 / ADR-0033)
- [ ] **Wiring del resto de paginas** a la policy de permiso `Perm:{route}:View`.
      Hoy solo estan cableados 3 modulos (inventario, usuarios, roles); el resto
      (tareas, flujos, formularios, reglas, conceptos, etc.) es wiring mecanico.
- [ ] Botones Crear/Editar/Eliminar gateados solo en esos 3 modulos; extender.
- [ ] Poblar el claim `Permissions` del token (hoy la consola resuelve por
      servicio `ICurrentPermissions`) y retirar las policies clasicas ya sin uso.

### Administracion de usuarios (ADR-0031)
- [ ] Invitacion por **correo real** (el `InvitationToken` de TenantUser existe
      pero no se explota) + self-service de cambio de clave del propio usuario.
- [ ] Salvaguarda "no dejar el tenant sin ultimo Owner".

### Menu configurable / Administrador de Menu (ADR-0030)
- [ ] Policy "paso 2" (restringir a Owner/Admin) - parcialmente cubierto por el
      enforcement de roles.
- [ ] Ruta/slug de Seccion/Subgrupo no editable en el panel de propiedades.
- [ ] "Guardar" del editor es confirmacion (cada accion persiste al vuelo).

## 2. Deudas de modulos anteriores
- **Inventario (ADR-0027):** imagenes por URL (sin upload de archivo); sin
  kardex de movimientos de stock.
- **Plantillas WhatsApp (ADR-0029):** Submit/SyncStatus son STUBS (sin
  integracion real con la WhatsApp Cloud API de Meta); header media
  (imagen/documento/video) modelado pero no editable.
- **Extraccion de datos / web scraping (ADR-0025):** scheduler (corridas
  programadas, ira en Ecorex.Workers); robots.txt + rate-limit por dominio;
  mitigacion de DNS rebinding; extraccion multipaso con credenciales cifradas.
- **Ficha de empresa (ADR-0026):** 9 secciones placeholder (los flujos SQL
  peligrosos del legacy que a proposito NO se reconstruyen; ver los 9 errores).
- **AI Gateway / Wompi:** `IWompiApiClient` es stub (tokenizacion de tarjeta +
  debito recurrente servidor-a-servidor); "Validar conexion" (IA y Wompi) es
  validacion estructural; modelos de Gemini/ChatGPT/DeepSeek por refrescar
  (Claude ya al dia).

## 3. Modulos aun sin construir (stubs `modulo/...` en el menu)
- **Sistema . CRM:** conceptos actividades (000125), estados CRM (000272),
  perfiles cliente/prov (000166), servicios/productos (000249), tipos de
  empresas (000231), vendedores (000124), origen clientes (000324), grupos de
  actividades (000126).
- **Sistema . Desarrollo:** autocompletado formularios (000801), notificaciones
  (000288), objetos del sistema (000137), parametros XML (000057), servicios
  web (000053), SQL Admin (000077), tipos de documentos/consecutivos (000136).
- **Sistema . Actividades:** prioridades (000621), tipos de proyecto (000690),
  estados (000653).
- **Negocio:** creacion de clientes (000232), seguimiento de clientes (000123).
- **Pendientes del tracker:** Programar actividad (000889), Power BI Service (000788).

## 4. Fases mayores de la hoja de ruta
- [ ] **FASE 6 - ETL desde `db3dev`** (tenant 01 = BITCODE): la pieza grande.
      REQUIERE autorizacion del usuario para la conexion a db3dev (solo lectura;
      NO esta en el repo). Ver [[Conexion a la BD del sistema actual (db3dev)]].
- [ ] Primera corrida del **plan de validacion completo** (atada a FASE 6).
      Ver [[Plan de validacion del sistema]] y [[00 - Registro de corridas]].

## 5. Abiertas de proceso
- [ ] Fixes de `DynamicFormRenderer` (SemaphoreSlim disposed + carrera de
      DbContext) lanzados en sesiones aparte: integrar y re-correr el E2E de
      formularios.
- [ ] **Merge a `main`**: todo va en `fase-0/clon-backbone` (push directo a main
      bloqueado); el merge es decision del usuario en GitHub.
- [ ] Verificar el CI de GitHub Actions (no verificable desde la maquina de dev;
      falta `gh`).
- Nota: `qa.usuario@sky-system.local` quedo con el rol demo "Asesor limitado"
  como caso visible del filtrado por permisos (revertible).

## 6. Foco siguiente
- **Flujos de trabajo (WorkflowEngine, modulo 000291).** El motor y el editor
  canvas ya existen (ADR-0014 / ADR-0022); la proxima sesion trabaja sobre
  flujos. Ver [[00 - Vision Flujos]] y [[AdmWorkflow - Motor de ejecucion]].

---

*Documento vivo. Marcar/quitar items al cerrarlos; cada deuda nueva de un modulo
va tambien en su ADR.*
