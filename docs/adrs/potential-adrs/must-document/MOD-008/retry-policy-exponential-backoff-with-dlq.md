# Retry Policy: 5 Attempts with Exponential Backoff and Dead Letter Queue

## Score
Total: 150/150 | Step 0: 75 | Scope+Impact: 25/25 | Cost to Change: 25/25 | Team Knowledge: 25/25

## Evidence
- Transcript: `[09:15] Diego` — "cinco tentativas. 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas."
- Transcript: `[09:16] Larissa` — "cobre uma janela de aproximadamente 15 horas de indisponibilidade do cliente"
- Transcript: `[09:16] Diego` — "se passar das cinco, vai pra dead letter"
- Transcript: `[09:17] Larissa` — "a dead letter fica numa tabela separada pra não sujar o outbox"
- Transcript: `[09:15] Bruno` — "três tentativas é agressivo demais pra B2B"
- File: `prisma/schema.prisma` — `webhook_outbox` will have `retryCount`, `nextRetryAt`, `status` fields; `webhook_dead_letter` is a separate table

## Decision Summary
Failed webhook deliveries are retried up to 5 times using exponential backoff intervals of 1m, 5m, 30m, 2h, and 12h (covering a ~15-hour outage window). After the 5th failure, the event is moved to a `webhook_dead_letter` table and removed from `webhook_outbox`. ADMIN users can replay dead-letter events via `POST /admin/webhooks/dead-letter/:id/replay`.

## Context
B2B customers may experience planned or unplanned downtime. A retry policy must balance between giving sufficient time for recovery and not accumulating stale events indefinitely in the outbox. The DLQ provides an escape hatch for permanently-failed events without blocking the outbox polling query.

## Alternatives Considered
- **3 attempts only:** Rejected as too aggressive for B2B context — a 3-attempt policy with shorter intervals would fail permanently after ~36 minutes, insufficient for customers with overnight maintenance windows. ([09:15] Bruno, Diego)
- **Retry indefinitely:** Rejected because events can become permanently stale (e.g., order already cancelled by the time the endpoint recovers); indefinite retry pollutes the outbox and makes the worker harder to reason about. ([09:16] Larissa)

## Consequences
**Positive:**
- Covers ~15-hour customer outage window before escalating to DLQ
- `webhook_outbox` stays clean — delivered and permanently-failed events are removed or moved
- DLQ replay endpoint allows manual intervention by ADMIN without code changes
- `nextRetryAt` field enables the worker to skip events not yet due, avoiding unnecessary processing

**Negative / Trade-offs:**
- 5 retries over 15 hours means some events may be delayed significantly before the client receives them
- DLQ table requires monitoring; events left in DLQ indefinitely may confuse operators
- `webhook_outbox` needs indexes on `(status, nextRetryAt)` for efficient worker polling

## Related Modules
- MOD-008 (Webhooks) — owns the outbox, DLQ table, and processor retry logic
- MOD-002 (Auth) + MOD-005 (Shared) — `requireRole('ADMIN')` guards the DLQ replay endpoint
