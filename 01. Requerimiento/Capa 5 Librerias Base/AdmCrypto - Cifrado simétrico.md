---
tipo: ficha-clase
clase: AdmCrypto
archivo: C:\Desarrollo\core\MotherData\Crypto\AdmCrypto.vb
dependencias: System.Security.Cryptography (MD5 + TripleDES)
target_origen: .NET Framework 4.8.1
reemplazo_destino: IDataProtectionProvider (DataProtection) + AES-256-GCM
estado: adaptado a vision destino
---

# AdmCrypto -- Cifrado simetrico (TripleDES + MD5) -> DataProtection

Clase de utilidad criptografica del origen. **Dos metodos publicos:
`Encrypt(text, hash)` y `Decrypt(text, hash)`.** Cifra los `Config.xml` y valores
sensibles cortos. En el destino .NET 10 su rol lo asume la **API DataProtection**
(`IDataProtectionProvider`) para secretos gestionados por el framework, y
**AES-256-GCM** para cifrado explicito de valores nuevos. Ver
[[00 - Visión MotherData|00 - Vision MotherData]] y [[Visión y entorno|Vision y entorno]] seccion 14.

## 1. Que hacia (algoritmo del origen)

1. La `hash` (string clave) se reduce a 16 bytes con `MD5.ComputeHash(...)`.
2. Esos 16 bytes se usan como **key de TripleDES**.
3. Modo: **`CipherMode.ECB`** (sin IV -- mismo plaintext = mismo ciphertext).
4. Padding: `PKCS7`.
5. Encrypt: UTF8 -> TripleDES -> Base64. Decrypt: Base64 -> TripleDES -> UTF8.

```vb
Public Function Encrypt(ByVal text As String, ByVal hash As String) As String
    Dim data As Byte() = UTF8Encoding.UTF8.GetBytes(text)
    Using md5 As MD5CryptoServiceProvider = New MD5CryptoServiceProvider()
        Dim keys As Byte() = md5.ComputeHash(UTF8Encoding.UTF8.GetBytes(hash))
        Using tripleDES As New TripleDESCryptoServiceProvider() With {
            .Key = keys, .Mode = CipherMode.ECB, .Padding = PaddingMode.PKCS7
        }
            Return Convert.ToBase64String(tripleDES.CreateEncryptor()
                                                    .TransformFinalBlock(data, 0, data.Length))
        End Using
    End Using
End Function

Public Function Decrypt(ByVal text As String, ByVal hash As String) As String
    Try
        ... (espejo del Encrypt)
    Catch
        Return text   ' <-- silenciamiento: si falla, retorna el input crudo
    End Try
End Function
```

## 2. Uso real en el origen

- **`CargaConfig.ReadConfig(Ruta)`** llama `Decrypt(fileReader, key)` con `key`
  literal embebida en el fuente para descifrar `Datos\Config.xml`. Ver
  [[CargaConfig - Decode Config.xml]].
- Puntualmente se cifran/descifran valores cortos (tokens, parametros sensibles).

## 3. Riesgos documentados (por que NO se porta tal cual)

| Riesgo | Detalle |
|---|---|
| **ECB sin IV** | Patrones repetidos del plaintext se reflejan en el ciphertext. |
| **MD5 key derivation** | MD5 esta roto; no es derivacion moderna (PBKDF2/Argon2). |
| **TripleDES** | Bloque de 64 bits, deprecated por NIST; nuevo desarrollo debe usar AES-256-GCM. |
| **Catch silencioso** | `Decrypt` retorna el input crudo si falla: un `Config.xml` mal cifrado se interpreta como XML plano y la app arranca con conexiones rotas. |
| **Key en codigo fuente** | Ver [[CargaConfig - Decode Config.xml]]: clave embebida en literal VB. Quien lee el repo descifra todos los configs. Es el error #2 heredado. |

## 4. Como se reemplaza en .NET 10

| Aspecto | Origen (`AdmCrypto`) | Destino (.NET 10) |
|---|---|---|
| Secretos de config | TripleDES/ECB con key embebida | `IDataProtectionProvider` (keys gestionadas fuera del codigo) |
| Cifrado explicito de valores | TripleDES/MD5 | **AES-256-GCM** con IV aleatorio por mensaje + tag de autenticacion |
| Derivacion de clave | `MD5.ComputeHash` | **PBKDF2** o **Argon2id** |
| Custodia de claves | literal en `.vb` | **Azure Key Vault** / DataProtection (keys persistidas cifradas) |
| Fallo de descifrado | catch silencioso -> input crudo | **fail-fast** con excepcion y mensaje claro |

- Mantener compatibilidad **bidireccional** durante la transicion: descifrar
  legacy (TripleDES) y cifrar nuevo (AES-256-GCM) hasta re-encriptar todo.
- Ningun secreto vive en el repositorio (regla dura de [[Visión y entorno|Vision y entorno]] sec. 14).

## 5. Relacionado

- [[CargaConfig - Decode Config.xml]] -- unico consumidor critico (key embebida).
- [[VariablesGlobales - Conexiones y Empresa]] -- consume el XML descifrado.
- [[00 - Visión MotherData|00 - Vision MotherData]] -- vision de la capa.
