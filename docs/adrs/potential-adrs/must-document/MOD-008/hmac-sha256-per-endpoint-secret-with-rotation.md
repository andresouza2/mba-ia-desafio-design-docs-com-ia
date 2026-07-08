# HMAC-SHA256 Authentication with Per-Endpoint Secret and Rotation Grace Period

## Score
Total: 150/150 | Step 0: 75 | Scope+Impact: 25/25 | Cost to Change: 25/25 | Team Knowledge: 25/25

## Evidence
- Transcript: `[09:20] Sofia` — "cada endpoint tem um secret próprio. Se vazar o secret de um cliente não compromete os outros."
- Transcript: `[09:21] Sofia` — "HMAC-SHA256 no payload completo, manda no header X-Signature"
- Transcript: `[09:22] Larissa` — "rotação com grace period de 24 horas. Durante esse período os dois secrets são válidos."
- Transcript: `[09:22] Diego` — "concordo. Evita janela de falha durante a rotação."
- File: `prisma/schema.prisma` — `WebhookEndpoint` model will have `secret` field (per-endpoint, not global)

## Decision Summary
Each registered webhook endpoint has its own unique HMAC-SHA256 secret stored in the `webhook_endpoints` table. The worker signs the full outgoing payload with this secret and includes the signature in the `X-Signature` header. When a customer rotates their secret, both the old and new secrets remain valid for a 24-hour grace period to avoid delivery failures during the transition.

## Context
Without payload signing, a malicious actor who intercepts or forges HTTP requests to a customer's endpoint could inject false order-status events. A global platform secret was considered but rejected because a single leaked secret would compromise all customers simultaneously. Per-endpoint secrets limit the blast radius of any individual leak.

## Alternatives Considered
- **Single global platform secret:** Rejected because a single leaked secret would compromise all webhook consumers simultaneously; per-endpoint isolation limits blast radius. ([09:20] Sofia)
- **API key in header (no signature):** Not explicitly discussed; HMAC was chosen because it signs the payload content, preventing tampering even if the key is known but the payload is intercepted.
- **No authentication (IP allowlisting only):** Not discussed; the team proceeded directly to HMAC as the standard approach.

## Consequences
**Positive:**
- Per-endpoint secret isolation: one leaked secret does not affect other customers
- HMAC-SHA256 signs the full payload — recipients can verify both authenticity and integrity
- 24h grace period prevents delivery failures during planned secret rotations
- Follows industry standard (same pattern as Stripe, GitHub webhooks)

**Negative / Trade-offs:**
- `webhook_endpoints` table must store secrets securely (encrypted at rest recommended)
- During grace period, the worker must attempt verification with both old and new secrets — adds complexity to the rotation logic
- Customers must implement HMAC verification on their end; increases integration complexity for receivers

## Related Modules
- MOD-008 (Webhooks) — owns `webhook_endpoints` table and signing logic in `WebhookProcessor`
- MOD-004 (Customers) — `webhook_endpoints.customerId` FK references the `customers` table
