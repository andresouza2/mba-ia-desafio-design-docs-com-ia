# Explicit State Machine for OrderStatus Transitions

## Score
Total: 140/150 | Step 0: 75 | Scope+Impact: 20/25 | Cost to Change: 22/25 | Team Knowledge: 23/25

## Evidence
- File: `src/modules/orders/order.status.ts` — frozen `transitions` map (`Record<OrderStatus, ReadonlyArray<OrderStatus>>`), `canTransition(from, to)`, `allowedTransitions(from)`, `isTerminal(status)`, `shouldDebitStock(from, to)`, `shouldReplenishStock(from, to)`
- File: `src/modules/orders/order.service.ts` — `changeStatus` calls `canTransition` guard before any mutation; throws `InvalidStatusTransitionError` on invalid transitions
- Transcript: `[09:04] Bruno` — stock side-effects and state transitions are co-located in the same operation

## Decision Summary
All valid `OrderStatus` transitions are encoded in a single, frozen `transitions` map in `src/modules/orders/order.status.ts`. Guard functions (`canTransition`, `isTerminal`) and stock-side-effect predicates (`shouldDebitStock`, `shouldReplenishStock`) are co-located in the same file, making the complete transition graph and its business consequences visible in one place. `OrderService.changeStatus` is the sole entry point for any status mutation and always enforces the guard.

## Context
Without an explicit state machine, any code path could call a Prisma update directly and place an order in an invalid state (e.g., DELIVERED → PENDING). The state machine makes invalid transitions impossible at the application level, not just at the database level, and makes the business rules self-documenting.

## Alternatives Considered
- **Ad-hoc if/switch in the service layer:** Rejected — business logic would be scattered across methods, making it easy to miss a transition check and hard to audit the full allowed graph.
- **Database constraints (CHECK constraints on status column):** Insufficient alone — cannot enforce directed transitions (e.g., cannot prevent DELIVERED → PENDING at the DB level without complex triggers).

## Consequences
**Positive:**
- The full transition graph is readable in one file; new transitions require only a map update
- `canTransition` can be called by any code path (service, tests) without duplicating logic
- Stock side-effect predicates (`shouldDebitStock`, `shouldReplenishStock`) are co-located with transition logic, reducing the risk of missed side effects

**Negative / Trade-offs:**
- The transition map must be kept in sync with the Prisma `OrderStatus` enum; adding a new enum value without updating the map causes a silent gap
- Business logic (stock side-effects) is coupled to the state file, not the service — a new developer must know to look in `order.status.ts` for stock rules

## Related Modules
- MOD-007 (Orders) — owns the state machine and its enforcement
- MOD-008 (Webhooks) — `publishWebhookEvent` is called after a successful transition; depends on the same `canTransition` guard running first
