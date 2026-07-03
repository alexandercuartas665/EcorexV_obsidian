---
tipo: nota-deploy
capa: transversal
proposito: pipeline CI/CD del sistema destino ECOREX Tareas (.NET 10) - build, test dual, deploy, rollback
estado: planeado (destino de migracion)
---

# CI/CD Pipeline

> GitHub Actions ejecuta build + test dual (PostgreSQL / SQL Server) + deploy
> automatico a environments segun rama. Rollback en <2 min. Pipeline del sistema
> DESTINO .NET 10 (el origen se publica manual por Web Deploy — ver
> [[Deploy IIS y perfiles de publicacion]]).

## 1. Ambientes

| Ambiente | Rama | URL | Auto-deploy | Aprobacion |
|---|---|---|---|---|
| Dev | `develop` | `dev.tareas.bitcode.com.co` | Si (cada push) | No |
| Staging | `main` | `staging.tareas.bitcode.com.co` | Si (cada merge) | No |
| Produccion | tag `v*.*.*` | `app.tareas.bitcode.com.co` | Si (con gate) | Lider tecnico |
| PlatformAdmin | tag `admin-v*.*.*` | `admin.tareas.bitcode.com.co` | Si (gate) | Doble aprobacion |

## 2. Estructura de workflows

```
.github/workflows/
├── pr-check.yml              (PR -> build + test dual + coverage)
├── deploy-dev.yml            (push develop -> deploy dev)
├── deploy-staging.yml        (merge main -> deploy staging)
├── deploy-prod.yml           (tag -> deploy prod con gate)
├── nightly-loadtest.yml      (2am -> k6 contra staging)
├── weekly-security.yml       (Sabado -> OWASP ZAP + dependency check)
└── monthly-mutation.yml      (dia 1 -> Stryker.NET)
```

## 3. PR check (`pr-check.yml`) - 8 min target

El punto NO negociable: **los tests corren en AMBOS motores** (matriz Postgres +
SQL Server), porque el DAL es dual.

```yaml
name: PR Check
on:
  pull_request:
    branches: [develop, main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        provider: [Postgres, SqlServer]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
      - name: Restore
        run: dotnet restore
      - name: Build
        run: dotnet build --no-restore -c Release
      - name: Test (${{ matrix.provider }})
        run: |
          dotnet test --no-build -c Release \
            --collect:"XPlat Code Coverage" \
            --settings coverlet.runsettings
        env:
          TEST_PROVIDER: ${{ matrix.provider }}
      - name: Coverage gate (>=70%)
        run: ./scripts/coverage-gate.sh 70

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: dotnet format --verify-no-changes

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Vulnerable packages
        run: dotnet list package --vulnerable --include-transitive
      - name: SAST
        uses: github/codeql-action/analyze@v3
```

El test cross-tenant de aislamiento (un tenant NO ve datos de otro) es obligatorio
en la suite y DEBE fallar si alguien rompe el `HasQueryFilter` o la RLS.

## 4. Deploy Dev (`deploy-dev.yml`) - 5 min

```yaml
name: Deploy Dev
on:
  push:
    branches: [develop]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Build image
        run: |
          docker build -t bitcode/ecorex-tareas:${{ github.sha }} .
          docker tag bitcode/ecorex-tareas:${{ github.sha }} bitcode/ecorex-tareas:dev
      - name: Push
        run: |
          echo "${{ secrets.DOCKER_TOKEN }}" | docker login -u bitcode --password-stdin
          docker push bitcode/ecorex-tareas:${{ github.sha }}
          docker push bitcode/ecorex-tareas:dev
      - name: Deploy (Railway o VPS Contabo)
        run: ./scripts/deploy.sh dev bitcode/ecorex-tareas:dev
      - name: Migraciones (motor de dev)
        run: docker run --rm bitcode/ecorex-tareas:${{ github.sha }} dotnet ef database update
      - name: Smoke test
        run: curl -f https://dev.tareas.bitcode.com.co/health || exit 1
```

## 5. Deploy Prod (`deploy-prod.yml`) - con gate manual

```yaml
name: Deploy Prod
on:
  push:
    tags: ['v*.*.*']
jobs:
  build:      # build imagen con version SemVer
    ...
  approval:
    needs: build
    environment: { name: production, url: https://app.tareas.bitcode.com.co }
    steps:
      - run: echo "Lider tecnico aprueba en el UI de GH Actions"
  deploy:
    needs: approval
    steps:
      - name: Blue/Green swap
        run: ./scripts/blue-green-swap.sh ${{ github.ref_name }}
      - name: Verify
        run: for i in {1..30}; do curl -f https://app.tareas.bitcode.com.co/health && break; sleep 5; done
      - name: Notify Slack
        run: ./scripts/notify.sh "Deploy ${{ github.ref_name }} completado"
  rollback-on-failure:
    if: failure()
    needs: deploy
    steps:
      - run: ./scripts/blue-green-swap.sh previous
```

## 6. Blue/Green swap

- **Blue**: version actual en produccion.
- **Green**: nueva version subida (recibe migraciones y smoke test antes del swap).
- **Switch**: cambia el slot activo via API del proveedor / reverse proxy.
- **Rollback**: vuelve a Blue en 1 comando (<30s).

## 7. Migraciones en deploy (dual)

**Regla**: las migraciones se aplican ANTES del swap, contra el slot Green que ya
tiene la imagen nueva pero aun no recibe trafico. Por proveedor:

```bash
# Green sube con imagen nueva
./scripts/deploy.sh green bitcode/ecorex-tareas:${TAG}

# Migrar segun el motor del entorno/tenant
docker exec ecorex-green dotnet ef database update --context PostgresDbContext
docker exec ecorex-green dotnet ef database update --context SqlServerDbContext

# Swap
./scripts/blue-green-swap.sh green
```

Si una migracion falla -> Green se descarta, Blue sigue sirviendo, alerta. Nunca
se hace swap con migraciones a medias.

## 8. Feature flags

`IFeatureManager` (Microsoft.FeatureManagement, self-hosted): el deploy siempre
incluye el codigo nuevo; los flags controlan quien lo ve (per-tenant, per-user,
per-plan). Rollback de una feature = apagar el flag, sin re-deploy. Encaja con los
planes del [[Gestion de Empresas - Admin multi-tenant|PlatformAdmin]].

## 9. Notificaciones y observabilidad

- **Slack #ecorex-deploys**: cada deploy exitoso/fallido.
- **Grafana annotations**: cada deploy anota metricas (correlacionar spikes con releases).
- **OpenTelemetry / APM**: p95 de latencia por endpoint pre/post deploy.

## 10. Rollback manual

```bash
git tag --list 'v*' --sort=-v:refname | head -5
gh workflow run rollback.yml -f target_version=v1.2.3
curl https://app.tareas.bitcode.com.co/health/version
```

Tiempo objetivo: **<2 min**.

## 11. Ambiente PlatformAdmin

Deploy separado por seguridad (no comparte host con los tenants). Imagen
`bitcode/ecorex-platform`, host `admin.tareas.bitcode.com.co` con acceso
restringido (VPN/Cloudflare Access + MFA), doble aprobacion de release.

## 12. Secrets

**Nunca en repo, nunca en logs**:
- Dev/Staging: GitHub Secrets.
- Produccion: Azure Key Vault + Managed Identity.
- Rotacion: API keys c/6 meses, passwords de BD c/3 meses.

Este es el reemplazo directo del `Config.xml` con key embebida del origen (ver
[[CargaConfig - Decode Config.xml]]) — el secreto ya no vive en el codigo.

## 13. Chequeo pre-deploy

- [ ] Tests pasan en AMBOS motores (Postgres y SQL Server)
- [ ] Test de aislamiento cross-tenant pasa
- [ ] Coverage no bajo del umbral
- [ ] Migraciones aplicables sin dataloss (dry-run)
- [ ] Changelog y documentacion actualizados
- [ ] Feature flags para features riesgosas
- [ ] Alertas post-deploy configuradas

## 14. Enlaces

- [[Deploy a Produccion - Docker e hibrido]] — infra y reparto de servicios
- [[Backup y DR por proveedor]] — recovery si algo sale mal
- [[Plan de validacion del sistema]] — pruebas que corren aqui
- [[Gestion de Empresas - Admin multi-tenant]] — planes y feature flags
