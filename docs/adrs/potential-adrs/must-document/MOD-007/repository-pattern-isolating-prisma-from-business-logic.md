# Repository Pattern Isolating All Prisma Calls from Business Logic

## Score
Total: 123/150 | Step 0: 75 | Scope+Impact: 22/25 | Cost to Change: 16/25 | Team Knowledge: 10/25

## Evidence
- File: `src/modules/orders/order.repository.ts` — `OrderRepository` class receives `PrismaClient` in constructor; exposes `list`, `findById`, `findByIdWithRelations`, `deleteById` with no business rules
- File: `src/modules/orders/order.service.ts` — `OrderService` receives `OrderRepository` and `PrismaClient` separately; direct `tx.*` calls inside transactions intentionally bypass the repository (known pattern per mapping.md)
- Mapping: "Cross-module access to data goes through the other module's service or repository."

## Decision Summary
Each domain module has a single `*Repository` class as the sole data access layer for non-transactional queries. Repositories receive a `PrismaClient` via constructor and expose named query methods. Services receive repository instances and contain all business logic. Inside transactions, services call `tx.*` directly rather than going through repositories, because repositories cannot participate in an externally-owned transaction client.

## Context
Without the repository layer, Prisma calls would be scattered across controllers and services, making them hard to find, mock in tests, and change when schema evolves. The repository provides a named, searchable boundary for all data access. The deliberate decision to bypass repositories inside transactions is documented: it is a known trade-off of the pattern, not an oversight.

## Alternatives Considered
- **Active Record pattern (models with built-in queries):** Prisma does not support Active Record; Prisma models are plain data objects.
- **Repository methods accepting optional TxClient:** Would require every repository method to have an optional `tx` parameter, complicating the interface. The team chose to keep repositories simple and use direct `tx.*` access inside transactions.

## Consequences
**Positive:**
- All non-transactional Prisma calls are discoverable in `*.repository.ts` files
- Repositories are independently testable with a mock `PrismaClient`
- Schema changes are localized: only the affected repository needs updating

**Negative / Trade-offs:**
- Inside transactions, services call `tx.product.update`, `tx.order.update`, etc. directly — bypassing the repository; this is an undocumented pattern that can surprise new contributors
- Two data access styles coexist (repository methods and direct `tx.*` calls), requiring developers to understand when each is appropriate

## Related Modules
- All modules (MOD-002 through MOD-008) — every module follows this repository pattern
- MOD-001 (Infrastructure) — `buildControllers(prisma)` is the composition root where repositories are instantiated
