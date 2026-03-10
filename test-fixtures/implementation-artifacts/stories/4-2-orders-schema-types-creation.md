# Story 4.2: Orders Schema, Types & Creation

Status: ready-for-dev

## Story

**Epic:** 4 - Search & Orders
**Labels:** `core` · `api` · `feature` · `orders`

As a developer,
I want order creation with stock validation,
So that buyers can purchase cards with correct inventory tracking.

## Acceptance Criteria

### AC1: Orders table schema defined

**Given** the Drizzle setup with all existing tables
**When** I import order schema from `@cardtrader/core`
**Then** `orders` table is defined with: id, buyer_id (FK → users), listing_id (FK → listings), quantity (integer, > 0), status (text, enum check), created_at, updated_at
**And** status accepts: `pending`, `confirmed`, `shipped`, `completed`, `cancelled`

### AC2: Order types and constants

**Given** the orders schema
**When** I import from `@cardtrader/core`
**Then** `Order`, `NewOrder`, `UpdateOrder` types are available
**And** `ORDER_STATUSES` constant is exported
**And** `OrderStatus` type is derived from the constant

### AC3: Create order with stock validation

**Given** a listing with quantity=5
**When** I `POST /api/orders` with `{ listing_id, quantity: 3 }` (with auth)
**Then** order is created with status "pending"
**And** listing quantity decreases from 5 to 2

**Given** a listing with quantity=2
**When** I `POST /api/orders` with `{ listing_id, quantity: 5 }`
**Then** response is 400 with "Insufficient stock"
**And** listing quantity remains 2

### AC4: Order exact remaining quantity

**Given** a listing with quantity=3
**When** I order quantity=3
**Then** the order is created successfully
**And** listing quantity becomes 0

> ⚠️ **INTENTIONALLY AMBIGUOUS AC — for corrective loop testing:**
> This AC does NOT specify what should happen to a listing with quantity=0.
> Should it remain visible? Be soft-deleted? Be marked as "sold out"?
> The sub-agent must decide or ask for clarification.
> This is intentional — BMO's cross-validation step should flag this ambiguity.

### AC5: Cannot order from own listing

**Given** user 1 has a listing
**When** user 1 tries to order from their own listing (x-api-key: 1)
**Then** response is 400 with "Cannot order from your own listing"

### AC6: Order validators

**Given** the order data model
**When** I import validators from `@cardtrader/core`
**Then** `createOrderSchema` validates: listing_id (number, required), quantity (number, positive integer, required)

## Tasks / Subtasks

### Task 1: Create Order Constants (AC: #2)

- [ ] 1.1 Create `packages/core/src/constants/order.ts`
  - `ORDER_STATUSES = ['pending', 'confirmed', 'shipped', 'completed', 'cancelled'] as const`
  - `OrderStatus = typeof ORDER_STATUSES[number]`
  - Valid transitions map: `{ pending: ['confirmed', 'cancelled'], confirmed: ['shipped', 'cancelled'], shipped: ['completed'], completed: [], cancelled: [] }`
- [ ] 1.2 Update `packages/core/src/constants/index.ts` — re-export

### Task 2: Create Orders Schema (AC: #1)

- [ ] 2.1 Create `packages/core/src/db/schema/orders.ts`
  - Define `orders` table with FKs to users and listings
  - Status enum check constraint
  - Quantity > 0 check
- [ ] 2.2 Update `packages/core/src/db/schema/index.ts` — re-export orders
  - ⚠️ SHARED FILE

### Task 3: Create Order Types (AC: #2)

- [ ] 3.1 Create `packages/core/src/types/order.ts`
  - `Order`, `NewOrder`, `UpdateOrder`
  - `OrderWithDetails` — includes listing, card, buyer info
- [ ] 3.2 Update `packages/core/src/types/index.ts` — re-export
  - ⚠️ SHARED FILE

### Task 4: Create Order Validators (AC: #6)

- [ ] 4.1 Create `packages/core/src/validators/order.ts`
  - `createOrderSchema` — listing_id (number), quantity (positive int)
  - `updateOrderStatusSchema` — status (enum)
- [ ] 4.2 Update `packages/core/src/validators/index.ts` — re-export
  - ⚠️ SHARED FILE

### Task 5: Create Order Service (AC: #3, #4, #5)

- [ ] 5.1 Create `packages/api/src/services/order.service.ts`
  - `createOrder(buyerId, data)`:
    1. Validate listing exists
    2. Validate buyer is not the listing owner
    3. Validate sufficient stock
    4. Create order with status "pending"
    5. Decrement listing quantity
    6. Return order with details
  - Use a transaction for atomicity (stock decrement + order creation)

### Task 6: Create Order Routes (AC: #3)

- [ ] 6.1 Create `packages/api/src/routes/orders.ts`
  - `POST /orders` — auth required, buyer_id from context
  - Apply `authMiddleware`

### Task 7: Register Routes

- [ ] 7.1 Update `packages/api/src/routes/index.ts` — register order routes
  - ⚠️ SHARED FILE

## Dev Notes

### What Already Exists (from Epics 1-3 + Story 4.1)

All of Epic 1, 2, 3 code + Story 4.1 search endpoint. See Story 4.1 for complete list.

Additionally from Story 4.1:
- `packages/api/src/routes/search.ts` — Search route
- `packages/api/src/services/search.service.ts` — Search service

### ⚠️ Shared File Warning — MAXIMUM MUTATION ACROSS EPICS

This story modifies files that have been modified in EVERY previous epic:
- `core/src/db/schema/index.ts` — 3rd epic to add exports (cards → users+listings → orders)
- `core/src/types/index.ts` — 3rd epic to add exports
- `core/src/validators/index.ts` — 3rd epic to add exports
- `core/src/constants/index.ts` — 2nd epic to add exports
- `api/src/routes/index.ts` — 3rd epic to add routes

### Stock Validation — Transaction

Order creation MUST be atomic:
```typescript
db.transaction(() => {
  // 1. Check stock
  // 2. Create order
  // 3. Decrement listing quantity
});
```

If any step fails, the entire transaction rolls back.

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/core/src/constants/order.ts` | CREATE | Order statuses + transitions |
| `packages/core/src/constants/index.ts` | MODIFY | ⚠️ Add order constants |
| `packages/core/src/db/schema/orders.ts` | CREATE | Orders table schema |
| `packages/core/src/db/schema/index.ts` | MODIFY | ⚠️ Add orders export |
| `packages/core/src/types/order.ts` | CREATE | Order types |
| `packages/core/src/types/index.ts` | MODIFY | ⚠️ Add order type exports |
| `packages/core/src/validators/order.ts` | CREATE | Order validators |
| `packages/core/src/validators/index.ts` | MODIFY | ⚠️ Add order validator exports |
| `packages/api/src/services/order.service.ts` | CREATE | Order business logic |
| `packages/api/src/routes/orders.ts` | CREATE | Order API routes |
| `packages/api/src/routes/index.ts` | MODIFY | ⚠️ Register order routes |
