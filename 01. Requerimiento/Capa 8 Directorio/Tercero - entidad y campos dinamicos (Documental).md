---
titulo: Tercero - entidad y campos dinamicos (Documental)
capa: Capa 8 - Directorio
tipo: documentacion-tecnica
fecha: 2026-09-03
version: v0.15.165
---

# Tercero: entidad y campos dinamicos (Documental)

Describe el codigo .NET REAL de la entidad `Tercero` y de su mecanismo de **campos
dinamicos por ficha**. Rutas relativas a `apps/backend/src/`. Contexto en
[[01 - Vision Directorio]] y [[Arquitectura - DirectorioSharedBase y vistas (Documental)]].

**Idea central**: los datos de un Tercero = unos pocos **campos base** (columnas) +
un grueso de **campos dinamicos** organizados en **fichas**, cuyos valores se guardan
como **un unico documento JSON por fila** (`Tercero.FichasJson`). No es EAV, no es una
tabla de atributos, no es el motor de formularios: es un modelo propio.

---

## 1. La entidad `Tercero`

Ruta: `Ecorex.Domain/Entities/Tercero.cs` (73 lineas).

- `public class Tercero : TenantEntity` -> multi-tenant (filtro global; hereda
  `Id`/`TenantId`/auditoria).
- **Campos base (columnas)**:
  - `Nombre`, `Tipo` (`TerceroTipo`: Empresa | Persona), `Perfiles`
    (`TerceroPerfil` [Flags]: cliente/proveedor/empleado/sospechoso...), `Estado`
    (`TerceroEstado`), `Ciudad`, `ImagenUrl`.
  - Identificacion: `IdTipo`, `IdValor`.
  - Vendedor: `Vendedor` (texto legado), `VendedorAsesorId`/`VendedorAsesor` (FK Asesor).
  - Empresa: `Sector`. Persona: `Cargo`, `Email`, `Telefono`.
  - Relacion self: `EmpresaId`/`Empresa` (una persona es contacto de una empresa).
  - `Contactos` (`ICollection<TerceroContacto>`): contactos embebidos.
  - `BolsaColumnaId`/`BolsaColumna`: pertenencia a la Bolsa kanban del Cargador
    (000740); null = solo vive en el Directorio. Ver
    [[Conexion con Cargador de contactos (Documental)]].
- **Campos dinamicos**: `FichasJson` (`string?`) -- el documento JSON con TODOS los
  valores de fichas/campos. Estructura logica:
  `Dictionary<string /*fichaKey*/, Dictionary<string /*fieldKey*/, string /*valor*/>>`.
  Un valor multi (AllowMultiple o repetido) se guarda como arreglo JSON dentro de la
  misma celda de texto.

---

## 2. Definicion de fichas y campos (configurables por tenant)

Dos entidades de definicion, ambas `TenantEntity`. Calcadas del patron
`PipelineFieldDefinition` de CUBOT.travels; a su vez, `TerceroFieldDefinition` es el
patron de referencia que copian otros modulos (Inventario, Tareas 000066 -- ver el
comentario en `EcorexDbContext.cs` "Calcado de TerceroFieldDefinition").

### 2.1 `TerceroFichaDefinition` (la ficha / pildora)

Ruta: `Ecorex.Domain/Entities/TerceroFichaDefinition.cs` (43 lineas). Fuente unica de
verdad de las fichas (antes estaban hardcodeadas en 3 sitios; ahora son data-driven).

| Propiedad | Rol |
| --------- | --- |
| `FichaKey` | clave estable de la ficha (slug) |
| `Title` | titulo visible de la pildora |
| `Description` | ayuda |
| `Color` | color de la pildora |
| `Perfil` | lista CSV de perfiles que la hacen visible; null = siempre (base) |
| `SortOrder` | orden |
| `IsHidden` | ocultar sin borrar |
| `IsSystem` | sembrada por defecto (no se debe borrar a la ligera) |

### 2.2 `TerceroFieldDefinition` (el campo)

Ruta: `Ecorex.Domain/Entities/TerceroFieldDefinition.cs` (66 lineas).

| Propiedad | Rol |
| --------- | --- |
| `FichaKey` | a que ficha pertenece |
| `FieldKey` | clave estable del campo (slug, ej. `tipo_de_persona`) |
| `Label` | etiqueta visible |
| `FieldType` | tipo de control (ver 2.3) |
| `Options` | opciones de Select (separadas por salto de linea); tambien config serializada para Lookup/DirectoryLookup |
| `Column` | ancho en la rejilla de 3 (1/2/3) |
| `SortOrder` | orden dentro de la ficha |
| `Description` | ayuda (tambien usada por MCP/IA) |
| `AllowMultiple` | permite varios valores |
| `Formula` | expresion (solo tipo `Calculated`) |
| `ShowInFilter` | expone el campo como filtro del listado del Directorio |
| `RepeatWithFieldKey` | el campo se repite N veces segun otro campo |
| `IsSystem` | sembrado por defecto |

### 2.3 Tipos de campo (`TerceroFieldType`)

Ruta: `Ecorex.Domain/Enums/TerceroFieldType.cs` (44 lineas). Se persiste como texto.

`Text`, `Number`, `Currency`, `TextArea`, `Select`, `Date`, `Phone`, `Separator`
(titulo divisor), `Calculated` (formula; ADR-0029), `Lookup` (trae una fila de un
**Contenedor de datos**), `DirectoryLookup` (referencia a **otro Tercero**; reusa
`FormSourceKind.Tercero`).

---

## 3. Almacenamiento (EF Core, DAL dual)

Ruta: `Ecorex.Infrastructure/Persistence/EcorexDbContext.cs`.

- Tipo de columna JSON segun proveedor:
  `var jsonColumnType = isNpgsql ? "jsonb" : "nvarchar(max)";`
- `Tercero.FichasJson` mapeado a ese tipo:
  `b.Property(x => x.FichasJson).HasColumnType(jsonColumnType);`
  -> **jsonb en PostgreSQL, nvarchar(max) en SQL Server**.
- `TerceroFieldDefinition`: indice unico `(TenantId, FichaKey, FieldKey)`.
- `TerceroFichaDefinition`: indice unico `(TenantId, FichaKey)`.

Serializacion/validacion en `Ecorex.Application/Directorio/TerceroService.cs`:
`ValidateFichas`, `ParseFichas`, `Normalize` (al guardar). El JSON se valida contra las
definiciones vigentes del tenant antes de persistir.

---

## 4. Servicios de fichas y campos

- **`TerceroFichaService.cs`** (203 lineas): `EnsureDefaultsAsync` siembra 6 fichas
  por defecto (base, fiscal, comercial, cliente, proveedor, empleado); CRUD;
  `NormalizePerfil`; `Slugify` (genera `FichaKey`).
- **`TerceroFieldService.cs`** (437 lineas): `Defaults`/`BuildDefaultFields` (campos
  sembrados por ficha); `EnsureDefaultsAsync`; CRUD; `MoveFieldToFichaAsync`;
  `ValidateFormulaAsync`; `ComputeCalculatedAsync`; clave unica por tenant/ficha;
  `Slugify` genera `FieldKey`.
- **`TerceroFormService.cs`** (83 lineas): formularios adjuntos ofrecidos en el modal
  (ver seccion 6).

---

## 5. Configuracion (UI) y renderizado

### 5.1 Donde se configuran los campos: DENTRO del Directorio

El boton **"Configurar campos"** vive en la propia vista del Directorio
(`DirectorioGeneral.razor` / `DirectorioEspecializado.razor` -> `OpenConfigAsync`),
y abre el modal "Configurar campos de las fichas". La **logica** la orquesta
`DirectorioSharedBase` sobre los tres servicios (`FichaSvc`, `FieldSvc`, `FormsSvc`):
crear/renombrar/color/perfil/ocultar/reordenar fichas; CRUD de campos, mover de ficha,
reordenar, validar formula.

> NO se configuran en `/configuracion-entidad` (esa pagina es de la Entidad =
> agencias/areas). Ver la aclaracion en
> [[Arquitectura - DirectorioSharedBase y vistas (Documental)]] seccion 5.1.

### 5.2 Como se renderizan en el modal

En `TerceroModal.razor` (inyecta `FieldSvc`/`FichaSvc`/`FormsSvc`):

- Las fichas visibles se filtran por perfil del tercero
  (`!IsHidden && (Perfil vacio || perfil del tercero)`).
- Por cada ficha, `@foreach` de sus campos con `switch (f.FieldType)`:
  - `Separator` -> titulo divisor.
  - `RepeatWithFieldKey` -> N inputs.
  - `Select` -> desplegable con `Options`.
  - `TextArea` -> area de texto.
  - `Lookup` -> componente `<DataLookupField>` (Contenedor de datos).
  - `Calculated` -> input readonly con prefijo `fx` (valor de la formula).
  - default -> `<input type=...>` segun `InputType(...)`; ancho por `WidthClass` (rejilla).
- Estado en memoria: `_fichaValues` (dict ficha->campo->valor), `GetFicha`/`SetFicha`;
  carga en `_fichas`/`_fichaFields`.
- Guardado: `SaveAsync` empaqueta `FichasJson`.
- Campos calculados en vivo: `RecalcCalculated` aplana los valores de TODAS las fichas
  y usa `FormulaCalculator.EvaluateAll`.

### 5.3 Como se filtran en el listado

`TerceroService.cs` lee las `FieldKey` con `ShowInFilter = true` (una sola consulta) y
`ExtractFilterables` deserializa `FichasJson` para exponer SOLO esas claves como un
diccionario por fila en `TerceroListItemDto`. Las claves de filtro se calculan en
`DirectorioSharedBase`.

---

## 6. Relacion con el motor de formularios (Capa 4)

**El Tercero NO reusa `DynamicFormRenderer` / `FormDefinition` para sus campos de
ficha**: tiene su propio mecanismo (fichas + `TerceroFieldDefinition` + `FichasJson`).
Los puntos de contacto con la Capa 4 son:

- **Formularios adjuntos (opcionales)**: via `TerceroFormLink`
  (`Ecorex.Domain/Entities/TerceroFormLink.cs`) el tenant decide ofrecer ciertas
  `FormDefinition` dentro del modal del tercero. Ahi SI se usa `<DynamicFormRenderer>`,
  anclado por `Reference = "TERCERO:{id}"` (`ITerceroFormService.ReferenceFor`). Las
  respuestas viven como `FormResponse` aparte, NO en `FichasJson`.
- **`DirectoryLookup`** (tipo de campo) reusa el motor de lookups de formularios
  (`FormSourceKind.Tercero`, `TerceroLookupSource.cs`).
- **`Lookup`** reusa el Contenedor de datos (`Ecorex.Application.DataLookups`), no los
  formularios.
- **Campos `Calculated`** reusan el motor de formulas compartido
  (`Ecorex.Application.Formulas`: `FormulaEngine`, `FormulaCalculator`,
  `CalculatedField`), documentado en `docs/decisiones/ADR-0029-*`.

Resumen: la idea "Capa 4 EAV->jsonb" aplica al Tercero de forma **conceptual** (campos
configurables cuyos valores acaban en un jsonb), pero esta **implementada con su
propio conjunto de entidades**, no con `FormDefinition`. El motor de formularios entra
solo como funcionalidad adjunta y como proveedor de lookups/formulas reutilizados.

---

## 7. Conteo de lineas (linea base)

| Archivo | Lineas |
| ------- | ------ |
| `Domain/Entities/Tercero.cs` | 73 |
| `Domain/Entities/TerceroFieldDefinition.cs` | 66 |
| `Domain/Entities/TerceroFichaDefinition.cs` | 43 |
| `Domain/Enums/TerceroFieldType.cs` | 44 |
| `Application/Directorio/TerceroService.cs` | 710 |
| `Application/Directorio/TerceroFieldService.cs` | 437 |
| `Application/Directorio/TerceroFichaService.cs` | 203 |
| `Application/Directorio/TerceroFormService.cs` | 83 |
| `Application/Directorio/TerceroFieldDtos.cs` | 61 |
| `SuperAdmin/Components/Shared/TerceroModal.razor` | 2112 |

`EcorexDbContext.cs` (secciones relevantes): tipo JSON, mapeo `Tercero`, indices de
`TerceroFieldDefinition` y `TerceroFichaDefinition`, y config de `TerceroFormLink`.
