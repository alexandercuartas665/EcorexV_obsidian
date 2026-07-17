---
tipo: nota-desarrollo
capa: Capa 2 - Formularios dinamicos
proposito: Bitacora de necesidades de DESARROLLO del motor de formularios, detectadas al DISENAR formularios reales de clientes. La sesion de diseno anota aqui; la sesion de desarrollo las implementa.
estado: activo
---

# Notas para el programador - ECOREX Formularios

> Convencion: la **sesion de diseno** (que arma formularios reales sobre el motor) escribe aqui cada
> vez que el diseno pide algo que el motor todavia no cubre. La **sesion de desarrollo** toma estos
> apuntes, los implementa y marca el estado. Solo ASCII. Referencias de codigo con ruta relativa al
> repo `apps/backend/src/...`.
>
> Estados: `[ ] pendiente`  `[~] en curso`  `[x] hecho`  `[-] descartado`.
> Prioridad: **P1** bloquea el formulario / **P2** mejora fuerte / **P3** deseable.

---

## 2026-07-17 - Diseno de CONTACTO CLIENTE (SOLDARCO, FRM-00005)

Contexto: primer formulario real que se disena sobre el motor nuevo. Origen legacy
`M700_GEN.dbo.ENCUESTAS_MOV` (SOLDARCO). Cabezotes ya migrados a `form_definitions`; las preguntas se
construyen a partir del mapa de diseno. El diseno visual y el mapa de campos completo estan en el
artefacto de diseno entregado al usuario (maqueta + decisiones + mapa legacy->nuevo).

Decisiones de diseno ya cerradas con el usuario (NO requieren desarrollo, son configuracion):
- Consecutivo: el formulario es **transaccional** (`identity_mode = Sequence`); el numero lo asigna el
  motor de consecutivos y se muestra read-only. Ya soportado (el renderer pinta el chip de
  `FormResponse.RecordNumber`).
- Cliente (nombre/codigo/NIT): **texto libre** por ahora (sin lookup a Terceros).
- Personas de contacto: **GridDetail** (filas en el mismo registro).
- Valor: visible/obligatorio **solo si** "Concreto venta = Si".
- Fecha default = Hoy (`DefaultDynamic`); Asesor comercial default = usuario actual.

### Necesidades de desarrollo detectadas

**D1 [x] P2 - Control de HORA (Time).** HECHO 2026-07-17 (opcion A). Se agrego `Time` y `DateTime`
al enum (input `type=time`/`datetime-local`), validacion `ValidateTime`, entrada en la paleta y en
la lista de tipos elegibles. Verificado en Chrome: un campo Time se pinta como selector de hora.
El enum `Ecorex.Domain/Enums/FormControlType.cs` tiene `Date` pero NO tiene `Time` ni `DateTime`. El
formato viejo separa Fecha y Hora (ambas venian como tipo "Fecha" en el legacy, pero la 2a es una hora).
- Opcion A (recomendada): agregar `FormControlType.Time` (input `type=time`) + su rama en
  `Components/Shared/Forms/DynamicFormRenderer.razor` (hoy `Date` esta en el `case` ~linea 557) y en el
  `FormDesigner.razor` (paleta de controles).
- Opcion B: un solo `DateTime`. Menos fiel al formato viejo.
- Mientras no exista: el campo Hora queda FUERA del formulario (no se pone texto-mascara como parche,
  por decision del usuario).

**D2 [x] P2 - Consecutivo visible en BORRADOR.** HECHO 2026-07-17. Decision del usuario:
PREVISUALIZAR (no reservar, para no dejar huecos en la secuencia si se descarta un borrador). El
numero se sigue emitiendo al confirmar; en el borrador de un formulario transaccional el renderer
muestra el chip "N.o por asignar". Verificado en Chrome.
`FormResponse.RecordNumber` se asigna al **confirmar** el registro (es null en Draft; ver
`Domain/Entities/FormResponse.cs` y la ola F3). El renderer ya muestra el chip cuando existe
(`DynamicFormRenderer.razor` ~linea 54-56), pero durante el llenado de un registro nuevo aparece vacio.
El formato viejo muestra el consecutivo desde el inicio.
- Necesidad: **reservar/mostrar el numero al crear el borrador** (o pintar un placeholder tipo
  "se asignara al guardar"). Definir con negocio si el numero se reserva (y por tanto se puede
  "quemar" si se descarta el borrador) o si solo se previsualiza.

**D3 [x] P1 - Columnas tipo LISTA dentro del GridDetail.** HECHO 2026-07-17. Una columna del grid
declara ahora su tipo (`text`/`select`), sus opciones y si es `required`. El JSON de columna quedo
`{id,label,type,required,options:[{id,label}]}` (las viejas `[{id,label}]` siguen valiendo). El
renderer pinta `<select>` en las columnas lista; el disenador tiene editor por columna (nombre +
tipo + opciones + obligatoria); el servidor valida requerido/opcion por celda. Verificado en Chrome
con una columna Estado (Instalado/Pendiente) obligatoria.
La grilla de contactos necesita columnas Select con opciones fijas: "Medio contacto"
(MAIL / VISITA PRESENCIAL / TELEFONO / REDES) y "Otros/Resultado" (PQRS / Soporte / Oportunidad), ademas
de "obligatorio" por columna (Nombre persona y Descripcion son requeridas).
- `GridDetail` (ADR-0021) guarda columnas en `OptionsJson` como `[{id,label}]` y el valor es un arreglo
  de filas `[{colId:"valor"}]`. Hay que CONFIRMAR/EXTENDER que una columna pueda declarar su propio
  tipo de control (texto vs select) y su lista de opciones, y que valide "requerido" por columna.
- Es P1 porque la grilla es el corazon del formulario; si las columnas solo son texto, se pierde la
  parametrizacion de Medio/Resultado.

**D4 [x] P2 - Regla condicional "mostrar Valor si venta = Si".** HECHO 2026-07-17. El runtime YA
existia (dispatcher + estado de UI aplican hide/show/required); el verbo BLOQUEAR_CAMPO_XCONDICION
ya evaluaba la condicion con accion inversa. Faltaba la AUTORIA: el tab Reglas del constructor tiene
ahora un editor inline "si {campo} {=/!=/vacio/con valor} {valor} -> {mostrar/ocultar/obligatorio/
opcional} {campo}", que crea y vincula la regla sin salir al modulo de Reglas. Verificado en Chrome:
escribir el valor oculta el campo y cambiarlo lo vuelve a mostrar.
El motor ya tiene el vinculo `Domain/Entities/FormFieldRule.cs` (pregunta -> `Rule` del RulesEngine, con
acciones de UI ocultar/mostrar/set/required al cambiar el valor). Falta CONFIRMAR que el **FormDesigner**
permita configurar esto sin codigo: elegir campo disparador ("Concreto venta"), condicion (= "Si") y
accion (mostrar + hacer requerido "Valor"). Si el disenador aun no expone reglas de campo, esa es la
tarea.

### Puntos menores / a definir (no bloquean)
- Branding por formulario: el formato viejo tiene el logo de SOLDARCO en el encabezado. Hoy `Heading` es
  solo texto e `Image` es placeholder. Definir si el encabezado del formulario admite logo/imagen del
  tenant. P3.
- `VERSION` del legacy (V01/V0/0.0) no se migro (revision es numerico). Si se quiere conservar la etiqueta
  de version textual, definir donde. P3.

---
