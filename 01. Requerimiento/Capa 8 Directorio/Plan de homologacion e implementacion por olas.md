---
titulo: Plan de homologacion e implementacion por olas - Directorio Modular
capa: Capa 8 - Directorio
tipo: contrato-de-trabajo
fecha: 2026-09-03
alcance: construir el 2do motor de contactos (Directorio Modular) sin romper el motor actual
---

# Plan de homologacion e implementacion por olas (Directorio Modular)

> Contrato para la **sesion dedicada** que construira el segundo motor de contactos
> (la version avanzada, "Directorio Modular"). Trabaja en **rama/worktree aparte** y con
> sesion de Claude propia. Lee primero [[Directorio Modular avanzado - prototipo y modelo (Spec)]]
> (que hace) y [[Arquitectura - DirectorioSharedBase y vistas (Documental)]] +
> [[Tercero - entidad y campos dinamicos (Documental)]] (que existe hoy). Aplica ademas
> [[Reglas de codigo para el refactor]] (tope 2000 lineas, backend compartido).

---

## 1. Principios que NO se negocian

1. **Seleccionable, como una tercera variante.** El motor Modular se elige por tenant con
   el mismo mecanismo de hoy (`IDirectoryVariantService`, ADR-0088): la idea es extender
   `enum DirectoryVariant { Ligero, Especializado }` con `Modular` (o el nombre que se
   decida). Un tenant que no lo elija sigue viendo su Directorio de siempre, intacto.
2. **Quirurgico: no romper el motor actual.** Ligero/Especializado deben seguir
   funcionando EXACTAMENTE igual. Se toca el motor por EXTENSION, no modificando el
   comportamiento existente. Cualquier cambio en `DirectorioSharedBase`, `TerceroService`
   o `TerceroModal` que afecte a ambos motores es de alto riesgo y debe justificarse y
   probarse contra el motor actual.
3. **Misma tabla `Tercero`, con marca de motor.** Los registros de ambos motores viven en
   la MISMA tabla `Tercero`. Se agrega una **marca** clara del motor que creo/gestiona el
   registro (ej. campo `DirectoryEngine` / `OrigenMotor`: `Clasico` | `Modular`). No se
   crea una tabla de contactos paralela.
4. **Interoperabilidad total.** El **Gestor de contactos (000740)** y cualquier consumidor
   de `Tercero` deben **entender los contactos de ambos motores** sin romperse. Un contacto
   Modular tiene que ser legible por `ITerceroService`/`TerceroModal` clasico (aunque la
   ficha rica solo la pinte el motor Modular).
5. **MVP visual primero, luego olas.** El primer entregable es SOLO la parte visual (el
   front del Directorio Modular, fiel al prototipo), sin backend real. Despues se avanza
   por olas.
6. **Multi-tenant y DAL dual** (heredado): todo `TenantEntity` con filtro global, jsonb/
   nvarchar segun proveedor, pruebas en PG y SQL Server.

---

## 2. La marca de motor y la tabla compartida

- Nuevo campo en `Tercero` (o en una tabla 1:1 si se prefiere no ensanchar): p.ej.
  `DirectoryEngine` (enum `Clasico` | `Modular`), con default `Clasico` para todo lo
  existente. Se estampa al crear segun el motor activo.
- El motor Modular guarda ademas su estructura propia (categorias/secciones/campos y sus
  valores). Decision abierta a tomar en Ola 1: los **valores** siguen en
  `Tercero.FichasJson` (jsonb, reusa lo actual) o pasan a un EAV `tercero_valor`. Sea cual
  sea, la marca de motor permite que cada lector sepa como interpretar la ficha.
- **Migracion entre motores**: proceso idempotente que toma un Tercero `Clasico` y lo
  expresa como `Modular` (mapea fichas->secciones, perfiles->categorias) y viceversa. Es
  una OLA propia (ver 3.4), no bloquea el MVP. Debe ser reversible y auditable.

---

## 3. Olas de trabajo

### Ola 0 - MVP VISUAL (primer entregable)

- El front del Directorio Modular como **tercera variante seleccionable**, fiel al
  prototipo (`020. Soldarco/09.Directorio modular`): listado por categorias, pestanas,
  tarjetas de conteo, buscador, ficha por secciones en un scroll, modal de configuracion
  (secciones y campos, categorias), selector de rol/area (simulado o real).
- Puede leer datos existentes en modo lectura o usar datos de demostracion; NO exige el
  backend nuevo todavia.
- Criterio de aceptacion: se activa la variante Modular en un tenant de prueba y se
  navega el front sin tocar ni degradar Ligero/Especializado. Entrega visual.

### Ola 1 - HOMOLOGACION DEL SISTEMA DE GESTION DE COLUMNAS (campos/secciones)

> Es lo PRIMERO de backend, por decision del usuario: "primero la homologacion del sistema
> de gestion de columnas y luego lo grande".

- Unificar el modelo de definicion de campos/secciones. Puntos a resolver:
  - Mapear **seccion** <-> `TerceroFichaDefinition` (anadiendo `aplicaA` por naturaleza y
    `areas`/visibilidad) y **campo** <-> `TerceroFieldDefinition`.
  - Cubrir los **11 tipos** del prototipo vs `TerceroFieldType` actual (faltarian, entre
    otros, seleccion multiple, checkbox y **tabla** de filas dinamicas); decidir si se
    extiende `TerceroFieldType` o se mapea.
  - `filtrable` <-> `ShowInFilter`; clave autogenerada unica; campos de sistema no
    borrables; anchos 1/3-1/2-full.
  - Decidir la persistencia de valores (jsonb vs EAV) y dejarla estampada por la marca de
    motor.
- Regla dura: la homologacion **no debe cambiar** como el motor Clasico define/renderiza
  sus campos hoy. Si se comparte codigo de definicion, se hace por extension compatible.
- Criterio: el configurador de campos del motor Modular funciona; el del motor Clasico
  sigue identico; ambos persisten sin pisarse; matriz dual verde.

### Ola 2 - NUCLEO DE CONTACTOS MODULAR (lo grande)

- **Categorias componibles** (`tipos_directorio` + `tipo_seccion` + `tipo_area`) y
  **multi-membership** (`tercero_tipo`): un tercero en varias categorias, ficha = union de
  secciones.
- **Visibilidad por rol/area** en 3 niveles (categorias, secciones, acciones), integrada
  con la seguridad de la plataforma (no un permiso propio del modulo).
- **Naturaleza deducida** de lo que se llena; creacion dual organizacion+persona ya
  relacionadas; **relaciones con cargo por vinculo**; convertir persona en organizacion.
- **Listado**: tarjetas de conteo accionables (Por asignar, Nuevos), buscador global
  multi-categoria, avisos de datos repetidos (identificacion/correo/telefono, ignorando
  formato).

### Ola 3 - HOMOLOGACION FISCAL (RUT) y reglas finas

- Categoria Fiscal con bandera `homologa`: oculta la publica, RUT sobrescribe el
  directorio publico, IDE->NIT sin DV, naturaleza deducida del RUT; fuera de Fiscal la
  regla suave (llena lo vacio, no pisa, "Usar los del RUT").
- Campo **tabla** (Responsabilidades del RUT con autollenado desde catalogo).
- Reglas de integridad (seccion 8 de la spec).

### Ola 4 - MIGRACION ENTRE MOTORES + INTEROPERABILIDAD

- Proceso de migracion `Clasico <-> Modular` (idempotente, reversible, auditable).
- Verificar que el **Gestor de contactos (000740)** lee y opera contactos de ambos motores
  (bolsa, oportunidades, citas, llamada IA) sin romperse. Ver
  [[Conexion con Cargador de contactos (Documental)]].
- Verificar TaskWizard y demas consumidores de `Tercero`.

---

## 4. Riesgos y como mitigarlos

| Riesgo | Mitigacion |
| ------ | ---------- |
| Romper Ligero/Especializado al tocar `DirectorioSharedBase`/`TerceroService`/`TerceroModal` | Extender, no modificar; branch aparte; probar el motor Clasico en cada ola; el modal ya supera 2000 lineas: coordinar con el refactor de [[Reglas de codigo para el refactor]] |
| Divergencia de datos entre motores | Marca de motor + una sola tabla `Tercero`; contratos de lectura claros |
| Modelo de valores (jsonb vs EAV) mal elegido | Decidir en Ola 1 con la homologacion de columnas; dejar la marca de motor para poder interpretar |
| Permisos por rol/area mal integrados | Apoyarse en la seguridad de la plataforma, no inventar permisos del modulo |
| Gestor de contactos deja de entender contactos | Ola 4 dedicada a interoperabilidad + pruebas |

---

## 5. Modo de trabajo de la sesion dedicada

- **Rama/worktree aparte**; sesion de Claude propia; NO tocar el tronco hasta que una ola
  este verde y aprobada.
- Cada ola: entregable pequeno y demostrable; build + tests verdes; matriz dual si toca
  persistencia; el motor Clasico verificado sin regresion.
- Actualizar esta Capa 8 (spec + este plan + docs Documental) en el MISMO PR cuando el
  codigo mueva piezas. ADR nuevo si se decide algo estructural (ej. la marca de motor, el
  modelo de valores, extender `DirectoryVariant`).
- Fidelidad visual milimetrica al prototipo de `020. Soldarco/09.Directorio modular`
  (regla del proyecto). Considerar copiar el prototipo al vault (Prototipo/) como fuente
  de verdad versionada.

---

## 6. Checklist de arranque para la sesion dedicada

- [ ] Leer la spec [[Directorio Modular avanzado - prototipo y modelo (Spec)]] y abrir el
      prototipo (`index.html`) para ver el objetivo visual.
- [ ] Crear rama/worktree propio desde el tronco.
- [ ] Ola 0: entregar SOLO el front del motor Modular como variante seleccionable, sin
      romper Ligero/Especializado. Demostrar y parar para validacion del usuario.
- [ ] No avanzar a Ola 1 (homologacion de columnas) hasta que el MVP visual este aprobado.
- [ ] En cada ola: marca de motor respetada, tabla `Tercero` unica, Gestor de contactos
      sin regresion, matriz dual verde, docs de Capa 8 actualizados.
