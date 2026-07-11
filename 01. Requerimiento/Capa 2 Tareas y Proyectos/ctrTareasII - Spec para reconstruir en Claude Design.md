---
tipo: spec-reconstruccion
control: ctrTareasII (wizard modal de crear/editar tarea)
consumido_por: NEWFRONT_admtareas.aspx (000636), NEWFRONT_tar_tareas.aspx (000038), NEWFRONT_proyectos.aspx (000042)
archivo_origen: Bootstrap\Formularios\Modulos\Tareas\controles\ctrTareasII.ascx
tamanio: 2393 lineas markup + 1263 lineas code-behind
verificado_en_prod: 2026-07-01 app.bitcode.com.co usuario 80001976
---

# ctrTareasII - Spec para reconstruir en Claude Design

> **Modal wizard grande** (90vh) que se abre desde el boton "+ Nuevo Registro" del topbar
> en Administrar Actividades. Cubre **crear** y **editar** una tarea con toda su
> parametrizacion. Tiene su propio design system moderno (CSS variables, Inter,
> paleta clara).
>
> Datos verificados en produccion `app.bitcode.com.co` con usuario `80001976`.
>
> **Naturaleza de esta nota (ORIGEN + DESTINO):** las secciones 1 a 12 documentan
> la **UX del control ORIGEN** (WebForms) tal como opera hoy; se conservan como
> referencia de experiencia de usuario y catalogo de reglas de negocio del wizard
> de creacion. La seccion **0 (abajo)** fija la **VISION DESTINO**: como se
> reconstruye este wizard en **Blazor (.NET 10)** con el aspecto del prototipo
> final. Vision maestra en [[Visión y entorno]]; aspecto definitivo en
> [[Visión y entorno|Prototipo Final ECOREX]].

---

## 0. Vision destino - reconstruccion en Blazor (.NET 10)

En el destino, `ctrTareasII` deja de ser un `.ascx` con code-behind y postbacks y
se convierte en un **componente Blazor** `NuevaActividadDialog` (dialog modal
invocado desde la ruta `/tareas`). Se preserva la intencion de UX del wizard
(clasificacion -> contacto -> formulario -> documentos -> formacion) pero
alineada al estandar visual de [[Visión y entorno|Prototipo Final ECOREX]] (estilo
Linear/Height, tarjetas KPI, modo oscuro). Diferencias clave frente al origen:

1. **Cascadas sin postback**: los combos Empresa -> Tipo/Area -> Subcategoria ->
   Encargado se resuelven con llamadas asincronas al Task Service (o estado local
   Blazor), no con `AutoPostBack` + `UpdatePanel`. La respuesta es inmediata.
2. **Paso Formulario = `DynamicFormRenderer`**: el step condicional que hoy embebe
   `crtCargaEncuestaII` se reconstruye con el motor de formularios portado
   (patron EAV), que renderiza la definicion asociada al nodo BPMN inicial del
   proceso. Ver [[Visión y entorno]] seccion 8.
3. **Instanciacion de flujo transaccional**: al guardar, en **una sola
   transaccion** se crea la tarea (`tasks`), su instancia de flujo
   (`WorkflowEngine.StartInstance`, antes `AdmWorkflow.GuardarTablaSeguimiento`),
   los adjuntos y se dispara la notificacion. Si algo falla, rollback total (se
   corrige el error heredado de operaciones multi-tabla sin transaccion).
4. **Multi-tenant real**: `cmbEmpresa` opera sobre tenants aislados por
   `TenantId` + `HasQueryFilter` + RLS; el combo solo lista los tenants que el
   usuario puede ver por policy, no todos por disciplina. Ver
   [[Gestion de Empresas - Admin multi-tenant]].
5. **Adjuntos a object storage**: el dropzone `TDFileUpload` se reconstruye con
   `InputFile` Blazor + subida a Object Storage; metadatos cifrados y auditados.
6. **Consecutivo y permisos**: `A06` lo genera un servicio de consecutivos
   transaccional por tenant; los `PERMITE_*` (Asignar, VerTodo, proyectos) son
   **policies .NET** evaluadas con `<AuthorizeView>`, no propiedades string.
7. **Encabezado del prototipo**: al reconstruir, la cabecera adopta el patron del
   prototipo final (Estado, Asignados, Fecha limite, Etiquetas) cuando el wizard
   se usa en contexto de proyecto; el "Modo Visualizacion" (Actividades /
   Procesos / Todos) se conserva como control de densidad de campos.

> El bloque de paleta CSS y el prompt de Claude Design (secciones 3 y 10) se
> conservan porque **capturan el lenguaje visual objetivo**; en el destino ese
> lenguaje se implementa con los tokens del design system Blazor del prototipo,
> no como CSS suelto en el componente. Todo lo demas (secciones 1-12) es la
> referencia de comportamiento a reproducir.

---

## 1. Identidad del control

| Campo | Valor |
|---|---|
| Path | `Bootstrap\Formularios\Modulos\Tareas\controles\ctrTareasII.ascx` |
| Clase | `GestionMovil.ctrTareasII` |
| Modo | UserControl (`.ascx`) - se embebe como componente hijo |
| Se abre desde | `NEWFRONT_admtareas.aspx` (000636), `NEWFRONT_tar_tareas.aspx` (000038), `NEWFRONT_proyectos.aspx` (000042) |
| Tamanio modal | `width: 100%; height: 90vh` en desktop |
| Design system | Inter font + paleta clara + CSS variables propias (NO Bootstrap 4/Metronic) |
| Consecutivo | `A06` (via `Funciones.tipdoc.Consecutivo`) |
| Tabla destino | `SISTEMA` (cabecera del caso) + `TAR_SEGUIMIENTO_PROCESO` (motor BPMN) |

---

## 2. Proposito en una frase

Wizard modal de 5 pasos para **crear una nueva actividad** con toda su parametrizacion
(clasificacion, contacto, formulario dinamico opcional, adjuntos, formacion) que
instancia el flujo BPMN correspondiente en `TAR_SEGUIMIENTO_PROCESO`.

---

## 3. Paleta de diseno REAL (extraida del `.ascx`)

Copiar tal cual como CSS variables:

```css
:root {
  /* Fondos */
  --bg-page: #F4F5F7;
  --bg-modal: #FFFFFF;
  --bg-card: #FFFFFF;
  --bg-soft: #F9FAFB;
  --bg-soft-2: #F3F4F6;

  /* Bordes */
  --border: #E5E7EB;
  --border-md: #D1D5DB;
  --border-strong: #9CA3AF;

  /* Texto */
  --text: #111827;
  --text2: #4B5563;
  --text3: #9CA3AF;
  --text-muted: #6B7280;

  /* Azul principal (identidad) */
  --accent: #2563EB;
  --accent-hover: #1D4ED8;
  --accent-light: #EFF6FF;
  --accent-soft: #DBEAFE;
  --accent-border: #BFDBFE;

  /* Verde (exito) */
  --green: #16A34A;
  --green-light: #F0FDF4;
  --green-border: #BBF7D0;

  /* Ambar (advertencia) */
  --amber: #D97706;
  --amber-light: #FFFBEB;
  --amber-border: #FDE68A;

  /* Naranja (destacado) */
  --orange: #EA580C;
  --orange-light: #FFF7ED;
  --orange-border: #FED7AA;

  /* Rojo (error) */
  --red: #DC2626;
  --red-light: #FEF2F2;
  --red-border: #FECACA;

  /* Border radius escala */
  --r-sm: 6px; --r: 8px; --r-md: 10px; --r-lg: 12px; --r-xl: 16px; --r-2xl: 20px;

  /* Sombras */
  --shadow-sm: 0 1px 2px rgba(16,24,40,0.04);
  --shadow: 0 1px 3px rgba(16,24,40,0.06), 0 1px 2px rgba(16,24,40,0.04);
  --shadow-md: 0 4px 8px -2px rgba(16,24,40,0.06), 0 2px 4px -2px rgba(16,24,40,0.04);

  /* Tipografia */
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}
```

---

## 4. Layout visual (mapa ASCII del modal)

```
+==========================================================================+
| MODAL (90vh, border-radius 20px, sombra sutil)                           |
+==========================================================================+
| MODAL-HEADER (padding 20/28px)                                           |
|                                                                          |
| +------------------+                            +---------+  +---+       |
| | [icon] BITCODE / | +-----------------------+  | Modo    |  | X |       |
| |        Actividades| | Nueva actividad      |  | Actividades v |       |
| | (brand-crumb)    | | (brand-title 15px)    |  +---------+  +---+       |
| +------------------+ +-----------------------+                           |
+==========================================================================+
| STEPS-NAV (barra pasos)                                                  |
|                                                                          |
| (1) Información --- (2) Contacto --- (3) Formulario --- (3) Documentos   |
|                                                                          |
| Nota: hay pasos condicionales:                                           |
|   - Paso "Formulario" solo aparece si hay componentes/plugin asociado   |
|   - Paso "Formación" solo aparece si el tipo tiene modulos de formacion|
+==========================================================================+
| MODAL-BODY (scroll interno, gap grande entre secciones)                  |
|                                                                          |
| == STEP 1: Información (pg1) ==                                          |
|                                                                          |
|   +------------------------------------------------------------------+  |
|   | [icon] SECCION: Clasificacion y asignacion                       |  |
|   | Define a que proceso pertenece la actividad, quien la atiende... |  |
|   |                                                                  |  |
|   | [Empresa / Area v] [Proceso / Tipo v] [Actividad v]              |  |
|   | (Multi-tenant)     (DIRECCION COMER)   (subcategoria)             |  |
|   |                                                                  |  |
|   | [Encargado v]      [Fecha de entrega _____]                      |  |
|   | (dropdown)         (datepicker con hoy = default)                 |  |
|   +------------------------------------------------------------------+  |
|                                                                          |
|   +------------------------------------------------------------------+  |
|   | [icon] SECCION: Descripcion de la actividad                      |  |
|   | Detalla que se va a hacer y el contexto necesario para ejecutarla|  |
|   |                                                                  |  |
|   | [Titulo de la actividad ____________________________________]    |  |
|   | [Descripcion / Detalle                                        ]  |  |
|   | [(textarea 4 filas)                                          ]  |  |
|   +------------------------------------------------------------------+  |
|                                                                          |
|   +------------------------------------------------------------------+  |
|   | [icon] SECCION: Prioridad                                        |  |
|   | Que tan urgente es atender esta actividad?                       |  |
|   |                                                                  |  |
|   | [chips coloridos]  ALTA   MEDIA   BAJA                           |  |
|   | (radio buttons con estilo chip)                                  |  |
|   +------------------------------------------------------------------+  |
|                                                                          |
|   +------------------------------------------------------------------+  |
|   | [icon] SECCION: Etiquetas / Categorias                           |  |
|   | Marca las etiquetas que apliquen                                 |  |
|   |                                                                  |  |
|   | [tag input con chips: #proveedor #urgente ...]                   |  |
|   +------------------------------------------------------------------+  |
|                                                                          |
| == STEP 2: Contacto (pg2) ==                                             |
|                                                                          |
|   +------------------------------------------------------------------+  |
|   | [icon] SECCION: Informacion del contacto                         |  |
|   |                                                                  |  |
|   | [Nombre completo solicitante ___________________]                |  |
|   | [Identificacion ______]  [Email _______________]                 |  |
|   | [Telefono ___________]                                           |  |
|   | [Numero de actividad] (auto-consecutivo)                         |  |
|   | [Proyecto v]  [Hito Proyecto v]                                  |  |
|   +------------------------------------------------------------------+  |
|                                                                          |
|   +------------------------------------------------------------------+  |
|   | [icon] SECCION: Copiar a correos                                 |  |
|   |                                                                  |  |
|   | [email chips input: [email protected] x  [email protected] x  +]           |  |
|   +------------------------------------------------------------------+  |
|                                                                          |
| == STEP 3 (condicional): Formulario dinamico (pgP) ==                    |
|                                                                          |
|   +------------------------------------------------------------------+  |
|   | Renderer crtCargaEncuestaII (formulario ENCUESTAS_MOV asociado   |  |
|   | al proceso/subcategoria)                                          |  |
|   +------------------------------------------------------------------+  |
|                                                                          |
| == STEP 3/4: Documentos (pg3) ==                                         |
|                                                                          |
|   +------------------------------------------------------------------+  |
|   | [icon] SECCION: Archivos adjuntos                                |  |
|   |                                                                  |  |
|   | [Concepto Archivo v]                                             |  |
|   | +------------------------------------------------------------+   |  |
|   | | DROPZONE: Arrastra tus archivos aqui o click para elegir  |   |  |
|   | | (TDFileUpload control)                                    |   |  |
|   | +------------------------------------------------------------+   |  |
|   | Preview de archivos ya cargados con thumbnails                    |  |
|   +------------------------------------------------------------------+  |
|                                                                          |
| == STEP OPCIONAL: Formacion (pgF) ==                                     |
|                                                                          |
|   +------------------------------------------------------------------+  |
|   | [icon] SECCION: Formacion y ayudas                               |  |
|   | ctrImodulosformacion (videos/tutoriales asociados al proceso)    |  |
|   +------------------------------------------------------------------+  |
|                                                                          |
+==========================================================================+
| MODAL-FOOTER (sticky, borde superior)                                    |
|                                                                          |
| [<- Atras]  [Nuevo]  [Cancelar]      [Guardar y crear otra] [Guardar    |
|                                                              actividad]  |
|                                                              [Siguiente->]
+==========================================================================+
```

---

## 5. Componentes ASP.NET originales por seccion

### Header
| Control | ID | Valor tipico |
|---|---|---|
| Label empresa | `lblHeaderEmpresa` | "BITCODE / Actividades" |
| Combo modo vista | `cmbModoVisualizacion` | "Actividades" / "Procesos" / "Todos" (afecta que campos se muestran) |

### Step 1: Clasificacion y asignacion
| Control | ID | Bind SQL |
|---|---|---|
| Combo Empresa/Area | `cmbEmpresa` | `SUCURSAL` (multi-tenant) |
| Combo Tipo | `cmbTipo` | `TIPO_TAR_R.CATEGORIA` (Direccion Comercial, GH, Financiera, General) |
| Combo Subcategoria | `cmbsubcategoria` | `TIPO_TAR_R.CODIGO` (dispara cascada al proceso BPMN) |
| Combo Cliente | `Droplistaclientes` | `SUCURSAL` filtrada por tipo cliente |
| Combo Encargado | `cmbEncargado` | `USUARIO` (filtrado por permiso PERMITE_ASIGNAR + fn_ConsultaCargo del cargo asociado al nodo BPMN) |
| Combo Busqueda encargado | `cmbBusquedaEncargado` | filtro tipo autocompletado sobre USUARIO |
| Fecha entrega | `txtFEntrega` | Date input, default = hoy |

### Step 1: Descripcion
| Control | ID | Notas |
|---|---|---|
| Titulo | `txttitulo` | required, va a `SISTEMA.TITULO` |
| Observacion | `txtObservacion` | textarea 4 filas, va a `SISTEMA.OBSERVACION` |

### Step 1: Prioridad
| Control | ID | Comportamiento |
|---|---|---|
| Hidden field prioridad | `HiddenFieldPrioridad` | Guarda la seleccion (ALTA/MEDIA/BAJA), UI son chips clickables (no radio nativo) |

### Step 1: Etiquetas
| Control | ID | Comportamiento |
|---|---|---|
| Hidden field tags | `HiddenFieldTags` | Chips input estilo tag - guarda como CSV |

### Step 2: Contacto
| Control | ID | Notas |
|---|---|---|
| Panel manejo encargado | `PanelManejoencargado` | Sub-panel condicional segun tipo |
| Panel pedir usuario | `PanelPedirusuario` | Aparece solo si `SOLICITAR_TELS_Y_USUARIO="1"` |
| Nombre solicitante | `txtnombresolicitante`, `txtnombre`, `txtusuario` | 3 campos redundantes segun modo |
| Email | `txtmail` | Con autocompletado desde USUARIO/clientes |
| Telefono | `txtcontacto` | |
| Numero actividad | `txtcaso` | Consecutivo automatico o manual editable |
| Combo proyecto | `cmbProyecto` | `DOC_PROYECTOS.CODIGO` filtrado por permisos |
| Combo Hito proyecto | `cmbHitoProyecto` | Filtrado por proyecto seleccionado |
| Hidden emails CC | `HiddenFieldEmailsCC` | Chips CSV |

### Step 3: Formulario dinamico (opcional)
Se muestra si `IdFlujoProceso <> ""` y el nodo BPMN inicial tiene componente asociado.

Renderer: `<uc1:crtCargaEncuestaII />` con propiedades pobladas del proceso/subcategoria.

Hidden `HiddenFieldPluginTitulo` guarda el titulo dinamico del step.

### Step 3/4: Documentos
| Control | Origen | Notas |
|---|---|---|
| `TDFileUpload` | `~/Controles/TDFileUpload.ascx` | Dropzone estilo moderno con thumbnails |
| `ctrlDocumentosCentral` | `~/Formularios/Modulos/Documentos/controles/` | Documentos centrales del sistema |
| `CtrlArchivoDirectorios` | `~/Formularios/Modulos/Documentos/controles/` | Navegador de directorios |
| Hidden archivo | `HiddenFieldArchivo` | Recibe eventos de upload y dispara postback |

### Step opcional: Formacion
Renderer: `<uc1:ctrImodulosformacion />` - lista modulos de formacion asociados al proceso.

---

## 6. Propiedades publicas del control (25 props)

Estas son las que el padre `.aspx` inyecta para configurar el modal:

```vb
Public Property IdTempActividad() As String        ' Temporal, se resuelve al guardar
Public Property SOLICITAR_TELS_Y_USUARIO() As String  ' "1" muestra PanelPedirusuario
Public Property Modulo() As String                  ' 000038 / 000636 / 000042
Public Property XMLTareaNueva() As String           ' Plantilla XML de nueva tarea (GEN_PARAMETROS)
Public Property XMLTareaNotificacion() As String    ' Plantilla del mail (GEN_PARAMETROS)
Public Property TareaSoporte() As String            ' "1" para sucursal 01 (3DEV)
Public Property Sucursal() As String
Public Property Proyecto() As String                ' Codigo del proyecto asociado
Public Property CargarArchivos() As String
Public Property Asignar() As String                 ' "1" si tiene PERMITE_ASIGNAR
Public Property VerTodo() As String                 ' "1" si tiene PERMITE_VER_TODAS_LAS_ACTIVIDADES
Public Property man_categoria As String             ' Preseleccion via ?cat=
Public Property man_subcategoria As String          ' Preseleccion via ?sub=
Public Property PERMITE_VER_MIS_PROYECTOS() As String
Public Property PERMITE_VER_PROYECTOS_TODOS() As String
Public Property SOLO_PROYECTOS_DE_ORGANIZACION() As String
Public Property AREAS_CLIENTES() As String
Public Property IdFlujoProceso() As String           ' Si viene de un proceso BPMN
Public Property IdProceso() As String
Public Property Usuario() As String

' Evento que dispara al guardar exitoso
Public Event Addtar As EventHandler
```

---

## 7. Metodos publicos clave (code-behind)

| Metodo | Cuando se llama | Que hace |
|---|---|---|
| `ListarClientes()` | Al cambiar empresa | Recarga `Droplistaclientes` filtrado por sucursal |
| `ListarProyectos()` | Al cargar y al cambiar permisos | Combo proyectos filtrado por (PERMITE_VER_MIS_PROYECTOS + PROYECTOS_RES) |
| `ListaTiposGrupos(sql_filtro, sql_sucursal)` | Al cambiar empresa | Recarga cmbTipo con las areas de la empresa |
| `CargarBusquedaEncargado()` | Al cambiar tipo/subcategoria | Recarga cmbEncargado filtrado por cargo asociado al nodo BPMN |
| `DatosUsuarioSucursal()` | Al cambiar usuario | Autocompletado de datos del usuario (email, telefono) |
| `DatosUsuario()` | Al cambiar solicitante | Similar |
| `ValidaComponentes() As Boolean` | Antes de guardar | Valida campos obligatorios, muestra errores por seccion |
| `NofificarCaso(caso)` | Tras guardar | Envia mail con XMLTareaNotificacion |
| `NofificarCasoCliente(caso)` | Tras guardar | Mail al cliente si aplica |
| `NofificarAsignado(caso, mensaje, anotacion)` | Cuando cambia encargado | Notifica al nuevo asignado por SignalR + mail |
| `Limpiar(tipo)` | Al cancelar o "Nuevo" | Resetea campos segun tipo (1=todo, 0=solo mantiene sucursal) |

---

## 8. Eventos y cascadas (order matters)

```
Usuario abre modal
   |
   v
Page_Load                    <- carga combos, aplica XMLTareaNueva default
   |
   v
Usuario cambia cmbEmpresa    <- ListarClientes + ListaTiposGrupos + ListarProyectos
   |
   v
Usuario cambia cmbTipo       <- Filtra cmbsubcategoria
   |
   v
Usuario cambia cmbsubcategoria (000477 etc) <- CargarBusquedaEncargado + resuelve IdProceso via TIPO_TAR_R_PRO
   |                                            + Si el proceso tiene componente, muestra step "Formulario"
   |                                            + Si tiene formacion, muestra step "Formacion"
   v
Usuario cambia cmbProyecto   <- Filtra cmbHitoProyecto
   |
   v
Usuario cambia txtnombre     <- DatosUsuario (autocompletado)
   |
   v
Usuario sube archivo         <- HiddenFieldArchivo_ValueChanged (via TDFileUpload)
   |
   v
Usuario click "Siguiente"    <- ValidaComponentes(step_actual) - si OK avanza wzGoToTarget('pgN')
   |
   v
Usuario click "Guardar actividad"
   |
   v
Guardar completo:
   - INSERT SISTEMA (cabecera del caso)
   - Funciones.tipdoc.Consecutivo("A06") si txtcaso vacio
   - AdmWorkflow.GuardarTablaSeguimiento(sucursal, caso, categoria, subcategoria)
   - Insert archivos en GEN_ARCHIVOS con REFERENCIA=caso
   - NofificarCaso + NofificarAsignado
   - RaiseEvent Addtar (el padre recarga la lista)
   - Cerrar modal
```

---

## 9. Datos REALES capturados en produccion

**Combos observados en `app.bitcode.com.co`:**

| Combo | Primeras opciones |
|---|---|
| Empresa | Selecciona la Empresa \| AMIGOS DE 4 PATAS SAS \| ANDINA \| APLOGISTICA \| ARCHIMENTE |
| Tipo | Selecciona Tipo de Solicitud \| DIRECCION COMERCIAL \| DIRECCION DE GESTION HUMANA \| DIRECCION FINANCIERA \| DIRECCION GENERAL |
| Modo vista | Actividades \| Procesos \| Todos |

**Metricas del render:**
- 22 inputs totales en el DOM
- 8 secciones visibles simultaneamente (todas en scroll)
- 5 tabs de wizard (2 condicionales)
- 6 botones en footer

**Header confirmado:**
- Titulo: **"Nueva actividad"** (al crear) o **"Editando actividad T0042"** (al editar - hipotetico)
- Breadcrumb: **"BITCODE / Actividades"**

---

## 10. Prompt directo para Claude Design / Artifact

> Copia-pega para regenerar el modal desde cero:

```
Construye un WIZARD MODAL grande (90vh, max 1400px de ancho, border-radius 20px)
para crear una nueva actividad/tarea. Estetica moderna tipo Linear/Notion, tema
claro, tipografia Inter.

USA ESTAS CSS VARIABLES:
--bg-modal: #FFFFFF
--bg-soft: #F9FAFB
--border: #E5E7EB
--text: #111827
--text2: #4B5563
--text3: #9CA3AF
--accent: #2563EB (azul primary)
--green: #16A34A
--amber: #D97706
--red: #DC2626
--r: 8px  (border-radius base)
--r-2xl: 20px  (border-radius del modal)
--shadow-sm: 0 1px 2px rgba(16,24,40,0.04)

ESTRUCTURA (top-down):

1) HEADER (padding 20px 28px, borde inferior):
   - Izquierda: icono pill azul + breadcrumb pequeno "BITCODE / Actividades"
     y titulo "Nueva actividad" (font 15px bold)
   - Centro: nada
   - Derecha: dropdown pill "Modo: Actividades v" (opciones: Actividades,
     Procesos, Todos) + boton cerrar (X, 32x32px, border sutil)

2) STEPS NAV (barra de pasos, 5 pasos):
   Paso activo: numero en pill azul, label bold negro
   Paso inactivo: numero en pill gris, label gris
   Separadores: linea horizontal 20px
   Pasos: (1) Informacion  (2) Contacto  (3) Formulario*  (3) Documentos
          (4) Formacion*
   * Estos 2 son OPCIONALES (aparecen segun tipo de tarea seleccionado)

3) BODY (scroll interno, padding 24px):

   TAB "Informacion" tiene 4 secciones (cards con icono + titulo + subtitulo):

   SECCION "Clasificacion y asignacion" (icono briefcase azul)
     - Subtitulo: "Define a que proceso pertenece la actividad, quien la
       atiende y para cuando"
     - Row 1 (3 cols): [Empresa/Area v] [Proceso/Tipo v] [Actividad v]
       Empresa: dropdown con "Selecciona la Empresa", "ANDINA", "APLOGISTICA",
                "AMIGOS DE 4 PATAS SAS", etc (~50 tenants)
       Tipo: dropdown con "DIRECCION COMERCIAL", "DIRECCION GESTION HUMANA",
             "DIRECCION FINANCIERA", "DIRECCION GENERAL"
       Actividad: dropdown de subcategoria (cascada del Tipo)
     - Row 2 (2 cols): [Encargado v]  [Fecha de entrega _date_]

   SECCION "Descripcion de la actividad" (icono edit lapiz)
     - Subtitulo: "Detalla que se va a hacer y el contexto necesario para
       ejecutarla"
     - Titulo (input full width)
     - Descripcion / Detalle (textarea 4 filas full width)

   SECCION "Prioridad" (icono flag)
     - Subtitulo: "Que tan urgente es atender esta actividad?"
     - 3 CHIPS clickables como radio: ALTA (rojo pastel), MEDIA (ambar pastel),
       BAJA (verde pastel)
     - Al click el chip se marca con anillo azul (accent-border)

   SECCION "Etiquetas / Categorias" (icono tag)
     - Chips input con placeholder "Escribe una etiqueta y presiona Enter"
     - Chips existentes son azul soft con X

   TAB "Contacto" tiene 2 secciones:

   SECCION "Informacion del contacto"
     - Row 1 (full): Nombre completo solicitante
     - Row 2 (2 cols): [Identificacion] [Email]
     - Row 3 (2 cols): [Telefono] [Numero de actividad (auto-consecutivo A06)]
     - Row 4 (2 cols): [Proyecto v] [Hito Proyecto v]

   SECCION "Copiar a correos"
     - Chips input de emails con boton "+ Agregar"

   TAB "Formulario" (SOLO si el tipo tiene formulario asociado):
     - Renderer de formulario dinamico. Muestra los 19 tipos de control
       (Texto, Lista, Fecha, Firma, etc.) segun definicion en ENCUESTAS_MOV
       del proceso BPMN.

   TAB "Documentos":
     - SECCION "Archivos adjuntos"
       - Combo "Concepto Archivo v"
       - DROPZONE grande (border dashed) con texto "Arrastra tus archivos aqui
         o click para elegir"
       - Cuando hay archivos, mostrar grid de thumbnails con nombre, tamano,
         boton X para quitar

   TAB "Formacion" (SOLO si aplica):
     - Lista de tarjetas de video/tutorial asociadas al proceso

4) FOOTER (sticky bottom, padding 16px 28px, border top):
   Izquierda: [<- Atras] (ghost, aparece desde step 2)
              [Nuevo] (limpia el form)
              [Cancelar] (cierra modal)
   Derecha:   [Guardar y crear otra] (outline)
              [Guardar actividad] (primary azul)
              [Siguiente ->] (primary azul, en steps 1-3)

COMPORTAMIENTO:
- Al cambiar Empresa: recargar combos dependientes
- Al cambiar Tipo -> filtrar Actividad (subcategoria)
- Al cambiar Actividad -> resolver Encargado por cargo, decidir si mostrar
  tabs opcionales Formulario/Formacion
- Al cambiar Proyecto -> filtrar Hito Proyecto
- Prioridad: click chip marca uno solo (radio), guarda en hidden
- Etiquetas: Enter agrega chip, X del chip lo quita
- Emails CC: mismo patron chips
- Fecha entrega: default a hoy, no permite fechas pasadas
- Consecutivo Numero actividad: se autocompleta si vacio (formato T0042)
- Al guardar: validar Titulo, Empresa, Tipo, Actividad, Encargado (obligatorios)
  y mostrar errores por seccion (no toast global)

DATOS DEMO para el prototipo:
Empresas: ANDINA, APLOGISTICA, AMIGOS DE 4 PATAS SAS, ARCHIMENTE, BITCODE
Tipos:    DIRECCION COMERCIAL, DIRECCION GESTION HUMANA, DIRECCION FINANCIERA
Actividades ejemplo bajo COMERCIAL: "Cotizacion", "Seguimiento cliente",
                                    "Requerimiento infraestructura"
Encargados: JUAN.PEREZ, MARIA.GOMEZ, ANDRES.MARTINEZ

STACK: HTML + Tailwind CSS o CSS vanilla + JS vanilla. Persistir en localStorage.
Un solo archivo HTML autocontenido.
```

---

## 11. Reglas de comportamiento criticas

1. **Steps 3 y 4 son OPCIONALES** - se muestran/ocultan con clase `step-plugins-only` y `step-formacion-only` segun la subcategoria seleccionada dispare el proceso BPMN que los tenga configurados.
2. **Cambio de "Modo Visualizacion"** (dropdown superior derecho) altera que campos se ven: "Actividades" muestra el flow normal, "Procesos" oculta contacto, "Todos" muestra todo sin filtro.
3. **cmbEmpresa** en produccion tiene 50+ tenants (multi-tenant real).
4. **cmbTipo** son las **direcciones/areas** de la empresa seleccionada. Cambian por empresa.
5. **cmbEncargado** se RESUELVE del cargo asignado al nodo BPMN del proceso, via UDF `fn_ConsultaCargo`.
6. **`Modo` recomendado por default**: `Actividades` (la pantalla mas usada).
7. **Al guardar**, si `SOLICITAR_TELS_Y_USUARIO="1"`, valida ademas `PanelPedirusuario`.
8. **`AREAS_CLIENTES`** cuando esta activa, filtra los combos por areas de cliente (soporte 3DEV).
9. **Modal NO se cierra al click fuera** - solo con X o Cancelar (data-backdrop="static").

---

## 12. Checklist para validar el rebuild

- [ ] Modal ocupa 90vh, tiene border-radius 20px, no scroll externo
- [ ] Header con logo + crumb + titulo + modo select + close
- [ ] Steps nav muestra 3 fijos + 2 condicionales
- [ ] Todas las secciones tienen icono + titulo + subtitulo
- [ ] Combos Empresa/Tipo/Subcategoria hacen cascada correcta
- [ ] Prioridad son chips con colores rojo/ambar/verde
- [ ] Etiquetas es chips input con Enter/X
- [ ] Emails CC igual
- [ ] Fecha entrega valida no-pasado
- [ ] Dropzone acepta drag&drop y click
- [ ] Botones footer con orden y colores correctos
- [ ] Al presionar Siguiente valida el step y avanza
- [ ] Al Guardar hace INSERT y RaiseEvent Addtar
- [ ] Modal no se cierra por click fuera
- [ ] Responsivo hasta tablet (colapsa cols)
- [ ] Font Inter cargada
- [ ] Sombras y bordes sutiles (no aggressive)

---

## Enlaces

- Vision maestra del destino: [[Visión y entorno]]
- Aspecto y navegacion definitivos: [[Visión y entorno|Prototipo Final ECOREX]]
- Ver control complementario: [[ctrVertareasII - Spec para reconstruir en Claude Design]]
- Paginas que lo consumen: [[Tareas y Proyectos - paginas basicas]]
- Motor de formularios destino: [[00 - Visión Formularios]]
- Motor BPMN al guardar: [[Ejecucion - SiguienteEstado y Reinicios]]
