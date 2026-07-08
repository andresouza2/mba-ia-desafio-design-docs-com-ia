# TransactionClient Passed as Parameter to Sub-functions

## Score
Total: 135/150 | Step 0: 75 | Scope+Impact: 20/25 | Cost to Change: 18/25 | Team Knowledge: 22/25

## Evidence
- File: `src/modules/orders/order.service.ts` — `type TxClient = Prisma.TransactionClient` alias; private methods `debitStock(tx: TxClient, items)`, `replenishStock(tx: TxClient, items)`, `reserveOrderNumber(tx: TxClient)` all receive `tx` as first argument
- Transcript: `[09:41] Bruno` — "Vou propor uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que aceita o tx client da transação atual."
- Transcript: `[09:41] Diego` — "Boa, função pura recebendo o tx. Não precisa injetar repository inteiro."

## Decision Summary
Functions that must participate in an existing database transaction receive the `Prisma.TransactionClient` (aliased as `TxClient`) as their first parameter, rather than opening their own transaction or holding a reference to the root `PrismaClient`. This pattern enables composability: the transaction boundary is owned by `changeStatus`, and all sub-functions are pure with respect to the database — they do their work and return, without controlling commit/rollback. The planned `publishWebhookEvent(tx, order, fromStatus, toStatus)` follows this exact convention.

## Context
Node.js does not have implicit transaction context (no thread-local storage equivalent). Passing `tx` explicitly is the idiomatic Prisma pattern for sharing a transaction across multiple operations. The alternative — having each sub-function create its own transaction — would result in independent commits that break atomicity guarantees.

## Alternatives Considered
- **Each sub-function creates its own `prisma.$transaction`:** Rejected — separate transactions break atomicity; stock debit could commit while history insertion fails, leaving inconsistent state.
- **Passing the full `PrismaClient` and letting sub-functions decide:** Rejected — sub-functions could accidentally use the non-transactional client, silently bypassing the transaction. The `TxClient` type enforces participation in the caller's transaction.

## Consequences
**Positive:**
- `publishWebhookEvent(tx, ...)` integrates naturally into `changeStatus` without any additional plumbing
- Sub-functions are easily unit-testable by passing a mock `TxClient`
- The `TxClient` alias makes the intent clear: this function must be called within a transaction
- Pattern is already established — `debitStock`, `replenishStock`, `reserveOrderNumber` all follow it

**Negative / Trade-offs:**
- Every function that participates in the transaction must be refactored to accept `tx` if not already designed that way
- New contributors unfamiliar with Prisma's interactive transactions may be confused by the `TxClient` parameter

## Related Modules
- MOD-007 (Orders) — establishes and owns the `TxClient` pattern
- MOD-008 (Webhooks) — `publishWebhookEvent` adopts this pattern to integrate atomically into `changeStatus`
