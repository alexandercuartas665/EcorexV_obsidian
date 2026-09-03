---
titulo: Directorio Modular avanzado - prototipo y modelo (Spec)
capa: Capa 8 - Directorio
tipo: spec-prototipo
fecha: 2026-09-03
prototipo: "C:/Users/acuartas/OneDrive - Bitcode IT Services S.A.S/Bitcode/13. Proyectos/020. Soldarco/09.Directorio modular"
estado: prototipo front (en memoria, sin backend); es la version AVANZADA seleccionable del Directorio
---

# Directorio Modular avanzado - prototipo y modelo (Spec)

> Este documento describe el **segundo motor de contactos** de ECOREX: la version
> AVANZADA del Directorio, llamada "Directorio Modular". Es **seleccionable** por el
> tenant, igual que hoy se elige Ligero | Especializado (ADR-0088): seria una tercera
> variante. Su fuente es el prototipo front en `020. Soldarco/09.Directorio modular`
> (`index.html` + `js/core/*` + `js/ui/*` + `README.md`). Aqui se documenta QUE hace y
> COMO esta modelado; el plan para construirlo (por olas, quirurgico, tabla compartida)
> esta en [[Plan de homologacion e implementacion por olas]].
>
> Comparar siempre contra el motor actual descrito en
> [[Tercero - entidad y campos dinamicos (Documental)]] y
> [[Arquitectura - DirectorioSharedBase y vistas (Documental)]].

---

## 1. La idea central: la ficha se ARMA por secciones que la categoria escoge

En el motor actual, la ficha del Tercero son "fichas" (pildoras) por perfil con campos.
En el motor Modular la ficha **no es fija ni por perfil**: se compone de **secciones**
que cada **categoria** decide incluir, y todo se filtra por el **rol/area** del usuario.

Dos conceptos nuevos:

- **Seccion**: un grupo de campos de la ficha (publica, comercial, tributaria,
  condiciones de cliente...). Cada seccion declara: a que **naturaleza** aplica
  (organizacion / persona / ambas), que **areas** la ven, y que **campos** pide.
- **Categoria** (en el codigo del prototipo, "tipo de directorio"): arma su ficha
  marcando 1+ secciones y declara que areas la tienen autorizada. Es la **pestana**
  sobre la que se para el usuario en el listado; al crear un tercero ahi, la ficha pide
  exactamente las secciones de esa categoria.

Diferencias clave frente al motor actual:

| Aspecto | Motor actual (Ligero/Especializado) | Motor Modular (avanzado) |
| ------- | ----------------------------------- | ------------------------ |
| Agrupador de campos | Ficha (pildora) por `Perfil` | Seccion + Categoria componible |
| Pertenencia | Perfiles [Flags] en el Tercero | Un tercero en 1+ **categorias** (multi-membership) |
| Visibilidad | Por perfil del tercero | Por **rol/area** del usuario (3 niveles) |
| Naturaleza | Campo `Tipo` (Empresa/Persona) elegido | **Deducida** de lo que se llena |
| Ficha desde | Un modal unico | Se abre segun la **categoria** desde la que se crea |
| Valores | `FichasJson` (jsonb, dict) | EAV `tercero_valor` (o jsonb; ver homologacion) |

---

## 2. Configuracion ligada al rol/area del usuario (3 niveles)

Todo lo que un usuario ve depende de su **rol** (parametrizacion del modulo aplicada a
quien lo abre), en tres niveles:

| Nivel | Que controla |
| ----- | ------------ |
| **Categorias** | Que pestanas ve, en cuales puede crear, y **que registros existen para el**: los que solo viven en una categoria no autorizada NO aparecen en listado ni busqueda |
| **Secciones** | Que secciones de la ficha se le piden; a las ocultas la ficha avisa "N seccion(es) oculta(s) para tu area" sin mostrar contenido |
| **Acciones** | El atajo "+" para crear categorias desde el listado solo lo ve Administracion |

Administracion (`admin`) ve todas las categorias y secciones, siempre. En el prototipo
las areas son **Administracion, Comercial, Contabilidad, Logistica**, simuladas con el
selector "Ver como"; en el producto saldran del **rol del usuario autenticado**
(seguridad de la plataforma), no de este modulo.

---

## 3. Las cinco categorias predefinidas (+ personalizables)

| Categoria | Secciones que arma | Areas autorizadas |
| --------- | ------------------ | ----------------- |
| **Publico** | Seccion publica | todas |
| **Comercial** | Publica, comercial, condiciones de cliente | Comercial |
| **Proveedores** | Publica, condiciones de proveedor | Logistica, Contabilidad |
| **Laboral** | Publica, datos laborales | Administracion |
| **Fiscal** | Seccion tributaria (RUT) | Contabilidad |

- **Publico** es la base del catalogo, va marcado por defecto en toda ficha nueva y **no
  se elimina**. La seccion publica se puede activar/desactivar dentro de cada categoria.
- Un tercero puede pertenecer a **varias categorias** a la vez (cliente y proveedor); su
  ficha muestra la **union** de las secciones de todas. Las pestanas se ordenan solas
  alfabeticamente; Publico primero.
- La categoria de un tercero **no se elige en su ficha**: la define desde donde se crea,
  o el boton "+ <categoria>" del listado.

### 3.1 La regla de la categoria Fiscal (homologacion desde el RUT)

Bandera `homologa: 'publica'` en la categoria Fiscal:

- Mientras un tercero esta en Fiscal, su ficha **oculta la seccion publica** (se llena
  sola desde el RUT); un aviso lo explica.
- Al guardar, los datos del RUT **homologan y sobrescriben** el directorio publico
  (aqui el RUT SI pisa). En particular **IDE pasa a ser el NIT sin digito de
  verificacion** (`890325264-2` -> `890325264`).
- Como no hay campos publicos donde escribir el nombre, la **naturaleza se deduce del
  propio RUT** (razon social/nombre comercial -> organizacion; nombres+apellidos ->
  persona).
- Fuera de Fiscal la regla es mas suave: lo vacio se llena solo (destella verde) y lo que
  ya tiene valor **no se pisa** (aviso comparativo + boton "Usar los del RUT").

Mapeo RUT -> directorio publico: IDE<-NIT/numero de identificacion;
Nombre empresa<-razon social/nombre comercial; Contacto<-primer nombre+otros
nombres+apellidos; Correo<-correo del RUT; Telefono<-Telefono 1; Ciudad/Pais<-del RUT.

---

## 4. Secciones (modelo del prototipo)

Cada seccion declara (ver `js/core/schema.js`): `id`, `nombre`, `icono`, `color`,
`sistema`, `protegida`, `aplicaA` (['empresa','contacto']), `areas`, `descripcion` y
`campos`. Las seis secciones de sistema:

| Seccion (id) | aplicaA | areas | Nota |
| ------------ | ------- | ----- | ---- |
| `publica` | empresa, contacto | todas | protegida, no se elimina; identificacion minima |
| `comercial` | empresa, contacto | comercial | perfilamiento comercial (17 campos) |
| `tributaria` | empresa, contacto | contabilidad | datos del RUT por casillas; incluye el campo `tabla` de Responsabilidades |
| `ficha_cliente` (Condiciones de cliente) | empresa, contacto | comercial, contabilidad | cupo, plazo, forma de pago, retencion |
| `ficha_proveedor` (Condiciones de proveedor) | empresa, contacto | logistica, contabilidad | tipo, plazo entrega, banco/cuenta |
| `ficha_empleado` (Datos laborales) | **contacto** | admin | solo personas |

- **A que naturaleza aplica**: una seccion se pide solo en organizaciones, solo en
  personas o en ambas (por eso "Datos laborales" no sale en una organizacion).
- **Que areas la ven**: admin todas; la publica es visible para todas y protegida; las
  demas solo las areas marcadas.
- En la ficha, todas las secciones se pintan en un solo scroll y las pestanas de arriba
  se marcan solas segun donde va el usuario.

---

## 5. Campos (el sistema de "gestion de columnas" a homologar)

Se configuran en "Configurar directorio > Secciones y campos". Cada campo:

| Propiedad | Que hace |
| --------- | -------- |
| `etiqueta` | nombre visible en la ficha |
| `tipo` | uno de **once**: texto, texto largo (textarea), numero, moneda, fecha, lista (select), seleccion multiple, casilla (checkbox), correo (email), telefono, y **tabla** de filas dinamicas |
| `ancho` | pequena (1/3), media (1/2) o completa -> compone la reticula de la seccion |
| `opciones` | solo en lista y seleccion multiple: una por linea |
| `obligatorio` (`requeridoEn`) | se exige al guardar (puede ser por naturaleza) |
| `descripcion` | ayuda bajo el campo |
| `orden` | flechas suben/bajan |
| `filtrable` | expone el campo para acotar/buscar (analogo a `ShowInFilter` actual) |
| `soloLectura` | de solo lectura (codigo, fecha_creacion, usuario) |

- **La clave tecnica (`clave`) se genera sola** desde la etiqueta (sin acentos,
  minusculas, guiones bajos) y se garantiza unica con sufijo. Es la que usa la BD.
- **Campos de sistema** (`sistema: true`): se reordenan, cambian de ancho y renombran,
  pero **no se eliminan** (sostienen identificacion/busqueda/homologacion).
- El tipo **tabla** crece por filas y tiene `columnas` (ej. Responsabilidades del RUT:
  al escoger el codigo se autocompleta la descripcion desde el catalogo - `autollena`).

> Este es el punto que el usuario quiere **homologar PRIMERO**: unificar el sistema de
> definicion de campos/secciones (el actual `TerceroFieldDefinition`/`TerceroFichaDefinition`
> vs este modelo de secciones+campos con 11 tipos, `aplicaA`, `filtrable`, `tabla`,
> claves autogeneradas). Ver [[Plan de homologacion e implementacion por olas]] Ola 1.

---

## 6. Organizaciones y personas

- Ambas son **elementos principales** del listado (la persona tiene fila propia).
- **No se pregunta la naturaleza**: la decide lo que se llena en la seccion publica.
  - Nombre/telefono de empresa -> organizacion.
  - Contacto/telefono de contacto/cargo -> persona.
  - Los dos bloques -> **dos registros, ya relacionados** (contacto y correo a la
    persona; identificacion y demas secciones a la organizacion; ubicacion a ambos).
  - Nada -> el guardado avisa que falta un nombre.
- Auto-llenado: **Codigo** consecutivo `TER-000001`, **Fecha de creacion**, **Usuario**
  (los tres solo lectura).
- **Estado** activo/inactivo (chip en el listado; inactivar no borra ni rompe vinculos).
- **Validacion**: cada pestana lleva contador rojo de obligatorios faltantes; al guardar
  con pendientes, salta a la seccion del primero y lo enfoca.
- **La naturaleza no cambia** tras crear (para el caso real existe "Convertir en
  organizacion").

### 6.1 Relaciones (acordeon)

- Persona <-> organizacion; el **cargo pertenece al vinculo, no a la persona** (Marcela
  es Gerente en una y Asesora en otra). En el store: `vinculos: [{empresaId, cargo}]`.
- Una persona puede estar ligada a varias organizaciones. Eliminar una organizacion no
  borra a sus personas: solo pierden el vinculo.
- Autocompletado paginado (no trae todo al navegador).

### 6.2 Convertir persona en organizacion

En la ficha de una persona: crea una organizacion con sus datos y deja a la persona
relacionada como su contacto; ambas siguen existiendo por separado.

---

## 7. Listado, tarjetas de conteo y buscador

- **Tarjetas de conteo** (5, sobre la categoria activa; pulsarlas acota, volver a pulsar
  suelta): <categoria>, Organizaciones, Personas, **Por asignar** (solo en Publico: cola
  por clasificar), **Nuevos (30 dias)**. Sustituyen al viejo panel de filtros por campo.
- **Buscador unico**: busca a la vez por nombre de empresa, contacto, razon social,
  nombre comercial, sigla, telefonos, correos, IDE, NIT, numero de identificacion y
  codigo. Ignora acentos y mayusculas. Con texto, **recorre todas las categorias** (para
  ver si ya existe antes de recrear), respetando el rol. Resultados: solo el elemento
  principal, contraidos; chips dicen en que categorias esta.
- **Avisos de datos repetidos**: al escribir identificacion/correo/telefono se comprueba
  si otro tercero ya lo usa (comparacion ignorando formato); aviso informativo con enlace
  a esa ficha, no bloquea el guardado.

---

## 8. Reglas de integridad

| Regla | Motivo |
| ----- | ------ |
| Solo se relacionan **personas con organizaciones** | El vinculo es "trabaja en"/"representa a" |
| Solo se convierten **personas** en organizacion | Una organizacion ya lo es |
| La **seccion publica** no se elimina | Identificacion minima (si se puede desactivar por categoria) |
| Los **campos de sistema** no se eliminan | Sostienen identificacion/busqueda/homologacion |
| **Publico** no se elimina como categoria | Base del catalogo |
| Debe existir **al menos una categoria** | Sin ninguna no hay donde clasificar |
| La **naturaleza** no cambia tras crear | Evitar datos huerfanos; existe "Convertir en organizacion" |

---

## 9. Modelo de datos del prototipo y mapeo a persistencia

Entidades en memoria (`store.js`): un tercero tiene `tipo` ('empresa'|'contacto'),
`valores` (dict clave->valor, EAV), `tiposDirectorio` (array de categorias,
multi-membership), `vinculos` (array {empresaId, cargo}), `estado`, y `codigo`
`TER-000006`.

Mapeo previsto a BD (del README del prototipo, Fase 2 SQLite):

```
tipos_directorio (id, nombre, icono, color, sistema, protegido, descripcion)  -- categorias
tipo_seccion     (tipo_id, seccion_id, orden)   -- que secciones arma cada categoria
tipo_area        (tipo_id, area_id)             -- que areas la tienen autorizada
secciones        (id, nombre, icono, color, orden, sistema, protegida)
seccion_tipo_ter (seccion_id, naturaleza)       -- aplicaA: organizacion / persona
seccion_area     (seccion_id, area_id)          -- visibilidad por area
campos           (id, seccion_id, clave, etiqueta, tipo, ancho, orden, ...)
campo_opcion     (campo_id, valor, etiqueta)
terceros         (id, naturaleza, estado)
tercero_empresa  (contacto_id, empresa_id, cargo)  -- cargo por vinculo
tercero_tipo     (tercero_id, tipo_id)          -- multi-membership
tercero_valor    (tercero_id, campo_id, valor)  -- EAV; el tipo tabla guarda JSON
```

### 9.1 Como se homologa contra el modelo ECOREX actual

Este es el corazon del trabajo (detalle en el plan). Correspondencias:

| Prototipo | ECOREX actual | Nota de homologacion |
| --------- | ------------- | -------------------- |
| `terceros` | `Tercero` (TenantEntity) | **misma tabla**; se agrega marca de motor (ver plan) |
| naturaleza empresa/contacto | `Tercero.Tipo` (Empresa/Persona) | 1:1 |
| `tercero_empresa` (cargo por vinculo) | hoy `EmpresaId` self-FK + `TerceroContacto` | el vinculo con cargo es mas rico; homologar |
| `secciones` | `TerceroFichaDefinition` | seccion anade `aplicaA` y `areas` |
| `campos` | `TerceroFieldDefinition` | 11 tipos vs `TerceroFieldType`; `filtrable`~`ShowInFilter`; `tabla` nuevo |
| `tercero_valor` (EAV) | `Tercero.FichasJson` (jsonb) | decidir: seguir en jsonb o EAV (ver plan) |
| `tipos_directorio` + `tercero_tipo` | (no existe) `Perfil` [Flags] | **concepto NUEVO**: categorias componibles + multi-membership |
| `tipo_area`/`seccion_area` + rol | permisos por `DirectorioSubPermisos` | visibilidad por area/rol es mas fina; homologar con seguridad de plataforma |
| categoria Fiscal `homologa` | (no existe) | regla de homologacion RUT->publico, nueva |

Conclusiones para el plan:
- Lo mas cercano y por ende lo primero a **homologar**: el sistema de definicion de
  **secciones+campos** (columnas) contra `TerceroFichaDefinition`/`TerceroFieldDefinition`.
- Lo **grande y nuevo**: categorias componibles + multi-membership + visibilidad por
  rol/area + naturaleza deducida + homologacion fiscal.
- **Los valores siguen en la misma tabla `Tercero`** (jsonb o EAV a decidir), con una
  **marca del motor** que creo cada registro, para que ambos motores y el Gestor de
  contactos (000740) entiendan los datos.

---

## 10. Estructura del prototipo (para el que lo implemente)

```
index.html                 Shell: sidebar, barra superior, contenedores de modales
assets/css/app.css         Diseno sobre Bootstrap 5
js/core/catalogos.js       Listas (ciudades, CIIU, responsabilidades DIAN...)
js/core/schema.js          Secciones de sistema (6) y categorias predefinidas (5)
js/core/permisos.js        Areas (4), roles y visibilidad de categorias/secciones
js/core/seed.js            Datos demo
js/core/store.js           UNICA frontera con los datos (825 lineas)
js/ui/campo-render.js      Dibuja y lee un campo segun su tipo (11 tipos)
js/ui/lista.js             Listado, categorias, tarjetas de conteo, buscador
js/ui/ficha-modal.js       Ficha del tercero por secciones (1054 lineas)
js/ui/config-modal.js      Parametrizacion: categorias + secciones y campos
js/app.js                  Arranque y rol activo
```

Pendientes conocidos del prototipo (no bloquean): catalogos parciales (RUT ~20/80, CIIU,
ciudades); "Configurar directorio" sin restriccion por rol; Importar/Exportar Excel
decorativo; paginacion fija.

> La **fidelidad visual** al prototipo es la fuente de verdad del MVP (Ola 0). El plan de
> construccion, las olas, la regla de no romper el motor actual, la tabla compartida con
> marca de motor y la migracion estan en [[Plan de homologacion e implementacion por olas]].
