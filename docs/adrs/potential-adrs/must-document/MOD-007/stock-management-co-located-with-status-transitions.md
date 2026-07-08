# Stock Management Co-located with Status Transitions

## Score
Total: 136/150 | Step 0: 75 | Scope+Impact: 18/25 | Cost to Change: 20/25 | Team Knowledge: 23/25

## Evidence
- File: `src/modules/orders/order.status.ts` — `shouldDebitStock(from, to)` (true only PENDING→PAID) and `shouldReplenishStock(from, to)` (true for PAID/PROCESSING→CANCELLED) exported from the state machine file
- File: `src/modules/orders/order.service.ts` — `debitStock(tx, items)` and `replenishStock(tx, items)` called inside `changeStatus` based on these predicates; stock validation (checks `stockQuantity` before decrement) happens inside `debitStock`
- File: `src/modules/products/product.repository.ts` — has no stock logic; `ProductRepository` is a pure CRUD repository
- Transcript: `[09:04] Bruno` — "decrementa stock_quantity dos produtos do pedido" listed as part of the status-change transaction

## Decision Summary
Stock quantity mutations (debit and replenish) are triggered exclusively by order status transitions and are co-located with the state machine logic. The predicates `shouldDebitStock` and `shouldReplenishStock` in `order.status.ts` encode which transitions cause stock side-effects. `OrderService.changeStatus` evaluates these predicates and calls the appropriate stock mutation function inside the same `prisma.$transaction`. `ProductRepository` has no stock management methods — stock is only ever modified by the Orders module.

## Context
Decoupling stock mutations from the transition logic would risk inconsistencies: a developer adding a new transition might forget to wire the stock side-effect. By placing the predicates in `order.status.ts` alongside the transition map, all business consequences of a transition are visible in one file.

## Alternatives Considered
- **Stock mutations in ProductService/ProductRepository:** Rejected — would require the Orders module to call ProductService during the transaction, creating a bidirectional module dependency and hiding the connection between transition logic and stock effects.
- **Database triggers on order_status_history INSERT:** Rejected — triggers create hidden side effects invisible to the application layer and cannot participate in the Prisma transaction rollback.

## Consequences
**Positive:**
- All business consequences of a status transition (stock, history, outbox) are enforced in one place (`changeStatus`)
- Stock predicates in `order.status.ts` make it impossible to add a transition without consciously deciding whether it has stock side-effects
- `InsufficientStockError` is thrown inside the transaction, rolling back the entire operation cleanly

**Negative / Trade-offs:**
- `OrderService` directly writes to the `products` table via `tx.product.update` — bypassing `ProductRepository`; this cross-module Prisma access is a known pattern but must be understood by maintainers
- Adding stock-affecting transitions in the future requires updating both the `transitions` map and the predicate functions

## Related Modules
- MOD-007 (Orders) — owns all stock mutation logic
- MOD-006 (Products) — `products.stockQuantity` is mutated by MOD-007 inside transactions
