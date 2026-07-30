---
tipo: nota-desarrollador
tema: Motor de Reportes y BI - despliegue a produccion EN ESPERA
fecha: 2026-07-30
estado: BLOQUEADO por clave de licencia Bold Reports + confirmacion Docker
---

# Motor de Reportes - despliegue a produccion EN ESPERA (clave de licencia Bold)

> [!warning] NO desplegar main a prod todavia
> El Motor de Reportes (incluida la Ola 2 con el diseniador/visor **Bold Reports**) y la
> **gobernanza por rol** ya estan en `main` (commit `94187f2`), compilando y con el test dual
> `ReportGovernanceTests` en **4/4 verde (PostgreSQL + SQL Server)**. PERO el deploy a prod queda
> EN ESPERA hasta resolver dos cosas del vendor Bold. Decision del usuario (2026-07-30):
> "para la clave, dame una espera" -> se despliega cuando la clave este lista y pasen los tests.

## Que esta LISTO (en main / github, sin desplegar)

- Rama `feat/motor-reportes` unida a `main`. `main` = `origin/main` = `origin/fase-0/clon-backbone`
  = `94187f2`. **Prod sigue corriendo `bcdcb27`** (reportes Olas 0/1/3/4 SIN Bold, SIN gobernanza).
- Reportes: dashboards ECharts (`/reportes/tablero`), autoria por IA (`/reportes/ia`), imprimibles
  Bold RDL (`/reportes/imprimibles`, visor y diseniador drag-drop), showcase (`/reportes/actividades-sistema`).
- Gobernanza por rol (ADR-0051): entidad `ReportDefinitionRole` + migracion dual `AddReportDefinitionRole`,
  policies `Perm:reportes/{tablero,ia,imprimibles,admin}:{View,Create,Edit,Delete}`, grupo de menu
  "Reportes", y la pagina `/reportes/admin` (asignar reportes a roles). El editor exige `Disenar`
  (`Perm:reportes/imprimibles:Edit`).

## El BLOQUEO (2 gates del vendor Bold)

1. **CLAVE DE LICENCIA Bold (Community)** -> PENDIENTE, el usuario la registra.
   - Es un SECRETO: va en `user-secrets` (`Bold:LicenseKey`) o env var (`Bold__LicenseKey`).
   - NUNCA versionar la clave (repo publico). Debe quedar gitignored.
   - Sin la clave, el diseniador/visor Bold corre en modo evaluacion o falla.
2. **Bold en el Docker Linux de prod** -> CONFIRMAR que `BoldReports.Net.Core` construye y corre en
   el contenedor de prod (Debian slim). Riesgo: la community de Bold agrupa Docker/Kubernetes en el
   tier de pago, y prod es Linux Docker. Depende de `System.Drawing.Common` /
   `Microsoft.Windows.Compatibility` en Linux.

Runbook turnkey (paquetes, registro de licencia, Web Reporting API tenant-safe, verificacion):
`docs/motor-reportes-ola2-embed-bold.md` (repo EcorexV).

## Que hacer CUANDO la clave este lista (pasos de deploy)

1. Registrar la clave Bold como secreto en prod (env `Bold__LicenseKey`, en `.env` de `/opt/ecorex`,
   NO en el repo).
2. `./backup.sh` en prod (siempre antes de desplegar).
3. Doble push ya hecho; en prod: `docker compose -f docker-compose.from-git.yml build --no-cache`
   (construye `fase-0/clon-backbone`; si Bold NO compila en Linux, el build FALLA aqui y prod NO
   cambia -> ese es el gate real) `&& ... up -d`. La migracion `AddReportDefinitionRole` se aplica
   sola (`ECOREX_RUN_MIGRATIONS=true`).
4. Sembrar menu + modulos de Reportes en los tenants Standard: arrancar una vez con
   `ECOREX_MENU_REPORTES=true` (idempotente; mismo patron que `ECOREX_MENU_GESDOC`).
5. Verificar: `/login`=200, tabla `report_definitions_roles` existe, el grupo "Reportes" aparece en
   el menu, y un rol sin asignacion NO ve un reporte asignado a otro rol.

## Mientras tanto (sin la clave)

Se puede validar la GOBERNANZA en dev local con `ECOREX_MENU_REPORTES=true` (el menu, la matriz de
roles, la pagina `/reportes/admin` y el filtrado por rol funcionan sin Bold; solo el diseniador/visor
Bold necesita la clave).

Relacionado: `docs/decisiones/ADR-0051-motor-de-reportes-y-bi-stack-y-gates.md`,
`docs/motor-reportes-ola2-embed-bold.md`, memoria de proyecto `motor-reportes-decision`.
