# Suite conventions (defaults)

These are defaults. A repo's `docs/conventions.md` overrides them, and the repo's existing code
overrides both. Each rule has a short reason so it can be judged rather than followed blindly.

## Platform

- Target the framework in `global.json` or the `.csproj`. For new projects, use the current .NET LTS.
- `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`, and
  `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` in `Directory.Build.props`. Warnings ignored today become bugs later.
- Central package management (`Directory.Packages.props`) when a solution has more than one project.

## API (controllers)

- `[ApiController]` with attribute routing; routes are plural nouns: `api/v1/orders/{id}`.
- Controllers stay thin: bind, authorize, call a service, map the result. No EF or `HttpClient` in controllers.
- Request and response DTOs are separate from EF entities; never return entities.
- Errors are `ProblemDetails` (`AddProblemDetails()` plus an exception handler). Validation errors return 400 with `ValidationProblemDetails`.
- Status codes: 201 with a `Location` header on create, 204 on delete, 404 when a resource is missing, 409 on conflicts.
- Every action accepts a `CancellationToken` and passes it all the way down.
- Collections are paged (`page`/`pageSize` or a continuation token) with a maximum page size.
- Versioning uses the URL segment (`/v1/`) via `Asp.Versioning.Mvc`.
- Authorization is explicit on every controller (`[Authorize(Policy = ...)]`), and `[AllowAnonymous]` is the rare, commented exception.

## Data (EF Core)

- One `DbContext` per bounded area; configuration in `IEntityTypeConfiguration<T>` classes, not attributes.
- Reads use `AsNoTracking()` and project to DTOs with `Select`. No `ToList()` before filtering.
- Every query that can grow is paged or bounded.
- Watch for N+1 queries: use `Include` deliberately or project.
- Migrations are reviewed as SQL (`dotnet ef migrations script`) before they run anywhere shared. Breaking changes use expand and contract.
- Money uses `decimal` with an explicit precision; time uses `DateTimeOffset` in UTC.

## Azure

- Secrets come from Key Vault via Managed Identity (`DefaultAzureCredential`). Nothing secret in `appsettings*.json`.
- Settings bind to the Options pattern with `ValidateOnStart()`.
- Messaging uses Azure Service Bus. Publishing after a database save goes through an outbox, and consumers are idempotent (deduplicate by message ID).
- Observability: OpenTelemetry exporting to Application Insights. Use structured logging with message templates, never string interpolation.
- Health checks at `/health/live` and `/health/ready`, with dependency checks only on ready.

## Outbound integrations

- Typed clients via `IHttpClientFactory`. Never `new HttpClient()` per call.
- `AddStandardResilienceHandler()` (Microsoft.Extensions.Http.Resilience) with timeouts sized to the partner's SLA.
- Retries only on idempotent operations, or with an idempotency key.
- Partner models stay at the edge: map them to domain types in one place (anti-corruption layer).
- Log the partner name, operation, status, and duration for every call, and never log payloads that contain personal data or secrets.

## Testing (MSTest)

- MSTest (`MSTest` meta-package / MSTest.Sdk) with `[TestClass]`/`[TestMethod]`; `[DataRow]` for cases.
- Assertions use AwesomeAssertions (`result.Should().Be(...)`) for readable tests and failure messages.
- Name tests `Method_Scenario_ExpectedResult`.
- Unit tests for domain and service logic; no mocks of EF Core. Use real databases instead.
- API tests use `WebApplicationFactory<Program>` and a real `HttpClient`.
- Databases and Azure emulators run in Testcontainers (SQL Server, Azurite, Service Bus emulator), started once in `[AssemblyInitialize]`.
- Partner APIs are faked with WireMock.Net, including timeouts, 429s, 5xx responses, and malformed bodies.
- Each acceptance criterion in `spec.md` maps to at least one named test.

## Git and PRs

- Branch names: `<type>/<issue-or-key>-<short-name>`, for example `feat/PAY-311-refund-sync`.
- Small PRs; one acceptance criterion or slice per commit when practical.
- The PR description comes from `pr.md` (see `ship-pr`).
