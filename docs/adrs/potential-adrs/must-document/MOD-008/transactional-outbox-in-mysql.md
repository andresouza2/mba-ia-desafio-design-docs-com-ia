# Transactional Outbox Pattern in MySQL

## Score
Total: 150/150 | Step 0: 75 | Scope+Impact: 25/25 | Cost to Change: 25/25 | Team Knowledge: 25/25

## Evidence
- File: `src/modules/orders/order.service.ts` — `changeStatus` already uses `prisma.$transaction(async (tx) => { ... })`, making the outbox insert naturally atomic
- File: `prisma/schema.prisma` — all models use MySQL as the data source; no external broker infrastructure exists
- Transcript: `[09:06] Diego` — "a gente precisa garantir que o evento não se perca se o processo cair no meio"
- Transcript: `[09:07] Larissa` — "a forma mais simples dado o que a gente já tem é escrever na mesma transação. Outbox no MySQL mesmo."
- Transcript: `[09:08] Diego` — "Redis Streams seria mais elegante mas adiciona infra nova. O time não tem experiência com Redis Cluster."

## Decision Summary
Webhook notification events are captured using the Transactional Outbox pattern: when `OrderService.changeStatus` commits a status transition, a `publishWebhookEvent(tx, order, fromStatus, toStatus)` call inserts a row into `webhook_outbox` inside the same `prisma.$transaction`. This guarantees that an event is recorded if and only if the status change is persisted — no external broker required.

## Context
The system needs to notify B2B customers (Atlas Comercial, MaxDistribuição, Nova Cargo) when order status changes. A naive approach (calling the customer's HTTP endpoint directly inside `changeStatus`) risks losing the event if the HTTP call fails or the process crashes mid-transaction. The team needed a durable, atomic event capture mechanism that works with the existing MySQL infrastructure.

## Alternatives Considered
- **Synchronous HTTP call inside changeStatus:** Rejected because a slow or unavailable customer endpoint would block the order status update and could cause transaction timeouts. ([09:06] Diego)
- **Redis Streams / Redis Cluster:** Rejected because it introduces new infrastructure the team has no operational experience with, adding operational risk and cost. ([09:08] Diego, Larissa)
- **MySQL triggers:** Rejected because MySQL triggers cannot reliably call external systems and would create hidden side-effects invisible to application code. (implied in mapping, [09:09] Larissa)

## Consequences
**Positive:**
- Atomic event capture: event is guaranteed to be recorded with the status change or not at all
- No new infrastructure — reuses existing MySQL instance and Prisma ORM
- `publishWebhookEvent(tx, ...)` receives the `Prisma.TransactionClient` already typed in `order.service.ts`
- Worker can poll at its own pace without coupling to the API request cycle

**Negative / Trade-offs:**
- `webhook_outbox` table grows and requires periodic cleanup (DLQ + delivered record pruning)
- Polling introduces latency (up to 2s before dispatch begins)
- MySQL is not optimized for high-throughput event streaming; if webhook volume grows significantly, this design may need revisiting

## Related Modules
- MOD-007 (Orders) — integration point is `OrderService.changeStatus`
- MOD-008 (Webhooks) — owns `webhook_outbox`, worker, and processor
- MOD-001 (Infrastructure) — `prisma/schema.prisma` requires new models
