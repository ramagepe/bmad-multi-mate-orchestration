# Story 2.1: Cards Schema & Types

Status: ready-for-dev

## Story

**Epic:** 2 - Card Catalog
**Labels:** `core` · `feature` · `catalog`

As a developer,
I want the cards table schema and TypeScript types defined,
So that the API can interact with card data.

## Acceptance Criteria

### AC1: Cards table schema defined

**Given** the Drizzle setup from Epic 1
**When** I import card schema from `@cardtrader/core`
**Then** `cards` table is defined with columns: id (integer PK autoincrement), name (text NOT NULL), set (text NOT NULL), rarity (text NOT NULL, enum check), image_url (text nullable), created_at (text DEFAULT now), updated_at (text DEFAULT now)
**And** rarity accepts only: `common`, `uncommon`, `rare`, `mythic`

### AC2: Card types available

**Given** the cards schema
**When** I import types from `@cardtrader/core`
**Then** `Card` type is inferred from Drizzle `$inferSelect`
**And** `NewCard` type is inferred from Drizzle `$inferInsert`
**And** `UpdateCard` is a `Partial<NewCard>` (all fields optional)

### AC3: Card validators available

**Given** the card data model
**When** I import validators from `@cardtrader/core`
**Then** `createCardSchema` validates: name (string, required), set (string, required), rarity (enum, required), image_url (string, optional)
**And** `updateCardSchema` validates same fields but all optional
**And** invalid rarity values are rejected

### AC4: Card constants available

**Given** the card domain
**When** I import constants from `@cardtrader/core`
**Then** `CARD_RARITIES` is `['common', 'uncommon', 'rare', 'mythic'] as const`
**And** `CARD_CONDITIONS` is `['NM', 'LP', 'MP', 'HP'] as const`
**And** `CardRarity` and `CardCondition` types are derived from these

## Tasks / Subtasks

### Task 1: Create Card Constants (AC: #4)

- [ ] 1.1 Create `packages/core/src/constants/card.ts`
  - `CARD_RARITIES = ['common', 'uncommon', 'rare', 'mythic'] as const`
  - `CARD_CONDITIONS = ['NM', 'LP', 'MP', 'HP'] as const`
  - `CardRarity = typeof CARD_RARITIES[number]`
  - `CardCondition = typeof CARD_CONDITIONS[number]`
- [ ] 1.2 Update `packages/core/src/constants/index.ts` — re-export card constants

### Task 2: Create Cards Schema (AC: #1)

- [ ] 2.1 Create `packages/core/src/db/schema/cards.ts`
  - Define `cards` table using Drizzle `sqliteTable`
  - Columns as specified in AC1
  - Rarity check constraint
- [ ] 2.2 Update `packages/core/src/db/schema/index.ts` — re-export cards schema

### Task 3: Create Card Types (AC: #2)

- [ ] 3.1 Create `packages/core/src/types/card.ts`
  - `Card = typeof cards.$inferSelect`
  - `NewCard = typeof cards.$inferInsert`
  - `UpdateCard = Partial<Omit<NewCard, 'id'>>`
- [ ] 3.2 Create `packages/core/src/types/api.ts`
  - `ApiResponse<T>` — `{ data: T, meta: { timestamp: string } }`
  - `ApiListResponse<T>` — `{ data: T[], meta: { timestamp: string, pagination: PaginationMeta } }`
  - `PaginationMeta` — `{ page, pageSize, total, totalPages }`
  - `ApiError` — `{ error: { code: string, message: string, status: number } }`
- [ ] 3.3 Update `packages/core/src/types/index.ts` — re-export all

### Task 4: Create Card Validators (AC: #3)

- [ ] 4.1 Install `zod` in `packages/core`
- [ ] 4.2 Create `packages/core/src/validators/card.ts`
  - `createCardSchema` — z.object with name, set, rarity (enum), image_url (optional)
  - `updateCardSchema` — createCardSchema.partial()
- [ ] 4.3 Update `packages/core/src/validators/index.ts` — re-export

### Task 5: Verify (AC: #1, #2, #3, #4)

- [ ] 5.1 Run `npm run typecheck` — zero errors
- [ ] 5.2 Run `npm test` — Epic 1 tests still pass
- [ ] 5.3 Run `npm run lint` — zero warnings

## Dev Notes

### What Already Exists (from Epic 1)

- `packages/core/src/db/index.ts` — DB connection with `getDb()` factory
- `packages/core/src/db/schema/index.ts` — Empty barrel export
- `packages/core/src/types/index.ts` — Empty barrel export
- `packages/core/src/validators/index.ts` — Empty barrel export
- `packages/core/src/constants/index.ts` — Empty barrel export
- `packages/core/src/db/test-helpers.ts` — In-memory test DB helper

### ⚠️ Shared File Warning

This story MODIFIES these shared files by adding exports:
- `core/src/db/schema/index.ts` — adds `export * from './cards'`
- `core/src/types/index.ts` — adds `export * from './card'` and `export * from './api'`
- `core/src/validators/index.ts` — adds `export * from './card'`
- `core/src/constants/index.ts` — adds `export * from './card'`

Epic 3 will ALSO modify these same files. BMO step-09 conflict detection applies.

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/core/src/constants/card.ts` | CREATE | Card rarities + conditions |
| `packages/core/src/constants/index.ts` | MODIFY | Re-export card constants |
| `packages/core/src/db/schema/cards.ts` | CREATE | Cards table schema |
| `packages/core/src/db/schema/index.ts` | MODIFY | Re-export cards |
| `packages/core/src/types/card.ts` | CREATE | Card types |
| `packages/core/src/types/api.ts` | CREATE | API response types |
| `packages/core/src/types/index.ts` | MODIFY | Re-export card + api types |
| `packages/core/src/validators/card.ts` | CREATE | Card Zod validators |
| `packages/core/src/validators/index.ts` | MODIFY | Re-export card validators |
| `packages/core/package.json` | MODIFY | Add zod dependency |
