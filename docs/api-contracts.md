# Trellis Public API Contracts & Drift Guide

This document specifies the contract definitions, request/response formats, authentication, and error codes for Trellis Frontend endpoints. Automated drift tests enforce that live endpoints match these specifications.

---

## Endpoint Catalog

### 1. Affiliate Endpoints

#### `GET /api/affiliates`
- **Description**: Lists all affiliate sub-endpoints.
- **Auth**: None
- **Response Format**:
```json
{
  "message": "Affiliate API endpoints",
  "endpoints": {
    "stats": "GET /api/affiliates/stats?wallet=<address>",
    "earnings": "GET /api/affiliates/earnings?wallet=<address>&days=<number>",
    "payouts": "GET /api/affiliates/payouts?wallet=<address>",
    "program": "GET /api/affiliates/program",
    "referrals": "GET /api/affiliates/referrals?wallet=<address>",
    "validate": "GET /api/affiliates/validate?wallet=<address>"
  }
}
```

#### `GET /api/affiliates/program`
- **Description**: Returns affiliate program tiers and commission rates.
- **Response Format**:
```json
{
  "tiers": [
    {"name": "Bronze", "commissionRate": 0.05, "minVolumeXlm": 0},
    {"name": "Silver", "commissionRate": 0.10, "minVolumeXlm": 5000},
    {"name": "Gold", "commissionRate": 0.15, "minVolumeXlm": 25000}
  ],
  "cookieDurationDays": 30
}
```

---

### 2. Metrics & Observability

#### `GET /api/metrics/panels`
- **Description**: Returns all configured dashboard panels.
- **Response Format**:
```json
{
  "source": "prometheus | mock",
  "panels": [
    { "id": "agent-executions", "title": "Agent Invocations", "type": "timeseries" }
  ]
}
```

---

### 3. Bulk Ingestion Pipeline

#### `POST /api/import/dry-run`
- **Description**: Previews import changes with zero persistent writes.
- **Query Params**:
  - `entityType` (optional, default: `general`): `agents` | `test-cases` | `provenance` | `submissions`
- **Response Format (200 OK)**:
```json
{
  "success": true,
  "dryRun": true,
  "entityType": "agents",
  "summary": {
    "totalRows": 1,
    "validRows": 1,
    "invalidRows": 0,
    "createCount": 1,
    "updateCount": 0,
    "skipCount": 0,
    "errorCount": 0,
    "duplicateCount": 0
  },
  "rows": [
    { "rowNumber": 1, "action": "create", "valid": true, "errors": [] }
  ]
}
```

---

## Cross-Service Contract Map: Domain Events & Integration Boundaries

This section is the authoritative map of domain events that cross service boundaries in the Trellis system. Every event has exactly one producer and one or more consumers. The map is enforced by `___tests___/contracts/domain-events.test.ts` (see %Contract Verification% below).

### Event Envelope

All domain events are transported as JSON objects with the following envelope. The envelope is stable across all event types; only `data` changes per event.

```json
{
  "id": "<canonical uuid>",
  "type": "<domain.event.name>",
  "version": "<major>`.`<minor>`.`<patch>`",
  "occurredAt": "<ISO-8601 timestamp>",
  "producer": "<service name>",
  "correlationId": "<uuid or null>",
  "data": { }
}
```

### Event Registry

| Event type | Producer | Consumers | Required `data` fields | Version rule | Allowed failure behavior |
|------------|----------|-----------|----------------------|--------------|--------------------------|
| `affiliate.registered` | Affiliate API | Metrics service, Notifications service | `wallet`, `tier`, `referralCode` | additive only; consumers must ignore unknown fields | consumer retries with exponential backoff; event is dropped after 5 attempts with a dead-letter record |
| `affiliate.earnings.accrued` | Affiliate API | Metrics service, Dashboard read model | `wallet`, `amountXlm`, `currency` | additive only; major bump required to remove or rename a field | consumer must not block the producer; failure is logged and retried out-of-band |
| `affiliate.payout.requested` | Affiliate API | Payments service, Notifications service | `wallet`, `amountXlm`, `payoutId` | additive only; consumers must ignore unknown fields | consumer retries with exponential backoff; event is dropped after 5 attempts with a dead-letter record |
| `affiliate.payout.completed` | Payments service | Affiliate API, Metrics service, Notifications service | `wallet`, `payoutId`, `txHash` | additive only; consumers must ignore unknown fields | consumer retries with exponential backoff; event is dropped after 5 attempts with a dead-letter record |
| `import.dry-run.completed` | Bulk Ingestion Pipeline | Metrics service, Dashboard read model | `entityType`, `summary`, `durationMs` | additive only; major bump required to remove or rename a field | consumer must not block the producer; failure is logged and retried out-of-band |
| `import.commit.completed` | Bulk Ingestion Pipeline | Metrics service, Dashboard read model, Notifications service | `entityType`, `summary`, `durationMs` | additive only; major bump required to remove or rename a field | consumer must not block the producer; failure is logged and retried out-of-band |
| `metrics.panel.configured` | Metrics service | Dashboard read model | `panelId`, `title`, `type` | additive only; consumers must ignore unknown fields | consumer must not block the producer; failure is logged and retried out-of-band |

### Version Rules

1. Every event carries a semantic version in `version` using `SemVer` format.
2. **Additive changes** (adding a new optional field) bump the **patch** version and do not require consumer changes. Consumers must ignore unknown fields.
3. **Behavioral changes** (changing a field's meaning or default) bump the **Minor** version. Consumers must handle both old and new minor versions during a migration window.
4. **Breaking changes** (removing or renaming a required field, changing types) bump the **Major** version and require a documented migration plan in this file.
5. Producers must not remove a field without a major version bump.
6. Consumers must not reject an event solely because it contains unknown fields.

### Allowed Failure Behavior

- **Producer failure**: If a producer cannot emit an event, the underlying user operation must still succeed when the event is not required for correctness (e.g. metrics, notifications). The failure is logged and retried out-of-band.
- **Consumer failure**: Consumers must not block the producer. Failed consumption is retried with exponential backoff and dropped after 5 attempts with a dead-letter record.
- **Ordering**: Events are not globally ordered. Consumers that require ordering must use `occurredAt` and `correlationId` to reconstruct sequence.
- **Duplicates**: Events are at-least-once delivered. Consumers must be idempotent keyed on the envelope `id`.

### Operational Edge Cases

- **Out-of-order delivery**: A consumer may receive `affiliate.payout.completed` before `affiliate.payout.requested`. Consumers must tolerate this and reconcile via `payoutId`.
- **Duplicate delivery**: The same event may be delivered more than once. Consumers must de-dupe on `id`.
- **Partial failure**: If a consumer fails after partially processing an event, the retry must resume idempotently.
- **Schema drift**: If a producer emits a field not in this map, consumers must ignore it and not fail.
- **Missing required field**: If a required field is missing, the consumer must log and route the event to the dead-letter queue without blocking other events.

### Failure Modes

| Mode | Detection | Response |
|------|----------|----------|
| Producer emits unknown event type | Consumer receives type not in registry | Log and drop; alert if rate exceeds threshold |
| Required field missing | Schema validation fails | Route to dead-letter queue; do not block producer |
| Version major mismatch | Consumer sees major bump without support | Reject and alert; require consumer upgrade |
| Dead-letter queue growing | Queue depth monitor alert | Operator investigates and replays after fix |
| Consumer down for extended period | Health check fails | Events buffer in queue; replay on recovery |

### Rollback / Recovery Steps

1. **Disable the consumer** by removing it from the consumer group and letting events buffer in the queue.
2. **Revert the consumer deployment** to the last known-good commit.
3. **Replay from the dead-letter queue** after the fix is deployed, using the original `correlationId`.
4. **Verify idempotency** by re-playing the same event id and confirming no duplicate side effects.
5. **Restore the consumer** to the consumer group and confirm lag returns to baseline.

### Contract Verification

The map is enforced by `___tests___/contracts/domain-events.test.ts`, which asserts:

- Every event type in the registry has at least one producer and one consumer.
- Every required `data` field is present in the event's schema.
- Every event carries a valid `SemVer` `version`.
- Every consumer declares an allowed failure behavior.
- The envelope fields (`id`, `type`, `version`, `occurredAt`, `producer`, `correlationId`, `data`) are present on every event.

---

## Error Handling Standards
All API error responses follow the standard Trellis error payload:
```json
{
  "error": "Human readable error message",
  "code": "ERROR_CODE_STRING",
  "details": []
}
```
