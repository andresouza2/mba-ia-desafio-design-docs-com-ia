# Audit Trail via OrderStatusHistory Table

## Score
Total: 131/150 | Step 0: 75 | Scope+Impact: 18/25 | Cost to Change: 18/25 | Team Knowledge: 20/25

## Evidence
- File: `prisma/schema.prisma` — `OrderStatusHistory` model with `fromStatus` (nullable, capturing creation), `toStatus`, `changedById` (FK to User), `changedAt`, `reason`
- File: `src/modules/orders/order.service.ts` — initial history row created at order creation with `fromStatus: null`; subsequent rows created inside `changeStatus` transaction
- Transcript: `[09:36] Sofia` — "o endpoint de admin tem que logar quem fez o replay, pra auditoria"

## Decision Summary
Every order status change — including the initial creation — is recorded as a row in the `order_status_history` table, capturing `fromStatus` (nullable on creation), `toStatus`, `changedById` (the authenticated user who triggered the change), `changedAt`, and an optional `reason`. The history row is inserted inside the same `prisma.$transaction` as the status update, guaranteeing that no status change occurs without an audit record.

## Context
B2B order management requires a complete audit trail for dispute resolution, compliance, and debugging. Knowing that an order transitioned from PAID to CANCELLED is insufficient without knowing who did it, when, and why. The nullable `fromStatus` elegantly captures the order creation event in the same table without a separate `order_created_at` column.

## Alternatives Considered
- **Audit log in a separate audit service or table with CDC:** Rejected as over-engineered for the current scale; the `order_status_history` table in the same MySQL instance is sufficient and queryable with Prisma.
- **Storing history in a JSON column on the orders table:** Rejected — JSON columns cannot be queried efficiently for individual history entries or joined for reporting.

## Consequences
**Positive:**
- Full, queryable audit trail of every status transition including actor and timestamp
- `fromStatus: null` semantically captures the "created" event without a separate event type
- History rows are part of the same transaction as the status update — no inconsistency possible
- DLQ replay audit (per [09:36] Sofia) will add a history row via the same mechanism

**Negative / Trade-offs:**
- `order_status_history` grows proportionally with order volume and transition frequency; requires a retention/archival strategy at scale
- Every `changeStatus` call requires an additional `INSERT` inside the transaction, adding latency under high load

## Related Modules
- MOD-007 (Orders) — owns `OrderStatusHistory` creation
- MOD-003 (Users) — `changedById` FK references the `users` table
- MOD-008 (Webhooks) — DLQ replay will create a history entry to satisfy audit requirements
