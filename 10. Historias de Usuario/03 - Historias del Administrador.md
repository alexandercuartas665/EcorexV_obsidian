---
tipo: historias-usuario
protagonista: Julia (Administradora del Sistema)
modulos_cubiertos: [Usuarios 000073, Roles 000194, Dependencias 000850, Config Entidad 000615, Plantillas 000893]
---

# Historias del Administrador - Julia

> **Julia** es la persona menos glamurosa del equipo pero la mas critica: sin
> ella nadie tiene login, ni permisos, ni firma en las cotizaciones. Es la
> guardiana de los datos maestros de SKY SYSTEM. Es **Administradora del tenant**
> (no PlatformAdmin): gobierna usuarios, roles y organigrama DENTRO de su empresa,
> nunca planes ni otros tenants. Trabaja en el sistema destino .NET 10 con el
> aspecto del [[Visión y entorno|Prototipo Final ECOREX]]; vision en [[Visión y entorno]].

---

## Escenario 1 - Dar de alta un usuario nuevo

### User Story
**Como** administradora del sistema,
**quiero** crear el usuario para un empleado que ingresa hoy,
**para que** pueda entrar al sistema y ver solo lo que le corresponde por su cargo.

### Precondiciones
- RRHH le envio: nombre, cedula, cargo, email, dependencia
- El cargo ya esta creado en el sistema (sino, primero crea el cargo)

### Pasos concretos
1. Va a **Usuarios** (`000073`) en Configuracion del sidebar SKY SYSTEM.
2. Click "+ Nuevo usuario".
3. Llena:
   - Nombre: `MARIA.G`
   - ID (cedula): `1086137278`
   - Email: `[email protected]`
   - Cargo: `Analista Comercial`
   - Dependencia: `Ventas`
   - Tenant: `SKY SYSTEM` (fijo - Julia solo administra su empresa)
   - Es proveedor externo (3DEV): **NO**
4. Click "Guardar".
5. El sistema envia un correo de activacion; al primer login el usuario define su clave.

### Criterios de aceptacion
- El usuario se crea con el `tenant_id` de SKY SYSTEM (nunca visible desde otro tenant)
- Se le asignan los **roles** por defecto de su cargo (tabla `user_role`)
- Al primer login, el flujo de activacion fuerza definir clave
- Recibe las **policies** por defecto de su cargo via el PermissionsManager

### Mejora sobre el origen (corregida en destino)
El sistema legacy usaba **10 flags binarios** en la tabla USUARIO (`FLAG_ADM,
FLAG_ADTEC, FLAG_COMER, ...`) como roles fijos. El destino los reemplaza por un
sistema de **roles dinamicos** (`user_role`) traducidos a policies .NET, mas
flexible y auditable.

---

## Escenario 2 - Cambiar los permisos de un cargo

### User Story
**Como** administradora,
**quiero** ajustar que puede hacer el cargo "Analista Comercial",
**para que** todos los analistas hereden ese cambio sin editar usuario por usuario.

### Pasos concretos
1. Va a **Roles y Permisos** (`000194`).
2. Ve la **matriz de permisos**: filas = cargos, columnas = permisos (policies).
3. Marca la celda `Analista Comercial × PERMITE_VER_MIS_PROYECTOS = SI`.
4. Guarda.

### Que pasa por detras
Se crea/actualiza el permiso del cargo dentro del tenant. En runtime, el
**PermissionsManager** evalua la policy `PERMITE_VER_MIS_PROYECTOS` para ese usuario
y tenant antes de servir la vista; las policies se cachean en Redis para no
recalcularlas en cada request.

### Criterios de aceptacion
- Todos los usuarios con ese cargo obtienen el permiso al refrescar sus claims
- El cambio queda aislado al tenant SKY SYSTEM
- La lista de usuarios del cargo se actualiza para las asignaciones de tarea

---

## Escenario 3 - Configurar los permisos de un nodo BPMN especifico

### User Story
**Como** administradora,
**quiero** decir "en el nodo 'Aprobar Cotizacion' del proceso 00001, participan los cargos 583 (Comprador) y 511 (Analista)",
**para que** cuando el flujo llegue a ese paso, el sistema sepa a quien mostrar la tarea.

### Pasos concretos
Esto es **diferente** al escenario anterior. Los permisos por nodo BPMN
NO se configuran en Roles y Permisos, sino desde **Flujos del proceso** (el editor
BPMN que usa Andres sobre el WorkflowEngine).

Julia interviene solo si Andres no tiene acceso al editor.

### Detalle tecnico
Los permisos por nodo se guardan en `PERMISO_CARGO` con:
```
CARGO           = <numero del cargo>
MODULO          = 'PROCESOS_USUARIOS'
REFERENCIA      = ''
REFERENCIA2     = <codigo del proceso, ej 00024>
REFERENCIA3     = <ID_ELEMENTO del nodo, ej Activity-0gyhzby>
```

Ejemplos reales vistos en `db3dev`:
```
CARGO=583  REFERENCIA2=00024  REFERENCIA3=Activity-0gyhzby
CARGO=583  REFERENCIA2=00024  REFERENCIA3=Gateway-1y2uon5
```

Ver detalle en [[Ejecucion - SiguienteEstado y Reinicios#7. Esquema de PERMISO_CARGO]].

---

## Escenario 4 - Reorganizar el organigrama

### User Story
**Como** administradora,
**quiero** cambiar la estructura organizacional cuando se crea un nuevo departamento,
**para que** los reportes jerarquicos y las asignaciones de tarea reflejen la realidad.

### Pasos concretos
1. Va a **Dependencias** (`000850`), el organigrama del tenant (Org Service).
2. Ve el organigrama tal como en el prototipo: KPIs (Dependencias, Usuarios, Areas),
   arbol `ECOREX S.A.S > Gerencia General > Comercial / Operaciones / Tecnologia`
   con conteo de usuarios por nodo, y panel de detalle de la dependencia
   seleccionada (tipo Area, estado Activa, codigo `DEP-100`).
3. Pulsa **"Nueva dependencia"** y crea "Investigacion y Desarrollo" bajo "Direccion Tecnica".
4. Reasigna algunos usuarios a la nueva dependencia.

### Criterios de aceptacion
- Se persiste como nodo del arbol con su padre, ligado al `tenant_id`
- Los KPIs y reportes de estructura ven la dependencia de inmediato
- Alimenta la asignacion de tareas y los permisos por area

---

## Escenario 5 - Actualizar el logo y datos de la empresa

### User Story
**Como** administradora,
**quiero** cambiar el logo, NIT y direccion cuando la empresa cambia de marca,
**para que** todos los documentos generados (cotizaciones, facturas) usen los datos nuevos.

### Pasos concretos
1. Va a **Configuracion de Entidad** (`000615`).
2. Edita:
   - Nombre: `SKY SYSTEM S.A.S`
   - NIT: `900123456-7`
   - Direccion: `Cali, Valle del Cauca`
   - Logo (sube archivo o pega URL)
3. Ajusta las **variables por tenant** que aplican (ej. plantilla del asunto de mail).
4. Guarda.

### Que pasa por detras
Los datos se guardan como parametros del tenant (variables por tenant del Module
Registry). El layout Blazor de SKY SYSTEM los lee para pintar la marca del sidebar
(`SKY SYSTEM / Plan Empresa - ECOREX`) y el titulo de las paginas.

### Criterios de aceptacion
- El logo y la marca aparecen en el sidebar de todos los usuarios del tenant al recargar
- Las plantillas de impresion usan el nuevo NIT
- El cambio no afecta a ningun otro tenant

---

## Escenario 6 - Configurar plantillas de impresion

### User Story
**Como** administradora,
**quiero** editar la plantilla de cotizacion cuando marketing rediseña el layout,
**para que** las cotizaciones que salen del sistema tengan la imagen corporativa.

### Pasos concretos
1. Va a **Plantillas de Impresion** (000893).
2. Ve la lista de plantillas: `COTIZ-01`, `FAC-01`, `REQ-01`, etc.
3. Click editar en `COTIZ-01`.
4. En el editor sube nuevo PDF/DOCX template.
5. Marca variables `@@nombre_cliente@@`, `@@fecha@@`, `@@total@@`.
6. Guarda.

### Criterios de aceptacion
- La plantilla se persiste como plantilla del tenant
- Cuando el **RulesEngine** ejecuta el verbo `IMPRIMIR_PLANTILLA` (documento 00008), usa esta version nueva
- El PDF generado abre en Adobe Reader sin errores

---

## Escenario 7 - Auditar quien vio o cambio que

### User Story
**Como** administradora,
**quiero** ver un log de cambios y ejecuciones,
**para que** pueda demostrar cumplimiento en auditorias externas.

### Como lo resuelve el sistema destino
El destino unifica la auditoria en un **AdminAuditLog inmutable** por tenant. Julia
entra a **Auditoria General** y ve, en una sola vista filtrable, todo lo que en el
origen estaba disperso:
- Ejecuciones de reglas (usuario, fecha, duracion, error) del RulesEngine
- Acciones de usuario (el antiguo `GEN_LOG`)
- Creacion/modificacion de tareas y avances de flujo (eventos `task.state.changed`)
- Cambios en usuarios, roles y policies

### Como esta construido
Cada servicio emite eventos de dominio al Event Bus; un worker los materializa en el
`AdminAuditLog`, que es de solo anexado (append-only) y esta filtrado por
`tenant_id`. Julia exporta el rango que un auditor externo pida sin salir de la
consola del tenant. Trazabilidad distribuida (OpenTelemetry, correlation IDs)
respalda cada entrada.

---

## Escenario 8 - Restringir el acceso de un usuario mientras esta de vacaciones

### User Story
**Como** administradora,
**quiero** desactivar temporalmente el acceso de un usuario sin borrarlo,
**para que** las tareas queden reasignadas sin perder el historial de sus casos previos.

### Pasos concretos
1. Va a **Usuarios**.
2. Encuentra a `JUAN.B` y hace click en Editar.
3. Marca el usuario como **Inactivo**.
4. Guarda.
5. Va a **Gestor de tareas / Tableros**, filtra por `Usuario = JUAN.B`.
6. Reasigna las pendientes arrastrandolas a otro responsable.

### Criterios de aceptacion
- JUAN.B no puede loguearse mientras esta inactivo (su JWT/refresh queda invalidado)
- Sus pasos ya ejecutados siguen registrados con su ejecutor
- Al reactivarlo, vuelve a ver el sistema sin perder historial

---

## Recap de los modulos que Julia usa

```
SISTEMA / General
  +-- Configuracion de Entidad     000615  <- logo, NIT
  +-- Plantillas                   000893  <- cotizacion, factura
  +-- Dependencias                 000850  <- organigrama
  +-- Roles y Permisos             000194  <- matriz cargo x permiso
  +-- Administracion de Usuarios   000073  <- alta/baja

SISTEMA / CRM (parametrizacion)
  +-- Perfiles de Cliente/Proveedor 000166
  +-- Tipos de Empresas             000231
  +-- Vendedores                    000124

Desarrollo
  +-- Tipos de Documento            000136  <- consecutivos
```

---

## Lo que Julia NO toca

- Editar flujos BPMN
- Crear formularios
- Configurar reglas
- Gobernar planes, limites o habilitar/deshabilitar tenants (eso es del **PlatformAdmin**, en la consola de plataforma fuera de SKY SYSTEM)

---

## Enlaces a documentacion tecnica

- Aspecto y navegacion destino: [[Visión y entorno|Prototipo Final ECOREX]]
- Vision maestra del sistema: [[Visión y entorno]]
- Multi-tenant real y correccion de errores: [[Gestion de Empresas - Admin multi-tenant]]
- Manejo de datos y permisos: [[Manejo de Datos - Alias, parametros, UDFs, consecutivos]]
- ER logico multi-tenant: [[Modelo Entidad-Relacion logico]]
