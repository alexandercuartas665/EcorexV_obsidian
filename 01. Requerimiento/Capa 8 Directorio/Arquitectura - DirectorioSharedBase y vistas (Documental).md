---
titulo: Arquitectura del Directorio - DirectorioSharedBase y vistas (Documental)
capa: Capa 8 - Directorio
tipo: documentacion-tecnica
fecha: 2026-09-03
version: v0.15.165
---

# Arquitectura del Directorio: capa compartida + dos fronts (Documental)

Describe el codigo .NET REAL. Todas las rutas son relativas a
`apps/backend/src/`. Los numeros de linea son del estado a v0.15.165 y pueden
correrse con el refactor; el rol de cada archivo es lo estable. Contexto en
[[01 - Vision Directorio]]; decision en `docs/decisiones/ADR-0088-*`.

---

## 1. Vista de conjunto

```
IDirectoryVariantService.GetAsync()  ->  Ligero | Especializado   (tenant config)
        |                                          |
        v                                          v
DirectorioGeneral.razor                 DirectorioEspecializado.razor
  @page /directorio-general               @page /directorio-especializado
  ViewVariant = Ligero                    ViewVariant = Especializado
  <TerceroModal @ref>                     <TerceroModal Especializado="true" @ref>
        \                                          /
         \____________ @inherits __________________/
                          v
              DirectorioSharedBase.cs      (abstract ComponentBase)
              estado + handlers + [Inject] de todos los servicios
                          |
       usa: ITerceroService, ITerceroFieldService, ITerceroFichaService,
            ITerceroFormService, ICurrentPermissions, IDataLookupService,
            IAsesorService, IDirectoryVariantService, IJSRuntime, NavigationManager
```

Regla base (ADR-0088): **la logica vive UNA vez** en `DirectorioSharedBase`; las dos
`.razor` son solo el "front" (markup + CSS scoped) y heredan la base con `@inherits`.

---

## 2. `DirectorioSharedBase.cs` - la capa compartida

Ruta: `Ecorex.SuperAdmin/Components/Pages/DirectorioSharedBase.cs` (~1088 lineas).

- Declaracion: `public abstract class DirectorioSharedBase : ComponentBase`.
- **Servicios inyectados** (`[Inject]`): `ITerceroService TerceroSvc`,
  `ITerceroFieldService FieldSvc`, `ITerceroFichaService FichaSvc`,
  `ITerceroFormService FormsSvc`, `ICurrentPermissions Perms`, `IJSRuntime JS`,
  `IDataLookupService LookupSvc`, `IAsesorService AsesorSvc`,
  `IDirectoryVariantService DirVariant`, `NavigationManager Nav`.
- **Estado principal**: `_loading/_reloading/_busy`; `_kpis` (`TerceroKpisDto`);
  `_allRows` (`IReadOnlyList<TerceroListItemDto>`); tabs `_tipo`/`_naturaleza`;
  `_search`; arbol empresa->contactos (`_expanded`/`_kids`); permisos
  (`_canCrearEmpresa/Cliente/Sospechoso`, `_canEdit`, `_canDelete`); `_terceroModal`
  (`@ref` al `TerceroModal`); `_fieldFilters`; paginacion (`_pageSize`, `_page`);
  estado de importacion Excel; y todo el estado del configurador de fichas/campos.
- **El mecanismo de variante** (clave del diseno):
  - `protected abstract DirectoryVariant ViewVariant { get; }` -- cada vista lo fija.
  - En `OnInitializedAsync`: lee `DirVariant.GetAsync()`; si la variante configurada
    del tenant **no coincide** con `ViewVariant`, hace
    `Nav.NavigateTo(<otra ruta>, replace: true)` y retorna. Asi cada vista atiende
    solo su variante y reenvia a la otra si el tenant eligio la contraria.
- **Handlers (agrupados por responsabilidad)**:
  - Ciclo de vida y carga: `OnInitializedAsync`, `ReloadAsync`, `SetTipo`,
    `OnSearch`, `ToggleExpandAsync`.
  - Alta/edicion via modal: `OpenCreateAsync`, `OnModalChangedAsync`.
  - Relacion empresa/persona: `OpenAssign`, `AssignToAsync`; borrado
    `DeleteAsync`/`DeleteContactoAsync`/`UnassignContactoAsync` (soft-delete).
  - Importacion Excel: `OpenImportAsync`, `OnImportFileAsync`,
    `ComputeDuplicatesAsync`, `RunImportAsync`.
  - Configurador de fichas y campos: crear/renombrar/color/perfil/ocultar/reordenar
    fichas (via `FichaSvc`); CRUD de campos, mover de ficha, reordenar, validar
    formula (via `FieldSvc`); formularios adjuntos (via `FormsSvc`).
  - Helpers de presentacion: `TipoTags`, `EstadoInfo`, `AvatarColor`, `Initials`.

> Para el refactor: este archivo (1088) esta por debajo del tope de 2000 pero es el
> segundo mas grande del subsistema. El configurador de fichas/campos es un
> candidato natural a extraerse a su propio componente/mixin. Ver
> [[Reglas de codigo para el refactor]].

---

## 3. Las dos vistas `.razor`

Ambas comparten estructura casi identica; difieren en textos y CSS scoped.

### 3.1 `DirectorioGeneral.razor` (~968 lineas) - variante Ligero

- `@page "/directorio-general"`
- `@attribute [Authorize(Policy = "TenantMember")]`
- `@rendermode InteractiveServerRenderMode(prerender: false)`
- `@inherits DirectorioSharedBase`
- `@code`: `protected override DirectoryVariant ViewVariant => DirectoryVariant.Ligero;`
- Instancia el modal: `<TerceroModal @ref="_terceroModal" OnChanged="OnModalChangedAsync" />`
- Boton "Configurar campos" -> `OpenConfigAsync`; el modal de configuracion de
  fichas/campos vive en el markup y se respalda en la base.
- CSS: `DirectorioGeneral.razor.css`.

### 3.2 `DirectorioEspecializado.razor` (~968 lineas) - variante Especializado

- `@page "/directorio-especializado"`, mismos atributos.
- `@code`: `ViewVariant => DirectoryVariant.Especializado;`
- Instancia el modal con la bandera:
  `<TerceroModal Especializado="true" @ref="_terceroModal" OnChanged="OnModalChangedAsync" />`
- Layout/clase propia (`dg-especializado`) y CSS: `DirectorioEspecializado.razor.css`.

> Las dos vistas son hoy casi iguales (la Especializada cambia titulo/eyebrow y pasa
> `Especializado=true` al modal). Estan asi a proposito: son el punto de divergencia
> visual por tenant, y la duplicacion de PRESENTACION es aceptada (la de LOGICA no,
> por eso vive en la base). Un cambio de LAYOUT comun hay que reflejarlo en ambos
> markups.

---

## 4. `TerceroModal.razor` - editor compartido de Tercero

Ruta: `Ecorex.SuperAdmin/Components/Shared/TerceroModal.razor` (~2112 lineas).
CSS: `TerceroModal.razor.css`.

Editor unico de crear/editar Tercero, con pestanas **Datos / Relaciones / Contacto
Cliente / Actividades** y un submodal de contacto. Lo usan el Directorio (ambas
variantes), el Cargador de contactos y el asistente de tareas.

- **Parametros `[Parameter]`**:
  - `EventCallback OnChanged` -- pide al host refrescar su lista.
  - `EventCallback<Guid> OnCreated` -- al crear, avisa el id (TaskWizard no se queda en edicion).
  - `bool CrmWiring` -- habilita el cableado CRM (notas, panel de oportunidades) para el Cargador 000740.
  - `EventCallback<Guid> OnIncorporarProspecto` -- incorporar un prospecto scrapeado.
  - `bool Especializado` -- variante Especializada (ADR-0088); solo detalles visuales.
- **Metodos publicos de apertura** (se invocan por `@ref` desde el host):
  `OpenCreate(TerceroPerfil, string?)`, `OpenCreateContactAsync(Guid, Guid?)`,
  `OpenFromProspectoAsync(...)`, `OpenEditAsync(Guid, Guid?)`,
  `OpenContacto(Guid, TerceroContactoDto?)`.
- **Consumidores** (grep `<TerceroModal`):
  - `DirectorioGeneral.razor` -- `OnChanged`.
  - `DirectorioEspecializado.razor` -- `Especializado="true"` + `OnChanged`.
  - `GestorContactos.razor` -- `CrmWiring="true"`, `OnChanged`, `OnIncorporarProspecto` (000740).
  - `TaskWizard.razor` -- `OnCreated` (alta rapida desde el asistente de tareas).
- El render y el guardado de los **campos dinamicos (fichas)** viven aqui; se
  detallan en [[Tercero - entidad y campos dinamicos (Documental)]].

> ATENCION refactor: es el UNICO archivo del subsistema que **ya supera el tope de
> 2000 lineas** (2112). Es el primer candidato a partirse (por pestanas y por el
> editor de fichas). Ver plan en [[Reglas de codigo para el refactor]].

---

## 5. `IDirectoryVariantService` - la variante por tenant

Ruta: `Ecorex.Application/Directorio/IDirectoryVariantService.cs` (~85 lineas;
enum + interfaz + implementacion en un archivo).

- `enum DirectoryVariant { Ligero, Especializado }` (Ligero = default).
- `interface IDirectoryVariantService`: `Task<DirectoryVariant> GetAsync(ct)` y
  `Task SetAsync(DirectoryVariant, ct)`.
- `sealed class DirectoryVariantService`: inyecta `IApplicationDbContext _db` e
  `ITenantContext _tenant`.
  - Constante `ConfigKey = "directorio.variante"`; valores `"ligero"`/`"especializado"`.
  - `GetAsync`: lee `_db.TenantConfigurations` (`AsNoTracking`) por `ConfigKey`;
    `"especializado"` -> `Especializado`, cualquier otra cosa -> `Ligero` (default
    seguro). Tenant-scoped por el filtro global de `TenantConfiguration`.
  - `SetAsync`: exige tenant activo; upsert de la fila `TenantConfiguration`
    (crea con `TenantId`/`ConfigKey`/`ConfigValue`, o actualiza el valor).

### 5.1 Donde se elige: `/configuracion-entidad`

Ruta: `Ecorex.SuperAdmin/Components/Pages/ConfiguracionEntidad.razor`
(`@page "/configuracion-entidad"`). Inyecta `IDirectoryVariantService DirVariant`.
Un `<select>` con opciones Ligero/Especializado, enlazado a `_dirVariant`; el
handler `OnDirVariantChangedAsync` llama `DirVariant.SetAsync(...)` (guarda de
inmediato) y el valor inicial se carga en `OnInitializedAsync` con
`DirVariant.GetAsync()`.

> Ojo de nomenclatura: esta pagina es de la **Entidad** (agencias/areas/sucursales)
> y su boton "Campos personalizados" es para la Entidad, NO para el Tercero. Lo unico
> del Directorio que vive aqui es el selector de variante. Los campos del **Tercero**
> se configuran DENTRO del propio Directorio (boton "Configurar campos"). Ver
> [[Tercero - entidad y campos dinamicos (Documental)]].

---

## 6. Servicios de aplicacion del Directorio

Todos en `Ecorex.Application/Directorio/` (interfaces y DTOs aparte).

| Archivo | Rol | Lineas |
| ------- | --- | ------ |
| `TerceroService.cs` / `ITerceroService.cs` | CRUD + listado + KPIs + contactos embebidos + notas + asignacion empresa/persona + dedup para importacion | 710 / 76 |
| `TerceroFieldService.cs` / `ITerceroFieldService.cs` | CRUD de definiciones de campos dinamicos por ficha (crear/editar/reordenar/mover/validar formula/calcular) | 437 / 49 |
| `TerceroFichaService.cs` / `ITerceroFichaService.cs` | CRUD de fichas configurables + sembrado de fichas por defecto | 203 / 39 |
| `TerceroFormService.cs` / `ITerceroFormService.cs` | Links de formularios ofrecidos en el modal (`Reference = "TERCERO:{id}"`) | 83 / 36 |
| `TerceroImportXlsx.cs` | Parseo/validacion de la plantilla Excel de importacion | 199 |
| `TerceroTemplateXlsx.cs` | Generacion de la plantilla Excel con listas desplegables | 168 |
| `TerceroDtos.cs` | DTOs de listado/detalle/KPIs/filtros | 164 |
| `TerceroFieldDtos.cs` | DTOs/requests de campos dinamicos | 61 |
| `TerceroResult.cs` | Resultado tipado (ok/errores) | 26 |
| `DirectorioSubPermisos.cs` | Sub-permisos nombrados del Directorio | 47 |

Ningun servicio de la capa Application supera 1500 lineas (el mayor es
`TerceroService.cs`, 710). El unico archivo del subsistema por encima de 2000 es la
UI `TerceroModal.razor`.

---

## 7. Tabla resumen - archivos mas grandes del subsistema (linea base del refactor)

| # | Archivo (`apps/backend/src/...`) | Rol | Lineas |
| - | -------------------------------- | --- | ------ |
| 1 | `Ecorex.SuperAdmin/Components/Shared/TerceroModal.razor` | Editor crear/editar Tercero (SUPERA 2000) | 2112 |
| 2 | `Ecorex.SuperAdmin/Components/Pages/DirectorioSharedBase.cs` | Logica base compartida de las dos vistas | 1088 |
| 3 | `Ecorex.SuperAdmin/Components/Pages/DirectorioGeneral.razor` | Vista Ligero | 968 |
| 3 | `Ecorex.SuperAdmin/Components/Pages/DirectorioEspecializado.razor` | Vista Especializado | 968 |
| 5 | `Ecorex.Application/Directorio/TerceroService.cs` | Servicio CRUD/listado/KPIs | 710 |
| 6 | `Ecorex.Application/Directorio/TerceroFieldService.cs` | CRUD campos dinamicos | 437 |
| 7 | `Ecorex.Application/Directorio/TerceroFichaService.cs` | CRUD fichas | 203 |

El plan de division esta en [[Reglas de codigo para el refactor]].
