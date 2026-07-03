---
tipo: mapa-arbol
proyecto: 044. Tareas (ECOREX)
estado: indice visual completo del menu del sistema
relacionado: [[INVENTARIO GENERAL]] (tabla con paths + tablas SQL)
stack_destino: .NET 10 / ASP.NET Core 10 / Blazor
---

# Mapa de Modulos - Arbol ASCII

> **Arbol del menu ORIGEN, conservado como plano de navegacion a migrar.** Es la
> vista jerarquica de TODOS los menus del sistema legacy `das_module.aspx`. En el
> DESTINO (.NET 10) este arbol NO se copia literal: el sidebar del prototipo
> ([[00 - Prototipo Final ECOREX]]) reorganiza la navegacion en dos bloques
> **PRINCIPAL** (Inicio, Anuncios, Gestor de tareas, Configuracion) y **MODULOS**
> (los procesos del tenant, cada uno con su codigo `000XXX`). Los codigos de 6
> digitos se preservan como clave de negocio (`Module.LegacyModuleCode`); la
> habilitacion por tenant la gobierna el Module Registry (`000109`). Vision
> maestra en [[Visión y entorno]].
>
> Cada hoja muestra: codigo modulo (6 digitos) + path `.aspx` relativo.

```
ECOREX v1.0 (SKY SYSTEM)
|
|-- [DASHBOARD]                            das_module.aspx
|
|== OPERACIONES ====================================================
|   |
|   |-- MIS PROCESOS / GESTION COMERCIAL
|   |   |
|   |   |-- Requerimientos Equipos Computo (filtrado)
|   |   |       000477  tareas/NEWFRONT_admtareas.aspx?cat=32&sub=00477
|   |   |
|   |   |-- Crear Una Actividad
|   |   |       000038  tareas/NEWFRONT_tar_tareas.aspx
|   |   |
|   |   |-- Proyectos
|   |   |       000042  tareas/NEWFRONT_proyectos.aspx
|   |   |
|   |   |-- Administrar Actividades                       <-- VISTA PRINCIPAL
|   |   |       000636;000620;000039  tareas/NEWFRONT_admtareas.aspx
|   |   |
|   |   `-- Programar Una Actividad
|   |           000889  utilidades/adm_programador.aspx
|   |
|   `-- (vistas auxiliares)
|       |-- Mis Tareas (cliente)        tareas/NEWFRONT_tar_tareascli.aspx
|       |-- Calendario tareas           tareas/NEWFRONT_adm_calendar.aspx
|       |-- Jefe tareas                 tareas/NEWFRONT_adm_jefetareas.aspx
|       |-- Compacto (movil)            tareas/NEWFRONT_admtareas_compacto.aspx
|       |-- Encuesta servicio           tareas/NEWFRONT_tar_encuesta_servicio.aspx
|       |-- PQR                         tareas/NEWFRONT_tar_pqr.aspx
|       `-- Visita                      tareas/NEWFRONT_tar_visita.aspx
|
|== NEGOCIO =========================================================
|   |
|   |-- Clientes
|   |   |-- Creacion Clientes
|   |   |       000232  documentos/NEWFRONT_doc_clientes.aspx
|   |   |
|   |   `-- Seguimiento De Clientes
|   |           000123  cartera/car_seguimientoNEWFRONT.aspx
|   |
|   `-- OFERTA - CATALOGO
|       `-- Items De Inventarios
|               000066  inventarios/NEWFRONT_items.aspx
|
|== AUTOMATIZACION =================================================  *** CORAZON ***
|   |
|   |-- *** Flujos Del Proceso  (Editor BPMN) ***
|   |       000291  documentos/NEWFRONT_doc_procesos.aspx
|   |       Tabla: DOC_PROCESOS + _R + _RULES + _PLUGINS + _GRUPOS
|   |       Motor: AdmWorkflow + bpmn-js v18.7.0
|   |       Ver: [[00 - Visión Flujos]]
|   |
|   |-- *** Formularios  (Constructor visual) ***
|   |       000131  documental/NEWFRONT_docu_formulario.aspx
|   |       Tabla: ENCUESTAS_MOV + _PREGUNTAS + _PREGUNTAS_T
|   |       Motor: ctrFormCreator + cl_FormCreator + crtCargaEncuestaII
|   |       19 tipos de control. Ver: [[00 - Visión Formularios]]
|   |
|   |-- Analitica / Power BI
|   |   `-- Power Bi Service
|   |           000788  PowerBI/bi_servicios.aspx
|   |
|   `-- Agentes IA
|       `-- Agentes De Inteligencia Artificial
|               000867  documental/docu_viewform.aspx?token=REDACTED...
|               Clase: GestionMovil.cl_ia_reglas_formularios
|
|== SISTEMA ========================================================
|   |
|   |-- Configuracion / Inventarios
|   |   |-- Bodegas De Inventarios
|   |   |       000556  inventarios/NEWFRONT_inv_bodegas.aspx
|   |   |-- Grupo Inventarios
|   |   |       000506  Inventarios/NEWFRONT_catgrupos.aspx
|   |   |-- Marcas
|   |   |       000502  Inventarios/NEWFRONT_catmarcas.aspx
|   |   |-- Subgrupos Inventarios
|   |   |       000606  Inventarios/NEWFRONT_catsubgrupos.aspx
|   |   `-- Tipos De Inventarios
|   |           000498  Inventarios/NEWFRONT_catgrupoinvetaros.aspx
|   |
|   |-- Actividades (parametria)
|   |   |-- Prioridades
|   |   |       000621  tareas/NEWFRONT_tar_prioridades.aspx
|   |   |-- Tipos De Proyecto
|   |   |       000690  tareas/tar_tipoproyecto.aspx
|   |   |-- Actividades (conceptos)
|   |   |       000270  tareas/NEWFRONT_tar_conceptos.aspx
|   |   `-- Estados
|   |           000653  tareas/NEWFRONT_tar_estados.aspx
|   |
|   |-- CRM
|   |   |-- Conceptos Actividades
|   |   |       000125  cartera/car_actividadcomerNEWFRONT.aspx
|   |   |-- Estados (relacion comercial)
|   |   |       000272  cartera/car_estadosNEWFRONT.aspx
|   |   |-- Perfiles De Proveedor/Cliente
|   |   |       000166  cartera/car_perfilesNEWFRONT.aspx
|   |   |-- Servicios O Productos
|   |   |       000249  cartera/car_canalesNEWFRONT.aspx
|   |   |-- Tipos De Empresas
|   |   |       000231  documentos/NEWFRONT_doc_tiposempresa.aspx
|   |   |-- Vendedores
|   |   |       000124  cartera/car_vendedoresNEWFRONT.aspx
|   |   |-- Origen Clientes
|   |   |       000324  comercial/comer_origen.aspx
|   |   `-- Grupos De Actividades
|   |           000126  cartera/car_actividadgrupNEWFRONT.aspx
|   |
|   |-- General
|   |   |-- Configuracion De Entidad
|   |   |       000615  general/gen_config_empresa_cli.aspx
|   |   |-- Plantillas
|   |   |       000893  general/gen_admplantillas.aspx
|   |   |-- Dependencias (organigrama)
|   |   |       000850  general/gen_dependencias.aspx
|   |   |-- Roles Y Permisos
|   |   |       000194  general/gen_rol_administrador.aspx
|   |   `-- Administracion De Usuarios
|   |           000073  General/NEWFRONT_gen_usuarios.aspx
|   |
|   `-- Desarrollo  ***************************************************
|       |
|       |-- Autocompletado Formularios
|       |       000801  documental/docu_viewform.aspx?token=REDACTED
|       |
|       |-- Biblioteca Recursos
|       |       000133  configuracion/NEWFRONT_bib_biblioteca.aspx
|       |       Tabla: BIB_BIBLIOTECA
|       |
|       |-- Controles  (UI shell)
|       |       000282  configuracion/NEWFRONT_adm_controles.aspx
|       |       Tabla: ADM_CONTROLES + _I + _V (7 filas: topBar, sideNavBar, navBarTop, etc.)
|       |
|       |-- Flujos Del Proceso (duplicado en Desarrollo)
|       |       000291  documentos/NEWFRONT_doc_procesos.aspx
|       |
|       |-- Grupos Biblioteca De Codigo
|       |       000132  configuracion/NEWFRONT_bib_grupos.aspx
|       |
|       |-- Manejo De Los Sms De Error
|       |       000301  configuracion/NEWFRONT_adm_errores.aspx
|       |
|       |-- Modulos Web (este catalogo)
|       |       000109  Configuracion/NEWFRONT_adm_modulos.aspx
|       |       Tabla: ADM_WEB (define los 6-digit codes que aparecen en este arbol)
|       |
|       |-- Notificaciones (SignalR)
|       |       000288  tareas/NEWFRONT_tar_notificaciones.aspx
|       |
|       |-- Objetos Sistemas
|       |       000137  Configuracion/newfront_adm_objetos.aspx
|       |       Tabla: ADM_HTML + _R
|       |
|       |-- Parametros Xml
|       |       000057  configuracion/newfront_adm_par_xml.aspx
|       |       Tabla: PARAMXML
|       |
|       |-- *** Reglas (motor) ***
|       |       000802  general/gen_reglas.aspx
|       |       Tabla: CONTROL_REGLAS + _R + _F + _H + PROPIEDADES_REGLAS
|       |       Ver: [[Reglas - Motor y discrepancia]]
|       |
|       |-- Reportes
|       |       000119  reportes/newfront_reportes_admin.aspx
|       |
|       |-- Servicios Web
|       |       000053  configuracion/NEWFRONT_adm_webservice.aspx
|       |       Tabla: ADM_SERVICIOS
|       |
|       |-- *** SQL Admin (ejecuta SQL en la nube) ***
|       |       000077  Configuracion/NEWFRONT_adm_sql.aspx
|       |
|       |-- Tipos De Documentos (consecutivos)
|       |       000136  Configuracion/newfront_gen_tiposdocumento.aspx
|       |
|       |-- Tipos Tecnologias
|       |       000134  configuracion/bib_tecnologia.aspx
|       |       Tabla: BIB_TECNOLOGIA
|       |
|       `-- Pruebas Formularios
|               000891  documental/docu_viewform.aspx?token=REDACTED...
|
`-- [TOPBAR ACCIONES GLOBALES]
    |-- Mi Perfil          PERFIL DEL USUARIO  (LinkButtonPerfilUsuario)
    |-- Tareas             dropdown con "Tienes N Tareas"
    |-- Alertas            dropdown alertas de sistema
    |-- Tema               Light / Dark / Sistema
    |-- Mensajes           (chat interno - pendiente confirmar)
    `-- Cerrar Sesion      SALIR DEL SISTEMA  (LinkButtonCerrarII)
```

---

## Resumen numerico

| Seccion        | Modulos | Estado prioritario |
|----------------|---------|--------------------|
| OPERACIONES    | 5 (+7 aux) | Tareas (000038, 000636) son el core operativo |
| NEGOCIO        | 3       | Clientes + Inventarios |
| AUTOMATIZACION | 4       | **Flujos (000291) y Formularios (000131) son el "corazon"** |
| SISTEMA - Inv. | 5       | Catalogos de inventario |
| SISTEMA - Act. | 4       | Parametria de tareas |
| SISTEMA - CRM  | 8       | Catalogos comerciales |
| SISTEMA - Gen. | 5       | Usuarios + Roles + Dependencias |
| Desarrollo     | 17      | Reglas (000802), Modulos Web (000109), SQL Admin (000077) |
| **TOTAL menu** | **~51 modulos** unicos en el dashboard |

---

## Codigos por orden numerico (lookup rapido)

```
000038  Crear Actividad             tareas/NEWFRONT_tar_tareas.aspx
000039  (parte de admtareas)
000042  Proyectos                    tareas/NEWFRONT_proyectos.aspx
000053  Servicios Web                configuracion/NEWFRONT_adm_webservice.aspx
000057  Parametros XML               configuracion/newfront_adm_par_xml.aspx
000066  Items Inventarios            inventarios/NEWFRONT_items.aspx
000073  Administracion Usuarios      General/NEWFRONT_gen_usuarios.aspx
000077  SQL Admin                    Configuracion/NEWFRONT_adm_sql.aspx
000109  Modulos Web                  Configuracion/NEWFRONT_adm_modulos.aspx
000119  Reportes                     reportes/newfront_reportes_admin.aspx
000123  Seguimiento Clientes         cartera/car_seguimientoNEWFRONT.aspx
000124  Vendedores                   cartera/car_vendedoresNEWFRONT.aspx
000125  Conceptos Actividades CRM    cartera/car_actividadcomerNEWFRONT.aspx
000126  Grupos Actividades CRM       cartera/car_actividadgrupNEWFRONT.aspx
000131  *** Formularios ***          documental/NEWFRONT_docu_formulario.aspx
000132  Grupos Biblioteca            configuracion/NEWFRONT_bib_grupos.aspx
000133  Biblioteca Recursos          configuracion/NEWFRONT_bib_biblioteca.aspx
000134  Tipos Tecnologias            configuracion/bib_tecnologia.aspx
000136  Tipos de Documento           Configuracion/newfront_gen_tiposdocumento.aspx
000137  Objetos Sistemas             Configuracion/newfront_adm_objetos.aspx
000166  Perfiles Cliente/Proveedor   cartera/car_perfilesNEWFRONT.aspx
000194  Roles y Permisos             general/gen_rol_administrador.aspx
000231  Tipos de Empresas            documentos/NEWFRONT_doc_tiposempresa.aspx
000232  Creacion Clientes            documentos/NEWFRONT_doc_clientes.aspx
000249  Servicios/Productos          cartera/car_canalesNEWFRONT.aspx
000270  Actividades (conceptos)      tareas/NEWFRONT_tar_conceptos.aspx
000272  Estados CRM                  cartera/car_estadosNEWFRONT.aspx
000282  Controles                    configuracion/NEWFRONT_adm_controles.aspx
000288  Notificaciones (SignalR)     tareas/NEWFRONT_tar_notificaciones.aspx
000291  *** Flujos del Proceso ***   documentos/NEWFRONT_doc_procesos.aspx
000301  Mensajes Error               configuracion/NEWFRONT_adm_errores.aspx
000324  Origen Clientes              comercial/comer_origen.aspx
000477  Requerimientos Equipos       tareas/NEWFRONT_admtareas.aspx?cat=32&sub=00477
000498  Tipos de Inventarios         Inventarios/NEWFRONT_catgrupoinvetaros.aspx
000502  Marcas                       Inventarios/NEWFRONT_catmarcas.aspx
000506  Grupo Inventarios            Inventarios/NEWFRONT_catgrupos.aspx
000556  Bodegas de Inventarios       inventarios/NEWFRONT_inv_bodegas.aspx
000606  Subgrupos Inventarios        Inventarios/NEWFRONT_catsubgrupos.aspx
000615  Configuracion Entidad        general/gen_config_empresa_cli.aspx
000620  (parte de admtareas)
000621  Prioridades                  tareas/NEWFRONT_tar_prioridades.aspx
000636  Administrar Actividades      tareas/NEWFRONT_admtareas.aspx
000653  Estados (tareas)             tareas/NEWFRONT_tar_estados.aspx
000690  Tipos de Proyecto            tareas/tar_tipoproyecto.aspx
000788  Power BI Service             PowerBI/bi_servicios.aspx
000801  Autocompletado Formularios   documental/docu_viewform.aspx?token=...
000802  *** Reglas ***               general/gen_reglas.aspx
000850  Dependencias                 general/gen_dependencias.aspx
000867  Agentes IA                   documental/docu_viewform.aspx?token=...
000889  Programar Actividad          utilidades/adm_programador.aspx
000891  Pruebas Formularios          documental/docu_viewform.aspx?token=...
000893  Plantillas Impresion         general/gen_admplantillas.aspx
```

---

## Notas de lectura del arbol

1. **`***`** marca los modulos criticos del "corazon" del sistema (Flujos, Formularios, Reglas, SQL Admin).
2. **`-->`** son flechas de "vista principal" o lugar al que el usuario llega primero por su flujo.
3. Cada modulo tiene su **codigo de 6 digitos** que sirve para:
   - Buscar en `ADM_WEB` (000109)
   - Validar permisos via `PermissionsManager.ValidaPermiso(usuario, empresa, permiso, modulo)`
   - Filtrar `GEN_PARAMETROS` por (sucursal, modulo, codigo)
4. Algunos modulos comparten 2-3 codigos separados por `;` (ej. `000636;000620;000039` para admtareas) porque la misma pagina representa 3 modulos para fines de permisos.
5. **Desarrollo** es la seccion mas grande (17 modulos) - es el "panel de admin/configurador del sistema".
6. **Power BI** y **Agentes IA** son las dos extensiones modernas del sistema.

---

## Como esta organizado fisicamente en el disco

```
C:\Desarrollo\core\Bootstrap\Formularios\
    |
    +-- Modulos\Dashboard\        das_module.aspx + Master
    +-- Modulos\Login\            LoginBitcode.aspx
    +-- modulos\tareas\           ~21 paginas Tareas
    +-- Modulos\Documentos\       NEWFRONT_doc_procesos (Flujos) + clientes
    +-- Modulos\Documental\       NEWFRONT_docu_formulario (Constructor)
    |                              + docu_viewform (Visor token)
    |                              + Controles\ctrFormCreator + crtCargaEncuesta...
    +-- Modulos\General\          Usuarios + Roles + Dependencias + Plantillas + Reglas
    +-- Modulos\Inventarios\      Catalogos
    +-- Modulos\Configuracion\    Controles + Modulos + SQL + Biblioteca + Errores + ParamXml
    +-- Modulos\PowerBI\
    +-- Modulos\reportes\
    +-- Modulos\cartera\          CRM
    +-- Modulos\comercial\
    +-- Modulos\utilidades\
```

---

*Si el menu del sistema cambia (se agrega/quita un modulo en `ADM_WEB`), regenerar este archivo
con un grep masivo de `topBar.Modulo = "..."` en todos los `.aspx.vb`.*
