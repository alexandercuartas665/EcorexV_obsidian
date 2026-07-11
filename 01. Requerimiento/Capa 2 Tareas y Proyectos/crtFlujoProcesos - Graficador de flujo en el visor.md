---
tipo: documentacion-control
control: crtFlujoProcesos (graficador + editor del flujo del proceso)
modulo: 000291 DOC_PROCESOS
host: NEWFRONT_doc_procesos.aspx
archivo_origen: Bootstrap\Formularios\Modulos\Documentos\controles\crtFlujoProcesos.ascx
tamanio: 1074 lineas markup + 1161 lineas code-behind
motor_grafico: bpmn-js@18.7.0 (BpmnJS / BpmnModeler)
verificado_en_codigo: 2026-07-11
---

# crtFlujoProcesos - Graficador de flujo en el visor

> **Control que dibuja el diagrama del flujo de un proceso** y expone un panel
> lateral (inspector) para configurar cada nodo: nombre, reglas, componentes,
> documentos, usuarios asignados y permisos por cargo. Se apoya en el motor BPMN
> del host `NEWFRONT_doc_procesos.aspx` (modulo 000291 `DOC_PROCESOS`).
>
> **CORRECCION IMPORTANTE SOBRE LA TECNOLOGIA.** El encargo describia esta
> herramienta como "HTML5 Canvas, sin bpmn-js". El codigo real dice lo contrario:
> el `<div id="canvas">` no es un elemento `<canvas>` de HTML5, es el contenedor
> DOM donde se monta **bpmn-js version 18.7.0** (`new BpmnJS(...)`). El graficado
> se hace con la libreria bpmn.io renderizando **SVG**, no con dibujo manual sobre
> Canvas. Los helpers `openDiagram`, `restartbmpnModeler`, `exportDiagram` y la
> variable `diagramUrl` viven en `Bootstrap\Frontend4\js\plugins\bpmnio\bpmncontrol.js`.
> Esta nota documenta la realidad del codigo. Ver [[Portabilidad BPMN - prueba en bpmn.io]].
>
> **Naturaleza de la nota (ORIGEN vs DESTINO).** Las secciones 1 a 10 documentan
> el control ORIGEN (WebForms + bpmn-js embebido, tal como opera hoy). La seccion
> 11 fija el DESTINO en Blazor (.NET 10). Vision maestra en [[Visión y entorno]];
> aspecto en el prototipo visual (`01. Requerimiento/Prototipo/ECOREX.dc.html`); motor de ejecucion en
> [[AdmWorkflow - Motor de flujo de la tarea]]; teoria de flujos en [[00 - Visión Flujos]].

---

## 1. Que es y donde aparece

`crtFlujoProcesos.ascx` es un user control ASP.NET WebForms (clase
`GestionMovil.crtFlujoProcesos`, namespace raiz). Se registra e incrusta en el
host `NEWFRONT_doc_procesos.aspx` (linea 953 del host: `<uc1:crtFlujoProcesos ... />`),
la pagina del modulo `000291 DOC_PROCESOS` donde se construye la definicion del
proceso. Su mitad izquierda es el lienzo del diagrama (`<div id="canvas">`, alto
800px), y su mitad derecha es el inspector `#bs-canvas-right` ("Detalle de
Actividad"), separados por un divisor arrastrable (`flow-pro-resizer`).

Conviene desmentir dos supuestos del encargo. Primero, no es un lienzo Canvas: es
bpmn-js (ver seccion 2). Segundo, no es solo "visor" de solo lectura: el control
tiene CRUD completo de nodos (crear, renombrar, eliminar) y persiste el XML del
diagrama. El "editor de Capa 3" (`NEWFRONT_doc_procesos`) y este control **son la
misma pagina**: `crtFlujoProcesos` es el cuerpo de esa pagina. La diferencia con
[[ctrVertareasII - Spec para reconstruir en Claude Design]] si es real: aquel es
el visor donde el operador **trabaja un caso** (una instancia viva de la tarea, con
worklog y avance), mientras que `crtFlujoProcesos` es donde se **disena la
plantilla** del proceso (los nodos, sus reglas y asignaciones). El resaltado del
"nodo actual" que el operador ve en su tarea corresponde a `ctrVertareasII`, no a
este control.

## 2. Tecnologia de graficado: bpmn-js embebido (no Canvas manual)

El motor es **bpmn-js@18.7.0**, cargado por el host desde unpkg
(`bpmn-modeler.development.js` mas los CSS `diagram-js.css`, `bpmn-js.css`,
`bpmn.css`). La instancia se crea en `bpmncontrol.js`:

```
var bpmnModeler = new BpmnJS({
    container: document.querySelector('#canvas'),
    additionalModules: [ window.BpmnJSColorPicker, window.MiModuloPaletteBootstrap ]
});
```

O sea: el `#canvas` de este ascx es solo el hueco DOM. bpmn-js dibuja dentro de el
un **SVG** con las figuras BPMN estandar (tareas, gateways, eventos, flujos de
secuencia, anotaciones). No hay codigo que calcule coordenadas ni pinte aristas a
mano; de eso se encarga diagram-js (el motor de bpmn-js) con su propio layout y su
paleta de herramientas lateral. Dos modulos adicionales personalizan la
experiencia: `BpmnJSColorPicker` (context pad de color) y
`MiModuloPaletteBootstrap`, que inyecta en la paleta un dropdown "Ver
configuracion" que abre el inspector `#bs-canvas-right`.

La **interaccion clic-en-nodo** se cablea en `openDiagram(bpmnXML)`, que tras
`bpmnModeler.importXML(bpmnXML)` registra cuatro eventos de bpmn-js y los traduce a
postbacks de WebForms escribiendo campos ocultos:

- `element.click`: si el elemento no es `SequenceFlow`/`Process`/`label`, escribe
  `HiddenSelectedName`, `txtnombreactividad`, `HiddenParent`, `HiddenId` y dispara
  `__doPostBack(HiddenId)`.
- `shape.added`: escribe `HiddenParent` + `HiddenCreate` y hace postback (nodo nuevo).
- `shape.remove`: escribe `HiddenParent` + `HiddenDelete` y hace postback (nodo borrado).
- `element.changed`: actualiza `HiddenParent` y llama `exportDiagram()`.

`exportDiagram()` hace `bpmnModeler.saveXML({format:true})`, vuelca el XML en
`HiddenXML` y dispara `__doPostBack(HiddenXML)` para que el servidor lo persista.
`restartbmpnModeler()` destruye y recrea el modeler (limpiar lienzo). Un detalle
fragil: el id del nodo se normaliza con `id.replace("_","-")` en JS, y en el
servidor las consultas SQL vuelven a quitar `-` y `_` para casar el id del XML con
`DOC_PROCESOS_R.ID_ELEMENTO`. Ademas hay botones flotantes (abajo a la izquierda del
canvas) para exportar SVG (`saveSVG`), exportar `.bpmn` y reimportar un `.bpmn/.xml`.

## 3. Fuente de datos del diagrama

El diagrama vive en **dos representaciones sincronizadas**:

1. **El XML BPMN completo**, guardado como blob en `DOC_PROCESOS_G.GRAFICO` (una
   fila por proceso: `SUCURSAL` + `PROCESO`). Es lo que bpmn-js importa y exporta.
2. **La tabla normalizada de nodos** `DOC_PROCESOS_R` (una fila por elemento:
   `ID_ELEMENTO`, `ID_ELEMENTO_PADRE`, `NOMBRE`, `DETALLE`, `TIPO_BPMN`, `ID_BPMN`,
   `FLUJO`, `ID_REINICIO`, `PERMITE_ASIGNACION`, `PASO`, `ORIGEN`). Es lo que
   consultan todos los paneles del inspector.

El XML no llega inline: el JS lo pide por AJAX a un web service. En `bpmncontrol.js`
`diagramUrl = '../../../Servicios/sweb_flujoxml.asmx/FlujoXML?Code='` y al cargar se
ejecuta `$.get(diagramUrl, openDiagram, 'text')`. Del lado servidor, el metodo
`DibujarDiagrama(ParamXML)` registra ese mismo GET con el codigo del proceso
anexado:

```
$.get(diagramUrl+'<ParamXML>', openDiagram, 'text');
```

donde `ParamXML` suele ser `Me.RegProceso`. Es decir, el servidor no serializa el
grafo: solo le dice al navegador que pida al ASMX el XML del proceso y lo importe.
Existe una plantilla de "grafico base" (diagrama vacio) en
`[dbx.TRABAJO].dbo.PARAMXML` con `CODIGO = 'S015'`, usada en `GuardarFlujo` para no
sobrescribir con un lienzo vacio no modificado.

## 4. Propiedades del control

Todas respaldadas en `ViewState` (sobreviven al postback):

- **`CodigoProceso`**: codigo del proceso/plantilla que se esta editando. Clave de
  casi todas las consultas (`PROCESO = ...`).
- **`RegProceso`**: consecutivo/registro del proceso; se pasa a `DibujarDiagrama`
  como codigo para el ASMX.
- **`IdActividad`**: id del nodo seleccionado en el lienzo. Es el eje del inspector:
  determina que reglas, componentes, documentos y usuarios se muestran.
- **`IdActividadOrigen`**: id del nodo anterior (padre) en el flujo; usado por el
  modal de reglas de continuidad para saber "desde que actividad" viene el usuario.
- **`Sucursal`**: tenant/sucursal (filtro `SUCURSAL`). Nota: parte del codigo usa
  `Session("Empresa")` en vez de `Me.Sucursal`, lo que es una inconsistencia (ver
  seccion 10).
- **`Usuario`**: usuario en sesion; se reutiliza como filtro `SUCURSAL` en la
  consulta de tareas activas de `HiddenDelete_ValueChanged` (mezcla de semantica).

## 5. Metodos del code-behind

- **`Page_Load`**: limpia `LiteralError`; en primer render llama
  `ListarAsignaciones()`, llena `cmbflujoif` (SI/NO) y lo deshabilita; fija
  `ctrlComponentes.ModuloPermisos = "000291"`; engancha
  `ctrReglas.SolicitarUpdatePanelDatos`.
- **`DibujarDiagrama(ParamXML)`**: registra el `$.get(diagramUrl+ParamXML, openDiagram)`
  para que bpmn-js cargue el XML del proceso. Es el metodo que "grafica".
- **`LimpiarDiagrama`**: registra `restartbmpnModeler();` (destruye y recrea el modeler).
- **`GuardarFlujo(Proceso)`**: normaliza saltos de linea de `HiddenXML`; si el
  proceso no existe en `DOC_PROCESOS_G` inserta, si existe compara contra la
  plantilla base `S015` y hace UPDATE del `GRAFICO`.
- **`RegActividad`**: inserta en `DOC_PROCESOS_R` el nodo recien creado
  (`HiddenCreate`) si no existe, con su `ID_ELEMENTO_PADRE` (salvo gateways/events);
  luego `GuardarFlujo`.
- **`AgregaActividad`**: valida que haya proceso e inserta el nodo con nombre y
  detalle desde `HiddenUpdate`; luego `GuardarFlujo`.
- **`ConsultarActividad`**: el metodo central del inspector. Al seleccionar un nodo
  carga `NOMBRE`, `DETALLE`, `ID_REINICIO`, `PERMITE_ASIGNACION`, `FLUJO`; habilita
  `cmbflujoif` solo si el padre es un `Gateway`; y encadena
  `ConsultarDetallePlugin()`, `UsuariosDiagramaGrupo()`, `MostrarTRD()`, mas
  `ctrReglas.MostrarReglasCargadas()` y `ctrFlujoActividades.CargarFlujoGuardado()`.
- **`QuitarActividad`**: reengancha los hijos del nodo borrado a su abuelo
  (heredando `FLUJO`/`ID_REINICIO`) y luego hace DELETE del nodo en `DOC_PROCESOS_R`.
  (Aviso: en `HiddenDelete_ValueChanged` la llamada a `QuitarActividad` esta
  comentada; hoy el borrado real lo hace la reconciliacion XML, ver seccion 8.)
- **`ConsultarActividad` auxiliares** `LimpiarActividad` / `Limpiar`: resetean el
  inspector y refrescan grids.
- **`ConsultarDetallePlugin`**: llena `GridComponentes` con los componentes/plugins
  del nodo desde `GEN_COMPONENTES_R` (join a `ENCUESTAS_MOV` para el nombre del
  formulario).
- **`UsuariosDiagramaGrupo`**: llena `GridDiagramaUsuario` con los cargos/usuarios
  asignados al nodo desde `PERMISO_CARGO` (MODULO `PROCESOS_USUARIOS`) join
  `DOC_ENTREVISTAS_ORG`.
- **`MostrarTRD`**: llena `GridTRD` con los documentos/series asociados al nodo
  desde `DOC_PROCESOS_TRD` (MODULO `FLUJOS_PROCESO`).
- **`ListaUsuarioOrigen(Idactividad)`**: llena el checkbox-list de usuarios origen
  del modal de reglas de continuidad.
- **`ListaReglasUsuario(Tipo)`**: para el modal de continuidad, calcula la actividad
  anterior (`DOC_PROCESOS_R.ID_ELEMENTO_PADRE`) y llena origen (Tipo 1) o destino
  (Tipo 2, con opcion especial "ID_MISMO" = asignar al mismo usuario).
- **`ListarActividades`**: llena `cmbreinicioact` (nodo de reinicio) con las
  actividades del proceso.
- **`ListarAsignaciones`**: llena `cmbreasignacion` con vacio/SI/NO.
- **`EliminarNodosSobrantes`**: reconciliacion masiva XML->tabla (seccion 8).
- **`RepararNodosFaltantes`**: reconciliacion inversa tabla<-XML (seccion 8).
- **`PrepararInspeccion`**: audita el modelo y vuelca hallazgos en `txtinpeccion`
  (seccion 8).
- **`CambiosAprocesos`**: propaga cambios de nombre/flujo/reinicio a las instancias
  vivas en `TAR_SEGUIMIENTO_PROCESO`.
- **`AgregarRegla`** + **`AddNewRule`**: crean reglas de continuidad usuario->usuario
  en `DOC_PROCESOS_RULES` (con validaciones anti-duplicado y regla especial
  "mismo usuario").
- **`MostrarReglasUsuario`**: llena `GridRules` con las reglas de continuidad del nodo.
- **`EliminarReglas`**: borra de `DOC_PROCESOS_RULES` las reglas marcadas en el grid.
- **`VerEstructuraTabla`**: utilidad de depuracion que hace `Response.Write` de
  `PERMISO_CARGO` (codigo muerto; llamada comentada en `Page_Load`).

## 6. Los 5 sub-controles registrados

1. **`ctrlComponentes`** (modal `PanelComponente`): gestiona componentes/plugins del
   nodo (formularios de `ENCUESTAS_MOV`, modulo 000131). Se configura con
   `Modulo="FLUJO_PROCESO"`, `ModuloPermisos="000291"`, `Codigo=CodigoProceso`,
   `Referencia=IdActividad`. Su evento `Save` refresca `ConsultarDetallePlugin`.
2. **`ctrlAsociarDocumentos`** (modal `PanelDocumento`): asocia series/tipologias
   documentales (TRD) al nodo. Se configura con `Modulo="FLUJOS_PROCESO"`,
   `Codigo=CodigoProceso`, `Referencia=IdActividad`; alimenta `DOC_PROCESOS_TRD`.
3. **`ctrReglas`** (inline, sub-acordeon "Reglas de Negocio"): reglas de negocio de
   la actividad (condiciones/campos). Se configura con `CodigoConsecutivo=CodigoProceso`
   y `Pregunta=IdActividad`; se refresca con `MostrarReglasCargadas()`. **Distinto**
   de las reglas de continuidad de usuario del modal `PanelReglasUsuario`.
4. **`ctrFlujoActividades`** (inline, sub-acordeon): define el sub-flujo/secuencia de
   actividades del nodo; se refresca con `CargarFlujoGuardado()`.
5. **`gen_ctrpermisocargos`** (modal `PanelUsuario`): permisos por cargo; asigna
   usuarios/cargos al nodo. Se configura con `Modulo="PROCESOS_USUARIOS"`,
   `Referencia2=CodigoProceso`, `Referencia3=IdActividad`; su `Save` refresca
   `UsuariosDiagramaGrupo`.

## 7. Interaccion nodo -> paneles

El puente es puramente por campos ocultos + postback (seccion 2). Al hacer clic en
un nodo del SVG, bpmn-js escribe `HiddenId` y dispara postback; el servidor entra a
`HiddenId_ValueChanged`, fija `Me.IdActividad` y llama `ConsultarActividad()`, que
en cascada llena TODO el inspector para ese nodo:

- **Cabecera/basica**: `txtnombreactividad`, `txtdetalle`, `cmbreinicioact`,
  `cmbreasignacion`, `cmbflujoif` (habilitado solo si el padre es Gateway).
- **Asignar Usuarios**: `GridDiagramaUsuario` via `UsuariosDiagramaGrupo`.
- **Recursos y Componentes**: `GridComponentes` (via `ConsultarDetallePlugin`),
  `GridTRD` (via `MostrarTRD`), mas los subcontroles `ctrReglas` y
  `ctrFlujoActividades`.

`HiddenCreate`/`HiddenUpdate`/`HiddenDelete` disparan alta, renombrado y baja del
nodo respectivamente. El borrado tiene guarda de negocio: `HiddenDelete_ValueChanged`
cuenta casos vivos en `TAR_SEGUIMIENTO_PROCESO` para esa actividad y, si hay,
bloquea la eliminacion mostrando un error y re-dibuja el diagrama con
`DibujarDiagrama(Me.RegProceso)`.

## 8. Utilidades de mantenimiento del grafo

Como el XML (`DOC_PROCESOS_G`) y la tabla (`DOC_PROCESOS_R`) pueden desincronizarse
(edicion concurrente, imports, fallos de postback), hay tres utilidades:

- **`EliminarNodosSobrantes`**: parsea el XML con `xml_data.nodes('//*')` (T-SQL
  XML), arma una tabla temporal de nodos reales, borra de `DOC_PROCESOS_R` los tipos
  estructurales (`sequenceFlow`, `BPMNShape`, `BPMNEdge`, etc.) y los nodos que ya no
  existen en el XML, reconstruye `ID_ELEMENTO_PADRE` desde los `sourceRef/targetRef`,
  reescribe `TIPO_BPMN`/`ID_BPMN`/`NOMBRE`, y con un CTE recursivo recalcula el nivel
  (`PASO`). Cierra con `CambiosAprocesos()`. Resuelve el problema de "nodos fantasma"
  en la tabla que ya no estan en el dibujo.
- **`RepararNodosFaltantes`**: la operacion inversa. Detecta nodos presentes en el
  XML pero ausentes en `DOC_PROCESOS_R` y los inserta (reconstruyendo el id con
  guion antes del numero y su padre). Resuelve el problema de "nodos dibujados pero
  sin fila", que dejarian al operador sin poder configurarlos.
- **`PrepararInspeccion`**: auditoria de calidad del modelo. Lista (en `txtinpeccion`,
  textarea negra) las actividades sin usuario asignado (nodos sin `PERMISO_CARGO`) y
  los `exclusiveGateway` cuyas salidas no cubren ambos flujos SI y NO. No corrige,
  solo reporta antes de publicar el proceso.

## 9. Tablas SQL que toca

- **`DOC_PROCESOS_G`**: XML BPMN del proceso (`GRAFICO`).
- **`DOC_PROCESOS_R`**: nodos normalizados del proceso (eje del inspector).
- **`DOC_PROCESOS_RULES`**: reglas de continuidad usuario->usuario por actividad.
- **`DOC_PROCESOS_TRD`**: documentos/series (TRD) asociados a cada nodo.
- **`GEN_COMPONENTES_R`**: componentes/plugins por nodo (join a `ENCUESTAS_MOV`).
- **`ENCUESTAS_MOV`**: formularios (modulo 000131), para el nombre del formulario.
- **`PERMISO_CARGO`**: usuarios/cargos asignados (MODULO `PROCESOS_USUARIOS`,
  `REFERENCIA2`=proceso, `REFERENCIA3`=actividad).
- **`DOC_ENTREVISTAS_ORG`**: organigrama/cargos (`NOMBRE_CARGO`, `REG`).
- **`TAR_SEGUIMIENTO_PROCESO`**: instancias vivas del proceso (para guardas de
  borrado y propagacion de cambios). Enlaza con [[AdmWorkflow - Motor de flujo de la tarea]].
- **`PARAMXML`** (`[dbx.TRABAJO]`): plantilla de diagrama base (`CODIGO='S015'`).

Todas las consultas usan el alias de portabilidad `[dbx.GENE]`.

## 10. Riesgos y deuda tecnica

1. **SQL concatenado por string en todos los metodos**: `Me.Sucursal`,
   `Me.CodigoProceso`, `Me.IdActividad`, texto de filtros y hasta el XML entero se
   interpolan directo en la sentencia. Riesgo de inyeccion y de romperse con
   comillas. Solo `txtnumpaso_TextChanged` escapa comillas (`Replace("'","''")`).
2. **Doble fuente de verdad XML vs tabla**: el grafo vive en `DOC_PROCESOS_G` y en
   `DOC_PROCESOS_R`, y su consistencia depende de utilidades de reparacion manual
   (seccion 8). Fragilidad estructural.
3. **Acoplamiento por campos ocultos + postback**: cada clic en el SVG es un
   `__doPostBack`; la UX depende de que `HiddenId`/`HiddenCreate`/etc. y el nombre
   del control coincidan. La normalizacion de id `_`<->`-` (JS y SQL) es fragil.
4. **`Session("Empresa")` vs `Me.Sucursal`**: se usan de forma intercambiable en
   distintas consultas; si difieren, los datos se mezclan entre tenants.
5. **`Me.Usuario` usado como `SUCURSAL`** en la consulta de tareas activas del
   borrado: semantica confusa y potencial guarda incorrecta.
6. **Motor bpmn-js cargado desde unpkg (CDN externo)**: dependencia de red en
   produccion; sin CDN el lienzo no dibuja.
7. **Acoplamiento a 5 sub-controles + estado en ViewState**: el inspector es una
   enredo de UpdatePanels anidados; el estado del nodo vive en ViewState y se
   reparte entre subcontroles con propiedades sueltas.
8. **Codigo muerto/comentado**: `VerEstructuraTabla` (con `Response.Write`),
   `QuitarActividad` comentado, restos de `flowsvg`/`svg.js` en el host.

## 11. DESTINO .NET 10 (Blazor)

En el destino, el graficado del flujo dentro del visor de tareas se reconstruye
como un componente Blazor que **embebe bpmn-js en modo lectura** (JS interop) para
mostrar el diagrama del proceso y **resaltar el nodo actual** de la tarea del
operador (ese resaltado es responsabilidad del visor, ver
[[ctrVertareasII - Spec para reconstruir en Claude Design]]). El diseno/edicion
formal del proceso (crear nodos, mover, reglas) se queda como herramienta de la
**Capa 3**, no dentro del visor. Lineamientos:

- **Motor**: seguir con bpmn-js (mismo formato BPMN 2.0 ya guardado en
  `DOC_PROCESOS_G`), montado sobre un `<div>` gestionado por interop; en modo
  visor se deshabilita la paleta y el modeling (solo navegacion + highlight). Se
  puede evaluar `bpmn-js` viewer (mas liviano) para el caso solo-lectura, o un
  canvas Blazor propio si se busca eliminar la dependencia JS.
- **Paneles laterales**: el inspector (reglas, componentes, documentos, usuarios,
  permisos) se rehace como componentes Blazor independientes que reciben
  `CodigoProceso` + `IdActividad` como parametros y consultan por servicios
  tipados (no SQL concatenado), reemplazando el enredo de UpdatePanels.
- **Sincronizacion**: eliminar la doble fuente XML/tabla. La tabla normalizada de
  nodos debe derivarse del XML por un servicio unico de importacion, retirando las
  utilidades manuales `EliminarNodosSobrantes`/`RepararNodosFaltantes`.
- **Puente evento->estado**: reemplazar `HiddenId`+postback por eventos de interop
  (`DotNetObjectReference`) que notifican la seleccion de nodo al componente
  Blazor, el cual reactivamente pinta los paneles.

Ver aspecto final en el prototipo visual (`01. Requerimiento/Prototipo/ECOREX.dc.html`), parametrizacion de nodo en
[[Parametrizacion por nodo - panel Propiedades]] y prueba de portabilidad del XML
en [[Portabilidad BPMN - prueba en bpmn.io]].

## 12. Enlaces

- [[Visión y entorno]]
- el prototipo visual (`01. Requerimiento/Prototipo/ECOREX.dc.html`)
- [[00 - Visión Flujos]]
- [[AdmWorkflow - Motor de flujo de la tarea]]
- [[ctrVertareasII - Spec para reconstruir en Claude Design]]
- [[Parametrizacion por nodo - panel Propiedades]]
- [[Portabilidad BPMN - prueba en bpmn.io]]
