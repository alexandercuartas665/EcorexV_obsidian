---
tipo: spec-modulo
modulo: NEWFRONT_tar_conceptos.aspx
carpeta: Tareas
proposito: Catalogo maestro de Categorias y Sub-Categorias de Actividades (tipos de tarea) del modulo ECOREX Tareas
---

# NEWFRONT_tar_conceptos - Spec para reconstruir

## 1. Que hace este modulo

Es el catalogo maestro de "Actividades" (tipos de tarea) del modulo ECOREX Tareas. Administra una jerarquia de dos niveles: **Categorias** (tabla `TIPO_TAR`) y **Sub-Categorias** (tabla `TIPO_TAR_R`). Cada categoria agrupa varias sub-categorias, y cada sub-categoria es el elemento operativo real: define la lista de chequeo, si requiere cliente, si inicia un modulo (proceso automatico), si muestra boton de cierre manual y sus parametros por defecto (titulo y detalle auto).

Sobre cada sub-categoria se cuelgan capas de configuracion: procesos BPMN relacionados (`TIPO_TAR_R_PRO`), empresas donde aplica (`TIPO_TAR_EMPRESA`), agencias (sucursales secundarias), permisos por cargo, permisos por usuario, componentes fijos, modulos de formacion y usuarios que reciben notificacion por email o integracion (`TIPO_TAR_N`, `TIPO_TAR_NR`).

El modulo actua como panel de control de todo el sistema de tareas: al crear una tarea real (en otros modulos) el sistema consulta este catalogo para saber que reglas aplicar. Es modulo `000270`, ubicacion de menu `Home/tareas/Actividades`, titulo TopBar "ACTIVIDADES".

## 2. Ubicacion en el menu / quien lo usa

- **Ruta fisica:** `Bootstrap/Formularios/Modulos/Tareas/NEWFRONT_tar_conceptos.aspx`
- **Codigo modulo (permisos):** `000270` (asignado en `topBar.Modulo` dentro de `Page_Load`)
- **Ubicacion breadcrumb:** `Home/tareas/Actividades`
- **Master:** `~/Formularios/Modulos/Dashboard/MasterFormsii.Master`
- **Namespace clase:** `GestionMovil.NEWFRONT_tar_conceptos`

Grep en la solucion (`NEWFRONT_tar_conceptos`) solo devuelve las 5 lineas del propio `.vbproj` y los tres archivos del modulo. No hay link duro desde otros aspx: el acceso ocurre por menu dinamico (sideNavBar) filtrado por permisos de modulo `000270`. Las tablas que produce (`TIPO_TAR`, `TIPO_TAR_R`) si son consumidas por otros modulos operativos de tareas y por el modulo `000291` (procesos) via `ctrlComponentes` con `ModuloPermisos = "000291"`.

## 3. Layout visual (wireframe ASCII)

```
+==============================================================================+
|  [ TopBar: ACTIVIDADES   Home/tareas/Actividades       Guardar Borrar Limpiar]|
+==============================================================================+
| [ Actividades ] [ Detalle ]                                                  |
+------------------------------------------------------------------------------+
| TAB 1 - ACTIVIDADES                                                          |
|                                                                              |
|  Codigo: [ dropdown cmbcodigo v ]   Nombre: [ txtnombre_______________ ]     |
|  Descripcion: [ textarea 5 rows _______________________________________ ]    |
|                                                                              |
|  [Panel colapsable] Sub-Categorias                                           |
|     Titulo + descripcion                                                     |
|                          [Agregar nueva sub-categoria] [Eliminar]           |
|     +--------------------------------------------------------------+         |
|     | [x] | Sub-Categoria | Lista chequeo | Plugins | Usuario | ed |         |
|     +--------------------------------------------------------------+         |
|                                                                              |
|  [Panel colapsable] Notificaciones                                           |
|     Usuario: [cmbusuarioid v]   Sub-Categoria: [cmbsubcategoria v]           |
|                                                    [Agregar]  [Quitar]      |
|     +-----------------------------------------------------------------+     |
|     | [x] | Id | Nombre | Email | Usar integracion (nested grid)      |     |
|     +-----------------------------------------------------------------+     |
|                                                                              |
|  [Panel colapsable] Uso restringido                                          |
|     <uc1:gen_ctrpermisousuario />                                            |
|                                                                              |
+------------------------------------------------------------------------------+
| TAB 2 - DETALLE                                                              |
|  [Panel Filtros] Empresa | Estado | Categoria | Subcategoria  [Aplicar]     |
|  +---------------------------------------------------------------+           |
|  | Categoria(dd) | Codigo | Categoria | Descripcion | SubCat |    |          |
|  | Empresas | Cant Tareas | Categoria inactivo (chk)          |    |          |
|  +---------------------------------------------------------------+           |
+==============================================================================+

+-----------------  MODAL: PanelSubCategoria (Bootstrap modal-lg)  -------------+
| Nombre: [txtsubnombre__________________]                                     |
| [x] La actividad requiere asignar un cliente                                 |
| Lista de chequeo requerida (separada por ;) [textarea 3 rows]                |
|                                                                              |
| == Gestion del Inicio del Proceso ==                                         |
|   La actividad inicia en un modulo?   [ switch chkIniciaModulo ]             |
|   [x] Activar Boton de Cierre Manual  ( chkBotonCierre )                     |
|                                                                              |
|   +--- panelParametrosAuto (slideDown si chkIniciaModulo) ---+                |
|   | Titulo Automatico: [txtTituloAuto___________________]    |                |
|   | Detalle Predeterminado: [textarea txtDetalleAuto]        |                |
|   +---------------------------------------------------------+                |
|                                                                              |
|            [Crear/Actualizar categoria] [Nuevo] [Eliminar]                   |
|                                                                              |
|  [Acc.] Permisos Cargos     <uc1:gen_ctrpermisocargos />                     |
|  [Acc.] Componentes fijos   <uc1:ctrlComponentes />                          |
|  [Acc.] Documentacion       <uc1:ctrlFormacion />                            |
|  [Acc.] Procesos relacionados   cmbprocesos + Grid                           |
|  [Acc.] Empresas activas (visible solo si Session Empresa=01)               |
|  [Acc.] Sedes que aplica (cmbagencia)                                        |
+------------------------------------------------------------------------------+
```

## 4. Controles servidor (principales)

| ID | Tipo | Proposito | Bindings / Notas |
|---|---|---|---|
| topBar | MasterTopBar (uc) | Toolbar Guardar/Borrar/Limpiar | Title="ACTIVIDADES", Modulo="000270" |
| cmbcodigo | DropDownList | Selector de Categoria (TIPO_TAR) | AutoPostBack; carga desde `ListarCodigo`; disparo `VerServicio` |
| txtnombre | TextBox | Nombre de la categoria | Validado en AddCatalogo |
| txtdescripcion | TextBox (MultiLine 5) | Descripcion categoria | |
| GridSubcategoria | GridView | Lista sub-categorias de la categoria | DataSource = `consultarSubcategoria`; columnas: check, NOMBRE, CHEQUEO, PLUGINS, USUARIO, editar |
| cmbusuarioid | DropDownList | Usuario para notificacion | ListarUsuario |
| cmbsubcategoria | DropDownList | Sub-categoria filtro notificacion | ListaSubcategorias |
| GridNotifica | GridView | Usuarios notificados; incluye grid anidado GridDetalleNotifica | RowDataBound rellena cmbintegracion (XML PARAMXML 0202) y GridDetalleNotifica |
| gen_ctrpermisousuario | UC | Restringe usuarios | Modulo="PERMISOS_ACTIVIDADES" |
| cmbfiltroempresa, cmbestado, txtfiltrocategoria, txtfiltrosubcategoria | Filtros TAB 2 | | FiltroDetalle() los concatena |
| GridDetalle | GridView | Vista completa Categoria x Sub-categoria | Muestra CANT_EMPRESAS y CANT_USADO; edicion inline categoria/subcat y check inactivar |
| PanelSubCategoria | asp:Panel modal | Modal editor de sub-categoria | Abierto por ScriptManager.RegisterStartupScript + `new bootstrap.Modal(...).show()` |
| txtsubnombre, txtsublistachequeo | TextBox | Nombre y lista de chequeo separada por `;` | |
| chkFlagCliente | CheckBox | Obliga a asignar cliente al crear tarea | FLAG_CLIENTE |
| chkIniciaModulo | CheckBox switch | Marca la sub-categoria como iniciada por un modulo | ClientIDMode Static; toggle JS muestra panelParametrosAuto |
| chkBotonCierre | CheckBox | Activa boton de cierre manual | FLAG_BOTON_CIERRE |
| txtTituloAuto, txtDetalleAuto | TextBox | Titulo/detalle por defecto de tarea auto | TITULO_AUTO, DETALLE_AUTO |
| LiteralGuardado | Literal | Mensaje de exito/fallo del modal | modsms_success / modsms_error |
| gen_ctrpermisocargos | UC | Cargos autorizados | Modulo="CARGOS_ACTIVIDADES" |
| ctrlComponentes | UC | Componentes fijos asociados | Modulo="CATEGORIA_SUBPROCESO", ModuloPermisos="000291" |
| ctrlFormacion | UC | Modulos de formacion relacionados | Modulo="FORMACION_ACTIVIDADES" |
| cmbprocesos + GridProcesos | dd + grid | Procesos BPMN relacionados (DOC_PROCESOS) | Insert/Delete `TIPO_TAR_R_PRO` |
| cmbempresaseleccion + GridViewEmpresa | dd + grid | Empresas donde aplica (SUCURSAL) | Solo visible si `Session("Empresa")="01"` (panelempresa) |
| cmbagencia + GridViewAgencia | dd + grid | Agencias (SUCURSAL_R) | Ambos usan `TIPO_TAR_EMPRESA` |
| LinkbuttoNew | LinkButton | "Agregar todas las empresas" | AgregarTodasAgenciaDetalles |

## 5. Eventos servidor

- **Page_Load:** setea topBar (titulo/modulo/breadcrumb); en primer postback carga combos `ListaEstado`, `ListarCodigo`, `DetalleServicio`, `ListarUsuario`, `ListaProcesos`, `ListaSucursal`, `ListaAgencias`. Muestra `panelempresa` solo si empresa "01". Inicializa referencias de los user controls (gen_ctrpermisousuario, ctrlFormacion, gen_ctrpermisocargos, ctrlComponentes).
- **cmbcodigo.SelectedIndexChanged -> VerServicio(codigo):** carga nombre/descripcion en pantalla, refresca sub-categorias, notificaciones y permisos.
- **topBar.Save -> AddCatalogo:** INSERT si no existe el CODIGO (usando `Funciones.tipdoc.Consecutivo("01T",...)`), sino UPDATE. Refresca `DetalleServicio` y `ListarCodigo`.
- **topBar.Delete -> DelCatalogo:** valida que no existan sub-categorias antes de borrar. Si hay, muestra error via `MValidaError.sms_nerror(topBar)`.
- **topBar.Clear -> Limpiar(1):** limpia controles y recarga codigos.
- **LinkButton2_Click (Agregar nueva sub-categoria):** limpia modal y lo abre con `ScriptManager.RegisterStartupScript` + `new bootstrap.Modal(...).show()`.
- **LinkButton13_Click (editar en grid):** carga sub-categoria en el modal via `CargarSubcategorias(reg)` y lo abre.
- **LinkbuttonGuardar_Click / LinkButtonAgregarEstado_Click -> AgregaSubcategoria:** INSERT/UPDATE en `TIPO_TAR_R` con OUTPUT INSERTED.REG. Setea flags de RQ07 (FLAG_INICIA_MODULO, FLAG_BOTON_CIERRE, TITULO_AUTO, DETALLE_AUTO, FLAG_CLIENTE).
- **Linkbutton3_Click:** QuitarSubcategoriaModal + limpiar modal.
- **LinkButton1_Click:** QuitarSubcategoria por checkboxes marcados del grid (borra `TIPO_TAR_R` y `TIPO_TAR_R_PRO`).
- **LinkButtonuseradd_Click / LinkButtonuserdel_Click:** AgregaNotificacion / QuitarNotificacion sobre `TIPO_TAR_N`.
- **cmbrowintegracion_SelectedIndexChanged:** carga cuentas de `SUCURSAL_INT` filtradas por integracion elegida.
- **LinkButtonFiltrarDatos_Click:** INSERT en `TIPO_TAR_NR` (cuenta+grupo) y refresca grid anidado.
- **LinkButton1_Click1 (X del grid anidado):** DELETE fila `TIPO_TAR_NR`.
- **cmbrowsubcategoria_SelectedIndexChanged:** arma UPDATE de `TIPO_TAR_N.SUBCATEGORIA` (nota: la sentencia se arma pero NO se ejecuta — ver riesgos).
- **Linkbutton4_Click / Linkbutton5_Click:** AgregaProcesoSubcategoria / borra fila `TIPO_TAR_R_PRO`.
- **LinkbuttonAddEmpresa_Click / LinkbuttonDelEmpresa_Click / LinkbuttoNew_Click:** ABM sobre `TIPO_TAR_EMPRESA` (por empresa individual o "todas").
- **LinkbuttonaddAgencia_Click / LinkbuttonDelAgencia_Click:** ABM sobre `TIPO_TAR_EMPRESA` para agencias (`SUCURSAL_R`).
- **LinkButton1_Click2:** aplica filtros y refresca `DetalleServicio`.
- **CheckReporta_CheckedChanged:** UPDATE `TIPO_TAR.FLAG_INA` (activo/inactivo) desde grid Detalle.
- **cmbrowcategoria_TextChanged / SubCategoria_TextChanged:** edicion inline de `TIPO_TAR_R.CATEGORIA` o `.NOMBRE` desde GridDetalle.
- **GridDetalle_RowDataBound:** enlaza dropdown `cmbrowcategoria` con el DataSet `CategoriasTrabajo` (ViewState).
- **GridNotifica_RowDataBound:** en cada fila enlaza cmbintegracion (leyendo XML `PARAMXML.CONTENIDO WHERE CODIGO='0202'` de `[dbx.TRABAJO]`) y hidrata GridDetalleNotifica.

## 6. Tablas SQL usadas

Base: alias `[dbx.GENE]` (`manejo_dbx = "GENE"`), tabla ancla `manejo_tabla = "TIPO_TAR"`, consecutivos `01T` (categoria) y `02T` (sub-categoria).

- **TIPO_TAR** — Categoria maestra. Campos: CODIGO, NOMBRE, DESCRIPCION, SUCURSAL, REG, FLAG_INA.
- **TIPO_TAR_R** — Sub-categoria. Campos usados: REG, CATEGORIA, CODIGO, NOMBRE, CHEQUEO, SUCURSAL, PLUGINS, TITULO_ACT, USUARIO, PROCESO, FLAG_INICIA_MODULO, FLAG_BOTON_CIERRE, TITULO_AUTO, DETALLE_AUTO, FLAG_CLIENTE.
- **TIPO_TAR_R_PRO** — Procesos BPMN relacionados (PEDREG=REG sub-categoria, PROCESO=DOC_PROCESOS.CODIGO).
- **TIPO_TAR_EMPRESA** — Empresas/agencias donde aplica la sub-categoria (SUCURSAL, PEDREG, CATEGORIA, EMPRESA).
- **TIPO_TAR_N** — Usuarios notificados por categoria (TIPO=CATEGORIA, SUBCATEGORIA, ID_USUARIO, SUCURSAL).
- **TIPO_TAR_NR** — Cuentas/grupos por integracion asociados a cada notificacion (PEDREG=REG de TIPO_TAR_N, CUENTA, GRUPO).
- **DOC_PROCESOS** — Catalogo de procesos (solo lectura para combo).
- **SUCURSAL, SUCURSAL_R** — Empresas y agencias (solo lectura).
- **USUARIO** — Solo lectura para combos.
- **SISTEMA** — Consultada como referencia cruzada (CANT_USADO): cuenta cuantos registros usan la sub-categoria.
- **SUCURSAL_INT** — Cuentas de integracion por empresa.
- **[dbx.TRABAJO].dbo.PARAMXML WHERE CODIGO='0202'** — XML con lista de integraciones disponibles (parseado con `XDocument`).

Snippet clave (insert sub-categoria con OUTPUT):

```sql
INSERT INTO [dbx.GENE].dbo.TIPO_TAR_R
   (CATEGORIA, CODIGO, NOMBRE, CHEQUEO, SUCURSAL, PLUGINS, TITULO_ACT, USUARIO,
    FLAG_INICIA_MODULO, FLAG_BOTON_CIERRE, TITULO_AUTO, DETALLE_AUTO, FLAG_CLIENTE)
OUTPUT INSERTED.REG
VALUES (@categoria, @consecutivo02T, @nombre, @chequeo, @sucursal, '', 0, '',
        @flagIniciaModulo, @flagBotonCierre, @tituloAuto, @detalleAuto, @flagCliente)
```

Snippet clave (grid detalle con conteos):

```sql
SELECT TIPO_TAR.CODIGO, TIPO_TAR.NOMBRE, TIPO_TAR.DESCRIPCION, TIPO_TAR.REG, TIPO_TAR.FLAG_INA,
       TIPO_TAR_R.CODIGO AS TIPO_TAR_R_CODIGO, TIPO_TAR_R.NOMBRE AS TIPO_TAR_R_NOMBRE, TIPO_TAR_R.REG AS TIPO_TAR_R_REG,
       ISNULL((SELECT count(distinct A.EMPRESA) FROM [dbx.GENE].dbo.TIPO_TAR_EMPRESA A
               WHERE A.SUCURSAL=TIPO_TAR.SUCURSAL AND A.PEDREG=TIPO_TAR_R.REG),0) AS CANT_EMPRESAS,
       ISNULL((SELECT count(distinct A.CODIGO) FROM [dbx.GENE].dbo.SISTEMA A
               WHERE A.SUCURSAL=TIPO_TAR.SUCURSAL AND A.CONCEPTO=TIPO_TAR.CODIGO
                     AND A.SUBCATEGORIA=TIPO_TAR_R.CODIGO),0) AS CANT_USADO
FROM [dbx.GENE].dbo.TIPO_TAR
LEFT JOIN [dbx.GENE].dbo.TIPO_TAR_R
       ON TIPO_TAR.SUCURSAL=TIPO_TAR_R.SUCURSAL AND TIPO_TAR.CODIGO=TIPO_TAR_R.CATEGORIA
WHERE TIPO_TAR.SUCURSAL=@empresa {FILTRO}
ORDER BY TIPO_TAR.NOMBRE
```

## 7. Directivas Register / dependencias

Register en el ASPX:

- `~/Controles/MasterTopBar.ascx` (uc1:MasterTopBar) — barra Guardar/Borrar/Limpiar.
- `~/Controles/form_contacto.ascx` (declarado, no usado en el markup visible).
- `~/Controles/ModalDialog.ascx` (declarado, no usado en el markup visible).
- `~/Formularios/Modulos/General/controles/gen_ctrpermisousuario.ascx` — permisos por usuario.
- `~/formularios/modulos/general/controles/ctrlFormacion.ascx` — modulos de formacion.
- `~/Formularios/Modulos/General/controles/ctrlComponentes.ascx` — componentes fijos.
- `~/Formularios/Modulos/General/controles/gen_ctrpermisocargos.ascx` — permisos por cargo.
- Assembly `AjaxControlToolkit` (cc1) — declarado, no se ve uso directo.

CSS/JS externo (CDN):

- DataTables 1.10.20 + Buttons 1.4.2 (css y js).
- moment.min.js (validacion fecha).
- spinners.css (icono progreso local Frontend4).
- pdfmake 0.1.32, jszip 3.1.3, buttons.html5, buttons.print.

Framework local: Bootstrap 4 + Metronic (KT) via Frontend4/, FontAwesome 4.7. Notificaciones toastr (global de la Master).

JS inline destacado:

- `pageLoad()` oculta botones Guardar/Borrar/Limpiar del TopBar con clase `mostrarbotonestopbar` (asi el modulo depende de OnClick propios). Tambien engancha el toggle de `chkIniciaModulo` para slide del panel `panelParametrosAuto`.
- `MarcarElementos(clicked_id, Grilla)` — check-all/uncheck-all para los grids.
- Handlers `BeginRequest/EndRequest` conservan scroll durante async postbacks.

Nota: hay dos caracteres sueltos "s" fuera de cualquier tag en el Content del script pagina (lineas 80 y 82) — probablemente residuo de una edicion. Cosmetico, no rompe render.

## 8. Modales / popups

Un unico modal (`asp:Panel PanelSubCategoria`, CssClass `modal fade`) que aloja el editor completo de sub-categoria. Se abre por servidor con:

```vb
ScriptManager.RegisterStartupScript(Page, Page.GetType(), "pnlTestPanel",
    "new bootstrap.Modal(document.getElementById('" & PanelSubCategoria.ClientID & "')).show();", True)
UpdatePanelSubcategoria.Update()
```

Contiene 6 acordeones internos: Permisos Cargos, Componentes fijos, Documentacion del proceso, Procesos relacionados, Empresas activas (visible solo si Session("Empresa")="01" via `panelempresa.Visible`) y Sedes que aplica. Mensajes de guardado se muestran en `LiteralGuardado` (bloque literal dentro de un UpdatePanel `UpdateMode="Always"`).

## 9. Reglas de negocio detectadas

1. **Jerarquia dos niveles obligatoria:** solo se puede crear sub-categoria si hay categoria seleccionada (`cmbcodigo`); se valida en `AgregaSubcategoria`.
2. **No se puede borrar categoria si tiene sub-categorias** — DelCatalogo hace SELECT COUNT antes.
3. **Consecutivos autogenerados:** `Funciones.tipdoc.Consecutivo` con `01T` (categoria) y `02T` (sub-categoria) — solo si el usuario no eligio codigo previo.
4. **FLAG_INA:** una categoria puede quedar "inactiva" desde el grid detalle (checkbox por fila) sin borrarla.
5. **FLAG_CLIENTE:** cuando esta activo obliga a asignar cliente al crear la tarea real (regla consumida por otros modulos).
6. **RQ07 — Gestion del inicio del proceso:** `FLAG_INICIA_MODULO` marca sub-categorias auto-iniciadas por un modulo; abre panel dinamico con `TITULO_AUTO`/`DETALLE_AUTO`. `FLAG_BOTON_CIERRE` activa boton de cierre manual en la tarea generada.
7. **Panel Empresas activas** solo se muestra si `Session("Empresa")="01"` — tenant "master" / holding.
8. **Notificacion tiene doble tabla:** `TIPO_TAR_N` (usuario) + `TIPO_TAR_NR` (cuentas/grupos por integracion). Las integraciones vienen de un XML almacenado en `PARAMXML.CODIGO='0202'` de la base TRABAJO.
9. **Contexto pasado a los UC:** modulos usados como llaves de permisos: `PERMISOS_ACTIVIDADES`, `CARGOS_ACTIVIDADES`, `FORMACION_ACTIVIDADES`, `CATEGORIA_SUBPROCESO` (ModuloPermisos `000291`).
10. **Filtros TAB 2:** Estado (Activo/Inactivo/Todos/vacio), Empresa (existencia en `TIPO_TAR_EMPRESA`), Categoria y Subcategoria (LIKE).

## 10. Puntos de reconstruccion en Claude Design

- **Frame principal en dos pestanas:** tab "Actividades" (editor + acordeones) y tab "Detalle" (grid maestro con filtros).
- **Formulario superior** (Codigo, Nombre, Descripcion) actua como cabecera; el boton Guardar del TopBar guarda esta capa (Categoria).
- **Grid de Sub-Categorias con boton "Agregar nueva"** que abre modal grande (aprox 1000px). El check-all por columna es requerido.
- **Modal Sub-Categoria** con secciones colapsables:
  1. Datos basicos (Nombre, checkbox cliente, lista de chequeo con separador ";").
  2. Bloque destacado "Gestion del Inicio del Proceso" (RQ07) con switch + checkbox + panel condicional (Titulo/Detalle auto).
  3. Botones Crear/Actualizar, Nuevo, Eliminar + Literal para feedback inline.
  4. Acordeones anidados: Cargos, Componentes, Formacion, Procesos, Empresas (condicional), Sedes.
- **Grids anidados:** el grid de Notificaciones contiene subgrids (una fila = un usuario, dentro un grid con sus cuentas/grupos). Rediseno recomendado: usar drawer o expandable row.
- **Combos filtrados por tenant** (`Session("Empresa")`) — decision: mantener multi-tenant.
- **Modulo `000270`** para permisos — mantener el codigo.
- **Estado UI conservado en ViewState:** `RegSubcategoria` (id abierto en modal) y `CategoriasTrabajo` (dataset). En Claude Design usar estado de componente / query cache equivalente.
- **Feedback usuario:** toastr para exito global (via TopBar/MValidaError), Literal inline en modal.
- **Boton Panel `panelempresa`** oculto salvo tenant master — reproducir esta condicion (o hacer feature flag).

## 11. Riesgos / cosas raras

- **Concatenacion SQL directa:** todo el modulo arma queries con `& "..." &` — riesgo de inyeccion SQL y comilla-hostilidad en nombres. Al reconstruir, parametrizar.
- **`cmbrowsubcategoria_SelectedIndexChanged`** arma un UPDATE pero NUNCA lo ejecuta (`tbrec.Execute` no se llama). Bug latente — al reconstruir decidir si esa columna debe persistir.
- **Handlers duplicados:** `txtusuencargado_TextChanged` y `txtflujoproceso_TextChanged` estan definidos pero los controles `txtusuencargado`/`txtflujoproceso` no existen en el markup del aspx (solo en versiones anteriores). Codigo muerto.
- **Handlers Wireup manual:** `LinkButton2_Click`, `LinkButton13_Click`, `LinkButton1_Click1`, `LinkButton1_Click2` — nombres genericos sin relacion con lo que hacen. Renombrar en la reconstruccion.
- **Dos "s" sueltas** en el bloque de scripts (lineas 80 y 82 del aspx) — cosmetico.
- **`AjaxControlToolkit` y `form_contacto.ascx` / `ModalDialog.ascx`** registrados pero no usados — eliminar.
- **Dependencia de XML externo** `PARAMXML CODIGO='0202'` en base TRABAJO — acoplamiento fuerte, documentar el contrato (lista `<CorXml><CODIGO/><NOMBRE/></CorXml>`).
- **Cross-base access:** el modulo lee `[dbx.TRABAJO]` desde codigo cuyo default es `GENE` — falta `COLLATE Modern_Spanish_CI_AS` en joins entre bases (regla del CLAUDE.md).
- **`AutoPostBack=true` en muchos combos** — cada seleccion causa full postback via UpdatePanel, latencia notable en tenants con muchos usuarios.
- **`p_factura` sin inicializar** en `AddCatalogo`: si `cmbcodigo.SelectedValue` esta vacio y luego el codigo generado por `Consecutivo` tambien es vacio, no se hace nada y no se avisa al usuario.
- **Deuda tecnica visible:** `#Region "Documentacion"` vacio, `TODO`s implicitos por comentarios como "comentar cualquier cosa" en CheckReporta.
- **`ClientIDMode="Static"`** solo en chkIniciaModulo/chkBotonCierre — depende de un JS que hace `$('#chkIniciaModulo')` por ID literal. Si se agregan mas instancias del modal en el futuro se rompen los IDs.
- **`chkBotonCierre` no dispara panel** — el JS solo engancha `chkIniciaModulo`. Si el diseno RQ07 pedia que el boton de cierre tuviera panel propio, falta implementarlo.
