---
name: dev-flow
description: Hub for the .NET development workflow (Frame, Design, Build, Verify, Ship, Learn). Use at the start of any task in a .NET repo, when asked "where am I" or "what's next", or when starting work from a GitHub issue or Jira key.
---

# dev-flow

The hub of the `dotnet-flow` skill suite. It owns the shared process, the work folder, and the
conventions every other skill follows. It does not write production code itself; it sets up the
task and routes to the right phase skill.

## The process

| Phase | Skill | Writes | Exit criteria |
|---|---|---|---|
| Frame | `frame-task` | `spec.md` | Acceptance criteria are testable; open questions listed |
| Design | `api-design`, `integration-design`, `ef-migration` | `design.md` | Contract and data changes decided; decisions logged |
| Build | `build-slice` + `dotnet-test` | code, tests | Every acceptance criterion has a passing MSTest test; build is clean |
| Verify | `simplify-pass`, then `review-gate` | `review.md` | Unneeded complexity removed with tests still green; checklist passes, or each deviation is recorded with a reason |
| Ship | `ship-pr` | `pr.md` | PR text covers testing, migrations, config, rollback |
| Learn | `learn` | conventions edits | Lessons turned into rules or skill edits |

Any phase may be skipped, but the skip is recorded in `decisions.md` with a one-line reason.
Bugs enter at Build with a failing test as the spec (via `diagnose` when the cause is unknown).

## Work folder

Each task gets `.work/<slug>/` at the repo root, where `<slug>` is the issue or Jira key plus a
short name (for example `gh-42-order-webhook` or `PAY-311-refund-sync`).

- `.work/` is personal and gitignored. If `.gitignore` lacks `.work/`, add it.
- Every phase skill starts by reading everything in the task's folder and ends by writing its file.
- Templates live in `templates/` next to this file. Copy them; don't invent new layouts.
- `decisions.md` is append-only: `- YYYY-MM-DD: <decision> because <reason>`.

## Conventions

Load conventions in this order; later ones win:

1. `references/conventions.md` (suite defaults: controllers, EF Core, Azure, MSTest)
2. `docs/conventions.md` in the repo, if present
3. What the repo actually does. Read `global.json`, `Directory.Build.props`, the `*.csproj` files,
   and one existing controller and test before assuming anything. When the code and the
   conventions disagree, follow the code and mention the gap.

## Steps

1. **Locate the repo context.** Find the solution file, target framework, test projects, and
   whether `docs/conventions.md` exists.
2. **Identify the task.**
   - `#123` or a GitHub issue URL: `gh issue view <n> --comments`.
   - A Jira key like `ABC-123`: read it through the Atlassian connector if available; otherwise
     ask the user to paste the ticket.
   - Free text: use it as-is.
3. **Find or create the work folder.** Reuse an existing `.work/<slug>/` if one matches.
   Otherwise create it and copy `templates/decisions.md` into it.
4. **Work out the current phase** from the files present:
   - no `spec.md` → Frame
   - `spec.md` but no `design.md` and no recorded skip → Design
   - `design.md` (or a recorded skip) and acceptance criteria without passing tests → Build
   - all criteria covered but no `review.md` → Verify (start with `simplify-pass`)
   - `review.md` has a Simplification section but no checklist results → Verify (`review-gate`)
   - `review.md` but no `pr.md` → Ship
   - `pr.md` exists → Learn
5. **Report and route.** Tell the user in two or three lines: the task, the phase, and the next
   skill. Then invoke it unless the user only asked for status.

## Sizing

Scale the ceremony to the task. A one-line fix can have a two-line spec and a recorded Design skip.
A new partner integration gets the full loop. When unsure, ask once: "full flow or quick fix?"

## Skills in this suite

`frame-task`, `api-design`, `integration-design`, `ef-migration`, `build-slice`, `dotnet-test`,
`simplify-pass`, `review-gate`, `ship-pr`, `diagnose`, `learn`, `scaffold-service`. If a skill isn't installed yet,
do that phase directly by following the exit criteria above and the template for its file.
