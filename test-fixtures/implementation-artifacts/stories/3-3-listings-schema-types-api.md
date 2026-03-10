# Story 3.3: Listings Schema, Types & API

Status: ready-for-dev

## Story

**Epic:** 3 - Listings & Users
**Labels:** `core` · `api` · `feature` · `listings`

As a developer,
I want listing CRUD with foreign keys to cards and users,
So that sellers can list cards for sale.

## Acceptance Criteria

### AC1: Listings table schema with FKs

**Given** the Drizzle setup with cards and users tables
**When** I import listing schema from `@cardtrader/core`
**Then** `listings` table is defined with: id, card_id (FK → cards), user_id (FK → users), price (real, > 0), condition (text, enum check: NM/LP/MP/HP), quantity (integer, >= 0), created_at, updated_at
**And** foreign keys reference cards and users tables

### AC2: Listing types available

**Given** the listings schema
**When** I import types from `@cardtrader/core`
**Then** `Listing`, `NewListing`, `UpdateListing` types are available
**And** all card and user types still work

### AC3: Listing validators available

**Given** the listing data model
**When** I import validators from `@cardtrader/core`
**Then** `createListingSchema` validates: card_id (number, required), price (number, positive, required), condition (enum NM/LP/MP/HP, required), quantity (number, positive integer, required)
**And** `updateListingSchema` validates: price (optional), quantity (optional)

### AC4: Create listing with auth

**Given** a user and card exist
**When** I `POST /api/listings` with valid data and `x-api-key: <user_id>` header
**Then** response is 201 with created listing
**And** user_id is set from the auth context (not from body)

### AC5: List listings with filters

**Given** listings exist
**When** I `GET /api/listings?card_id=1`
**Then** only listings for card 1 are returned

**Given** listings exist
**When** I `GET /api/listings?user_id=1`
**Then** only listings by user 1 are returned

**Given** listings exist
**When** I `GET /api/listings?condition=NM`
**Then** only NM condition listings are returned

### AC6: Listing detail with relationships

**Given** a listing with id=1 exists
**When** I `GET /api/listings/1`
**Then** response includes listing data WITH card name and user username

### AC7: FK validation

**Given** no card with id=999 exists
**When** I `POST /api/listings` with card_id=999
**Then** response is 400 with `{ error: { code: "BAD_REQUEST", message: "Card not found" } }`

## Tasks / Subtasks

### Task 1: Create Listing Schema (AC: #1)

- [ ] 1.1 Create `packages/core/src/db/schema/listings.ts`
  - Define `listings` table with FKs to cards and users
  - Condition check constraint
  - Price > 0 check
  - Quantity >= 0 check
- [ ] 1.2 Update `packages/core/src/db/schema/index.ts` — re-export listings
  - ⚠️ SHARED FILE

### Task 2: Create Listing Types (AC: #2)

- [ ] 2.1 Create `packages/core/src/types/listing.ts`
  - `Listing`, `NewListing`, `UpdateListing`
  - `ListingWithDetails` — includes card name + user username
- [ ] 2.2 Update `packages/core/src/types/index.ts` — re-export
  - ⚠️ SHARED FILE

### Task 3: Create Listing Validators (AC: #3)

- [ ] 3.1 Create `packages/core/src/validators/listing.ts`
  - `createListingSchema` — card_id, price (positive), condition (enum), quantity (positive int)
  - `updateListingSchema` — price (optional), quantity (optional)
- [ ] 3.2 Update `packages/core/src/validators/index.ts` — re-export
  - ⚠️ SHARED FILE

### Task 4: Create Listing Service (AC: #4, #5, #6, #7)

- [ ] 4.1 Create `packages/api/src/services/listing.service.ts`
  - `listListings(filters)` — optional card_id, user_id, condition
  - `getListingById(id)` — with card + user details
  - `createListing(userId, data)` — validate card exists, validate user exists
  - `updateListing(id, data)` — update price/quantity
  - `deleteListing(id)` — remove

### Task 5: Create Listing Routes (AC: #4, #5, #6, #7)

- [ ] 5.1 Create `packages/api/src/routes/listings.ts`
  - Apply `authMiddleware` to POST, PUT, DELETE
  - `GET /listings` — public, with filter query params
  - `GET /listings/:id` — public
  - `POST /listings` — auth required, user_id from context
  - `PUT /listings/:id` — auth required
  - `DELETE /listings/:id` — auth required, returns 204

### Task 6: Register Routes (AC: #4)

- [ ] 6.1 Update `packages/api/src/routes/index.ts` — register listing routes
  - ⚠️ SHARED FILE

## Dev Notes

### What Already Exists (from Epics 1 + 2 + Stories 3.1-3.2)

- `packages/core/src/db/schema/index.ts` — Exports cards + users schemas
- `packages/core/src/types/index.ts` — Exports card + user + api types
- `packages/core/src/validators/index.ts` — Exports card + user validators
- `packages/api/src/routes/index.ts` — Registers health + card + user routes
- `packages/api/src/middleware/auth.ts` — Auth middleware
- `packages/api/src/services/card.service.ts` — Card service
- `packages/api/src/services/user.service.ts` — User service

### ⚠️ Shared File Warning — MAXIMUM MUTATION

This story modifies the SAME 4 shared files that Story 3.1 already modified in THIS EPIC:
- `core/src/db/schema/index.ts` — adds listings export
- `core/src/types/index.ts` — adds listing type exports
- `core/src/validators/index.ts` — adds listing validator exports
- `api/src/routes/index.ts` — adds listing routes

**Two stories in the same epic both modify the same files. This tests BMO intra-epic conflict detection.**

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/core/src/db/schema/listings.ts` | CREATE | Listings table schema |
| `packages/core/src/db/schema/index.ts` | MODIFY | ⚠️ Add listings export |
| `packages/core/src/types/listing.ts` | CREATE | Listing types |
| `packages/core/src/types/index.ts` | MODIFY | ⚠️ Add listing type exports |
| `packages/core/src/validators/listing.ts` | CREATE | Listing validators |
| `packages/core/src/validators/index.ts` | MODIFY | ⚠️ Add listing validators |
| `packages/api/src/services/listing.service.ts` | CREATE | Listing business logic |
| `packages/api/src/routes/listings.ts` | CREATE | Listing API routes |
| `packages/api/src/routes/index.ts` | MODIFY | ⚠️ Register listing routes |
