---
name: dotnet-test
description: Write the right MSTest tests for .NET APIs and Azure integrations - unit, WebApplicationFactory API tests, Testcontainers for SQL Server and Azurite, WireMock.Net for partner APIs. Use when build-slice needs a failing test, when asked to add or fix tests, or to cover existing untested code.
---

# dotnet-test

Part of the `dotnet-flow` suite (see `dev-flow`). Used in **Build** (test first) and **Verify**
(coverage gaps). Its question: *what is the cheapest test that would catch this criterion breaking?*

## When to use

- `build-slice` needs the failing test for a criterion.
- "Add tests for X", "this test is flaky", or "what isn't covered?".
- Before refactoring untested code: write characterization tests first.

## Reads

- `.work/<slug>/spec.md` (criteria to cover) and `design.md` (failure modes for integrations).
- Existing test projects: framework version, base classes, fixtures, naming, how the app is booted.
  Reuse what exists before adding new infrastructure.
- Conventions (via `dev-flow`). Code patterns are in `references/patterns.md`.

## Choosing the test

| What's being checked | Test type | Tools |
|---|---|---|
| Pure logic: calculations, mapping, rules | Unit | MSTest only |
| Endpoint behavior: status, body, auth, validation | API | `WebApplicationFactory<Program>` + `HttpClient` |
| EF Core queries, constraints, migrations | Integration | Testcontainers SQL Server (never the in-memory provider) |
| Blob/queue/table storage | Integration | Testcontainers Azurite |
| Service Bus publish and consume | Integration | Service Bus emulator container, or a fake sender at the edge |
| Partner HTTP API behavior and failures | Integration | WireMock.Net |

Prefer API tests for acceptance criteria: they test what the caller sees and survive refactoring.
Use unit tests for logic with many cases (`[DataRow]`).

## Steps

1. Map each criterion to one test, named `Method_Scenario_ExpectedResult` (for API tests,
   `Post_Orders_WithMissingSku_Returns400`). Put the criterion ID in a comment or `[Description]`.
2. Check the infrastructure exists (factory, containers, WireMock server). If not, add it once at
   assembly level using the patterns reference, then reuse it.
3. Write the test with Arrange / Act / Assert. Assert on what the criterion says: the status code,
   the `ProblemDetails` fields, the persisted row, the outbound call WireMock received. Don't assert
   on internals.
4. Run it and confirm it fails for the expected reason before the code exists (test first),
   or passes against the existing code (characterization).
5. For integrations, cover the failure modes from `design.md`: timeout, 429 with `Retry-After`,
   5xx, malformed body, and duplicate delivery.

## Rules

- Tests are independent: no ordering, and each one creates the data it needs. Use unique keys
  or reset the database between tests (Respawn, or a transaction per test).
- No `Thread.Sleep`. Wait on a condition with a timeout instead.
- Don't mock `DbContext`, `HttpClient`, or `ILogger` unless there's no other way. Use the real thing
  at the edge (container, WireMock) and assert on effects.
- Secrets and connection strings in tests come from the containers, never from developer config.
- A flaky test is a bug. Find the cause (shared state, time, ordering) rather than retrying.

## Exit criteria

- Every acceptance criterion has a named test, and the suite passes with `dotnet test`.
- New test infrastructure is shared, not duplicated per class.

## Writes

- Test code in the repo's test projects.
- Test names next to each criterion in `.work/<slug>/spec.md`.

## Next skill

Back to `build-slice` for the implementation, or `review-gate` when used for coverage in Verify.
