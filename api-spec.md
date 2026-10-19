# Payment Approval API  Input Contract

## Common conventions
- Base path: `/api/v1`
- JSON requests and responses.
- All routes require an authenticated caller; permissions depend on request ownership and role.
- Monetary values use integer minor units (EUR cents) to avoid floating-point rounding errors.
- Threshold: **500000 cents = €5,000**. Amounts >500000 cents require two approvals.
- Server identity is derived from authentication, not from request body fields.
- Error response shape: `{ "error": { "code": "...", "message": "..." } }`.

## Endpoints

### `POST /api/v1/payment-requests`
Creates a draft request for the authenticated requester.

Example request:
```json
{
  "amountMinor": 820000,
  "currency": "EUR",
  "beneficiaryId": "ben_201",
  "reference": "Supplier invoice INV-2026-18"
}
```
Example response — `201 Created`:
```json
{
  "id": "pay_1024",
  "status": "DRAFT",
  "version": 1,
  "amountMinor": 820000,
  "currency": "EUR"
}
```
Validation: amount must be positive; supported currency required; beneficiary and reference must be valid.

### `POST /api/v1/payment-requests/{id}/submit`
Only the requester who owns a valid `DRAFT` can submit it. Snapshot the applicable approval policy. Result: `PENDING_APPROVAL`.

Example response — `200 OK`:
```json
{"id":"pay_1024","status":"PENDING_APPROVAL","version":2,"requiredApprovals":2}
```

### `POST /api/v1/payment-requests/{id}/decisions`
Only a permitted approver can approve or reject an eligible request. Each transition must be atomic and version-checked. The same person cannot be requester and approver, or serve as both first and second approver.

Example request:
```json
{
  "decision": "APPROVE",
  "expectedVersion": 2,
  "reason": "Beneficiary verified"
}
```
For requests above €5,000, successful first approval response — `200 OK`:
```json
{
  "id": "pay_1024",
  "status": "PENDING_LEVEL_2",
  "version": 3,
  "requiredApprovals": 2,
  "completedApprovals": 1
}
```
Senior approval by a distinct actor results in `APPROVED`; rejection results in `REJECTED`. Duplicate or stale decisions must not result in repeated state changes.

### `GET /api/v1/payment-requests/{id}`
Returns state, version, amount, approval progress, and safe view of actions available to the caller. Visibility is restricted to permitted users.

### `GET /api/v1/payment-requests/{id}/audit-events`
Returns paginated decision events to authorized audit viewers. Each event records request ID, actor ID, action, previous/new state, timestamp, and correlation ID. No unauthorized read access.

## HTTP error cases
| Status | Example |
| --- | --- |
| `400 Bad Request` | Invalid JSON/required field |
| `401 Unauthorized` | Missing or invalid authentication |
| `403 Forbidden` | Actor lacks approval permission or violates separation of duties |
| `404 Not Found` | Resource not found or hidden from caller |
| `409 Conflict` | Stale `expectedVersion`, duplicate/conflicting decision, invalid state transition |
| `422 Unprocessable Content` | Well-formed request that violates a domain validation rule |

## Idempotency, concurrency, and event consistency
- A conditional update or optimistic locking check should ensure a single valid transition per request version.
- Support an idempotency key for write operations when clients can retry due to network failure.
- Persist a durable audit/outbox record in the same transaction as the state change; publish asynchronously if needed.
- Never equate approval success with executed/settled payment.

## Traceability
| Requirement | API coverage |
| --- | --- |
| Lifecycle | Create, submit, get details |
| Authorization | All endpoints, especially decisions |
| Threshold and distinct approval levels | Submit policy selection + decision validation |
| Audit, concurrency, failure handling | Decision expectedVersion, audit endpoint, separate execution lifecycle |
