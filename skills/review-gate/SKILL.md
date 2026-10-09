---
name: review-gate
description: Run a .NET-specific self-review of the current change before a human sees it - correctness against the spec, async and cancellation, EF Core queries, validation, authorization, secrets, HttpClient and resilience, logging, and schema safety. Use after simplify-pass, before opening a PR, or when asked to review a .NET diff.
---

# review-gate

Part of the `dotnet-flow` suite (see `dev-flow`). Runs in the **Verify** phase, after `simplify-pass`.
Its question: *would a careful senior .NET reviewer approve this as-is?*

## When to use

- After `simplify-pass` on the current task.
- On request against a branch or PR ("review this before I open it").

## Reads

- `.work/<slug>/spec.md`, `design.md`, `decisions.md`, and `review.md` (the Simplification section).
- The diff: `git diff <base>...HEAD`, where base is the repo's default branch.
- Conventions (via `dev-flow`).

## Steps

1. **Build and test first.** Run `dotnet build` (warnings as errors) and `dotnet test`. A red build is the
   first finding; fix it before reviewing anything else.
2. **Check the spec.** Every acceptance criterion is ticked with a named test, the tests actually
   assert the criterion, and nothing in the diff is outside the spec's scope.
3. **Go through the checklist below** on the changed code only. For each item, mark pass, fail, or
   n/a in `review.md` with a file and line for every failure.
4. **Fix what's clearly a defect** (a missing `CancellationToken`, an unbounded query, a missing
   `[Authorize]`) one at a time, re-running the tests after each. For anything that's a judgment call,
   list it with a recommendation and ask the user.
5. **Record deviations.** If a rule is deliberately not followed, add it to the Deviations section
   with the reason, and to `decisions.md` if it's a lasting choice.

## Checklist

**Async and cancellation**
- No `.Result`, `.Wait()`, or `GetAwaiter().GetResult()`; no `async void` outside event handlers.
- `CancellationToken` accepted by every action and passed to every async call (EF, HttpClient, Service Bus).
- No fire-and-forget `Task` without error handling.

**EF Core**
- Reads use `AsNoTracking()` and project to DTOs; no entity returned from a controller.
- No N+1 (a query inside a loop, or lazy loading in a loop).
- Every collection query is paged or bounded; no `ToList()` before `Where`.
- No raw SQL built by string concatenation; `FromSql` uses interpolated parameters.
- One `SaveChangesAsync` per unit of work, with concurrency handled where the design requires it.

**API surface**
- Status codes and response shapes match `design.md`.
- Errors return `ProblemDetails` through the shared handler; no exception details leak to callers.
- Input is validated; IDs from the route are checked against what the caller is allowed to see.

**Security**
- Every new endpoint has an explicit policy; `[AllowAnonymous]` is justified in a comment.
- Authorization covers the resource itself, not only the endpoint (no reading another tenant's
  or user's data by changing an ID).
- No secrets, connection strings, or tokens in code, `appsettings*.json`, logs, or test files.
- Webhook endpoints verify signatures.

**Outbound calls and messaging**
- `HttpClient` comes from `IHttpClientFactory` (typed client), with the resilience handler and timeouts from `design.md`.
- Retries only on idempotent calls or with an idempotency key.
- Messages are published through the outbox; consumers are idempotent and dead-letter poison messages.

**Observability**
- Structured logs with message templates (no string interpolation in log calls), no personal data or secrets.
- Correlation ID flows through outbound calls and messages.
- New dependencies have health checks where the service depends on them to work.

**Schema**
- If the repo uses EF Core migrations, the migration matches `design.md`, and rollback is `Down()` or a written plan (see `ef-migration`).
- If it doesn't, the schema change is written up in `design.md` for the user to apply.

**Tests**
- Failure paths from the spec and `design.md` are tested, not just the happy path.
- Tests use AwesomeAssertions, real dependencies at the edge (Testcontainers, WireMock.Net), and no `Thread.Sleep`.

## Exit criteria

- Build is clean and all tests pass.
- Every checklist item is pass, n/a, or a recorded deviation; no open failures.

## Writes

- `.work/<slug>/review.md` from `dev-flow/templates/review.md` (keeping the Simplification section).
- `.work/<slug>/decisions.md` for lasting deviations.

## Next skill

`ship-pr`.
