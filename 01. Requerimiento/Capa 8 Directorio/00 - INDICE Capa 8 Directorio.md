---
tipo: indice-carpeta
capa: Capa 8 - Directorio
proposito: Documentacion del sistema de Directorio (Tercero, campos dinamicos, dos variantes de vista) y de su conexion con el Cargador de contactos. Incluye las reglas de codigo del refactor.
stack: .NET 10 / ASP.NET Core / EF Core 10 / Blazor Server
fecha: 2026-09-03
auditado: si (3 subagentes de exploracion sobre el codigo real)
---

# Capa 8 - Directorio

> Documenta **lo que ya esta construido** en el nuevo ECOREX (.NET 10 / Blazor),
> no un modulo legacy a reconstruir. El Directorio General (modulo 000232) gira
> alrededor de la entidad `Tercero` (empresa o persona) con **fichas y campos
> dinamicos configurables por tenant**, se presenta en **dos variantes de vista
> intercambiables** (Ligero | Especializado) sobre **una sola capa de backend**, y
> **comparte esa misma entidad y su modal** con el Cargador de contactos (000740).
>
> Esta capa es el ancla de contexto para el refactor en curso: separar mejor las
> responsabilidades y respetar el tope de tamano por archivo. Empezar por
> [[01 - Vision Directorio]].

## Documentos de la carpeta

| #  | Documento | Que cubre |
| -- | --------- | --------- |
| 00 | [[00 - INDICE Capa 8 Directorio]] | Este indice |
| 01 | [[01 - Vision Directorio]] | Que es el Directorio, las dos variantes, por que una sola capa de backend, contexto (ADR-0088) |
| 02 | [[Arquitectura - DirectorioSharedBase y vistas (Documental)]] | La capa compartida, las dos vistas .razor, el modal, la variante por tenant, la pagina de configuracion |
| 03 | [[Tercero - entidad y campos dinamicos (Documental)]] | La entidad Tercero, como se definen/almacenan/renderizan las fichas y campos dinamicos, relacion con el motor de formularios |
| 04 | [[Conexion con Cargador de contactos (Documental)]] | Como el Cargador (000740) reusa Tercero, ITerceroService y TerceroModal; la frontera de entidades; acciones (Bolsa, Oportunidades, Llamada IA) |
| 05 | [[Reglas de codigo para el refactor]] | El contrato del proyecto: tope de 2000 lineas por archivo, capa de backend compartida, como partir, checklist para recibir cambios sin perder contexto |

## Mapa de una linea (el sistema de un vistazo)

```
                         IDirectoryVariantService (tenant: "directorio.variante")
                                        |
             elige   Ligero  <----------+----------> Especializado
                        |                                   |
          DirectorioGeneral.razor              DirectorioEspecializado.razor
          (/directorio-general)                (/directorio-especializado)
                        \                                   /
                         \        @inherits (misma logica) /
                          v                               v
                             DirectorioSharedBase.cs
                             (estado + handlers + servicios)
                                        |
                                        | usa
        +-------------------------------+-------------------------------+
        v                v              v               v               v
  ITerceroService   ITerceroField   ITerceroFicha   ITerceroForm   IDataLookup / IAsesor
     (CRUD/KPIs)      (campos)        (fichas)      (form adjunto)      (lookups)
        |
        v
   Tercero (TenantEntity)  --- FichasJson (jsonb) = valores de los campos dinamicos
        ^
        | MISMA entidad + MISMO TerceroModal.razor (CrmWiring=true)
        |
  GestorContactos.razor (000740, /cargador-contactos)
  Prospecto -> (promocion) -> Tercero -> Bolsa / Oportunidad / Cita / Llamada IA
```

## Convenciones aplicadas

- **Solo ASCII** (regla del proyecto): sin tildes, sin enye, sin emojis.
- **Frontmatter YAML**: tipo, capa, stack, fecha.
- **Documental**: cada doc "(Documental)" describe el codigo .NET REAL con rutas
  `apps/backend/src/...` y numeros de linea, para que el mapa siga siendo verificable.
- Los conteos de linea son del estado a 2026-09-03 (v0.15.165); sirven de linea base
  para el refactor. Ver [[Reglas de codigo para el refactor]].

## Decisiones de arquitectura relacionadas (repo `docs/decisiones/`)

- **ADR-0088** - Variante del Directorio elegible por el tenant (Ligero | Especializado).
- **ADR-0029** - Motor de formulas para campos calculados (reusado por los campos `Calculated` del Tercero).
- **ADR-0082** - Acceso a secciones por Dependencia/Cargo (patron de permisos afin).
