---
tipo: referencia-sql
base: db3dev (SQL Server 2022 Developer)
servidor: sql.bitcode.com.co,44566
catalogo: db3dev
estado: tablas listadas, columnas pendientes de detallar
stack_destino: .NET 10 / EF Core 10 (DAL dual)
---

# Tablas en `db3dev` (agrupadas por familia de modulo)

> **Inventario de tablas del ORIGEN (`db3dev`, SQL Server) = fuente del ETL.**
> Este documento es el esquema fisico legacy que alimenta la migracion: es el
> punto de partida del esquema DESTINO. Cada familia de tablas aqui listada se
> reproyecta a entidades EF Core con `TenantId` -- ese mapeo ORIGEN -> DESTINO
> vive en la seccion 9 de [[Modelo Entidad-Relacion logico]] (usar ese documento
> para crear las entidades; usar este para saber que existe hoy y con que
> columnas). Vision maestra en [[Visión y entorno]].
>
> Regla de migracion: cada tabla con columna `SUCURSAL` -> entidad `ITenantScoped`
> (`SUCURSAL varchar` legacy pasa a `TenantId uuid` + filtro global + RLS); `REG`
> se conserva como `LegacyId`; `ntext` -> `text`/`nvarchar(max)`; EAV (`FORX_DATA`)
> -> `jsonb`. Estas son las 946 tablas del catalogo legacy, de las cuales este
> documento cubre las ~180 del menu visible (Tareas + Flujos + Formularios).

> Conteo realizado contra `INFORMATION_SCHEMA.TABLES` el día de exploración. **No incluye el detalle de columnas** — para cada tabla relevante, ver la ficha del módulo correspondiente o hacer:
> ```sql
> SELECT COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE
> FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'XXX'
> ```

---

## Tareas / Proyectos / Workflow (23 tablas)

| Tabla | Cols | Comentario |
|---|---|---|
| `DOC_PROYECTOS` | 19 | Cabecera de proyecto |
| `DOC_PROYECTOS_FIRMAS` | 8 | Firmas asociadas |
| `DOC_PROYECTOS_FORMS` | 5 | Formularios asociados al proyecto |
| `DOC_PROYECTOS_ORG` | 8 | Organización (jerarquía/dependencias) |
| `DOC_PROYECTOS_PRORROGAS` | 13 | Prórrogas/extensiones |
| `DOC_PROYECTOS_ROLES` | 8 | Roles dentro del proyecto |
| `DOC_PROYECTOS_SER` | 18 | Servicios del proyecto |
| `DOC_PROYECTOS_SER_HIS` | 21 | Historial de servicios |
| `DOC_PROYECTOS_SERIE` | 6 | Serialización |
| `ING_PRESUPUESTO_PROYECTO` | 4 | Presupuesto |
| `MED_TAREAS_GANTT` | 19 | Vista Gantt |
| `MED_TAREAS_PROCESO` | 7 | Pasos del proceso |
| `TAR_ESTADOS` | 6 | Catálogo de estados (parametría 000653) |
| `TAR_SEGUIMIENTO_PROCESO` | 31 | Audit/log de seguimiento (tabla ancha) |
| `TAR_TIPOS_PROYECTO` | 7 | Tipos (parametría 000690) |
| `TAR_TIPOS_PROYECTO_HITOS` | 8 | Hitos por tipo |
| `TAR_TIPOS_PROYECTO_HITOS_ACT` | 12 | Hitos x actividades |
| `TAR_TIPOS_PROYECTO_R` | 8 | Relaciones |
| `TAR_TIPOS_PROYECTO_RES` | 12 | Resultados |
| `TAR_VISITA` | 9 | Visitas (módulo `tar_visita`) |
| `TAR_WORKFLOW_HORARIOS` | 15 | **Programación temporal** del workflow |
| `TAR_WORKFLOW_PASOS` | 9 | **Pasos** de workflow en ejecución |
| `TAR_WORKFLOW_PROYECTOS` | 5 | **Asociación** workflow ↔ proyecto |

---

## Documentos (familia `DOC_*` — ~76 tablas, módulo compartido)

Familias visibles: `DOC_APROBADOS`, `DOC_ARCHIVO_CENTRAL*` (2), `DOC_BARRAS`, `DOC_BODEGA_R`, `DOC_CONTENEDORES*` (4), `DOC_DIGITALES`, `DOC_DOCUMENTOS*` (5), `DOC_EMPRESAGENERO*` (2), `DOC_ENTREVISTAS*` (8), `DOC_IMAGENES`, `DOC_INVENTARIO_SIMPLE`, `DOC_LECTURA*` (2), `DOC_ORDTRA*` (3), **`DOC_PROCESOS*`** (10), `DOC_PROPIEDADES*` (3), `DOC_PROYECTOS*` (10), `DOC_REQUERIMIENTOS*` (5), `DOC_SERIES*` (3), `DOC_TARIFA*` (2), `DOC_TIPO_NOTA`, `DOC_TIPOALM*`, `DOC_TIPOCONTENEDOR*` (2), `DOC_TRDPROCESOS`, `DOC_UBICAR`, `DOC_VALIDA`.

### Foco: `DOC_PROCESOS*` (motor de Flujos — módulo 000291)

| Tabla | Comentario |
|---|---|
| `DOC_PROCESOS` | Cabecera del proceso |
| `DOC_PROCESOS_G` | Grupos del proceso |
| `DOC_PROCESOS_GRUPOS` | Definición de grupos |
| `DOC_PROCESOS_GRUPOS_USUARIOS` | Membresía |
| `DOC_PROCESOS_PLUGINS` | Plugins por nodo |
| `DOC_PROCESOS_R` | Relación (¿roles?) |
| `DOC_PROCESOS_RU` | **Nodos visuales (Unidades del editor)** |
| `DOC_PROCESOS_RULES` | **Reglas / transiciones / aristas** |
| `DOC_PROCESOS_TRD` | Vínculo con TRD |

→ Ver [[00 - Visión Flujos]]

---

## Documental (familia `DOCU_*` y `ENCUESTAS*` — motor de Formularios)

| Tabla | Cols | Comentario |
|---|---|---|
| `DOCU_CARPETADOCUMENTO` | | Organización en carpetas |
| `DOCU_CEDU_TEMP` / `DOCU_CEDULAS` | | Validación de cédulas |
| `DOCU_GRUPODOCUMENTO` | | Agrupador |
| `DOCU_HAS` / `DOCU_HAS_SERVERII` | | Hash de contenido |
| `DOCU_HOJA_CONTROL*` (3) | | Hojas de control |
| `DOCU_INVEN*` (2) | | Inventario doc |
| `DOCU_LOTESENTREGA` | | Lotes |
| `DOCU_METROCALI_*` / `DOCU_SUPERGIROS_*` | | **Verticales por cliente** — tablas custom específicas |
| `DOCU_RANGOS_NOMINA` | | |
| `DOCU_RENDIMIENTO` | | |
| `DOCU_TAGS` / `DOCU_TAGS_R` | | Tags |
| `DOCU_TIPODOCUMENTO` | | Catálogo |
| **`ENCUESTAS_MOV`** | | **Cabecera de FORMULARIOS** ⭐ |
| `ENCUESTAS_MOV_HISTORIAL` | | Versiones del form |
| `ENCUESTAS_MOV_PREGUNTAS` | | Preguntas individuales |
| `ENCUESTAS_MOV_PREGUNTAS_CONTENEDOR` | | Agrupador |
| **`ENCUESTA_RESP`** | | **Respuestas** ⭐ |
| `ENCUESTAS_DOCU_ARCHIVOS` | | Archivos adjuntos |
| `ENCUESTAS_DOCUMENTOS` / `_RECURSOS` | | Documentos asociados |
| `ENCUESTAS_GRUPOS` / `_X_MODULOS` | | Agrupación + asociación a módulos |
| `ENCUESTAS_MODULOS` | | Asociación form ↔ módulo |

→ Ver [[00 - Visión Formularios]]

---

## Plataforma — `GEN_*` (40 tablas)

| Tabla | Cols | Comentario |
|---|---|---|
| `GEN_ACTAS` | 18 | Actas |
| `GEN_ALERTAS` | 9 | Alertas |
| `GEN_ARCHIVOS` + `_GR` + `_OCR` (3) | | Archivos del sistema + grupos + OCR |
| `GEN_BACKUP` | 9 | |
| `GEN_BIBLIOTECA` | 8 | |
| `GEN_BIOMETRICO_FACIAL` | 8 | Reconocimiento facial |
| `GEN_COMPONENTES_R` | 12 | Componentes UI |
| `GEN_DATAWARE*` (5) | | Datawarehouse |
| `GEN_DEPENDENCIA_ORGANICA` | 16 | Dependencias del organigrama (módulo 000850) |
| `GEN_ERROR` + `_R` | | Catálogo de mensajes de error (módulo 000301) |
| `GEN_FONDO_DOCUMENTAL` | 16 | TRD: fondo |
| `GEN_FORMACION_R` | 8 | Formación |
| `GEN_GESTIONIMP` | 8 | Gestión impresión |
| `GEN_HISTORICO_FILTROS` | 6 | Histórico de filtros usados |
| `GEN_IMPRESORAS` + `_R` | | Impresoras |
| `GEN_LOG` | 13 | Log de actividades |
| `GEN_MUNICIPIOS` | 5 | Catálogo de municipios CO |
| `GEN_NOTAS` | 4 | Notas del usuario (dashboard) |
| `GEN_NOTIFICA_SUC` / `_USU` / `GEN_NOTIFICACIONES` | | Notificaciones |
| `GEN_ORDEN_GRID` | 18 | Orden de columnas en grids |
| `GEN_PAIS` | 8 | Catálogo de países |
| **`GEN_PARAMETROS`** | 7 | **Parámetros por (SUCURSAL, MODULO, CODIGO) — el más usado del sistema** |
| `GEN_PLANTILLAS_IMPRESION` | 15 | Plantillas |
| `GEN_RESOLUCIONES` | 6 | Resoluciones DIAN |
| `GEN_RESPONSABILIDADES` | 5 | |
| `GEN_TABLEROS` + `_M` + `_M_R` | | Tableros / Kanban |
| `GEN_TIPOSNIT` | 6 | Tipos de NIT |
| **`GEN_TOKEN`** | 9 | **Tokens públicos para visor de formularios** ⭐ |

---

## Admin — `ADM_*` (18 tablas)

| Tabla | Cols | Comentario |
|---|---|---|
| **`ADM_CONTROLES`** + `_I` + `_V` (3) | 4-5 | **Catálogo de controles del form engine** (input, select, date, etc.) |
| `ADM_EXE` / `ADM_EXER` | | Ejecutables |
| `ADM_HTML` / `ADM_HTMLR` | | HTML/CSS storage |
| `ADM_MOD` | 3 | Modificadores |
| `ADM_PROC` / `ADM_PROR` | | Procedures |
| `ADM_REC` | 4 | Recursos |
| `ADM_SERVICIOS` | 9 | Servicios web |
| `ADM_SITIOS` + `_P` | | Sitios web |
| `ADM_USU` | 9 | Usuarios admin |
| `ADM_VAR` | 3 | Variables |
| `ADM_VEN` | 4 | Ventas (?) |
| **`ADM_WEB`** | 9 | **Módulos web (donde se definen los códigos `000xxx` del menú)** |

---

## Formularios extras

| Tabla | Cols | Comentario |
|---|---|---|
| `FORX_DATA` | 13 | Persistencia genérica de data de formulario |
| `FORX_DATA_FLUJO` | 8 | Estado del form dentro de flujo BPMN |
| `PARAMXML` | 3 | Definición XML por módulo (módulo 000057) |
| `BIB_BIBLIOTECA` | 8 | Biblioteca de snippets (módulo 000133) |
| `BIB_GRUPOS` | 5 | |
| `BIB_TECNOLOGIA` | 6 | |

---

## Pendientes de mapear

- **`USUARIO`** (en `[dbx.GENE]`) — tabla central de usuarios, columna `FLAG_3DEV`.
- **`SUCURSAL_PAR`** — parámetros por sucursal (incluye personalización de login).
- **`PRIORIDADCCS`** — catálogo de prioridades de casos.
- Tablas de **Reglas** del módulo 000802 (buscar `REGLA%`).
- Esquemas completos de las **946 tablas** restantes — completar con `INFORMATION_SCHEMA.COLUMNS` cuando se documente cada módulo.

---

## Total de tablas en `db3dev`

**946 BASE TABLE** (`SELECT COUNT(*) FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE='BASE TABLE'`).

Lo listado en este documento cubre las familias del menú visible (~ 180 tablas). Las restantes son verticales de cliente (`DOCU_METROCALI_*`, `DOCU_SUPERGIROS_*`), módulos de cartera, comercial, inventarios, contabilidad, RRHH, etc. — fuera del alcance de este levantamiento centrado en Tareas + Flujos + Formularios.

---

## Esquemas confirmados (consultados directo a db3dev)

### `DOC_PROCESOS` (cabecera de Flujos — módulo 000291)

| Columna | Tipo | Notas |
|---|---|---|
| `REG` | int (NOT NULL) | PK numérico |
| `CODIGO` | varchar(5) | Código del proceso (5 chars) |
| `NOMBRE` | varchar(300) | |
| `DESCRIPCION` | varchar(300) | |
| `SUCURSAL` | varchar(15) | Tenant |
| `GPT_CODE` | int | Vínculo con prompt GPT (campo entero, no string como sugiere el INSERT) |

> Nota: el INSERT en `NEWFRONT_doc_procesos.aspx.vb` línea ~153 inserta `GPT_CODE = ''` (string vacío). Como la columna es `int`, esto puede estar produciendo error silencioso o conversión. **Bug latente.**

### `DOC_PROCESOS_RU` (nodos visuales del BPMN — el grafo del editor)

| Columna | Tipo | Notas |
|---|---|---|
| `REG` | int | PK |
| `SUCURSAL` | varchar(5) | Tenant |
| `PROCESO` | varchar(10) | FK lógica a `DOC_PROCESOS.CODIGO` |
| `PASO` | int | Orden / step en el flujo |
| `ROL` | varchar(10) | Rol que ejecuta el nodo |
| `ID_ELEMENTO` | varchar(20) | ID del elemento visual en el canvas |

> Cada nodo del editor BPMN es una fila aquí. Las **transiciones** entre nodos están en `DOC_PROCESOS_RULES` (pendiente detallar columnas).

### `ENCUESTAS_MOV` (cabecera de FORMULARIOS — módulo 000131)

| Columna | Tipo | Notas |
|---|---|---|
| `REG` | int | PK |
| `CODIGO` | varchar(50) | Código del formulario |
| `TITULO` | varchar(300) | |
| `DESCRIPCION` | text | Largo libre |
| `CODIGO_HSEQ` | varchar(50) | Código HSEQ (Health/Safety/Environment/Quality) — relevante para formularios SST |
| `VERSION` | varchar(50) | Versión del form |
| `TIPO_FORMATO` | varchar(50) | Clasificación |
| `SUCURSAL` | varchar(10) | Tenant |

### `FORX_DATA` (respuestas genéricas)

| Columna | Tipo | Notas |
|---|---|---|
| `REG` | int | PK |
| `TABLA` | varchar | Tabla a la que se refiere |
| `REFERENCIA` | varchar | Registro padre |
| `CODIGO` | varchar | Código del campo |
| `PROPIEDAD` | varchar | Nombre de la propiedad |
| `TIPO` | varchar | Tipo de dato del valor |
| `VALOR` | ntext | **El valor mismo (siempre como string ntext)** |
| `TOKEN` | varchar | Token de sesión / acceso público |
| `USUARIO` | varchar | Quién registró |
| `FECHA_REG` | datetime | Cuándo |
| `SUCURSAL` | varchar | Tenant |
| **`ID_REGLA`** | int | **Regla que disparó/produjo este dato** (vínculo con motor de Reglas 000802) |
| **`ORDEN_REGLA`** | int | **Orden de ejecución de la regla** |

> **Patrón EAV (Entity-Attribute-Value)**: la persistencia es genérica — cualquier formulario guarda sus respuestas como filas `(TABLA, REFERENCIA, CODIGO, PROPIEDAD, VALOR)`. Esto permite que el constructor genere formularios sin migrar esquema. Las consultas son lentas por su naturaleza vertical. **Punto crítico para migración**.

### `GEN_TOKEN` (tokens para visor público)

| Columna | Tipo | Notas |
|---|---|---|
| `REG` | int | PK |
| `TOKEN` | varchar | Token aleatorio (vistos en URL: 30+ chars) |
| `REFERENCIA` / `REFERENCIA1` / `REFERENCIA2` | varchar | Hasta 3 referencias contextuales |
| `TIPO_REFERENCIA` | varchar | Discrimina qué representa el token (form, agente IA, prueba, etc.) |
| `SUCURSAL` | varchar | Tenant |
| `FECHA_REG` | datetime | Creación |
| `FECHA_ENVIO` | datetime | Cuándo se envió al destinatario |

> No hay columna explícita de **caducidad** ni de **uso/revocación** — los tokens parecen permanentes. **Riesgo de seguridad** si el token se filtra.
