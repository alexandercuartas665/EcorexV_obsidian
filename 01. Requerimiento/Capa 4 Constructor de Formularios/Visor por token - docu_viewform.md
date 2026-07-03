---
tipo: ficha-pagina
capa: Capa 4 - Constructor de Formularios
pagina_origen: Bootstrap\Formularios\Modulos\Documental\docu_viewform.aspx
motor_destino: FormTokenService + DynamicFormRenderer
ruta_url_origen: /Formularios/modulos/documental/docu_viewform.aspx?token={TOKEN}
---

# Visor por token -- publicacion de formularios por URL

> Como se **comparte y renderiza un formulario por URL con token** para
> responderlo. Describe el `FormTokenService` destino (con caducidad, uso unico y
> revocacion) y conserva la resolucion del origen (`GEN_PARAMETROS`,
> `docu_viewform.aspx`) y sus fallas de seguridad como reglas de negocio a
> corregir. Stack en [[Visión y entorno]]; renderer en
> [[Motores y renderers - cl_FormCreator + crtCargaEncuestaII + cl_gestion_reglas]].

---

## 1. Vision destino -- `FormTokenService`

El destino publica un formulario emitiendo un **token opaco firmado** que un
endpoint minimal-API resuelve para servir el `DynamicFormRenderer`. Reemplaza la
resolucion fragil del origen (dos filas en `GEN_PARAMETROS` + join a otro
catalogo) por una entidad de primera clase:

```txt
form_token
   id (uuid PK), tenant_id, token (hash), form_code,
   reference, reference2, node_id (opcional, para nodo BPMN),
   created_at, expires_at, single_use, used_at, revoked_at,
   allow_anonymous (bool)
```

Flujo de resolucion destino:

1. El endpoint recibe el token, lo hashea y busca `form_token` filtrando por
   `TenantId` (RLS + `HasQueryFilter`). Consulta **parametrizada** (EF Core), no
   concatenada -- se cierra el SQLi del origen.
2. Valida estado: no expirado (`expires_at`), no usado si `single_use`, no
   revocado (`revoked_at IS NULL`).
3. Segun `allow_anonymous`, exige o no autenticacion (el origen siempre exige
   sesion; el destino lo hace politica del formulario).
4. Resuelve `form_code` -> `FormDefinition` e inyecta `DynamicFormRenderer` en
   modo produccion/visor.
5. Al enviar (`single_use`), marca `used_at`.

**Correcciones frente al origen:** tokens con caducidad, uso unico y revocacion;
sin dependencia de un catalogo externo (`[dbx.TRABAJO]`); parametrizacion total;
acceso anonimo opcional y controlado.

## 2. Renderizado (destino)

Resuelto el `form_code` y el modo, el visor monta el `DynamicFormRenderer`
(sucesor de `crtCargaEncuestaII`) con: `TenantId`, `form_code`, `reference`
(modulo/nodo), modo produccion, botones segun politica, y -- si el token trae
`node_id` -- el vinculo `form_flow_link` para que la respuesta alimente el nodo
BPMN. Config heredada como parametros: con tabs vs plano (`FORM_CARPETA_VISOR`),
ocultar guardado, manejo propio de errores. Acciones topbar del origen (Save /
Clear / Delete / Importar) -> comandos del `FormResponseService` + importador de
Excel.

## 3. Importacion masiva desde Excel

El visor origen carga datos masivos desde Excel (`gen_ctrExcelFormularios`) hacia
un contenedor `ENCUESTAS_MOV_PREGUNTAS_T` con `TIPO='TABLA'`. En el destino, un
servicio de importacion mapea columnas Excel -> preguntas del `FormContainer`
tipo tabla, valida por tipo y persiste en transaccion (respuestas -> `data jsonb`).

---

# ORIGEN (referencia) -- `docu_viewform.aspx`

> Pagina que renderiza un formulario para responderlo a partir de un token de
> URL. Usada por Autocompletado (000801), Agentes IA (000867), Pruebas
> Formularios (000891). Se conserva por sus reglas de resolucion y sus riesgos.

## O.1 Como resuelve el token (origen)

> Sorpresa: el token **NO** se busca en `GEN_TOKEN` sino en `GEN_PARAMETROS`
> (`VALOR = token`), con join a `[dbx.TRABAJO].dbo.PRO_MODULOS`.

```vb
sql_t = "
SELECT  PRO_MODULOS.NOMBRE as MODULO_NOMBRE
       ,PRO_MODULOS.CODIGO AS MODULO_CODIGO
       ,isnull((SELECT TOP 1 VALOR
                FROM [dbx.GENE].dbo.GEN_PARAMETROS AS A
                WHERE A.MODULO = GEN_PARAMETROS.MODULO
                  AND A.CODIGO = 'FORMULARIO_VIRTUAL'
                  AND A.SUCURSAL = '" & Session("Empresa") & "'),'') as FORMULARIO
FROM [dbx.GENE].dbo.GEN_PARAMETROS
     LEFT JOIN [dbx.TRABAJO].dbo.PRO_MODULOS ON GEN_PARAMETROS.MODULO = PRO_MODULOS.CODIGO
WHERE GEN_PARAMETROS.VALOR = '" & man_token & "'
  AND GEN_PARAMETROS.SUCURSAL = '" & Session("Empresa") & "'"
```

Pasos: (1) lee `Request.QueryString("token")` -> `man_token`; (2) busca fila en
`GEN_PARAMETROS` con `VALOR = man_token`; (3) esa fila tiene un `MODULO` ->
resuelve en `PRO_MODULOS`; (4) subconsulta busca `FORMULARIO_VIRTUAL` del mismo
modulo; (5) sin match -> redirect a login.

Convencion: cada visor por token = **2 filas en `GEN_PARAMETROS`** por
(SUCURSAL, MODULO): una con `VALOR=<token>` y otra con `CODIGO='FORMULARIO_VIRTUAL'`,
`VALOR=<codigo ENCUESTAS_MOV>`.

## O.2 Validacion de sesion (origen)

Tras resolver el token, exige `Session("Nombre")` != "" -- si no, redirect a
login. **El token NO funciona anonimo**; es "compartido entre usuarios
autenticados". El destino lo hace opcional (`allow_anonymous`).

## O.3 Configuracion del renderer (origen)

```vb
crtCargaEncuestaII.Sucursal = Session("Empresa")
crtCargaEncuestaII.Usuario = Session("Usuario")
crtCargaEncuestaII.Referencia = topBar.Modulo
crtCargaEncuestaII.Modulo = topBar.Modulo
crtCargaEncuestaII.Formulario = Me.FormularioBase
crtCargaEncuestaII.Produccion = 1
crtCargaEncuestaII.BotonesVisibles = False
crtCargaEncuestaII.ModoVisor = 0
crtCargaEncuestaII.noRegisterScript = "1"
crtCargaEncuestaII.FormularioManejaError = True
Call crtCargaEncuestaII.CargarModelo()
```

| Propiedad | Comportamiento |
|---|---|
| `OCULTAR_BOTON_GUARDADO = "1"` | Esconde boton Guardar |
| `FORM_CARPETA_VISOR = "NO"` | Render plano (sin tabs) |
| `FORM_CARPETA_VISOR != "NO"` | Render con tabs |

Topbar origen: `Save` -> `GuardarFormulario(0, True)`; `Clear` ->
`NuevoRegistro()`; `Delete` -> `EliminarRegistro()`; `Importar` -> modal
`PanelImportacion` con `gen_ctrExcelFormularios`.

## O.4 `GEN_TOKEN` vs `GEN_PARAMETROS` (no confundir)

`GEN_TOKEN` contiene tokens de OTRO tipo (Power BI `BI_SERVICIOS`, cotizaciones,
disenios `DISENIO_XXXXX`, codigos de modulo numericos). El visor de formularios NO
lo usa -- usa `GEN_PARAMETROS`. En el destino ambos sistemas convergen en
`form_token` (formularios) con estados y caducidad; los tokens no-formulario se
tratan aparte.

## O.5 Riesgos del origen (corregidos en destino, seccion 1)

| Riesgo | Correccion destino |
|---|---|
| SQLi en `WHERE ... VALOR = '<token>'` (string literal) | EF Core parametrizado |
| Tokens sin caducidad ni revocacion (`GEN_PARAMETROS`/`GEN_TOKEN`) | `expires_at`, `single_use`, `revoked_at` |
| Exige sesion (no anonimo) | `allow_anonymous` por politica |
| Dependencia de catalogo `[dbx.TRABAJO]` | resolucion propia sin catalogo externo |
| Multi-tenant por `Session("Empresa")` | `TenantId` + `HasQueryFilter` + RLS |

## O.6 Config de un nuevo visor (origen, para referencia ETL)

```sql
INSERT INTO ENCUESTAS_MOV (CODIGO, TITULO, ...) VALUES ('MIFORM01', 'Mi Formulario', ...);
INSERT INTO [dbx.TRABAJO].dbo.PRO_MODULOS (CODIGO, NOMBRE) VALUES ('001234', 'Mi Visor');
INSERT INTO GEN_PARAMETROS (SUCURSAL, MODULO, CODIGO, VALOR) VALUES
  ('01', '001234', 'TOKEN_ACCESO', 'ABC123XYZ...'),
  ('01', '001234', 'FORMULARIO_VIRTUAL', 'MIFORM01');
-- URL: /Formularios/modulos/documental/docu_viewform.aspx?token=ABC123XYZ...
```

En el destino esto es una sola llamada a `FormTokenService.Emit(form_code,
options)` que devuelve el token y la URL.
