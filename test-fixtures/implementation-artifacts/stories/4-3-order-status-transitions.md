# Story 4.3: Order Status Transitions & Listing

Status: ready-for-dev

## Story

**Epic:** 4 - Search & Orders
**Labels:** `api` · `feature` · `orders`

As a developer,
I want order status transitions and order listing,
So that orders can progress through their lifecycle.

## Acceptance Criteria

### AC1: Valid status transitions

**Given** an order with status "pending"
**When** I `PATCH /api/orders/:id/status` with `{ status: "confirmed" }`
**Then** status changes to "confirmed"

**Given** an order with status "confirmed"
**When** I transition to "shipped"
**Then** status changes to "shipped"

**Given** an order with status "shipped"
**When** I transition to "completed"
**Then** status changes to "completed"

### AC2: Invalid transitions rejected

**Given** an order with status "completed"
**When** I attempt to transition to "pending"
**Then** response is 400 with `{ error: { code: "BAD_REQUEST", message: "Invalid status transition from completed to pending" } }`

**Given** an order with status "cancelled"
**When** I attempt to transition to "confirmed"
**Then** response is 400 with "Invalid status transition"

### AC3: Cancellation restores stock

**Given** an order with status "pending" and quantity=3, and the listing now has quantity=2
**When** I cancel the order
**Then** order status changes to "cancelled"
**And** listing quantity increases from 2 to 5 (restored)

**Given** an order with status "confirmed" and quantity=2
**When** I cancel the order
**Then** order status changes to "cancelled"
**And** listing quantity is restored

### AC4: List orders by buyer

**Given** orders exist for multiple buyers
**When** I `GET /api/orders?buyer_id=1`
**Then** only buyer 1's orders are returned

**When** I `GET /api/orders` (with auth)
**Then** only the authenticated user's orders are returned

### AC5: Order detail

**Given** an order with id=1 exists
**When** I `GET /api/orders/1`
**Then** response includes order details, listing info, card name, buyer info

**Given** no order with id=999 exists
**When** I `GET /api/orders/999`
**Then** response is 404

## Tasks / Subtasks

### Task 1: Add Status Transition Logic (AC: #1, #2)

- [ ] 1.1 Update `packages/api/src/services/order.service.ts`
  - `updateOrderStatus(orderId, newStatus)`:
    1. Get current order
    2. Validate transition is allowed (using `VALID_TRANSITIONS` map)
    3. Update status + updated_at
    4. If cancelling: restore listing stock
  - Import `VALID_TRANSITIONS` from `@cardtrader/core/constants`

### Task 2: Add Cancellation Stock Restoration (AC: #3)

- [ ] 2.1 In `updateOrderStatus`, when new status is "cancelled":
  - Increment listing quantity by order quantity
  - Use transaction for atomicity

### Task 3: Add Order Listing (AC: #4)

- [ ] 3.1 Update `packages/api/src/services/order.service.ts`
  - `listOrders(buyerId)` — returns orders for buyer with listing/card details
  - `getOrderById(orderId)` — returns order with full details

### Task 4: Add Order Routes (AC: #1, #4, #5)

- [ ] 4.1 Update `packages/api/src/routes/orders.ts`
  - `PATCH /orders/:id/status` — auth required, validate transition
  - `GET /orders` — auth required, filter by authenticated user
  - `GET /orders/:id` — auth required

## Dev Notes

### What Already Exists (from Story 4.2)

- `packages/core/src/constants/order.ts` — `ORDER_STATUSES`, `VALID_TRANSITIONS`
- `packages/core/src/db/schema/orders.ts` — Orders table
- `packages/core/src/validators/order.ts` — `createOrderSchema`, `updateOrderStatusSchema`
- `packages/api/src/services/order.service.ts` — `createOrder()` (this story adds more methods)
- `packages/api/src/routes/orders.ts` — `POST /orders` (this story adds more routes)

### Valid Transitions Map

```
pending    → confirmed, cancelled
confirmed  → shipped, cancelled
shipped    → completed
completed  → (terminal)
cancelled  → (terminal)
```

### Cancellation is special

Only `pending` and `confirmed` orders can be cancelled. `shipped` cannot (it must complete). Stock restoration MUST be in a transaction.

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/api/src/services/order.service.ts` | MODIFY | Add transitions, listing, detail |
| `packages/api/src/routes/orders.ts` | MODIFY | Add PATCH, GET routes |
