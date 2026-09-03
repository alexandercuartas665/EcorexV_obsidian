---
titulo: Reglas de codigo para el refactor del Directorio
capa: Capa 8 - Directorio
tipo: contrato-de-trabajo
fecha: 2026-09-03
version_base: v0.15.165
---

# Reglas de codigo para el refactor del Directorio

> Contrato del proyecto largo: separar mejor las responsabilidades del Directorio y
> del Cargador **sin perder contexto** entre sesiones. La meta funcional NO cambia
> (dos variantes de vista sobre una sola capa de backend); lo que cambia es el
> **tamano y la reparticion** de los archivos, para que cada cambio sea legible,
> revisable y facil de cargar en contexto. Estas reglas complementan las nueve reglas
> inviolables de `CLAUDE.md`, no las reemplazan.

---

## 1. Tope de tamano por archivo (la regla que motiva el refactor)

- **Tope DURO: 2000 lineas por archivo/clase.** Ningun archivo nuevo o modificado
  debe superarlo. Si un cambio deja un archivo por encima de 2000, hay que partirlo en
  el mismo PR.
- **Umbral de ALARMA: 1500 lineas.** Al cruzarlo, el archivo entra en "lista de
  vigilancia": el siguiente cambio que lo toque debe empezar por extraer una
  responsabilidad, no por seguir agregando.
- **Objetivo sano: < 1000 lineas.** No es obligatorio, pero es la referencia de un
  archivo comodo de leer.
- El tope aplica a `.cs` y a `.razor` (contando markup + `@code`). Un `.razor.css`
  scoped no cuenta para el tope del `.razor`.

### 1.1 Linea base a 2026-09-03 (v0.15.165)

| Archivo | Lineas | Estado |
| ------- | ------ | ------ |
| `Components/Shared/TerceroModal.razor` | 2112 | **VIOLA el tope (>2000): partir primero** |
| `Components/Pages/GestorContactos.razor` | 1763 | En alarma (>1500): vigilar |
| `Components/Pages/DirectorioSharedBase.cs` | 1088 | Holgado, pero es el nucleo compartido |
| `Components/Pages/DirectorioGeneral.razor` | 968 | Ok |
| `Components/Pages/DirectorioEspecializado.razor` | 968 | Ok |
| `Application/Gestor/GestorContactosService.cs` | 1017 | Holgado |
| `Application/Directorio/TerceroService.cs` | 710 | Ok |

---

## 2. Una sola capa de backend (regla estructural, ya vigente)

- **La logica de negocio vive en `Ecorex.Application` (servicios) y, para el Directorio,
  en `DirectorioSharedBase`.** Las vistas `.razor` son SOLO front: markup + CSS +
  binding. Prohibido meter reglas de negocio (queries, calculos, validaciones de
  dominio) dentro de una vista.
- **Los dos directorios (Ligero/Especializado) comparten el backend.** No duplicar
  logica: si algo es comun, va a la base compartida o a un servicio. Lo unico que
  puede divergir por variante es la PRESENTACION (markup/CSS de cada `.razor`).
- **La entidad de contacto es unica: `Tercero`.** Directorio y Cargador la comparten
  via `ITerceroService` y `TerceroModal`. Prohibido crear una entidad de contacto
  paralela o un segundo servicio de CRUD de Tercero.
- **Un servicio = una responsabilidad.** Ya esta bien partido: `TerceroService`
  (CRUD/KPIs), `TerceroFieldService` (campos), `TerceroFichaService` (fichas),
  `TerceroFormService` (formularios adjuntos). Al crecer, se parte por responsabilidad,
  no por conveniencia.

---

## 3. Como partir (patrones permitidos)

Preferencia, de mayor a menor:

1. **Subcomponentes Blazor** (un `.razor` grande -> varios `.razor` hijos con
   `[Parameter]`/`EventCallback`). Es la via principal para `TerceroModal`.
2. **Partials de clase** (`partial class` en varios archivos) cuando una clase grande
   tiene grupos claros de metodos (ej. `DirectorioSharedBase` -> un partial para el
   configurador de fichas/campos).
3. **Servicios de aplicacion nuevos** cuando emerge una responsabilidad de negocio
   propia (ej. importacion, KPIs) que hoy vive dentro de otro servicio.
4. **Componentes compartidos reutilizables** cuando dos vistas repiten markup no trivial
   (ej. un editor de campos de ficha que sirva al modal y al configurador).

Lo que NO cuenta como "partir": mover codigo a `#region`, meter helpers estaticos en un
archivo suelto sin cambiar la responsabilidad, o duplicar para "aliviar" un archivo.

### 3.1 Plan concreto para `TerceroModal.razor` (2112 -> partir)

Es el unico archivo que ya viola el tope. Division sugerida por PESTANA + editor de
fichas (nombres tentativos):

- `TerceroModal.razor` (contenedor: estado de apertura, parametros, orquestacion,
  guardado) -- deberia bajar bien por debajo de 1000.
- `TerceroDatosTab.razor` (pestana Datos: campos base + render de fichas).
- `TerceroRelacionesTab.razor` (pestana Relaciones).
- `TerceroContactoClienteTab.razor` (pestana Contacto Cliente + `CrmWiring`).
- `TerceroActividadesTab.razor` (pestana Actividades / formularios adjuntos).
- `FichaFieldsEditor.razor` (render de los campos dinamicos de una ficha; el `switch
  (FieldType)`), reutilizable por la pestana Datos y por el configurador.

Cada pestana recibe el estado por `[Parameter]` y avisa cambios por `EventCallback`; el
contenedor sigue siendo el dueno del guardado (`SaveAsync`/`FichasJson`). No cambiar el
comportamiento observable ni los metodos publicos de apertura (`OpenCreate`,
`OpenEditAsync`, `OpenFromProspectoAsync`, ...): otros modulos los invocan por `@ref`.

### 3.2 Vigilancia de `GestorContactos.razor` (1763) y `DirectorioSharedBase.cs` (1088)

- `GestorContactos.razor`: al siguiente cambio grande, extraer cada pestana
  (Prospectos / Bolsa / Oportunidades / Agenda) a su subcomponente.
- `DirectorioSharedBase.cs`: candidato a `partial class` separando el bloque
  "configurador de fichas/campos" del bloque "listado/CRUD/import".

---

## 4. Invariantes que el refactor NO debe romper

- **Multi-tenant real**: `Tercero` y las definiciones siguen siendo `TenantEntity` con
  filtro global. Cero queries tenant-scoped sin filtro.
- **DAL dual**: `FichasJson` sigue mapeado como `jsonb` (PG) / `nvarchar(max)` (SQL
  Server). Toda prueba de integracion corre en ambos motores.
- **Variante por tenant (ADR-0088)**: el mecanismo `ViewVariant` + redireccion en
  `OnInitializedAsync` no cambia; el modal sigue recibiendo `Especializado`.
- **API publica del modal**: los metodos `Open*` invocados por `@ref` desde
  Directorio/Cargador/TaskWizard no cambian de firma sin actualizar a los tres
  consumidores en el mismo PR.
- **Soft-delete + auditoria + concurrencia optimista** como en el resto del sistema.
- **Solo ASCII** en archivos nuevos.

---

## 5. Checklist para RECIBIR un cambio del refactor (revision)

- [ ] Ningun archivo tocado supera 2000 lineas; si alguno cruzo 1500, el PR explica
      por que no se partio aun (o lo parte).
- [ ] El cambio NO metio logica de negocio en un `.razor` ni duplico backend entre
      Ligero y Especializado.
- [ ] No aparecio una entidad de contacto paralela ni un segundo CRUD de Tercero.
- [ ] Los metodos publicos del `TerceroModal` conservan firma, o se actualizaron los
      tres consumidores.
- [ ] `dotnet build` verde; tests de la parte tocada verdes; matriz dual si toca
      persistencia.
- [ ] Un solo tema por PR (una pestana, un servicio, una extraccion): PRs pequenos y
      legibles, para no perder contexto entre sesiones.
- [ ] Si el cambio altera una decision de arquitectura, hay ADR nuevo en
      `docs/decisiones/` y se refleja aqui (Capa 8) + `PROGRESO.md`.

---

## 6. Como mantener el contexto entre sesiones (meta del proyecto)

- Cada sesion arranca leyendo esta Capa 8 (empezando por [[00 - INDICE Capa 8 Directorio]]).
- Los documentos "(Documental)" se actualizan cuando el refactor mueve piezas: si
  `TerceroModal` se parte, la tabla de archivos y los nombres se corrigen aqui MISMO,
  en el mismo PR. La documentacion desactualizada es peor que ninguna.
- La tabla de "linea base" (seccion 1.1) se re-mide tras cada ola grande del refactor,
  para saber cuanto se avanzo hacia el objetivo < 1000 y que sigue en alarma.
- PRs pequenos y de un solo tema: es la mejor defensa contra la perdida de contexto.
