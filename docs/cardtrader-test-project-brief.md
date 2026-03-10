# CardTrader API — BMO Test Project Brief

**Date:** 2026-03-10
**Status:** Approved — ready for artifact creation
**Purpose:** Real product to test BMO end-to-end (greenfield → incremental → retomar)

---

## Decision Context

BMO needs a **functional product built from scratch** as its test fixture — not a scaffold, not a mock. The product must:
1. Be implementable from ZERO (greenfield)
2. Have tests that PASS at every stage
3. Be verifiable functionally (`npm install && npm test`)
4. Cover all BMO orchestration scenarios (greenfield, incremental, shared files, resume)
5. Be self-contained (zero external infrastructure dependencies)
6. Be understandable by any developer in 5 minutes

**Why not TCG?** TCG depends on AWS (SST, Cognito, RDS), MercadoPago, and has 26 epics / 165+ stories / 7000+ line epics.md. Too complex, too coupled to external infra.

**Why a new product?** Full control over complexity, zero inherited debt, standalone test fixtures that anyone can run.

---

## Product Concept

**CardTrader API** — a simplified trading card marketplace. Users can list cards for sale, search the catalog, and place orders. Inspired by TCG but stripped to essentials.

**Domain model (simple):**
- Cards (name, set, rarity, image_url)
- Users (username, email)
- Listings (user_id, card_id, price, condition, quantity)
- Orders (buyer_id, listing_id, quantity, status)

---

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Runtime** | Node.js 22+ | Current LTS |
| **Language** | TypeScript 5 (strict) | Industry standard |
| **API Framework** | Hono 4 | Lightweight, built-in test client (`app.request()`), no server needed for tests |
| **ORM** | Drizzle | Type-safe, works with SQLite |
| **Database** | SQLite (better-sqlite3) | ZERO config — just a file. No Docker, no PostgreSQL, no server |
| **Test Runner** | Vitest | Fast, TypeScript native, compatible with Hono test patterns |
| **Linter** | Biome | Fast, zero config |
| **Monorepo** | npm workspaces | Simple, no turborepo/nx overhead |

**Key constraint:** `npm install && npm test` must work on ANY machine with Node.js. No Docker, no external DB, no cloud services.

---

## Monorepo Structure

```
packages/
├── core/          # Shared: DB schema (Drizzle), types, constants, validators
└── api/           # Hono API: routes, services, middleware, tests
```

> **Note:** Frontend package (`web/`) can be added in a future epic to test full-stack orchestration. Not in scope for initial 4 epics.

---

## Epic Design (4 Epics — Greenfield to Functional Product)

### Epic 1: Project Foundation
**BMO scenario:** Greenfield total — empty directory → working project

Stories should cover:
- Monorepo setup (package.json workspaces, tsconfig, vitest config)
- Drizzle + SQLite setup (connection, migration system)
- Hono app bootstrap (server entry, health endpoint)
- First passing test (health endpoint returns 200)
- Biome config + lint passes
- Dev scripts (dev, test, lint, db:migrate)

**Exit criteria:** `npm install && npm test` passes. Health endpoint works.

---

### Epic 2: Card Catalog
**BMO scenario:** Building on existing — Epic 1's foundation exists

Stories should cover:
- Cards schema (Drizzle table definition, migration)
- Cards CRUD API (GET list with pagination, GET by id, POST create, PUT update, DELETE)
- Input validation (zod or similar)
- Seed data script (populate with sample cards)
- Tests for every endpoint (success + error cases)

**Exit criteria:** All CRUD operations work. Tests pass. Seed data loads.

**Shared files with future epics:** `schema/`, routes index, types

---

### Epic 3: Listings & Users
**BMO scenario:** Shared file mutation — modifies schema and routes from Epic 2

Stories should cover:
- Users schema + CRUD
- Listings schema (references cards AND users — foreign keys)
- Listings API (create listing, list by card, list by user, update price/quantity)
- Auth middleware stub (simple API key or header-based, no JWT complexity)
- Tests for listings with relationships

**Exit criteria:** Users + Listings work. Card catalog still works. All tests pass.

**Shared file mutation points:**
- `core/src/db/schema/index.ts` — Epic 2 defined cards, Epic 3 adds users + listings + relations
- `api/src/routes/index.ts` — Epic 2 registered card routes, Epic 3 adds user + listing routes
- `core/src/types/index.ts` — Epic 2 defined card types, Epic 3 adds user + listing types

---

### Epic 4: Search & Orders
**BMO scenario:** Resume/retomar — BMO enters with 3 epics already implemented

Stories should cover:
- Search endpoint (search cards by name, filter by set/rarity)
- Orders schema (buyer, listing reference, quantity, status enum)
- Order creation (with stock validation — listing quantity decreases)
- Order status transitions (pending → confirmed → shipped → completed, or → cancelled)
- Order listing (by buyer, by seller)
- Tests for search + order flows including edge cases

**Exit criteria:** Full product functional. Search works. Orders work. All tests pass.

**Shared file mutation points:**
- `core/src/db/schema/` — adds orders table, modifies listings (stock tracking)
- `api/src/routes/` — adds search + order routes
- Possibly modifies listing service (stock decrement on order)

---

## BMO Orchestration Scenario Map

| Scenario | Epic | What BMO does |
|----------|------|---------------|
| **Greenfield** | 1 | Sub-agents create everything from scratch. No existing code. |
| **Incremental build** | 2 | Sub-agents extend existing project. Must not break Epic 1. |
| **Shared file mutation** | 3 | Multiple stories modify same files. Step-09 detects conflicts. |
| **Resume / retomar** | 4 | BMO enters a project with 3 epics done. Must understand existing codebase. |
| **Verification at every stage** | All | `npm test` must pass after every epic. Sub-agents run tests as exit criteria. |
| **Corrective loop** | 3 or 4 | At least one story should have an intentionally ambiguous AC to trigger cross-validation. |

---

## Artifact Creation Plan

BMAD artifacts to create (following TCG format, which is proven correct):

1. **PRD** (`test-fixtures/planning-artifacts/prd.md`)
   - Functional requirements for all 4 epics
   - Non-functional requirements (testing, linting, TypeScript strict)
   - Technical constraints (stack decisions)

2. **Architecture** (`test-fixtures/planning-artifacts/architecture.md`)
   - Monorepo structure
   - API endpoint inventory
   - Database schema (all tables)
   - Shared files warning matrix
   - Testing patterns (Hono test client, Drizzle mocks)

3. **Epics** (`test-fixtures/planning-artifacts/epics.md`)
   - COMPACT — target <500 lines (not 7000 like TCG)
   - 4 epics with stories inline (Given/When/Then ACs)
   - Shared file matrix
   - Dependency chain

4. **Individual Stories** (`test-fixtures/implementation-artifacts/stories/`)
   - Full BMAD format: Status, Story, ACs, Tasks/Subtasks, Dev Notes, File List
   - "What Already Exists" sections (critical for sub-agent context)
   - Shared Files Warning where applicable
   - One story with intentionally ambiguous AC (for corrective loop testing)

---

## What NOT to Do

- NO external infrastructure (no Docker, no cloud, no external DB)
- NO placeholder code — everything must be functional
- NO frontend in first 4 epics (can add later)
- NO over-engineering — simplest thing that works
- NO skipping tests — every story must include test tasks
- DO include at least one ambiguous AC in Epic 3 or 4 for corrective loop testing
- DO design shared files intentionally for mutation detection testing

---

## Verification Checklist (for after artifacts are created)

- [ ] PRD covers all 4 epics with clear FR/NFR
- [ ] Architecture defines complete schema, all endpoints, shared file matrix
- [ ] Epics.md is <500 lines with inline stories (Given/When/Then)
- [ ] Each epic has 3-5 stories (total 12-20 stories)
- [ ] All stories have Status: ready-for-dev
- [ ] Dev Notes include "What Already Exists" for Epics 2-4
- [ ] Shared file matrix identifies all collision points
- [ ] At least one AC is intentionally ambiguous
- [ ] Stack is confirmed: Hono + Drizzle + SQLite + Vitest + Biome + npm workspaces
- [ ] `npm install && npm test` is the verification command at every stage

---

_Decision made 2026-03-10 via BMAD Party Mode_
_Approved by: Rami (product owner) + Tito & team (engineering)_
