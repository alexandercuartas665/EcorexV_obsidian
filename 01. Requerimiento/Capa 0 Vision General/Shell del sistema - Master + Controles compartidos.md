---
tipo: ficha-arquitectura
capa: UI shell + componentes globales
estado: vision del sistema destino (.NET 10 / Blazor) + shell del sistema origen como referencia
stack_destino: .NET 10 / ASP.NET Core / Blazor / SignalR
---

# Shell del sistema - Layout global y componentes compartidos

> El "shell" es la estructura que envuelve TODAS las pantallas del sistema: el
> chrome (rail de iconos, sidebar, barra superior, breadcrumbs, footer) dentro del
> cual cada pantalla renderiza solo su cuerpo. Este documento describe el shell
> DESTINO sobre **.NET 10 / Blazor** - el layout de doble panel del prototipo - y
> conserva al final el shell ORIGEN (WebForms `MasterFormsII`) como referencia de
> migracion. El aspecto definitivo esta en [[Visión y entorno|Prototipo Final ECOREX]]; la
> arquitectura global en [[Visión y entorno]].

---

## DESTINO (.NET 10 / Blazor)

## 1. Concepto: shell como composicion de componentes

En el origen el shell es una Master Page WebForms (`MasterFormsII.Master`) con tres
UserControls incrustados (`topBar`, `sideNavBar`, `navBarTop`) que cada `.aspx`
configura por propiedades y `FindControl`. En el destino el shell es un **layout
Blazor** (`MainLayout.razor`) compuesto por componentes reutilizables tipados, cada
uno recibiendo su estado por parametros y por servicios inyectados (DI), no por
`Session` ni por busqueda dinamica de controles. El resultado visible es el layout
de doble panel del prototipo: un rail de iconos permanente a la extrema izquierda,
un sidebar contextual `SKY SYSTEM / Plan Empresa - ECOREX`, una barra superior
global con breadcrumbs, buscador (Cmd+K), campana de notificaciones y perfil, y el
cuerpo central donde se enruta cada pagina.

La composicion recomendada:

```txt
MainLayout.razor
 |- <IconRail />            rail de iconos permanente (extrema izquierda)
 |- <ContextSidebar />      sidebar SKY SYSTEM: PRINCIPAL + MODULOS (por tenant/rol)
 |- <TopBarGlobal />        buscador Cmd+K, notificaciones, tema, perfil, salir
 |- <Breadcrumbs />         Equipos / SKY SYSTEM / <seccion>
 |- <PageActionsBar />      acciones por pagina (Guardar, Nuevo, Eliminar, Compartir...)
 |- @Body                   contenido de la pagina enrutada (RouteView)
 |- <ToastHost />           notificaciones toastr -> Blazor toast/SignalR
```

Cada componente es autonomo, testeable y se alimenta de servicios de aplicacion
(`ITenantContext`, `IMenuService`, `IPermissionService`, `INotificationHub`), no de
variables de sesion sueltas. El estado de UI compartido (tenant activo, usuario,
tema, contadores de notificaciones) vive en servicios scoped y en `CascadingValue`,
de modo que cualquier pantalla lo consume sin volver a resolverlo.

## 2. Contexto de tenant y branding

El branding y la cultura los resuelve el shell una sola vez por peticion/circuito.
Un `TenantResolutionMiddleware` (host/subdominio/claim JWT) fija el `TenantId`
(Guid) y `ITenantContext` expone nombre comercial, logo, tema y cultura. **A
diferencia del origen** (que leia `SUCURSAL` con `Session("Empresa")` concatenado en
un `SELECT`), el destino carga el branding por `IEcorexDbContext` con el filtro
global de tenant ya aplicado y cachea el resultado en Redis. La cultura `es-CO`
(separadores `.`/`,`) se configura por `RequestLocalization` a partir del tenant, no
por hilo global. El titulo de pagina (`SKY SYSTEM | <seccion>`) se compone con un
servicio `IPageTitleService` o el `<PageTitle>` de Blazor, reemplazando el patron
`Master.ExplicitTitle`.

## 3. Rail de iconos + sidebar contextual (menu por permisos)

El menu es data-driven y se filtra por policies del usuario en el tenant activo. El
`IMenuService` construye el arbol (bloques **PRINCIPAL** y **MODULOS**, cada modulo
con su codigo `000XXX`) desde el **Module Registry** (evolucion de `ADM_WEB`) y lo
poda contra el `IPermissionService` (policies .NET portadas de `PermissionsManager`;
ver [[Puntos ciegos y motores transversales]]). El resultado se cachea por
(tenant, rol) en Redis con invalidacion por evento cuando cambian permisos o
modulos habilitados.

El rail de iconos ancla las secciones raiz; el sidebar despliega el detalle
contextual de la seccion activa. Los items no permitidos **no se renderizan** (no
solo se ocultan por CSS): la autorizacion es de servidor, con `[Authorize(Policy=...)]`
en cada pagina Blazor y verificacion en el endpoint, de modo que ocultar un item
nunca es la unica defensa. El estado activo, el resaltado morado de marca y el modo
oscuro (toggle en el pie del sidebar) son estado de UI del componente.

## 4. Barra superior global, breadcrumbs y acciones de pagina

El origen mezcla dos barras: `navBarTop` (global: buscador, perfil, tareas,
alertas, tema, salir) y un `topBar` por pagina (titulo, breadcrumb, botones Guardar
/ Eliminar / Limpiar / Nuevo / Importar / Auditoria / GPT, mensajes). El destino los
separa con claridad:

- **`TopBarGlobal`**: buscador global con atajo Cmd+K (indice de modulos, usuarios,
  proyectos, tareas via un `ISearchService`), campana de notificaciones alimentada
  por SignalR, selector de tema, ficha de usuario y cierre de sesion (logout que
  revoca refresh token).
- **`Breadcrumbs`** y **`PageActionsBar`**: la ruta jerarquica y los botones de
  accion de la pantalla. En Blazor, cada pagina declara sus acciones y sus handlers
  fuertemente tipados; sustituye al patron de eventos `__doPostBack('topBar$...')`
  y a los metodos `ActivarBoton`/`ShowError`/`OcultarGuardado` por parametros y
  eventos de componente (`EventCallback`) mas un `IToastService` para mensajes.

El boton `Compartir` del prototipo y las acciones sensibles (Eliminar, Importar
masivo) pasan por autorizacion de policy y por confirmacion, y toda accion de
escritura queda registrada en la auditoria inmutable (ver seccion 6).

## 5. Tiempo real con SignalR nativo

El shell hospeda la conexion en vivo. Un **hub nativo de ASP.NET Core**
(`NotificationHub`) reemplaza al SignalR clasico de WebForms. Al iniciar el circuito
Blazor, el cliente se suscribe al **grupo por tenant** (`tenant:{TenantId}`) y a su
grupo de usuario (`user:{UserId}`), de modo que un usuario **solo** recibe eventos
de su empresa - cerrando el punto ciego del origen (canales sin aislamiento
multi-tenant). El hub autentica por JWT en el handshake y escala horizontalmente con
**Redis backplane**, superando la limitacion single-instance del origen. Los eventos
de dominio (asignacion de tarea, cambio de estado de flujo, nuevo anuncio) se
publican via RabbitMQ/MassTransit y un consumidor los reenvia al grupo de tenant, de
modo que los tableros Kanban y los badges de notificaciones se actualizan sin
recargar. El shell expone estos avisos en la campana de `TopBarGlobal` y en toasts.

## 6. Auditoria, seguridad y asistente IA del shell

Toda accion iniciada desde el shell (login, cambio de tenant, acciones de pagina)
se registra en `AdminAuditLog` inmutable (ver [[Puntos ciegos y motores transversales]]),
con `CorrelationId` propagado. La resolucion de tenant, las policies y el JWT + MFA
(para PlatformAdmin) blindan cada peticion; nunca hay fuga entre empresas. El
**asistente GPT en la topbar** que en el origen quedo comentado (`topBarBotGPT`) se
retoma como componente opcional `<AssistantWidget>` conectado a un servicio de IA
por tenant, con el contexto de modulo activo, cuota y secretos en Key Vault - nunca
en config cifrado con clave hardcodeada.

## 7. Correccion de errores heredados en el shell

- **SQL concatenado** (`SELECT NOMBRE FROM SUCURSAL WHERE CODIGO ='" & Session(...)`)
  -> consultas parametrizadas por `IEcorexDbContext`; el tenant viene del claim, no
  de `Session` manipulable.
- **`Session` como estado global** -> servicios scoped, `CascadingValue` y claims.
- **Aislamiento por columna `SUCURSAL` + disciplina** -> `TenantId` (Guid) con
  `HasQueryFilter` global + RLS; el shell no puede pintar datos de otro tenant.
- **Autorizacion solo por ocultar controles** -> policies de servidor en cada pagina
  y endpoint; ocultar en UI es cosmetico, no la defensa.
- **Menu por `FindControl` y `Session("Usuario")`** -> `IMenuService` + policies
  cacheadas; menu deterministico y testeable.

---

## ORIGEN (WebForms) - referencia de migracion

> Lo que sigue describe el shell legacy `MasterFormsII` + `topBar` + `sideNavBar` +
> `navBarTop`. Se conserva como plano del comportamiento a preservar (que hace cada
> pieza, que catalogos alimenta el menu) y como referencia del ETL. NO es el diseno
> destino.

### O.1 `MasterFormsII.Master` - Master Page

**Path:** `Bootstrap\Formularios\Modulos\Dashboard\MasterFormsII.Master`

Responsabilidades: configurar cultura `es-CO`; cargar el nombre del tenant desde
`SUCURSAL`; construir el titulo `{nombre_tenant} | {explicit_title}`; renderizar los
controles del shell (`topBar`, `sideNavBar`, `navBarTop`).

```vb
Public Class MasterFormsII
    Inherits System.Web.UI.MasterPage

    Dim tbrec As New MotherData.AdmDatos
    Dim BASE_SISTEMA As String = "SERVERI_MAR"
    Dim BASE_SUCURSAL As String
    Public Property ExplicitTitle As String

    Sub New()
        Call carga_empresa()
        BASE_SISTEMA = mbase_empresa.base_pricipal
        BASE_SUCURSAL = mbase_empresa.sucursal
    End Sub

    Protected Sub Page_Load(...)
        ' Cultura es-CO (hilo global - anti-patron en destino)
        System.Threading.Thread.CurrentThread.CurrentCulture = New System.Globalization.CultureInfo("es-CO")

        ' Nombre del tenant - SQL CONCATENADO (error heredado #1)
        Dim man_title = tbrec.FindReader(
            "SELECT NOMBRE FROM [dbx.GENE].dbo.SUCURSAL WHERE CODIGO ='" & Session("Empresa") & "'",
            mbase_empresa.base_pricipal)
        If man_title = "" Then man_title = "ECOREX"

        If String.IsNullOrWhiteSpace(Me.ExplicitTitle) Then
            Me.MasterPageTitle.Text = man_title
        Else
            Me.MasterPageTitle.Text = man_title & " | " & Me.ExplicitTitle
        End If
    End Sub
End Class
```

Cada pagina setea su titulo con `CType(Master, MasterFormsII).ExplicitTitle = "..."`
-> `<title>` = `SKY SYSTEM | FLUJO DE PROCESO`. Layout HTML inferido: un contenedor
`barramenu` con los tres controles del shell y un `ContentPlaceHolder` para el cuerpo.

> Mapeo destino: `MainLayout.razor` + `ITenantContext` + `IPageTitleService`.
> La cultura pasa a `RequestLocalization`; el SELECT concatenado a
> `IEcorexDbContext` parametrizado con `TenantId` del claim.

### O.2 `topBar` - barra superior por pagina

Provee titulo, breadcrumb, botones de accion y mensajes. Propiedades: `Title`,
`Modulo` (codigo(s) `000XXX`, separables por `;`), `Idioma`, `MenuName`, `Ubicacion`.
Eventos por `__doPostBack`: `Save`, `Delete`, `Clear`, `Nuevo`, `Importar`. Metodos:
`ClearError`, `ShowError(titulo,msg)`, `OcultarGuardado`, `ActivarBoton(nombre)`. En
`ADM_CONTROLES` figura como control `"00001"` (TOPBAR) dentro de un catalogo de 7
controles del shell.

> Mapeo destino: `PageActionsBar` + `Breadcrumbs`; eventos `__doPostBack` ->
> `EventCallback` tipados; `ShowError` -> `IToastService`.

### O.3 `sideNavBar` - menu lateral por permisos

Cada pagina lo obtiene por `Me.Master.FindControl("sideNavBar")` y le pasa
`Usuario = Session("Usuario")`. Lee permisos y muestra/oculta secciones
(OPERACIONES, NEGOCIO, AUTOMATIZACION, SISTEMA, ...). Origen probable del menu:
tabla `ADM_WEB` (~9 columnas: `CODIGO 000xxx`, `NOMBRE`, `URL`, `ICONO`, `SECCION`,
`ORDEN`, y flags de visibilidad/permisos). Esquema pendiente de confirmar.

> Mapeo destino: `IconRail` + `ContextSidebar` alimentados por `IMenuService`
> (Module Registry, evolucion de `ADM_WEB`) y podados por policies .NET.

### O.4 `navBarTop` - barra superior global

Menu top global: buscador, perfil ("Bienvenido GUADALUPE.LA"), Mis Tareas
("Tienes N Tareas"), Alertas, tema Light/Dark/Sistema, mensajes, Salir
(`LinkButtonCerrarII`). En `ADM_CONTROLES` es el control `"00003"` (NAVBARTOP).

> Mapeo destino: `TopBarGlobal` + `ISearchService` + campana por SignalR + logout
> con revocacion de token.

### O.5 Construccion del dashboard `das_module.aspx`

Secuencia al login: (1) Master carga cultura/titulo; (2) `sideNavBar` y `navBarTop`
se renderizan por permiso; (3) `topBar` de la pagina se configura (`Title="Home"`,
`Modulo="000004"` TILE MODULOS); (4) la pagina pinta la grilla de tarjetas ("Ir al
modulo") consultando `ADM_WEB` filtrado por permisos. Las secciones (OPERACIONES,
NEGOCIO, AUTOMATIZACION, SISTEMA...) son agrupadores visuales del menu.

### O.6 Patron de inicio en CADA pagina

```vb
Protected Sub Page_Load(...) Handles Me.Load
    If Not IsPostBack Then
        topBar.Title = "MI MODULO"
        topBar.Modulo = "000xxx"
        topBar.Idioma = "EN"
        topBar.MenuName = "BASIC_TOPBAR"
        topBar.Ubicacion = "Home/Seccion/Mi Modulo"

        Dim meControl As sideNavBar = TryCast(Me.Master.FindControl("sideNavBar"), sideNavBar)
        meControl.Usuario = Session("Usuario")

        Dim meNavBarTop As navBarTop = TryCast(Me.Master.FindControl("navBarTop"), navBarTop)
        meNavBarTop.Usuario = Session("Usuario")
    End If

    If Session("Nombre") = "" Then Response.Redirect("~/" & PAG_LOGIN)
End Sub
```

Plantilla estandar en NEWFRONT_doc_procesos, NEWFRONT_docu_formulario,
NEWFRONT_admtareas, NEWFRONT_proyectos, NEWFRONT_tar_tareas, gen_reglas,
docu_viewform. En el destino este boilerplate por pagina desaparece: el shell y las
policies (`[Authorize]`) resuelven tenant, usuario y menu de forma transversal.

### O.7 SignalR heredado

`NEWFRONT_admtareas.aspx.vb:1` importa `Microsoft.AspNet.SignalR`; el modulo
Notificaciones (`000288`) lo usa para refrescar grids, avisar "Tienes una nueva
tarea" y (posible) chat interno. Hub probablemente en `App_Start\Startup.vb` o
`Global.asax` (no leidos). Single-instance, sin backplane, sin grupos por tenant
confirmados. Ver seccion 5 (destino) y [[Puntos ciegos y motores transversales]].

### O.8 Asistente GPT en topbar (deshabilitado)

`NEWFRONT_admtareas.aspx.vb:235-239` tiene comentado `topBarBotGPT` (BotSucursal,
BotModule `000636`, BotUser, BotUserName, `LoadData()`). Iba a ser un asistente IA
embebido con contexto del modulo; deshabilitado. En destino se retoma como
`<AssistantWidget>` (seccion 6) con `Funciones.clChatGPT` -> servicio IA por tenant.

---

## Relacion con otras notas

- [[Visión y entorno|Prototipo Final ECOREX]] - aspecto y navegacion definitivos del shell destino
- [[Visión y entorno]] - arquitectura global y stack destino
- [[Puntos ciegos y motores transversales]] - PermissionsManager (policies), SignalR, ChatGPT, auditoria
- [[Gestion de Empresas - Admin multi-tenant]] - resolucion de tenant y branding por empresa

---

## TODO de migracion

- [ ] Confirmar esquema de `ADM_WEB` y `ADM_MOD` para modelar el Module Registry destino.
- [ ] Definir el contrato de `IMenuService` (arbol PRINCIPAL/MODULOS + poda por policy).
- [ ] Especificar canales/grupos SignalR por tenant y el backplane Redis.
- [ ] Portar `topBarBotGPT` a `<AssistantWidget>` con secretos en Key Vault.
- [ ] Mapear cada evento `__doPostBack` de `topBar` a su `EventCallback` en `PageActionsBar`.
