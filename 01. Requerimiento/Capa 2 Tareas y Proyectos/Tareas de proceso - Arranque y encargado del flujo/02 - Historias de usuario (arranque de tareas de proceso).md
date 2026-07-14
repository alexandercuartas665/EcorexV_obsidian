---
tipo: historias-usuario
proyecto: Tareas de proceso - Arranque y encargado del flujo
fecha: 2026-07-14
---

# 02 - Historias de usuario (arranque de tareas de proceso)

> Convencion del capitulo: **el SISTEMA (ECOREX) tambien es un actor**. No es un formulario que
> guarda; es quien **lee la configuracion** (concepto, flujo, organigrama, formulario) y **decide**
> que abrir, a quien enrutar y a quien notificar. Las historias se escriben en ese lenguaje.

Personajes de ejemplo (tenant demo SKY SYSTEM):

- **Ana** - Auxiliar administrativa. Inicia procesos. No conoce el BPMN, solo el menu.
- **Beto** - Comprador (cargo). Atiende el primer paso del flujo de compras.
- **Carla** - Jefe de Compras (cargo). Aprueba.
- **Dario** - Administrador del tenant. Configura conceptos, flujos y formularios.
- **ECOREX** - el sistema.

---

## HU-01 - Arrancar un proceso desde el menu (camino feliz)

**Como** Ana (auxiliar), **quiero** iniciar una compra desde el menu sin tener que saber en que
tablero vive ni que flujo la gobierna, **para** que el proceso arranque solo y llegue a quien
corresponde.

```
Ana                          ECOREX
 |                             |
 |-- clic en                   |
 |   Mis Procesos > Procesos   |
 |   > Compras > "Compra"  --->|
 |                             |-- lee el CONCEPTO "Compra"
 |                             |     tiene flujo? SI  -> es una actividad-proceso
 |                             |     tiene tablero?    -> abre ESE tablero
 |                             |     es form-first?    -> NO (ver HU-02)
 |                             |
 |                             |-- abre el WIZARD ya resuelto:
 |                             |     Categoria    = Compras      (fijo)
 |                             |     Subcategoria = Compra       (fijo)
 |                             |     Encargado    = Beto         <-- lo DICTA el flujo
 |                             |                    (unico ocupante del cargo
 |                             |                     "Comprador" del 1er nodo)
 |<----------------------------|   etiqueta: "Paso 1 - Comprador"
 |                             |
 |-- escribe titulo/detalle    |
 |-- clic "Crear actividad" -->|
 |                             |-- EN UNA TRANSACCION:
 |                             |     1. crea la tarea (T00231) en el tablero del concepto
 |                             |     2. arranca la INSTANCIA del flujo publicado
 |                             |     3. el startEvent y los gateways se auto-resuelven
 |                             |     4. el 1er nodo Task queda Pending/IsCurrent
 |                             |     5. ASIGNA ese paso a Beto  <-- lo nuevo
 |                             |     6. notifica a Beto (campana + email)
 |                             |     7. audita
 |<-- "Actividad T00231 creada"|
```

**Criterios de aceptacion**
- Ana **nunca** eligio el tablero ni el flujo: los dedujo ECOREX del concepto.
- El combo Encargado llego **preseleccionado** con Beto y con la etiqueta del cargo del paso.
- Ana **no puede** elegir a alguien que no ocupe el cargo del primer nodo.
- Tras crear, la tarea aparece en el tablero **y** en el alcance "Pendientes mios" **de Beto**.
- Beto recibe la notificacion **al nacer la tarea**, no cuando alguien la reclame.

---

## HU-02 - Arrancar un proceso que empieza por FORMULARIO (form-first)

**Como** Ana, **quiero** que los procesos marcados "Inicia modulo al crear la tarea" me abran
**directamente el formulario**, **para** no caminar un wizard de 4 pasos que me pregunta cosas que
el sistema ya sabe.

```
Ana                          ECOREX
 |                             |
 |-- clic en                   |
 |   Mis Procesos > Procesos   |
 |   > Comercial >             |
 |   "Requerimiento infra" --->|
 |                             |-- lee el CONCEPTO
 |                             |     IniciaModulo = SI   +  FormDefinitionId != null
 |                             |     ==> ES FORM-FIRST
 |                             |
 |                             |-- NO abre el wizard.
 |                             |   Abre el FORMULARIO del concepto (modo Fill),
 |                             |   y resuelve por su cuenta:
 |                             |     Titulo    <- TituloAuto  ("Requerimiento infra - @cliente")
 |                             |     Detalle   <- DetalleAuto
 |                             |     Tablero   <- TaskBoardId del concepto
 |                             |     Encargado <- candidato del 1er nodo del flujo
 |<----------------------------|
 |                             |
 |-- diligencia los campos     |
 |-- clic "Enviar"          -->|
 |                             |-- EN UNA TRANSACCION:
 |                             |     1. valida el formulario EN SERVIDOR
 |                             |     2. crea la tarea (T00232)
 |                             |     3. arranca el flujo y asigna el 1er paso
 |                             |     4. persiste la FormResponse enlazada (ref = T00232)
 |                             |     5. notifica
 |<-- "Actividad T00232 creada"|
```

**Criterios de aceptacion**
- Ana **no ve** el wizard de 4 pasos en absoluto.
- El token `@cliente` del titulo se resuelve con el dato que Ana escribio **en el formulario**.
- Si el formulario **no valida**, **no se crea** la tarea (hoy la crea al entrar al paso 3: eso
  cambia; nada se persiste hasta que el formulario pasa validacion).
- La `FormResponse` queda **enlazada** a la tarea por su numero.
- **Escape hatch**: si el concepto es form-first pero el formulario esta archivado o sin permisos,
  ECOREX **cae al wizard normal** con un aviso claro, no rompe.

---

## HU-03 - Recibir y atender el paso que me toca (sin bandeja)

**Como** Beto (Comprador), **quiero** enterarme de que un proceso me esta esperando y atenderlo
**dentro de la tarea**, **para** no tener que revisar una bandeja aparte (ADR-0038).

```
ECOREX                        Beto
 |                             |
 |-- notifica (campana+email)  |
 |   "T00231 - Compra:         |
 |    tienes el paso           |
 |    'Cotizar' pendiente"  -->|
 |                             |
 |                             |-- abre el TABLERO
 |                             |-- alcance "Pendientes mios"
 |<----------------------------|
 |-- lista las tareas donde    |
 |   el PASO ACTUAL esta       |
 |   asignado a Beto (o su     |
 |   cargo es candidato)    -->|
 |                             |
 |                             |-- abre T00231
 |-- muestra, DENTRO de la     |
 |   tarea, la seccion FLUJO:  |
 |     ruta del proceso,       |
 |     paso actual resaltado,  |
 |     formulario del paso,    |
 |     adjuntos, y las         |
 |     acciones del paso    -->|
 |                             |
 |                             |-- adjunta cotizaciones
 |                             |-- clic "Completar paso"
 |<----------------------------|
 |-- AVANZA el flujo:          |
 |     cierra el paso de Beto  |
 |     resuelve el gateway     |
 |     activa el paso de Carla |
 |     ASIGNA y NOTIFICA a     |
 |     Carla                -->|
```

**Criterios de aceptacion**
- **No existe** una pagina "Mis pasos": el descubrimiento es el **tablero** (ADR-0038).
- El paso actual esta **visualmente resaltado** en la ruta del proceso dentro de la tarea.
- Al completar, el siguiente paso queda **asignado y notificado** sin intervencion humana.

---

## HU-04 - Aprobar / rechazar y que el flujo se desvie solo

**Como** Carla (Jefe de Compras), **quiero** aprobar o rechazar, **para** que ECOREX enrute la
tarea por la rama correcta del flujo sin que yo tenga que decidir a quien se la paso.

```
Carla                        ECOREX
 |                             |
 |-- abre T00231               |
 |-- seccion Flujo             |
 |-- "Rechazar" + motivo    -->|
 |                             |-- registra la decision en el paso
 |                             |-- evalua el GATEWAY con esa decision
 |                             |-- toma la rama "rechazado"
 |                             |-- reactiva el paso "Cotizar" (de Beto)
 |                             |-- ASIGNA y NOTIFICA a Beto con el motivo
 |                             |-- audita (quien, cuando, que decidio)
 |                             |
 |                             |   [si hubiera aprobado]
 |                             |-- toma la rama "aprobado" -> siguiente paso
 |                             |-- si NO hay siguiente paso:
 |                             |     cierra la instancia del flujo
 |                             |     mueve la tarea a la columna de CIERRE
 |                             |     del concepto (TaskBoardColumnId)
```

**Criterios de aceptacion**
- Carla **nunca** elige a quien le devuelve la tarea: lo dicta el flujo.
- El rechazo **vuelve** al paso correcto con el motivo visible.
- Al terminar el flujo, la tarea cae en la **columna de cierre configurada en el concepto**, no en
  una columna cualquiera.
- Todo queda auditado.

---

## HU-05 - Configurar un proceso nuevo y verlo aparecer en el menu (autoservicio)

**Como** Dario (administrador), **quiero** crear un concepto, dibujarle un flujo y asignarle cargos,
**para** que el proceso **aparezca solo** en el menu de la gente sin pedirle nada a desarrollo.

```
Dario                        ECOREX
 |                             |
 |-- /flujos: dibuja el BPMN   |
 |   asigna CARGO por nodo     |
 |-- "Publicar"             -->|
 |                             |-- valida: tiene al menos 1 nodo Task?
 |                             |           los nodos tienen cargo?
 |                             |           los cargos tienen ocupantes?
 |                             |-- si falta algo -> AVISA y no publica
 |                             |
 |-- /conceptos: crea la       |
 |   sub-categoria             |
 |   asocia el FLUJO           |
 |   (y el FORMULARIO si va    |
 |    a ser form-first)        |
 |-- "Guardar"              -->|
 |                             |-- si no tiene tablero -> LO CREA solo
 |                             |   (4 columnas, cierre = "Completado")
 |                             |-- lo enlaza al concepto (1:1)
 |                             |
 |                             |-- el concepto ahora tiene flujo publicado
 |                             |   ==> la hoja APARECE en Mis Procesos > Procesos
 |                             |       para todos los que tengan permiso
 |<----------------------------|
 |                             |
 |   [si Dario quita el flujo] |
 |                             |-- la hoja DESAPARECE del menu
```

**Criterios de aceptacion**
- Dario **no toca codigo ni el editor de menu**: el menu se dibuja de los datos.
- ECOREX **no publica** un flujo incompleto (sin nodos Task, sin cargo, o con cargos vacios).
- Un concepto guardado **siempre** queda con tablero (auto-creado si no se eligio).
- Si el flujo **no esta publicado**, el concepto **no debe** ofrecerse como proceso en el menu
  (hoy si aparece y crea tareas sin flujo, en silencio: es la guarda B6 a cerrar).

---

## Trazabilidad con las olas

| Historia | Olas que la habilitan (ver [[03 - Plan por olas pequenas]]) |
|---|---|
| HU-01 | A1, A2, A3 |
| HU-02 | B1, B2 |
| HU-03 | ya construida (ADR-0038); A3 la completa (notificar al nacer) |
| HU-04 | ya construida (ADR-0037 gateways); C1 la verifica (rechazo) |
| HU-05 | C2 (guardas de publicacion y de menu) |
