# Story 2.3: Card Seed Data

Status: ready-for-dev

## Story

**Epic:** 2 - Card Catalog
**Labels:** `core` · `feature` · `catalog`

As a developer,
I want a seed script that populates sample cards,
So that the API has data to work with in development and tests.

## Acceptance Criteria

### AC1: Seed script populates cards

**Given** an empty database
**When** I run the seed script
**Then** at least 20 sample cards are inserted
**And** cards span multiple sets (at least 3 sets)
**And** cards span all rarities (common, uncommon, rare, mythic)

### AC2: Seed is idempotent

**Given** the seed script has already run
**When** I run it again
**Then** no duplicate entries are created
**And** the script completes without errors

### AC3: Seed is usable from tests

**Given** the seed function is exported
**When** I call `seedCards(db)` in a test
**Then** the in-memory test database is populated
**And** I can query the seeded cards

## Tasks / Subtasks

### Task 1: Create Seed Data (AC: #1)

- [ ] 1.1 Create `packages/core/src/db/seed.ts`
  - Define array of sample cards (20+ cards)
  - Include cards from sets: "Alpha", "Beta", "Unlimited", "Revised", "Innistrad"
  - Include all 4 rarities
  - Example cards: Black Lotus, Lightning Bolt, Counterspell, Serra Angel, etc.
  - Export `seedCards(db)` function

### Task 2: Make Seed Idempotent (AC: #2)

- [ ] 2.1 Use `INSERT OR IGNORE` or check existence before inserting
  - Unique on `name + set` combination to prevent duplicates

### Task 3: Add Seed Script (AC: #1)

- [ ] 3.1 Add `"db:seed": "npx tsx packages/core/src/db/seed.ts"` to root `package.json`
  - Script should create/connect to DB and call `seedCards(db)`

### Task 4: Export for Tests (AC: #3)

- [ ] 4.1 Export `seedCards(db)` and `SAMPLE_CARDS` from `packages/core/src/db/seed.ts`
  - Test helper can call `seedCards(testDb)` to populate in-memory DB

## Dev Notes

### What Already Exists (from Epic 1 + Story 2.1)

- `packages/core/src/db/index.ts` — DB connection
- `packages/core/src/db/schema/cards.ts` — Cards table schema
- `packages/core/src/db/test-helpers.ts` — In-memory DB helper

### Sample cards should include

- Cards with varying name lengths (for search testing in Epic 4)
- Cards with "Dragon" in name (at least 3 — for search testing)
- Cards from at least 3 different sets
- All 4 rarities represented

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/core/src/db/seed.ts` | CREATE | Seed data script |
| `/package.json` | MODIFY | Add db:seed script |
