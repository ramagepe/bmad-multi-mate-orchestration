---
stepsCompleted: ['step-01-init', 'step-02-discovery', 'step-03-success', 'step-04-journeys', 'step-05-domain', 'step-06-innovation', 'step-07-project-type', 'step-08-scoping', 'step-09-functional', 'step-10-nonfunctional', 'step-11-polish', 'step-12-complete']
currentStep: 'completed'
currentStepStatus: 'completed'
completedAt: '2026-03-10'
inputDocuments:
  - cardtrader-test-project-brief.md
workflowType: 'prd'
classification:
  projectType: web_api
  domain: ecommerce_marketplace
  complexity: low
  projectContext: greenfield
  notes:
    - Simplified trading card marketplace API
    - SQLite for zero-infrastructure testing
    - BMO test fixture product
---

# Product Requirements Document - CardTrader API

**Author:** Rami
**Date:** 2026-03-10

---

## Executive Summary

### Vision

CardTrader API is a simplified trading card marketplace backend. Users can list cards for sale, search the catalog, and place orders. It serves as a **real, functional product** built from scratch to test BMO orchestration end-to-end.

### Product Differentiator

- **Zero infrastructure:** SQLite database, no Docker, no cloud — `npm install && npm test` on any machine
- **Complete domain model:** Cards, Users, Listings, Orders with proper relationships
- **Test-driven:** Every feature has tests that pass at every stage

### Target Users

| User | Description |
|------|-------------|
| **Buyers** | Users searching cards and placing orders |
| **Sellers** | Users listing cards for sale with price and condition |
| **Admin** | System operators managing catalog and orders |

### Core Value Proposition

A fully functional trading card marketplace API that demonstrates all BMO orchestration scenarios (greenfield, incremental, shared file mutation, resume) while remaining simple enough for any developer to understand in 5 minutes.

---

## Success Criteria

### User Success

**Buyer:**
- Search cards by name, filter by set/rarity
- View card details with available listings
- Place orders with stock validation

**Seller:**
- List cards with price, condition, and quantity
- Manage listings (update price, quantity)

### Technical Success

| Metric | Target |
|--------|--------|
| **Test command** | `npm install && npm test` passes on any machine with Node.js |
| **Zero external deps** | No Docker, no cloud, no external DB |
| **TypeScript strict** | Zero type errors |
| **Lint clean** | Biome passes with zero warnings |

---

## Product Scope

### MVP — 4 Epics

**Epic 1: Project Foundation (Greenfield)**
- Monorepo setup with npm workspaces
- Drizzle + SQLite configuration
- Hono app bootstrap with health endpoint
- Vitest + Biome configuration
- First passing test

**Epic 2: Card Catalog (Incremental Build)**
- Cards schema and migration
- Full CRUD API for cards
- Input validation with Zod
- Seed data script
- Tests for all endpoints

**Epic 3: Listings & Users (Shared File Mutation)**
- Users schema + CRUD
- Listings schema with foreign keys to cards and users
- Listings API (create, list, update, delete)
- Auth middleware stub (API key header)
- Tests with relationships

**Epic 4: Search & Orders (Resume/Retomar)**
- Search endpoint (name, set, rarity filters)
- Orders schema with status enum
- Order creation with stock validation
- Order status transitions (pending → confirmed → shipped → completed | cancelled)
- Tests for search + order flows

---

## Functional Requirements

### Catalog (Epic 2)

| ID | Requirement |
|----|-------------|
| FR1 | Cards table with: name, set, rarity, image_url, created_at, updated_at |
| FR2 | GET /api/cards — list with pagination (page, pageSize) |
| FR3 | GET /api/cards/:id — single card detail |
| FR4 | POST /api/cards — create card (admin) |
| FR5 | PUT /api/cards/:id — update card (admin) |
| FR6 | DELETE /api/cards/:id — delete card (admin) |
| FR7 | Input validation on create/update (name required, rarity enum) |
| FR8 | Seed script populates sample cards |

### Users (Epic 3)

| ID | Requirement |
|----|-------------|
| FR9 | Users table with: username, email, created_at |
| FR10 | GET /api/users — list users |
| FR11 | GET /api/users/:id — user detail |
| FR12 | POST /api/users — create user |
| FR13 | PUT /api/users/:id — update user |

### Listings (Epic 3)

| ID | Requirement |
|----|-------------|
| FR14 | Listings table with: user_id (FK), card_id (FK), price, condition, quantity |
| FR15 | GET /api/listings — list with filters (card_id, user_id, condition) |
| FR16 | GET /api/listings/:id — listing detail |
| FR17 | POST /api/listings — create listing (requires auth header) |
| FR18 | PUT /api/listings/:id — update listing price/quantity |
| FR19 | DELETE /api/listings/:id — remove listing |

### Search (Epic 4)

| ID | Requirement |
|----|-------------|
| FR20 | GET /api/search?q=name — search cards by name (LIKE) |
| FR21 | Search filters: set, rarity |
| FR22 | Search returns cards with listing count and min price |

### Orders (Epic 4)

| ID | Requirement |
|----|-------------|
| FR23 | Orders table with: buyer_id (FK), listing_id (FK), quantity, status, created_at |
| FR24 | POST /api/orders — create order (validates stock, decrements listing quantity) |
| FR25 | GET /api/orders — list orders (filter by buyer_id) |
| FR26 | GET /api/orders/:id — order detail |
| FR27 | PATCH /api/orders/:id/status — transition status |
| FR28 | Status enum: pending, confirmed, shipped, completed, cancelled |
| FR29 | Stock validation: order quantity ≤ listing quantity |
| FR30 | On order creation: listing quantity decreases by order quantity |
| FR31 | On order cancellation: listing quantity restored |

---

## Non-Functional Requirements

| ID | Area | Requirement |
|----|------|-------------|
| NFR1 | **Runtime** | Node.js 22+ |
| NFR2 | **Language** | TypeScript 5 with strict mode |
| NFR3 | **Framework** | Hono 4 (API), Drizzle (ORM), SQLite (better-sqlite3) |
| NFR4 | **Testing** | Vitest — all tests pass with `npm test` |
| NFR5 | **Linting** | Biome — zero warnings with `npm run lint` |
| NFR6 | **Monorepo** | npm workspaces (packages/core, packages/api) |
| NFR7 | **Zero infra** | No Docker, no external DB, no cloud services |
| NFR8 | **Portability** | `npm install && npm test` works on any machine with Node.js |
| NFR9 | **Type safety** | Zero TypeScript errors in strict mode |
| NFR10 | **Validation** | Zod schemas for all API inputs |

---

## Domain Model

```
┌──────────┐     ┌──────────┐     ┌───────────┐     ┌──────────┐
│  Cards   │◄────│ Listings │────►│   Users   │◄────│  Orders  │
│          │     │          │     │           │     │          │
│ id       │     │ id       │     │ id        │     │ id       │
│ name     │     │ card_id  │     │ username  │     │ buyer_id │
│ set      │     │ user_id  │     │ email     │     │listing_id│
│ rarity   │     │ price    │     │ created_at│     │ quantity │
│ image_url│     │ condition│     └───────────┘     │ status   │
│ created  │     │ quantity │                       │ created  │
│ updated  │     │ created  │                       │ updated  │
└──────────┘     │ updated  │                       └──────────┘
                 └──────────┘
```

**Relationships:**
- Listing → Card (many-to-one)
- Listing → User (many-to-one, seller)
- Order → Listing (many-to-one)
- Order → User (many-to-one, buyer)

**Enums:**
- `card_rarity`: common, uncommon, rare, mythic
- `card_condition`: NM, LP, MP, HP
- `order_status`: pending, confirmed, shipped, completed, cancelled

---

## BMO Orchestration Scenario Map

| Scenario | Epic | What BMO Does |
|----------|------|---------------|
| **Greenfield** | 1 | Creates everything from scratch. No existing code. |
| **Incremental build** | 2 | Extends existing project. Must not break Epic 1. |
| **Shared file mutation** | 3 | Multiple stories modify same files (schema index, routes index, types). |
| **Resume / retomar** | 4 | BMO enters with 3 epics done. Must understand existing codebase. |
| **Verification** | All | `npm test` must pass after every epic. |

---

## What NOT to Do

- NO external infrastructure (no Docker, no cloud, no external DB)
- NO placeholder code — everything must be functional
- NO frontend in these 4 epics
- NO over-engineering — simplest thing that works
- NO skipping tests — every story must include test tasks
