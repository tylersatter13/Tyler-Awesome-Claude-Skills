---
name: ship-pr
description: Write and open the pull request for a finished .NET task - description from the spec, decisions, and review, plus testing steps, migrations, config and secrets, and rollback - linked to its GitHub issue or Jira ticket. Use after review-gate, or when asked to open or describe a PR.
---

# ship-pr

Part of the `dotnet-flow` suite (see `dev-flow`). Runs in the **Ship** phase.
Its question: *does a reviewer and whoever deploys this have everything they need?*

## When to use

- After `review-gate` passes.
- When asked to write or update a PR description for the current branch.

## Reads

- `.work/<slug>/spec.md`, `design.md`, `decisions.md`, and `review.md`.
- The diff and commit list against the base branch.
- The repo's PR template, if one exists (`.github/pull_request_template.md`,
  `.github/PULL_REQUEST_TEMPLATE.md`, or `docs/PULL_REQUEST_TEMPLATE.md`). If it exists, its
  structure wins over the suite template.
- The task source, to link it.

## Steps

1. **Check readiness.** `review.md` has no open failures, the build is clean, tests pass, and the branch is
   pushed and up to date with the base branch. If not, say what's missing and stop.
2. **Write `pr.md`** from the repo template or `dev-flow/templates/pr.md`:
   - **Title**: what changed, in plain words. Use the repo's convention for prefixes or ticket keys
     (for Jira, usually `ABC-123: <summary>`).
   - **Before / After**: what a caller or operator sees, in one short paragraph each.
   - **How to test**: the test names that cover each acceptance criterion, plus manual steps
     (sample requests) when a reviewer would want to try it.
   - **Migrations, config, and secrets**: new settings, Key Vault secrets, Service Bus queues or topics,
     feature flags, and schema changes, with what must happen before or during deploy. Write "None" when empty.
   - **Rollback**: how to undo it, especially for schema and message-contract changes.
   - **Notes for reviewers**: decisions and deviations worth a second look, taken from `decisions.md`
     and `review.md`. Keep it short.
   - **Link**: `Closes #123` for GitHub issues; the Jira key in the title or body for Jira, so the
     integration links it.
3. **Show the user `pr.md`** and wait for a go-ahead before creating anything on GitHub.
4. **Open the PR** with `gh pr create --title "<title>" --body-file .work/<slug>/pr.md`
   (add `--draft` if the user wants a draft). Return the URL.

## Rules

- Describe what the change does, not the history of how it was built.
- Never paste secrets, connection strings, or internal-only hostnames into the PR.
- Keep the PR to the task. If the diff includes unrelated changes, point them out and suggest splitting.

## Exit criteria

- `pr.md` covers testing, deploy steps (or "None"), and rollback.
- The PR is open and linked to its issue or ticket, or the user chose to open it themselves.

## Writes

- `.work/<slug>/pr.md`.
- The PR on GitHub, after the user agrees.

## Next skill

`learn`, after the PR merges.
