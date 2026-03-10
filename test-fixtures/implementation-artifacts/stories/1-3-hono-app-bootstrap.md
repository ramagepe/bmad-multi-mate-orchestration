# Story 1.3: Hono App Bootstrap

Status: ready-for-dev

## Story

**Epic:** 1 - Project Foundation
**Labels:** `api` · `setup`

As a developer,
I want the Hono API configured with error handling and health endpoint,
So that I have a working API to build routes on.

## Acceptance Criteria

### AC1: Health endpoint returns 200

**Given** the Hono app is running
**When** I request `GET /api/health`
**Then** response status is 200
**And** body is `{ data: { status: "ok" }, meta: { timestamp: "..." } }`

### AC2: 404 for unknown routes

**Given** the Hono app is running
**When** I request `GET /api/nonexistent`
**Then** response status is 404
**And** body matches `{ error: { code: "NOT_FOUND", message: "...", status: 404 } }`

### AC3: Global error handler

**Given** a route handler throws an error
**When** the error propagates
**Then** response is `{ error: { code: "INTERNAL_ERROR", message: "...", status: 500 } }`
**And** the error is not leaked to the client

### AC4: App is testable without server

**Given** the Hono app is exported from `app.ts`
**When** I call `app.request('/api/health')`
**Then** I get a Response object without starting a server

## Tasks / Subtasks

### Task 1: Install Hono (AC: #1)

- [ ] 1.1 Verify `hono@~4` is in `packages/api/package.json` dependencies

### Task 2: Create App Module (AC: #1, #4)

- [ ] 2.1 Create `packages/api/src/app.ts`
  - Create Hono instance with basePath `/api`
  - Export the app instance (for testing)

### Task 3: Create Error Handler Middleware (AC: #2, #3)

- [ ] 3.1 Create `packages/api/src/middleware/error-handler.ts`
  - Handle `notFound` → 404 with `{ error: { code: "NOT_FOUND", message, status: 404 } }`
  - Handle `onError` → 500 with `{ error: { code: "INTERNAL_ERROR", message, status: 500 } }`
  - In development: include error details; in production: generic message
  - Register on app instance

### Task 4: Create Health Route (AC: #1)

- [ ] 4.1 Create `packages/api/src/routes/health.ts`
  - `GET /health` → `{ data: { status: "ok" }, meta: { timestamp: new Date().toISOString() } }`

### Task 5: Create Routes Index (AC: #1, #2)

- [ ] 5.1 Create `packages/api/src/routes/index.ts`
  - Import and register health route
  - This file will be ⚠️ SHARED — future epics add routes here

### Task 6: Wire Up App (AC: #1, #2, #3)

- [ ] 6.1 Update `packages/api/src/app.ts`
  - Register error handler middleware
  - Register routes from routes/index.ts

- [ ] 6.2 Update `packages/api/src/index.ts`
  - Import app
  - Start server with `serve()` for development: `export default { port: 3000, fetch: app.fetch }`

## Dev Notes

### Key patterns

- `app.ts` exports the Hono app for testing (no server)
- `index.ts` wraps app with `serve()` for running
- Error responses always follow: `{ error: { code, message, status } }`
- Success responses always follow: `{ data, meta: { timestamp } }`
- All routes are under `/api` basePath

### Testing approach

Tests import `app` from `app.ts` and use `app.request()` — no HTTP server needed:
```typescript
const res = await app.request('/api/health');
const body = await res.json();
```

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/api/src/app.ts` | CREATE | Hono app instance |
| `packages/api/src/index.ts` | MODIFY | Server entry point |
| `packages/api/src/middleware/error-handler.ts` | CREATE | Error handling middleware |
| `packages/api/src/routes/health.ts` | CREATE | Health endpoint |
| `packages/api/src/routes/index.ts` | CREATE | Route registry (⚠️ SHARED) |
