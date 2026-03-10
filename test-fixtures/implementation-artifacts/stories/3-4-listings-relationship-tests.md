# Story 3.4: Listings Relationship Tests

Status: ready-for-dev

## Story

**Epic:** 3 - Listings & Users
**Labels:** `api` · `test` · `listings`

As a developer,
I want comprehensive tests for listings including relationships and auth,
So that I can verify listings work correctly with foreign keys.

## Acceptance Criteria

### AC1: CRUD tests with auth

**Given** the listings API with auth middleware
**When** I test listing operations
**Then** create/update/delete without auth header → 401
**And** create with auth header → 201 (user_id from context)
**And** list and detail are public (no auth needed)

### AC2: FK validation and Zod validation tests

**Given** the listings API validates foreign keys
**When** I create listing with nonexistent card_id
**Then** response is 400 with "Card not found"

**When** I create listing with nonexistent user (invalid API key)
**Then** appropriate error response

**Given** the listings API validates input via Zod
**When** I create listing with missing price
**Then** response is 400 with validation error
**And** invalid condition enum (e.g., "PERFECT") returns 400
**And** negative price returns 400
**And** negative quantity returns 400

**Given** a nonexistent listing id
**When** I `GET /api/listings/999`
**Then** response is 404

### AC3: Filter tests

**Given** listings exist for multiple cards and users
**When** I filter by card_id, user_id, or condition
**Then** only matching listings are returned

### AC4: Relationship detail tests

**Given** a listing exists with card_id=1 and user_id=1
**When** I `GET /api/listings/1`
**Then** response includes card name and user username

### AC5: Zero regressions

**Given** all previous tests from Epics 1 + 2
**When** I run `npm test`
**Then** health, card, and listing tests all pass

## Tasks / Subtasks

### Task 1: Create Listing Test File (AC: #1, #2, #3, #4)

- [ ] 1.1 Create `packages/api/src/__tests__/listings.test.ts`
  - Setup: create in-memory DB, seed cards + users + listings
  - Test groups:
    - **Auth:** create without header (401), create with header (201)
    - **CRUD:** create, list, detail, update, delete
    - **Filters:** by card_id, by user_id, by condition
    - **FK validation:** invalid card_id (400), valid card + user (201)
    - **Zod validation:** missing price (400), invalid condition (400), negative price (400), negative quantity (400)
    - **Not found:** GET /api/listings/999 → 404
    - **Detail with relationships:** card name + user username in response

### Task 2: Full Verification (AC: #5)

- [ ] 2.1 Run `npm test` — all tests pass (health + cards + listings)
- [ ] 2.2 Run `npm run lint` — zero warnings
- [ ] 2.3 Run `npm install && npm test` — full verification

## Dev Notes

### What Already Exists (from Epics 1 + 2 + Stories 3.1-3.3)

- `packages/api/src/__tests__/health.test.ts` — Health test
- `packages/api/src/__tests__/cards.test.ts` — Card tests
- `packages/core/src/db/seed.ts` — Card seed data
- `packages/api/src/middleware/auth.ts` — Auth middleware
- `packages/api/src/routes/listings.ts` — Listing routes
- `packages/api/src/services/listing.service.ts` — Listing service

### Test setup complexity

Listing tests require more setup than card tests:
1. Create in-memory DB
2. Seed cards (from existing seed)
3. Create test users
4. Create test listings
5. Then run assertions

Consider creating a `test-utils.ts` helper for common setup.

### Test count target

13-18 tests:
- 2 auth tests (401 without, 201 with)
- 3 filter tests (by card, by user, by condition)
- 2 FK tests (invalid card, valid)
- 4 Zod validation tests (missing price, invalid condition, negative price, negative quantity)
- 1 not-found test (404)
- 1 detail test (with relationships)
- 2-3 CRUD tests (create, update, delete)
- 1-2 edge cases

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/api/src/__tests__/listings.test.ts` | CREATE | Listing API tests |
