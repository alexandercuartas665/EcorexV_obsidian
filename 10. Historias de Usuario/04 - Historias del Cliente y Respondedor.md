---
tipo: historias-usuario
protagonistas: Carlos (Cliente externo), Maria (Analista interna)
modulos_cubiertos: [Visor por token, Mis Tareas Cliente 000620, Formulario 00031, Agentes IA 000867]
---

# Historias del Cliente y Respondedor - Carlos y Maria

> Dos personajes que interactuan con el sistema SIN configurarlo:
> - **Carlos** es un proveedor externo que responde formularios sin cuenta
> - **Maria** es una analista interna del tenant SKY SYSTEM que procesa lo que llega
>
> En el sistema destino .NET 10, Carlos ve una **pagina publica aislada** (nunca el
> shell del tenant) y Maria trabaja dentro del layout del
> [[Visión y entorno|Prototipo Final ECOREX]]. Vision en [[Visión y entorno]].

---

## Parte 1 - Historias de Carlos (cliente externo)

### Escenario 1 - Recibir un mail y responder una cotizacion

#### User Story
**Como** proveedor de equipos de computo,
**quiero** cotizar rapidamente sin tener que aprenderme un sistema nuevo,
**para que** pueda enviar la propuesta en 10 minutos y volver a mis pendientes.

#### Contexto
Carlos NO es usuario del sistema. Nunca vera el dashboard, ni el menu lateral.
Solo recibe un mail:

```
Asunto: Solicitud de cotizacion - SKY SYSTEM

Hola Carlos,

Necesitamos cotizar 20 laptops Lenovo ThinkPad para uno de nuestros clientes.
Por favor completa el formulario aqui:

https://app.ecorex.io/f/REDACTED

Fecha limite: 3 dias.

Gracias,
Andres - SKY SYSTEM
```

#### Pasos concretos
1. Carlos abre el link.
2. Ve **solo el formulario** (pagina publica Blazor), sin sidebar ni barra del sistema.
3. El formulario, renderizado por el **DynamicFormRenderer**, tiene:
   - Razon social (autocompletado con IA cuando escribe el NIT)
   - NIT
   - Fecha de la cotizacion
   - Tipo de requerimiento (Lista)
   - Productos requeridos (Multiples opciones checkbox)
   - Descripcion detallada (textarea)
   - Firma (canvas)
4. Llena todo. Al escribir el NIT, la razon social se autocompleta (regla `GENERAR_TABLAS_IA` del RulesEngine).
5. Firma con el mouse.
6. Click "Enviar respuestas".
7. Ve toast: "Se recibio tu cotizacion. Gracias!"

#### Criterios de aceptacion
- El link funciona mientras el token sea valido (no vencido, no revocado, dentro del rate limit)
- No hay login, pero si validacion del token contra su `tenant_id`, caducidad y estado
- Las respuestas se guardan como respuesta EAV (`jsonb`/`nvarchar(max)`) ligada al tenant SKY SYSTEM
- La firma se guarda como base64 en el valor del campo
- Carlos recibe copia por mail de sus respuestas

#### Seguridad (mejorada en destino)
A diferencia del origen, el token del destino **expira, se puede revocar y tiene
rate limiting**. Un link reenviado a un tercero puede invalidarse. Ver
[[Visor por token - docu_viewform]] para el estado del origen.

---

### Escenario 2 - Volver a editar una respuesta enviada

#### User Story
**Como** cliente,
**quiero** poder editar una respuesta si detecto un error antes del cierre,
**para que** el operador reciba la version corregida y no la original.

#### Como lo resuelve el sistema destino
En .NET 10 el visor publico **precarga** las respuestas previas: al reabrir el link,
el DynamicFormRenderer busca la respuesta EAV existente asociada a ese token (y su
tenant) y rellena el formulario. Carlos ve lo que envio, corrige el error y reenvia,
mientras el token siga vigente (respetando `USADO_UNA_VEZ` si el flujo lo exige).

#### Criterios de aceptacion
- Al reabrir un token con respuesta previa, el formulario aparece precargado
- El reenvio actualiza la respuesta sin duplicar registros
- Si el token ya fue marcado de un solo uso o esta vencido, el visor lo rechaza

---

## Parte 2 - Historias de Maria (analista interna)

### Escenario 1 - Recibir la respuesta y procesar

#### User Story
**Como** analista comercial interna,
**quiero** ver las respuestas de los proveedores organizadas en un solo lugar,
**para que** pueda comparar cotizaciones sin abrir 5 mails distintos.

#### Pasos concretos
1. Maria abre **Gestor de tareas / Tableros** (su vista "Mis tareas" del tenant).
2. Ve la tarea `T0042 - Cotizar 20 laptops` asignada a ella (Andres se la delego).
3. Click en la tarjeta > abre el panel de detalle.
4. En la seccion "Seguimiento del Proceso" ve los nodos de la instancia de flujo:
   - Paso 1 (Requerimiento) - Completado
   - Paso 2 (Cotizacion a Proveedores) - **En progreso** (aqui es donde esta)
   - Paso 3 (Aprobacion Cliente) - Por hacer
   - Paso 4-6 - Por hacer
5. Click "Abrir visor del formulario" > se abre el formulario con las respuestas del proveedor cargadas.
6. Revisa que este todo correcto.
7. Click "Marcar como completo" > el WorkflowEngine avanza el flujo al paso 3.

#### Criterios de aceptacion
- El detalle del proceso muestra los pasos reales de la instancia de flujo del tenant
- El paso activo se resalta
- Al marcar completo, el WorkflowEngine avanza el estado y resuelve el siguiente nodo
- El motor respeta el limite de iteraciones para nunca entrar en ciclo infinito

---

### Escenario 2 - Un formulario tiene 3 versiones (proveedor A, B, C)

#### User Story
**Como** analista,
**quiero** ver una tabla comparativa cuando 3 proveedores respondan la misma cotizacion,
**para que** pueda elegir el mejor en 2 minutos.

#### Como lo resuelve el sistema destino
El destino incluye la vista **Comparativa de Respuestas**: agrupa las respuestas del
mismo caso (mismo formulario y referencia, dentro del tenant) y muestra una matriz
-- filas = campos del formulario, columnas = respuestas de proveedores -- resaltando
el mejor precio y el menor tiempo de entrega. Maria elige el ganador en dos minutos
y el WorkflowEngine puede usar ese resultado como condicion de la siguiente
transicion.

#### Criterios de aceptacion
- La matriz solo cruza respuestas del mismo caso y del mismo tenant
- El mejor precio / menor plazo se resaltan automaticamente

---

## Parte 3 - Interaccion con Agentes IA

### Escenario 1 - Un cliente hace una pregunta general (bot conversacional)

#### User Story
**Como** proveedor,
**quiero** preguntar por WhatsApp "cuando pagan?" y que el sistema responda automaticamente,
**para que** no tenga que buscar la factura ni llamar.

#### Como funciona
1. El proveedor escribe en un chat de WhatsApp (integracion Evolution/`EvoApi`).
2. Un webhook validado entra por la API del tenant y publica el mensaje en el Event Bus.
3. Se dispara un agente IA (modulo `000867`).
4. El agente:
   - Identifica al proveedor por el numero de telefono (dentro del tenant SKY SYSTEM)
   - Consulta su ultima factura
   - Consulta el estado de la instancia de flujo
   - Responde: "Tu factura #12345 esta en el paso 'Entrega para gestion de pago', estimamos pago en 5 dias habiles."
5. El agente se apoya en el **RulesEngine** (verbo Ensamblado que llama al servicio de IA).

#### Criterios de aceptacion
- La respuesta llega en menos de 8 segundos
- Si la IA no encuentra la factura, escala a un humano (crea una tarea con responsable del cargo "Post-venta")
- La conversacion se guarda como respuesta EAV del tenant (`CHAT_WPP`)
- El agente opera solo con datos de SKY SYSTEM; nunca cruza a otro tenant

---

### Escenario 2 - Configurar un nuevo agente IA (esto lo hace Andres, no Maria)

**Como** configurador,
**quiero** crear un agente IA especializado en cotizaciones,
**para que** cuando un cliente escriba "necesito 10 laptops", el bot arme un draft de cotizacion.

Ver [[02 - Historias del Configurador#Escenario 5 - Crear una regla que autollene un campo con GPT]].

---

## Recap de los modulos que ve Carlos vs Maria

### Carlos (cliente externo)
```
/f/{token}   <- unico punto de entrada (pagina publica Blazor, aislada)
  |
  +-- Formulario renderizado por DynamicFormRenderer
  +-- Reglas asociadas al formulario (autocompletado IA via RulesEngine)
```

### Maria (analista interna, dentro del tenant SKY SYSTEM)
```
Inicio / Resumen
  |
  +-- Bloque PRINCIPAL
  |     +-- Gestor de tareas / Tableros   <- "Mis tareas" + carga del equipo (Kanban)
  |
  +-- Bloque MODULOS
      +-- Formularios (000131) LECTURA      <- ver respuestas de proveedores
      +-- Agentes IA (000867)               <- ver historial de conversaciones
```

Maria puede ver Reglas y Flujos pero NO tiene la policy para editarlos (solo Andres).

---

## Enlaces a documentacion tecnica

- Aspecto y navegacion destino: [[Visión y entorno|Prototipo Final ECOREX]]
- Vision maestra del sistema: [[Visión y entorno]]
- Visor por token: [[Visor por token - docu_viewform]]
- Constructor de formularios: [[00 - Visión Formularios]]
- Agentes IA y clases: [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]]
- Paginas basicas de tareas cliente: [[Tareas y Proyectos - paginas basicas]]
