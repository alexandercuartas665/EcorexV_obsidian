---
tipo: spec-cliente
proyecto: Agente Conector On-Prem
proposito: Arquitectura del agente de escritorio en VB.NET + WPF (Servicio Windows headless + app de configuracion), como conecta por SignalR, como ejecuta la consulta contra la fuente local y como se empaqueta/instala.
---

# 04 - Cliente VB.NET WPF (Servicio Windows + config)

> El agente es tonto (doc 01 seccion 5). Solo mantiene su identidad + URL del hub, se conecta,
> espera ordenes `FetchRequest`, ejecuta contra la fuente local y responde `FetchResult`.

## 1. Forma (D4): Servicio Windows + WPF

Dos ejecutables que comparten un nucleo comun:

- **EcorexAgent.Service** (Servicio Windows, headless): mantiene la conexion SignalR 24/7
  (sobrevive logout/reinicio). Es el que realmente ejecuta las ordenes. Recomendado
  `Worker Service` (host generico) instalado con `sc.exe`/instalador.
- **EcorexAgent.Config** (WPF): app de escritorio para **configurar y monitorear**: pegar
  ClientId+secreto, URL del hub, whitelist de fuentes locales, ver estado (en linea/offline),
  ver el log de ordenes, y un boton "probar conexion". NO ejecuta ordenes por si sola (o si el
  usuario prefiere solo-WPF sin servicio, ver alternativa abajo).
- **EcorexAgent.Core** (libreria compartida): cliente SignalR, ejecucion de consultas,
  serializacion de mensajes, cifrado local (DPAPI).

Alternativa simple (si no quieren servicio): solo la app WPF minimizada a bandeja que mantiene
la conexion. Menos robusto (depende de sesion iniciada). El diseno soporta ambos porque el
Core es el mismo.

## 2. Stack tecnico

- **Lenguaje: VB.NET** (pedido del usuario). WPF para la config.
- **Runtime recomendado: .NET 8 (LTS)** con `Microsoft.AspNetCore.SignalR.Client` (cliente
  SignalR oficial, soporta VB.NET). WPF corre en .NET 8 en Windows.
  - Nota: el legacy "Doom" usa .NET Framework 4.8.1; si se exige Framework, el cliente SignalR
    tambien existe para 4.8, pero se recomienda .NET 8 por soporte/perf.
- Acceso a datos segun motor: `Microsoft.Data.SqlClient` (SQL Server), `MySqlConnector`
  (MySQL/MariaDB), `Npgsql` (PostgreSQL), `Oracle.ManagedDataAccess` (Oracle). El agente carga
  el proveedor segun `connector.dbEngine` del `FetchRequest`.
- Serializacion: System.Text.Json (o el protocolo por defecto de SignalR).
- Cifrado local de secretos: **DPAPI** (`ProtectedData`, scope LocalMachine) para el secreto
  del `DataClient` en disco.

## 3. Nucleo: ciclo de vida del agente (pseudocodigo VB.NET)

```vbnet
' EcorexAgent.Core - AgentClient.vb (esquema)
Public Class AgentClient
    Private _conn As HubConnection
    Private ReadOnly _cfg As AgentConfig   ' ClientId, Secret (DPAPI), HubUrl, TokenUrl, Whitelist

    Public Async Function StartAsync() As Task
        Dim jwt = Await GetTokenAsync()    ' HMAC del secreto -> POST /api/agente/token
        _conn = New HubConnectionBuilder() _
            .WithUrl(_cfg.HubUrl, Sub(o) o.AccessTokenProvider = Function() Task.FromResult(jwt)) _
            .WithAutomaticReconnect() _
            .Build()

        ' El servidor empuja ordenes:
        _conn.On(Of FetchRequest)("FetchRequest", AddressOf OnFetchRequest)
        _conn.On("Ping", Sub() ' responder pong opcional)

        AddHandler _conn.Reconnected, Async Function(id) Await HelloAsync()
        Await _conn.StartAsync()
        Await HelloAsync()                 ' AgentHello(clientId, version, host, capabilities)
    End Function

    Private Async Sub OnFetchRequest(req As FetchRequest)
        Try
            EnforceWhitelist(req.connector)             ' seguridad: solo hosts/BD permitidos
            EnsureReadOnly(req.query.text)              ' seguridad: solo SELECT
            Dim reader = OpenReader(req.connector, req.query)  ' abre conexion LOCAL segun dbEngine
            Dim chunkIndex = 0
            For Each chunk In ReadInChunks(reader, req.paging.pageSize, req.paging.maxRows)
                Await _conn.InvokeAsync("FetchResult", New FetchResult With {
                    .correlationId = req.correlationId,
                    .chunkIndex = chunkIndex,
                    .isLast = chunk.IsLast,
                    .fields = If(chunkIndex = 0, chunk.Fields, Nothing),
                    .rows = chunk.Rows,
                    .rowCount = chunk.Rows.Count })
                chunkIndex += 1
            Next
        Catch ex As Exception
            Await _conn.InvokeAsync("FetchFailed", New FetchError With {
                .correlationId = req.correlationId, .code = Classify(ex), .message = ex.Message, .retryable = True })
        End Try
    End Sub
End Class
```

## 4. Ejecucion contra la fuente local

- El agente NO decide la consulta: la trae `FetchRequest.query.text` + `params`. Debe usar
  **comando parametrizado** (nunca concatenar) con los `params`.
- Abre la conexion con `connector.host/port/database/username` + la credencial que venga
  (`secretRef`). Politica de credencial (elegir en construccion, ver doc 01 seccion 7):
  - (a) el servidor manda la credencial (cifrada en reposo, viaja por WSS), o
  - (b) la credencial de la fuente vive SOLO en el agente (config local) y el servidor solo
    manda `secretRef` -> el agente resuelve el secreto local. **(b) es mas seguro**; se puede
    ofrecer como opcion "credencial gestionada por el agente".
- Lectura por lotes (`pageSize`) para poder chunkear; respeta `timeoutSeconds` y `maxRows`.
- Convierte cada fila a `Dictionary(Of String, String)` (valor como texto; el servidor mapea).

## 5. Seguridad del lado agente

- **Whitelist**: `AgentConfig.AllowedSources` (lista de host:puerto/BD permitidos). Si un
  `FetchRequest` apunta fuera de la whitelist -> `FetchFailed(code="BLOCKED")`. Evita que un
  servidor comprometido use al agente para escanear la LAN.
- **Solo lectura**: rechazar sentencias que no empiecen por SELECT/WITH (parse simple) y/o
  exigir que el usuario de BD sea de solo lectura. Documentarlo en la instalacion.
- **Secreto del DataClient** en disco cifrado con DPAPI (LocalMachine). Nunca en texto plano.
- **TLS**: exigir `https/wss`; rechazar certificados invalidos (no deshabilitar validacion).
- **Logs locales**: rotan, sin volcar credenciales ni filas completas (solo conteos/errores).

## 6. App WPF de configuracion (pantallas)

```
+------------------ EcorexAgent - Configuracion --------------------+
| Estado:  [ EN LINEA ]   Ultima conexion: 2026-07-11 05:40         |
|                                                                   |
| Identidad                                                         |
|   ClientId:   [ cli_2846098d60c5            ]                     |
|   Secreto:    [ ****************  ] [Pegar] (se cifra con DPAPI)   |
|   URL del hub:[ https://app.ecorex.co/hubs/agente ]              |
|                          [ Probar conexion ]                      |
|                                                                   |
| Fuentes locales permitidas (whitelist)                            |
|   [+] 10.0.0.20:1433 / db3dev   [x]                               |
|   [+] localhost:3306 / ventas   [x]                               |
|   [ Agregar fuente ]                                              |
|   [x] Solo permitir consultas SELECT (solo lectura)               |
|                                                                   |
| Servicio Windows: [ Instalado - En ejecucion ]  [Reiniciar]       |
|                                                                   |
| Registro de ordenes (ultimas)                                     |
|   05:40  FetchRequest corr=... -> 318 filas OK (1.2s)             |
|   05:35  FetchRequest corr=... -> ERROR DB_CONN                   |
+-------------------------------------------------------------------+
```

## 7. Empaque e instalacion

- Instalador (MSI/EXE, ej. WiX o Inno Setup) que: copia binarios, registra el Servicio
  Windows (`sc create EcorexAgent ...` o el instalador de Worker Service), crea la config
  inicial y lanza la WPF para pegar ClientId+secreto.
- Requisitos: Windows 10/11 o Server; .NET 8 Desktop Runtime (o self-contained); salida a
  internet por 443 hacia el host de ECOREX; acceso de red a la(s) fuente(s) locales.
- Actualizacion (backlog): auto-update del agente (el servidor puede exigir version minima).

## 8. Estados del agente (maquina de estados)

```
  Desconectado --(StartAsync + token OK)--> Conectando --(hub OK + Hello)--> EnLinea
      ^                                                                        |
      |------------------- (perdida de red / reconnect fail) ------------------|
  EnLinea --(FetchRequest)--> Ejecutando --(FetchResult/Failed)--> EnLinea
```

## 9. Checklist de construccion (cliente)

- [ ] `EcorexAgent.Core`: config (DPAPI), cliente SignalR, token HMAC, ejecucion por motor,
      chunking, whitelist, solo-lectura.
- [ ] `EcorexAgent.Service`: Worker Service host que arranca el Core y lo mantiene vivo.
- [ ] `EcorexAgent.Config` (WPF): pantallas de identidad, whitelist, estado, log, probar.
- [ ] Proveedores de BD por motor (SqlClient/MySqlConnector/Npgsql/Oracle) cargados segun
      `dbEngine`.
- [ ] Instalador + registro del servicio.
- [ ] Pruebas: contra una BD local real (SQL Server) + un hub de pruebas; simular offline y
      reconexion; verificar chunking con dataset grande.
