# Story 4.1: Search Endpoint

Status: ready-for-dev

## Story

**Epic:** 4 - Search & Orders
**Labels:** `api` · `feature` · `search`

As a developer,
I want a search endpoint that finds cards by name with filters,
So that buyers can discover cards in the catalog.

## Acceptance Criteria

### AC1: Search by name

**Given** cards with "Dragon" in the name exist (e.g., "Shivan Dragon", "Dragon Whelp", "Dragonlord Atarka")
**When** I request `GET /api/search?q=dragon`
**Then** all cards matching "dragon" (case-insensitive) are returned
**And** results are wrapped in standard list response format

### AC2: Search results include listing data

**Given** a card "Shivan Dragon" has 3 listings with prices $5, $8, $12
**When** I search and find "Shivan Dragon"
**Then** the result includes `listing_count: 3` and `min_price: 5`

**Given** a card "Dragon Whelp" has 0 listings
**When** I search and find "Dragon Whelp"
**Then** the result includes `listing_count: 0` and `min_price: null`

### AC3: Filter by set and rarity

**Given** cards exist across multiple sets and rarities
**When** I request `GET /api/search?q=dragon&set=Alpha`
**Then** only dragons from "Alpha" set are returned

**When** I request `GET /api/search?q=dragon&rarity=rare`
**Then** only rare dragons are returned

### AC4: Search validation

**Given** the search endpoint
**When** I request `GET /api/search` without `q` parameter
**Then** response is 400 with "Search query is required"

### AC5: Search pagination

**Given** many cards match the search
**When** I request `GET /api/search?q=dragon&page=1&pageSize=5`
**Then** results are paginated with correct meta

## Tasks / Subtasks

### Task 1: Create Search Service (AC: #1, #2, #3)

- [ ] 1.1 Create `packages/api/src/services/search.service.ts`
  - `searchCards(query, filters)` — search by name using LIKE
  - Join with listings to get count + min price per card
  - Apply optional set and rarity filters
  - Return paginated results

### Task 2: Create Search Validators

- [ ] 2.1 Create `packages/core/src/validators/search.ts`
  - `searchQuerySchema` — q (string, required, min 1), set (optional), rarity (optional enum), page (optional), pageSize (optional)
- [ ] 2.2 Update `packages/core/src/validators/index.ts` — re-export

### Task 3: Create Search Route (AC: #1, #4, #5)

- [ ] 3.1 Create `packages/api/src/routes/search.ts`
  - `GET /search` — validate query, call search service, return paginated results
  - Public endpoint (no auth)

### Task 4: Register Route

- [ ] 4.1 Update `packages/api/src/routes/index.ts` — register search route
  - ⚠️ SHARED FILE

## Dev Notes

### What Already Exists (from Epics 1 + 2 + 3)

This is the **resume/retomar** scenario. BMO enters with 3 complete epics:

**Core package:**
- `packages/core/src/db/schema/index.ts` — Exports: cards, users, listings
- `packages/core/src/db/schema/cards.ts` — Cards table
- `packages/core/src/db/schema/users.ts` — Users table
- `packages/core/src/db/schema/listings.ts` — Listings table with FKs
- `packages/core/src/types/index.ts` — Exports: Card, User, Listing, Api types
- `packages/core/src/validators/index.ts` — Exports: card, user, listing validators
- `packages/core/src/constants/index.ts` — Card rarities + conditions
- `packages/core/src/db/seed.ts` — Card seed data (20+ cards including "dragons")
- `packages/core/src/db/test-helpers.ts` — In-memory DB helper

**API package:**
- `packages/api/src/app.ts` — Hono app
- `packages/api/src/routes/index.ts` — Registers: health, cards, users, listings
- `packages/api/src/routes/health.ts`, `cards.ts`, `users.ts`, `listings.ts`
- `packages/api/src/services/card.service.ts`, `user.service.ts`, `listing.service.ts`
- `packages/api/src/middleware/auth.ts` — API key auth
- `packages/api/src/middleware/error-handler.ts`
- `packages/api/src/utils/response.ts`

**All tests passing:** health, cards (8-12), users (7-10), listings (10-15)

### SQL Query Pattern for Search

```sql
SELECT c.*, 
  COUNT(l.id) as listing_count,
  MIN(l.price) as min_price
FROM cards c
LEFT JOIN listings l ON l.card_id = c.id
WHERE c.name LIKE '%dragon%'
GROUP BY c.id
```

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/api/src/services/search.service.ts` | CREATE | Search business logic |
| `packages/core/src/validators/search.ts` | CREATE | Search validators |
| `packages/core/src/validators/index.ts` | MODIFY | ⚠️ Add search validators |
| `packages/api/src/routes/search.ts` | CREATE | Search API route |
| `packages/api/src/routes/index.ts` | MODIFY | ⚠️ Register search route |
