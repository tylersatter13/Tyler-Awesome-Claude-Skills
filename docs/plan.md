# .NET API & Integration Skill Suite: Proposed Plan

Status: **draft for Tyler's review**. No skills have been created yet.

## 1. The idea in one paragraph

Each skill does one phase of the development loop well, and all of them share the same
**process**, **work folder**, and **conventions file**. Skills hand off to each other through files,
not memory: one skill writes a short artifact, and the next skill reads it. A small hub skill
(`dev-flow`) knows the whole process, tells you where you are, and points to the next skill.
That keeps every skill small and focused while making the set behave like one system.

## 2. The overarching process

```
 Frame ──► Design ──► Build ──► Verify ──► Ship ──► Learn
   │         │          │         │          │        │
 spec.md  design.md  code+tests  review.md  PR text  conventions.md updates
```

| Phase | Question it answers | Exit criteria (gate) |
|---|---|---|
| **Frame** | What are we building and how will we know it works? | Acceptance criteria written as testable statements; open questions listed |
| **Design** | What does the contract look like? | API contract (OpenAPI/endpoint table) or integration design agreed; decisions logged |
| **Build** | Implement in thin vertical slices, test first | Each acceptance criterion has a passing test; builds clean with warnings-as-errors |
| **Verify** | Is it correct, secure, observable, and fast enough? | Review checklist passes or deviations are recorded |
| **Ship** | Is it ready for someone else to review and deploy? | PR description, migration and config notes, rollback plan |
| **Learn** | What should future work do differently? | New rules added to the conventions file |

Every phase can be skipped with a one-line reason (for example, "trivial fix, no design needed").
Recording that reason is the rule; following every phase is not.

### Shared spine (what makes the skills work together)

1. **Work folder**: `.work/<ticket-or-slug>/` in the repo (gitignored or committed, your choice).
   Holds `spec.md`, `design.md`, `decisions.md`, `review.md`, `pr.md`. Every skill starts by
   reading whatever already exists there.
2. **Conventions file**: `docs/conventions.md` (or a shared reference inside the hub skill).
   Contains your stack choices: .NET version, minimal APIs vs controllers, EF Core vs Dapper,
   test libraries, error format, logging. Every skill loads it, so the skills never disagree.
3. **Same skill shape**: each SKILL.md has the same sections: *When to use*, *Reads*,
   *Steps*, *Exit criteria*, *Writes*, *Next skill*. Easy to learn once and extend later.
4. **Decision log**: any non-obvious choice is added to `decisions.md` as one line
   (date, decision, why). Later phases and PR descriptions draw from it.

## 3. The skills

### Core loop (build these first)

| # | Skill | Phase | What it does |
|---|---|---|---|
| 1 | **`dev-flow`** (hub) | All | Explains the process, creates the work folder, detects which phase you're in from existing files, and names the next skill. Holds the shared templates and conventions reference. |
| 2 | **`frame-task`** | Frame | Turns a ticket, Slack thread, or rough idea into `spec.md`: problem, scope and non-scope, acceptance criteria in Given/When/Then, and open questions. Asks clarifying questions before writing code. |
| 3 | **`api-design`** | Design | Designs HTTP contracts: resources and routes, request and response DTOs, status codes, `ProblemDetails` errors, pagination, filtering, versioning, auth scopes, and idempotency keys for POSTs. Outputs an endpoint table plus an OpenAPI snippet. |
| 4 | **`integration-design`** | Design | For calls to or from external systems (REST/SOAP partners, queues, webhooks, file drops). Covers auth (OAuth client credentials, API keys, certs), resilience (`Microsoft.Extensions.Http.Resilience`/Polly: timeouts, retries, circuit breakers), idempotency, outbox/inbox, mapping and anti-corruption layer, rate limits, and a failure-mode table ("partner is down / slow / returns garbage"). |
| 5 | **`build-slice`** | Build | Implements one acceptance criterion at a time, test first: endpoint, handler or service, validation, persistence, DI registration. Follows the conventions file for structure. Stops after each slice so you can review. |
| 6 | **`dotnet-test`** | Build / Verify | Writes the right kind of test for the job: MSTest unit tests, `WebApplicationFactory` API tests, Testcontainers for real databases and brokers, WireMock.Net for faking partner APIs, and contract tests for integrations. Also handles "add tests to this existing code". |
| 7a | **`simplify-pass`** | Verify | Looks over the task's diff for unnecessary complexity: single-implementation interfaces, generic repositories over EF Core, pass-through layers, hand-rolled versions of framework features, deep nesting, and dead code. Applies changes one at a time with tests kept green. Added at Tyler's request. |
| 7 | **`review-gate`** | Verify | A .NET-specific self-review before a human sees it: async misuse (`.Result`, missing `CancellationToken`), `HttpClient` lifetime, nullable warnings, EF query issues (N+1, tracking, unbounded queries), secrets in config, input validation, authZ on every endpoint, structured logging and correlation IDs, and health checks. Writes `review.md`. |
| 8 | **`ship-pr`** | Ship | Writes the PR description from `spec.md`, `decisions.md`, and `review.md`: before and after, how to test, migrations, new config or secrets, feature flags, and rollback steps. |

### Supporting skills (add once the core loop feels right)

| Skill | When | What it does |
|---|---|---|
| **`diagnose`** | A bug or incident | Reproduce, then read logs, traces, and exceptions, and form a hypothesis. Write a failing test first, then hand off to `build-slice` for the fix. A bug report enters the loop at Build with a failing test as its spec. |
| **`learn`** | End of a task | Reviews what went wrong or slowly and proposes edits to the conventions file or to a skill. This is how the suite improves over time. |
| **`ef-migration`** (promoted to core, see §6) | Schema changes | Safe EF Core migrations: expand and contract, data backfills, idempotent scripts, and how to roll back. |
| **`scaffold-service`** | New API or worker | Creates a new solution with your standard layout, `Directory.Build.props`, analyzers, health checks, OpenTelemetry, auth, and test projects already wired. |

## 4. How a typical task flows

**New partner integration:** `dev-flow` → `frame-task` → `integration-design` (+ `api-design`
if you expose a webhook) → `build-slice` ×N with `dotnet-test` → `simplify-pass` → `review-gate` → `ship-pr` → `learn`.

**Small endpoint change:** `dev-flow` → `frame-task` (two-line spec) → skip Design with a recorded
reason → `build-slice` → `simplify-pass` → `review-gate` → `ship-pr`.

**Production bug:** `diagnose` → `build-slice` → `review-gate` → `ship-pr`.

## 5. Suggested build order

1. `dev-flow` + conventions file + templates (the spine, so everything else plugs in)
2. `frame-task`, `build-slice`, `dotnet-test` (smallest useful loop)
3. `api-design`, `integration-design`
4. `review-gate`, `ship-pr`
5. Supporting skills as needed

Each skill would be built, tried on a real task of yours, and then adjusted before the next one.

## 6. Decisions so far (from Tyler, 2026-10-09)

| Topic | Decision | Effect on the suite |
|---|---|---|
| Stack | .NET with **controllers** and **EF Core** | `build-slice` scaffolds controllers + services + EF Core; `review-gate` checks EF query patterns; `ef-migration` moves up to the core set |
| Cloud | **Azure** for everything | `integration-design` defaults to Service Bus, Key Vault + Managed Identity, App Configuration, Application Insights/OpenTelemetry; Testcontainers uses Azurite and the Service Bus emulator |
| Task sources | **GitHub Issues** (personal), **Jira** (work) | `frame-task` reads an issue via `gh issue view` or a Jira key via the Atlassian connector, and detects which from the input |
| Where skills run | **Claude Code inside repos** | Skills install once as personal skills (`~/.claude/skills/`, or packaged as a plugin from one Git repo) so they work in every repo; each repo keeps its own `docs/conventions.md` for local overrides |

### Conventions defaults (draft)

- .NET version: assume the current LTS (.NET 10) unless the repo's `global.json` or target framework says otherwise; skills always read the repo first.
- Web: controllers with `[ApiController]`, `ProblemDetails` for every error, FluentValidation or DataAnnotations (whichever the repo already uses).
- Data: EF Core, `AsNoTracking` for reads, no unbounded queries, migrations reviewed before apply.
- Config and secrets: Options pattern, Key Vault via Managed Identity, nothing secret in `appsettings.json`.
- HTTP clients: typed `HttpClient` via `IHttpClientFactory` with `Microsoft.Extensions.Http.Resilience`.
- Messaging: Azure Service Bus, with an outbox for publish-after-save.
- Observability: OpenTelemetry to Application Insights, correlation IDs on every inbound and outbound call.
- Tests: MSTest, `WebApplicationFactory`, Testcontainers (SQL Server, Azurite), WireMock.Net for partners.

## 7. Resolved questions

1. **Test framework**: MSTest (decided).
2. **Work folder**: `.work/` is gitignored (decided).
3. **Home for the skills**: the GitHub repo `tylersatter13/Tyler-Awesome-Claude-Skills` packaged as a Claude Code plugin, so you install once and update everywhere. (decided).
