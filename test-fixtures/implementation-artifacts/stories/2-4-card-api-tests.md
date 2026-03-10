# Story 2.4: Card API Tests

Status: ready-for-dev

## Story

**Epic:** 2 - Card Catalog
**Labels:** `api` · `test` · `catalog`

As a developer,
I want comprehensive tests for all card endpoints,
So that I can verify the catalog works correctly.

## Acceptance Criteria

### AC1: CRUD tests pass

**Given** the cards API is complete
**When** I run `npm test`
**Then** tests pass for: list cards, get card by id, create card, update card, delete card

### AC2: Validation tests pass

**Given** the cards API validates input
**When** I send invalid payloads
**Then** tests verify: missing name returns 400, invalid rarity returns 400, empty body returns 400

### AC3: Pagination tests pass

**Given** cards are seeded in the test database
**When** I request `GET /api/cards?page=1&pageSize=5`
**Then** tests verify pagination meta is correct (page, pageSize, total, totalPages)

### AC4: Error case tests pass

**Given** the cards API handles errors
**When** I request nonexistent cards
**Then** tests verify: GET /api/cards/999 returns 404

### AC5: Zero regressions

**Given** all previous tests from Epic 1
**When** I run `npm test`
**Then** health endpoint test still passes
**And** all new card tests pass

## Tasks / Subtasks

### Task 1: Create Card Test File (AC: #1, #2, #3, #4)

- [ ] 1.1 Create `packages/api/src/__tests__/cards.test.ts`
  - Import app from `../app`
  - Import `seedCards` from `@cardtrader/core/db/seed`
  - Use `beforeEach` to set up in-memory DB with seed data
  - Test groups:
    - **List cards:** default pagination, custom pagination, empty list
    - **Get card by id:** existing card (200), nonexistent card (404)
    - **Create card:** valid payload (201), missing name (400), invalid rarity (400)
    - **Update card:** valid update (200), nonexistent card (404)
    - **Delete card:** existing card (204), verify deleted (404 on re-fetch)

### Task 2: Full Verification (AC: #5)

- [ ] 2.1 Run `npm test` — all tests pass including health.test.ts
- [ ] 2.2 Run `npm run lint` — zero warnings
- [ ] 2.3 Run `npm install && npm test` from clean state — passes

## Dev Notes

### What Already Exists (from Epic 1 + Stories 2.1-2.3)

- `packages/api/src/app.ts` — Hono app with routes
- `packages/api/src/routes/cards.ts` — Card CRUD routes
- `packages/api/src/services/card.service.ts` — Card service
- `packages/api/src/__tests__/health.test.ts` — Health test (must not break)
- `packages/core/src/db/test-helpers.ts` — In-memory DB helper
- `packages/core/src/db/seed.ts` — Seed data + `seedCards()` function

### Testing pattern

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import app from '../app';

describe('Cards API', () => {
  beforeEach(async () => {
    // Reset in-memory DB and seed
  });

  describe('GET /api/cards', () => {
    it('returns paginated list', async () => {
      const res = await app.request('/api/cards');
      expect(res.status).toBe(200);
      const body = await res.json();
      expect(body.data).toBeInstanceOf(Array);
      expect(body.meta.pagination).toBeDefined();
    });
  });
});
```

### Test count target

8-12 tests covering:
- 2-3 list tests (default, custom pagination, empty)
- 2 get tests (found, not found)
- 3 create tests (valid, missing name, invalid rarity)
- 1 update test
- 2 delete tests (success, re-verify)

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/api/src/__tests__/cards.test.ts` | CREATE | Card API tests |
