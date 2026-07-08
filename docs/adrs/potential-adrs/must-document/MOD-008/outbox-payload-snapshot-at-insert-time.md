# Outbox Stores Pre-Rendered Payload Snapshot at Insert Time

## Score
Total: 135/150 | Step 0: 75 | Scope+Impact: 20/25 | Cost to Change: 25/25 | Team Knowledge: 15/25

## Evidence
- Transcript: `[09:51] Larissa` — "o payload tem que ser o snapshot do momento da transição. Não pode buscar o pedido na hora de enviar."
- Transcript: `[09:52] Diego` — "se alguém editar o pedido depois, o webhook já foi. O cliente recebe o estado exato do momento."
- Transcript: `[09:52] Bruno` — "faz sentido. Grava o JSON completo no outbox."
- File: `src/modules/orders/order.service.ts` — `changeStatus` re-fetches the full order with relations at line ~175 (`tx.order.findUnique({ include: { items: true, customer: true } })`); this fetched object is the snapshot source
- File: `prisma/schema.prisma` — `webhook_outbox.payload` will be a `Json` column storing the pre-rendered event JSON

## Decision Summary
When `publishWebhookEvent(tx, order, fromStatus, toStatus)` inserts a row into `webhook_outbox`, it serializes the complete event payload (including order data, items, customer snapshot, `fromStatus`, `toStatus`) into the `payload` JSON column at that moment. The worker reads and dispatches this stored JSON without re-querying the order. This ensures the payload reflects the exact state of the order at the moment the status changed.

## Context
If the worker lazily fetched the order at dispatch time, subsequent order mutations (e.g., address update, item correction) could cause the dispatched payload to reflect a state the customer never explicitly triggered. Snapshot-at-insert ensures deterministic, reproducible payloads and makes the outbox a faithful event log of what actually happened.

## Alternatives Considered
- **Store only order_id and re-fetch at dispatch time (lazy rendering):** Rejected because order data may change between the status transition and the dispatch attempt (especially given the 15-hour retry window), causing the customer to receive a payload inconsistent with the event that triggered the notification. ([09:51] Larissa, [09:52] Diego)

## Consequences
**Positive:**
- Payload is deterministic: retries always send the same JSON regardless of subsequent order changes
- Outbox doubles as an auditable event log of exact system state at each transition
- Worker is stateless with respect to order data — no need to re-query orders module

**Negative / Trade-offs:**
- `payload` JSON column can be large (full order + items + customer); storage grows proportionally with webhook volume
- Schema evolution: if the payload structure changes, old outbox rows have the old format — worker must handle both versions if rolling deployment is used
- `publishWebhookEvent` must receive the full order object (with relations), not just the ID

## Related Modules
- MOD-007 (Orders) — `changeStatus` provides the full order object to `publishWebhookEvent`
- MOD-008 (Webhooks) — owns `webhook_outbox.payload` schema and worker dispatch logic
