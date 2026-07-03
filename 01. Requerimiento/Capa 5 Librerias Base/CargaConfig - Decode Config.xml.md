---
tipo: ficha-clase
clase: CargaConfig
archivo: C:\Desarrollo\core\MotherData\Crypto\CargaConfig.vb
target_origen: .NET Framework 4.8.1
reemplazo_destino: Configuracion .NET (appsettings / user-secrets) + Key Vault
estado: adaptado a vision destino
---

# CargaConfig -- Bootstrap del Config.xml cifrado -> Configuracion .NET + Key Vault

Clase trivial (un solo metodo) que **lee y descifra `Datos\Config.xml`** al
arrancar el AppDomain. Es el punto donde el error #2 heredado -- la key de cifrado
embebida en el fuente -- se hace visible. En el destino .NET 10 este bootstrap
desaparece: la configuracion la provee el sistema de **configuracion de .NET**
(`appsettings.json` + user-secrets en dev) y los secretos viven en **Key Vault**.
Ver [[00 - Visión MotherData|00 - Vision MotherData]] y [[Visión y entorno|Vision y entorno]] seccion 14.

## 1. Que hacia (codigo completo del origen)

```vb
Public Class CargaConfig
    Public Function ReadConfig(Ruta As String) As String
        Dim man_resultado As String = ""
        Dim MAdmCrypto As New AdmCrypto
        Try
            Dim key As String = "AAAAB3NzaC3yc3####EAAAADAQABAAABAQCI5UGJloiW7...DNd"
            Dim fileReader As String = My.Computer.FileSystem.ReadAllText(Ruta)
            man_resultado = MAdmCrypto.Decrypt(fileReader, key)
        Catch ex As Exception
            man_resultado = ""
        End Try
        Return man_resultado
    End Function
End Class
```

Flujo del origen:

1. Recibe la ruta del XML cifrado (`Datos\Config.xml`).
2. Lee el archivo como texto (Base64 ciphertext).
3. Llama `AdmCrypto.Decrypt(contenido, key)` con la **key embebida**.
4. Devuelve el XML plano (o `""` si fallo).
5. `VariablesGlobales.carga_base` parsea con `XElement.Parse` y llena `Conexion()`.

## 2. Hallazgos (por que NO se porta tal cual)

| Punto | Detalle |
|---|---|
| **Key embebida en codigo** | Linea 7: literal VB. Cualquiera con acceso al repo descifra TODOS los `Config.xml` de TODOS los entornos. **Error #2 heredado.** |
| **Catch silencioso** | Si falla, devuelve `""`: el sistema arranca con `Conexion()` vacio y tira errores difusos en cada DAL call. |
| **Sin versionado de algoritmo** | Si cambia el algoritmo, el codigo no detecta version. |
| **Comentario fosil** | Linea 6: una key alterna comentada -- rastro de una migracion inconclusa. |

## 3. Como se reemplaza en .NET 10

| Aspecto | Origen (`CargaConfig`) | Destino (.NET 10) |
|---|---|---|
| Fuente de config | `Config.xml` cifrado en disco | `appsettings.json` + `appsettings.{Environment}.json` |
| Secretos (cadenas, PW) | dentro del XML cifrado | **Key Vault** (prod) / user-secrets (dev), nunca en el repo |
| Descifrado | `AdmCrypto.Decrypt` con key en `.vb` | no aplica: el provider entrega valores ya resueltos |
| Custodia de la key | literal en linea 7 | DataProtection / Key Vault; ninguna key en codigo |
| Fallo de carga | catch -> `""` (arranque roto) | **fail-fast** en el host builder con excepcion clara |
| Seleccion de motor | dentro de `Config.xml` | `Database:Provider` (ver [[00 - Visión MotherData|00 - Vision MotherData]]) |

Boceto destino:

```csharp
builder.Configuration
    .AddJsonFile("appsettings.json")
    .AddAzureKeyVault(new Uri(kvUri), new DefaultAzureCredential()); // secretos fuera del codigo
var provider = builder.Configuration["Database:Provider"]; // "Postgres" | "SqlServer"
```

## 4. Reglas duras del destino

- **Quitar la key del fuente.** Ningun secreto vive en el repositorio.
- **Versionar el formato** si algo cifrado persiste (`version` + `algorithm`).
- **No silenciar** el error de arranque -- fail-fast con mensaje accionable.

## 5. Relacionado

- [[AdmCrypto - Cifrado simétrico|AdmCrypto - Cifrado simetrico]] -- el algoritmo que este bootstrap invoca.
- [[VariablesGlobales - Conexiones y Empresa]] -- consume el XML descifrado.
- [[00 - Visión MotherData|00 - Vision MotherData]] -- vision de la capa.
