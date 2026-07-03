# ECOREX Tareas - Vault de documentacion (OBSIDIAN.tareas)

Cerebro de ingenieria del sistema **ECOREX - Sistema de Tareas**: la reconstruccion
sobre **.NET 10** del gestor de tareas / flujos BPMN / formularios dinamicos / reglas
que hoy corre en WebForms (plataforma `GestionMovil` de Bitcode).

Este vault Obsidian describe el **sistema DESTINO** (ASP.NET Core 10 / EF Core 10 /
Blazor / DAL dual PostgreSQL o SQL Server / multi-tenant real) y conserva el **sistema
ORIGEN** (WebForms / VB.NET / `MotherData`) como referencia de migracion. Cada nota
tecnica separa ORIGEN (que hacia el legacy, plano del ETL) de DESTINO (como se
reconstruye).

- **Repo de codigo destino:** https://github.com/alexandercuartas665/EcorexV.git
- **Punto de entrada:** abrir [`00 - INDICE.md`](00%20-%20INDICE.md)
- **Aspecto objetivo:** `01. Requerimiento/Prototipo/00 - Prototipo Final ECOREX.md`

## Estructura

```
01. Requerimiento/     Capas 0-7 (Vision, Gestion de tenant, Tareas, Flujos BPMN,
                       Formularios, Librerias Base, Modulos Ecorex, Agentes de IA)
                       + Prototipo (HTML navegable + capturas)
02. Inventario de modulos/   Inventario, mapa de modulos, modelo E-R, tablas
03. Hoja de Ruta desarrollo/ Plan de construccion .NET 10 (setup, ETL, fases)
04. Notas para desarrollador/ Manejo de datos, Seguridad/Auth, ADRs
05. Pruebas/           Estrategia de Testing (.NET 10) + validacion del origen
06. Deploy/            Docker/hibrido, CI/CD, Backup y DR
10. Historias de Usuario/    Personas + narrativa E2E
```

## Como leerlo

1. `01. Requerimiento/Capa 0 Vision General/Visión y entorno.md` (que se construye y por que)
2. `01. Requerimiento/Prototipo/00 - Prototipo Final ECOREX.md` (aspecto)
3. `03. Hoja de Ruta desarrollo/HOJA DE RUTA DESARROLLO.md` (como empezar)
4. Las capas tecnicas segun el modulo que se vaya a construir

> Documentacion generada como espejo del esquema del vault CUBOT.nails. Convencion:
> notas en ASCII, wikilinks por nombre de archivo.
