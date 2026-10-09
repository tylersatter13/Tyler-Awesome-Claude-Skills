---
name: api-design
description: Design ASP.NET Core controller endpoints before coding - routes, DTOs, status codes, ProblemDetails errors, paging, versioning, auth, and idempotency. Use when a task adds or changes HTTP endpoints, when asked to design an API or contract, or when dev-flow routes to Design.
---

# api-design

Part of the `dotnet-flow` suite (see `dev-flow`). Runs in the **Design** phase.
Its question: *what exactly will callers send and get back, including when things go wrong?*

## When to use

- The spec adds or changes an endpoint.
- Someone asks "what should this API look like?" or wants an OpenAPI contract to share.
- Before building a webhook receiver for a partner (pair with `integration-design`).

## Reads

- `.work/<slug>/spec.md` (criteria drive the contract) and `decisions.md`.
- Existing controllers in the repo: route style, versioning, DTO naming, auth policies, paging shape.
  Match them unless the spec says otherwise.
- The current OpenAPI document if one is generated (Swashbuckle or `Microsoft.AspNetCore.OpenApi`).
- Conventions (via `dev-flow`).

## Steps

1. **List resources and operations** from the acceptance criteria. Prefer nouns and standard verbs;
   use an action route (`POST /orders/{id}/cancel`) only for a real state transition.
2. **Fill in the endpoint table** in `design.md` for each operation: method, route, request DTO,
   success status and response DTO, error statuses, and auth policy.
3. **Define the DTOs** as C# records with types, nullability, and validation rules. Keep them
   separate from entities. Name them `<Verb><Resource>Request` and `<Resource>Response`.
4. **Decide the cross-cutting behavior**, following the repo first and conventions second:
   - Errors: `ProblemDetails`, with `ValidationProblemDetails` for 400 and a stable `type` URI or
     `errorCode` extension when callers need to branch on it.
   - Paging: `page`/`pageSize` with a maximum, or a continuation token for large or changing sets.
     Return the paging info in a consistent envelope.
   - Concurrency: `ETag` / `If-Match` (mapped to an EF Core concurrency token) when two callers
     can update the same resource.
   - Idempotency: an `Idempotency-Key` header for POSTs that create money movement, orders, or
     partner calls, with the stored result replayed on retry.
   - Versioning: a new version only for breaking changes. Adding optional fields is not breaking.
   - Auth: the policy and scopes per endpoint; who can see whose data.
5. **Check backward compatibility** for changed endpoints: removed or renamed fields, tighter
   validation, and changed status codes break callers. Record any breaking change in
   `decisions.md` with the migration path.
6. **Show the user** the endpoint table and DTOs. Agree on the contract before `build-slice` starts.

## Exit criteria

- Every acceptance criterion maps to an endpoint and an expected status.
- Every endpoint has request and response shapes, error statuses, and an auth policy.
- Breaking changes are recorded with a plan, or there are none.

## Writes

- `.work/<slug>/design.md`: the Endpoints section, plus DTO definitions.
- `.work/<slug>/decisions.md`: contract choices that weren't obvious.

## Next skill

`integration-design` if the task calls external systems, `ef-migration` if the schema changes,
otherwise `build-slice`.
