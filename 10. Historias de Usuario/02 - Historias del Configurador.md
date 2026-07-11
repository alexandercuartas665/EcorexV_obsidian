---
tipo: historias-usuario
protagonista: Andres (Lider Tecnico)
modulos_cubiertos: [Flujos 000291, Formularios 000131, Reglas 000802, Agentes IA 000867, Parametros XML 000057]
---

# Historias del Configurador - Andres

> **Andres** es el "power user" tecnico del sistema. Cuando el negocio cambia
> (nuevo proveedor, nuevo cliente que exige aprobacion extra), Andres se encarga
> de reflejarlo en el sistema SIN tocar codigo. En el sistema destino .NET 10 modela
> sobre los tres motores portados: **WorkflowEngine** (BPMN), **DynamicFormRenderer**
> (formularios EAV) y **RulesEngine** (reglas). Todo lo que configura queda aislado
> dentro de su tenant SKY SYSTEM. Aspecto en [[Visión y entorno|Prototipo Final ECOREX]]; vision
> en [[Visión y entorno]].

---

## Escenario 1 - Dibujar un flujo BPMN nuevo

### User Story
**Como** lider tecnico,
**quiero** disenar visualmente un flujo de proceso arrastrando cajitas y flechas,
**para que** cuando el negocio cambie no tenga que llamar al proveedor 3DEV.

### Precondiciones
- El proceso ya esta pensado en papel (categoria + subcategoria + nodos)
- La categoria "Cotizacion Comercial" existe como actividad (`TIPO_TAR`) del tenant

### Pasos concretos
1. Abre **Flujos del proceso** (`000291`) en el bloque MODULOS del sidebar SKY SYSTEM.
2. Selecciona en el combo `Seleccione el Proceso` la opcion **"— Nuevo flujo —"**.
3. El canvas queda en blanco con un StartEvent inicial.
4. Arrastra desde la paleta lateral **bpmn-js** (embebido en Blazor) elementos: `Task`, `ExclusiveGateway`, `EndEvent`.
5. Los conecta con flechas de secuencia.
6. Nombra cada nodo (dobleclick > escribe > Enter).
7. Escribe titulo del proceso: `Aprobacion de Compras Mayores a 10M`.
8. Click **Guardar cambios**.

### Que pasa por detras
- El editor **bpmn-js** genera XML BPMN 2.0 estandar OMG (portable a bpmn.io, Camunda, Activiti)
- La definicion se guarda como **definicion de flujo** del tenant; el WorkflowEngine la valida y la cachea en Redis
- Cada nodo y cada transicion quedan persistidos con su `ID_ELEMENTO` (formato `Activity-xxxxx`, `Gateway-xxxxx`, `Event-xxxxx`), tipo BPMN y coordenadas

### Criterios de aceptacion
- Al reabrir el flujo, se ve identico
- El XML exportado abre limpio en https://demo.bpmn.io/new (validado - portabilidad BPMN)
- Cada nodo tiene un `ID_ELEMENTO` unico
- La definicion queda ligada al `tenant_id` (otro tenant nunca la ve)

---

## Escenario 2 - Parametrizar un nodo (asignar usuarios, formulario y reglas)

### User Story
**Como** configurador,
**quiero** decir "en este paso del flujo participa el CARGO Comprador y llena el formulario X, y al terminar dispara la regla Y",
**para que** el sistema sepa que hacer cuando el caso pase por ese nodo.

### Pasos concretos
1. En el editor de **Flujos del proceso**, hace **click sobre un nodo** (ej. "Aprobar Cotizacion").
2. En el panel lateral derecho aparecen los **acordeones** de propiedades del nodo.
3. Expande **"Asignar Usuarios"**:
   - Selecciona el cargo "COMPRADOR" (esto se traduce en una policy del PermissionsManager para ese nodo).
4. Expande **"Recursos y Componentes"**:
   - Elige Formulario "00031 SKY SYSTEM COMERCIAL CAPTURA" (lo renderizara el DynamicFormRenderer).
5. Expande **"Reglas asociadas"**:
   - Grupo = FORMULARIOS
   - Documento = 00005 OPERACIONES DE FORMULARIOS
   - Verbo = ASIGNAR_CONSECUTIVO (verbo Ensamblado del RulesEngine)
6. Click **Aplicar**.

### Criterios de aceptacion
- Al ejecutar una instancia del flujo, cuando llegue al paso "Aprobar Cotizacion":
  - Solo los usuarios con cargo COMPRADOR (via policy) ven la tarea pendiente
  - El formulario 00031 se renderiza con el **DynamicFormRenderer**
  - Al guardar el formulario, el WorkflowEngine invoca al RulesEngine que ejecuta ASIGNAR_CONSECUTIVO
- Ver detalle en [[Parametrizacion por nodo - panel Propiedades]]

---

## Escenario 3 - Disenar un formulario dinamico con 15 preguntas

### User Story
**Como** configurador,
**quiero** disenar un formulario arrastrando tipos de control (Texto, Lista, Fecha, Firma, etc.),
**para que** el operador tenga una UI limpia para capturar informacion sin campos libres desordenados.

### Precondiciones
- Andres sabe que campos necesita capturar (idealmente lista escrita)

### Pasos concretos
1. Abre **Formularios** (`000131`) - el constructor visual sobre **DynamicFormRenderer**.
2. Click **"+ Nuevo formulario"**. Se crea un formulario vacio con codigo autogenerado.
3. Escribe titulo: `Cotizacion Proveedores Q4 2026`.
4. **Arrastra** desde la paleta lateral:
   - `Titulo` (para header de seccion)
   - `Texto` (razon social del proveedor)
   - `Numero` (NIT)
   - `Fecha` (fecha de la cotizacion)
   - `Lista` (tipo de producto: Hardware / Software / Servicio)
   - `Multiples Opciones` (checkboxes para productos requeridos)
   - `Abierto` (descripcion detallada, textarea 4 filas)
   - `Firma` (canvas para firmar)
5. Para el campo `Lista`, define las opciones en el panel derecho: `Hardware|Software|Servicio`.
6. Marca `Firma` como **Obligatorio=SI**.
7. Click **Guardar**.

### Que pasa por detras
- La definicion del formulario (cabecera, preguntas, tipos de control, validaciones y orden) se guarda con el patron **EAV** que gestiona el DynamicFormRenderer, ligada al `tenant_id`
- Las respuestas del usuario final iran despues a columnas `jsonb` (PostgreSQL) / `nvarchar(max)` (SQL Server) segun el DAL dual del tenant
- Las secciones (un `Titulo` actua como separador) se agrupan como contenedor tipo arbol

### Criterios de aceptacion
- Click "Vista previa" muestra el formulario renderizado como el usuario final lo veria
- El drag&drop respeta el orden de las preguntas
- Los 19 tipos de control existen en la paleta (documentados en el catalogo real)
- El formulario queda disponible solo dentro del tenant SKY SYSTEM

---

## Escenario 4 - Compartir un formulario con proveedor externo via token

### User Story
**Como** configurador,
**quiero** generar un link publico con token que permita a un proveedor externo (no usuario del sistema) responder el formulario,
**para que** capturemos datos sin crear cuentas para gente que va y viene.

### Pasos concretos
1. Andres tiene el formulario `00031` listo.
2. Desde el formulario pulsa **"Generar link publico"**: el Module Registry crea un token de acceso ligado al formulario `00031` y al `tenant_id` de SKY SYSTEM.
3. La URL publica queda: `https://app.ecorex.io/f/REDACTED`
4. Envia este link por mail al proveedor.

### Criterios de aceptacion
- Al abrir el link, el visor publico (pagina Blazor aislada) resuelve el token, valida su `tenant_id`, caducidad y revocacion, y carga el formulario `00031`
- El proveedor ve solo el formulario, no el resto del sistema ni nada de otros tenants
- Al responder, los datos se persisten como respuesta EAV (`jsonb`/`nvarchar(max)`) ligada al tenant

### Mejora sobre el origen (corregida en destino)
En el sistema legacy los tokens vivian en `GEN_PARAMETROS`/`GEN_TOKEN` **sin caducidad
ni revocacion** (riesgo documentado en [[Visor por token - docu_viewform]]). El
destino formaliza el token con `FECHA_EXPIRA`, `USADO_UNA_VEZ` y `REVOCADO`, rate
limiting y validacion en cada apertura. Un link vencido o revocado se rechaza.

---

## Escenario 5 - Crear una regla que autollene un campo con GPT

### User Story
**Como** configurador,
**quiero** que cuando el usuario escriba la razon social de un cliente, el sistema autocomplete el NIT y direccion consultando una IA,
**para que** el operador ahorre 2 minutos por caso.

### Pasos concretos
1. Va a **Reglas** (`000802`) - el **RulesEngine**.
2. Selecciona el documento **`00002 LLENAR DATOS CON IA`** (grupo INTELIGENCIA ARTIFICIAL).
3. Click "Agregar nuevo regla de trabajo" (abre modal grande).
4. En el modal:
   - Nombre: `AUTOCOMPLETAR_DATOS_CLIENTE`
   - Tipo ejecucion: **Ensamblado**
   - Orden: 1
   - Estado: Activo
   - PARAM_XML: pega el XML apuntando a `GestionMovil.cl_ia_reglas_formularios` + evento `doc_generar_tabla`:

```xml
<PagXml>
   <CorXml>
      <NOMBRE>ENSAMBLADO_DATA</NOMBRE>
      <PROCESO>GestionMovil.cl_ia_reglas_formularios</PROCESO>
      <EVENTO>doc_generar_tabla</EVENTO>
   </CorXml>
   <CorXml>
      <NOMBRE>TEST_PARAM</NOMBRE>
      <CAMPO>CAMPO_RAZON_SOCIAL</CAMPO>
      <ETIQUETA>Campo que dispara:</ETIQUETA>
      <TIPO>TEXTO</TIPO>
   </CorXml>
   <CorXml>
      <NOMBRE>RETURN_SMS</NOMBRE>
      <DATA>Se autocompleto: NIT @@NIT@@, Direccion @@DIR@@</DATA>
   </CorXml>
</PagXml>
```
5. Click **"Crear o actualizar regla"**.
6. Ahora asocia la regla al campo del formulario:
   - Va a **Formularios** (`000131`).
   - Abre el formulario donde esta el campo `razon_social`.
   - Click en el campo.
   - En el panel derecho, en "Reglas asignadas" agrega la regla que acaba de crear.

### Que pasa por detras al ejecutar
Cuando el usuario final escribe la razon social:
1. El **DynamicFormRenderer** detecta el evento `change` en el campo.
2. Consulta que reglas aplican a esa pregunta del formulario.
3. Encuentra la regla `AUTOCOMPLETAR_DATOS_CLIENTE`.
4. Invoca al **RulesEngine** con el id de la regla y los datos del formulario.
5. El motor lee el PARAM_XML, resuelve el verbo Ensamblado del registro tipado e invoca su handler.
6. Ese handler llama a OpenAI (via el servicio de IA/`clChatGPT` portado), recibe el NIT y la direccion.
7. Actualiza los campos del formulario en vivo (SignalR).
8. Escribe historial de ejecucion (usuario, fecha, duracion, resultado) en la auditoria del tenant.

### Criterios de aceptacion
- La regla aparece en el grid del documento `00002`
- Al probarla con el boton "Ejecutar" del modal, el toast muestra la respuesta
- La ejecucion queda registrada en la auditoria del tenant con usuario, fecha, duracion, filas afectadas
- La regla y su historial jamas se ven desde otro tenant

---

## Escenario 6 - Copiar un flujo existente para adaptarlo

### User Story
**Como** configurador,
**quiero** duplicar un flujo BPMN que ya funciona para crear una variante con pequenios cambios,
**para que** no tenga que dibujar todo desde cero.

### Como lo resuelve el sistema destino
En .NET 10 el editor de **Flujos del proceso** trae boton **"Duplicar flujo"**:
1. Andres abre el flujo original.
2. Click en **"Duplicar flujo"**.
3. El WorkflowEngine clona la definicion completa (cabecera, nodos, transiciones, reglas y plugins) con un codigo nuevo dentro del mismo tenant.
4. Genera nuevos `ID_ELEMENTO` con hash aleatorio para evitar colisiones.
5. Andres renombra y ajusta la variante.

### Alternativa manual
Si prefiere, sigue disponible el ciclo Exportar XML BPMN -> crear flujo vacio ->
Importar XML -> renombrar, util para llevar un flujo entre entornos o a Camunda.

---

## Escenario 7 - Probar una regla antes de activarla en produccion

### User Story
**Como** configurador,
**quiero** ejecutar una regla en modo "prueba" con datos ficticios,
**para que** verifique que hace lo esperado antes de activarla para todos los usuarios.

### Pasos concretos
1. En el modal de edicion de la regla (Reglas > editar verbo).
2. Ve el editor de PARAM_XML del lado izquierdo.
3. Ve el **preview en vivo** del formulario que genera el XML en el lado derecho.
4. Llena los TEST_PARAM con valores de prueba.
5. Click "Ejecutar prueba" (verde).
6. Recibe un toast con resultado + una entrada de auditoria marcada como prueba (`USUARIO='TEST'`).

### Criterios de aceptacion
- La ejecucion en modo prueba NO afecta datos reales del formulario
- El log queda claramente marcado como prueba
- Si falla, muestra el error tal cual (para debug)
- La prueba corre en el sandbox del RulesEngine, dentro del tenant, sin salir a produccion

---

## Recap de los modulos que Andres domina

```
Bloque MODULOS del sidebar SKY SYSTEM
  +-- Flujos del proceso           000291  <- editor BPMN sobre WorkflowEngine
  +-- Formularios                  000131  <- constructor sobre DynamicFormRenderer
  +-- Agentes IA                   000867  <- prompts + integraciones GPT
  +-- Reglas                       000802  <- RulesEngine (8 docs + 21 verbos Ensamblado)
  +-- Biblioteca de Recursos       000133  <- snippets reutilizables
  +-- Modulos web                  000109  <- registro de modulos + variables por tenant
```

Todo lo que Andres configura queda ligado al `tenant_id` de SKY SYSTEM.

---

## Lo que Andres NO hace

- Asignar tareas dia a dia (lo hace Guadalupe)
- Dar de alta usuarios (lo hace Julia)
- Responder formularios como usuario final (lo hace el operador o cliente)
- Gobernar planes/limites o habilitar tenants (eso es del PlatformAdmin)

---

## Enlaces a documentacion tecnica

- Aspecto y navegacion destino: [[Visión y entorno|Prototipo Final ECOREX]]
- Vision maestra del sistema: [[Visión y entorno]]
- Motor BPMN: [[00 - Visión Flujos]] y [[AdmWorkflow - Motor de ejecucion]]
- Parametrizacion por nodo: [[Parametrizacion por nodo - panel Propiedades]]
- Motor de Reglas: [[Reglas - Catalogo real y verbos Ensamblado]]
- Constructor Formularios: [[00 - Visión Formularios]] y [[Constructor - Patron EAV y motor visual]]
- Spec de gen_reglas: [[gen_reglas - Spec para reconstruir en Claude Design]]
