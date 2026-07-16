---
tipo: spec-cliente
proyecto: Agente Conector On-Prem
proposito: Arquitectura del agente Windows (Servicio headless + colmena WPF), como conecta por SignalR, como ejecuta contra la fuente local, como se reparten identidad/boveda/escritorio y como se empaqueta e instala.
estado: CONSTRUIDO y verificado (Olas A-C, 3, 5a-5d). Este doc refleja lo que EXISTE; lo pendiente esta marcado.
fecha: 2026-07-11 (reescrito 2026-07-16 tras D7/D8/D9 + ADR-0039)
---

# 04 - Cliente (Servicio Windows + colmena WPF)

> **Reescrito el 2026-07-16.** El titulo y el stack anteriores decian "VB.NET + WPF + .NET 8" y una
> app "de configuracion". Nada de eso sobrevivio: **D7** confirmo **.NET 10 + C#**, y **ADR-0039**
> (tras **D8**: el despliegue real son estaciones Y servidores sin sesion) reparte las piezas de otra
> forma. Este doc ya no es una propuesta: describe lo que **esta construido y verificado**.

## 1. Forma real (D4 revisada por ADR-0039)

Tres proyectos en `apps/agent/` (+ los contratos), no dos ejecutables sueltos:

| Proyecto | Que es | Por que es asi |
|---|---|---|
| **`Ecorex.Agent.Core`** | Libreria (`net10.0-windows`, **`UseWPF=false`**): canal SignalR, Gateway, Archivos, boveda, allow-lists, consentimiento, servidor MCP y canal local. | Es el MISMO codigo que hospedan el servicio y la colmena. Que compile sin WPF **es la prueba** de que no depende del escritorio. |
| **`Ecorex.Agent.Service`** | Worker Service (`UseWindowsService`), **LocalSystem**, arranque automatico. **Dueno de la identidad, del canal y de la boveda.** Atiende Gateway y Archivos. | Debe correr 24/7 **aunque nadie inicie sesion**. |
| **`Ecorex.Agent.Gui`** | La colmena WPF. **CLIENTE del servicio** por named pipe: pinta el estado real, configura (con permisos) y **presta el escritorio al Navegador**. | WebView2 exige sesion interactiva: **no vive en la sesion 0** de un servicio. |
| **`Ecorex.Contracts.Agent`** | Contratos compartidos con el servidor (protocolo, seams, DTOs). | El agente referencia SOLO esto; nunca internals del backend. |

> La **"alternativa simple" (solo WPF, sin servicio)** del doc anterior **queda DESCARTADA** por
> ADR-0039: la colmena ya no puede ser duena de nada, porque la boveda tiene ACL cerrada y ella corre
> sin elevar.

## 2. Stack real

- **Lenguaje: C#. Runtime: .NET 10** (D7). *(El doc anterior decia VB.NET + .NET 8: obsoleto.)*
- `Microsoft.AspNetCore.SignalR.Client` 10.0.0 para el canal (libreria cliente, no internals).
- `Microsoft.Data.SqlClient` 6.0.2 para el Gateway. **Solo SQL Server por ahora**; MySQL/PostgreSQL/
  Oracle siguen pendientes.
- `Microsoft.Web.WebView2` para el Navegador. *(El doc 06 s2 recomendaba Playwright; se conservo
  WebView2 -ya construido y con prior-art propio en Doom-. Playwright headless queda como add-on si
  hiciera falta navegador en un servidor SIN sesion; el seam `IBrowserSubAgent` admite ambos.)*
- Serializacion: System.Text.Json.
- **Cifrado local: DPAPI de MAQUINA** (`CRYPTPROTECT_LOCAL_MACHINE`) en `%ProgramData%\Ecorex\Agent`.
  > Nota historica que vale la pena: **este doc ya decia "scope LocalMachine" desde el principio, y
  > tenia razon**. La implementacion derivo a DPAPI de USUARIO, y eso hizo imposible por construccion
  > que el servicio leyera lo que escribia la colmena. Se corrigio en la Ola 5b (`AgentVault`).

## 3. Nucleo: ciclo de vida (lo que hacen `AgentWorker` + `RealHiveConnection`)

```csharp
// Ecorex.Agent.Service/AgentWorker.cs (esquema de lo construido)
var config = new DpapiConfigStore().Load();          // de la boveda de maquina
if (!config.IsComplete) { /* avisa y reintenta: no se cae en un equipo recien instalado */ }

var hive = new RealHiveConnection(config, new DelegatedBrowserSubAgent(ipc));
await hive.TestConnectionAsync(config);              // token HMAC -> JWT -> hub
// conectado: SignalR reconecta con backoff. El worker solo despierta si se apaga el
// servicio o si la colmena manda una identidad nueva por el pipe.
```

El canal expone `LastError` con el motivo del fallo (secreto cambiado, ClientId inexistente, reloj
desfasado >120s, URL inalcanzable): sin eso, un equipo on-prem sin escritorio es indepurable.

## 4. Ejecucion contra la fuente local (Gateway)

- El agente **no decide la consulta**: la trae `FetchRequest.query.text` + `params`, y se ejecuta
  **parametrizada** (nunca concatenada).
- **Politica de credencial: opcion (b) CONSTRUIDA** ("credencial gestionada por el agente"): la cadena
  de conexion de la fuente vive SOLO en el agente (`GatewaySourceStore`, en la boveda) y **nunca viaja
  por el canal**.
- Lectura por lotes -> chunking en varios `FetchResult`.

> **Divergencia real con el doc anterior, anotada a proposito**: se especificaba una **whitelist de
> fuentes** (`host:puerto/BD` permitidos). Lo construido es mas simple: el agente guarda **una** cadena
> de conexion local y `QueryGuard` exige solo-lectura. Es coherente con la opcion (b) -si el agente es
> quien elige a que BD apunta, un servidor comprometido no puede redirigirlo a escanear la LAN-, pero
> **no cubre** varias fuentes por agente. Si eso hace falta, ahi vuelve la whitelist.

## 5. Seguridad del lado agente (estado REAL)

| Control | Estado |
|---|---|
| Solo lectura (`QueryGuard` rechaza lo que no sea SELECT) | **CONSTRUIDO**, verificado |
| Credencial de la fuente solo en el agente (opcion b) | **CONSTRUIDO** |
| Secreto del `DataClient` cifrado (DPAPI de maquina + ACL SYSTEM/Admins) | **CONSTRUIDO** (Ola 5b) |
| Allow-list de dominios (Navegador) y de rutas (Archivos), fail-closed | **CONSTRUIDO** |
| Consentimiento local del operador por capacidad, fail-closed | **CONSTRUIDO** |
| JS del servidor **firmado** (HMAC sobre `correlationId\|payload`), verificado fail-closed | **CONSTRUIDO** |
| **Mutar config/allow-lists exige Administrador** (impersonacion en el pipe) | **CONSTRUIDO** (Ola 5c) |
| **TLS estricto: exigir https/wss y rechazar cert invalido** | **PENDIENTE (Ola 6)**. La validacion de certificados de .NET esta activa por defecto y nadie la desactiva, pero **no se exige** que la URL sea https: hoy un `http://` se acepta. |
| Logs locales sin credenciales ni filas (conteos y motivos) | **CONSTRUIDO** |

## 6. La colmena (lo que el operador ve)

No es la "pantalla de configuracion" del doc anterior: es el **panal**. Celda de Configuracion (ancla)
+ Gateway / Archivos / Navegador, que se encienden al atender. Abajo, el estado (**En linea** /
Offline) que **publica el servicio**, no ella. Clic en una capacidad sensible abre su flyout
(consentimiento + allow-list). Icono de bandeja: un panal ambar.

Detalle que se paga solo: si el operador no es administrador, el servicio **rechaza** el cambio con un
motivo legible, en vez de fallar en silencio.

## 7. Empaque e instalacion - **CONSTRUIDO** (Ola 5d)

En `deploy/agent/` del repo (ver su `README.md`):

- **`publish.ps1`**: publica **autocontenido** (~271 MB). Decision del usuario (2026-07-16): el peso
  esta bien a cambio de **no exigir el runtime de .NET** en la maquina del cliente.
- **`install.ps1`** (administrador): **crea la boveda el** (owner = Administradores + ACL cerrada),
  copia binarios a `%ProgramFiles%\ECOREX\Agente`, guarda la identidad cifrada, registra el origen del
  Visor de eventos, registra `EcorexAgent` (LocalSystem, automatico, con reintentos escalonados) y deja
  la colmena en auto-arranque de sesion.
- **`uninstall.ps1`**: quita todo y **conserva la boveda** salvo `-RemoveVault`.
- Requisitos: Windows 10/11 o Server x64; **Runtime de WebView2** solo para el Navegador (de serie en
  Win11; si falta, el Navegador falla con motivo y Gateway/Archivos siguen); salida HTTPS; **ningun
  puerto entrante**.
- **Pendiente**: instalador `.exe` firmado (Inno/WiX) que envuelva estos pasos; auto-update.

## 8. Estados del agente

```
  Offline --(identidad en boveda + token OK)--> Conectando --(hub OK + AgentHello)--> En linea
     ^                                                                                  |
     |----------------- (perdida de red / reconexion con backoff) ----------------------|
  En linea --(FetchRequest/BrowserRequest/FileRequest)--> Atendiendo --(Result/Failed)--> En linea
```

Sin colmena abierta, una orden de Navegador se responde con un **NO explicito** ("exige una sesion
interactiva"); **nunca se cuelga**. Gateway y Archivos siguen atendiendo.

## 9. Checklist de construccion (estado real)

- [x] `Ecorex.Agent.Core`: boveda (DPAPI de maquina), cliente SignalR, token HMAC, ejecucion contra
      SQL Server, chunking, solo-lectura, allow-lists, consentimiento, MCP, canal local.
- [x] `Ecorex.Agent.Service`: Worker Service que hospeda el Core y lo mantiene vivo.
- [x] `Ecorex.Agent.Gui`: la colmena (panal, estado real, flyouts, bandeja) como cliente del servicio.
- [x] Instalador + registro del servicio (`deploy/agent/`), **verificado instalando y desinstalando**.
- [x] Pruebas contra BD real (`M700_GEN`/`ciudades`) + hub real; reconexion; chunking.
- [ ] **Proveedores de BD por motor** (MySqlConnector / Npgsql / Oracle): hoy solo SQL Server.
- [ ] `.iss` + firma del ejecutable.
- [ ] TLS estricto (exigir https/wss) - Ola 6.
- [ ] Arranque tras REINICIO real del equipo (consta `StartMode=Auto` y que arranca y conecta solo).
