# Story 1.2: Drizzle + SQLite Setup

Status: ready-for-dev

## Story

**Epic:** 1 - Project Foundation
**Labels:** `core` · `setup`

As a developer,
I want Drizzle ORM configured with SQLite (better-sqlite3),
So that I have a working database with zero infrastructure.

## Acceptance Criteria

### AC1: Database connection works

**Given** the core package has Drizzle configured
**When** I import the DB connection from `@cardtrader/core`
**Then** a SQLite database is created at `data/cardtrader.db`
**And** the connection is usable for queries

### AC2: Migration system works

**Given** Drizzle is configured
**When** I run `npx drizzle-kit generate`
**Then** migration files are generated in `packages/core/src/db/migrations/`

**Given** migration files exist
**When** I run `npx drizzle-kit migrate`
**Then** migrations apply successfully to the SQLite database

### AC3: In-memory database for tests

**Given** the test helper module exists
**When** I create a test database
**Then** it uses in-memory SQLite (`:memory:`)
**And** tables can be created and queried within the test

## Tasks / Subtasks

### Task 1: Install Drizzle Dependencies (AC: #1)

- [ ] 1.1 Add to `packages/core/package.json` dependencies:
  - `drizzle-orm`
  - `better-sqlite3`
  - DevDependencies: `drizzle-kit`, `@types/better-sqlite3`

### Task 2: Create Database Connection (AC: #1)

- [ ] 2.1 Create `packages/core/src/db/index.ts`
  - Import `drizzle` from `drizzle-orm/better-sqlite3`
  - Import `Database` from `better-sqlite3`
  - Create `getDb(dbPath?: string)` factory function
  - Default path: `data/cardtrader.db`
  - Enable WAL mode: `pragma journal_mode = WAL`
  - Enable foreign keys: `pragma foreign_keys = ON`
  - Export the factory and a default `db` instance

### Task 3: Create Drizzle Config (AC: #2)

- [ ] 3.1 Create `packages/core/drizzle.config.ts`
  - Schema: `./src/db/schema/index.ts`
  - Out: `./src/db/migrations`
  - Driver: `better-sqlite3`
  - dbCredentials: `{ url: "data/cardtrader.db" }`

### Task 4: Create Test Helper (AC: #3)

- [ ] 4.1 Create `packages/core/src/db/test-helpers.ts`
  - Export `createTestDb()` function that returns in-memory Drizzle instance
  - Include `applySchema(db)` to create tables programmatically for tests

### Task 5: Verify (AC: #1, #2, #3)

- [ ] 5.1 Verify DB file is created when `getDb()` is called
- [ ] 5.2 Verify `data/` directory is in `.gitignore`
- [ ] 5.3 Verify in-memory test DB works

## Dev Notes

### Key decisions

- `better-sqlite3` is synchronous — no async/await needed for DB calls
- WAL mode for better concurrent read performance
- Foreign keys enabled by pragma (SQLite default is OFF)
- Test helper creates in-memory DB — fast, isolated, no cleanup needed
- `data/` directory auto-created if missing (or use `mkdir -p`)

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/core/package.json` | MODIFY | Add drizzle dependencies |
| `packages/core/src/db/index.ts` | MODIFY | DB connection factory |
| `packages/core/drizzle.config.ts` | CREATE | Drizzle Kit config |
| `packages/core/src/db/test-helpers.ts` | CREATE | Test DB helper |
