# Tyler-Awesome-Claude-Skills: dotnet-flow

A coordinated set of Claude Code skills for building .NET APIs and Azure integrations. Every skill
follows one process (Frame → Design → Build → Verify → Ship → Learn), shares a per-task work folder
(`.work/<slug>/`), and reads the same conventions.

## Install

```
/plugin marketplace add tylersatter13/Tyler-Awesome-Claude-Skills
/plugin install dotnet-flow@tylersatter13-skills
```

## Start a task

Ask Claude to "start work on #42" or "start PAY-311" in any .NET repo. `dev-flow` sets up the work
folder, works out the phase, and hands off to the next skill.

## Skills

| Skill | Phase | Status |
|---|---|---|
| dev-flow | Hub | ready |
| frame-task | Frame | planned |
| api-design | Design | planned |
| integration-design | Design | planned |
| ef-migration | Design | planned |
| build-slice | Build | planned |
| dotnet-test | Build / Verify | planned |
| simplify-pass | Verify | draft |
| review-gate | Verify | planned |
| ship-pr | Ship | planned |
| diagnose, learn, scaffold-service | Supporting | planned |

## Overriding conventions

Suite defaults are in `skills/dev-flow/references/conventions.md`. Put repo-specific rules in
`docs/conventions.md` in that repo; they win over the defaults.
