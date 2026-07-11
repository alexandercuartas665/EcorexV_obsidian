---
tipo: ficha-clase
clase: AdmWorkflow
perspectiva: la TAREA (como un caso carga y avanza su flujo)
archivo_origen: C:\Desarrollo\core\Bootstrap\Formularios\Modulos\Tareas\workflow\AdmWorkflow.vb
tamanio: 1613 lineas VB (WebForms, sin namespace)
dependencias: MotherData.AdmDatos, Funciones, Funciones.Cadena, cl_gestion_reglas
destino_clase: WorkflowEngine (.NET 10 / EF Core / multi-tenant)
verificado_contra_codigo: 2026-07-11
---

# AdmWorkflow - Motor de flujo de la tarea

> [!abstract] De que trata esta nota
> `AdmWorkflow` es el **proceso que acompania a una tarea** cuando esa tarea
> maneja un flujo de trabajo. Esta nota lo mira desde la TAREA: como un caso
> nace, descubre en que paso esta, avanza, ejecuta reglas y se cierra. Es
> complementaria a [[AdmWorkflow - Motor de ejecucion]] (Capa 3), que lo mira
> desde la mecanica interna del motor BPMN. Aqui interesa el ciclo de vida del
> caso. Vision maestra en [[Visión y entorno]]; aspecto destino en
> el prototipo visual (`01. Requerimiento/Prototipo/ECOREX.dc.html`); el modelo de flujos en [[00 - Visión Flujos]].
>
> **Doble encuadre:** las secciones 1 a 9 documentan el ORIGEN (como funciona
> hoy en VB/WebForms). La seccion 10 fija el DESTINO (.NET 10, `WorkflowEngine`).
> Mapeo raiz: `AdmWorkflow` -> `WorkflowEngine`; `SiguienteEstado` ->
> `AdvanceAsync`; `TAR_SEGUIMIENTO_PROCESO` / `DOC_PROCESOS*` ->
> `workflow_instance` / `workflow_step_history`; `cl_gestion_reglas` ->
> `RulesEngine`.

---

## 1. Que es y donde encaja

Una TAREA en ECOREX no siempre es una nota suelta con un responsable. Cuando la
categoria/subcategoria de la tarea tiene un **proceso** asociado (un diagrama
BPMN disenado en el modulo Flujos del Proceso, mod 000291, tabla
`DOC_PROCESOS_R`), la tarea deja de ser un item plano y se convierte en un
**caso** que recorre una secuencia de actividades. `AdmWorkflow` es la clase de
negocio que gobierna ese recorrido: no pinta nada, no es un control `.ascx`; es
el motor que lee el diagrama, escribe el estado de ejecucion en
`TAR_SEGUIMIENTO_PROCESO` y decide que paso sigue, quien lo debe ejecutar y que
reglas se disparan.

La clase no declara `Namespace` (global del proyecto Bootstrap). Instancia
internamente `tbrec As New MotherData.AdmDatos` y resuelve el catalogo fisico con
`BASE_SISTEMA = mbase_empresa.base_pricipal` (default literal `"SERVERI_MAR"`).
Todos los queries usan el alias de portabilidad `[dbx.GENE]`. La consumen las
paginas del ciclo de vida de la tarea (el detalle `ctrVertareasII`, ver
[[ctrVertareasII - Spec para reconstruir en Claude Design]]) y la pagina de
diseno del diagrama (`NEWFRONT_doc_procesos`, que llama `ValidarCambiosFlujo` al
guardar). El "quien invoca realmente" cada regla se cierra en
[[Reglas - Quien invoca realmente (cierre)]].

---

## 2. Modelo mental: Caso, Actividad, Ciclo, Flujo, Encargado

El motor razona sobre cinco ejes, que son exactamente las dimensiones que
recibe el constructor con 7 parametros
(`New(sucursal, caso, categoria, subcategoria, encargado, actividad, ActividadCiclo)`):

- **Caso** (`Caso` / columna `ID_CASO`): la instancia viva de la tarea. Si viene
  vacio, el motor esta ante un caso NUEVO y debe partir del `startEvent`.
- **Actividad** (`Actividad` / `ACTIVIDAD` = `DOC_PROCESOS_R.ID_ELEMENTO`): el
  nodo BPMN concreto donde esta parada la tarea.
- **Ciclo** (`ActividadCiclo` / `ACTIVIDAD_CICLO`, entero): la iteracion. Un
  flujo con reinicios (loops) genera el mismo nodo varias veces; el ciclo lo
  distingue. Ciclo 0 = primera pasada.
- **Flujo** (`Flujo` / columna `FLUJO`, valores `SI`/`NO`): marca la rama de una
  compuerta (gateway). Un mismo gateway abre una salida `SI` y una `NO`.
- **Encargado** (`Encargado` / `ENCARGADO`): el usuario responsable del paso,
  resuelto por ACL de cargo (`PERMISO_CARGO` + UDF `fn_ConsultaCargo`).

La categoria y subcategoria (`Categoria` + `SubCategoria`) no son estado del
caso: son la LLAVE que resuelve QUE proceso aplica, cruzando `TIPO_TAR_R` con
`TIPO_TAR_R_PRO`. Una (categoria, subcategoria) puede mapear a varios procesos.

---

## 3. Propiedades de estado

| Propiedad | Tipo | Rol para la tarea |
|---|---|---|
| `Sucursal` | String | Tenant (columna `SUCURSAL` en TODAS las tablas) |
| `Caso` | String | `ID_CASO`; vacio = caso nuevo |
| `Proceso` | String | Codigo del proceso BPMN |
| `Categoria` / `SubCategoria` | String | Resuelven el proceso via `TIPO_TAR_R`/`TIPO_TAR_R_PRO` |
| `Encargado` | String | Usuario del paso actual |
| `Actividad` | String | Nodo BPMN (`ID_ELEMENTO`) |
| `ActividadCiclo` | String | Iteracion del loop |
| `Flujo` | String | Rama de gateway (`SI`/`NO`) |
| `CantComponentes` | String | Conteo de componentes/formularios del paso (lo setea `ConsultaPlugin`) |
| `HayReglasManualesPendientes` | Boolean | La cadena de reglas se detuvo esperando accion del usuario |
| `OrdenSiguienteManual` | Integer (-1) | Orden de la proxima regla manual pendiente |
| `EsPrimeraReglaPendienteEnTarea` | Boolean | True si solo se ejecuto el primer paso; NO mostrar modal |
| `UltimaActividadAbrir` / `UltimoCicloAbrir` / `UltimoProcesoAbrir` | String | Deduplican la ejecucion autonoma al abrir el caso |

Las tres propiedades de la cadena de reglas (`HayReglasManualesPendientes`,
`OrdenSiguienteManual`, `EsPrimeraReglaPendienteEnTarea`) son la interfaz entre
el motor y la UI: cuando el motor no puede avanzar solo porque toca una regla
manual, deja estas banderas puestas y el frontend abre el modal de la regla.

---

## 4. Ciclo de vida y metodos clave

**Descubrir el paso actual: `ActividadActual() As DataSet`.** Dos ramas segun
`Caso`. Si `Caso = ""` (nuevo), busca el `startEvent` (`DOC_PROCESOS_R.PASO=1`
`AND TIPO_BPMN='startEvent'`) del proceso mapeado por categoria/subcategoria y
resuelve el `USUARIO_ACTIVIDAD` por ACL. Si el caso ya existe, hace `TOP 1` sobre
`TAR_SEGUIMIENTO_PROCESO` con `FLAG_SIGUIENTE=1` y `ENCARGADO=@encargado`
`ORDER BY REG DESC`: la fila mas reciente marcada como "siguiente" es donde va la
tarea.

**Permiso de arranque: `PermisoActividadInicial(Usuario) As Boolean`.** Igual que
la rama nueva de `ActividadActual` pero filtrando `t.USUARIO=@Usuario`; devuelve
False si el usuario no aparece en la ACL del `startEvent`.

**Sembrar el caso: `GuardarTablaSeguimiento(Sucursal, Caso, Categoria, SubCategoria) As Boolean`.**
Es el nacimiento del caso. (1) `INSERT` masivo de TODAS las actividades validas
del proceso en `TAR_SEGUIMIENTO_PROCESO` con `ESTADO='PENDIENTE'`,
`FLAG_APROBACION=1` cuando el `ID_ELEMENTO like '%Gateway%'`, y el encargado
resuelto por cargo; con `NOT EXISTS` para no duplicar. (2) `UPDATE FLAG_INA` que
apaga las filas cuyo (actividad, encargado) ya no existe en la ACL. (3) llama
`SiguienteEstado(..., EjecutarReglasAutonomas:=False)` para calcular el primer
paso sin disparar reglas al crear. Devuelve Boolean con `LogDebug` en la
excepcion.

**Cerrar el primer paso: `ActualizaEstadoTareaInicial(activ, usuario) As Boolean`.**
`UPDATE` de esa actividad a `ESTADO='TERMINADO'`, `FECHA_RES=getdate3dev()` y
`USUARIO_EJECUTOR` = usuario solo si coincide con `ENCARGADO`; luego
`SiguienteEstado`. Es el camino de cierre de la actividad inicial.

**Avanzar: `SiguienteEstado(Sucursal, Caso, Optional EjecutarReglasAutonomas=True)`.**
El corazon. Es un bucle `While hayAgente` con tope `maxIteraciones=50`. Cada
vuelta: (a) un `UPDATE ... OUTPUT` con CTE `datos` recalcula `FLAG_SIGUIENTE`
de cada fila: un nodo se marca "siguiente" si su padre esta `TERMINADO` en el
mismo ciclo, aplicando la logica de gateway (`APROBADO`/`RECHAZADO` cruzado con
`FLUJO=SI/NO`) o si `CYCLESTART=1`; (b) llama `AplicarReglas(Caso)`,
`FlujoRechazado`, `DetectarReinicios`; (c) recorre las filas recien activadas y
detecta si el nodo es un AGENTE (`TIPO_BPMN='serviceTask'` o el nombre contiene
"AGENTE"); si es agente y `EjecutarReglasAutonomas=True`, lee en `FORX_DATA` el
parametro `REGLA_AUTONOMA`; si vale `SI`, ejecuta las reglas del nodo y, si no
quedan manuales pendientes, marca el nodo `TERMINADO` y repite el bucle
(`hayAgente=True`) para encadenar actividades autonomas sin intervencion humana.
Si quedan reglas manuales, se detiene. Este metodo es el llamado por casi todos
los caminos de cierre (aprobacion, asignacion, cierre inicial).

**Ejecutar reglas de un nodo: `EjecutarReglasActividad(Sucursal, Caso, Actividad, Proceso, Usuario, Optional OrdenInicio=-1)`.**
Resetea `HayReglasManualesPendientes=False`, `OrdenSiguienteManual=-1`. Decide la
"tabla de reglas": si el nodo tiene un `FORMULARIO` en `GEN_COMPONENTES_R`
(`MODULO='FLUJO_PROCESO'`) usa ese formulario, si no usa el `Proceso`. Carga las
reglas cruzando `FORX_DATA` con `CONTROL_REGLAS_R`/`CONTROL_REGLAS`
(`CODIGO='FLUJO_PROCESO'`, excluyendo `ABRIR_FORMULARIO_MODAL`) hacia una
instancia de `cl_gestion_reglas`. En modo formulario itera cada referencia y
llama `EjecutarRegla(refItem,"0","",-1)`; en modo proceso delega en
`EjecutarReglaConCadenaAutomatica`.

**Cadena de reglas: `EjecutarReglaConCadenaAutomatica(...)`.** Recorre los
distintos `ORDEN` de las reglas de la actividad, con tope `maxPasos=50`. El
parametro `ordenInicio` codifica el modo: `-1` = solo el primer paso (al ABRIR
el caso), `-2` = ejecutar toda la cadena (al CERRAR la actividad), `>=0` = desde
ese orden. Para cada orden construye `preDOC_PARAMXML` (los parametros de esa
regla como XML) y llama `mcl_reglas.EjecutarRegla(refItem,"0","",ordenActual)`.
Antes de avanzar consulta `REGLA_AUTONOMA` por orden: si es `SI` sigue la cadena,
si es `NO` se detiene y deja `HayReglasManualesPendientes=True` +
`OrdenSiguienteManual=<siguiente>` para que el usuario la ejecute desde la
ventana flotante. `EsPrimeraReglaPendienteEnTarea` se pone True cuando la parada
ocurre en el primer paso.

**Autonomas al abrir: `EjecutarReglasAutonomasAlAbrirCaso(Sucursal, Caso, Actividad, ActividadCiclo, Proceso, Usuario)`.**
Guarda contra reejecutar: si `(Actividad, Ciclo, Proceso)` coincide con
`UltimaActividadAbrir`/`UltimoCicloAbrir`/`UltimoProcesoAbrir` retorna sin hacer
nada; si no, memoriza y llama `EjecutarReglasActividad(..., -1)`. Es el gancho
que al abrir la tarea dispara el primer paso automatico del nodo.

**Movimiento de foco: `MoverAFlujo(Caso, idActividad, TipoMovimiento, PermiteVerTodo, Usuario) As String`.**
Devuelve el `ACTIVIDAD` con `FLAG_SIGUIENTE=1` hacia adelante
(`TipoMovimiento=1`, hijos: `ID_ELEMENTO_PADRE=idActividad`) o hacia atras
(`=2`, el padre). Con `PermiteVerTodo=0` filtra por `ENCARGADO=Usuario`.

**Componentes del paso: `ConsultaPlugin(id_elemento, Caso, Formulario) As DataSet`.**
Devuelve los formularios/componentes a renderizar en el nodo (`GEN_COMPONENTES_R`
`MODULO='FLUJO_PROCESO'` UNION ALL formularios independientes de
`MODULO='CATEGORIA_SUBPROCESO'` con `ORDEN+100`), y setea `CantComponentes`. Es el
puente Flujos <-> Formularios (ver [[00 - Visión Flujos]]).

---

## 5. Integracion con el motor de reglas (cl_gestion_reglas)

Cada nodo del flujo puede llevar una **cadena de reglas** ordenadas. El motor
distingue reglas AUTONOMAS (parametro `FORX_DATA.PROPIEDAD='REGLA_AUTONOMA'`,
`CODIGO='EJECUTA_PARAM'`, valor `SI`) que corren solas al avanzar, de reglas
MANUALES (valor distinto de `SI`) que exigen que el usuario las dispare desde una
ventana flotante. `AdmWorkflow` no interpreta las reglas: arma el `DataReglas`
(desde `FORX_DATA` + `CONTROL_REGLAS`) y el `preDOC_PARAMXML`, y delega la
ejecucion en `cl_gestion_reglas.EjecutarRegla`. La cadena avanza orden por orden
mientras las reglas sean `SI`; en el primer `NO` para y publica el punto de
corte en `OrdenSiguienteManual`. Esto es lo que permite que una tarea "corra
sola" varios pasos y luego se detenga a pedir intervencion. El catalogo real de
verbos y la discrepancia entre modos declarados/implementados esta en
[[Reglas - Quien invoca realmente (cierre)]].

---

## 6. Tablas SQL que toca

- **`TAR_SEGUIMIENTO_PROCESO`** (estado de ejecucion, tabla ancha, casi
  append-only): columnas clave `SUCURSAL`, `PROCESO`, `ACTIVIDAD`,
  `ACTIVIDAD_PADRE`, `ACTIVIDAD_PASO`, `ACTIVIDAD_CICLO`, `ACTIVIDAD_NOMBRE`,
  `ENCARGADO`, `ASIGNADO`, `USUARIO_EJECUTOR`, `USUARIO_APROBADO`, `ESTADO`
  (`PENDIENTE`/`TERMINADO`/`INACTIVO`), `APROBADO` (`APROBADO`/`RECHAZADO`),
  `FLUJO` (`SI`/`NO`), `FLAG_APROBACION`, `FLAG_SIGUIENTE`, `FLAG_INA`,
  `CYCLESTART`, `ID_CASO`, `ID_REINICIO`, `PERMITE_ASIGNACION`, `TAR_CARGO`,
  `TEXTO_APROBADO`, `FECHA_REG`/`FECHA_INI`/`FECHA_RES`/`FECHA_ENTREGA`. La toca
  con INSERT (siembra, reinicio), UPDATE (avance, cierre, aprobacion, FLAG_INA) y
  DELETE (reglas de reasignacion).
- **`DOC_PROCESOS_R`**: nodos del BPMN (`ID_ELEMENTO`, `ID_ELEMENTO_PADRE`,
  `PASO`, `TIPO_BPMN`, `FLUJO`, `ID_REINICIO`, `PERMITE_ASIGNACION`).
- **`DOC_PROCESOS_RULES`**: reglas de reasignacion origen/destino (`ORIGEN`,
  `DESTINO`, `ID_ACTIVIDAD`, `ID_ACTIVIDAD_ORIGEN`, `ID_MISMO`).
- **`GEN_COMPONENTES_R`** + **`ENCUESTAS_MOV`**: componentes/formularios por nodo.
- **`TIPO_TAR_R`** + **`TIPO_TAR_R_PRO`**: mapping categoria/subcategoria ->
  proceso.
- **`PERMISO_CARGO`** + UDF **`fn_ConsultaCargo(CARGO)`**: ACL por nodo
  (`MODULO='PROCESOS_USUARIOS'`, `REFERENCIA2=PROCESO`, `REFERENCIA3=ID_ELEMENTO`).
- **`FORX_DATA`**: parametros de regla (`REGLA_AUTONOMA`, `ORDEN_REGLA`, `VALOR`).
- **`CONTROL_REGLAS`** / **`CONTROL_REGLAS_R`**: definicion de las reglas.
- **`SISTEMA`**: cabecera del caso (`CONCEPTO`=categoria, `SUBCATEGORIA`), usada
  por `ValidarCambiosFlujo`.
- Funciones SQL: `dbo.getdate3dev()` (fecha del tenant), `fn_ConsultaCargo`.

---

## 7. Reinicios y ciclos (DetectarReinicios / ProcesarReinicio)

Un flujo puede volver atras (loop). El mecanismo:
`DetectarReinicios()` busca actividades con `ID_REINICIO<>''`,
`FLAG_SIGUIENTE=1` y sin hijo en `ACTIVIDAD_CICLO+1` (es decir, un punto de
reinicio activo que aun no ha generado su nueva iteracion); las marca
`TERMINADO` y llama `ProcesarReinicio(Caso, Actividad, Proceso)`. Ese metodo
ejecuta un bloque T-SQL con `declare` de variables y un **CTE recursivo**:
calcula `@num_ciclo = MAX(ACTIVIDAD_CICLO)+1`, resuelve la rama a partir del
`ID_REINICIO` y hace un `INSERT` de las actividades del nuevo ciclo (marcando
`FLAG_SIGUIENTE=1` y `CYCLESTART=1` en el nivel 1). Finalmente hace un `DELETE`
que limpia los `CYCLESTART=1` de usuarios que no participaron en el proceso
inicial. Asi la tarea "reabre" el tramo del flujo con un ciclo incrementado, sin
perder el historico de la pasada anterior. Detalle mecanico en
[[AdmWorkflow - Motor de ejecucion]].

---

## 8. Autorizacion, reasignacion, aprobacion y rechazo

**`ControlAutorizacion() As DataSet`**: `TOP 1` de la actividad actual con
`FLAG_APROBACION=1`; trae el estado de aprobacion, `TAREAS_SUPERIORES_TERMINADAS`
y los nombres de las ramas siguientes `FLUJO_SI`/`FLUJO_NO`. Es lo que alimenta
el panel de aprobacion de un gateway.

**`ControlReasignacion() As DataSet`**: variante con `PERMITE_ASIGNACION='SI'`;
incluye la columna `ASIGNADO`. Habilita el panel de reasignar.

**`ActualizaEstadoAsignacion(UsuarioEncargado, Observacion)`**: reasigna. Hace
dos UPDATE (uno sobre los hijos por `ACTIVIDAD_PADRE`, otro sobre la actividad
por `ACTIVIDAD`) cambiando `ENCARGADO`/`ASIGNADO` y `TEXTO_APROBADO`; luego
`SiguienteEstado`.

**`ActualizaEstadoAprobacion(Aprobar As Boolean, Observacion)`**: aprueba o
rechaza un gateway. Setea `APROBADO='APROBADO'|'RECHAZADO'`, `USUARIO_APROBADO`,
`ESTADO='TERMINADO'`, fechas y `USUARIO_EJECUTOR`; luego `SiguienteEstado`, que
propaga la decision.

**`FlujoRechazado(Sucursal, Caso)`**: la poda de ramas. Primero reactiva
`INACTIVO -> PENDIENTE`. Luego itera ciclos `i=0..100`, arma con un CTE recursivo
las cadenas concatenadas de `APROBADO` y `FLUJO`, y decide con `Select Case`: si
el nodo esta `APROBADO` y su `FLUJO='NO'`, o `RECHAZADO` y `FLUJO='SI'`, lo pone
`INACTIVO` con `FLAG_SIGUIENTE=0`. Es decir, apaga la rama del gateway que NO se
tomo.

**`ValidarQuitarTerminado() As String`** y **`ValidarPonerTerminado() As String`**:
guardas de consistencia. La primera devuelve un mensaje si hay actividades
posteriores dependientes ya `TERMINADO` (no se puede revertir); la segunda avisa
si hay dependientes posteriores sin terminar o previas sin terminar (no se puede
cerrar aun). Devuelven cadena vacia cuando la operacion es valida.

**`AplicarReglas(IdActividad)`** (y su version obsoleta `AplicarReglas_ANT`): un
`DELETE` sobre `TAR_SEGUIMIENTO_PROCESO` que elimina las filas cuyo encargado no
concuerda con las reglas de reasignacion `DOC_PROCESOS_RULES` (soporta el destino
especial `ID_MISMO`, que exige que el ejecutor del origen sea el mismo usuario).
El comentario de `_ANT` documenta por que la version vieja rompia: borraba el
origen de la regla y al recruzar no encontraba los usuarios.

**`ValidarCambiosFlujo(Proceso, Sucursal)`**: se llama al GUARDAR el diagrama.
Detecta casos en curso cuyo conteo de actividades difiere de la definicion
actual del proceso y les re-siembra las faltantes con `GuardarTablaSeguimiento`;
ademas apaga (`FLAG_INA=1`) o reactiva (`FLAG_INA=0`) filas segun si el usuario
sigue o no en la ACL del proceso. No versiona: repara los casos vivos contra el
grafo nuevo.

---

## 9. Riesgos / deuda tecnica

- **SQL concatenado en todos los metodos.** Categoria, subcategoria, usuario,
  caso, observacion, actividad se interpolan directo en el string. Solo algunos
  `FORX_DATA` usan `.Replace("'","''")`. Vector de inyeccion si algun valor viene
  de UI sin sanear.
- **Topes de iteracion magicos.** `SiguienteEstado` corta en 50 vueltas,
  `EjecutarReglaConCadenaAutomatica` en 50 pasos, `FlujoRechazado` itera ciclos
  fijos `0..100`. Un flujo mal disenado puede toparse con esos limites en
  silencio.
- **Dependencia de estado ambiente.** `BASE_SISTEMA` sale de `mbase_empresa`
  (Session/global de empresa); el tenant es una columna `SUCURSAL` interpolada,
  no un filtro forzado por infraestructura.
- **UDF `fn_ConsultaCargo` opaca.** La expansion cargo->usuarios vive en la BD;
  el codigo .NET no la puede testear ni portar sin reimplementarla.
- **Sin FK ni versionado.** El vinculo nodo->formulario es por codigo string
  (`GEN_COMPONENTES_R.FORMULARIO` -> `ENCUESTAS_MOV.CODIGO`); guardar un diagrama
  con casos vivos los reejecuta contra el grafo nuevo (`ValidarCambiosFlujo`
  mitiga pero no versiona).
- **Logica de negocio en T-SQL.** Reinicios, poda de ramas y calculo de
  `FLAG_SIGUIENTE` viven en SQL con CTEs recursivos, dificiles de leer, depurar y
  cubrir con pruebas.

---

## 10. DESTINO .NET 10: reconstruccion como WorkflowEngine

En el destino, `AdmWorkflow` se porta a `WorkflowEngine`, un servicio de
aplicacion (Clean Architecture) sin SQL concatenado. Puntos del contrato:

- **Estado por instancia.** El estado migra de `TAR_SEGUIMIENTO_PROCESO` a dos
  tablas: `workflow_instance` (una por caso) y `workflow_step_history`
  (append-only, una fila por paso/ciclo, con `Guid` v7 ordenable). `FLAG_SIGUIENTE`
  -> `is_current`; `ACTIVIDAD_CICLO` -> `cycle_index`; `ESTADO`/`APROBADO`/`FLUJO`
  -> enums; `FLAG_INA` -> `is_inactive`.
- **Multi-tenant real.** No hay columna `Sucursal` interpolada: cada entidad
  lleva `TenantId` (`Guid`) y EF Core aplica `HasQueryFilter` global +
  `ITenantProvider` resuelto por middleware. Ningun query puede leer/escribir
  cruzando tenants por olvido.
- **`AdvanceAsync` transaccional.** `SiguienteEstado` -> `AdvanceAsync(caseId,
  runAutonomousRules, ct)`: el bucle de avance corre dentro de una transaccion,
  con el tope de iteraciones explicito y observable, y devuelve un
  `AdvanceResult` tipado (por ejemplo `PendingManualRule { Order }`) en vez de
  mutar propiedades globales (`HayReglasManualesPendientes`/`OrdenSiguienteManual`).
- **Reglas via `RulesEngine`.** `cl_gestion_reglas` -> `RulesEngine` con verbos en
  un registro tipado (`IReadOnlyDictionary<string, IRuleVerb>`), no
  `Activator.CreateInstance` sobre un nombre que viene del XML. `REGLA_AUTONOMA`
  se modela como propiedad del paso de regla, no como lookup ad hoc en `FORX_DATA`.
- **Persistencia y CTEs a codigo.** Los CTE recursivos de reinicio y poda de
  ramas se reescriben como logica de dominio en C# (o vistas SQL portadas y
  testeables), con concurrencia optimista (`rowversion`/`xmin`) para que dos
  cierres simultaneos del mismo caso no se pisen.
- **Eventos SignalR.** Cada avance emite un evento de dominio (`task.state.changed`)
  que se publica por SignalR: el Kanban, el detalle y los grids de otros usuarios
  se refrescan sin recargar. Antes esto era postback + literal inyectado.
- **Constructor -> contexto inmutable.** Los 7 parametros posicionales se agrupan
  en `WorkflowContext { TenantId, CaseId, Category, SubCategory, AssignedTo,
  ActivityId, CycleId }`, validado, en lugar de siete strings sueltos.

El detalle del contrato completo (`GetCurrentStepAsync`, `SeedCaseAsync`,
`GetNodeComponentsAsync`, `ValidateDefinitionChangesAsync`) esta en la ficha de
ejecucion; esta nota se limita al recorrido de la tarea.

---

## 11. Enlaces

- [[Visión y entorno]] - vision maestra del sistema y del destino .NET 10.
- el prototipo visual (`01. Requerimiento/Prototipo/ECOREX.dc.html`) - aspecto visual definitivo.
- [[00 - Visión Flujos]] - modelo conceptual de los flujos BPMN.
- [[AdmWorkflow - Motor de ejecucion]] - la misma clase desde la mecanica del
  motor (contrato destino y analisis linea a linea).
- [[ctrVertareasII - Spec para reconstruir en Claude Design]] - el panel donde el
  operador trabaja el caso y consume este motor.
- [[Reglas - Quien invoca realmente (cierre)]] - como se disparan realmente las
  reglas del nodo.
- [[Modelo Entidad-Relacion logico]] - tablas y relaciones del modelo de datos.
