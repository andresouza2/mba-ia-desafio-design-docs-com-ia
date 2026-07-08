# Transactional Atomicity in changeStatus (Status + Stock + History in Single $transaction)

## Score
Total: 144/150 | Step 0: 75 | Scope+Impact: 22/25 | Cost to Change: 24/25 | Team Knowledge: 23/25

## Evidence
- File: `src/modules/orders/order.service.ts` — `changeStatus` wraps order fetch, `canTransition` guard, stock debit/replenish, `order.update`, `orderStatusHistory.create`, and the planned `publishWebhookEvent` in a single `prisma.$transaction(async (tx) => { ... })`
- Transcript: `[09:06] Diego` — "a gente precisa garantir que o evento não se perca se o processo cair no meio"
- Transcript: `[09:40] Bruno` — "inserir na webhook_outbox dentro da mesma transação. Se a outbox falhar de inserir, rollback."
- Transcript: `[09:41] Diego` — "Boa, função pura recebendo o tx."

## Decision Summary
`OrderService.changeStatus` performs all mutations — status update, stock debit/replenish, audit history insertion, and (planned) outbox event insertion — inside a single `prisma.$transaction`. If any step fails, the entire operation rolls back atomically. The transaction client (`tx: Prisma.TransactionClient`) is passed as a parameter to sub-functions (`debitStock`, `replenishStock`, `reserveOrderNumber`, and the planned `publishWebhookEvent`), keeping them composable without requiring their own Prisma client.

## Context
Without transactional atomicity, a process crash between the stock debit and the status update would leave the system in an inconsistent state (stock debited but order still PENDING). The webhook outbox integration reinforces this need: the event must be captured if and only if the status change commits.

## Alternatives Considered
- **Separate transactions per step (status, then stock, then history):** Rejected — any failure between steps leaves partial state visible to other requests; compensating transactions would be complex and error-prone.
- **Saga pattern with compensating actions:** Rejected as over-engineered for the current scale; the single-DB transaction satisfies all consistency requirements without distributed coordination.

## Consequences
**Positive:**
- All-or-nothing atomicity: stock, status, history, and outbox are always consistent with each other
- `publishWebhookEvent(tx, ...)` receives the same transaction client — outbox insert is atomic with the status change
- Sub-functions receiving `tx` are purely functional and easily unit-testable with a transaction mock

**Negative / Trade-offs:**
- Long-running transactions (e.g., slow HTTP calls inside the tx) would hold DB locks — `publishWebhookEvent` must only do DB work, never network I/O, inside the transaction
- All sub-functions must accept `tx: Prisma.TransactionClient` — introduces a parameter threading convention that must be followed by all future contributors

## Related Modules
- MOD-007 (Orders) — owns `changeStatus` and the transaction boundary
- MOD-008 (Webhooks) — `publishWebhookEvent(tx, ...)` must operate inside this transaction
- MOD-006 (Products) — stock mutations (`debitStock`, `replenishStock`) operate via `tx` on the products table
