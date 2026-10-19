# Multi-Level Payment Approval System

## Goal
Design a secure payment approval workflow that turns payment requests into auditable authorization decisions before a separate execution process attempts payment.

## Roles
| Role | Responsibility |
| --- | --- |
| Requester | Creates and submits requests; cannot approve their own requests |
| Approver | Makes first-level approval or rejection decisions |
| Senior Approver | Makes second-level decisions for higher-value requests; must differ from the first approver |
| Administrator | Configures policies but cannot bypass approval requirements |

## Illustrative business rules
- Payments of **€5,000 or less** require one authorized approval.
- Payments **above €5,000** require two distinct levels of approval.
- The requester cannot approve their own payment; no approver can supply both approvals on one request.
- Approval thresholds are examples for this exercise, not regulatory requirements.
- The approval policy and threshold applied at submission should be captured or versioned for consistent review.
- Every decision and state change should be auditable.
- Payment approval does **not** mean payment execution has succeeded.

## Logical architecture
1. **Frontend** : request entry, review screens, user feedback.
2. **API layer** : caller authentication, input validation, API contract.
3. **Authorization** : role, ownership, scope, and segregation-of-duties checks.
4. **Approval engine** : threshold policy, transition evaluation, concurrency guard.
5. **Payment store** : requests, decisions, version numbers, policy snapshot.
6. **Audit/outbox** : durable decision history and event publishing.
7. **Execution service** : handles approved requests independently and tracks execution outcomes.

## State model
- `DRAFT` → `PENDING_APPROVAL` on valid submission.
- `PENDING_APPROVAL` => `APPROVED` after one approval for a request of €5,000 or less.
- `PENDING_APPROVAL` => `PENDING_LEVEL_2` after a first-level approval for a request above €5,000.
- `PENDING_LEVEL_2` => `APPROVED` after a distinct senior approver authorizes it.
- `PENDING_APPROVAL` or `PENDING_LEVEL_2` => `REJECTED` on authorized rejection.
- `APPROVED` => `EXECUTING` => `COMPLETED` or `EXECUTION_FAILED` in the separate payment execution lifecycle.
- Invalid transitions must not mutate the request.

## Security and reliability principles
- The server derives actor identity from the authenticated session/token; never trust client-supplied actor IDs.
- An approval decision checks current state, authorization, and expected version atomically.
- Requests for the same action may be retried: use idempotency protection where appropriate.
- Persist audit events consistently with state changes (for example, a transactional outbox).
- Restrict audit-history access and avoid exposing sensitive beneficiary data unnecessarily.
