# Story 3.2: Auth Middleware

Status: ready-for-dev

## Story

**Epic:** 3 - Listings & Users
**Labels:** `api` · `feature` · `auth`

As a developer,
I want a simple auth middleware using API key header,
So that listing creation and management requires authentication.

## Acceptance Criteria

### AC1: Unauthenticated requests rejected

**Given** the API has protected endpoints
**When** I send a request without `x-api-key` header
**Then** response is 401 with `{ error: { code: "UNAUTHORIZED", message: "Missing API key", status: 401 } }`

### AC2: Authenticated requests proceed

**Given** a valid `x-api-key` header with value `"1"` (user ID)
**When** I send a request to a protected endpoint
**Then** `userId` is available in the request context
**And** the request proceeds to the handler

### AC3: Middleware is reusable

**Given** the auth middleware is defined
**When** I apply it to a route group
**Then** all routes in the group require authentication
**And** unprotected routes (health, cards list) remain accessible

## Tasks / Subtasks

### Task 1: Create Auth Middleware (AC: #1, #2)

- [ ] 1.1 Create `packages/api/src/middleware/auth.ts`
  - Check for `x-api-key` header
  - If missing → 401 response
  - If present → set `userId` in Hono context via `c.set('userId', apiKey)`
  - Export `authMiddleware`

### Task 2: Verify Non-Interference (AC: #3)

- [ ] 2.1 Verify health endpoint still works without auth header
- [ ] 2.2 Verify card endpoints still work without auth header
- [ ] 2.3 Middleware is NOT applied globally — only to specific route groups

## Dev Notes

### What Already Exists (from Epics 1 + 2 + Story 3.1)

- `packages/api/src/middleware/error-handler.ts` — Error handling middleware
- `packages/api/src/routes/index.ts` — Route registry
- All existing routes are public (no auth required)

### Design decisions

- API key = user ID for simplicity (no JWT, no sessions)
- Middleware is applied per-route-group, not globally
- Protected routes: POST/PUT/DELETE on listings, POST on orders
- Public routes: health, cards (all), users (list/detail), search

### Usage pattern

```typescript
import { authMiddleware } from '../middleware/auth';

const listings = new Hono();
listings.use('*', authMiddleware); // All listing routes require auth
listings.post('/', createListing);
```

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/api/src/middleware/auth.ts` | CREATE | API key auth middleware |
