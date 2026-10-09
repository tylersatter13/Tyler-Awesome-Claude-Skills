---
name: simplify-pass
description: Find and remove unnecessary complexity in .NET code that was just written or changed, without changing behavior. Use after build-slice and before review-gate, or when asked to simplify, clean up, or de-over-engineer C# code.
---

# simplify-pass

Part of the `dotnet-flow` suite (see `dev-flow`). Runs in the **Verify** phase, before `review-gate`.
Its question: *could this be written more simply and still pass the same tests?*

## When to use

- After `build-slice` finishes, on the diff for the current task.
- On request against a file, folder, or PR ("simplify this", "this feels over-engineered").

## Reads

- `.work/<slug>/spec.md` and `design.md`, so you know what the code actually has to do.
- `.work/<slug>/decisions.md`, so you don't undo a deliberate choice.
- Conventions (suite defaults, then `docs/conventions.md`, then the code itself).
- The diff: `git diff <base>...HEAD` (default base is the repo's default branch). Stay inside the
  changed code unless the user widened the scope.

## What to look for

Each finding names the smell, where it is, the simpler shape, and why it is simpler.

**Structure**
- Interfaces with exactly one implementation and no test or DI seam that needs them.
- Abstract base classes, generic repositories, or "Manager/Helper/Processor" layers that only forward calls.
  A generic repository wrapping EF Core is almost always removable: `DbContext` already is one.
- Factories, strategies, or builders for a single case.
- Mapping layers that copy identical shapes more than once (entity → model → DTO with no change).
- Options or flags that nothing sets to a non-default value.

**Code shape**
- Deep nesting that early returns or guard clauses would flatten.
- Hand-written loops that a short LINQ expression states more clearly, and the reverse:
  LINQ chains so long they need comments.
- Custom code that duplicates the framework: hand-rolled validation next to `[ApiController]`,
  manual retry loops instead of the resilience handler, custom JSON handling `System.Text.Json` already does,
  try/catch blocks that only rethrow or turn exceptions into what the `ProblemDetails` handler already produces.
- `async` methods that only `return await` one call where the wrapper adds nothing.
- Duplicate logic across slices that one small private method would cover (but don't create shared
  abstractions for two similar-looking things that may diverge).
- Dead code: unused parameters, private methods, usings, and settings.

**Tests**
- Mocks of things that could be real (EF Core, simple value objects).
- Setup copied across tests that one helper or `[TestInitialize]` would hold.
- Tests that assert implementation details instead of the acceptance criterion.

## What not to change

- Public API contracts (routes, DTO shapes, status codes) or anything in `design.md`, unless the user agrees.
- Choices recorded in `decisions.md`.
- Seams that exist for testing or for a documented upcoming need.
- Behavior. If a simplification changes behavior, it is a bug fix or a feature, not this skill.

## Steps

1. Read the inputs above and list findings, most valuable first. Skip anything that saves fewer than a few
   lines and adds no clarity.
2. Show the list to the user with a one-line "before → after" for each, and ask which to apply
   (default: all low-risk ones).
3. Apply them one at a time. Run `dotnet build` and `dotnet test` after each; revert any change
   that breaks the build or a test.
4. Record removed abstractions or rejected suggestions in `decisions.md` when the reason is not obvious.

## Exit criteria

- Build is clean and all tests pass.
- Every finding is applied, rejected with a reason, or deferred.

## Writes

Appends a `## Simplification` section to `.work/<slug>/review.md` (create it from the template if missing):
the findings, what was applied, and the net lines removed.

## Next skill

`review-gate`.
