---
name: integration-design
description: Design integrations between a .NET service and external systems on Azure - partner REST/SOAP APIs, inbound webhooks, Azure Service Bus, and file drops - covering auth, resilience, idempotency, mapping, and failure modes. Use when a task calls or is called by another system, or when dev-flow routes to Design for an integration.
---

# integration-design

Part of the `dotnet-flow` suite (see `dev-flow`). Runs in the **Design** phase.
Its question: *what happens when the other side is slow, down, wrong, or sends the same thing twice?*

## When to use

- Calling a partner or internal API over HTTP or SOAP.
- Receiving webhooks or callbacks.
- Publishing or consuming Azure Service Bus messages.
- Exchanging files through Blob Storage or SFTP.

## Reads

- `.work/<slug>/spec.md`, `decisions.md`, and `design.md` if `api-design` already ran.
- The partner's documentation, OpenAPI/WSDL, or sample payloads. Ask the user for them if missing;
  don't guess field meanings.
- Existing integrations in the repo: typed clients, resilience setup, messaging library, outbox.
  Reuse the existing pattern.
- Conventions (via `dev-flow`). Code patterns are in `references/patterns.md`.

## Steps

1. **Draw the flow** in a few lines: who calls whom, sync or async, and what data moves.
   Note the trigger (user request, schedule, message, webhook).
2. **Fill in the integrations table** in `design.md` for each connection:
   - **Auth**: Managed Identity for Azure resources; OAuth client credentials, API key, or
     certificate for partners, with secrets in Key Vault.
   - **Timeouts and retries**: per-attempt and total timeouts sized to the partner's SLA. Retries
     with backoff and jitter only for transient failures (408, 429, 5xx, network), and only for
     idempotent calls or calls with an idempotency key. A circuit breaker for partners that go down.
   - **Rate limits**: honor 429 and `Retry-After`; throttle on our side if the partner publishes limits.
   - **Idempotency**: how a retried request, a redelivered message, or a replayed webhook is detected
     (idempotency key, message ID, partner event ID) and where it's stored.
3. **Choose the delivery guarantees.**
   - Need to publish after a database save? Use an outbox in the same transaction, never "save,
     then publish" as two separate steps.
   - Consuming Service Bus? Use peek-lock, process idempotently, then complete. Send poison messages to
     the dead-letter queue with a reason, and decide who watches it.
   - Inbound webhooks: validate the signature, store the event, return 2xx fast, and process it
     asynchronously.
4. **Design the mapping layer.** Partner models live in their own namespace and are mapped to
   domain types in one place. Decide what happens with unknown enum values, missing fields, and
   time zones.
5. **Write the failure-mode table** in `design.md`: partner down, slow, returns 4xx, returns a
   malformed body, duplicate delivery, out-of-order messages, and our own service crashing midway.
   For each, say what the caller sees, what's retried, and what's alerted on.
6. **Plan observability**: one structured log per call with the partner, operation, status, and
   duration. Propagate the correlation ID. Add health checks for critical dependencies, and alerts on
   the dead-letter count and error rate.
7. **Plan the tests** for `dotnet-test`: WireMock.Net scenarios for each failure mode, plus a
   contract test against recorded partner payloads.
8. **Show the user** the flow, the failure-mode table, and any new secrets or config.

## Exit criteria

- Every connection has auth, timeouts, a retry policy, and an idempotency strategy.
- Every failure mode has a defined behavior and a planned test.
- New secrets, settings, queues, and topics are listed for deployment.

## Writes

- `.work/<slug>/design.md`: the Integrations, Failure modes, and Config and secrets sections.
- `.work/<slug>/decisions.md`: choices like "no retries on POST /payments because there's no idempotency key".

## Next skill

`ef-migration` if the outbox or inbox needs tables and the repo uses EF Core migrations, otherwise `build-slice`.
