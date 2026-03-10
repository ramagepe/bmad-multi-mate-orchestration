---
stepsCompleted: ['step-01-init', 'step-02-context', 'step-03-starter', 'step-04-decisions', 'step-05-patterns', 'step-06-structure', 'step-07-validation', 'step-08-complete']
inputDocuments:
  - cardtrader-test-project-brief.md
  - prd.md
workflowType: 'architecture'
project_name: 'cardtrader'
user_name: 'Rami'
date: '2026-03-10'
status: 'complete'
completedAt: '2026-03-10'
---

# Architecture Decision Document — CardTrader API

---

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**

The PRD defines **31 functional requirements** across 4 domains:

| Category | Count | Highlights |
|----------|-------|------------|
| Card Catalog | 8 | CRUD + pagination + seed data |
| Users | 5 | CRUD with email uniqueness |
| Listings | 6 | CRUD with FK relationships, auth header |
| Search & Orders | 12 | Text search, stock validation, status transitions |

**Non-Functional Requirements:**

| Area | Requirement |
|------|-------------|
| **Runtime** | Node.js 22+, TypeScript 5 strict |
| **Zero infra** | SQLite, no Docker, no cloud |
| **Testing** | Vitest, all tests pass with `npm test` |
| **Portability** | `npm install && npm test` on any machine |

**Scale & Complexity:**

- **Primary domain:** REST API
- **Complexity level:** Low
- **Estimated components:** 2 packages (core, api)

### Technical Constraints

**Hard Constraints:**

1. **SQLite only** — zero infrastructure dependencies
2. **npm workspaces** — no Turborepo, no Nx
3. **Hono** — lightweight, built-in test client
4. **`npm install && npm test`** — must work on any machine with Node.js

---

## Technology Stack

| Layer | Technology | Version | Why |
|-------|-----------|---------|-----|
| **Runtime** | Node.js | 22+ | Current LTS |
| **Language** | TypeScript | 5 | Strict mode, industry standard |
| **API Framework** | Hono | 4 | Lightweight, `app.request()` test client, no server needed |
| **ORM** | Drizzle | latest | Type-safe, first-class SQLite support |
| **Database** | SQLite | better-sqlite3 | Zero config, just a file |
| **Test Runner** | Vitest | latest | Fast, TypeScript native |
| **Linter** | Biome | latest | Fast, zero config |
| **Monorepo** | npm workspaces | — | Simple, no overhead |
| **Validation** | Zod | latest | Runtime + TypeScript inference |

---

## Monorepo Structure

```
cardtrader/
├── package.json                    # Root workspace config
├── tsconfig.json                   # Base TypeScript config
├── biome.json                      # Biome linter config
├── vitest.workspace.ts             # Vitest workspace config
├── packages/
│   ├── core/                       # Shared: DB, types, validators, constants
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── drizzle.config.ts       # Drizzle Kit config (Epic 1)
│   │   └── src/
│   │       ├── index.ts            # Barrel export
│   │       ├── db/
│   │       │   ├── index.ts        # DB connection (SQLite)
│   │       │   ├── test-helpers.ts  # In-memory DB for tests (Epic 1)
│   │       │   ├── seed.ts          # Sample card data (Epic 2)
│   │       │   ├── schema/
│   │       │   │   ├── index.ts    # ⚠️ SHARED FILE — all schemas re-exported
│   │       │   │   ├── cards.ts    # Epic 2
│   │       │   │   ├── users.ts    # Epic 3
│   │       │   │   ├── listings.ts # Epic 3
│   │       │   │   └── orders.ts   # Epic 4
│   │       │   └── migrations/     # Drizzle migrations
│   │       ├── types/
│   │       │   ├── index.ts        # ⚠️ SHARED FILE — all types re-exported
│   │       │   ├── api.ts          # ApiResponse<T>, PaginationMeta
│   │       │   ├── card.ts         # Epic 2
│   │       │   ├── user.ts         # Epic 3
│   │       │   ├── listing.ts      # Epic 3
│   │       │   └── order.ts        # Epic 4
│   │       ├── validators/
│   │       │   ├── index.ts        # ⚠️ SHARED FILE
│   │       │   ├── card.ts         # Epic 2
│   │       │   ├── user.ts         # Epic 3
│   │       │   ├── listing.ts      # Epic 3
│   │       │   ├── search.ts       # Epic 4
│   │       │   └── order.ts        # Epic 4
│   │       └── constants/
│   │           ├── index.ts
│   │           ├── card.ts         # Rarities, conditions
│   │           └── order.ts        # Order statuses
│   └── api/                        # Hono API: routes, services, middleware
│       ├── package.json
│       ├── tsconfig.json
│       ├── vitest.config.ts
│       └── src/
│           ├── index.ts            # Hono app entry
│           ├── app.ts              # App creation (testable)
│           ├── routes/
│           │   ├── index.ts        # ⚠️ SHARED FILE — all routes registered
│           │   ├── health.ts       # Epic 1
│           │   ├── cards.ts        # Epic 2
│           │   ├── users.ts        # Epic 3
│           │   ├── listings.ts     # Epic 3
│           │   ├── search.ts       # Epic 4
│           │   └── orders.ts       # Epic 4
│           ├── utils/
│           │   └── response.ts     # API response helpers (Epic 2)
│           ├── services/
│           │   ├── card.service.ts
│           │   ├── user.service.ts
│           │   ├── listing.service.ts
│           │   ├── search.service.ts
│           │   └── order.service.ts
│           ├── middleware/
│           │   ├── error-handler.ts
│           │   └── auth.ts         # Epic 3 — simple API key header
│           └── __tests__/
│               ├── health.test.ts  # Epic 1
│               ├── cards.test.ts   # Epic 2
│               ├── users.test.ts   # Epic 3
│               ├── listings.test.ts# Epic 3
│               ├── search.test.ts  # Epic 4
│               └── orders.test.ts  # Epic 4
```

---

## Database Schema

### cards (Epic 2)

```sql
CREATE TABLE cards (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  set TEXT NOT NULL,
  rarity TEXT NOT NULL CHECK(rarity IN ('common','uncommon','rare','mythic')),
  image_url TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### users (Epic 3)

```sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT NOT NULL UNIQUE,
  email TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### listings (Epic 3)

```sql
CREATE TABLE listings (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  card_id INTEGER NOT NULL REFERENCES cards(id),
  user_id INTEGER NOT NULL REFERENCES users(id),
  price REAL NOT NULL CHECK(price > 0),
  condition TEXT NOT NULL CHECK(condition IN ('NM','LP','MP','HP')),
  quantity INTEGER NOT NULL CHECK(quantity >= 0),
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### orders (Epic 4)

```sql
CREATE TABLE orders (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  buyer_id INTEGER NOT NULL REFERENCES users(id),
  listing_id INTEGER NOT NULL REFERENCES listings(id),
  quantity INTEGER NOT NULL CHECK(quantity > 0),
  status TEXT NOT NULL DEFAULT 'pending'
    CHECK(status IN ('pending','confirmed','shipped','completed','cancelled')),
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

---

## API Endpoint Inventory

### Epic 1 — Foundation

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /api/health | public | Health check |

### Epic 2 — Card Catalog

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /api/cards | public | List cards (paginated) |
| GET | /api/cards/:id | public | Card detail |
| POST | /api/cards | admin | Create card |
| PUT | /api/cards/:id | admin | Update card |
| DELETE | /api/cards/:id | admin | Delete card |

### Epic 3 — Users & Listings

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /api/users | public | List users |
| GET | /api/users/:id | public | User detail |
| POST | /api/users | public | Create user |
| PUT | /api/users/:id | auth | Update user |
| GET | /api/listings | public | List listings (filterable) |
| GET | /api/listings/:id | public | Listing detail |
| POST | /api/listings | auth | Create listing |
| PUT | /api/listings/:id | auth | Update listing |
| DELETE | /api/listings/:id | auth | Delete listing |

### Epic 4 — Search & Orders

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | /api/search | public | Search cards (q, set, rarity) |
| POST | /api/orders | auth | Create order (stock validation) |
| GET | /api/orders | auth | List orders (by buyer) |
| GET | /api/orders/:id | auth | Order detail |
| PATCH | /api/orders/:id/status | auth | Transition order status |

---

## Shared File Matrix

⚠️ **These files are modified by multiple epics. BMO step-09 must detect conflicts.**

| File | Epic 2 | Epic 3 | Epic 4 | Mutation Type |
|------|--------|--------|--------|---------------|
| `core/src/db/schema/index.ts` | Adds cards | Adds users, listings | Adds orders | Export append |
| `core/src/types/index.ts` | Adds card + api types | Adds user, listing types | Adds order types | Export append |
| `core/src/validators/index.ts` | Adds card validators | Adds user, listing validators | Adds search + order validators | Export append |
| `api/src/routes/index.ts` | Registers card routes | Registers user, listing routes | Registers search, order routes | Route registration |
| `core/src/constants/index.ts` | Adds card constants | — | Adds order constants | Export append |

---

## API Response Conventions

### Success Response

```typescript
interface ApiResponse<T> {
  data: T;
  meta: {
    timestamp: string;
  };
}

interface ApiListResponse<T> {
  data: T[];
  meta: {
    timestamp: string;
    pagination: {
      page: number;
      pageSize: number;
      total: number;
      totalPages: number;
    };
  };
}
```

### Error Response

```typescript
interface ApiError {
  error: {
    code: string;
    message: string;
    status: number;
  };
}
```

---

## Testing Patterns

### Hono Test Client

No server needed — use `app.request()` directly:

```typescript
import { describe, it, expect } from 'vitest';
import app from '../app';

describe('GET /api/health', () => {
  it('returns 200 with status ok', async () => {
    const res = await app.request('/api/health');
    expect(res.status).toBe(200);
    const body = await res.json();
    expect(body.data.status).toBe('ok');
  });
});
```

### Database Testing

Each test file uses an in-memory SQLite database:

```typescript
import { beforeEach } from 'vitest';
import { drizzle } from 'drizzle-orm/better-sqlite3';
import Database from 'better-sqlite3';
import * as schema from '@cardtrader/core/db/schema';

let db: ReturnType<typeof drizzle>;

beforeEach(() => {
  const sqlite = new Database(':memory:');
  db = drizzle(sqlite, { schema });
  // Run migrations or create tables
});
```

### Test Coverage Expectations

| Epic | Test Files | Min Tests |
|------|-----------|-----------|
| 1 | health.test.ts | 1-2 |
| 2 | cards.test.ts | 8-12 (CRUD + validation + pagination) |
| 3 | users.test.ts, listings.test.ts | 10-15 (CRUD + auth + relationships) |
| 4 | search.test.ts, orders.test.ts | 12-18 (search + orders + stock + status) |

---

## Auth Strategy

**Epic 1-2:** No auth needed (all public or implicit admin).

**Epic 3+:** Simple API key header — no JWT complexity.

```typescript
// middleware/auth.ts
const AUTH_HEADER = 'x-api-key';

export const authMiddleware = (c, next) => {
  const apiKey = c.req.header(AUTH_HEADER);
  if (!apiKey) return c.json({ error: { code: 'UNAUTHORIZED', message: 'Missing API key', status: 401 } }, 401);
  // For simplicity: API key = user ID
  c.set('userId', apiKey);
  return next();
};
```

---

## Key Architecture Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Database | SQLite (better-sqlite3) | Zero config, portable, in-memory for tests |
| API Framework | Hono 4 | Built-in test client, lightweight |
| Monorepo | npm workspaces | Simplest option, no build tools overhead |
| ORM | Drizzle | Type-safe, excellent SQLite support |
| Validation | Zod | Runtime + TypeScript inference |
| Auth | API key header | Simplest thing that works for testing |
| Test isolation | In-memory SQLite | Each test gets fresh DB |
| No frontend | API only | 4 epics focus on backend; frontend can come later |
