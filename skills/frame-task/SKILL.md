---
name: frame-task
description: Turn a GitHub issue, Jira ticket, or rough idea into a short spec with testable acceptance criteria before any .NET code is written. Use when starting a task, when asked to "spec", "scope", or "write acceptance criteria", or when dev-flow routes to the Frame phase.
---

# frame-task

Part of the `dotnet-flow` suite (see `dev-flow`). Runs in the **Frame** phase.
Its question: *what are we building, and how will we know it works?*

## When to use

- At the start of a task, before design or code.
- When a ticket is vague and needs pinning down.
- When an existing `spec.md` needs updating because scope changed.

## Reads

- The task source:
  - GitHub issue (`#123` or a URL): `gh issue view <n> --comments`
  - Jira key (`ABC-123`): the Atlassian connector if available; otherwise ask the user to paste it
  - Free text from the user
- `.work/<slug>/` if it exists (an earlier spec, decisions).
- The code the task touches. Find the relevant controllers, services, entities, and tests so
  the spec uses the repo's real names and doesn't promise something the code can't do.
- Conventions (via `dev-flow`).

## Steps

1. **Restate the problem** in two to four sentences: who needs what, and why. If you can't, the
   ticket is too vague. Go to step 3 first.
2. **Draw the scope line.** List what is in and what is explicitly out. Out-of-scope items
   that someone might assume are included matter most.
3. **Ask before writing.** Collect the questions whose answers change the work (validation rules,
   error behavior, who may call it, volumes, partner quirks) and ask them in one batch. Offer a
   default for each so the user can answer "defaults are fine". Don't ask what the code or ticket
   already answers.
4. **Write acceptance criteria** as Given/When/Then, one observable outcome each. Good criteria:
   - name the HTTP status, response shape, persisted state, or message published
   - cover the main failure paths (invalid input, not found, conflict, unauthorized, partner down)
   - are specific enough that `dotnet-test` can name a test after each one
5. **Pick the size**: quick fix, standard, or full flow (see `dev-flow` Sizing). For a quick fix,
   the spec can be two lines and one criterion.
6. **Suggest the next phase.** If the task adds or changes endpoints, the next skill is
   `api-design`. If it calls an external system or uses Service Bus, it is `integration-design`.
   If it changes the schema, it is `ef-migration`. If none apply, record a Design skip in
   `decisions.md` and go straight to `build-slice`.

## Exit criteria

- Every acceptance criterion is testable and has an ID (AC1, AC2, ...).
- Open questions are answered, or listed with an owner and a default the work will assume.
- The user has seen the spec. For standard and full-flow tasks, they have agreed to it.

## Writes

- `.work/<slug>/spec.md` from `dev-flow/templates/spec.md`.
- `.work/<slug>/decisions.md`: assumptions taken as defaults, and any Design skip.

## Next skill

`api-design`, `integration-design`, or `ef-migration` as chosen in step 6; otherwise `build-slice`.
