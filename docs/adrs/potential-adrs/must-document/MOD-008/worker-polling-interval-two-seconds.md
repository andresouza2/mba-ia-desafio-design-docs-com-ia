# Worker Polls webhook_outbox Every 2 Seconds

## Score
Total: 125/150 | Step 0: 75 | Scope+Impact: 20/25 | Cost to Change: 15/25 | Team Knowledge: 15/25

## Evidence
- Transcript: `[09:09] Diego` — "dois segundos de polling. Atende o SLA de menos de 10 segundos."
- Transcript: `[09:10] Larissa` — "MySQL não tem NOTIFY nativo pra aplicação externa. Polling é o caminho."
- Transcript: `[09:10] Diego` — "dois segundos é razoável. Não sobrecarrega o banco."
- File: `src/config/database.ts` — `createPrismaClient()` is designed to be called independently, enabling the worker to have its own connection pool
- File: `prisma/schema.prisma` — `webhook_outbox` will need indexes on `(status, nextRetryAt)` and `createdAt` to keep polling queries efficient

## Decision Summary
The webhook worker polls the `webhook_outbox` table every 2 seconds for rows with `status = PENDING` and `nextRetryAt <= NOW()`, ordered by `createdAt ASC`. This interval satisfies the <10-second end-to-end delivery SLA discussed in the meeting while keeping MySQL load minimal. MySQL does not support native external notifications, making polling the only viable approach without introducing a message broker.

## Context
The team considered event-driven delivery (triggered immediately on status change) but MySQL has no native mechanism to notify an external process of row insertions without a broker. A 2-second polling interval was chosen as a pragmatic balance: short enough to meet the latency SLA, long enough to avoid hammering the database with constant queries.

## Alternatives Considered
- **MySQL database triggers with external notification:** Rejected because MySQL triggers cannot call external systems reliably, and any trigger-based approach would require additional infrastructure (e.g., UDF or external listener process). ([09:10] Larissa)
- **Shorter polling interval (e.g., 500ms):** Not explicitly discussed; the team settled on 2s as sufficient for the <10s SLA.
- **Redis-based queue for immediate dispatch:** Considered as part of the Redis Streams discussion and rejected with it — same reasoning: no Redis infrastructure in place. ([09:08] Diego)

## Consequences
**Positive:**
- Simple implementation: a `setInterval` loop with a Prisma query
- Satisfies the <10-second delivery SLA (worst case: 2s polling + network time)
- Low database load: one indexed SELECT every 2 seconds
- No additional infrastructure beyond MySQL

**Negative / Trade-offs:**
- Up to 2-second latency before a newly-inserted outbox event is picked up
- Under high webhook volume, a single worker processing one event at a time per poll cycle may lag — batch processing or parallel dispatch may be needed in the future
- `webhook_outbox` needs proper indexes to keep the polling query fast as the table grows

## Related Modules
- MOD-008 (Webhooks) — `WebhookProcessor` owns the poll loop and dispatch logic
- MOD-001 (Infrastructure) — `src/worker.ts` drives the polling lifecycle and shutdown
