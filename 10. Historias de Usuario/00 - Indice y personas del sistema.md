---
tipo: indice-historias
proposito: entender el sistema desde el punto de vista de las personas que lo usan
---

# Historias de Usuario - Sistema ECOREX

> Estas notas cuentan **como se usa el sistema en la practica**, sin tecnicismos.
> Si nunca has visto ECOREX o vienes del lado no-tecnico (negocio, ventas,
> operaciones), empieza AQUI antes de leer las carpetas 03-09.
>
> Las historias transcurren en el **sistema destino .NET 10** - cuando una persona
> "abre el sistema", ve el layout del prototipo final: rail de iconos, sidebar
> `SKY SYSTEM / Plan Empresa - ECOREX`, dashboard "Buenos dias" con tarjetas KPI y
> tableros Kanban. El aspecto y la navegacion definitivos estan en
> [[00 - Prototipo Final ECOREX]] y la vision maestra en [[Visión y entorno]].

---

## Indice

| # | Historia | Personaje principal | Modulos que toca |
|---|----------|---------------------|------------------|
| 01 | [[01 - Historias del Operativo]] | Guadalupe (Coordinadora Operativa) | Gestor de tareas/Tableros, Proyectos, Inicio |
| 02 | [[02 - Historias del Configurador]] | Andres (Lider Tecnico) | Flujos del proceso (WorkflowEngine), Formularios (DynamicFormRenderer), Reglas (RulesEngine) |
| 03 | [[03 - Historias del Administrador]] | Julia (Admin del Sistema) | Usuarios, Roles, Dependencias, Config Entidad |
| 04 | [[04 - Historias del Cliente y Respondedor]] | Carlos (Cliente externo) + Maria (Analista) | Visor por token, Agente IA, Gestor de tareas |
| 05 | [[05 - Un dia en SKY SYSTEM (narrativa E2E)]] | Los 4 personajes | Todos - flujo completo end-to-end |

> Un quinto rol vive fuera del tenant: el **PlatformAdmin** (operador de la
> plataforma) gobierna varios tenants desde la consola de plataforma - planes,
> limites, habilitacion de modulos y salud global. No es una de las 4 personas de
> negocio de SKY SYSTEM; aparece cuando una historia cruza el borde del tenant.
> Ver [[Gestion de Empresas - Admin multi-tenant]].

---

## Las 4 personas que usan el sistema

### 1. Guadalupe - Coordinadora Operativa

**Perfil**: 32 anos, coordina un equipo de 8 operadores. Recibe requerimientos de
clientes y los distribuye. Es la primera en abrir el sistema por la manana. En el
prototipo es la usuaria demo (**Guadalupe Lopez - Administradora**), cuya ficha
aparece al pie del sidebar de SKY SYSTEM. Vive en **Gestor de tareas / Tableros**.

**Su rutina diaria**:
- 8:00 AM abre **Inicio / Resumen** (dashboard "Buenos dias, Guadalupe") y revisa
  las tarjetas KPI (Tareas activas, Proyectos en curso, Flujos ejecutandose,
  Alertas) y luego el tablero Kanban con la columna "Por hacer"
- Arrastra las tareas a los responsables por cargo (via SignalR el tablero se
  actualiza en vivo, sin recargar)
- Al mediodia filtra por "En progreso" para saber donde va cada caso
- 4:00 PM mueve tareas a "Completado" y revisa que ninguna haya vencido

**Sus dolores actuales que ECOREX resuelve**:
- Antes usaba Excel + WhatsApp para asignar (perdia trazabilidad)
- No sabia cuando una tarea se atascaba (ahora el KPI "Flujos ejecutandose" y las
  Alertas lo muestran)
- No podia demostrar a su jefe la carga real del equipo

**Modulos que domina** (nombres del prototipo):
- Inicio / Resumen (dashboard KPI)
- Gestor de tareas / Tableros (Kanban `Por hacer / En progreso / En revision /
  Completado`, filtros por Usuario/Etiqueta/Categoria/Subcategoria/Fecha)
- Proyectos (`000042`)
- Crear actividad (`000038`) para pedidos nuevos

**No entra a**: Flujos del proceso (BPMN), Reglas, SQL, modulos de desarrollo.

---

### 2. Andres - Lider Tecnico

**Perfil**: 40 anos, ingeniero de sistemas. Configura como se comporta el sistema
para que Guadalupe no tenga que llamarlo cada vez que cambia un proceso.
Vive en Flujos, Formularios y Reglas.

**Su rutina**:
- Recibe un requerimiento nuevo del cliente ("agregamos un paso de aprobacion antes de facturar")
- Abre **Flujos del proceso**, el editor BPMN embebido (bpmn-js) sobre el WorkflowEngine, y mueve el flujo
- Define quien puede aprobar (por cargo, no por usuario)
- Ajusta un formulario en el constructor EAV (DynamicFormRenderer) para capturar el motivo de aprobacion
- Configura una regla en el RulesEngine que notifique al gerente cuando alguien rechaza

**Sus dolores que ECOREX resuelve**:
- Antes cada cambio requeria tocar codigo VB.NET y republicar; ahora modela sin codigo y el motor lo ejecuta
- Los procesos vivian en la cabeza del gerente
- No podia probar cambios sin afectar produccion (ahora prueba reglas en modo test)

**Modulos que domina** (nombres del prototipo + motor destino):
- Flujos del proceso (`000291`) - editor BPMN sobre **WorkflowEngine** (portado de `AdmWorkflow`)
- Formularios (`000131`) - constructor drag&drop sobre **DynamicFormRenderer** (patron EAV)
- Reglas (`000802`) - **RulesEngine** (portado de `cl_gestion_reglas`, verbos Ensamblado)
- Agentes IA (`000867`)
- Biblioteca de Recursos (`000133`)
- Parametros / Modulos web (`000109`, variables por tenant)

**Trabaja con**: Julia (admin) y ocasionalmente con proveedor externo (3DEV).

---

### 3. Julia - Administradora del Sistema

**Perfil**: 45 anos, contadora publica. Es la duena de los datos maestros y la
seguridad. Cada vez que entra alguien nuevo a la empresa ella lo da de alta.

**Su rutina**:
- Alta y baja de usuarios cuando cambia el equipo
- Mantiene el organigrama actualizado
- Revisa quien tiene acceso a que modulo
- Configura plantillas de impresion cuando marketing cambia el logo

**Modulos que domina**:
- Usuarios (000073)
- Roles y Permisos (000194)
- Dependencias (000850)
- Configuracion de Entidad (000615)
- Plantillas de Impresion (000893)
- Tipos de Documento (000136) - consecutivos

**No toca**: nada en la seccion Automatizacion. Le dan miedo los flujos.

---

### 4. Carlos - Cliente Externo (respondedor)

**Perfil**: 28 anos, proveedor de equipos de computo. NO tiene cuenta en el
sistema principal. Solo recibe **links con tokens** para responder cotizaciones.

**Su interaccion**:
- Recibe un mail con "Nueva cotizacion solicitada" + link
- Abre el link, ve un formulario simple
- Llena datos (marca, modelo, precio, tiempo de entrega, firma digital)
- Envia

**Modulos que ve** (solo lectura y respuesta):
- Visor publico por token (una pagina Blazor aislada, `/f/{token}`) - renderiza el
  formulario con **DynamicFormRenderer** y NADA del shell del tenant
- Formulario 00031 SKY SYSTEM COMERCIAL CAPTURA REQUERIMIENTO
- Modo "Agente IA" cuando ese formulario tiene una regla del RulesEngine que autollena con GPT

**No conoce el sistema**. Para el, es simplemente "un formulario web bonito".
Nunca ve el sidebar SKY SYSTEM ni el dashboard.

---

### Personaje bonus: Maria - Analista

**Perfil**: 26 anos, entra tareas del cliente al sistema cuando llegan por mail
o WhatsApp. Es "la que interpreta lo que el cliente quiere". Trabaja como
puente entre Carlos y Andres.

**Su interaccion**:
- Ve las respuestas de Carlos en el Formulario
- Si algo esta incompleto, agrega comentarios
- Marca la tarea como "lista para asignar"

Es una version simplificada de Guadalupe.

---

## Como usar estas historias

Cada archivo `0N - Historias del ...md` tiene:

1. **User Stories formales** (Como X, quiero Y, para Z)
2. **Criterios de aceptacion** (checklist de "esto funciona si...")
3. **Escenarios de uso** (paso a paso concreto)
4. **Que hace click al leerlo** (el "aha!" que le da a un nuevo miembro del equipo)

El ultimo archivo `05 - Un dia en SKY SYSTEM` cuenta una historia **cronologica**
que atraviesa los 4 personajes en un mismo caso real - util para entender
como los modulos se conectan.

---

## Para quien es este vault

- Nuevos ingenieros que se suman al proyecto (leer 01, 02, 05)
- Product managers que van a proponer cambios (leer 05)
- Diseniadores UX que van a rediseñar pantallas (leer 01, 03, 04)
- Comerciales/preventa que hablan del producto (leer 05)
- Auditores externos (leer 03 + los docs tecnicos)

---

## Que no encontraras aca

- Codigo fuente. Ver `01. Requerimiento/Capa 5 Librerias Base` (MotherData y Funciones).
- Esquemas SQL detallados. Ver `02. Inventario de modulos` (ER logico + tablas por modulo).
- Detalles de arquitectura. Ver `01. Requerimiento/Capa 0 Vision General`.
- Como reconstruir un modulo. Ver `01. Requerimiento/Capa 6 Modulos Ecorex Especificaciones` y la spec de gen_reglas.
