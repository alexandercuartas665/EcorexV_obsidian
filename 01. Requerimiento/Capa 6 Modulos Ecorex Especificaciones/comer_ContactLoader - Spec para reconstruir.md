---
tipo: spec-modulo
modulo: comer_ContactLoader.aspx
carpeta: Comercial
proposito: Explorador y segmentador de contactos capturados por N8N (LinkedIn / Maps / Facebook / Instagram) con constructor de filtros dinamico, presets guardados y salto al pipeline comercial.
---

# comer_ContactLoader - Spec para reconstruir

## 1. Que hace este modulo

`comer_ContactLoader.aspx` (modulo 000873, ruta de menu `Home/comercial/contaclead`, titulo topbar `CONTACLOADER`) es la vista comercial de una base de contactos que **NO se carga desde Excel/CSV**. La ingesta la hace un flujo externo N8N que raspa perfiles/empresas de LinkedIn, Google Maps, Facebook e Instagram y los deposita en la base `dbn8n`. La pagina es un **explorador/segmentador**: permite al comercial elegir la fuente, buscar por nombre de lista, construir filtros multi-condicion sobre el JSON crudo del scraper, guardar esos filtros como presets con conteo, revisar cada contacto en un modal 360, disparar acciones (workflow) sobre un preset y saltar al pipeline de leads confirmados.

Es un ASPX contenedor extremadamente delgado: solo hospeda el UserControl `ctrlParrillaContactLead.ascx`, que contiene el 100% de la logica funcional. El aspx aplica la master `MasterFormsii.Master`, importa el `MasterTopBar` (donde oculta por JS los botones Guardar / Borrar / Limpiar porque no aplica el ciclo CRUD estandar) y registra los controles auxiliares `form_contacto.ascx`, `ModalDialog.ascx` y `ctrlParrillaContactLead.ascx`.

Estructura interna del UserControl: pestanas `Contactos` y `Lead confirmados`. La segunda pestana es lazy-load: se dispara un `btnActivarPipeline` oculto en `shown.bs.tab` y ese boton llama a `ctrlPipelineContactLoader1.CargarPipeline()`. Los detalles de cada contacto se abren en un modal Bootstrap que hospeda `ctrlGestionLoader.ascx` (Ficha Comercial 360). Las acciones sobre un preset abren otro modal con `ucWorkflowDesigner.ascx`. El modulo NO importa contactos - los consume.

## 2. Ubicacion / consumidores

- Ruta fisica: `Bootstrap/Formularios/Modulos/Comercial/comer_ContactLoader.aspx`
- Codebehind: `comer_ContactLoader.aspx.vb` (clase `GestionMovil.comer_ContactLoader`)
- Master: `~/Formularios/Modulos/Dashboard/MasterFormsii.Master`
- Modulo topbar: `000873` (idioma EN, menu `BASIC_TOPBAR`)
- Ubicacion breadcrumb: `Home/comercial/contaclead`
- Consumidores: menu comercial (referenciado unicamente por `Bootstrap.vbproj`). No hay otras aspx que la llamen.
- Depende de: `MasterTopBar.ascx`, `ModalDialog.ascx`, `form_contacto.ascx`, `ctrlParrillaContactLead.ascx`, `ctrlGestionLoader.ascx`, `ctrlPipelineContactLoader.ascx`, `ucWorkflowDesigner.ascx`, servicio `Servicios/Form/fm_autocompletado.asmx` (metodo `ControlAutocompletadoExterno`).

## 3. Layout visual

```
+--------------------------------------------------------------------------------------+
| MasterTopBar - Titulo: CONTACLOADER   (Guardar/Borrar/Limpiar ocultos por JS)        |
+--------------------------------------------------------------------------------------+
| [Contactos]  [Lead confirmados]                                                      |
+--------------------------------------------------------------------------------------+
|  RESUMEN GENERAL       |   FILTROS                                                   |
|  +------------------+  |   +-------------------------------------------------------+ |
|  | [Limpiar filtros]|  |   | Fuente v | Nombre lista       | [Mas filtros][Buscar] | |
|  +------------------+  |   +-------------------------------------------------------+ |
|  | (icono) filtro 1 |  |                                                             |
|  |  descripcion     |  |   Mi base de datos de contactos [ filtro activo ]           |
|  |  1,234           |  |   +-------------------------------------------------------+ |
|  |  contactos ...(:)|  |   | [x] | Acciones | Perfil | URL | nombre_completo | ... | |
|  +------------------+  |   |-----+----------+--------+-----+-----------------+-----| |
|  | filtro 2 activo  |  |   | []  | (ojo)... | (img)  | (^) | Juan Perez      | ... | |
|  +------------------+  |   +-------------------------------------------------------+ |
|                        |   (GridView paginado, PageSize=30, sticky header)           |
+--------------------------------------------------------------------------------------+

MODAL: Configurar filtro avanzado (cl-fb-modal)
+--------------------------------------------------------------------------------------+
| [1] Definir condicion                                                                 |
|   Fuente v | Campo v | Condicion v | Valor _______ | [+]                              |
|   Conector siguiente: [AND|OR|AND NOT]                                                |
| [2] Condiciones activas (GridFiltrosActivos)                                          |
|   AND (Ciudad LIKE 'Medellin')                                                        |
|   OR  (Cargo LIKE 'gerente')                                    [trash]               |
| [3] Guardar como preset (opcional)                                                    |
|   Nombre_______ Descripcion______  [Guardar preset] [Eliminar]                        |
| [Limpiar todo]                                    [Aplicar busqueda avanzada]         |
+--------------------------------------------------------------------------------------+

MODAL: Detalle contacto (PanelDetalleContacto -> ctrlGestionLoader)
MODAL: Configurar acciones (PanelAcciones -> ucWorkflowDesigner) [Guardar Workflow]
```

Nota: el usuario **no sube archivos**. No hay `FileUpload`, no hay preview de columnas, no hay wizard de mapeo. Es un lector/segmentador sobre datos previamente sembrados por N8N.

## 4. Formatos de entrada soportados

**No hay carga de archivos.** El termino "Loader" es enganoso: se refiere a que los contactos son cargados por un pipeline externo (n8n) y este modulo los explora. Las fuentes disponibles en el dropdown `cmbfuentedatos`:

| Valor    | Origen real                     | Notas                                        |
|----------|---------------------------------|----------------------------------------------|
| (vacio)  | Todo                            | Comportamiento tratado como LINKEDIN         |
| LINKEDIN | Scraping N8N de perfiles        | Datos ricos en JSON dentro de `DATA`         |
| MAPS     | Scraping N8N Google Maps        | Datos en columnas REF2..REF6                 |
| FACEBOOK | (declarada en combo, sin query) | No implementada en `CargarGrid`              |
| INSTAGRAM| (declarada en combo, sin query) | No implementada en `CargarGrid`              |

El modal de busqueda avanzada solo ofrece `LINKEDIN` y `MAPS`.

## 5. Controles servidor (principales)

Los del `.aspx` (contenedor):

| ID                    | Tipo               | Rol                                    |
|-----------------------|--------------------|----------------------------------------|
| topBar                | MasterTopBar       | Titulo modulo, breadcrumb, permisos    |
| ModalDialog           | ModalDialog        | Mensajes globales                      |
| HiddenFieldsucursal   | HiddenField        | Sucursal del usuario                   |
| ctrlParrillaContactLead | UserControl      | Nucleo funcional                       |

Los del UserControl `ctrlParrillaContactLead.ascx`:

| ID                        | Tipo             | Rol                                            |
|---------------------------|------------------|------------------------------------------------|
| cmbfuentedatos            | DropDownList     | Fuente principal (autopostback)                |
| txtidentificacion         | TextBox + AutoComplete | Busqueda por nombre de lista            |
| btnBusquedaAvanzada       | LinkButton       | Abre modal filtros                             |
| Linkbutton1               | LinkButton       | Boton Buscar (no ejecuta accion util)          |
| LinkbuttonGuardar         | LinkButton       | Recarga grilla                                 |
| btnQuitarFiltros          | LinkButton       | Reset de filtros                               |
| GridFiltro                | GridView         | Grilla principal, AutoGenerateColumns=True, PageSize=30 |
| lvFiltrosGuardados        | ListView         | Sidebar de presets con conteo                  |
| litSinFiltros             | Literal          | Empty state del sidebar                        |
| LiteralTituloTabla        | Literal          | Titulo dinamico con badge de filtro activo     |
| LiterarError              | Literal          | Zona de errores                                |
| ModalctrFiltro            | Panel modal      | Constructor de condiciones                     |
| ddlFiltroFuenteDatos      | DropDownList     | Fuente para el constructor (autopostback)      |
| ddlFiltroCampo            | DropDownList     | Campos dinamicos segun fuente                  |
| ddlFiltroOperador         | DropDownList     | LIKE / = / <> / STARTS                         |
| ddlFiltroConector         | DropDownList     | AND / OR / AND NOT                             |
| txtFiltroValor            | TextBox          | Valor a comparar                               |
| btnAgregarCondicion       | LinkButton       | Agrega condicion a `ListaFiltros`              |
| GridFiltrosActivos        | GridView         | Condiciones acumuladas                         |
| txtFiltroNombreGuardar    | TextBox          | Nombre del preset                              |
| txtFiltroDescGuardar      | TextBox          | Descripcion del preset                         |
| btnGuardarFiltro          | LinkButton       | Guarda/actualiza preset                        |
| btnAplicarFiltros         | LinkButton       | Aplica filtro y recarga grilla                 |
| btnLimpiarFiltros         | LinkButton       | Limpia condiciones                             |
| PanelDetalleContacto      | Panel modal      | Ficha 360 (ctrlGestionLoader)                  |
| PanelAcciones             | Panel modal      | Workflow (ucWorkflowDesigner)                  |
| btnActivarPipeline        | LinkButton oculto| Lazy-load del tab Lead confirmados             |
| ctrlPipelineContactLoader1| UserControl      | Kanban de pipeline                             |

## 6. Eventos servidor

Ubicados en `ctrlParrillaContactLead.ascx.vb`:

- `Page_Load` -> `ListasFuente()`, `CargarFiltrosGuardados()`
- `cmbfuentedatos_SelectedIndexChanged` -> `LimpiarTodosLosFiltros` + `ParrillaCargaContactos` + `CargarFiltrosGuardados`
- `ddlFiltroFuenteDatos_SelectedIndexChanged` -> `ActualizarCamposPorFuente`
- `btnBusquedaAvanzada_Click` -> `AbrirModalBusquedaAvanzada`
- `btnAgregarCondicion_Click` -> `AgregarCondicionFiltro`
- `btnEliminarFiltro_Click` -> `EliminarCondicionFiltro(index)`
- `btnLimpiarFiltros_Click` -> `LimpiarTodosLosFiltros`
- `btnAplicarFiltros_Click` -> `ProcederAplicarBusquedaAvanzada`
- `btnGuardarFiltro_Click` -> `GuardarFiltro` (INSERT o UPDATE)
- `btnEditarFiltro_Click` -> `PrepararEdicionFiltro(ID_FILTRO)`
- `LinkEliminarFiltro_Click` -> `EliminarFiltroGuardado(ID_FILTRO)`
- `btnCargarFiltro_Click` -> `RecuperarFiltroGuardado(ID_FILTRO)`
- `btnPrepararAcccionesRow_Click` -> inicializa `ucWorkflowDesigner` y abre `PanelAcciones`
- `btnQuitarFiltros_Click` -> reset total del estado de filtros
- `GridFiltro_PageIndexChanging` -> paginacion + recarga
- `GridFiltro_RowDataBound` -> oculta columnas tecnicas
- `LinkButton13_Click` -> abre modal detalle
- `ctrlPipelineContactLoader1_VerContactoDetalle` -> abre modal detalle desde pipeline
- `btnActivarPipeline_Click` -> lazy-load del pipeline

## 7. Mapeo columnas -> campos destino

No hay mapeo tipo cargador. El SQL de la grilla proyecta valores del JSON crudo (columna `DATA` en `N8N_NAVEGACION_R`) a columnas visibles:

### LINKEDIN

| Columna visible               | Origen                                                        |
|-------------------------------|---------------------------------------------------------------|
| PEDREG                        | `N8N_NAVEGACION_R.PEDREG`                                     |
| TRABAJO_EN_EMPRESA (SI/NO)    | Cruce con tabla temporal @table por REG + FRASE_BUSQUEDA      |
| nombre_completo               | `JSON_VALUE(DATA,'$.nombre_completo')`                        |
| cargo_actual                  | `JSON_VALUE(DATA,'$.cargo_actual')`                           |
| comparacion_empresa           | `JSON_VALUE(DATA,'$.comparacion_empresa')`                    |
| empresa_actual                | `JSON_VALUE(DATA,'$.empresa_actual')`                         |
| telefono                      | `JSON_VALUE(DATA,'$.telefono')`                               |
| correo                        | `JSON_VALUE(DATA,'$.correo')`                                 |
| ultimos_estudios_titulo/institucion/periodo | `JSON_VALUE(DATA,'$.ultimos_estudios.*')`       |
| bayer_person / bayer_motivo   | `JSON_VALUE(DATA,'$.buyer_person' / '$.buyer_motivo')`        |
| ciudad / pais                 | `JSON_VALUE(DATA,'$.ciudad' / '$.pais')`                      |
| resumen_perfil                | `JSON_VALUE(DATA,'$.resumen_perfil')`                         |
| REG / FRASE_BUSQUEDA / ORIGEN_CONTACTOS (=PAQUETE) / ID_BUSQUEDA | `N8N_NAVEGACION.*`         |
| URL_CONTACTO / NOMBRE_CONTACTO / IMAGEN_PERFIL_CONTACTO / YA_ACTIVO_EN_LA_RED / ESTADO_CONEXION / FECHA_REGISTRO | `N8N_NAVEGACION.REF1/REF2/REF4/REF5/REF6/FECHA_REG` |

### MAPS

| Columna visible       | Origen                          |
|-----------------------|---------------------------------|
| FRASE_BUSQUEDA        | `N8N_NAVEGACION.FRASE_BUSQUEDA` |
| NOMBRE_EMPRESA        | `N8N_NAVEGACION.REF2`           |
| PAGINA_WEB            | `N8N_NAVEGACION.REF3`           |
| TELEFONO              | `N8N_NAVEGACION.REF4`           |
| URL_CONTACTO          | `N8N_NAVEGACION.REF1`           |
| IMAGEN_PERFIL_CONTACTO| Cadena vacia                    |
| REG                   | `N8N_NAVEGACION.REG`            |

Columnas ocultadas en `RowDataBound`: `IMAGEN_PERFIL_CONTACTO, URL_CONTACTO, REG, PEDREG, ID_BUSQUEDA, PAQUETE, REF1, REF2, REF4, REF5, REF6, FECHA_REGISTRO` (se usan solo internamente o en TemplateFields).

## 8. Validaciones por fila

Practicamente ninguna. El modulo asume datos ya limpiados por N8N. Solo hay:

- Filtro fijo `WHERE JSON_VALUE(DATA,'$.nombre_completo') IS NOT NULL` (descarta registros LinkedIn sin nombre).
- Filtro por sucursal: `AND N8N_NAVEGACION.SUCURSAL = Session("Empresa")`.
- `AgregarCondicionFiltro`: exige `txtFiltroValor` no vacio.
- `GuardarFiltro`: exige `txtFiltroNombreGuardar` no vacio y `ListaFiltros.Count > 0`.
- Escape SQL: solo `Replace("'", "''")` (SQL inline, sin parametros).

No hay validacion de email, telefono, formato ni duplicados de contacto.

## 9. Tablas SQL destino

Base fija en codigo: `dbn8n.dbo.*` (no usa alias `[dbx.GENE]`).

- **`dbn8n.dbo.N8N_NAVEGACION`**: cabecera del scraping. Columnas usadas: `REG, SUCURSAL, PAQUETE, FRASE_BUSQUEDA, ID_BUSQUEDA, REF1..REF6, FECHA_REG`.
- **`dbn8n.dbo.N8N_NAVEGACION_R`**: detalle con JSON crudo. Columnas usadas: `PEDREG, DATA (nvarchar(max) JSON)`. Relacion: `N8N_NAVEGACION_R.PEDREG = N8N_NAVEGACION.REG`.
- **`dbn8n.dbo.N8N_FILTROS_GUARDADOS`**: presets del usuario. Columnas: `ID_FILTRO, SUCURSAL, ID_USUARIO, NOMBRE_FILTRO, DESCRIPCION_FILTRO, FUENTE_DATOS, JSON_DATA`.

Snippets clave:

```sql
-- Grilla LINKEDIN (fragmento)
DECLARE @table TABLE (REG INT, EMPRESA VARCHAR(MAX));
INSERT INTO @table
SELECT PEDREG, JSON_VALUE(EXPERIENCIA.value,'$.empresa') AS empresa
FROM dbn8n.dbo.N8N_NAVEGACION_R
CROSS APPLY OPENJSON(DATA,'$.experiencia_laboral') AS EXPERIENCIA;

SELECT PEDREG,
  CASE WHEN EXISTS (SELECT * FROM @table A
                    WHERE A.REG = N8N_NAVEGACION.REG
                      AND A.EMPRESA COLLATE Modern_Spanish_CI_AS
                        = N8N_NAVEGACION.FRASE_BUSQUEDA COLLATE Modern_Spanish_CI_AS)
       THEN 'SI' ELSE 'NO' END AS TRABAJO_EN_EMPRESA,
  JSON_VALUE(CONVERT(nvarchar(max),DATA),'$.nombre_completo') AS nombre_completo,
  /* ... otros JSON_VALUE ... */
  N8N_NAVEGACION.REG, N8N_NAVEGACION.FRASE_BUSQUEDA,
  N8N_NAVEGACION.REF1 AS URL_CONTACTO, N8N_NAVEGACION.REF4 AS IMAGEN_PERFIL_CONTACTO
FROM dbn8n.dbo.N8N_NAVEGACION_R
LEFT JOIN dbn8n.dbo.N8N_NAVEGACION ON N8N_NAVEGACION_R.PEDREG = N8N_NAVEGACION.REG
WHERE JSON_VALUE(CONVERT(nvarchar(max),DATA),'$.nombre_completo') IS NOT NULL
  AND N8N_NAVEGACION.SUCURSAL = '{empresa}'
  {whereDinamico};

-- Grilla MAPS
SELECT FRASE_BUSQUEDA, REF2 AS NOMBRE_EMPRESA, REF3 AS PAGINA_WEB,
       REF4 AS TELEFONO, REF1 AS URL_CONTACTO, '' AS IMAGEN_PERFIL_CONTACTO, REG
FROM dbn8n.dbo.N8N_NAVEGACION
WHERE SUCURSAL = '{empresa}' {whereDinamico};

-- Guardar preset
INSERT INTO dbn8n.dbo.N8N_FILTROS_GUARDADOS
  (SUCURSAL, ID_USUARIO, NOMBRE_FILTRO, DESCRIPCION_FILTRO, FUENTE_DATOS, JSON_DATA)
VALUES ('{empresa}','{usuario}','{nombre}','{desc}','{fuente}','{jsonSerializado}');
```

## 10. Manejo de duplicados

No aplica: el modulo no inserta contactos. Los duplicados a nivel de scraping son responsabilidad del flujo N8N. Los presets se identifican por `ID_FILTRO` (autonumerico), sin control de duplicidad por nombre.

## 11. Log de errores / reporte final

Muy limitado:

- Existe un `Literal LiterarError` dentro de `UpdatePanel25` para pintar HTML de error, pero **nunca se asigna en el codigo actual**.
- `GuardarFiltro` envuelve la SQL en `Try/Catch` con catch vacio (comentario `' Manejo de error`) - la excepcion se silencia.
- `CalcularConteoContactos` en cada preset envuelve el JSON parse en `Try/Catch` y asigna `conteo = 0` en caso de fallo.
- No hay archivo de log ni bitacora persistida.
- Feedback al usuario: `toastr` global de la Master (no invocado explicitamente aqui) y el badge visual del filtro activo en `LiteralTituloTabla`.

## 12. Directivas Register / dependencias

En el `.aspx`:

```asp
<%@ Page Language="vb" MasterPageFile="~/Formularios/Modulos/Dashboard/MasterFormsii.Master"
        CodeBehind="comer_ContactLoader.aspx.vb" Inherits="GestionMovil.comer_ContactLoader" %>
<%@ Register Src="~/Controles/MasterTopBar.ascx" TagPrefix="uc3" TagName="MasterTopBar" %>
<%@ Register Assembly="AjaxControlToolkit" Namespace="AjaxControlToolkit" TagPrefix="cc1" %>
<%@ Register Src="~/Controles/form_contacto.ascx" TagPrefix="uc1" TagName="form_contacto" %>
<%@ Register Src="~/Controles/ModalDialog.ascx" TagPrefix="uc1" TagName="ModalDialog" %>
<%@ Register Src="~/Formularios/Modulos/Comercial/controles/ctrlParrillaContactLead.ascx"
            TagPrefix="uc1" TagName="ctrlParrillaContactLead" %>
```

En el UserControl:

```asp
<%@ Register Assembly="AjaxControlToolkit" Namespace="AjaxControlToolkit" TagPrefix="cc1" %>
<%@ Register Src="~/Formularios/Modulos/Tareas/controles/ucWorkflowDesigner.ascx" TagPrefix="uc1" TagName="ucWorkflowDesigner" %>
<%@ Register Src="~/Formularios/Modulos/Comercial/controles/ctrlGestionLoader.ascx" TagPrefix="uc2" TagName="ctrlGestionLoader" %>
<%@ Register Src="~/Formularios/Modulos/Comercial/controles/ctrlPipelineContactLoader.ascx" TagPrefix="uc3" TagName="ctrlPipelineContactLoader" %>
```

Assets externos: DataTables 1.10.20 + Buttons 1.4.2 (CDN), moment.js locales, spinners de Metronic, FontAwesome 4.7 y `style.bundle.css`. Servicio de autocompletado ASMX: `Servicios/Form/fm_autocompletado.asmx`. No hay `FileUpload`, no hay OpenXml, no hay `ImportarExcel`.

## 13. Riesgos / deuda tecnica

1. **SQL Injection**: toda la SQL se arma por concatenacion (`Session("Empresa")`, `Session("Usuario")`, `IDE_Usuario`, valores del ID de filtro, valores de texto) con solo `Replace("'","''")`. Vulnerable si algun campo permite comillas dobles o si `Session("Empresa")` fuera controlado por el usuario.
2. **Base hardcoded**: usa `dbn8n.dbo.*` directamente, sin alias `[dbx.GENE]`. Rompe la portabilidad entre entornos que exige `AdmDatos.PreparaCadena`.
3. **AutoGenerateColumns=True + RowDataBound para ocultar**: es fragil, dependiente del texto del header en mayusculas y del orden de columnas. Cambios menores en el SELECT rompen la visualizacion.
4. **CatchAll vacio**: `GuardarFiltro` traga excepciones sin notificar al usuario ni loguear.
5. **`FACEBOOK` e `INSTAGRAM`** aparecen en el combo pero `CargarGrid` no las implementa: seleccionarlas devuelve grilla vacia sin mensaje.
6. **`Linkbutton1_Click`** esta declarado pero vacio; el boton visible "Buscar" no dispara nada util (el postback de `cmbfuentedatos` es el que recarga).
7. **Comentario del handler en el aspx** apunta a `OnClick="Linkbutton1_Click"` (ok) pero la logica real de busqueda esta en `LinkbuttonGuardar_Click`. Confuso.
8. **ViewState pesado**: `ListaFiltros` como `List(Of FiltroCondicion)` serializable en ViewState. Con muchas condiciones o valores largos puede inflar el payload.
9. **Conteo por preset ejecuta un `COUNT(*)`** por cada tarjeta cargada en el sidebar: N+1 en el `Page_Load` inicial.
10. **`GenerarFiltroSQL` lee `ddlFiltroFuenteDatos.SelectedValue`** para decidir la expresion; si el filtro se cargo desde un preset y el modal esta cerrado, el estado del combo debe estar sincronizado - `RecuperarFiltroGuardado` lo hace pero `ParrillaCargaContactos` desde el combo principal usa `cmbfuentedatos`, quedando implicita la coherencia entre ambos.
11. **Modulo del topbar hard-coded** (`000873`, idioma `EN`) sin lectura del maestro de modulos.

## 14. Puntos de reconstruccion en Claude Design

Al reconstruir este modulo:

1. **Reencuadrar el nombre**: "ContactLoader" NO es un cargador tipo Excel. Es un **explorador/segmentador de contactos capturados por N8N**. Considerar renombrar a `comer_ContactExplorer` o similar para no confundir con `ctrlGestionLoader` / `ctrlPipelineContactLoader`.
2. **Extraer capa de datos**: crear un `ContactosRepository` (o servicio) que exponga `BuscarContactos(fuente, filtros, empresa, page)` y `Preset CRUD`, usando parametros SQL. Elimina la concatenacion inline.
3. **Reconstruir el constructor de filtros como componente reutilizable** (`ctrlFiltroDinamico`): recibe metadatos de campos por fuente y emite `List(Of FiltroCondicion)`. Puede alimentar tambien el pipeline y otros modulos comerciales.
4. **Definir el catalogo de campos por fuente en datos**, no en `Select Case` hardcoded. Tabla `N8N_META_CAMPOS(FUENTE, CAMPO, NOMBRE_VISIBLE, EXPRESION_SQL, TIPO_OPERADORES)`. Cierra el gap FACEBOOK / INSTAGRAM.
5. **Grilla**: pasar a `AutoGenerateColumns=False`, declarar `TemplateField`/`BoundField` visibles explicitamente, eliminar el hack de `RowDataBound`. Ideal: reemplazar por DataTables server-side con AJAX handler ASHX.
6. **Modal de detalle**: mantener el patron `ctrlGestionLoader` como Ficha 360, pero cargar via `WebMethod` o `ASHX` para no forzar postback completo.
7. **Pipeline lazy-load**: conservar el patron `btnActivarPipeline` oculto disparado por `shown.bs.tab`; funciona bien.
8. **Presets**: agregar columnas `FECHA_CREACION`, `FECHA_MODIFICACION`, `ULTIMO_USO`, `COMPARTIDO_SUCURSAL`. Reemplazar el `COUNT(*)` por preset por una tabla materializada (`N8N_FILTROS_CONTEO`) refrescada por job, o por caching en memoria.
9. **Errores**: sustituir el `LiterarError` sin uso por integracion `toastr` + escritura a un log via `MotherData` (`APP_LOG_ERRORES`).
10. **Workflow (`ucWorkflowDesigner`)**: dejar el modal tal cual pero pasar tambien el conjunto de contactos filtrados (no solo `ID_FILTRO`) para que el disenador pueda ejecutar en el snapshot exacto.
11. **Namespace / codebehind**: dejar el aspx delgado (Page_Load solo carga topbar), TODA la logica en el UserControl. Ese patron ya esta bien.
12. **Codificacion**: UTF-8 con BOM (65001) en todos los archivos; no envolver el codigo VB en `Namespace` (regla del proyecto).
