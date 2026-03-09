# Technical Specifications — Architecture Decision Records

> Portable ADR-like specs derived from a production distributed system.  
> Not tied to any single implementation. Use as a guide for structuring similar projects.

---

## Table of Contents

1. [Structural Map](#1-structural-map)
2. [ADR-001: Solution & Project Organization](#adr-001-solution--project-organization)
3. [ADR-002: Layered Architecture & Dependency Direction](#adr-002-layered-architecture--dependency-direction)
4. [ADR-003: Domain Layer Design](#adr-003-domain-layer-design)
5. [ADR-004: Data Access Strategy](#adr-004-data-access-strategy)
6. [ADR-005: Database Versioning via Migrations](#adr-005-database-versioning-via-migrations)
7. [ADR-006: Application Layer & Service Contracts](#adr-006-application-layer--service-contracts)
8. [ADR-007: Worker Services & Feature Organization](#adr-007-worker-services--feature-organization)
9. [ADR-008: Web API Design](#adr-008-web-api-design)
10. [ADR-009: Messaging & Async Communication](#adr-009-messaging--async-communication)
11. [ADR-010: Saga Orchestration](#adr-010-saga-orchestration)
12. [ADR-011: External API Client Pattern](#adr-011-external-api-client-pattern)
13. [ADR-012: Shared Libraries & Cross-Cutting Concerns](#adr-012-shared-libraries--cross-cutting-concerns)
14. [ADR-013: Dependency Injection Strategy](#adr-013-dependency-injection-strategy)
15. [ADR-014: Observability & Structured Logging](#adr-014-observability--structured-logging)
16. [ADR-015: Containerization & Deployment](#adr-015-containerization--deployment)
17. [ADR-016: Infrastructure as Code](#adr-016-infrastructure-as-code)
18. [ADR-017: Caching & Distributed Locking](#adr-017-caching--distributed-locking)
19. [ADR-018: Authentication & Authorization](#adr-018-authentication--authorization)
20. [ADR-019: Testing Strategy](#adr-019-testing-strategy)
21. [ADR-020: Feature Flags](#adr-020-feature-flags)
22. [ADR-021: Frontend Architecture](#adr-021-frontend-architecture)
23. [ADR-022: Background Job Scheduling](#adr-022-background-job-scheduling)
24. [ADR-023: Extending the System](#adr-023-extending-the-system)

---

## 1. Structural Map

```
Solution Root
├── Domain Projects          (pure models, zero external deps)
│   ├── Domain/              Core entities, enums, constants, queue contracts
│   ├── Domain_{Context}/    Context-specific entities (e.g., Processor, Reporting)
│   └── Messages/            Shared message interfaces
│
├── Data Projects            (persistence, depends on Domain only)
│   ├── Data/                Main DbContext + migrations
│   ├── Data_{Context}/      Context-specific DbContext, repos, Dapper, migrations
│   └── Data_Users/          Identity DbContext
│
├── Application Projects     (use cases, depends on Domain only)
│   ├── Application/         Interfaces, DTOs, service contracts, validations
│   └── Application_{Context}/  Context-specific queries
│
├── Infrastructure Projects  (external integrations, depends on Application)
│   ├── {ExternalApi}/       HTTP client wrappers (one per vendor)
│   ├── CloudStorage/        Blob storage abstraction
│   ├── Redis/               Distributed cache abstraction
│   └── HangFire/            Background job infrastructure
│
├── Shared Projects          (DI wiring hub, cross-cutting)
│   ├── Shared/              Startup extensions, DI registration, config
│   ├── Shared.Auth/         Authentication middleware & providers
│   ├── CustomLogging/       Centralized log handlers
│   └── RimDevFeatureFlags/  Feature flag provider
│
├── Service Projects         (deployable units)
│   ├── {Service}Service/    Worker services (queue consumers)
│   ├── WebApi/              REST API (trigger/command entry point)
│   ├── {Name}BFF/           Backend-for-Frontend (UI-facing API)
│   └── HangFire.UI/        Job dashboard
│
├── Saga Projects
│   └── {Project}.Saga/      Saga definitions & orchestration
│
├── Test Projects            (mirror source project names)
│   ├── {Project}Tests/      Per-project test suites
│   └── Tests.Shared/        Shared test utilities, base classes, mocks
│
├── Frontend
│   └── UI/{app}/            SPA (React/Vite/TypeScript)
│
├── IaC
│   └── Pulumi/              Infrastructure-as-code stacks
│
└── Infra Config
    ├── docker-compose.yml          Network & volume definitions
    ├── docker-compose.*.yml        Layered compose files
    ├── RabbitMQ/                   Broker config + Dockerfile
    ├── Redis/                      Cache config + Dockerfile
    └── Seq/                        Log aggregator config + Dockerfile
```

### Dependency Flow (top = no deps, bottom = depends on everything above)

```
          ┌─────────────────┐
          │     Domain      │  ← Zero dependencies
          └────────┬────────┘
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
   ┌───────┐  ┌────────┐  ┌───────┐
   │ Data  │  │  App   │  │ Saga  │  ← Depend on Domain only
   └───┬───┘  └────┬───┘  └───┬───┘
       │           │           │
       └─────┬─────┘───────────┘
             ▼
   ┌─────────────────┐
   │ Infrastructure  │  ← External API clients, storage, cache
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │     Shared      │  ← DI hub, wires everything
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │    Services     │  ← Deployable entry points
   └─────────────────┘
```

---

## ADR-001: Solution & Project Organization

**Status:** Accepted

### Context
Large distributed systems need a project structure that scales with the team and domain complexity. Monorepo vs polyrepo, flat vs grouped — each trade-off impacts discoverability and build times.

### Decision
- **Monorepo with multiple .NET projects** in a single solution, grouped by architectural layer via solution folders.
- **One project per bounded context per layer**: e.g., `Domain_Service`, `Data_Service`, `Application_Reporting`.
- **Naming convention**: `{Layer}_{Context}` for context-specific projects; plain `{Layer}` for shared/core.
- **Test projects mirror source**: `{ProjectName}Tests` co-located at solution root.
- **Solution folders** group projects by role: Core, Infrastructure, Services, Tests, Web, Shared, Deploy.

### Consequences
- All code in one repo — single CI pipeline, atomic cross-project changes.
- Clear boundaries: adding a new bounded context = add `Domain_X`, `Data_X`, `Application_X`.
- Build times scale linearly; mitigate with `dotnet build --no-dependencies` for leaf projects.

### How to Extend
1. New bounded context → create `Domain_{Name}`, `Data_{Name}`, optionally `Application_{Name}`.
2. New service → create `{Name}Service/` project (Worker SDK or Web SDK), reference Shared for DI.
3. New external integration → create `{VendorName}/` class library project.

---

## ADR-002: Layered Architecture & Dependency Direction

**Status:** Accepted

### Context
Need clear separation of concerns that prevents domain logic from coupling to infrastructure.

### Decision
Enforce **strict inward dependency direction** following Clean Architecture principles:

| Layer | May Reference | Must NOT Reference |
|-------|--------------|-------------------|
| Domain | Nothing (pure) | Data, Application, Services |
| Data | Domain | Application, Services |
| Application | Domain | Data (implementations), Services |
| Infrastructure | Application (interfaces) | Domain internals |
| Shared | All lower layers | Services |
| Services | Shared, Application | Other Services directly |

- Domain projects carry **zero NuGet dependencies** beyond abstractions (e.g., Mediator.Abstractions).
- Data depends on Domain for entity types. Never on Application.
- Application defines interfaces; Data and Infrastructure implement them.
- Shared acts as the **composition root helper** — wires DI registrations.
- Services are leaf nodes — they compose everything via Shared extensions.

### Consequences
- Domain changes don't ripple to infrastructure.
- Swap database, message broker, or cloud provider without touching domain or application.
- Shared project grows; split into `Shared.{Concern}` if it becomes unwieldy.

---

## ADR-003: Domain Layer Design

**Status:** Accepted

### Context
Domain entities need consistent auditing, tracking, and lifecycle management across bounded contexts.

### Decision
- **Anemic domain model**: Entities are data containers with relationships. Business logic lives in Application/Service layer.
- **Base classes** for cross-cutting entity concerns:
  - `AuditableEntity` — `CreatedAt`, `UpdatedAt`, `CreatedBy`, `UpdatedBy`
  - `ProcessedEntity` — extends auditable with Processing-specific timestamps
- **Interfaces** for optional behaviors: `IAuditableEntity`, `IProcessededEntity`.
- **Enums and Constants** co-located in Domain: `Domain/Enums/`, `Domain/Constants/`.
- **Queue message contracts** defined in Domain: `IMessage`, `BaseMessage` with `Priority` enum.
- **Naming**: Context-specific entity prefixes where disambiguation is needed (e.g., `Acc` for account/advertising entities).

### Consequences
- Uniform audit trail across all entities via base class inheritance.
- Message contracts are shared across services without pulling service dependencies.
- Anemic model trades domain richness for simplicity — acceptable when business logic is coordination-heavy rather than rule-heavy.

### Best Practice Override
Rich domain models (DDD tactical patterns) are preferable when complex invariants exist. For integration/orchestration-heavy systems, anemic + service layer is pragmatic.

---

## ADR-004: Data Access Strategy

**Status:** Accepted

### Context
Different query patterns need different tools. ORM for CRUD, raw SQL for analytics and complex joins.

### Decision
- **Hybrid approach**: EF Core for CRUD/migrations + Dapper for complex queries.
- **Multi-database**: Each bounded context owns its database and DbContext.
  - Main domain → SQL Server, EF Core
  - Processor data → SQL Server, EF Core (pooled) + Dapper
  - Reporting/analytics → PostgreSQL, Dapper only
  - Identity → SQL Server, ASP.NET Core Identity
- **Repository pattern**: Interface-first (`IXxxRepository`), implementations in Data project.
- **DbContext pooling** for high-throughput contexts with `AddDbContextPool`.
- **Retry policies** on DbContext: `EnableRetryOnFailure(maxRetryCount, maxRetryDelay)`.
- **Dapper contexts**: Thin wrappers (`IDapperXxxContext`) providing `IDbConnection` for raw queries.
- **Custom models**: `CustomModels/` folder for Dapper result-set DTOs that don't map to entities.
- **Command timeout**: Generous for long-running ETL queries (configurable per context).

### Folder Convention
```
Data_{Context}/
├── Context/
│   ├── {Name}DbContext.cs          EF Core DbContext (partial if large)
│   ├── {Name}DbContextExtensions.cs  Fluent config extensions
│   └── Dapper{Name}Context.cs      Dapper IDbConnection wrapper
├── Repositories/
│   ├── BaseRepository.cs           Shared error handling
│   ├── I{Entity}Repository.cs      Interface
│   └── {Entity}Repository.cs       Implementation
├── CustomModels/                   Dapper-specific DTOs
└── Migrations/                     EF Core migrations
```

### Consequences
- EF Core handles schema evolution and simple CRUD; Dapper handles performance-critical reads.
- Multi-database adds operational complexity but enforces bounded context isolation.
- Repository interfaces in Application layer → Data layer implements → testable via mocks.

---

## ADR-005: Database Versioning via Migrations

**Status:** Accepted

### Context
Schema changes must be versioned, reproducible, and deployable without manual intervention.

### Decision
- **EF Core Migrations** for all SQL Server databases.
- **One migration folder per DbContext**: `Data_{Context}/Migrations/`.
- **Naming convention**: `YYYYMMDDHHmmss_DescriptiveTitle.cs` (auto-generated timestamp + human-readable name).
- **Model snapshots**: Auto-maintained `{Context}ModelSnapshot.cs` tracks current schema state.
- **Migration application**: At startup or via CLI — never manually.
- **No down migrations in production**: Forward-only. Rollback = new migration that reverts.
- **Dapper-backed databases** (e.g., PostgreSQL reporting): Schema managed externally or via migration scripts outside EF.

### Commands
```bash
# Add migration
dotnet ef migrations add DescriptiveTitle -p Data_{Context} -s {StartupProject}

# Apply
dotnet ef database update -p Data_{Context} -s {StartupProject}
```

### Consequences
- Schema is version-controlled alongside code.
- CI/CD can validate migrations compile and apply to a test database.
- Large DbContexts (300+ entities) produce large snapshot files — acceptable trade-off.

---

## ADR-006: Application Layer & Service Contracts

**Status:** Accepted

### Context
Need a layer that defines what the system can do without specifying how.

### Decision
- **Application project** defines:
  - **Interfaces**: Grouped by business domain in `Interfaces/{Domain}/` (e.g., `ISpCampaigns`, `ISdTargeting`).
  - **DTOs**: Grouped by domain in `Dtos/{Domain}/`. Suffix: `*Dto`.
  - **Validations**: Request validators (Mediator pipeline behaviors).
  - **Infrastructure interfaces**: `IMailService`, `IDapperXxxContext`, `IRedisService`, `IStorageClient`.
- **No implementations** in Application — only contracts.
- **Mediator abstractions** for message dispatch (Mediator.Abstractions NuGet).
- **Resilience patterns** defined here (Polly policies for outbound calls).

### Consequences
- Services program against interfaces — implementations are swappable.
- DTOs isolate domain entities from API responses — entity shape changes don't break consumers.
- Validation runs in Mediator pipeline before handlers — fail-fast on invalid requests.

---

## ADR-007: Worker Services & Feature Organization

**Status:** Accepted

### Context
Background services process queued work. Need consistent structure across services for onboarding and maintenance.

### Decision
Every worker service follows this folder structure:

```
{Name}Service/
├── Program.cs              Composition root (DI, config, logging)
├── Worker.cs               BackgroundService — sets up queue subscribers
├── appsettings.json        Base configuration
├── appsettings.local.json  Local overrides (git-ignored)
├── Dockerfile              Multi-stage build
├── Features/               Business logic, grouped by domain
│   └── {Feature}/
│       ├── {Feature}Handler.cs          IRequestHandler<TMessage>
│       ├── {Feature}Validator.cs        IPipelineBehavior (optional)
│       ├── Services/
│       │   ├── I{Feature}.cs            Interface
│       │   └── {Feature}.cs             Implementation
│       └── Dtos/                        Feature-specific DTOs
├── Extensions/             DI extension methods
├── Interfaces/             Service-level interfaces
├── Middleware/              Service-specific middleware
└── {Name}Service.csproj
```

### Worker Lifecycle
1. `Program.cs` → configure host, register DI via Shared startup extensions.
2. `Worker.ExecuteAsync()` → call `queueReceiverService.SetupSubscriberAsync<TMessage>(queueName, options)`.
3. Message arrives → Mediator dispatches to `IRequestHandler<TMessage>`.
4. Handler orchestrates business logic via injected services.

### Queue Subscriber Options
- `ConsumerCount`: Parallel consumers (1-100).
- `SingleActiveConsumer`: Mutual exclusion — one worker processes queue at a time.
- `Deduplication`: Idempotency via Redis key tracking.

### Consequences
- Feature folder = unit of cohesion. Adding a feature = add a folder.
- Handlers are thin orchestrators; logic lives in services.
- Queue subscription is declarative; feature flags gate which subscriptions are active.

---

## ADR-008: Web API Design

**Status:** Accepted

### Context
APIs serve as entry points for on-demand operations (human-triggered) vs queue-triggered background work.

### Decision
- **Two API patterns**:
  - **Internal API** (`WebApi`): REST controllers, API key auth, primarily queues messages for async processing.
  - **BFF** (`{Name}BFF`): Backend-for-frontend, JWT auth, Mediator for request handling, CORS for SPA.
- **Controllers**: `[ApiController]` + route attributes. Thin — validate input, queue message, return.
- **No business logic in controllers**: Delegate to Mediator or queue sender.
- **Swagger/OpenAPI**: Auto-generated with enum string serialization.
- **Reverse proxy**: Traefik for TLS termination, routing, and domain-based service resolution.
- **Health checks**: Recommended for all dependencies (DB, Redis, RabbitMQ). Implement via `AddHealthChecks()` + `MapHealthChecks()`.

### Consequences
- BFF shields frontend from internal service topology.
- Internal API is for service-to-service or admin operations.
- Thin controllers are testable and don't accumulate logic.

---

## ADR-009: Messaging & Async Communication

**Status:** Accepted

### Context
Services must communicate asynchronously to decouple processing and enable horizontal scaling.

### Decision
- **Broker**: RabbitMQ with custom abstraction layer (not MassTransit for transport — custom `IQueueSenderService` / `IQueueReceiverService`).
- **Message contracts**: Defined in Domain layer (`IMessage`, `BaseMessage`).
- **Priority queues**: Supported (max 10 levels, recommend 1-5).
- **Persistent messages**: Default — survive broker restart.
- **Delayed messages**: `x-delayed-message` exchange plugin.
- **Deduplication**: `x-deduplication-header` + Redis tracking for idempotent processing.
- **Dead-letter exchange**: Failed messages routed to DLX for inspection/retry.
- **Serialization**: `System.Text.Json` with UTF-8 encoding.
- **Queue naming**: Defined as constants in `Domain/Constants/Queues.cs`.

### Message Flow
```
Producer (API/Service)
  → IQueueSenderService.SendAsync(message, queueName, priority?)
  → RabbitMQ Exchange → Queue
  → Consumer (Worker)
  → Mediator.Send(message)
  → IRequestHandler<TMessage>
```

### Consequences
- Custom abstraction over RabbitMQ = full control over features (priority, dedup, delay) but maintenance burden.
- Message contracts in Domain = all services share schema without coupling.
- DLX provides safety net — failed messages don't vanish.

---

## ADR-010: Saga Orchestration

**Status:** Accepted

### Context
Multi-step distributed workflows (e.g., create asset → create creative → check moderation) need coordination with compensation on failure.

### Decision
- **Custom saga orchestrator** — not using MassTransit sagas.
- **Components**:
  - `SagaOrchestrator` — starts and continues sagas.
  - `Saga<TData>` — state machine definition with steps.
  - `SagaStep` — individual step with forward action and compensation.
  - `SagaStepCompletedMessage` — triggers next step via queue.
- **Persistence**: Saga state stored in SQL Server (`SagaStates` table) — `SagaId`, `Status`, `SerializedData`, `CreatedAt`, `UpdatedAt`.
- **Registration**:
  ```
  services.RegisterSagaFactory()          → Singleton factory
  services.AddSqlServerSagaPersistence()  → Persistence provider
  services.AddSaga<TSagaDefinition>()     → Per-saga registration
  ```

### Consequences
- Full control over saga lifecycle and compensation logic.
- Database-backed state = survives service restarts.
- Adding a saga = define `SagaDefinition<TData>`, register in DI, wire completion handler.

---

## ADR-011: External API Client Pattern

**Status:** Accepted

### Context
Multiple external APIs (advertising platforms, proxy services, data providers) need consistent integration patterns.

### Decision
- **One project per external vendor**: `{VendorName}/` class library.
- **HttpClientFactory** for all HTTP clients — named clients per region/purpose.
- **DelegatingHandler** for auth concerns (e.g., OAuth token refresh via `AuthenticationDelegatingHandler`).
- **Polly resilience**: Retry with jittered backoff, bulkhead isolation.
- **Resolver pattern**: `Func<string, IClient>` factory for region-based client resolution.
- **Discriminated union returns** (via OneOf): `OneOf<SuccessResponse, DelayedResponse, None>` for multi-outcome operations.
- **Response logging**: Truncated response body (first N chars) for debugging without log bloat.

### Folder Convention
```
{VendorName}/
├── {VendorName}Client.cs          Main client implementation
├── I{VendorName}Client.cs         Interface
├── Models/                        Request/response models
├── Constants/                     API endpoints, config keys
├── Middleware/                    DelegatingHandlers
├── Exceptions/                   Vendor-specific exceptions
└── {VendorName}.csproj
```

### Consequences
- Vendor swap = replace one project, implement same interface.
- Auth concerns are transparent to business logic (delegating handler).
- Circuit breaking prevents cascade failures from downstream outages.

---

## ADR-012: Shared Libraries & Cross-Cutting Concerns

**Status:** Accepted

### Context
DI registration, configuration binding, and cross-cutting concerns (logging, caching, auth) need a central orchestration point.

### Decision
- **Shared project** acts as **DI composition hub** with startup extension methods:
  ```
  Shared/Startup/
  ├── Databases.cs       → ConfigureDatabases() — all DbContexts
  ├── Logging.cs         → ConfigureAppLogging() — Seq, sinks
  ├── Cache.cs           → ConfigureCache() — Redis, distributed cache
  ├── AmazonAds.cs       → ConfigureAmazonAdsClient() — HTTP clients
  └── ...
  ```
- Each extension method is self-contained: reads config section, registers services.
- **Shared.Authentication** — separate project for auth middleware (JWT, API Key, external providers).
- **CustomLogging** — Mediator handlers for structured log persistence.
- Services call these extensions in `Program.cs`:
  ```csharp
  services.ConfigureProcessorDatabases(config);
  services.ConfigureCache(config);
  services.ConfigureAppLogging(config);
  ```

### Consequences
- Services have minimal `Program.cs` — most wiring is reusable.
- Adding a new infrastructure concern = add extension method in Shared, call from services that need it.
- Shared grows over time — split into `Shared.{Concern}` if >15 files.

---

## ADR-013: Dependency Injection Strategy

**Status:** Accepted

### Context
Consistent service lifetimes prevent subtle bugs (captive dependencies, thread-safety issues).

### Decision

| Lifetime | Use For | Examples |
|----------|---------|---------|
| **Singleton** | Stateless infrastructure, config, connection pools | `IConnectionMultiplexer`, `HttpClientFactory`, config objects |
| **Scoped** | Per-request/per-message state, ORM contexts | `DbContext`, repositories, `IRequestHandler`, `IFeatureManagerSnapshot` |
| **Transient** | Stateless, lightweight, per-call creation | Factories, `IPipelineBehavior`, HTTP message handlers |

### Rules
- **Never inject Scoped into Singleton** — causes captive dependency.
- **DbContext is always Scoped** (or pooled-scoped via `AddDbContextPool`).
- **Repositories are Scoped** — share the DbContext lifetime.
- **Mediator handlers are Scoped** — one instance per message processing.
- **Validators (pipeline behaviors) are Transient** — stateless, run per request.

### Consequences
- Consistent lifetime rules across all services.
- `AddDbContextPool` improves throughput for high-volume contexts.

---

## ADR-014: Observability & Structured Logging

**Status:** Accepted

### Context
Distributed services need centralized, searchable, structured logs.

### Decision
- **Log aggregator**: Seq (self-hosted, structured log search).
- **Logger**: `ILogger<T>` (Microsoft.Extensions.Logging) — injected everywhere.
- **Structured logging**: Key-value properties on log events, not string interpolation.
- **Log levels**:
  - `Trace`: SQL queries (EF Core), verbose internals.
  - `Debug`: Detailed flow information.
  - `Information`: Business events (processing started, message processed).
  - `Warning`: Recoverable issues (retry triggered, cache miss).
  - `Error/Critical`: Failures requiring attention.
- **Correlation**: Message-based correlation via saga IDs and message headers.
- **Custom log persistence**: Domain-specific logs (e.g., processing logs) written to database via Mediator handlers.

### Best Practice Note
Consider adding OpenTelemetry for distributed tracing. Structured logging alone doesn't provide request-level tracing across services. Add `ActivitySource` and trace propagation headers for full observability.

### Consequences
- All services ship logs to Seq — single pane of glass.
- Structured properties enable filtering: `ServiceName=AuthorsService AND Level>=Warning`.
- Custom log tables provide domain-specific queryability beyond generic log search.

---

## ADR-015: Containerization & Deployment

**Status:** Accepted

### Context
Services must be deployable consistently across dev, staging, and production environments.

### Decision
- **Multi-stage Docker builds** for all services:
  1. `base` — runtime image (`aspnet:8.0` or `node:20-alpine`).
  2. `build` — SDK image, restore + build.
  3. `publish` — publish with trimming.
  4. `final` — copy published output to lean base image.
- **Docker Compose** — layered files for separation:
  - `docker-compose.yml` — networks, volumes (base).
  - `docker-compose.infrastructure.yml` — broker, cache, log aggregator.
  - `docker-compose.services.yml` — application services.
  - `docker-compose.dev.yml` — dev overrides (source mounts, debug ports).
- **Infrastructure services** persist data via named volumes.
- **Application services** are stateless — rebuild and replace on deploy.
- **Reverse proxy** (Traefik): TLS termination, domain-based routing, auto-cert via Let's Encrypt.
- **Environment variables** for all secrets and connection strings — never baked into images.
- **Health checks** on infrastructure containers (RabbitMQ, Redis).

### Production Deploy
```bash
# Infrastructure (preserves data)
docker compose -f docker-compose.yml -f docker-compose.infrastructure.yml up -d

# Application (rebuild with latest code)
docker compose -f docker-compose.yml -f docker-compose.services.yml up --build -d
```

### Consequences
- Dev/prod parity via identical images.
- Layered compose = deploy infrastructure independently of application code.
- Named volumes ensure data survives container recreation.

---

## ADR-016: Infrastructure as Code

**Status:** Accepted

### Context
Cloud infrastructure must be reproducible, version-controlled, and reviewable.

### Decision
- **Pulumi** with C# (same language as application code).
- **Stack-per-environment**: Separate stacks for dev, staging, production.
- **Resource definitions**:
  - Kubernetes cluster (managed — e.g., GKE, AKS, EKS).
  - Node pool configuration (machine type, disk size, count).
  - Container registry secrets for image pulls.
  - Service deployments as Kubernetes resources.
- **Outputs**: Kubeconfig, cluster endpoints, registry credentials.
- **Secret management**: Pulumi encrypted config for sensitive values.

### Consequences
- Infrastructure changes go through PR review like application code.
- Same language for IaC and application reduces context switching.
- Pulumi state tracks drift — `pulumi preview` shows planned changes before apply.

---

## ADR-017: Caching & Distributed Locking

**Status:** Accepted

### Context
High-throughput systems need caching for performance and distributed locks for coordination across service instances.

### Decision
- **Cache provider**: Redis via `StackExchange.Redis`.
- **Abstraction**: `IDistributedCache` (Microsoft) + custom `IRedisService` wrapper.
- **Connection**: `IConnectionMultiplexer` as Singleton.
- **Instance naming**: Prefixed keys to avoid collisions across environments.
- **Distributed locks**: RedLock algorithm via `RedLockNet` — used for:
  - Preventing duplicate message processing.
  - Coordinating multi-instance job execution.
- **Fallback**: `IDistributedMemoryCache` for local dev without Redis.
- **Deduplication pattern**: Redis key with TTL — check before processing, set on start, clear on complete.

### Consequences
- Redis is a single point of failure for cache — mitigate with Redis Sentinel or Cluster in production.
- Distributed locks prevent duplicate work but add latency — use only where idempotency matters.
- Fallback to memory cache enables local development without infrastructure.

---

## ADR-018: Authentication & Authorization

**Status:** Accepted

### Context
Multiple auth scenarios: SPA users, service-to-service, admin dashboards.

### Decision
- **Pluggable auth providers** via `Shared.Authentication`:
  - **JWT Bearer** — SPA authentication.
  - **API Key** — service-to-service and admin endpoints.
  - **External provider** (e.g., Clerk) — delegated identity management.
- **Identity storage**: ASP.NET Core Identity with custom `ApplicationUser` in dedicated DbContext.
- **Auth state caching**: Distributed cache for session/claim state across instances.
- **Dashboard auth**: Basic auth via reverse proxy for admin UIs (HangFire, Traefik).

### Consequences
- Swapping auth provider = implement new handler in Shared.Authentication, no service changes.
- API Key auth is simple but limited — adequate for internal services, not for public APIs.
- Dedicated identity database = independent scaling and security boundary.

---

## ADR-019: Testing Strategy

**Status:** Accepted

### Context
Tests must be reliable, fast, and maintainable across a large codebase.

### Decision
- **Framework**: xUnit (test runner) + FluentAssertions/Shouldly (assertions) + Moq (mocking) + Bogus (data generation).
- **Coverage**: Coverlet for code coverage reporting.
- **Test project naming**: `{SourceProject}Tests`.
- **Shared test utilities**: `Tests.Shared/` provides:
  - `MockedTest` base class with pre-configured common mocks.
  - Mock loggers, HTTP handlers, and service stubs.
- **DI-based fixtures**: `ServiceProviderFixture : IAsyncLifetime` — builds real DI container with test doubles swapped in.
- **Test doubles strategy**:
  - **Mock** external services (HTTP clients, message brokers).
  - **Real** in-memory or test DbContext for data layer tests.
  - **Fixture** provides scoped service resolution per test.

### Test Structure
```csharp
public class FeatureTests : IClassFixture<ServiceProviderFixture>
{
    public FeatureTests(ServiceProviderFixture fixture)
    {
        _scope = fixture.CreateServiceScope();
        _context = _scope.GetRequiredService<DbContext>();
    }

    [Fact]
    public async Task Operation_WhenCondition_ExpectedResult() { }
}
```

### Best Practice Note
Consider adding integration tests with Testcontainers for database and broker verification. Current approach (DI fixture with mocks) is fast but doesn't catch infrastructure integration issues.

### Consequences
- Tests are isolated — each test gets a fresh scope.
- Shared fixtures reduce boilerplate.
- Naming convention `When_Condition_ExpectedResult` improves test readability.

---

## ADR-020: Feature Flags

**Status:** Accepted

### Context
Need to enable/disable features at runtime without redeployment — especially for toggling external API providers.

### Decision
- **Provider**: SQL Server-backed feature flags (schema auto-created on startup).
- **Scoping**: `IFeatureManagerSnapshot` — scoped per request, consistent within a single operation.
- **Configuration binding**: `EnabledFeatures` config section → strongly-typed options.
- **Usage pattern**: Workers check flags **before** setting up queue subscribers — prevents message loss when a feature is disabled.
- **Flag types**: Boolean toggles and string selectors (e.g., `ReviewsSource: "ListingLeopard" | "Rainforest"`).

### Consequences
- Runtime toggles enable gradual rollout and quick rollback.
- SQL Server storage = flags are shared across all instances.
- Checking flags before subscriber setup (not inside handler) avoids consuming and dropping messages.

---

## ADR-021: Frontend Architecture

**Status:** Accepted

### Context
SPA frontend needs fast builds, type safety, and component-driven development.

### Decision
- **Framework**: React with TypeScript.
- **Build tool**: Vite (fast HMR, optimized builds).
- **Styling**: Tailwind CSS (utility-first).
- **Components**: Radix UI primitives (accessible, unstyled base components).
- **State management**: Zustand (lightweight, no boilerplate).
- **Data tables**: TanStack React Table.
- **Auth**: Clerk SDK (matches backend provider).
- **Testing**: Vitest + Testing Library.
- **Production serving**: Nginx (multi-stage Docker build: Node build → Nginx serve).
- **API communication**: Via BFF only — frontend never talks to internal services.

### Consequences
- Vite + TypeScript = fast dev loop with compile-time safety.
- Radix + Tailwind = accessible components without design system lock-in.
- Zustand over Redux = less boilerplate, adequate for most SPA state needs.

---

## ADR-022: Background Job Scheduling

**Status:** Accepted

### Context
Some work is time-triggered (cron) rather than event-triggered. Need reliable scheduled execution.

### Decision
- **Scheduler**: HangFire with SQL Server storage.
- **Separation**: `HangFire/` library project (config) + `HangFire.UI/` web project (dashboard).
- **Recurring jobs**: Registered in `Worker.StartAsync()` via `IRecurringJobStarter`.
- **Schema**: Dedicated schema per service (e.g., `authors`) to avoid table conflicts.
- **Serialization**: Newtonsoft.Json with CamelCase naming.
- **Tuning**: `QueuePollInterval`, `SlidingInvisibilityTimeout` configured per environment.

### Consequences
- HangFire dashboard provides visibility into job status and history.
- SQL Server storage = survives restarts, but adds DB load for polling.
- Dedicated schemas prevent cross-service job interference.

---

## ADR-023: Extending the System

**Status:** Accepted

### Context
Spec for how to add new capabilities without breaking existing conventions.

### Adding a New Bounded Context
1. Create `Domain_{Name}/` — entities, enums, constants.
2. Create `Data_{Name}/` — DbContext, repositories, migrations.
3. Optionally `Application_{Name}/` — if context-specific DTOs/interfaces are needed.
4. Add migration: `dotnet ef migrations add InitialCreate -p Data_{Name} -s {StartupProject}`.
5. Register DbContext in `Shared/Startup/Databases.cs`.

### Adding a New Service
1. Create `{Name}Service/` with Worker SDK template.
2. Add `Program.cs` → call Shared startup extensions.
3. Add `Worker.cs` → subscribe to queues.
4. Add `Features/{Feature}/` folders for each business capability.
5. Add `Dockerfile` following multi-stage pattern.
6. Add service to `docker-compose.services.yml`.
7. Add test project `{Name}ServiceTests/`.

### Adding a New Feature to Existing Service
1. Create `Features/{FeatureName}/` folder.
2. Add `{Feature}Handler.cs : IRequestHandler<TMessage>`.
3. Add `Services/I{Feature}.cs` + `{Feature}.cs`.
4. Optionally add `{Feature}Validator.cs` as pipeline behavior.
5. Define queue constant in `Domain/Constants/Queues.cs`.
6. Register subscriber in `Worker.cs`.
7. Add tests in `{Service}Tests/Features/{Feature}/`.

### Adding a New External Integration
1. Create `{VendorName}/` class library project.
2. Define `I{VendorName}Client` interface.
3. Implement with `HttpClientFactory` + Polly policies.
4. Add DI extension in `Shared/Startup/`.
5. Add test project `{VendorName}Tests/`.

### Adding a New Saga
1. Define `{Name}SagaDefinition : Saga<TData>` with steps and compensations.
2. Register: `services.AddSaga<{Name}SagaDefinition>()`.
3. Start from handler: `sagaOrchestrator.StartSagaAsync<TSaga, TData>(data)`.
4. Wire `SagaStepCompletedMessage` handler in the consuming service.

---

## Appendix: Best Practice Checklist

| Practice | Status | Notes |
|----------|--------|-------|
| Single Responsibility (projects) | ✅ | One project per concern per context |
| Open/Closed (features) | ✅ | New feature = new folder, no modification of existing |
| Liskov Substitution | ✅ | Interface-first, implementations swappable |
| Interface Segregation | ✅ | Granular interfaces per domain (ISpCampaigns, ISdTargeting) |
| Dependency Inversion | ✅ | Domain has zero deps, services depend on abstractions |
| DB versioning | ✅ | EF Core migrations, timestamped, per-context |
| Containerization | ✅ | Multi-stage Docker, layered compose |
| IaC | ✅ | Pulumi with same language as app |
| Structured logging | ✅ | Seq + ILogger, centralized |
| Distributed tracing | ⚠️ | **Gap**: No OpenTelemetry/trace propagation — add for full observability |
| Health checks | ⚠️ | **Gap**: Not implemented — add `AddHealthChecks()` for DB, Redis, RabbitMQ liveness/readiness |
| Resilience (Polly) | ✅ | Retry, bulkhead on HTTP clients and DB |
| Idempotency | ✅ | Redis dedup + message headers |
| Secret management | ⚠️ | Env vars adequate; consider vault integration for rotation |
| Integration tests | ⚠️ | **Gap**: No Testcontainers — mocked fixtures cover unit, not integration |
| API versioning | ⚠️ | **Gap**: No explicit API versioning strategy — add before public API exposure |
| Rate limiting | ⚠️ | **Gap**: No inbound rate limiting on APIs — add middleware for public endpoints |
