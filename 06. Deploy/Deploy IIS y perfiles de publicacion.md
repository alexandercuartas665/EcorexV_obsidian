---
tipo: nota-deploy
proposito: Deploy del sistema ORIGEN (WebForms a IIS) - referencia de migracion
estado: ORIGEN (legacy) - reemplazado en destino por Docker + CI/CD
---

# Deploy - IIS y perfiles de publicacion [ORIGEN]

> **Nota de ORIGEN (referencia).** Describe como se publica HOY el sistema legacy
> (Bootstrap / GestionMovil WebForms). El sistema DESTINO .NET 10 NO se despliega
> asi: ver [[Deploy a Produccion - Docker e hibrido]] (Docker + hibrido VPS/Railway)
> y [[CICD Pipeline]] (GitHub Actions). Esta nota se conserva porque el VPS Contabo
> y el SQL Server del origen son la base sobre la que se monta el destino.
>
> Bootstrap se publica con **Web Deploy / publish profiles** de Visual Studio hacia
> IIS. No hay pipeline CI/CD: el deploy es manual desde VS 2022 (un anti-patron que
> el destino corrige con el pipeline automatizado).

## Perfiles de publicacion activos

En `Bootstrap/My Project/PublishProfiles/`:

| Perfil | Entorno | Notas |
|--------|---------|-------|
| `contabo.app.pubxml` | Produccion (app) | VPS Contabo |
| `contabo.test1.pubxml` | Test 1 | VPS Contabo |
| `contabo.test2.pubxml` | Test 2 | VPS Contabo |
| `Default Settings.pubxml` | Plantilla base | |

Perfiles historicos eliminados: andina.produccion, andina.test,
azure.bitcode.teamtravels, soldarcoProduccion, test1final, test2final, test2nuevo.

## URLs de entorno

- **Produccion**: `https://app.bitcode.com.co/`
- **Desarrollo local**: `http://localhost:47640/GestionMovil/` (IIS Express desde VS)

## Procedimiento de publicacion

1. Compilar en Release: `msbuild DoomBitcode.sln /p:Configuration=Release /p:Platform="Any CPU"`
2. En VS: clic derecho en proyecto Bootstrap → Publicar → elegir perfil
3. Web Deploy empuja binarios + contenido al IIS destino
4. **Web.config NO se sobreescribe a ciegas**: cada entorno tiene su config
   (regla del proyecto: nunca modificar Web.config directamente)

## Base de datos por entorno

- La conexion se resuelve via `Config.xml` decodificado por
  [[CargaConfig - Decode Config.xml|MotherData.CargaConfig]]
- El alias `[dbx.GENE]` se traduce al catalogo fisico por entorno
  (ver [[Manejo de Datos - Alias, parametros, UDFs, consecutivos]])
- Dev/test: `db3dev` en `sql.bitcode.com.co,44566`

## Pendientes / mejoras sugeridas

- [ ] Documentar checklist post-deploy (smoke test de login + 1 modulo por capa)
- [ ] Evaluar pipeline CI/CD (en el destino CUBOT ya esta definido con GitHub Actions)
- [ ] Inventariar bindings y certificados de IIS en el VPS Contabo

## Enlaces

- [[00 - INDICE|INDICE maestro]]
- [[Plan de validacion del sistema]]
