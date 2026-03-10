---
stepsCompleted: ['step-01-validate-prerequisites', 'step-02-design-epics', 'step-03-create-stories', 'step-04-final-validation']
inputDocuments:
  - prd.md
  - architecture.md
workflowType: 'create-epics-and-stories'
---

# CardTrader API — Epic Breakdown

## Overview

4 epics, 18 stories total. Each epic maps to a BMO orchestration scenario. Every story has Given/When/Then acceptance criteria, clear separation of layer (core/api), and test requirements.

**Conventions:**
- Most stories target 1 layer (core or api). Some stories (3.1, 3.3, 4.2) span both layers when schema + API are tightly coupled.
- Stories within an epic are ordered by dependency
- Shared file mutations are marked with ⚠️

**API Response Conventions:**
- Success: `{ data: T, meta: { timestamp } }` or `{ data: T[], meta: { timestamp, pagination } }`
- Error: `{ error: { code, message, status } }`
- Verification command at every stage: `npm install && npm test`

---

## Requirements Inventory

### Functional Requirements (31 FRs)

**Catalog:** FR1-FR8 (Cards CRUD, pagination, validation, seed data)
**Users:** FR9-FR13 (Users CRUD)
**Listings:** FR14-FR19 (Listings CRUD with FK relationships, auth)
**Search:** FR20-FR22 (Text search, filters, aggregated results)
**Orders:** FR23-FR31 (Order creation, stock validation, status transitions)

### Non-Functional Requirements (10 NFRs)

NFR1-NFR10: Node.js 22+, TypeScript strict, Hono 4, Drizzle, SQLite, Vitest, Biome, npm workspaces, Zod, zero infra

---

## Epic List

| # | Epic | Layer | Stories | FRs Covered | BMO Scenario |
|---|------|-------|---------|-------------|--------------|
| 1 | Project Foundation | core, api | 5 | NFR1-NFR10 | Greenfield |
| 2 | Card Catalog | core, api | 4 | FR1-FR8 | Incremental build |
| 3 | Listings & Users | core, api | 5 | FR9-FR19 | Shared file mutation |
| 4 | Search & Orders | core, api | 4 | FR20-FR31 | Resume / retomar |

---

## Shared File Matrix

| File | Epic 1 | Epic 2 | Epic 3 | Epic 4 |
|------|--------|--------|--------|--------|
| `core/src/db/schema/index.ts` | Creates (empty) | Adds cards | Adds users, listings | Adds orders |
| `core/src/types/index.ts` | Creates (api types) | Adds card types | Adds user, listing types | Adds order types |
| `core/src/validators/index.ts` | Creates (empty) | Adds card validators | Adds user, listing validators | Adds search + order validators |
| `core/src/constants/index.ts` | Creates (empty) | Adds card constants | — | Adds order constants |
| `api/src/routes/index.ts` | Creates (health) | Adds card routes | Adds user, listing routes | Adds search, order routes |

---

## Dependency Chain

```
Epic 1 (Foundation) → Epic 2 (Cards) → Epic 3 (Users & Listings) → Epic 4 (Search & Orders)
```

Within epics, `.1` stories (core/types) must complete before `.2+` stories (api/routes).

---

## Epic 1: Project Foundation

**Goal:** Establish base infrastructure — monorepo, database, Hono app, first passing test.

**Priority:** P0 — Blocks all other epics.

**BMO Scenario:** Greenfield total — empty directory → working project.

### Story 1.1: Monorepo Setup

**Labels:** `core` · `setup`

As a developer,
I want a monorepo with npm workspaces for core and api packages,
So that I can start building features in the correct structure.

**Given** a fresh directory
**When** I run `npm install` at root
**Then** packages/core and packages/api install correctly
**And** TypeScript compilation works across packages
**And** `@cardtrader/core` is importable from api

---

### Story 1.2: Drizzle + SQLite Setup

**Labels:** `core` · `setup`

As a developer,
I want Drizzle ORM configured with SQLite (better-sqlite3),
So that I have a working database with zero infrastructure.

**Given** the core package exists
**When** I import the DB connection from `@cardtrader/core`
**Then** a SQLite database is created at `data/cardtrader.db`
**And** Drizzle migrations can be generated and applied

**Dependencies:** Story 1.1

---

### Story 1.3: Hono App Bootstrap

**Labels:** `api` · `setup`

As a developer,
I want the Hono API configured with error handling and health endpoint,
So that I have a working API to build routes on.

**Given** the api package exists
**When** I request `GET /api/health`
**Then** response is `{ data: { status: "ok" }, meta: { timestamp: "..." } }`
**And** invalid routes return 404 with error format
**And** errors return standardized `{ error: { code, message, status } }`

**Dependencies:** Story 1.1

---

### Story 1.4: Vitest + Biome Configuration

**Labels:** `core` · `setup`

As a developer,
I want Vitest and Biome configured for the monorepo,
So that I can run tests and lint from the root.

**Given** the monorepo is set up
**When** I run `npm test` at root
**Then** Vitest runs across all packages
**And** at least the health endpoint test passes

**Given** Biome is configured
**When** I run `npm run lint` at root
**Then** all files pass lint with zero warnings

**Dependencies:** Story 1.3

---

### Story 1.5: Dev Scripts & First Passing Test

**Labels:** `api` · `setup`

As a developer,
I want all dev scripts working and a passing health endpoint test,
So that the verification command `npm install && npm test` succeeds.

**Given** the complete foundation
**When** I run `npm install && npm test` on a clean checkout
**Then** all dependencies install
**And** the health endpoint test passes
**And** `npm run lint` passes

**Dependencies:** Story 1.4

---

## Epic 2: Card Catalog

**Goal:** Create the cards domain — schema, types, CRUD API, validation, seed data, tests.

**Priority:** P0 — Foundation of the catalog.

**BMO Scenario:** Incremental build — extends Epic 1's foundation. Must not break existing tests.

### Story 2.1: Cards Schema & Types

**Labels:** `core` · `feature` · `catalog`

As a developer,
I want the cards table schema and TypeScript types defined,
So that the API can interact with card data.

**Given** the Drizzle setup from Epic 1
**When** I import card schema from `@cardtrader/core`
**Then** cards table exists with: id, name, set, rarity (enum), image_url, created_at, updated_at
**And** Card, NewCard, UpdateCard types are available
**And** Zod validators exist for createCardSchema and updateCardSchema
**And** CARD_RARITIES constant is exported

⚠️ **Shared files modified:** `core/src/db/schema/index.ts`, `core/src/types/index.ts`, `core/src/validators/index.ts`, `core/src/constants/index.ts`

**Dependencies:** Epic 1 complete

---

### Story 2.2: Cards CRUD API

**Labels:** `api` · `feature` · `catalog`

As a developer,
I want full CRUD endpoints for cards,
So that cards can be managed in the catalog.

**Given** the cards schema exists
**When** I request the cards API
**Then** GET /api/cards returns paginated list with `{ data: [...], meta: { pagination } }`
**And** GET /api/cards/:id returns single card or 404
**And** POST /api/cards creates card with validation (name required, valid rarity)
**And** PUT /api/cards/:id updates card fields
**And** DELETE /api/cards/:id removes card and returns 204
**And** invalid input returns 400 with validation error details

⚠️ **Shared files modified:** `api/src/routes/index.ts`

**Dependencies:** Story 2.1

---

### Story 2.3: Card Seed Data

**Labels:** `core` · `feature` · `catalog`

As a developer,
I want a seed script that populates sample cards,
So that the API has data to work with in development and tests.

**Given** the cards schema exists
**When** I run the seed script
**Then** at least 20 sample cards are inserted
**And** cards span multiple sets and rarities
**And** the script is idempotent (can run multiple times safely)

**Dependencies:** Story 2.1

---

### Story 2.4: Card API Tests

**Labels:** `api` · `test` · `catalog`

As a developer,
I want comprehensive tests for all card endpoints,
So that I can verify the catalog works correctly.

**Given** the cards CRUD API and seed data exist
**When** I run `npm test`
**Then** tests cover: list cards (pagination), get card by id, create card (valid + invalid), update card, delete card
**And** tests verify validation errors (missing name, invalid rarity)
**And** tests verify 404 for nonexistent card
**And** all Epic 1 tests still pass (zero regressions)

**Dependencies:** Story 2.2, Story 2.3

---

## Epic 3: Listings & Users

**Goal:** Add users and listings domains with foreign key relationships and auth middleware.

**Priority:** P0 — Enables marketplace functionality.

**BMO Scenario:** Shared file mutation — multiple stories modify schema/index.ts, routes/index.ts, types/index.ts.

### Story 3.1: Users Schema, Types & API

**Labels:** `core` · `api` · `feature` · `users`

As a developer,
I want user CRUD with schema, types, and API endpoints,
So that the system has user accounts for listings and orders.

**Given** the project with cards working
**When** I interact with the users API
**Then** POST /api/users creates user (username + email required, both unique)
**And** GET /api/users returns list
**And** GET /api/users/:id returns user or 404
**And** PUT /api/users/:id updates user
**And** duplicate username/email returns 409 Conflict

⚠️ **Shared files modified:** `core/src/db/schema/index.ts`, `core/src/types/index.ts`, `core/src/validators/index.ts`, `api/src/routes/index.ts`

**Dependencies:** Epic 2 complete

---

### Story 3.2: Auth Middleware

**Labels:** `api` · `feature` · `auth`

As a developer,
I want a simple auth middleware using API key header,
So that listing creation and management requires authentication.

**Given** the API is running
**When** I send a request to a protected endpoint without `x-api-key` header
**Then** response is 401 with `{ error: { code: "UNAUTHORIZED" } }`

**Given** I send a request with `x-api-key: <user_id>`
**When** the middleware processes it
**Then** `userId` is set in the request context
**And** the request proceeds to the handler

**Dependencies:** Story 3.1

---

### Story 3.3: Listings Schema, Types & API

**Labels:** `core` · `api` · `feature` · `listings`

As a developer,
I want listing CRUD with foreign keys to cards and users,
So that sellers can list cards for sale.

**Given** users and cards exist in the system
**When** I interact with the listings API (with auth)
**Then** POST /api/listings creates listing (card_id, price, condition, quantity required)
**And** GET /api/listings returns list with optional filters (card_id, user_id, condition)
**And** GET /api/listings/:id returns listing with card and user details
**And** PUT /api/listings/:id updates price and/or quantity
**And** DELETE /api/listings/:id removes listing
**And** creating listing with nonexistent card_id returns 400
**And** creating listing with nonexistent user_id returns 400

⚠️ **Shared files modified:** `core/src/db/schema/index.ts`, `core/src/types/index.ts`, `core/src/validators/index.ts`, `api/src/routes/index.ts`

**Dependencies:** Story 3.1, Story 3.2

---

### Story 3.4: Listings Relationship Tests

**Labels:** `api` · `test` · `listings`

As a developer,
I want comprehensive tests for listings including relationships and auth,
So that I can verify listings work correctly with foreign keys.

**Given** users, cards, and listings APIs exist
**When** I run `npm test`
**Then** tests cover: create listing with valid FK, create listing with invalid FK (400), list with filters, update, delete
**And** tests verify auth middleware (401 without header, success with header)
**And** tests verify listing detail includes card and user info
**And** all Epic 1 + Epic 2 tests still pass (zero regressions)

**Dependencies:** Story 3.3

---

### Story 3.5: Users Tests

**Labels:** `api` · `test` · `users`

As a developer,
I want comprehensive tests for user endpoints,
So that I can verify user management works correctly.

**Given** the users API exists
**When** I run `npm test`
**Then** tests cover: create user, duplicate username (409), duplicate email (409), list users, get by id, update, get nonexistent (404)
**And** all previous tests still pass

**Dependencies:** Story 3.1

---

## Epic 4: Search & Orders

**Goal:** Add search functionality and order management with stock validation and status transitions.

**Priority:** P0 — Completes the marketplace.

**BMO Scenario:** Resume/retomar — BMO enters with 3 epics already implemented. Must understand existing codebase.

### Story 4.1: Search Endpoint

**Labels:** `api` · `feature` · `search`

As a developer,
I want a search endpoint that finds cards by name with filters,
So that buyers can discover cards in the catalog.

**Given** cards exist in the catalog
**When** I request GET /api/search?q=dragon
**Then** cards matching "dragon" in name are returned (case-insensitive LIKE)
**And** results include listing_count and min_price per card
**And** I can filter by set and/or rarity
**And** empty query returns validation error
**And** results are paginated

⚠️ **Shared files modified:** `api/src/routes/index.ts`

**Dependencies:** Epic 3 complete

---

### Story 4.2: Orders Schema, Types & Creation

**Labels:** `core` · `api` · `feature` · `orders`

As a developer,
I want order creation with stock validation,
So that buyers can purchase cards with correct inventory tracking.

**Given** a listing with quantity 5 exists
**When** I POST /api/orders with { listing_id, quantity: 3 } (with auth)
**Then** an order is created with status "pending"
**And** the listing quantity decreases from 5 to 2
**And** the response includes order details with listing and buyer info

**Given** a listing with quantity 2 exists
**When** I POST /api/orders with { listing_id, quantity: 5 }
**Then** response is 400 with "Insufficient stock"
**And** listing quantity remains unchanged

**Given** I order the exact remaining quantity of a listing
**When** the order is created
**Then** listing quantity becomes 0
**And** the listing should still be retrievable

⚠️ **Shared files modified:** `core/src/db/schema/index.ts`, `core/src/types/index.ts`, `core/src/validators/index.ts`, `core/src/constants/index.ts`, `api/src/routes/index.ts`

**Dependencies:** Story 4.1

---

### Story 4.3: Order Status Transitions & Listing

**Labels:** `api` · `feature` · `orders`

As a developer,
I want order status transitions and order listing,
So that orders can progress through their lifecycle.

**Given** an order exists with status "pending"
**When** I PATCH /api/orders/:id/status with { status: "confirmed" }
**Then** status changes to "confirmed"

**Given** the valid transitions: pending→confirmed, confirmed→shipped, shipped→completed, pending→cancelled, confirmed→cancelled
**When** I attempt an invalid transition (e.g., completed→pending)
**Then** response is 400 with "Invalid status transition"

**Given** an order with status "pending" and quantity 3
**When** I cancel the order
**Then** status changes to "cancelled"
**And** the listing quantity is restored (increased by 3)

**Given** orders exist for a buyer
**When** I GET /api/orders?buyer_id=1
**Then** only that buyer's orders are returned

**Dependencies:** Story 4.2

---

### Story 4.4: Search & Order Tests

**Labels:** `api` · `test` · `search` · `orders`

As a developer,
I want comprehensive tests for search and order flows,
So that I can verify the complete marketplace works.

**Given** the complete API exists
**When** I run `npm test`
**Then** search tests cover: search by name, filter by set, filter by rarity, pagination, empty results, listing aggregation
**And** order tests cover: create order (success), insufficient stock (400), order listing by buyer, status transitions (valid + invalid), cancellation restores stock
**And** edge cases: order exact remaining quantity, cancel already-completed order (400), order with nonexistent listing (404)
**And** all previous tests still pass (zero regressions)
**And** `npm install && npm test` succeeds from clean checkout

**Dependencies:** Story 4.3

---

## Verification Checklist

- [ ] PRD covers all 4 epics with clear FR/NFR
- [ ] Architecture defines complete schema, all endpoints, shared file matrix
- [ ] Epics.md is <500 lines with inline stories (Given/When/Then)
- [ ] Each epic has 4-5 stories (18 total)
- [ ] All stories have Status: ready-for-dev
- [ ] Dev Notes include "What Already Exists" for Epics 2-4
- [ ] Shared file matrix identifies all collision points
- [ ] At least one AC is intentionally ambiguous (Story 4.2, 3rd AC)
- [ ] Stack: Hono + Drizzle + SQLite + Vitest + Biome + npm workspaces
- [ ] `npm install && npm test` is the verification command at every stage
