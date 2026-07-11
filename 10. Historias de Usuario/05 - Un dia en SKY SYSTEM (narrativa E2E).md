---
tipo: narrativa-end-to-end
proposito: contar el sistema como una historia cronologica que toca todos los modulos
duracion_de_lectura: 12 minutos
---

# Un dia en SKY SYSTEM - narrativa end-to-end

> Esta es la historia real de un pedido de un cliente, contada desde las 8AM
> hasta que se cierra la tarea al final del dia, en el **sistema destino .NET 10**.
> Atraviesa los modulos del prototipo final y muestra como se conectan en la
> practica. Aspecto en [[Visión y entorno|Prototipo Final ECOREX]]; vision en [[Visión y entorno]].

---

## 8:03 AM - Guadalupe abre el sistema

Guadalupe llega a la oficina, abre el navegador y va a `app.ecorex.io`.

Se autentica y entra a su tenant. Su JWT lleva `tenant_id` de **SKY SYSTEM**, asi que
desde el primer request todo queda filtrado a su empresa.

**> Entrada: Auth & Tenant Resolver (JWT + middleware de tenant)**

Aparece el layout del prototipo: rail de iconos a la izquierda, sidebar contextual
`SKY SYSTEM / Plan Empresa - ECOREX` con los bloques **PRINCIPAL** (Inicio, Anuncios,
Gestor de tareas, Configuracion) y **MODULOS**, y al pie su ficha
(**Guadalupe Lopez - Administradora**).

Cae en **Inicio / Resumen**: el dashboard "Buenos dias, Guadalupe", con el resumen
`8 tareas y 3 alertas` y las 4 tarjetas KPI (Tareas activas, Proyectos en curso,
Flujos ejecutandose, Alertas urgentes). El badge del icono "Gestor de tareas" dice **8**.

**> Modulo tocado: Inicio / Resumen (dashboard KPI)**

---

## 8:15 AM - Llega un mail del cliente ALPES TECHNOLOGY

Guadalupe recibe un correo:

> "Buenos dias, necesitamos cotizacion urgente de 20 laptops Lenovo ThinkPad
> para el proyecto de infraestructura. Fecha limite: viernes.
> Gracias, Carlos - ALPES"

Ella pulsa el boton **"Nueva actividad"** del dashboard.

**> Modulo tocado: Crear actividad (000038)**

El panel carga:
- Combos de categorias (actividades `TIPO_TAR`) del tenant, servidos por el Task Service
- La plantilla de captura configurada como variable del tenant

Guadalupe llena:
- Categoria: `32 - Cotizacion Comercial`
- Subcategoria: `00477 - Requerimientos infraestructura`
- Titulo: `Cotizacion 20 laptops Lenovo - ALPES TECHNOLOGY`
- Prioridad: `ALTA`
- Encargado: (lo deja vacio - luego lo asigna en el Kanban)
- Fecha entrega: `viernes`

Click **"Crear tarea"**.

---

## 8:16 AM - Detras del telon: BPMN y WorkflowEngine

Cuando Guadalupe da click, un solo comando CQRS corre en una transaccion y pasan
estas cosas invisibles:

**1.** Se crea la tarea (`TaskItem`) con su `tenant_id` de SKY SYSTEM, titulo, fecha y actividad. Un consecutivo del tenant genera el ID `T0042`.

**2.** El **WorkflowEngine** (portado de `AdmWorkflow`) resuelve que flujo aplica a (categoria, subcategoria):
   - Encuentra el proceso `00001` (COMERCIAL REQUERIMIENTO INFRAESTRUCTURA), cacheado en Redis
   - La definicion tiene **42 nodos BPMN**
   - Crea la `workflow_instance` y precrea sus 42 pasos como pendientes
   - Resuelve los usuarios validos por nodo via las policies del PermissionsManager (permisos por cargo)
   - Marca el StartEvent (`Event_09gcxpb "Requerimiento"`) como nodo activo

**3.** El motor NO avanza reglas autonomas todavia porque el caso recien se creo (respeta el limite de iteraciones para nunca ciclar).

**4.** Se emite el evento `task.created` al Event Bus; el **Notification Service (SignalR)** empuja la notificacion al canal de la Coordinadora Operativa.

**> Componentes tocados: Crear actividad (000038), WorkflowEngine, definicion de flujo (Redis), PermissionsManager, Notification Service (SignalR)**

---

## 8:17 AM - Guadalupe asigna la tarea

Toast en pantalla: "Tarea T0042 creada. Proceso BPMN instanciado con 42 pasos".

Va a **Gestor de tareas / Tableros** en el sidebar y abre el tablero del proyecto.

**> Modulo tocado: Gestor de tareas / Tableros (Kanban)**

Ve las 4 columnas `Por hacer | En progreso | En revision | Completado`. La tarea T0042
aparece en **"Por hacer"**.

Arrastra la tarjeta a **"En progreso"** y la asigna a **Andres**.

Detras: un comando CQRS actualiza el responsable y el estado de la tarea (con
concurrencia optimista) y emite el evento; SignalR notifica a Andres en vivo.

---

## 8:20 AM - Andres recibe la notificacion

Andres esta al frente de su computador. En su topbar aparece un badge rojo con "1" en el icono de campana.

Click en la campana > ve "Nueva tarea asignada: T0042 - Cotizacion 20 laptops".

Click en la notificacion > lo lleva a su vista "Mis tareas" en **Gestor de tareas**.

**> Modulo tocado: Gestor de tareas / Tableros (Mis tareas)**

Ve la tarjeta con:
- Titulo, prioridad ALTA, fecha entrega viernes
- Acciones: "En progreso" | "Completar"

Click **"En progreso"** > el WorkflowEngine actualiza el estado del paso y registra la fecha de inicio.

---

## 8:30 AM - Andres decide contactar 3 proveedores externos

Andres piensa: "voy a mandarle la solicitud a Carlos de Lenovo, Diego de Dell y Miguel de HP".

Necesita un formulario publico. Va a **Formularios** (000131).

**> Modulo tocado: Formularios (000131) - DynamicFormRenderer**

Ve la lista de formularios. Selecciona `00031 SKY SYSTEM COMERCIAL CAPTURA REQUERIMIENTO`.

El formulario ya existe con 9 campos:
- Razon social
- NIT
- Fecha
- Tipo de requerimiento (Lista: Hardware/Software/Servicios/Mantenimiento)
- Productos requeridos (checkboxes)
- Descripcion detallada (textarea)
- Es cliente premium? (Si/No)
- Firma

Andres verifica que este bien y pulsa **"Generar link publico"** una vez por
proveedor. El Module Registry crea 3 tokens de acceso, cada uno ligado al formulario
`00031` y al `tenant_id` de SKY SYSTEM, con caducidad y opcion de revocacion.

**> Modulo tocado: Modulos web / Registro (000109) - tokens del tenant**

3 links generados:
```
/f/REDACTED... (Carlos - Lenovo)
/f/14646489845... (Diego - Dell)
/f/REDACTED... (Miguel - HP)
```

Los envia por mail a los 3 proveedores.

---

## 10:45 AM - Carlos (Lenovo) responde

Carlos recibio el mail hace 15 minutos. Abre el link.

**> Modulo tocado: Visor publico por token (`/f/{token}`, pagina Blazor aislada)**

El sistema:
1. Lee el token de la URL
2. Lo valida: existe, no vencido, no revocado, dentro del rate limit, y resuelve su `tenant_id` (SKY SYSTEM)
3. Del token obtiene que formulario cargar: `00031`
4. El **DynamicFormRenderer** lee la definicion EAV del formulario `00031` del tenant y construye la UI
5. Carlos nunca ve el shell del tenant ni nada de otra empresa

Carlos ve el formulario. Al escribir "ALPES TECHNOLOGY" en Razon Social...

**Se dispara la regla `AUTOCOMPLETAR_DATOS_CLIENTE`:**
- El renderer detecta evento `change` en el campo `razon_social`
- Resuelve que reglas aplican a esa pregunta
- Encuentra la regla (grupo INTELIGENCIA_ARTIFICIAL, verbo GENERAR_TABLAS_IA)
- Invoca al **RulesEngine**, que resuelve el verbo Ensamblado del registro tipado
- El handler llama al servicio de IA (`clChatGPT` portado) con "Dame NIT y direccion de ALPES TECHNOLOGY"
- GPT responde: `{"nit": "900.234.567-8", "direccion": "Bogota, calle 100"}`
- Los campos NIT y direccion se autocompletan en vivo
- Se registra en la auditoria del tenant: usuario=CARLOS, fecha, regla=00002, verbo=GENERAR_TABLAS_IA, duracion=1876ms

Carlos ve como los campos se llenan solos. **Impresionado.** Termina de llenar precio y firma con el mouse. Click **Enviar**.

**> Componentes tocados en un solo request: DynamicFormRenderer (000131), RulesEngine (000802), servicio de IA (000867), AdminAuditLog del tenant**

Las respuestas se guardan como respuesta EAV (`jsonb`/`nvarchar(max)`) del tenant.

---

## 11:15 AM - Andres cierra el paso 2 (Cotizacion a Proveedores)

Andres recibio 3 notificaciones de respuestas. Va a **Gestor de tareas / Tableros**, encuentra T0042 y abre su detalle.

Ve el **Seguimiento del Proceso**:

```
Paso 1: Requerimiento          [Completado]
Paso 2: Cotizacion Proveedores [En progreso]   <- Andres esta aqui
Paso 3: Aprobacion de Cliente  [Por hacer]
Paso 4: Generar Factura        [Por hacer]
...
```

Click en **"Marcar paso 2 como completo"**.

Detras el **WorkflowEngine** ejecuta, en una transaccion:

**1.** Completa el nodo `Activity_cot`: estado Completado, fecha de resolucion (UTC / zona del tenant), ejecutor `ANDRES.M`.

**2.** Avanza el flujo resolviendo el siguiente estado (con limite de iteraciones para nunca ciclar):
   - Iteracion 1: la transicion desde `Activity_cot` lleva al gateway `Gateway_aprob` (¿Aprueba?)
   - Como es un gateway sin usuario, evalua sus condiciones y busca la salida
   - Dispara las reglas autonomas del paso "Cotizacion" via el **RulesEngine**: un verbo consolida las 3 respuestas de proveedores y calcula el mejor precio (Lenovo gano por $2M menos)
   - El gateway toma la ruta "SI" (aprobado por el mas barato)
   - Iteracion 2: la salida "SI" lleva a `Activity_16toqtl` "Se aprueba compra por el cliente"
   - Marca ese nodo como activo y, via el PermissionsManager, resuelve el cargo Analista Comercial -> el responsable queda en `MARIA.G`
   - El loop termina porque el nuevo paso requiere accion humana (no es regla autonoma)

**3.** Se emite `task.state.changed`; SignalR notifica a Maria en vivo.

---

## 11:20 AM - Maria recibe la nueva tarea

Maria ve la notificacion en la campana. Click. Se abre su vista "Mis tareas" con la tarjeta:
- T0042 - Necesita aprobacion de cliente
- Prioridad: ALTA
- Responsable: MARIA.G

Ella tiene que confirmar con el cliente que la cotizacion Lenovo esta aprobada.

Llama al cliente. El cliente aprueba.

Maria click en la tarjeta > **Actualizar** > selecciona "Cliente aprueba" y agrega la nota "Cliente confirmo por telefono".

El sistema:
- Registra la aprobacion en el paso (aprobado por MARIA.G, con su nota)
- El WorkflowEngine avanza al paso 4 (Generar Factura), luego paso 5 (Entrega para gestion de pago), etc.

---

## 3:00 PM - Se dispara la regla de imprimir cotizacion PDF

El WorkflowEngine detecta que el paso "Generar Factura" tiene una regla autonoma
asociada: `IMPRIMIR_PLANTILLA` (verbo del documento 00008 PLANTILLAS DE IMPRESION).

Mientras avanza el flujo (respetando el limite de iteraciones), invoca al **RulesEngine**:
- Tipo: Ensamblado
- Verbo: `IMPRIMIR_PLANTILLA`

El handler:
1. Toma la plantilla del tenant `COTIZ-01`
2. Reemplaza `@@nombre_cliente@@`, `@@fecha@@`, `@@total@@` con los valores del caso
3. Genera el PDF (servicio de PDF portado)
4. Adjunta el PDF al Object Storage, ligado a la tarea `T0042` y al tenant
5. Registra la ejecucion en el AdminAuditLog

---

## 5:30 PM - Guadalupe revisa el dia

Guadalupe abre **Gestor de tareas / Tableros** y mira la columna **Completado** (o la vista Timeline del proyecto).

Ve el timeline con 12 tareas cerradas hoy. T0042 esta ahi con:
- Fecha cierre: hoy 5:15 PM
- Duracion total: 9 horas
- Responsable final: ANDRES.M
- Paso final: "Ingreso de la compra al sistema Alegra"

Tambien nota que el KPI "Flujos ejecutandose" bajo y "Alertas urgentes" quedo en 0.
Sonrie. Buen dia.

---

## 6:00 PM - Julia revisa la auditoria

Antes de irse, Julia (admin del tenant) abre **Auditoria General**.

El AdminAuditLog inmutable de SKY SYSTEM muestra 47 ejecuciones del dia:
- 12 ejecuciones de `GENERAR_TABLAS_IA` (autocompletados IA de proveedores)
- 8 de `PASAR_CAMPOS`
- 3 de `IMPRIMIR_PLANTILLA`
- 15 de consultas de consolidacion
- 9 de `ABRIR_MODAL`

Cada ejecucion muestra usuario, fecha, duracion, filas afectadas. Ninguna con error.
Todo aislado a su tenant. Cierra el navegador.

---

## Resumen de modulos tocados en esta narrativa

En este dia normal (un caso, 9 horas), el sistema toco estos modulos y componentes
del destino:

| Hora | Modulo / Componente | Codigo | Personaje |
|------|---------------------|--------|-----------|
| 8:03 | Auth & Tenant Resolver | (JWT) | Guadalupe |
| 8:03 | Inicio / Resumen (dashboard KPI) | - | Guadalupe |
| 8:15 | Crear actividad | 000038 | Guadalupe |
| 8:17 | Gestor de tareas / Tableros (Kanban) | - | Guadalupe |
| 8:20 | Gestor de tareas (Mis tareas) | - | Andres |
| 8:20 | Notification Service (SignalR) | 000288 | (sistema) |
| 8:30 | Formularios (DynamicFormRenderer) | 000131 | Andres |
| 8:30 | Modulos web / tokens del tenant | 000109 | Andres |
| 10:45 | Visor publico por token | /f/{token} | Carlos |
| 10:45 | RulesEngine | 000802 | (sistema) |
| 10:45 | Agentes IA | 000867 | (sistema) |
| 11:15 | WorkflowEngine (motor BPMN) | (interno) | (sistema) |
| 11:20 | Gestor de tareas (Mis tareas) | - | Maria |
| 3:00 | RulesEngine + Plantillas | 000802 + 000893 | (sistema) |
| 5:30 | Gestor de tareas (Timeline/Completado) | - | Guadalupe |
| 6:00 | Auditoria General (AdminAuditLog) | - | Julia |

Todo el caso vivio dentro del tenant **SKY SYSTEM**, aislado por el filtro global de
EF Core mas Row-Level Security en la BD (DAL dual PostgreSQL / SQL Server). La
persistencia cubrio: tarea e instancia de flujo, definicion BPMN, respuestas EAV del
formulario, ejecuciones de reglas, plantillas, adjuntos en Object Storage y el
AdminAuditLog inmutable.

---

## Que hace click al leer esto

1. **El sistema NO es "un CRUD"** - es un motor BPMN (WorkflowEngine) con formularios dinamicos (DynamicFormRenderer) y reglas ejecutables (RulesEngine).
2. **Cada modulo tiene su rol claro** - los operativos NO tocan configuracion.
3. **Los clientes externos participan sin cuenta** - via tokens con caducidad en una pagina publica aislada.
4. **Todo esta trazado** - la instancia de flujo + el AdminAuditLog inmutable cuentan cada movimiento.
5. **Las reglas son el "codigo" del negocio** - se cambian sin recompilar.
6. **Los flujos BPMN pueden migrarse a Camunda** - el XML es estandar OMG.
7. **El aislamiento multi-tenant es una invariante** - ningun paso pudo cruzar a otro tenant.

---

## Enlaces para profundizar

Cada seccion de esta narrativa tiene su detalle tecnico en otro doc del vault. Sugerido leer despues:

1. Aspecto y navegacion destino: [[Visión y entorno|Prototipo Final ECOREX]]
2. Vision maestra del sistema: [[Visión y entorno]]
3. Motor BPMN completo: [[00 - Visión Flujos]]
4. Motor de Reglas: [[Reglas - Catalogo real y verbos Ensamblado]]
5. Constructor formularios: [[00 - Visión Formularios]]
6. Multi-tenant real: [[Gestion de Empresas - Admin multi-tenant]]
7. Inventario de modulos: [[INVENTARIO GENERAL]]
