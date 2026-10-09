---
name: build-slice
description: Implement one acceptance criterion at a time in a .NET API (controllers, services, EF Core), writing the MSTest test first. Use when building a feature or fix from a spec, when dev-flow routes to the Build phase, or when asked to "implement AC2" or "build the next slice".
---

# build-slice

Part of the `dotnet-flow` suite (see `dev-flow`). Runs in the **Build** phase.
Its question: *what is the smallest change that makes the next acceptance criterion true?*

## When to use

- After `frame-task` (and any design skill) has produced `spec.md`.
- For a bug fix, after `diagnose` has produced a failing test.

## Reads

- `.work/<slug>/spec.md` for the acceptance criteria, `design.md` for contracts, routes, and
  data changes, and `decisions.md`.
- Conventions (via `dev-flow`).
- Nearby code: an existing controller, service, entity configuration, and test in the same area.
  New code should look like it was written by the same person.

## Slicing

A slice is one acceptance criterion end to end: request → controller → service → EF Core → response,
plus its test. Order the slices so the first one is the happy path and later ones add the
failure paths. Don't build layers horizontally (all DTOs, then all services); a half-built feature
should still pass its tests.

## Steps (per slice)

1. **Pick the next unchecked criterion** in `spec.md` and say which one you're building.
2. **Write the failing test first** using `dotnet-test`. Usually that's an API test through
   `WebApplicationFactory`; use a unit test for pure logic. Run it and confirm it fails for the
   right reason (404 or an assertion, not a compile error in unrelated code).
3. **Make it pass with the least code**, following the conventions:
   - Controller: thin, `[ApiController]`, attribute route, `[Authorize]` policy, `CancellationToken`,
     returns `ActionResult<TResponse>`.
   - Request and response DTOs as records, separate from entities. Validation lives where the
     repo already puts it.
   - Service: plain class registered in DI. Add an interface only when something needs to swap it.
   - EF Core: use the existing `DbContext`; configuration in an `IEntityTypeConfiguration<T>`.
     If the schema changes, hand off to `ef-migration` before continuing.
   - Errors map to `ProblemDetails` through the existing handler, not try/catch in the controller.
   - Outbound calls go through a typed `HttpClient` already designed in `design.md`.
4. **Run the tests**: `dotnet build` (warnings are errors), then `dotnet test` for the affected
   test project. Everything green, not just the new test.
5. **Tick the criterion** in `spec.md` and note the test name next to it.
6. **Stop and show the user** the slice's diff in a few lines: the files touched and the test that
   proves it. Continue to the next slice unless they want changes. Commit per slice if the user
   works that way (`<type>: <AC id> <summary>`).

## Rules

- Don't add anything the spec doesn't ask for: no speculative options, extra endpoints, or
  "while I'm here" refactors. Note ideas in `decisions.md` instead.
- Don't edit a test to make it pass unless the test was wrong; say so when it was.
- If a slice turns out bigger than expected (new table, new partner call), stop and say so; it
  probably needs the Design phase.

## Exit criteria

- Every acceptance criterion is ticked and names at least one passing test.
- `dotnet build` is clean and `dotnet test` passes for the solution.

## Writes

- Code and tests in the repo.
- Ticks and test names in `.work/<slug>/spec.md`; deviations in `decisions.md`.

## Next skill

`simplify-pass`, then `review-gate`.
