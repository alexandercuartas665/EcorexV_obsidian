---
titulo: Conexion Directorio <-> Cargador de contactos (Documental)
capa: Capa 8 - Directorio
tipo: documentacion-tecnica
fecha: 2026-09-03
version: v0.15.165
---

# Conexion Directorio <-> Cargador de contactos (Documental)

Como el **Cargador de contactos** reusa la entidad `Tercero`, el `ITerceroService` y
el `TerceroModal` del Directorio. Rutas relativas a `apps/backend/src/`. Contexto en
[[01 - Vision Directorio]] y [[Tercero - entidad y campos dinamicos (Documental)]].

---

## 0. Aclaracion: hay DOS modulos con nombre parecido

| Pagina | Ruta | Modulo | Trabaja sobre | Usa Tercero? |
| ------ | ---- | ------ | ------------- | ------------ |
| **`GestorContactos.razor`** | `/cargador-contactos` | **000740** | Prospectos, Bolsa, Oportunidades, Citas | **SI** (reusa Tercero + TerceroModal) |
| `CargadorContactos.razor` | `/cargador-csv` | 000873 | Importa CSV a la entidad `Lead` (pipeline) | NO (usa `Lead`) |

El que se conecta con el Directorio es **`GestorContactos` (000740)**. La pagina CSV
(`CargadorContactos`, 543 lineas) inserta filas como `Lead` en el pipeline y no toca
`Tercero`; queda fuera de esta capa.

---

## 1. La pagina del Cargador (000740)

Ruta: `Ecorex.SuperAdmin/Components/Pages/GestorContactos.razor` (1763 lineas).
**[> 1500 lineas: candidato a vigilar en el refactor]**.

- `@page "/cargador-contactos"`, modulo 000740.
- Cuatro pestanas (enum `Tab`): **Prospectos scrapeados**, **Bolsa de contactos**
  (kanban de terceros), **Oportunidades** (embudo) y **Agenda** de citas.
- Inyecta: `IGestorContactosService GestorSvc`, `ITerceroService TerceroSvc`,
  `IOportunidadEstadoService EstadosSvc`.

---

## 2. Servicios que respaldan el Cargador

| Archivo (`apps/backend/src/...`) | Clase | Rol | Lineas |
| -------------------------------- | ----- | --- | ------ |
| `Ecorex.Application/Gestor/GestorContactosService.cs` | `GestorContactosService` | Nucleo del 000740: prospectos, bolsa, oportunidades, citas, filtros | 1017 |
| `Ecorex.Application/Gestor/IGestorContactosService.cs` | contrato | — | 89 |
| `Ecorex.Application/Directorio/TerceroService.cs` | `TerceroService` | Servicio del Directorio **reusado** | 710 |
| `Ecorex.Application/Crm/OportunidadEstadoService.cs` | `OportunidadEstadoService` | Etapas configurables del embudo | 191 |
| `Ecorex.Application/Gestor/ContactWorkflowDispatcher.cs` | `ContactWorkflowDispatcher` | Ejecuta workflows (WhatsApp/Email/Llamada IA) sobre segmentos de Terceros | 505 |

Registro DI en `Ecorex.Application/DependencyInjection.cs`: `IContactLoaderService`,
`IOportunidadEstadoService`, `ITerceroService`, `IGestorContactosService`,
`IContactWorkflowDispatcher`. Componente de UI compartido: `TerceroModal.razor` (2112).

---

## 3. La conexion con Tercero (lo central)

**El Cargador reusa la MISMA entidad `Tercero`, el MISMO `ITerceroService` y el MISMO
`TerceroModal` que el Directorio General.** Un contacto del Cargador ES un cliente
(Tercero); ambos modulos se alimentan de los mismos registros -- es el mismo registro
visto desde dos modulos.

Puntos de llamada a `TerceroSvc` desde `GestorContactos.razor`:

- `TerceroSvc.ListAsync(new TerceroListFilter())` -- lee terceros del Directorio.
- `TerceroSvc.AssignToEmpresaAsync(terceroId, empresaId)` -- asigna persona a empresa
  (mismo patron que el Directorio).
- `TerceroSvc.DeleteAsync(terceroId)` -- baja logica (soft-delete).

Reuso del modal compartido:

- `<TerceroModal @ref="_terceroModal" OnChanged="ReloadAllAsync" CrmWiring="true"
  OnIncorporarProspecto="IncorporarProspectoAsync" />` -- el MISMO modal del
  Directorio, en modo CRM (`CrmWiring=true`).
- `_terceroModal.OpenFromProspectoAsync(...)` -- abre el modal desde un prospecto.
- En `TerceroModal.razor`, `CrmWiring=true` activa el panel de oportunidades del
  tercero (llamadas a `GestorSvc.ListOportunidadesByTerceroAsync`).

Escritura de Terceros desde el servicio (`GestorContactosService.PromoverProspectoAsync`):
crea registros `Tercero` directamente (`_db.Terceros.Add(...)`), tanto empresa como
persona encadenada a la empresa; marca `prospecto.TerceroId` para vincular. Los nuevos
terceros nacen con `Estado = TerceroEstado.Prospecto` y `BolsaColumnaId` = primera
columna de la Bolsa.

El vinculo a nivel de entidad esta en la propia `Tercero`: el campo `BolsaColumnaId`
(un Tercero "esta en la Bolsa" cuando tiene columna asignada; si es null, solo vive en
el Directorio).

---

## 4. Acciones especiales ligadas al Tercero

- **Oportunidades**: `Oportunidad` (`Ecorex.Domain/Entities/Oportunidad.cs`) cuelga de
  un Tercero por FK `TerceroId` (cascade). Se crean/leen con
  `GestorSvc.CreateOportunidadAsync(terceroId, ...)` /
  `ListOportunidadesByTerceroAsync(terceroId)`. En la UI nacen desde la pildora
  "Oportunidad" del modal.
- **Bolsa (kanban)**: arrastrar tarjetas mueve el Tercero de columna
  (`GestorSvc.MoverTerceroAsync(tid, columnaId)`); el estado se guarda en
  `Tercero.BolsaColumnaId`.
- **Agenda / Citas**: la cita se liga opcionalmente a un Tercero
  (`SaveCitaRequest(...)`); abrir una cita abre la ficha del cliente.
- **"Llamada IA" / workflows**: es el paso `ContactWorkflowStepType.Llamada` (voz
  Retell) del `ContactWorkflowDispatcher`, que resuelve un SEGMENTO de `Tercero` desde
  un `TerceroFiltro` y registra el run con `TerceroId`. Se disena en
  `Components/Shared/ContactWorkflowDesigner.razor`. Se enlaza al Tercero por
  filtro/segmento, no por tercero individual.

---

## 5. Frontera: entidades propias del Cargador vs. reuso de Tercero

**Reusa Tercero (no crea una entidad de contacto paralela):** `Tercero` es la unica
entidad de "contacto/cliente", compartida con el Directorio.

**Entidades propias del 000740 (giran alrededor de Tercero):**

- `ProspectoScrapeado.cs` (66 lineas): dato crudo scrapeado, aun NO es Tercero; tiene
  `TerceroId` nullable que se llena al **promover** (`PromoverProspectoAsync`).
  Frontera clara: prospecto = pre-tercero; al promover se materializa un `Tercero`.
- `Oportunidad.cs` (48): negocio que cuelga de un Tercero (FK).
- `BolsaColumna.cs` (24): columnas del kanban; el Tercero apunta a ellas por
  `BolsaColumnaId`. La "Bolsa" no es una entidad de contacto: es solo la columna; los
  ocupantes son Terceros.
- `OportunidadEstado.cs`: etapas configurables del embudo.
- Cita/Agenda y filtros dinamicos: DTOs propios del servicio.

En una linea:

```
ProspectoScrapeado  --(PromoverProspectoAsync)-->  Tercero (Estado=Prospecto, BolsaColumnaId)
                                                      |
                              +-----------------------+------------------------+
                              v            v           v            v          v
                         Bolsa(col)   Oportunidad    Cita      Llamada IA   Directorio
                       (kanban)      (FK Tercero)  (agenda)  (segmento)   (mismo registro)
```

El Cargador **no duplica** la entidad de contacto: la comparte con el Directorio via
`Tercero`, `ITerceroService` y `TerceroModal`.

---

## 6. Implicacion para el refactor

- La regla "una sola capa de backend" ya se cumple para la entidad de contacto:
  Directorio y Cargador comparten `Tercero` + `ITerceroService` + `TerceroModal`.
  **No introducir una entidad de contacto paralela ni un segundo servicio de CRUD de
  Tercero.**
- `GestorContactos.razor` (1763) y `TerceroModal.razor` (2112) son los dos archivos
  grandes de esta frontera; el modal ya supera el tope. El plan de division esta en
  [[Reglas de codigo para el refactor]].
