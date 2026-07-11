---
tipo: ficha-catalogo
fuente: SELECT real sobre CONTROL_REGLAS + CONTROL_REGLAS_R en db3dev (tenant 01)
estado: catalogo ORIGEN (datos vivos) + encuadre DESTINO (registro de verbos tipado)
destino_clase: RulesEngine / IRuleVerb registry
---

# Catalogo de verbos de Reglas (ORIGEN -> DESTINO)

> Catalogo REAL del motor de reglas del ORIGEN: 8 documentos y 21 reglas vivas,
> de las que **19 son `Ensamblado`** (clase .NET por reflexion), 2 `Execute`,
> 0 `mDATA`. Es la evidencia de que en la practica solo el modo Ensamblado esta
> en uso. Fija primero como el DESTINO formaliza estos verbos y conserva el
> catalogo. Enlaza con [[Visión y entorno]] (seccion 9),
> [[Visión y entorno|Prototipo Final ECOREX]] y [[Reglas - Quien invoca realmente (cierre)]].

## D. Encuadre DESTINO — registro de verbos tipado

Los 21 verbos del catalogo son las **reglas de negocio a preservar** en el ETL.
El destino los mapea a un registro tipado en `RulesEngine`, no a
`Activator.CreateInstance` sobre el `<PROCESO>` del XML (el origen carga clases
arbitrarias por nombre -> vector RCE):

```csharp
public interface IRuleVerb
{
    string Name { get; }                 // EXPANDIR_BARRAS, GENERAR_TABLAS_IA, ...
    Task<RuleResult> ExecuteAsync(RuleContext ctx, CancellationToken ct);
}
// Registro explicito en DI (no reflexion sobre input):
services.AddRuleVerb<ExpandirBarrasVerb>();
services.AddRuleVerb<GenerarTablasIaVerb>();
// ... un tipo por verbo, resuelto por Name
```

Correspondencia de familias de verbos origen -> destino:

- **Verbos IA** (`GENERAR_TABLAS_IA`, `GENERAR_TEXTO_IA`, doc `00002`) ->
  servicios que usan `Azure.AI.OpenAI` (el `clChatGPT` del origen).
- **Operaciones de formularios** (doc `00005`: `PASAR_CAMPOS`,
  `BLOQUEAR_CAMPO_XCONDICION`, `IMPORTAR_FORMULARIO_DB`, `ASIGNAR_CONSECUTIVO`,
  `ABRIR_MODAL`) -> verbos que operan sobre el `DynamicFormRenderer` destino.
- **Importadores/bots** (doc `00007`: `IMPORTAR_CSV*`, `DATA_SERVER*`) -> workers
  asincronos (RabbitMQ/MassTransit).
- **`PROPIEDADES_REGLAS`** (if-then declarativo campo->campo) -> reglas
  declarativas evaluadas sin codigo (motor de condiciones tipado).

El protocolo `PARAM_XML` con sustitucion `@@CAMPO@@` se preserva como formato de
configuracion (los operadores ya lo conocen), pero se **valida contra esquema** y
la sustitucion se hace parametrizada, nunca por concatenacion en SQL. El
historial (`CONTROL_REGLAS_H` -> `rule_history`) recibe siempre la escritura.

---

> A continuacion, el CATALOGO REAL del ORIGEN (datos vivos de produccion).

# Catálogo REAL [ORIGEN] del motor de Reglas — 8 documentos + 21 "verbos"

> Complementa [[Reglas - Motor y discrepancia]] con los datos vivos del sistema en producción. Cada documento agrupa reglas relacionadas; cada regla puede ser de tipo `Ensamblado` (clase .NET) o `Execute` (SQL).

---

## 1. Los 8 documentos de reglas en producción

| Documento | Nombre | Grupo | Estado |
|---|---|---|---|
| `00001` | GENERAR MATRIZ DE BARRAS | DOCUMENTALES | Desarrollo |
| `00002` | LLENAR DATOS CON IA | INTELIGENCIA ARTIFICIAL | Activo |
| `00003` | GENERAR ACTIVIDADES | SISTEMA ACTIVIDADES | Desarrollo |
| `00005` | OPERACIONES DE FORMULARIOS | FORMULARIOS | Activo |
| `00006` | MIGRAR SIGO | SIGO | Desarrollo |
| `00007` | PROCESOS DE IMPORTACION BOTS | PROCESOS | Activo |
| `00008` | PLANTILLAS DE IMPRESION | IMPRESIONES | Desarrollo |
| `00009` | SISTEMA ECOREX | GESTION DE USUARIOS | Desarrollo |

→ 8 documentos, **5 en Desarrollo** y **3 Activos** (en uso real).

---

## 2. Distribución de tipos de ejecución por documento

| Documento | Reglas total | `Ensamblado` | `Execute` | `mDATA` |
|---|---|---|---|---|
| `00001` GENERAR MATRIZ DE BARRAS | 1 | 1 | 0 | 0 |
| `00002` LLENAR DATOS CON IA | 2 | 1 | 1 | 0 |
| `00003` GENERAR ACTIVIDADES | 1 | 1 | 0 | 0 |
| `00005` OPERACIONES DE FORMULARIOS | 7 | 6 | 1 | 0 |
| `00006` MIGRAR SIGO | 2 | 2 | 0 | 0 |
| `00007` PROCESOS DE IMPORTACION BOTS | 6 | 6 | 0 | 0 |
| `00008` PLANTILLAS DE IMPRESION | 2 | 2 | 0 | 0 |
| `00009` SISTEMA ECOREX | 0 | 0 | 0 | 0 |

**Total: 21 reglas activas (19 `Ensamblado`, 2 `Execute`, 0 `mDATA`).**

> El modo `mDATA` que `cl_manejador_Reglas` soporta **NO se usa en este sistema**. El modo dominante es **`Ensamblado`** — clases .NET externas invocadas por reflexión. Esto refuerza que el motor `cl_manejador_Reglas` (que prioriza el flujo `mDATA`+`Execute`) está parcialmente desactualizado para el uso real.

---

## 3. Los 19 "verbos" Ensamblado — qué clases .NET invocan

Cada `CONTROL_REGLAS_R` tipo `Ensamblado` apunta a una clase .NET externa que ejecuta el verbo. Lista completa observada:

### Documento `00001` GENERAR MATRIZ DE BARRAS

| Verbo (NOMBRE) | Orden | Para qué (inferido) |
|---|---|---|
| `EXPANDIR_BARRAS` | 1 | Genera matriz de códigos de barras para etiquetado documental |

### Documento `00002` LLENAR DATOS CON IA

| Verbo | Orden | Para qué |
|---|---|---|
| `GENERAR_TABLAS_IA` | 1 | Construye tablas de datos vía GPT (`Funciones.clChatGPT`) |
| `GENERAR_TEXTO_IA` | (Execute) | Ejecuta consulta SQL que toca IA |

> Muestra real del PARAM_XML de `GENERAR_TEXTO_IA`:
> ```xml
> <PagXml>
>   <CorXml>
>     <NOMBRE>ENSAMBLADO_DATA</NOMBRE>
>     <PROCESO>GestionMovil.cl_ia_reglas_formularios</PROCESO>
>     <EVENTO>doc_generar_tabla</EVENTO>
>   </CorXml>
>   <CorXml>
>     <NOMBRE>TEST_PARAM</NOMBRE>
>     <CAMPO>CAMPO_FORMULARIO</CAMPO>
>     <OBLIGATORIO>NO</OBLIGATORIO>
>     <ETIQUETA>Formulario destino:</ETIQUETA>
>     <TIPO>TEXTO</TIPO>
>     <ROW>col-md-12</ROW>
>   </CorXml>
>   ...
> </PagXml>
> ```
> → Confirma: el `PROCESO` apunta a una clase del proyecto `Bootstrap` (`GestionMovil.cl_ia_reglas_formularios`), el `EVENTO` es el método a invocar (`doc_generar_tabla`).

### Documento `00003` GENERAR ACTIVIDADES

| Verbo | Orden | Para qué |
|---|---|---|
| `GENERAR TAREAS DESDE UNA TABLA` | 1 | Lee tabla del formulario → crea tareas en el sistema |

### Documento `00005` OPERACIONES DE FORMULARIOS ⭐ (el más rico)

| Verbo | Orden | Para qué |
|---|---|---|
| `PASAR_CAMPOS` | 1 | Copia valores entre campos de un formulario |
| `BLOQUEAR_CAMPO_XCONDICION` | 1 | Show/hide o lock de campo según condición de otro |
| `IMPORTAR_FORMULARIO_DB` | 1 | Carga datos desde BD al formulario |
| `ASIGNAR_CONSECUTIVO` | 1 | Genera consecutivo numérico (probablemente vía `Funciones.tipdoc`) |
| `EJECUTAR CONSULTA SQL` | 1 | (Execute) — corre SQL ad-hoc parametrizado |
| `ABRIR_MODAL` | 5 | Abre un modal Bootstrap desde regla |
| (+1 sin nombre extraído) | | |

### Documento `00006` MIGRAR SIGO

| Verbo | Orden | Para qué |
|---|---|---|
| `CREAR_TERCEROS` | 1 | Crea registros en tabla de terceros (importa de sistema legacy SIGO) |
| `CREAR_FACTURAS` | 2 | Crea facturas (segundo paso después de crear terceros) |

### Documento `00007` PROCESOS DE IMPORTACION BOTS ⭐

| Verbo | Orden | Para qué |
|---|---|---|
| `DATA_SERVER_EXECUTEA` | 0 | Ejecutor de operaciones contra servidor (DB?) |
| `IMPORTAR_CSV_DINAMICO` | 0 | Carga CSV con mapping dinámico |
| `CARGAR_POLIZAS_TRAVELS` | 0 | Importador específico para módulo CUBOT.travels |
| `IMPORTAR_CSV` | 1 | Carga CSV simple |
| `DATA_SERVER` | 1 | Cliente de servidor de datos (legacy) |
| `DATA_SERVER_FIREBIRD` | 1 | Cliente de servidor Firebird (BD legacy) |

### Documento `00008` PLANTILLAS DE IMPRESION

| Verbo | Orden | Para qué |
|---|---|---|
| `CREAR_COTIZACION` | 1 | Genera cotización a partir de datos |
| `IMPRIMIR_PLANTILLA` | 1 | Renderiza plantilla (probablemente `GEN_PLANTILLAS_IMPRESION`) |

---

## 4. Estructura del PARAM_XML — protocolo común

Todos los `PARAM_XML` siguen el mismo esquema:

```xml
<PagXml>
   <!-- Nodo OBLIGATORIO #1: define el ensamblado/SQL -->
   <CorXml>
      <NOMBRE>ENSAMBLADO_DATA | SQL_DATA</NOMBRE>
      <PROCESO>GestionMovil.NombreClase</PROCESO>     <!-- si Ensamblado -->
      <EVENTO>nombre_metodo</EVENTO>                    <!-- si Ensamblado -->
      <SQL>SELECT ... FROM ...</SQL>                    <!-- si mDATA/Execute -->
   </CorXml>

   <!-- Nodos OPCIONALES: parámetros configurables por el operador -->
   <CorXml>
      <NOMBRE>TEST_PARAM</NOMBRE>
      <CAMPO>CAMPO_FORMULARIO</CAMPO>                   <!-- nombre del campo del form -->
      <OBLIGATORIO>SI|NO</OBLIGATORIO>
      <DESCRIPCION>...</DESCRIPCION>
      <ETIQUETA>Formulario destino:</ETIQUETA>
      <TIPO>TEXTO|LISTA|FECHA|...</TIPO>
      <ROW>col-md-12</ROW>                              <!-- Bootstrap grid -->
      <INCRUSTADO>...</INCRUSTADO>                      <!-- HTML embebido opcional -->
      <DATA>...</DATA>                                  <!-- opciones para LISTA -->
   </CorXml>

   <!-- Nodo OPCIONAL: mensaje de retorno al usuario después de ejecutar -->
   <CorXml>
      <NOMBRE>RETURN_SMS</NOMBRE>
      <DATA>Se procesaron @@CANTIDAD@@ registros para @@CLIENTE@@</DATA>
   </CorXml>
</PagXml>
```

**Mecanismo de sustitución `@@CAMPO@@`:** Antes de ejecutar la regla, `cl_manejador_Reglas` (o el ensamblado equivalente) sustituye cada `@@NOMBRE@@` en el XML/SQL por el valor del campo correspondiente en `LlamadoXmlRegla` (datos del formulario).

---

## 5. Persistencia de regla ↔ campo de formulario

> Hallazgo del control `ctrReglas.ascx.vb`: **las reglas asignadas a campos se persisten en `FORX_DATA`**, no en una tabla puente dedicada.

### Patrón

| Columna `FORX_DATA` | Valor |
|---|---|
| `SUCURSAL` | tenant |
| `CODIGO` | `'EJECUTA_PARAM'` (constante) |
| `REFERENCIA` | ID de la pregunta (`ENCUESTAS_MOV_PREGUNTAS.REG` o `CODIGO`) |
| `ID_REGLA` | `CONTROL_REGLAS_R.REG` (la regla específica) |
| `ORDEN_REGLA` | Orden de ejecución |
| `TOKEN` | Identificador único de esta asignación |
| `VALOR` | Parámetros adicionales (si los hay) |

### Query del control `ctrReglas.LinkButtonEditarRegla_Click`

```sql
SELECT TOP 1 CONTROL_REGLAS.DOCUMENTO,
             CONTROL_REGLAS.GRUPO,
             CONTROL_REGLAS_R.REG
FROM FORX_DATA
LEFT JOIN CONTROL_REGLAS_R ON CONTROL_REGLAS_R.REG = FORX_DATA.ID_REGLA
LEFT JOIN CONTROL_REGLAS   ON CONTROL_REGLAS.DOCUMENTO = CONTROL_REGLAS_R.REGLA
WHERE FORX_DATA.SUCURSAL = @empresa
  AND FORX_DATA.CODIGO = 'EJECUTA_PARAM'
  AND FORX_DATA.REFERENCIA = @pregunta
  AND FORX_DATA.ID_REGLA = @idRegla
```

### Implicación

Cada vez que un diseñador del constructor de formularios:
1. Selecciona un campo (pregunta)
2. Abre el modal "Asignar regla"
3. Elige Grupo → Documento → Regla específica
4. Guarda

→ se crea una fila en `FORX_DATA` con `CODIGO='EJECUTA_PARAM'`. **Esta es la pieza que dispara la ejecución de la regla al interactuar con el campo en runtime.**

---

## 6. Esquemas SQL completos (las 7 tablas de reglas)

### `CONTROL_REGLAS` (cabecera — 12 cols)

| Columna | Tipo |
|---|---|
| `REG` | int |
| `DOCUMENTO` | varchar(25) — PK lógica |
| `SUCURSAL` | varchar(5) |
| `FECHA` | datetime |
| `FECHA_INI` / `FECHA_FIN` | datetime — ventana de vigencia |
| `OBSERVACION` | ntext |
| `USUARIO` | varchar(25) |
| `FECHA_NOV` | datetime |
| `ESTADO` | varchar(20) — `Activo`/`Desarrollo`/`Inactivo` |
| `NOMBRE` | varchar(100) |
| `GRUPO` | varchar(100) — agrupador (DOCUMENTALES, IA, SISTEMA ACTIVIDADES, FORMULARIOS, etc.) |

### `CONTROL_REGLAS_R` (detalle de reglas — 10 cols)

| Columna | Tipo |
|---|---|
| `REG` | int — PK |
| `NOMBRE` | varchar(100) — el "verbo" (ej. `EXPANDIR_BARRAS`) |
| `SUCURSAL` | varchar(5) |
| `TIPO` | varchar(20) — `mDATA`/`Execute`/`Ensamblado` |
| `ORDEN` | int — secuencia |
| `PARAM_XML` | ntext — config XML completo |
| `REGLA` | varchar(25) — FK lógica → `CONTROL_REGLAS.DOCUMENTO` |
| `DESCRIPCION` | ntext |
| `ESTADO` | varchar(50) — `Activo`/`Desarrollo`/`Inactivo` |
| `SCRIPT` | ntext — script auxiliar (¿JS pre-ejecución?) |

### `CONTROL_REGLAS_F` (formularios asociados — 7 cols)

| Columna | Tipo | Notas |
|---|---|---|
| `REG` | int | PK |
| `REGLA` | varchar | FK → `CONTROL_REGLAS_R.REG` |
| `SUCURSAL` | varchar | Tenant |
| **`FORMULARIO`** | varchar | FK → `ENCUESTAS_MOV.CODIGO` |
| `NOMBRE` | varchar | Etiqueta |
| `DESCRIPCION` | ntext | |
| `ESTADO` | varchar | `Activo`/`Inactivo` |

> Esta tabla **liga reglas a formularios directamente** (independiente de la asignación por campo en `FORX_DATA`). Probablemente para reglas a nivel form (validación general, post-submit).

### `CONTROL_REGLAS_H` (historial de ejecución — 9 cols)

| Columna | Tipo | Notas |
|---|---|---|
| `REG` | int | PK |
| `USUARIO` | varchar(25) | Quién disparó |
| `FECHA_REG` | datetime | |
| `REGLA` | varchar | FK |
| `CODIGO` | int | (¿REG de la regla?) |
| `LLAMADO` | ntext | El XML completo de la invocación |
| `REGISTROS` | int | Cantidad afectada |
| `MSG_ERROR` | ntext | Error si falló |
| `DURACION` | int | ms |

> **Este es el log de auditoría de cada ejecución de regla.** Permite reconstruir qué se ejecutó cuándo con qué datos. Es la tabla que `cl_manejador_Reglas.Historiallamado` debería estar escribiendo (pero la apunta a `TURNOS_REGLAS_H` que no existe → confirma la discrepancia documentada en [[Reglas - Motor y discrepancia]]).

### `PROPIEDADES_REGLAS` (reglas tipo if-then de campos — 9 cols)

| Columna | Tipo | Notas |
|---|---|---|
| `REG` | int | PK |
| `NOMBRE` | varchar | Etiqueta |
| `SUCURSAL` | varchar | Tenant |
| `PROPIEDAD` | varchar | (¿operador?) |
| `REGLA` | varchar | FK |
| **`ORIGEN`** | ntext | Campo/propiedad fuente |
| **`DESTINO`** | ntext | Campo/propiedad destino |
| **`ORIGEN_VALOR`** | ntext | Valor del campo fuente que dispara |
| **`DESTINO_VALOR`** | ntext | Valor a asignar al destino |

> Patrón "Si campo X tiene valor Y, asignar campo Z el valor W". Modelo declarativo para reglas simples.

### `REGLAS` + `REGLAS_HIS` + `REGLAS_SUC` (sistema LEGACY)

Tablas viejas:
- `REGLAS` (9 cols): REG, CODIGO, NOMBRE, OBSERVACION, CHK_INACTIVO, DOC_XML, PAR_XML, DESARROLLADOR, DET_DESARROLLADOR
- `REGLAS_HIS` (6 cols): REG, COD_REGLA, FECHA, NOMBRE, USUARIO, EVENTO
- `REGLAS_SUC` (3 cols): REG, COD_REGLA, SUCURSAL

> El sistema migró de este modelo al de `CONTROL_REGLAS*`. No usar en código nuevo.

---

## 7. Lo que aún no entiendo (TODO crítico)

- **Quién invoca realmente `EjecutarRegla`** — el código de `cl_manejador_Reglas` busca tablas que no existen en este server. Las reglas SÍ se ejecutan en producción (cierran cotizaciones, importan CSV, generan IA), entonces hay un código que las ejecuta de manera distinta. Posibilidades:
  1. Cada verbo `Ensamblado` se invoca directo desde algún sitio (no a través del manejador genérico)
  2. Hay otra clase manejadora más nueva que reemplaza `cl_manejador_Reglas`
  3. La invocación ocurre via JS desde el cliente → ASHX → clase específica
  
  Próxima sesión: **grep masivo de `GestionMovil.cl_ia_reglas_formularios`** o de `Activator.CreateInstance` para encontrar el real call site.
- Confirmar comportamiento de `RETURN_SMS` (mensaje al usuario después de ejecutar regla)
- Documentar las 19 clases .NET de los verbos (ubicación, código, dependencias)
