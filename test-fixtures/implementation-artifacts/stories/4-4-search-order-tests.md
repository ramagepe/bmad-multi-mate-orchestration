# Story 4.4: Search & Order Tests

Status: ready-for-dev

## Story

**Epic:** 4 - Search & Orders
**Labels:** `api` · `test` · `search` · `orders`

As a developer,
I want comprehensive tests for search and order flows,
So that I can verify the complete marketplace works.

## Acceptance Criteria

### AC1: Search tests pass

**Given** cards and listings are seeded
**When** I run search tests
**Then** tests cover: search by name, filter by set, filter by rarity, empty results, listing aggregation (count + min price), pagination

### AC2: Order creation tests pass

**Given** listings with known quantities exist
**When** I run order creation tests
**Then** tests cover: successful creation (stock decremented), insufficient stock (400), order from own listing (400), nonexistent listing (404)
**And** tests verify unauthenticated POST /api/orders returns 401
**And** tests verify Zod validation: missing listing_id (400), missing quantity (400), zero/negative quantity (400)

### AC3: Status transition tests pass

**Given** orders in various states
**When** I run transition tests
**Then** tests cover: valid transitions (pending→confirmed→shipped→completed), invalid transitions (completed→pending, shipped→cancelled), cancellation restores stock

### AC4: Order listing tests pass

**Given** orders exist for multiple buyers
**When** I run listing tests
**Then** tests cover: list by buyer, order detail with card/listing info, nonexistent order (404)

### AC5: Edge cases covered

**Given** edge case scenarios
**When** I run edge case tests
**Then** tests cover: order exact remaining quantity (quantity→0), cancel already-cancelled order (400), double-create race condition (sequential test)

### AC6: Full verification passes

**Given** the complete CardTrader API
**When** I run `npm install && npm test` from a clean checkout
**Then** ALL tests pass across ALL epics (health, cards, users, listings, search, orders)
**And** `npm run lint` passes
**And** `npm run typecheck` passes
**And** exit code is 0

## Tasks / Subtasks

### Task 1: Create Search Tests (AC: #1)

- [ ] 1.1 Create `packages/api/src/__tests__/search.test.ts`
  - Setup: in-memory DB, seed cards (include "Dragon" cards) + listings
  - Tests:
    - Search by name — finds matching cards
    - Case-insensitive search
    - Filter by set — narrows results
    - Filter by rarity — narrows results
    - Combined filters (set + rarity)
    - Empty results — valid response with empty array
    - Listing aggregation — count + min_price correct
    - Card with no listings — listing_count=0, min_price=null
    - Pagination — correct meta
    - Missing query parameter — 400

### Task 2: Create Order Tests (AC: #2, #3, #4, #5)

- [ ] 2.1 Create `packages/api/src/__tests__/orders.test.ts`
  - Setup: in-memory DB, seed cards + users + listings
  - Tests:
    - **Creation:** create order (201), verify stock decremented, insufficient stock (400), own listing (400), nonexistent listing (404)
    - **Transitions:** pending→confirmed (200), confirmed→shipped (200), shipped→completed (200), pending→cancelled with stock restore
    - **Auth:** POST /api/orders without auth header (401)
    - **Validation:** missing listing_id (400), missing quantity (400), quantity=0 (400)
    - **Invalid transitions:** completed→pending (400), cancelled→confirmed (400), shipped→cancelled (400)
    - **Listing:** list by buyer, order detail, nonexistent order (404)
    - **Edge cases:** exact remaining quantity (quantity→0), cancel already-cancelled (400)

### Task 3: Full Verification (AC: #6)

- [ ] 3.1 Run `npm test` — ALL tests pass (target: 55-70 total tests)
- [ ] 3.2 Run `npm run lint` — zero warnings
- [ ] 3.3 Run `npm run typecheck` — zero errors
- [ ] 3.4 Run `npm install && npm test` from clean state — passes with exit code 0
- [ ] 3.5 Verify test count per file:
  - health.test.ts: 1-2 tests
  - cards.test.ts: 8-12 tests
  - users.test.ts: 7-10 tests
  - listings.test.ts: 10-15 tests
  - search.test.ts: 8-10 tests
  - orders.test.ts: 14-18 tests

## Dev Notes

### What Already Exists (from Epics 1-3 + Stories 4.1-4.3)

Complete CardTrader API:
- All schemas: cards, users, listings, orders
- All services: card, user, listing, search, order
- All routes: health, cards, users, listings, search, orders
- Auth middleware
- All previous tests: health (1-2), cards (8-12), users (7-10), listings (10-15)

### Test setup for orders

Orders require the most setup:
1. Create in-memory DB
2. Seed cards
3. Create test users (buyer + seller)
4. Create test listings (with known quantities)
5. Create orders
6. Run assertions

Consider a shared `setupOrderTestData(db)` helper.

### Comprehensive test matrix

```
Search:  name, set, rarity, combined, empty, aggregation, no-listings, pagination, validation = ~10 tests
Orders:  auth(1), validation(3), create, stock-fail, own-listing, not-found, transitions(4), invalid-transitions(3), cancel-restore, list, detail, 404, edge-cases(2) = ~18 tests
```

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/api/src/__tests__/search.test.ts` | CREATE | Search API tests |
| `packages/api/src/__tests__/orders.test.ts` | CREATE | Order API tests |
