# Story 2.2: Cards CRUD API

Status: ready-for-dev

## Story

**Epic:** 2 - Card Catalog
**Labels:** `api` · `feature` · `catalog`

As a developer,
I want full CRUD endpoints for cards,
So that cards can be managed in the catalog.

## Acceptance Criteria

### AC1: List cards with pagination

**Given** cards exist in the database
**When** I request `GET /api/cards?page=1&pageSize=10`
**Then** response is 200 with `{ data: [...], meta: { pagination: { page: 1, pageSize: 10, total, totalPages } } }`
**And** default pagination is page=1, pageSize=20

### AC2: Get card by ID

**Given** a card with id=1 exists
**When** I request `GET /api/cards/1`
**Then** response is 200 with `{ data: { id: 1, name, set, rarity, ... }, meta: { timestamp } }`

**Given** no card with id=999 exists
**When** I request `GET /api/cards/999`
**Then** response is 404 with `{ error: { code: "NOT_FOUND", message: "Card not found", status: 404 } }`

### AC3: Create card with validation

**Given** a valid card payload `{ name: "Black Lotus", set: "Alpha", rarity: "mythic" }`
**When** I request `POST /api/cards`
**Then** response is 201 with the created card data
**And** the card has an auto-generated id, created_at, updated_at

**Given** an invalid payload `{ set: "Alpha" }` (missing name)
**When** I request `POST /api/cards`
**Then** response is 400 with validation error details

**Given** an invalid rarity `{ name: "Test", set: "X", rarity: "legendary" }`
**When** I request `POST /api/cards`
**Then** response is 400 with rarity validation error

### AC4: Update card

**Given** a card with id=1 exists
**When** I request `PUT /api/cards/1` with `{ name: "Updated Name" }`
**Then** response is 200 with updated card data
**And** updated_at is refreshed

### AC5: Delete card

**Given** a card with id=1 exists
**When** I request `DELETE /api/cards/1`
**Then** response is 204 (no content)
**And** the card is no longer retrievable

## Tasks / Subtasks

### Task 1: Create Card Service (AC: #1, #2, #3, #4, #5)

- [ ] 1.1 Create `packages/api/src/services/card.service.ts`
  - `listCards(page, pageSize)` → returns paginated cards
  - `getCardById(id)` → returns card or null
  - `createCard(data)` → inserts and returns card
  - `updateCard(id, data)` → updates and returns card
  - `deleteCard(id)` → deletes card

### Task 2: Create Card Routes (AC: #1, #2, #3, #4, #5)

- [ ] 2.1 Create `packages/api/src/routes/cards.ts`
  - `GET /cards` — list with pagination query params
  - `GET /cards/:id` — get by ID
  - `POST /cards` — create with Zod validation
  - `PUT /cards/:id` — update with Zod validation
  - `DELETE /cards/:id` — delete
  - Use `createCardSchema` and `updateCardSchema` from core

### Task 3: Register Card Routes (AC: #1)

- [ ] 3.1 Update `packages/api/src/routes/index.ts`
  - Import and register card routes under `/cards`
  - ⚠️ SHARED FILE — add to existing health route registration

### Task 4: API Response Helpers

- [ ] 4.1 Create `packages/api/src/utils/response.ts`
  - `success(data)` → `{ data, meta: { timestamp } }`
  - `paginated(data, pagination)` → `{ data, meta: { timestamp, pagination } }`
  - `error(code, message, status)` → `{ error: { code, message, status } }`

## Dev Notes

### What Already Exists (from Epic 1)

- `packages/api/src/app.ts` — Hono app instance
- `packages/api/src/routes/index.ts` — Route registry with health route
- `packages/api/src/routes/health.ts` — Health endpoint
- `packages/api/src/middleware/error-handler.ts` — Error handling

### From Story 2.1

- `packages/core/src/db/schema/cards.ts` — Cards table schema
- `packages/core/src/types/card.ts` — Card, NewCard, UpdateCard types
- `packages/core/src/types/api.ts` — ApiResponse, ApiListResponse types
- `packages/core/src/validators/card.ts` — createCardSchema, updateCardSchema

### ⚠️ Shared File Warning

Modifies `api/src/routes/index.ts` to add card routes. Epic 3 will add more routes to this same file.

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/api/src/services/card.service.ts` | CREATE | Card business logic |
| `packages/api/src/routes/cards.ts` | CREATE | Card API routes |
| `packages/api/src/routes/index.ts` | MODIFY | Register card routes |
| `packages/api/src/utils/response.ts` | CREATE | Response helpers |
