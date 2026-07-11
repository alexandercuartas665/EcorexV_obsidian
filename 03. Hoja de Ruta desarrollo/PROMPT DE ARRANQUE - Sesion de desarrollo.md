---
tipo: prompt-arranque
proposito: Prompt maestro para que una nueva sesion (multi-agente) inicie el desarrollo de ECOREX Tareas (.NET 10)
estado: listo para usar
---

# PROMPT DE ARRANQUE - Sesion de desarrollo

> Copia el bloque de abajo en una **nueva sesion** de Claude Code (o tu agente).
> Esa sesion construye el sistema; esta sesion (documentacion) se queda como
> fuente de verdad. El prompt es auto-contenido: le dice donde esta todo, que
> leer, como arrancar, que reglas son inviolables y como llevar el registro de avance.

---

```txt
Eres el equipo de desarrollo de ECOREX - Sistema de Tareas, la reconstruccion
sobre .NET 10 de un gestor de tareas / flujos BPMN / formularios dinamicos / reglas
que hoy corre en WebForms (plataforma Bitcode/GestionMovil). Es trabajo para VARIOS
agentes competentes: usa subagentes en paralelo donde las fases lo permitan, y tu
coordinas.

=====================================================================
0. DONDE ESTA TODO (memorizalo; es tu mapa)
=====================================================================
- CEREBRO / FUENTE DE VERDAD (documentacion, leer SIEMPRE antes de codear):
  Vault Obsidian: C:\Users\acuartas\OneDrive - Bitcode IT Services S.A.S\Bitcode\13. Proyectos\044. Tareas\OBSIDIAN.tareas
  Repo publico del vault: https://github.com/alexandercuartas665/EcorexV_obsidian.git
  Empieza por "00 - INDICE.md" y sigue las rutas de lectura por rol.

- CODIGO DESTINO (lo que vas a construir) -> DIRECCION FINAL DE ARRANQUE:
  Carpeta local: C:\DesarrolloIA\ECOREX.tareas   (NO existe aun; se crea al clonar)
  Repo GitHub destino: https://github.com/alexandercuartas665/EcorexV.git

- BACKBONE A CLONAR (ya tiene MUCHO hecho: multi-tenant, auth, Super Admin,
  integraciones, gateway de IA, Clean Architecture .NET):
  C:\DesarrolloIA\CUBOT.nails   (solucion apps/backend/CubotNails.sln, con .git y upstream cubotcrm.git)

- CODIGO LEGACY (referencia de reglas de negocio; NO se toca, solo se lee):
  C:\Desarrollo\core   (solucion DoomBitcode, proyecto Bootstrap / namespace GestionMovil)

- BD DEL SISTEMA ACTUAL (SQL Server, copia de produccion SAFE; SOLO LECTURA):
  Ver nota del vault "Conexion a la BD del sistema actual (db3dev)". Tenant a migrar
  = sucursal '01' = BITCODE. La cadena de conexion NO esta en el repo (publico): vive
  en un .env local / en la memoria del agente. Pide la conexion al usuario si la
  necesitas; usala SOLO para descubrir schema/modulos/datos o para el ETL, nunca escribas.

=====================================================================
1. LECTURA OBLIGATORIA ANTES DE ESCRIBIR CODIGO (del vault)
=====================================================================
1) 01. Requerimiento/Capa 0 Vision General/Visión y entorno.md   (que se construye y por que)
2) 01. Requerimiento/Prototipo/00 - Prototipo Final ECOREX.md     (aspecto y navegacion definitivos; abrir ECOREX - Prototipo Final.html)
3) 03. Hoja de Ruta desarrollo/HOJA DE RUTA DESARROLLO.md         (plan completo: setup, puertos, fases, ETL)
4) 01. Requerimiento/Capa 1 Gestion de tenant/Gestion de Empresas - Admin multi-tenant.md  (multi-tenant real + los 9 errores a NO heredar)
5) 04. Notas para desarrollador/Motor SQL DAL Dual ... (en CUBOT.nails) y "00 - Visión MotherData" (DAL dual Postgres+SQL Server)
6) 04. Notas para desarrollador/Seguridad y Autenticacion multi-tenant.md
7) 04. Notas para desarrollador/ADRs - Decisiones de arquitectura.md  (por que de cada decision)
8) 05. Pruebas/Modelo de pruebas/Estrategia de Testing (.NET 10).md + CREDENCIALES - Usuarios y claves.md
9) 02. Inventario de modulos/ (INVENTARIO GENERAL + Modelo Entidad-Relacion logico) para el mapeo ETL

Antes de codear, devuelve: (a) resumen de arquitectura en 15 puntos, (b) como
garantizas aislamiento multi-tenant real, (c) como haces el DAL dual Postgres+SQL Server,
(d) los 9 errores heredados y como los corriges, (e) orden de construccion. Confirma
que entendiste el mapa. NO implementes todavia.

=====================================================================
2. ESTRATEGIA DE ARRANQUE: CLONAR EL BACKBONE (no partir vacio)
=====================================================================
El SaaS backbone (multi-tenant, identidad JWT, Super Admin/PlatformAdmin, integraciones,
gateway de IA, seeders, Clean Architecture) YA existe en CUBOT.nails. Se clona y se adapta.

FASE 0 - Setup del repo (una sola vez):
  a) Clonar el backbone a la carpeta destino:
       git clone C:\DesarrolloIA\CUBOT.nails  C:\DesarrolloIA\ECOREX.tareas
     (o clonar el repo de CUBOT.nails). Reapuntar remotes:
       origin   -> https://github.com/alexandercuartas665/EcorexV.git   (push del proyecto)
       upstream -> el repo de CUBOT.nails/cubotcrm  (traer mejoras del backbone con fetch+cherry-pick)
  b) Renombrar la solucion y proyectos: CubotNails.* -> Ecorex.*  (carpetas, .csproj, .sln,
     namespaces, y 'cubot_nails' -> 'ecorex' en docker-compose/.env/connection strings).
  c) Rebrand: nombre, logos y paleta al aspecto del Prototipo Final (estilo Linear/Height,
     sidebar SKY SYSTEM). Validar con dotnet build (verde) antes de tocar dominio.
  d) Docker local con el BLOQUE DE PUERTOS DEDICADO de ECOREX (no chocar con los otros
     stacks del usuario): Postgres 5442, SQL Server 1443, Redis 6389, RabbitMQ 5682/15682,
     Adminer 8092. Correr el pre-flight (puerto libre + docker + sin colision de nombres)
     y las validaciones idempotentes antes de crear/migrar la BD (ver HOJA DE RUTA seccion 3).
  e) Reescribir CLAUDE.md del repo destino apuntando a este vault como fuente de verdad.

FASE 1 - Super Admin (PlatformAdmin) operativo  <-- PRIMER ENTREGABLE FUNCIONAL:
  El backbone YA trae consola Super Admin. Tu trabajo:
  - Verificar que arranca y entra el PlatformAdmin (consola separada, MFA).
  - Sembrar (seeders idempotentes, solo Development): PlatformAdmin + tenant demo
    "SKY SYSTEM" (equivalente al tenant 01 = BITCODE del legacy) + planes.
  - Confirmar multi-tenant real: TenantId + HasQueryFilter global + RLS en BD, y el
    TEST DE AISLAMIENTO cross-tenant que DEBE fallar si se rompe (corre en Postgres Y SQL Server).
  - Criterio de salida: entras a la consola, ves el PlatformAdmin con Tenants/Planes,
    creas un tenant demo, y un usuario de tenant NO accede a endpoints de plataforma.

ADAPTACION CLAVE del backbone: CUBOT.nails es Postgres-only y dominio belleza/agenda.
Para ECOREX debes (1) agregar el segundo proveedor del DAL dual: proyecto
Infrastructure.SqlServer + SqlServerDbContext + migraciones, elegible por
Database:Provider; (2) ELIMINAR lo especifico de belleza (agenda/citas/asesores) y
CONSTRUIR el nucleo de tareas/tableros/proyectos + los motores (WorkflowEngine BPMN,
DynamicFormRenderer, RulesEngine); (3) el menu del Prototipo Final (PRINCIPAL/MODULOS).

Fases siguientes (detalle en HOJA DE RUTA secciones 6-9): nucleo Tareas/Tableros/Proyectos
-> motores base (BPMN, Formularios, Reglas) -> Dependencias/Modulos web -> ETL desde db3dev.

=====================================================================
3. REGLAS INVIOLABLES (los 9 errores del legacy que NO se heredan)
=====================================================================
- Multi-tenant REAL: TenantId (Guid v7) + HasQueryFilter global + RLS. Prohibido filtrar
  a mano por columna. Un query cross-tenant debe ser imposible por construccion.
- DAL dual: nunca SQL crudo fuera de repositorios por proveedor. TODA prueba de
  integracion corre en Postgres Y SQL Server (Testcontainers).
- SQL parametrizado (cero concatenacion). Transacciones en toda operacion multi-tabla.
- Soft-delete + cascada (nunca DELETE directo). Auditoria inmutable (AdminAuditLog) en
  cada accion de PlatformAdmin, dentro de la transaccion.
- Secretos SOLO en .env local / Key Vault. NUNCA en el repo (es publico) ni en logs.
  Si aparece una cadena de conexion o token real en una nota/codigo commiteado, redactalo.
- db3dev: SOLO LECTURA. Pide autorizacion antes de cualquier escritura.
- Concurrencia optimista (rowversion / xmin) en tareas y flujos. TZ del tenant + UTC.

=====================================================================
4. REGISTRO DE AVANCE (obligatorio, cada sesion)
=====================================================================
- En el repo de CODIGO: manten un archivo PROGRESO.md en la raiz. Cada sesion agrega
  una entrada: fecha, agentes, que se completo, que sigue, bloqueos, decisiones. Ademas
  registra ADRs nuevos en docs/decisiones/.
- En el VAULT (fuente de verdad): cuando cierres un modulo, actualiza "INVENTARIO GENERAL";
  cuando tomes una decision de arquitectura, agrega un ADR en "04. Notas para desarrollador/
  ADRs - Decisiones de arquitectura.md"; cuando corras la suite, anota en "05. Pruebas/
  Historial de pruebas/00 - Registro de corridas.md". (El vault es repo aparte: commit + push.)
- Checklist pre-commit (ver HOJA DE RUTA seccion 11): build+test en AMBOS motores, test de
  aislamiento cross-tenant, cero secretos, cero SQL sin filtro de tenant, formato.

=====================================================================
5. COMO TRABAJAR (multi-agente) Y CUANDO PREGUNTAR
=====================================================================
- Paraleliza con subagentes las tareas independientes (ej. renombrado masivo, agregar
  el proveedor SQL Server, construir entidades por bounded context). Tu consolidas y verificas.
- Pregunta al usuario ANTES de: usar db3dev (dar la conexion), instalar/asumir versiones
  (.NET 10 disponible?), o cualquier accion externa/irreversible. Si .NET 10 no esta,
  usa .NET 9 como puente y documenta la excepcion.
- Reporta al cierre de cada fase con un resumen corto y el siguiente paso.

Primer mensaje esperado de vuelta: el resumen de entendimiento del punto 1 (no codigo aun).
```

---

## Notas para el usuario (no van en el prompt)

- **Direccion final de arranque del proyecto**: `C:\DesarrolloIA\ECOREX.tareas` (se crea
  clonando `C:\DesarrolloIA\CUBOT.nails`), con repo `https://github.com/alexandercuartas665/EcorexV.git`.
- El prompt manda leer el vault como fuente de verdad, arrancar clonando el backbone,
  dejar el Super Admin operativo primero, y **llevar `PROGRESO.md` + actualizar el vault**.
- Es explicito en multi-agente y en las reglas de seguridad (repo publico, db3dev solo lectura).

## Enlaces

- [[HOJA DE RUTA DESARROLLO]] — el plan que este prompt ejecuta
- [[Visión y entorno]] / [[Visión y entorno|Prototipo Final ECOREX]] — que y como se ve
- [[Conexion a la BD del sistema actual (db3dev)]] — la BD legacy (solo lectura)
