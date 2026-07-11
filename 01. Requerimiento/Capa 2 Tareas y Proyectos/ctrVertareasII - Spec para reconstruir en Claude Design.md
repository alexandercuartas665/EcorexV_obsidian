---
tipo: spec-reconstruccion
control: ctrVertareasII (vista detalle de una tarea existente)
consumido_por: NEWFRONT_admtareas.aspx (000636), NEWFRONT_tar_tareascli.aspx (000620)
archivo_origen: Bootstrap\Formularios\Modulos\Tareas\controles\ctrVertareasII.ascx
tamanio: 855 lineas markup + 1762 lineas code-behind
verificado_en_prod: 2026-07-01 app.bitcode.com.co
---

# ctrVertareasII - Spec para reconstruir en Claude Design

> **Vista de detalle/edicion** de una tarea existente. Se abre al hacer click en
> una fila de un grid o al abrir "Mis Tareas". Muestra el flujo BPMN del caso,
> los datos, un timer de trabajo (worklog), notas, y acciones (asignar, reasignar,
> suspender, guardar avance).
>
> **NO es el mismo que ctrTareasII** (ese es para crear/editar cabecera). Este
> es para trabajar EL CASO en su ciclo de vida.
>
> **Naturaleza de esta nota (ORIGEN + DESTINO):** las secciones 1 a 11 documentan
> la **UX del control ORIGEN** (WebForms) - el panel de trabajo del caso tal como
> opera hoy. Se conservan como referencia de experiencia y catalogo de reglas del
> ciclo de vida de la tarea. La seccion **0 (abajo)** fija la **VISION DESTINO**:
> como se reconstruye este panel en **Blazor (.NET 10)** con el aspecto del
> prototipo final. Vision maestra en [[Visión y entorno]]; aspecto definitivo en
> [[Visión y entorno|Prototipo Final ECOREX]].

---

## 0. Vision destino - reconstruccion en Blazor (.NET 10)

En el destino, `ctrVertareasII` se reconstruye como el componente Blazor
`TareaDetalle`, el panel donde el operador **ejecuta** la tarea. Preserva toda la
mecanica de trabajo (flujo BPMN horizontal, worklog con timer, checklist,
adjuntos, notas, navegacion entre casos) y adopta el lenguaje visual del prototipo
final. Reencuadre por bloque:

1. **Cabecera al estilo prototipo**: el HERO del origen (5 meta pills) se alinea a
   la cabecera de [[Visión y entorno|Prototipo Final ECOREX]]: **Estado, Asignados** (avatares),
   **Fecha limite** (con dias restantes), **Etiquetas**, mas Prioridad y Tiempo
   usado. Cuando el detalle se abre en contexto de proyecto, comparte las
   pestanas **Vista Tablero / Timeline / Calendario** del proyecto.
2. **Flujo BPMN vivo**: la barra horizontal de pasos la alimenta `WorkflowEngine`
   (antes `AdmWorkflow`). Avanzar un paso llama `WorkflowEngine.Advance()`, que
   ejecuta las reglas del nodo via `RulesEngine` y emite `task.state.changed` por
   **SignalR**: el Kanban y los grids de otros usuarios se actualizan sin
   recargar. La deteccion de ciclos (limite de iteraciones, "Ciclo N") se hereda
   del motor. Ver [[Visión y entorno]] secciones 7 y 9.
3. **Worklog transaccional**: el timer inline sigue siendo la fuente de verdad en
   cliente (JS/JS interop), pero cada entrada se persiste con concurrencia
   optimista (rowversion / xmin) para evitar que dos ediciones simultaneas del
   mismo caso se pisen. El tiempo se normaliza a UTC guardando la TZ del tenant
   (se corrige el error heredado de zonas horarias).
4. **Aprobacion / reasignacion**: los sub-controles `ctrAprobacion` /
   `ctrReasignacion1` se reconstruyen como componentes Blazor; el panel de
   aprobacion solo aparece cuando el nodo actual es un Gateway BPMN, y las
   acciones se gobiernan por **policies .NET** (antes `Asignar="1"` /
   `PERMITE_*`).
5. **Adjuntos y notas**: a Object Storage con metadatos auditados; las notas y el
   historial de worklog quedan en el registro de **auditoria inmutable**.
6. **Multi-tenant real**: `Me.Task` (id de `SISTEMA`) se resuelve dentro del
   filtro global por `TenantId` + RLS; un usuario nunca puede abrir por id una
   tarea de otro tenant aunque manipule la URL.
7. **Solo lectura seguro**: `FlagCerradoFijo="1"` se traduce a un estado de
   dominio "cerrado" que un `<AuthorizeView>` / render condicional respeta; el
   borrado de datos del caso es **soft-delete**.

> El prompt de Claude Design (seccion 9) y el design system `td2-*` se conservan
> como **descripcion del comportamiento y del lenguaje visual objetivo**; en el
> destino se implementan con los tokens del design system Blazor del prototipo.

---

## 1. Identidad del control

| Campo | Valor |
|---|---|
| Path | `Bootstrap\Formularios\Modulos\Tareas\controles\ctrVertareasII.ascx` |
| Clase | `GestionMovil.ctrVertareasII` |
| Modo | UserControl (`.ascx`) - se embebe como modal grande |
| Se abre desde | Grid de `NEWFRONT_admtareas` (000636), Kanban, Mis Tareas (000620) |
| Design system | `td2-*` (Task Detail v2) con clases prefijadas |
| Tabla origen | `TAR_SEGUIMIENTO_PROCESO` + `SISTEMA` |

---

## 2. Proposito en una frase

Panel de trabajo del caso: donde el operador **ejecuta la tarea**, registra
tiempo trabajado, avanza pasos del flujo BPMN, sube evidencias y navega entre
casos consecutivos sin cerrar el modal.

---

## 3. Layout visual (mapa ASCII)

```
+============================================================================+
| MODAL AMPLIO (95vw, ~90vh)                                                |
+============================================================================+
| PAGE HEADER                                                                |
|                                                                            |
| BITCODE / Actividades / T0042 - Cotizar 20 laptops                        |
|                                                                            |
|                                    [<- Anterior]  [Siguiente ->]  [X]     |
+============================================================================+
| TABS BAR                                                                   |
|                                                                            |
|  Desc. General    |    Avanzado    |    Aplicaciones    |    Archivos     |
|  ==============                                                            |
+============================================================================+
| BODY (con scroll)                                                          |
|                                                                            |
| == HERO SECTION (card destacada, fondo suave) ==                          |
|                                                                            |
| +---------------------------------------------------------------------+   |
| |  T0042 - Cotizar 20 laptops Lenovo ThinkPad                        |   |
| |  Encargado ANDRES.M | Cliente ALPES TECHNOLOGY                     |   |
| |                                                                     |   |
| |  META ITEMS (5 pills con icono):                                    |   |
| |  [Encargado] [Fecha entrega] [Tiempo usado] [Prioridad] [Estado]   |   |
| |                                                                     |   |
| |  ACCIONES (row de botones):                                        |   |
| |  [Asignar]  [Reasignar]  [Suspender]  [Guardar avance]             |   |
| |                                                                     |   |
| |  COLOR SWATCH: [selector color para etiquetar visualmente el caso] |   |
| +---------------------------------------------------------------------+   |
|                                                                            |
| == FLUJO DEL PROCESO (barra horizontal scrollable) ==                     |
|                                                                            |
| +---------------------------------------------------------------------+   |
| | FLUJO: Cotizacion Comercial (COMERCIAL REQUERIMIENTO INFRAESTR.)   |   |
| | Estado actual: paso 3 de 8                                          |   |
| |                                                                     |   |
| | (v) Requerimiento -> (v) Cotizacion -> [*] Aprobacion -> ( ) Fac.. |   |
| |     (done)             (done)             (active)        (pending)|   |
| |                                                                     |   |
| | [Continuar]  [Rechazar]  [Regresar]  [Reasignar cargo]             |   |
| +---------------------------------------------------------------------+   |
|                                                                            |
| == BODY GRID (2 cols: main + sidebar) ==                                  |
|                                                                            |
| +---------------------------------------+  +-------------------------+    |
| | MAIN COL                              |  | SIDEBAR                 |    |
| |                                       |  |                         |    |
| | ETAPA ACTUAL (card destacada)         |  | Resumen                 |    |
| |   . Circle icon: "Etapa actual"       |  |   - Datos rapidos       |    |
| |                                       |  |   - Datos cliente       |    |
| |   +-------------------------------+   |  |                         |    |
| |   | DETAIL SECTION                |   |  |                         |    |
| |   | Actividad: [textarea editable]|   |  +-------------------------+    |
| |   +-------------------------------+   |                                 |
| |                                       |  +-------------------------+    |
| |   +-------------------------------+   |  | Lista de chequeo        |    |
| |   | DETAIL SECTION                |   |  |   [ ] item 1            |    |
| |   | WORKLOG (inline timer)        |   |  |   [x] item 2 (done)     |    |
| |   |                               |   |  |   [+ Agregar item]      |    |
| |   | +---------+  +--------+       |   |  +-------------------------+    |
| |   | | 00:12:34| Manual: |         |   |                                 |
| |   | | PILL    | Detalle |         |   |  +-------------------------+    |
| |   | | +Iniciar| Duracion|         |   |  | Adjuntos y seguimiento |    |
| |   | +---------+ [Guardar]|         |   |  |   . archivos.pdf x3     |    |
| |   |             +--------+         |   |  |   [+ Subir archivo]     |    |
| |   |                                |   |  |                         |    |
| |   | Total: 4h 32m                  |   |  |   Notas:                |    |
| |   |                                |   |  |   [texto notas]         |    |
| |   | Entradas anteriores:           |   |  |   [+ Agregar nota]      |    |
| |   |   [entry] 2h - Cotizando       |   |  +-------------------------+    |
| |   |   [entry] 1h 30m - Espera      |   |                                 |
| |   |   [entry] 1h - Revision        |   |                                 |
| |   +-------------------------------+   |                                 |
| +---------------------------------------+  +-------------------------+    |
|                                                                            |
+============================================================================+
| FOOTER                                                                     |
| [Cerrar]                              [Volver a la lista]                 |
+============================================================================+
```

---

## 4. Elementos verificados en produccion `app.bitcode.com.co`

**Design system detectado:**
- Prefijo de clases: `td2-*` (Task Detail v2)
- Root class: `td2` (id termina en `_td2root`)

**Tabs principales (4):**
1. `Desc. General` - vista predeterminada
2. `Avanzado` - opciones avanzadas (SQL, XML, permisos por caso)
3. `Aplicaciones` - integraciones/apps asociadas
4. `Archivos` - gestor de adjuntos

**Meta items visibles en el HERO (5):**
1. Encargado
2. Fecha entrega
3. Tiempo usado
4. Prioridad
5. Estado

**Sidebar cards (3):**
1. **Resumen** (informacion condensada)
2. **Lista de chequeo** (checklist editable)
3. **Adjuntos y seguimiento** (archivos + notas)

**Card eyebrow del main:**
- "Etapa actual" (paso actual del flujo BPMN)

**Worklog form:**
- Timer inline con boton Iniciar/Pausar
- Campo Manual (Duracion + Detalle)
- Total acumulado
- Historial de entradas anteriores

**Breadcrumb:**
- `BITCODE / Actividades / [titulo del caso]`

---

## 5. Componentes ASP.NET clave

### Header
| Control | ID | Uso |
|---|---|---|
| Task ID | `LblTaskId` | Muestra "T0042" |
| Componentes | `LabelComponentes` | Cuenta plugins asociados |
| Archivos | `lblarchivos` | Cuenta archivos subidos |

### Hero
| Control | ID | Comportamiento |
|---|---|---|
| Nav Anterior | `btnHeroNavAnterior` | Dispara `NavAnterior` event, va al caso previo |
| Nav Posicion | `lblNavPosicion` | "3 de 45" |
| Nav Siguiente | `btnHeroNavSiguiente` | Event `NavSiguiente` |
| Asignar | `btnHeroAsignar` | Abre `PanelHeroAsignar` con dropdown `cmbAsignadoHero` |
| Cerrar asignar | `btnHeroCloseAsignar` | Colapsa panel |
| Reasignar | `btnHeroReasignar` | Abre control `ctrReasignacion1` |
| Suspender | `btnHeroSuspender` | Marca `ESTADO='SUSPENDIDO'` |
| Guardar avance | `btnHeroGuardarAvance` | Persiste worklog |
| Color swatch | `lblColorHeroSwatch` + `txtColorHeroInput` | Cambia color del caso |
| Encargado | `LblHeroEncargado` | Read-only |
| Fecha pentrega | `txtpentrega` | Datepicker editable |
| Horas / Minutos | `cmbHoras`, `cmbMinutos` | Hora entrega |
| Prioridad | `LblHeroPrioridad` | Read-only chip color |
| Estado | `cmbestadotarea` | Editable: Activa, Pendiente, En proceso, Terminado, etc. |

### Flujo del proceso
| Control | ID | Uso |
|---|---|---|
| Panel flujo | `PanelFlujoproceso` | Contenedor |
| Nombre proceso | `LiteralNombreProceso` | "COMERCIAL REQUERIMIENTO" |
| Detalle actividad | `LiteralDetalleactividad` | Nombre del paso actual |
| Continuar/aprobar | `Linkbutton1` | Avanza paso (dispara `AdmWorkflow.SiguienteEstado`) |
| Regresar/rechazar | `Linkbutton7` | Rechaza y va al paso anterior |

### Timer / Worklog
| Control | ID | Uso |
|---|---|---|
| Timer segundos | `HiddenFieldTimerSeconds` | Persiste tiempo |
| Actividad textarea | `txtAtividad` | Descripcion del trabajo hecho |
| Agregar entrada | `LinkButton2` | INSERT en tabla de historial |

### Aprobacion y Reasignacion
| Control | ID | Uso |
|---|---|---|
| Panel autorizacion | `PanelAutorizacion` | Visible si hay aprobacion pendiente |
| Panel reasignacion | `Panelreasignacion` | Visible si esta reasignando |
| ctrAprobacion | (custom) | Sub-control con logica de aprobacion |
| ctrReasignacion1 | (custom) | Sub-control con logica de reasignacion |

### Errores
| Control | ID | Uso |
|---|---|---|
| Error | `LtrError` | Muestra errores de operacion |
| Permisos | `LtrPermisos` | "No tienes permiso para esta accion" |

---

## 6. Propiedades publicas (23 props)

```vb
Public Property Modulo() As String                  ' 000636 / 000620
Public Property TareaSoporte() As String            ' "1" si sucursal 01
Public Property Asignar() As String                 ' "1" si PERMITE_ASIGNAR
Public Property Empresa() As String
Public Property Usuario() As String
Public Property XMLTareaNotificacion() As String
Public Property Task() As Integer                   ' Task ID (REG de SISTEMA)
Public Property TituloCaso() As String
Public Property ColorCaso() As String               ' Color hex personalizado
Public Property FechaCaso() As String
Public Property CargarArchivos() As String
Public Property ReferenciaIIFijo() As String
Public Property FlagCerradoFijo() As String         ' Si el caso ya esta cerrado (read-only)
Public Property PosicionNavegacion() As String      ' "3 de 45" para nav
Public Property IdFlujoProceso() As String
Public Property IdProceso() As String
Public Property IdEncargadoProceso() As String
Public Property IdEncargadoCiclo() As String        ' Para loops BPMN
Public Property PERMITE_VER_MIS_PROYECTOS() As String
Public Property PERMITE_VER_PROYECTOS_TODOS() As String
Public Property PERMITE_VER_TODAS_LAS_ACTIVIDADES() As String

' Eventos que el padre escucha
Public Event NotificarCambio As EventHandler   ' Refresca lista tras guardar
Public Event CambioColor As EventHandler        ' Cambio de color del caso
Public Event NavAnterior As EventHandler        ' Ir a caso anterior
Public Event NavSiguiente As EventHandler       ' Ir a caso siguiente
```

---

## 7. Metodos clave (code-behind)

- **`Page_Load`** - carga datos del caso via `Me.Task`, resuelve encargado, hidratar hero
- **`ctrReasignacion1_Asignar`** - handler cuando el sub-control reasigna
- **`ctrAprobacion_Autorizar`** - handler cuando aprueba en el sub-control de aprobacion
- **Metodos que hacen SELECT sobre `TAR_SEGUIMIENTO_PROCESO`** para armar el flujo visual

---

## 8. Datos capturados en produccion (2026-07-01)

**Tabs verificados:** Desc. General, Avanzado, Aplicaciones, Archivos
**Meta items:** Encargado, Fecha entrega, Tiempo usado, Prioridad, Estado
**Sidebar cards:** Resumen, Lista de chequeo, Adjuntos y seguimiento
**Worklog:** presente con timer inline
**Breadcrumb:** BITCODE / Actividades / [titulo]

---

## 9. Prompt directo para Claude Design / Artifact

```
Construye un MODAL AMPLIO (95vw, 90vh) de "Vista de detalle de tarea"
inspirado en Linear/Notion/Asana. Es DIFERENTE al modal de crear tarea:
este es para TRABAJAR en la tarea (registrar tiempo, avanzar paso, notas).

USA LAS MISMAS CSS VARIABLES del wizard de crear tarea (fondos claros,
azul primary #2563EB, Inter font, border-radius 8-20px).

ESTRUCTURA (top-down):

1) HEADER
   - Breadcrumb: "BITCODE / Actividades / T0042 - Cotizar 20 laptops"
   - Derecha: [<- Anterior] [Siguiente ->] flechas para navegar entre casos
     sin cerrar el modal, + boton X

2) TABS BAR (4 tabs):
   - Desc. General (activa)
   - Avanzado
   - Aplicaciones
   - Archivos

3) TAB "Desc. General" tiene:

   A) HERO SECTION (card con sombra suave, fondo blanco puro):
      - Titulo grande: "T0042 - Cotizar 20 laptops Lenovo ThinkPad"
      - Subtitulo pequeno: "Encargado: ANDRES.M | Cliente: ALPES TECHNOLOGY"
      - Meta pills (row de 5, iconos monocromo):
        * Encargado: ANDRES.M (avatar circular con inicial)
        * Fecha entrega: 15 jul 2026 (icono calendario)
        * Tiempo usado: 4h 32m (icono reloj)
        * Prioridad: ALTA (chip rojo)
        * Estado: EN PROCESO (chip amarillo)
      - Row de acciones (botones outline):
        [+ Asignar] [Reasignar] [Suspender] [Guardar avance]
      - Color swatch: circulo de color personalizado + input para cambiarlo

   B) FLUJO DEL PROCESO (card):
      - Header: nombre proceso "Cotizacion Comercial (COMERCIAL REQ. INFR.)"
        - Info: "Paso 3 de 8"
      - Steps horizontales scrollables (chevrons):
        [v Requerimiento] > [v Cotizacion] > [* Aprobacion] > [ Factura] > ...
        Los completados: verde con checkmark
        El activo: azul con anillo pulsante
        Los pendientes: gris opaco
      - Acciones:
        [Continuar ->] (verde, avanza paso)
        [Rechazar]    (rojo outline, regresa a paso anterior)
        [Reasignar cargo] (link)

   C) BODY GRID (2 columnas: 65% + 35%):

      MAIN COL (izquierda):
        - Card "Etapa actual" (eyebrow con circulo azul)
          - "Actividad: Aprobacion de Cliente"
          - Textarea readonly con descripcion del paso
        - Card "Worklog" (con timer inline):
          - PILL grande: [00:12:34] boton "Iniciar/Pausar"
          - Row: "Manual" con [Duracion (HH:MM)] + [Detalle textarea] + [+ Guardar]
          - "Total: 4h 32m"
          - Lista de entradas anteriores:
            [barra azul lateral] "2h - Cotizando con proveedor Lenovo" | usuario | fecha
            [barra azul lateral] "1h 30m - Espera respuesta cliente"    | usuario | fecha

      SIDEBAR COL (derecha):
        - Card "Resumen" con datos condensados
        - Card "Lista de chequeo":
          - Items con checkbox: [ ] Verificar precios | [x] Confirmar stock
          - [+ Agregar item]
        - Card "Adjuntos y seguimiento":
          - Lista de archivos con thumbnail
          - [+ Subir archivo]
          - Notas: textarea + [+ Agregar nota]

4) FOOTER:
   Izquierda: [Cerrar]
   Derecha:   [Volver a la lista]

INTERACCIONES:
- Click "Iniciar" en timer: cuenta segundos en vivo (setInterval)
- Click "+Guardar" del manual: crea entrada, resetea timer
- Click "Continuar ->": marca paso como TERMINADO, muestra toast "Paso avanzado",
  actualiza el flujo visual (proximo paso pasa a activo)
- Click "Rechazar": modal de confirmacion + campo motivo
- Click "Suspender": modal de motivo, cambia estado a SUSPENDIDO
- Anterior/Siguiente header: cambia todo el contenido al proximo caso
  (mantiene modal abierto)
- Color swatch: color picker HSL, guarda el color junto al caso

DATOS DEMO:
- Caso: T0042 - Cotizar 20 laptops Lenovo ThinkPad
- Cliente: ALPES TECHNOLOGY
- Encargado: ANDRES.M
- Estado: EN PROCESO
- Prioridad: ALTA
- Fecha entrega: 2026-07-15
- Proceso: COMERCIAL REQUERIMIENTO INFRAESTRUCTURA
- Pasos: Requerimiento (done), Cotizacion Proveedores (done), Aprobacion
  Cliente (active), Generar Factura, Entrega, Gestion pago, Recibir producto,
  Ingreso Alegra
- Worklog entradas: 3 con tiempos variados

STACK: HTML + Tailwind o CSS custom + JS vanilla. localStorage para persistir
timer y notas. Un solo archivo HTML.
```

---

## 10. Reglas de comportamiento criticas

1. **`FlagCerradoFijo="1"` deshabilita edicion completa** - solo lectura.
2. **`btnHeroAsignar` y `btnHeroReasignar` requieren `Asignar="1"`** (permiso `PERMITE_ASIGNAR`).
3. **Nav anterior/siguiente** requiere `PosicionNavegacion` no vacio - lo pobla el padre desde la lista de casos.
4. **Timer**: el `HiddenFieldTimerSeconds` es la fuente de verdad. Un `setInterval` de JS lo actualiza cada segundo.
5. **Al guardar worklog**: valida que `txtAtividad` no este vacio.
6. **Aprobacion**: solo aparece si el paso actual es un `Gateway` BPMN (`ID_ELEMENTO like '%Gateway%'`).
7. **Ciclos BPMN**: si el flujo tiene `ID_REINICIO`, la barra del proceso muestra la iteracion actual ("Ciclo 2").
8. **Color del caso**: se guarda en `TAR_SEGUIMIENTO_PROCESO.COLOR` (o similar) y se usa como border-left del kanban card.

---

## 11. Checklist para validar el rebuild

- [ ] 4 tabs (General, Avanzado, Aplicaciones, Archivos) funcionan
- [ ] Hero muestra 5 meta pills correctas
- [ ] Botones Anterior/Siguiente en header cambian el caso
- [ ] Flujo BPMN muestra pasos horizontales con estados visuales
- [ ] Timer inline cuenta segundos y se puede pausar
- [ ] Worklog acepta entradas manuales
- [ ] Lista de entradas muestra historial
- [ ] Sidebar tiene 3 cards (Resumen, Lista chequeo, Adjuntos)
- [ ] Color swatch permite cambiar color del caso
- [ ] Estado del caso editable con dropdown
- [ ] Aprobacion aparece cuando aplica
- [ ] Reasignacion abre panel deslizable
- [ ] Al continuar paso, actualiza flujo visual
- [ ] Modal responsivo en tablet

---

## Enlaces

- Vision maestra del destino: [[Visión y entorno]]
- Aspecto y navegacion definitivos: [[Visión y entorno|Prototipo Final ECOREX]]
- Control complementario (crear): [[ctrTareasII - Spec para reconstruir en Claude Design]]
- Paginas que lo consumen: [[Tareas y Proyectos - paginas basicas]]
- Motor BPMN al avanzar paso: [[Ejecucion - SiguienteEstado y Reinicios]]
- Historias de operativo: [[01 - Historias del Operativo]]
