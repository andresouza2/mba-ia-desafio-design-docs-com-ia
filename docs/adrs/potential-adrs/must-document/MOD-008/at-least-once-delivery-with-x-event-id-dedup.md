# At-Least-Once Delivery with Client-Side Deduplication via X-Event-Id

## Score
Total: 140/150 | Step 0: 75 | Scope+Impact: 25/25 | Cost to Change: 20/25 | Team Knowledge: 20/25

## Evidence
- Transcript: `[09:24] Diego` — "exactly-once é impossível sem coordenação dos dois lados. A gente entrega at-least-once."
- Transcript: `[09:25] Larissa` — "cada evento tem um UUID único, o X-Event-Id no header. O cliente usa pra deduplicar."
- Transcript: `[09:26] Diego` — "Stripe faz isso, GitHub faz isso. É o padrão da indústria."
- File: `prisma/schema.prisma` — `webhook_outbox.eventId` will be `@unique` UUID generated at insert time

## Decision Summary
The webhook system guarantees at-least-once delivery: every event will be delivered at least once but may be delivered more than once (e.g., after a retry where the first attempt actually succeeded but the response was not received). Each event carries a stable UUID in the `X-Event-Id` header, generated at outbox insert time. Customers are responsible for deduplicating events using this ID on their side.

## Context
Exactly-once delivery requires two-sided distributed coordination (the receiver must atomically acknowledge receipt and mark the event as consumed from the sender's perspective). This is complex to implement correctly and introduces tight coupling between the sender and receiver's reliability. At-least-once with a stable event ID is the accepted industry standard for webhook systems, used by Stripe, GitHub, and Shopify.

## Alternatives Considered
- **Exactly-once delivery:** Rejected because it requires two-sided coordination that is complex to implement and would couple the API's correctness to the receiver's availability and behavior. ([09:24] Diego)
- **No event ID (best-effort, no dedup mechanism):** Not considered — all participants agreed a stable event ID is necessary for receiver-side deduplication from the start.

## Consequences
**Positive:**
- Simple to implement on the sender side: generate UUID at insert, include in header, retry freely
- Industry-standard approach customers are familiar with (Stripe, GitHub model)
- `eventId` uniqueness constraint in `webhook_outbox` prevents duplicate outbox rows for the same logical event
- No tight coupling between sender retry logic and receiver acknowledgment

**Negative / Trade-offs:**
- Receiver must implement idempotency logic — increases integration complexity for customers
- If a customer fails to implement deduplication, they may process duplicate events (e.g., double-decrement stock)
- Documentation and onboarding must clearly explain the deduplication responsibility

## Related Modules
- MOD-008 (Webhooks) — owns `X-Event-Id` generation (`uuid` v4) and outbox `eventId` uniqueness
- MOD-007 (Orders) — `publishWebhookEvent` generates the eventId at the moment of status transition
