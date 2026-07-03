---
tipo: vision-libreria
libreria: Funciones
proyecto_path: C:\Desarrollo\core\Funciones\
target_origen: .NET Framework 4.8.1
target_destino: .NET 10 (servicios inyectables)
volumetria: ~80 archivos .vb en 40 subcarpetas (~726 MB con dependencias NuGet)
estado: adaptado a vision destino
---

# Funciones -- utilidades compartidas -> servicios .NET inyectables

> [!success] Que era y en que se convierte
> **Origen:** el "kitchen sink" de utilidades de la plataforma Bitcode. Cuelga TODA
> integracion externa, helpers, motores de reglas, cifrado, OCR, IA, mail,
> WhatsApp, Excel, PDF, redes sociales, Azure, AWS. La consume cada `.aspx.vb` que
> necesita algo mas alla del CRUD. Namespace raiz `Funciones`, usado como
> `Funciones.ValidaError`, `Funciones.permisos`, `Funciones.clChatGPT`, etc.
>
> **Destino (.NET 10):** cada dominio se reencarna como **servicio inyectable**
> (`IChatGptClient`, `IEmailSender`, `IExcelExporter`, `IPermissionService`, ...)
> registrado en el contenedor DI y consumido por constructor -- no via
> instanciacion `New Funciones.X` en la pagina. Ver [[Visión y entorno|Vision y entorno]] y
> [[00 - Visión MotherData|00 - Vision MotherData]].

## 1. Catalogo por dominio -> servicio destino

### Inteligencia Artificial / Bots

| Carpeta | Archivos | Para que | Servicio .NET 10 |
|---|---|---|---|
| `ChatGPT/` | `clChatGPT.vb`, `clGPTPerTasks.vb` | Wrapper OpenAI (Flujos, agentes IA) | `IChatGptClient` (Azure.AI.OpenAI) inyectable |
| `Bot/` | `MLDocumentBot.vb` | Bot de documentos ML | `IDocumentBot` |
| `EvoApi/` | `clEvoApi.vb` | Cliente Evolution API (WhatsApp) | `IWhatsAppSender` (typed `HttpClient`) |
| `NCalc/` | `FormulaInterpreter.vb`, `FormulasPersonalizadas.vb` | Motor de formulas custom (NCalc) | `IFormulaEvaluator` (motor de reglas destino) |

### Cloud providers

| Carpeta | Archivos | Para que | Servicio .NET 10 |
|---|---|---|---|
| `AzureBlobStorage/` | BlobStorage + DTO + Database | Blobs Azure | `IObjectStorage` (Azure.Storage.Blobs) |
| `AzureMap/` | `AzureHelper.vb` | Azure Maps | `IGeocodingService` |
| `AzureService/` | `OCRAzure.vb`, `ComputerVision.vb` | Azure Cognitive Services | `IOcrService` / `IVisionService` |
| `AmazonWebServices/` | `AWSTextract.vb` | AWS Textract OCR | `IOcrService` (impl AWS) |
| `Microsoft/` | `GraphAPI.vb` | Microsoft Graph (O365, Teams) | `IGraphClient` |

### Documentos / OCR / PDF / Excel

| Carpeta | Archivos | Para que | Servicio .NET 10 |
|---|---|---|---|
| `Documentos/` | `cierreperr.vb`, `tipdoc.vb` | `tipdoc.Consecutivo(...)` -- consecutivos por tenant | `IConsecutiveService` (o secuencias PG/SQL Server) |
| `Excel/` | `epplusExport.vb`, `FastExportingMethod.vb`, `LinqToExcelProvider.vb` | Export/import Excel (EPPlus) | `IExcelExporter` |
| `Pdf/` | `cHtmltoPDF.vb`, `PdftoHtml.vb` | PDF <-> HTML | `IPdfService` |
| `Printer/` | `GetLineXml.vb`, `GetLineXmlPdf.vb`, `HelpersPDF.vb` | Helpers de impresion | `IPrintService` |
| `LibreOffice/` | `OdsReaderWriter.vb` | ODS | (parte de `IExcelExporter`) |
| `OCRLocal/` | `OCRLocal.vb` | OCR sin cloud | `IOcrService` (impl local) |
| `TesseractEngine/` | `TesseractEngineHelp.vb` | Wrapper Tesseract | `IOcrService` (impl Tesseract) |
| `GleamTech/` | `DocumentAnalizer.vb` | Wrapper GleamTech | `IDocumentAnalyzer` |
| `Zip/` | `Compression.vb` | Zip | `ICompressionService` |

### Imagen / Visualizacion

| Carpeta | Archivos | Para que | Servicio .NET 10 |
|---|---|---|---|
| `Imagen/` | `FileSql.vb`, `ImageTools.vb`, `ManImagen.vb`, `chart_basic.vb`, `clChart.vb` | Imagenes y charts | `IImageService` / `IChartService` |

### Comunicacion

| Carpeta | Archivos | Para que | Servicio .NET 10 |
|---|---|---|---|
| `Mail/` | `MailService.vb` | SMTP generico | `IEmailSender` |
| `Notificaciones/` | `OneSignal.vb`, `mail_notificaciones.vb`, `SendSMS.vb` | Push + email + SMS | `INotificationService` (+ SignalR para tiempo real) |
| `UtilidadesVarias/` | `SendMailCl.vb`, `SendMailGrid.vb` | SendGrid | `IEmailSender` (impl SendGrid) |
| `Slack/` | `SlackClient.vb` | Webhook Slack | `ISlackNotifier` |
| `WhatsApp/GupShup/` | 5 archivos (WS, Helper, DTO, Database, SqlManager, 2FA) | GupShup (WA) | `IWhatsAppSender` (impl GupShup) |
| `WhatsApp/Nexmo/` | `NexmoHelper.vb` | Nexmo/Vonage | `ISmsSender` |

### Seguridad / Auth / Permisos

| Carpeta | Archivos | Para que | Servicio .NET 10 |
|---|---|---|---|
| `Seguridad/` | `MD5.vb`, `permisos.vb`, `PermissionsManager.vb`, `Token.vb` | `PermissionsManager.ValidaPermiso(...)` -- ACL en cada Page_Load | `IPermissionService` + `[Authorize]` / policies .NET |
| `Cryptography/` | `HelpersCrypto.vb` | Helpers crypto | `IDataProtectionProvider` / AES-256-GCM (ver [[AdmCrypto - Cifrado simétrico|AdmCrypto - Cifrado simetrico]]) |
| `Login/` | `Login.vb` | Helpers de login | ASP.NET Core Identity / JWT |
| `Validacion/` | `ValidaError.vb` | `Funciones.ValidaError` -- validacion con UI | FluentValidation + validacion Blazor |

### Reglas / Logica de negocio

| Carpeta | Archivos | Para que | Servicio .NET 10 |
|---|---|---|---|
| `Reglas/AdminReglas/` | `cl_manejador_Reglas.vb` | Motor de reglas administrativas (`gen_reglas.aspx`) | `RulesEngine` (ver [[Visión y entorno|Vision y entorno]] sec. 9) |
| `Reglas/Documentales/` | `cl_doc_reglas_documental.vb` | Reglas del modulo documental/formularios | `RulesEngine` (verbos Ensamblado) |
| `Negocio/` | `Calculo_DV.vb` | Digito verificacion NIT colombiano | `INitValidator` |

### API / Web / Redes / UI

| Carpeta | Archivos | Servicio .NET 10 |
|---|---|---|
| `ApiRest/` | `api_help.vb` | typed `HttpClient` |
| `JSon/` | `FJSON.vb` | `System.Text.Json` |
| `Xml/` | `FXML.vb` | `System.Xml.Linq` |
| `MasteApis/` | `ApiMaster.vb`, `FirefoxBrowser.vb` | `IHttpClientFactory` |
| `Otras/` | `ResponseHelper.vb` | middleware de respuesta |
| `String/` | `Cadena.vb` | helpers estaticos |
| `RedesSociales/` | `IBuffer.vb`, `OAuthUtil.vb`, `twitter.vb` | `ISocialClient` |
| `Stream/` | `man_stream.vb` | `Stream` nativo |
| `Controles/` | `TimerEX.vb` | componentes Blazor / SignalR |
| `Linq/` | `LinqManager.vb` | LINQ nativo |

## 2. TOP utilidades que se ven en CADA modulo (origen -> destino)

| Llamada legacy | Para que | Destino .NET 10 |
|---|---|---|
| `PermManager.ValidaPermiso(usuario, empresa, "PERMITE_XXX", modulo)` | ACL, devuelve Boolean | `IPermissionService` + policy `[Authorize(Policy="...")]` |
| `MValidaError.empty_field(...)` + `sms_nerror(topBar)` | Validacion con UI (toastr) | FluentValidation + toasts Blazor |
| `mantipo.Consecutivo("0D7", "", Empresa)` | Consecutivo por tenant | `IConsecutiveService` / secuencias BD |
| `Dim ChatGPT As New Funciones.clChatGPT` | OpenAI | `IChatGptClient` inyectado |
| `Imports Funciones.Optimizer` (`ComboFill`) | UI helpers (vive en `Bootstrap`) | componentes Blazor + `IEcorexDbContext` |

## 3. Dependencias externas (NuGets del origen)

`Newtonsoft.Json`, `Npgsql` (via MotherData), `EPPlus`, `Tesseract` +
`OpenCvSharp4`, `OneSignal SDK`, `Slack.Webhooks`, `Azure.AI.OpenAI`,
`Azure.Storage.Blobs`, `Azure.AI.Vision.*`, `AWSSDK.Textract`, `HtmlAgilityPack`,
`NCalc`, SMTP (MailKit o equiv.), `GleamTech.DocumentUltimate`. Tamano total:
**726 MB** (mayoria binarios `bin/`, `obj/`, `packages/`).

En destino se prefieren los **NuGets oficiales** directamente y `System.Text.Json`
en lugar de wrappers propios; cada dependencia se registra en DI.

## 4. Regla de altura (cuando NO usar Funciones)

- Acceso a datos: usa **[[00 - Visión MotherData|MotherData]]** ->
  `IEcorexDbContext`, no Funciones.
- UI: los componentes (`ctrFormCreator`, `crtFlujoProcesos`, `ctrGridTareas`,
  `topBar`, `sideNavBar`) viven en `Bootstrap/` -> Blazor en el destino, no aqui.

## 5. Para migracion

1. **Inventariar uso real**: muchos wrappers no se usan o solo en 1-2 sitios;
   feature-flag antes de migrar.
2. **Reemplazar por NuGets oficiales** donde la SDK ya haga el trabajo limpio.
3. **`Seguridad/permisos` + `PermissionsManager`** -> `IPermissionService` +
   `[Authorize]` declarativo (policies .NET).
4. **`tipdoc.Consecutivo`** -> `Guid` v7 o secuencias PostgreSQL/SQL Server.
5. **`Reglas/*`** -> portar al `RulesEngine` del motor de formularios/flujos destino.

Ver tambien [[00 - Visión MotherData|00 - Vision MotherData]] y [[Visión y entorno|Vision y entorno]].
