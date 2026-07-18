---
tipo: spec-modulo
modulo: NEWFRONT_web_scraping.aspx
carpeta: Utilidades
proposito: Configurador visual de flujos de web scraping (pasos, acciones, APIs, clientes, seguimiento) persistidos en SQL para ser ejecutados por el runtime WPF (Doom / consumowebpage).
---

# NEWFRONT_web_scraping - Spec para reconstruir

## 1. Que hace este modulo (2-3 parrafos)

`NEWFRONT_web_scraping.aspx` es el **configurador (front-end de diseno)** de un motor propio de web scraping / RPA. No ejecuta scraping por si mismo: sirve para definir de forma declarativa un *proceso de extraccion* compuesto por (a) un maestro (`WEB_SCRAPING`), (b) una lista ordenada de pasos-URL (`WEB_SCRAPING_R`), (c) para cada paso una lista de acciones JavaScript / SQL / consumo de API a ejecutar (`WEB_SCRAPING_RS`), (d) advertencias / etiquetas que detienen o notifican el proceso (`WEB_SCRAPING_RA`), (e) clientes/credenciales cifradas con sus variables (`WEB_SCRAPING_CLI` + `WEB_SCRAPING_RAV`), (f) definiciones de APIs con variables (`WEB_SCRAPING_API` + `WEB_SCRAPING_APIAV`) y (g) elementos en seguimiento tipo trading (`WEB_SCRAPING_SEGUIMIENTO`).

El usuario disena el flujo desde la web (ASP.NET WebForms) y el **runtime** que consume esa configuracion vive en el proyecto `Doom` (WPF), en los modulos `consumowebpage*.xaml.vb` (I..VI), `bot_metrotechnologyExcel`, `bot_impresitemExcel`, etc. Cada uno abre un `WebBrowser` de escritorio, itera los pasos, hace `document.execScript` o `InvokeScript("eval", ...)` con el `SCRIPT` de cada accion, y guarda resultados en el contenedor destino (`GEN_DATAWARE`) o en tablas/variables/APIs segun el `TIPO` (Tabla, TablaID, Variable, Api, WeBresponse, EjecutarSQL, Exploracion, Ensamblado, mouse, tramite, "cerran dron").

Por tanto este ASPX no usa `HttpClient`, `HtmlAgilityPack` ni `Selenium`: es un **CRUD de configuracion**. La ingesta de HTML se hace por script inyectado en un `WebBrowser` WPF del lado cliente pesado, sobre `.NET Framework 4.8.1`.

## 2. Ubicacion / consumidores

- **Ruta fisica:** `Bootstrap/Formularios/Modulos/Utilidades/NEWFRONT_web_scraping.aspx`
- **Modulo Ecorex:** `000730` - EXTRACCION DE DATOS (`topBar.Modulo = "000730"`)
- **Ubicacion menu:** `Home/datos/Extraccion de datos`
- **Master:** `~/Formularios/Modulos/Dashboard/MasterFormsii.Master`
- **Inherits:** `GestionMovil.NEWFRONT_web_scraping`
- **Consumidores del schema:**
  - `Doom/Modulos/Servicios/consumowebpageII..VI.xaml.vb` - motor de ejecucion WPF
  - `Doom/Modulos/Servicios/consumoweb_webserver.xaml.vb`
  - `Doom/Modulos/Bitcode/bot/bot_metrotechnologyExcel.xaml.vb`, `bot_impresitemExcel.xaml.vb`
  - `Bootstrap/Servicios/sweb_cotps*.ashx.vb` y `sweb_cotpsapi.asmx.vb` (endpoints para reportar estado)
  - `Bootstrap/Formularios/Modulos/Cotps/cotps_registro.aspx.vb`, `cotps_pagos.aspx.vb` (consumidores de datos ya scrapeados)

## 3. Layout visual (ASCII wireframe)

```
+---------------------- MasterTopBar (Guardar/Borrar/Limpiar OCULTOS) --------+
| Tab: [Paginas] [Detalle]                                                   |
+----------------------------------------------------------------------------+
| Codigo (DDL, AutoPostback) | Nombre | Ciclo | Estado (DDL)                 |
| Descripcion (multiline)                                                    |
|                                                                            |
| [+] Clientes           (accordion collapse1)                               |
|     [Agregar configuracion]  [Eliminar]                                    |
|     GridClientes: Nombre | Referencia | Token | [Editar]                   |
|                                                                            |
| [+] Configuracion      (accordion collapse0 = pasos del flujo)             |
|     [Agregar configuracion]  [Eliminar]                                    |
|     GridParametros: Orden | Tabla | Desde | Hasta | Espera | [Editar]      |
|                                                                            |
| [+] Apis               (accordion collapse3)                               |
|     [Agregar configuracion]  [Eliminar]                                    |
|     GridGesApis: Nombre | [Editar]                                         |
|                                                                            |
| [+] Datos Evaluacion   (accordion collapse4 - seguimiento tipo trading)    |
|     [Agregar Seguimiento] [Eliminar]                                       |
|     GridViewTrading: ID | Estado | [Editar]                                |
+----------------------------------------------------------------------------+
Tab Detalle: GridDetalle (DataTables + botones copy/csv/excel/pdf/print)
+----------------------------------------------------------------------------+

Modales (bootstrap 5, new bootstrap.Modal(...).show()):
+ PanelParametro (paso URL)
    Nombre | Orden | URL paso | Espera | Relevo | [ ] Repetir
    [ ] URL de login dinamica   [ ] No requiere renavegar
    (+) SQL exploracion (SQL_EXPLORA - TextArea)
    (+) Advertencias: Etiqueta | Accion (Notificar/Detener) -> GridAdvertencia
    (+) Ejecutar: Script | Orden | Espera | Tipo Resultado
                  Api | Contenedor | Nombre Variable
                  Condicion | Valor | Desde | Hasta | Operacion -> GridAcciones
+ PanelCliente
    Nombre | Correo | Slack | Token (autogenerado) | Estado | Referencia
    Variables: nombre + valor cifrado -> GridVariables
+ PanelApis
    Nombre | XML_CONFIG (multiline)
    Variables: nombre + valor -> GridViewApiVar
+ PanelTRading (Seguimiento)
    IdTrading | Estado
```

## 4. Controles servidor (tabla)

| ID | Tipo | Proposito |
|----|------|-----------|
| topBar | MasterTopBar | Barra superior; oculta Guardar/Borrar/Limpiar por CSS |
| ModalDialog | ModalDialog.ascx | Mensajes de confirmacion |
| cmbcodigo | DropDownList AutoPostback | Selector del maestro `WEB_SCRAPING.CODIGO` |
| txtnombre / txtdescripcion / txttiempociclo | TextBox | Datos del maestro |
| cmbestado | DropDownList | ACTIVO / DESACTIVADO / ERROR |
| GridClientes | GridView | Lista `WEB_SCRAPING_CLI` |
| GridParametros | GridView | Lista pasos `WEB_SCRAPING_R` |
| GridGesApis | GridView | Lista `WEB_SCRAPING_API` |
| GridViewTrading | GridView | Lista `WEB_SCRAPING_SEGUIMIENTO` |
| GridDetalle | GridView | Maestros existentes (tab Detalle, con DataTables) |
| PanelParametro / PanelCliente / PanelApis / PanelTRading | Panel modal | Popups Bootstrap |
| GridAdvertencia / GridAcciones / GridVariables / GridViewApiVar | GridView dentro de modales | Detalles del paso |
| txturlpaso, txtesperaurl, txtordenpaso, txtrelevo | TextBox | Paso URL |
| CheckRepetir, CheckBoxurltoken, Checknonavegar | CheckBox | Flags del paso |
| txtsqlproceso | TextBox multiline | SQL_EXPLORA - consulta que alimenta el paso |
| txtetiqueta / cmbaccion | TextBox+DDL | Advertencias HTML (Notificar/Detener) |
| txtscrip | TextBox multiline | Script JS a inyectar |
| cmbtiporesultado | DropDownList | Tabla / TablaID / Variable / Api / WeBresponse / EjecutarSQL / Exploracion / Ensamblado / mouse / tramite / cerran dron |
| cmbcontenedor | DropDownList (GEN_DATAWARE) | Destino del resultado |
| cmbapiservice | DropDownList | API a invocar (poblado desde `WEB_SCRAPING_API`) |
| cmbacciones | DropDownList | Operacion adicional (ej. "Salir de Relevo") |
| cmbcondicion | DropDownList | =, >, >=, <=, <> |
| txtvalor / txtinicio / txtfinal / txtscripVariable | TextBox | Condicion, rango paginacion, nombre variable resultado |
| txtnombrecliente / txtcorreo / txtslack / txttoken / txtreferencia | TextBox | Cliente |
| cmbestadocliente | DropDownList | Activo/Inactivo/Reiniciar/Pausado/Actualizacion |
| txtnombreApi / txtModeloApi | TextBox | API + payload XML |
| txtidtrading / cmbestadotrading | TextBox+DDL | Seguimiento (Activo/Perdedor/Espera) |
| HiddenFieldsucursal, HiddenDelete | HiddenField | Puentes JS |

## 5. Eventos servidor (handlers)

- `Page_Load` -> `ListarCodigo`, `DetalleServicio`, `ListaEstado`, `ListaContenedores`, `ListaAcciones`, `CargarCondiciones`, `CargarResultados`, `EstadoClientes`, `OperacionesProceso`, `ListaEstadoTrading`.
- `cmbcodigo.SelectedIndexChanged` -> `VerServicio(codigo)` que a su vez llama `ConsultarDetalle`, `ConsultarAccion`, `ConsultarClientes`, `ConsultarApi`, `ListaApis`, `ConsultarSeguimiento`.
- `topBar.Save` -> `AddCatalogo` (INSERT/UPDATE con `[dbx.GENE].dbo.WEB_SCRAPING`).
- `topBar.Delete` -> `DelCatalogo`.
- `topBar.Clear` -> `Limpiar(1)`.
- Botones: `LinkbuttonAddCliente_Click`, `LinkbuttonDellCliente_Click`, `LinkbuttonGuardar_Click` (abre modal paso), `Linkbutton3_Click` (quita paso), `Linkbutton2_Click` (abre modal API), `Linkbutton10_Click1` (abre modal seguimiento).
- Modales - Paso (`AgregarDetalles`, `CargarCaso`, `LimpiarModalPaso`), Advertencia (`AgregarAdvertencia`/`QuitarAdvertencia`), Accion (`AgregarAccion`/`QuitarAccion`), Cliente (`AgregarCliente`/`CargarCliente`), Variable (`AgregarVariable`/`Quitarvariable`), API (`AgregarApi`/`CargarApi`), Variable API (`AgregarVariableApi`), Seguimiento (`AgregarSeguimiento`/`CargarSeguimiento`).
- Los modales se abren con `ScriptManager.RegisterStartupScript(... "new bootstrap.Modal(document.getElementById('...')).show();" ...)` y se refrescan via `UpdatePanel.Update()`.

## 6. Librerias de scraping usadas

**En este ASPX: NINGUNA.** No importa `HttpClient`, `HtmlAgilityPack`, `Selenium`, `AngleSharp`, `Playwright`, `Puppeteer`.

Referencias externas relevantes:
- `MotherData.AdmDatos` (`tbrec`) para toda la persistencia SQL Server.
- `MotherData.AdmCrypto.Encrypt(valor, token)` para cifrar `WEB_SCRAPING_RAV.VALOR` (variables por cliente) usando como llave el token generado con `Funciones.Token.GenerateToken(30)`.
- `Funciones.Cadena.FormatCheckBox` / `ReadDBCheckBox` para persistir flags 0/1.
- `Funciones.tipdoc.Consecutivo("S01", "", empresa)` para generar el codigo del maestro cuando el usuario no lo digita.
- `Funciones.ValidaError` para reportar validaciones al `topBar`.
- `Optimizer.ComboFill` / `TablasHead` para poblar dropdowns y decorar grids.
- Cliente: DataTables 1.10.20 + buttons 1.4.2 (copy/csv/excel/pdf/print), moment.js, textSpinners.

El *motor* real (runtime WPF `consumowebpageII..VI.xaml.vb`) usa el `System.Windows.Controls.WebBrowser` de WPF (Internet Explorer / Trident) para navegar, `InvokeScript("eval", ...)` para inyectar los `SCRIPT` almacenados, y `WebRequest` para las acciones tipo `Api` / `WeBresponse`.

## 7. Motor de parsing (que estrategia usa)

Es una estrategia **script-driven / DOM injection**, no XPath ni CSS selectors declarativos:

1. El configurador guarda un `SCRIPT` de JavaScript por accion (`WEB_SCRAPING_RS.SCRIPT`).
2. El runtime carga la URL del paso (`URL_PASO`), espera `TIEMPO` segundos, y ejecuta cada `SCRIPT` en orden `ORDEN` invocandolo dentro del navegador.
3. El resultado se materializa segun `TIPO`:
   - `Tabla` / `TablaID` -> volcar el HTML/JSON al contenedor `GEN_DATAWARE` seleccionado.
   - `Variable` -> guardar como variable de sesion (`VARIABLE`).
   - `Api` -> invocar la API definida en `WEB_SCRAPING_API` con sus variables `WEB_SCRAPING_APIAV`.
   - `EjecutarSQL` -> correr el SQL del paso (`SQL_EXPLORA`) contra `[dbx.GENE]`.
   - `WeBresponse`, `Exploracion`, `Ensamblado`, `mouse`, `tramite`, `cerran dron` -> hooks para casos especiales del runtime.
4. `WEB_SCRAPING_RA` define "advertencias": si el runtime detecta la ETIQUETA en el DOM, dispara ACCION `Notificar` (mail/slack) o `Detener`.
5. `CONDICION` + `VALOR` + `PAGINA_DESDE`/`PAGINA_HASTA` habilitan bucles de paginacion controlada.
6. `FLAG_REPETIR`, `FLAG_NONAV` y `FLAG_TOKEN` cambian el modo: repetir el paso, no renavegar, y considerar la URL como URL de login dinamica (token en query string).

## 8. Tablas SQL / persistencia

Base: `[dbx.GENE].dbo.*` (alias del catalogo GENE, resuelto por `AdmDatos.PreparaCadena`).

| Tabla | Contenido |
|-------|-----------|
| `WEB_SCRAPING` | Maestro (CODIGO, NOMBRE, DESCRIPCION, SUCURSAL, URL, DESTINO, ESTADO, CICLO). |
| `WEB_SCRAPING_R` | Pasos-URL: TABLA, INICIA, TERMINA, TIEMPO, URL_PASO, ORDEN, FLAG_REPETIR, RELEVO, FLAG_NONAV, SQL_EXPLORA, FLAG_TOKEN. |
| `WEB_SCRAPING_RA` | Advertencias por paso: PEDREG, ETIQUETA, ACCION. |
| `WEB_SCRAPING_RS` | Acciones por paso: PEDREG, SCRIPT, TIPO, CONTENEDOR, ESPERA, ORDEN, CONDICION, VALOR, PAGINA_DESDE, PAGINA_HASTA, VARIABLE, OPERACION, API_NAME. |
| `WEB_SCRAPING_CLI` | Clientes/credenciales: NOMBRE, CORREO, SLACK, TOKEN (30 chars), ESTADO, REFERENCIA. |
| `WEB_SCRAPING_RAV` | Variables del cliente (VALOR cifrado con `AdmCrypto.Encrypt` usando TOKEN como llave). |
| `WEB_SCRAPING_API` | Definiciones de APIs: NOMBRE, XML_CONFIG. |
| `WEB_SCRAPING_APIAV` | Variables de API: PEDREG, VARIABLE, VALOR. |
| `WEB_SCRAPING_SEGUIMIENTO` | Elementos monitoreados (seguimiento tipo trading): SEGUIMIENTO (id), ESTADO. |
| `GEN_DATAWARE` (referencia) | Contenedores destino donde se vuelca la extraccion. |

Todas las tablas usan `SUCURSAL = Session("Empresa")` como discriminador multi-tenant y `REG` como PK autogenerada (SQL `OUTPUT INSERTED.REG`). Consecutivo del maestro: concepto `S01` via `Funciones.tipdoc`.

## 9. Integracion con IA (si aplica)

**No hay integracion directa con `clChatGPT`** en este ASPX. El campo `XML_CONFIG` de `WEB_SCRAPING_API` acepta payloads XML/JSON arbitrarios, por lo que el operador puede configurar una API que apunte a OpenAI/Anthropic/otro LLM y usar el tipo de resultado `Api` para post-procesar HTML capturado. La logica de invocacion vive en el runtime WPF (Doom), no aqui.

## 10. Directivas Register / dependencias

Directivas de la pagina:

```
<%@ Page Language="vb" AutoEventWireup="false"
        CodeBehind="NEWFRONT_web_scraping.aspx.vb"
        Inherits="GestionMovil.NEWFRONT_web_scraping"
        MasterPageFile="~/Formularios/Modulos/Dashboard/MasterFormsii.Master"
        ValidateRequest="false" %>
<%@ Register Src="~/Controles/MasterTopBar.ascx" TagPrefix="uc3" TagName="MasterTopBar" %>
<%@ Register Assembly="AjaxControlToolkit" Namespace="AjaxControlToolkit" TagPrefix="cc1" %>
<%@ Register Src="~/Controles/form_contacto.ascx" TagPrefix="uc1" TagName="form_contacto" %>
<%@ Register Src="~/Controles/ModalDialog.ascx" TagPrefix="uc1" TagName="ModalDialog" %>
```

`ValidateRequest="false"` es obligatorio porque los campos `txtscrip`, `txtsqlproceso` y `XML_CONFIG` almacenan HTML/JS/SQL crudo. CDNs cliente: `cdn.datatables.net/1.10.20`, `cdnjs.cloudflare.com` (pdfmake, jszip). Requiere Bootstrap 5 (usa `new bootstrap.Modal(...).show()`).

## 11. Reglas de negocio y limitaciones

- Un `CODIGO` de maestro identifica un flujo completo. Al seleccionarlo se recargan pasos + acciones + clientes + APIs + seguimiento.
- Cada paso pertenece al maestro (`PEDREG = 0` en creacion, luego se referencia). El campo `RELEVO` permite encadenar reintentos con otro paso.
- Las variables del cliente se cifran con AES/BouncyCastle (via `MotherData.AdmCrypto`) usando el TOKEN del cliente como llave. Perder el token invalida el descifrado.
- El maestro puede ser reciclado por `CICLO` (segundos entre corridas) y el estado `Reiniciar` en un cliente indica al runtime que debe reiniciar sesion en la proxima corrida.
- Todo el codigo SQL se compone por concatenacion de strings (`sql_t = sql_t & "..."`) - no hay parametros preparados.
- El maestro topBar oculta Guardar/Borrar/Limpiar por CSS (`mostrarbotonestopbar`) porque el guardado real se hace desde los botones por accordion.

## 12. Riesgos legales/tecnicos (rate limit, ToS, bot detection)

- **SQL injection amplia:** todos los INSERT/UPDATE concatenan `Session("Empresa")`, `cmbcodigo.SelectedValue`, `txtscrip.Text`, `txtnombrepaso.Text`, etc. sin sanitizar; solo se hace `.Replace("'", "''")` en `SCRIPT`, `SQL_EXPLORA` y `XML_CONFIG`. Un usuario autenticado puede inyectar TSQL.
- **XSS almacenado:** el campo SCRIPT se ejecuta luego dentro del `WebBrowser` del runtime WPF; cualquier operador con acceso al configurador puede lanzar codigo arbitrario en la maquina del bot (equivale a RCE en el host que corre Doom).
- **Cifrado dependiente de token:** las credenciales del cliente son visibles a quien tenga `TOKEN` en `WEB_SCRAPING_CLI` (esta en texto claro en BD). Considerar mover TOKEN a KeyVault.
- **Terminos de servicio de los sitios objetivo:** el modulo permite scrapear cualquier URL (`URL_PASO`). No hay lista blanca ni verificacion de `robots.txt`, ni rate-limit global; solo el `TIEMPO`/`ESPERA` por paso definido por el operador. Riesgo de bloqueo por bot-detection (Cloudflare, hCaptcha, reCAPTCHA) - el runtime usa `WebBrowser` de Trident (IE11) que ya no es soportado por muchos sitios.
- **Errores tipograficos con impacto:** `Session("Emmpresa")` en `DelCatalogo` (linea 412) provoca DELETE sin filtro real por sucursal - un maestro puede borrarse de OTRA empresa si coincide el CODIGO. Riesgo alto.
- **URL de login dinamica (`FLAG_TOKEN`):** implica que el runtime obtiene una URL con token en query string. Guardar esa URL en la BD deja el token expuesto en logs.
- **Sin auditoria:** no hay tabla de log de ejecucion; solo `WEB_SCRAPING.ESTADO` y `WEB_SCRAPING_SEGUIMIENTO` para trazabilidad.

## 13. Puntos de reconstruccion en Claude Design

Al portar a .NET Core / Blazor / React:

1. **Modelar la configuracion como schema versionado:** 10 tablas se pueden colapsar en un JSON documento por flujo (`ScrapingFlow` con `Steps[]`, `Actions[]`, `Warnings[]`, `Clients[]`, `Apis[]`, `Tracking[]`).
2. **Separar configurador y runtime:** el configurador puede ser SPA (React/Blazor) contra API REST; el runtime debe migrarse a **Playwright** o **Puppeteer-Sharp** en lugar de `WebBrowser`/Trident, para soportar sitios modernos y evadir bot-detection basado en huellas.
3. **Parametrizar SQL con Dapper/EF Core** para eliminar el riesgo de inyeccion.
4. **Reemplazar el JS inyectado:** ofrecer un DSL (o selectores CSS + XPath declarativos) y solo permitir JS crudo con role admin.
5. **Cifrado:** usar `IDataProtectionProvider` de ASP.NET Core en lugar de `AdmCrypto` con token en texto claro.
6. **UI:** los 4 modales del configurador (Paso, Cliente, API, Seguimiento) se convierten en 4 wizards paso-a-paso. El grid de acciones es un `SortableJS` para reordenar por `ORDEN`.
7. **Ejecutor headless en contenedor:** correr Playwright en Docker, con colas (RabbitMQ / Hangfire) y observabilidad (traces por paso, screenshot al fallar).
8. **Seguimiento tipo trading:** promoverlo a un tablero (websocket) con estados Activo/Perdedor/Espera para vigilancia continua.
9. **Corregir bugs:** typo `Emmpresa`, validaciones vacias (todas usan `REG='0'` como cheque de existencia lo que provoca inserciones duplicadas), y las llamadas `tbrec.FindReader(sql_t, ...)` cuando en realidad se hace un INSERT (usar `Execute` o `ExecuteScalar`).
10. **Contrato claro con el runtime:** publicar el schema en OpenAPI/GraphQL para que el motor headless (nuevo) y el configurador (nuevo) queden desacoplados del modelo antiguo `[dbx.GENE].dbo.WEB_SCRAPING*`.
