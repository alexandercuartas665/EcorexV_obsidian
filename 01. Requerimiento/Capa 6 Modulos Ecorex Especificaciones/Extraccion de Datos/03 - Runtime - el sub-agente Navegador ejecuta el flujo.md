---
tipo: spec-runtime
proyecto: Extraccion de Datos
proposito: Contrato de ejecucion de un flujo por el sub-agente Navegador. DIFERIDO - el alcance actual es la configuracion; esto se documenta para que el modelo se construya sin cerrar puertas.
---

# 03 - Runtime: el sub-agente Navegador ejecuta el flujo

> **Diferido.** El alcance de ahora es la CONFIGURACION (docs 00-02). Este doc fija el CONTRATO de
> ejecucion para que el modelo de datos no cierre puertas al cablearlo despues. No es trabajo de la v1.

## 1. Como se ejecuta un flujo determinista

1. El disparo llega igual que un refresco del Contenedor de datos: por horario (el `ImportProcess`
   reusado) o a mano ("Ejecutar ahora"). El servidor comprueba que el `DataClient` del flujo este en
   linea (`IAgentRegistry.IsOnline`); si no, aplica el mismo manejo offline que ya existe
   (PendingOffline + reintento al reconectar).
2. El servidor **compila** el flujo a una secuencia de acciones tipadas: cada `ScrapeStep` se traduce a
   uno o mas `BrowserAction` (`Navigate`, `Eval`, `Wait`, `Mouse`, `Html`, `Screenshot`). Antes de
   mandar, **sustituye las variables** (`{{USER}}`...) con los valores descifrados y **FIRMA** cada JS
   (`Eval`, `Mouse` con script, `Wait` con condicion).
3. Empuja un `BrowserRequestMsg(correlationId, tenantId, actions[])` al grupo del cliente por el hub
   (el mismo canal que el Gateway; recordar el limite de mensaje ya subido a 32 MB por las capturas).
4. El agente ejecuta la secuencia en WebView2, verificando la firma y la allow-list de dominios
   (fail-closed), y responde `BrowserResultMsg` con el resultado por accion.
5. Los pasos `Extract` devuelven filas; el servidor las ingiere con `IRowIngestService` en la tabla
   destino, y cierra la corrida en la bitacora (`ImportRun`), con contadores ins/upd/del.

Es, punto por punto, el mismo patron del Gateway (doc 03 del capitulo del Agente), cambiando "ejecutar
una consulta SQL" por "ejecutar una secuencia de acciones de navegador".

## 2. Como se ejecuta un paso de IA

1. Al llegar a un paso `Kind = AI`, el servidor NO manda un JS fijo: arranca una **orquestacion** con el
   modelo elegido (AI Provider Gateway, entre los que habilite el Super Admin, con cupos por plan).
2. El agente de IA maneja el navegador a traves del **servidor MCP local** del Navegador (`browser.*`),
   que ya existe: navega, evalua, captura, lee HTML, etc. El agente ve el resultado de cada tool y
   decide el siguiente paso, como lo haria un humano.
3. El bucle esta ACOTADO por lo que el operador configuro: la **allow-list de tools** (que `browser.*`
   puede usar), el **tope de pasos** y el **tope de tiempo**. Al agotarse, el paso termina con lo que
   haya logrado.
4. Lo que el agente "saca" se estructura contra la tabla/variable destino del paso y se ingiere igual
   que un paso determinista.

Gobierno del paso de IA: el JS que el agente inyecte por el MCP local va **sin firma** (confianza
loopback: el MCP solo escucha en 127.0.0.1), pero sigue sujeto a la **allow-list de dominios** y al
consentimiento local de la colmena. La superficie de riesgo es el modelo alucinando una accion; los
topes y la allow-list de tools son la contencion.

## 3. Los dos planos, y cuando usar cada uno

| | Determinista | IA |
|---|---|---|
| Como | JS que escribe el operador, firmado | Instruccion en lenguaje natural |
| Costo | Barato (sin tokens) | Cuesta tokens (cupo por plan) |
| Exactitud | Alta si el selector aguanta | Tolera cambios de layout |
| Fragilidad | Se rompe si el sitio cambia el DOM | Mas resiliente, menos predecible |
| Ideal para | Login, paginacion, grids estables | Paginas que cambian, "entiende esto" |

Un flujo real mezcla: pasos deterministas para lo mecanico (login, navegar) y un paso de IA para lo
que exige criterio. Esa mezcla es la razon de tener los dos.

## 4. Lo que este doc DEJA FUERA (a proposito, v1)

- El cableado real de la compilacion flujo -> `BrowserAction[]`.
- La orquestacion del bucle agente<->navegador del paso de IA.
- Paginacion controlada (el `CONDICION`/`PAGINA_DESDE`/`PAGINA_HASTA` del legacy).
- Advertencias (etiqueta -> notificar/detener).

Todo eso es runtime. La v1 construye el MODELO y la UI (docs 00-02) para que esto se pueda cablear sin
remodelar nada.
