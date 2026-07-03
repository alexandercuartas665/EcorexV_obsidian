---
tipo: nota-testing
capa: transversal
proposito: Estrategia de pruebas del sistema DESTINO ECOREX Tareas (.NET 10) - piramide, matriz Testcontainers dual, aislamiento cross-tenant, ETL, E2E
estado: diseno objetivo de migracion
---

# Estrategia de Testing (.NET 10)

> Plan de pruebas del sistema DESTINO. Complementa el [[Plan de validacion del sistema]]
> (que es la validacion MANUAL del sistema ORIGEN legacy antes de migrar). Aqui se
> define como se prueba el codigo .NET 10 automaticamente. Punto no negociable:
> **toda prueba de integracion corre en AMBOS motores (PostgreSQL y SQL Server)** —
> el DAL es dual (ver [[00 - Visión MotherData]]).

## 1. Piramide

```
              +------------------+
              |   E2E (10%)      |  Playwright .NET contra Blazor
              |   ~40 tests      |
              +------------------+
         +--------------------------+
         |  Integration (30%)       |  Testcontainers Postgres + SQL Server
         |  ~400 tests              |
         +--------------------------+
   +-----------------------------------+
   |       Unit (60%)                  |  xUnit + FluentAssertions
   |       ~2500 tests                 |
   +-----------------------------------+
```

## 2. Cobertura objetivo

| Capa | Target |
|---|---|
| Domain (entidades, value objects, maquinas de estado de flujo) | 95% |
| Application (comandos, servicios, RulesEngine, WorkflowEngine) | 85% |
| Infrastructure (repositorios) | 75% (via integration) |
| Web (Blazor components) | 40% (bUnit para logica critica) |
| **GLOBAL** | **>= 75%** (la CI falla el PR bajo 70%) |

## 3. Matriz Testcontainers dual (OBLIGATORIA)

Toda prueba de integracion se ejecuta en Postgres Y SQL Server. Esto es lo que la
[[CICD Pipeline]] exige y esta nota concreta.

```csharp
public abstract class DualDbTestBase : IAsyncLifetime {
    public static IEnumerable<object[]> Providers => new[] {
        new object[] { "Postgres" },
        new object[] { "SqlServer" }
    };
    protected IEcorexDbContext Ctx = default!;
    private IContainer _container = default!;

    protected async Task InitAsync(string provider) {
        _container = provider switch {
            "Postgres"  => new PostgreSqlBuilder().WithImage("postgres:16-alpine").Build(),
            "SqlServer" => new MsSqlBuilder().WithImage("mcr.microsoft.com/mssql/server:2022-latest").Build(),
            _ => throw new ArgumentException(provider)
        };
        await _container.StartAsync();
        Ctx = ProviderFactory.Create(provider, _container.GetConnectionString());
        await ((DbContext)Ctx).Database.MigrateAsync();
    }
    public Task InitializeAsync() => Task.CompletedTask;
    public async Task DisposeAsync() => await _container.DisposeAsync();
}
```

## 4. El test mas importante: aislamiento cross-tenant

Es el criterio de aceptacion del multi-tenant real (ver
[[Gestion de Empresas - Admin multi-tenant]] seccion 3). **Debe fallar si alguien
rompe el `HasQueryFilter` o la RLS.** Corre en ambos motores.

```csharp
[Theory]
[MemberData(nameof(Providers))]
public async Task Tenant_A_no_ve_datos_de_Tenant_B(string provider) {
    await InitAsync(provider);
    var a = Guid.NewGuid(); var b = Guid.NewGuid();

    using (TenantScope.For(a)) { await Ctx.Tasks.AddAsync(new TaskItem { Title = "de A" }); await Ctx.SaveChangesAsync(); }
    using (TenantScope.For(b)) { await Ctx.Tasks.AddAsync(new TaskItem { Title = "de B" }); await Ctx.SaveChangesAsync(); }

    using (TenantScope.For(a)) {
        var visibles = await Ctx.Tasks.ToListAsync();
        visibles.Should().OnlyContain(t => t.Title == "de A");   // nunca ve "de B"
    }
    using (TenantScope.None()) {
        (await Ctx.Tasks.ToListAsync()).Should().BeEmpty();       // sin tenant, cero filas
    }
}
```

## 5. Categorias de pruebas

### 5.1 Unit
- Value objects, maquinas de estado (`TaskItem` transiciones, ciclo de vida).
- `WorkflowEngine.AdvanceAsync` con definiciones sinteticas (incluye deteccion de ciclo: un flujo mal armado NO debe superar el limite de iteraciones).
- `RulesEngine` evaluando verbos Ensamblado con contexto simulado.
- Handlers CQRS con mocks.

### 5.2 Integration (dual)
- Repositorios contra BD real (ambos motores).
- Aislamiento cross-tenant (seccion 4).
- `HasQueryFilter` + RLS activos.
- Migraciones se aplican limpias en Postgres y SQL Server.
- `DynamicFormRenderer`: guardar/leer respuestas con `jsonb` (Postgres) y `nvarchar(max)` (SQL Server) — mismo resultado.
- WorkflowEngine persistiendo `workflow_instance` / `workflow_step_history`.
- Concurrencia optimista en tarea/flujo (rowversion / xmin) lanza `DbUpdateConcurrencyException`.

### 5.3 ETL (critico de la migracion)
Ver [[HOJA DE RUTA DESARROLLO]] seccion 9. Pruebas dedicadas al migrador:
- **Round-trip**: un registro legacy migrado se lee identico en el destino.
- **Conteos**: filas por tabla/tenant coinciden origen vs destino.
- **Idempotencia**: correr el ETL dos veces no duplica.
- **Aislamiento sobre datos migrados**: un tenant migrado no ve otro.
- **EAV -> jsonb**: cada tipo de control de `FORX_DATA` se mapea sin perdida.

### 5.4 API
- WebApplicationFactory + Testcontainers.
- Auth flow (login -> refresh -> logout), verificacion de claims.
- Aislamiento por endpoint (tenant A no accede a recurso de B).
- Webhooks con firma valida/invalida.

### 5.5 Blazor (bUnit)
- Tablero Kanban renderiza y mueve tarjetas.
- `DynamicFormRenderer` muestra/oculta campos por reglas de visibilidad.
- Listener SignalR actualiza el tablero.

### 5.6 E2E (Playwright .NET)
- Recorrido: login -> crear tarea -> avanzar por flujo -> completar.
- Cambio de tenant (PlatformAdmin).
- Aislamiento visual (un usuario no ve datos de otro tenant).
- El aspecto coincide con [[00 - Prototipo Final ECOREX]] (smoke visual).

## 6. Fixtures y seeds

`TestData/` con `MinimalTenant` (1 usuario, 1 actividad, 1 flujo) y `RichTenant`
(SKY SYSTEM demo: usuarios, proyectos, flujos BPMN, formularios, reglas). Fixtures
de flujos BPMN validos e invalidos (con ciclo) para probar la deteccion.

## 7. CI matrix

`.github/workflows/pr-check.yml` corre `[Postgres, SqlServer] x [.NET 10]`. Regla de
merge: no entra PR si falla build, el **test de aislamiento cross-tenant**, la matriz
dual, los tests de ETL, auth o `dotnet format`. Tiempo objetivo CI: < 8 min.

## 8. Herramientas

- xUnit + FluentAssertions (unit/integration).
- Testcontainers .NET (Postgres + SQL Server).
- bUnit (Blazor).
- Playwright .NET (E2E, sin Node).
- Coverlet + Codecov (cobertura por proveedor).
- Stryker.NET mensual (mutation testing sobre Domain + Application, score >= 60%).
- `dotnet list package --vulnerable` + CodeQL (seguridad, cada PR).

## 9. Enlaces

- [[Plan de validacion del sistema]] — validacion MANUAL del ORIGEN (contraparte)
- [[00 - Registro de corridas]] — bitacora de corridas
- [[CICD Pipeline]] — donde corre esta matriz
- [[Gestion de Empresas - Admin multi-tenant]] — el aislamiento que se prueba
- [[Seguridad y Autenticacion multi-tenant]] — tests de seguridad
- [[HOJA DE RUTA DESARROLLO]] — pruebas del ETL
