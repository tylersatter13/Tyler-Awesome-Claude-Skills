# Design: <Title>

## Endpoints
| Method | Route | Request | Success | Errors | Auth policy |
|---|---|---|---|---|---|
| POST | /api/v1/... | `CreateXRequest` | 201 `XResponse` | 400, 409 | `...` |

## Integrations
| Partner/system | Direction | Protocol | Auth | Timeout / retry | Idempotency |
|---|---|---|---|---|---|

### Failure modes
| Scenario | Behavior |
|---|---|
| Partner down | |
| Partner slow | |
| Bad response | |
| Duplicate message | |

## Data changes
<Entities, migrations, expand/contract steps, backfills.>

## Config and secrets
<New settings, Key Vault secrets, feature flags.>
