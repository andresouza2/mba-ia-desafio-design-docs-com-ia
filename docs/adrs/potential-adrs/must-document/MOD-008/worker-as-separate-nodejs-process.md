# Worker Runs as a Separate Node.js Process

## Score
Total: 140/150 | Step 0: 75 | Scope+Impact: 25/25 | Cost to Change: 20/25 | Team Knowledge: 20/25

## Evidence
- File: `src/server.ts` — existing process entry point pattern (build app, start listener, SIGINT/SIGTERM handlers) that the worker will mirror
- File: `src/config/database.ts` — exports `createPrismaClient()` callable from any process, enabling a separate PrismaClient instance for the worker
- Transcript: `[09:11] Diego` — "o worker tem que ser um processo separado. Se a gente reiniciar a API não pode parar de entregar webhooks"
- Transcript: `[09:29] Bruno` — "cada processo vai ter seu próprio PrismaClient, mesma DATABASE_URL"
- Transcript: `[09:30] Bruno` — "o worker importa o logger do shared, mesma config"

## Decision Summary
The webhook dispatcher (worker) runs as an independent Node.js process, separate from the Express API process. It has its own `PrismaClient` instance (same `DATABASE_URL`), its own polling loop, and its own `SIGINT`/`SIGTERM` shutdown handlers. The API and worker share `src/shared/logger`, `src/config/env.ts`, and `src/config/database.ts` (via import), but they execute in separate OS processes with no shared memory.

## Context
Node.js is single-threaded. Running the polling loop inside the API process would compete with request handling for the event loop. More importantly, API restarts (deployments, crashes) must not interrupt ongoing webhook delivery. Process isolation provides a clear fault boundary: the API handles inbound HTTP traffic; the worker handles outbound webhook dispatch.

## Alternatives Considered
- **Background task / setInterval inside the API process:** Rejected because API restarts would kill in-flight webhook dispatches and the polling loop would compete with request handling. ([09:11] Diego)
- **Separate microservice in a different language or container:** Not discussed; the team chose to stay in Node.js/TypeScript and reuse existing shared modules.

## Consequences
**Positive:**
- API restarts do not interrupt webhook delivery
- Independent scaling: worker can be scaled separately from the API
- Clear fault boundary; worker crash does not affect API availability
- Mirrors existing `src/server.ts` pattern — minimal new patterns to learn

**Negative / Trade-offs:**
- Two processes to deploy, monitor, and manage
- `package.json` needs a new `"worker"` npm script
- Shared modules (`logger`, `env`, `database`) must remain importable from both process entry points without side effects

## Related Modules
- MOD-001 (Infrastructure) — worker entry point `src/worker.ts` mirrors `src/server.ts`
- MOD-005 (Shared) — worker imports `logger`, `env`, `createPrismaClient`
- MOD-008 (Webhooks) — `WebhookProcessor` class instantiated by the worker
