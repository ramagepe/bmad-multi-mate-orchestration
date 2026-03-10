# Story 3.1: Users Schema, Types & API

Status: ready-for-dev

## Story

**Epic:** 3 - Listings & Users
**Labels:** `core` · `api` · `feature` · `users`

As a developer,
I want user CRUD with schema, types, and API endpoints,
So that the system has user accounts for listings and orders.

## Acceptance Criteria

### AC1: Users table schema defined

**Given** the Drizzle setup with cards already working
**When** I import user schema from `@cardtrader/core`
**Then** `users` table is defined with: id (integer PK autoincrement), username (text NOT NULL UNIQUE), email (text NOT NULL UNIQUE), created_at (text DEFAULT now)

### AC2: User types available

**Given** the users schema
**When** I import types from `@cardtrader/core`
**Then** `User`, `NewUser`, `UpdateUser` types are available
**And** all card types from Epic 2 still work

### AC3: User validators available

**Given** the user data model
**When** I import validators from `@cardtrader/core`
**Then** `createUserSchema` validates: username (string, min 3, required), email (string, email format, required)
**And** `updateUserSchema` — same fields but all optional

### AC4: User CRUD API works

**Given** the API is running
**When** I interact with user endpoints
**Then** `POST /api/users` creates user with username + email (201)
**And** `GET /api/users` returns list of users
**And** `GET /api/users/:id` returns user or 404
**And** `PUT /api/users/:id` updates user fields

### AC5: Unique constraints enforced

**Given** a user with username "alice" exists
**When** I `POST /api/users` with username "alice"
**Then** response is 409 with `{ error: { code: "CONFLICT", message: "Username already exists" } }`

**Given** a user with email "alice@test.com" exists
**When** I `POST /api/users` with email "alice@test.com"
**Then** response is 409 with `{ error: { code: "CONFLICT" } }`

## Tasks / Subtasks

### Task 1: Create User Constants & Types (AC: #1, #2)

- [ ] 1.1 Create `packages/core/src/db/schema/users.ts`
  - Define `users` table: id, username (unique), email (unique), created_at
- [ ] 1.2 Update `packages/core/src/db/schema/index.ts` — re-export users
- [ ] 1.3 Create `packages/core/src/types/user.ts`
  - `User`, `NewUser`, `UpdateUser`
- [ ] 1.4 Update `packages/core/src/types/index.ts` — re-export user types

### Task 2: Create User Validators (AC: #3)

- [ ] 2.1 Create `packages/core/src/validators/user.ts`
  - `createUserSchema` — username (min 3), email (email format)
  - `updateUserSchema` — partial
- [ ] 2.2 Update `packages/core/src/validators/index.ts` — re-export

### Task 3: Create User Service (AC: #4, #5)

- [ ] 3.1 Create `packages/api/src/services/user.service.ts`
  - `listUsers()`, `getUserById(id)`, `createUser(data)`, `updateUser(id, data)`
  - Handle unique constraint errors → throw with conflict info

### Task 4: Create User Routes (AC: #4, #5)

- [ ] 4.1 Create `packages/api/src/routes/users.ts`
  - `GET /users` — list
  - `GET /users/:id` — detail
  - `POST /users` — create with validation
  - `PUT /users/:id` — update with validation
  - Catch unique constraint → 409 response

### Task 5: Register Routes (AC: #4)

- [ ] 5.1 Update `packages/api/src/routes/index.ts` — register user routes
  - ⚠️ SHARED FILE — add to existing health + card route registrations

## Dev Notes

### What Already Exists (from Epics 1 + 2)

- `packages/core/src/db/index.ts` — DB connection
- `packages/core/src/db/schema/index.ts` — Exports cards schema
- `packages/core/src/db/schema/cards.ts` — Cards table
- `packages/core/src/types/index.ts` — Exports card + api types
- `packages/core/src/validators/index.ts` — Exports card validators
- `packages/api/src/routes/index.ts` — Registers health + card routes
- `packages/api/src/services/card.service.ts` — Card service
- `packages/api/src/utils/response.ts` — Response helpers
- All Epic 1 + 2 tests passing

### ⚠️ Shared File Warning — CRITICAL

This story modifies the SAME files that Epic 2 created:
- `core/src/db/schema/index.ts` — Must ADD `export * from './users'` WITHOUT removing `export * from './cards'`
- `core/src/types/index.ts` — Must ADD user type exports
- `core/src/validators/index.ts` — Must ADD user validator exports
- `api/src/routes/index.ts` — Must ADD user routes WITHOUT removing card routes

**This is the primary BMO shared file mutation test point.**

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/core/src/db/schema/users.ts` | CREATE | Users table schema |
| `packages/core/src/db/schema/index.ts` | MODIFY | ⚠️ Add users export |
| `packages/core/src/types/user.ts` | CREATE | User types |
| `packages/core/src/types/index.ts` | MODIFY | ⚠️ Add user type exports |
| `packages/core/src/validators/user.ts` | CREATE | User validators |
| `packages/core/src/validators/index.ts` | MODIFY | ⚠️ Add user validator exports |
| `packages/api/src/services/user.service.ts` | CREATE | User business logic |
| `packages/api/src/routes/users.ts` | CREATE | User API routes |
| `packages/api/src/routes/index.ts` | MODIFY | ⚠️ Register user routes |
