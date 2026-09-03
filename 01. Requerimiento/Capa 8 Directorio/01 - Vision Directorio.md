---
tipo: vision-modulo
capa: Capa 8 - Directorio
modulo_codigo: "000232"
stack: .NET 10 / ASP.NET Core / EF Core 10 / Blazor Server
entidad_nucleo: Tercero (TenantEntity)
persistencia_campos: Tercero.FichasJson jsonb (Postgres) / nvarchar(max) (SQL Server)
estado: documentacion del sistema construido (v0.15.165)
---

# Vision - Directorio (Capa 8)

> Vision del sistema de **Directorio** del nuevo ECOREX sobre .NET 10 / Blazor.
> El stack, la arquitectura y el aislamiento multi-tenant se definen en
> [[Visión y entorno]]; el aspecto y la navegacion en el Prototipo Final. Aqui se
> describe QUE es el Directorio, POR QUE tiene dos variantes de vista sobre una
> sola capa de backend, y COMO se relaciona con el resto del sistema. El detalle
> tecnico esta en los tres documentos "(Documental)" de esta carpeta.

## 1. Que es y por que existe

El **Directorio General** (modulo 000232) es el CRM base del sistema: el catalogo
de **Terceros** de cada tenant. Un Tercero es una **empresa** o una **persona
natural** con uno o varios **perfiles de negocio** (cliente, proveedor, empleado,
sospechoso). No es una simple agenda: es el registro maestro de contacto que otros
modulos reusan (el Cargador de contactos, las tareas, los formularios de tercero).

Su promesa: **el administrador del tenant modela sus propios campos** (que datos
guarda de una empresa fiscalmente, que datos comerciales, que datos de cliente)
**sin escribir codigo**, agrupados en **fichas** (pildoras tematicas), y el sistema
los renderiza en el modal de crear/editar, los guarda, los deja filtrar en el
listado y los expone a reportes y reglas.

## 2. Las dos variantes de vista (el corazon del diseno)

A unos clientes les gusta como se ve el Directorio y a otros no. En lugar de un
unico diseno, el tenant **elige** entre dos presentaciones del MISMO modulo:

- **Ligero** (por defecto): la vista `DirectorioGeneral` (`/directorio-general`).
- **Especializado**: la vista `DirectorioEspecializado` (`/directorio-especializado`),
  con su propio markup/CSS, pensada para divergir libremente en el futuro.

La eleccion es **tenant-wide** y self-service (se guarda en `TenantConfiguration`
bajo la clave `directorio.variante`). Un unico punto de menu (`/directorio-general`)
lee la variante y, si el tenant configuro la contraria, redirige a la otra ruta.

**La regla de oro (ADR-0088): una sola CAPA DE LOGICA compartida + dos FRONTS
delgados.** Toda la logica (estado, servicios, handlers) vive una sola vez en
`DirectorioSharedBase`; las dos vistas `.razor` solo aportan markup + CSS y heredan
esa base. Asi no se duplica el backend: un fix comun se hace en un solo lugar, y
cada front evoluciona por su lado. El modal de crear/editar Tercero (`TerceroModal`)
tambien es unico y compartido; solo recibe una bandera `Especializado` que cambia
detalles visuales. Ver el detalle en
[[Arquitectura - DirectorioSharedBase y vistas (Documental)]].

> Esta es exactamente la idea que guia el refactor en curso: **los dos directorios
> funcionan con la misma capa de backend**. Lo que sigue es partir mejor las piezas
> grandes (empezando por el modal) sin romper esa regla. Ver
> [[Reglas de codigo para el refactor]].

## 3. Campos dinamicos: fichas configurables por tenant

Los datos de un Tercero no son columnas fijas. Ademas de unos pocos campos base
(nombre, tipo, perfiles, estado, ciudad, identificacion), el grueso son **campos
dinamicos** organizados en **fichas**:

- Una **ficha** es una categoria/pildora (fiscal, comercial, cliente, riesgo,
  proveedor, empleado...). Es configurable: el tenant crea/renombra/oculta fichas y
  decide en que perfiles aparecen.
- Cada ficha tiene **campos** definidos por el tenant (texto, numero, moneda,
  seleccion, fecha, telefono, separador, calculado por formula, lookup a un
  Contenedor de datos, o lookup a otro tercero).
- Los **valores** de todos esos campos de una fila se guardan como **un documento
  JSON** en la propia fila del Tercero (`FichasJson`, columna `jsonb` en Postgres /
  `nvarchar(max)` en SQL Server). No es EAV ni una tabla de atributos.

Este mecanismo es propio del Tercero (no reusa el motor de formularios para las
fichas), aunque comparte con la Capa 4 el motor de formulas (campos calculados,
ADR-0029), el motor de lookups y la posibilidad de **adjuntar** formularios al
tercero. El detalle esta en
[[Tercero - entidad y campos dinamicos (Documental)]].

## 4. Como se conecta con el resto del sistema

- **Cargador de contactos (000740, `/cargador-contactos`)**: es el CRM comercial
  (prospectos scrapeados, bolsa kanban, oportunidades, agenda). **Reusa la misma
  entidad `Tercero`, el mismo `ITerceroService` y el mismo `TerceroModal`** que el
  Directorio. Un contacto del Cargador ES un Tercero del Directorio: son el mismo
  registro visto desde dos modulos. La frontera: un **Prospecto scrapeado** es un
  pre-tercero que, al **promoverse**, materializa un `Tercero`; sobre ese Tercero se
  montan bolsa, oportunidades, citas y la accion "Llamada IA". Ver
  [[Conexion con Cargador de contactos (Documental)]].
- **Asistente de tareas (TaskWizard)**: abre el mismo `TerceroModal` para dar de
  alta un tercero rapido al crear una actividad.
- **Formularios (Capa 4)**: un tenant puede **adjuntar** definiciones de formulario
  al modal del tercero (`TerceroFormLink`); ahi si se usa el `DynamicFormRenderer`.
- **Reportes (Capa 6/Motor de BI)**: el Tercero y sus campos son fuente reportable
  (fuente nativa del catalogo tenant-safe).

## 5. Multi-tenant y seguridad (heredado)

`Tercero` y todas las entidades de definicion (`TerceroFichaDefinition`,
`TerceroFieldDefinition`) y de configuracion (`TenantConfiguration`) son
`TenantEntity`: filtro global por tenant, imposible cruzar datos entre tenants por
construccion. Los permisos del Directorio se resuelven con `ICurrentPermissions`
(sub-permisos nombrados en `DirectorioSubPermisos`): crear empresa/cliente/sospechoso,
editar, eliminar. El borrado es **soft-delete**. Ver reglas transversales en
[[Seguridad y Autenticacion multi-tenant]] y las nueve reglas inviolables del
proyecto en `CLAUDE.md`.

## 6. Estado y siguiente paso

A 2026-09-03 (v0.15.165) el sistema esta en produccion y funcionando: dos variantes,
fichas/campos configurables, importacion Excel, y la integracion con el Cargador. El
siguiente trabajo es el **refactor de tamano y responsabilidades**: el modal
`TerceroModal.razor` (2112 lineas) ya supera el tope de 2000 lineas y debe partirse;
otros archivos estan holgados pero conviene fijar el contrato antes de crecer. Ese
contrato es [[Reglas de codigo para el refactor]].
