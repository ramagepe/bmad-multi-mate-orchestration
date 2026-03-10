# Story 3.5: Users Tests

Status: ready-for-dev

## Story

**Epic:** 3 - Listings & Users
**Labels:** `api` · `test` · `users`

As a developer,
I want comprehensive tests for user endpoints,
So that I can verify user management works correctly.

## Acceptance Criteria

### AC1: CRUD tests pass

**Given** the users API
**When** I run tests
**Then** create user (201), list users (200), get by id (200), update user (200) all pass

### AC2: Unique constraint tests pass

**Given** a user "alice" exists
**When** I create another user with username "alice"
**Then** response is 409 Conflict

**Given** a user with email "alice@test.com" exists
**When** I create another user with email "alice@test.com"
**Then** response is 409 Conflict

### AC3: Validation tests pass

**Given** invalid user payloads
**When** I submit them
**Then** missing username returns 400
**And** missing email returns 400
**And** invalid email format returns 400
**And** username too short (< 3 chars) returns 400

### AC4: Error case tests pass

**Given** the users API
**When** I request nonexistent user
**Then** GET /api/users/999 returns 404

### AC5: Zero regressions

**Given** all previous tests
**When** I run `npm test`
**Then** all tests pass (health + cards + users + listings)

## Tasks / Subtasks

### Task 1: Create User Test File (AC: #1, #2, #3, #4)

- [ ] 1.1 Create `packages/api/src/__tests__/users.test.ts`
  - Setup: in-memory DB
  - Test groups:
    - **Create:** valid user (201), duplicate username (409), duplicate email (409)
    - **Validation:** missing username (400), missing email (400), invalid email (400), short username (400)
    - **List:** returns array of users
    - **Detail:** existing user (200), nonexistent user (404)
    - **Update:** valid update (200)

### Task 2: Full Verification (AC: #5)

- [ ] 2.1 Run `npm test` — all tests pass
- [ ] 2.2 Run `npm run lint` — zero warnings
- [ ] 2.3 Run `npm install && npm test` — full Epic 3 verification

## Dev Notes

### What Already Exists (from Epics 1 + 2 + Stories 3.1-3.4)

- `packages/api/src/__tests__/health.test.ts` — Health test
- `packages/api/src/__tests__/cards.test.ts` — Card tests
- `packages/api/src/__tests__/listings.test.ts` — Listing tests
- `packages/api/src/routes/users.ts` — User routes
- `packages/api/src/services/user.service.ts` — User service

### Test count target

7-10 tests:
- 1 create test (valid)
- 2 uniqueness tests (duplicate username, duplicate email)
- 3-4 validation tests (missing fields, invalid email, short username)
- 1 list test
- 1 detail test (found)
- 1 not-found test (404)
- 1 update test

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/api/src/__tests__/users.test.ts` | CREATE | User API tests |
