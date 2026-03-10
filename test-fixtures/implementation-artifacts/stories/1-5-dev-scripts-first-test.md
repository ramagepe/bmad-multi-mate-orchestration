# Story 1.5: Dev Scripts & First Passing Test

Status: ready-for-dev

## Story

**Epic:** 1 - Project Foundation
**Labels:** `api` · `setup`

As a developer,
I want all dev scripts working and a passing health endpoint test,
So that the verification command `npm install && npm test` succeeds.

## Acceptance Criteria

### AC1: Health endpoint test passes

**Given** the Hono app with health endpoint
**When** I run `npm test`
**Then** the health endpoint test passes
**And** it verifies response status 200 and body structure

### AC2: npm install && npm test works on clean checkout

**Given** a fresh clone of the repository
**When** I run `npm install && npm test`
**Then** all dependencies install
**And** the health endpoint test passes
**And** exit code is 0

### AC3: All scripts work

**Given** the monorepo is set up
**When** I run each root script
**Then** `npm test` passes (Vitest)
**And** `npm run lint` passes (Biome)
**And** `npm run typecheck` passes (tsc --noEmit)

## Tasks / Subtasks

### Task 1: Create Health Endpoint Test (AC: #1)

- [ ] 1.1 Create `packages/api/src/__tests__/health.test.ts`
  ```typescript
  import { describe, it, expect } from 'vitest';
  import app from '../app';

  describe('GET /api/health', () => {
    it('returns 200 with status ok', async () => {
      const res = await app.request('/api/health');
      expect(res.status).toBe(200);
      const body = await res.json();
      expect(body.data.status).toBe('ok');
      expect(body.meta.timestamp).toBeDefined();
    });
  });
  ```

### Task 2: Add typecheck Script (AC: #3)

- [ ] 2.1 Add `"typecheck": "tsc --noEmit"` to root `package.json` scripts

### Task 3: Full Verification (AC: #2, #3)

- [ ] 3.1 Delete `node_modules/` and run `npm install && npm test`
- [ ] 3.2 Verify `npm run lint` passes
- [ ] 3.3 Verify `npm run typecheck` passes
- [ ] 3.4 Verify exit code 0 for all commands

## Dev Notes

### Exit criteria for Epic 1

This story is the **exit gate** for Epic 1. After this story completes:
- `npm install && npm test` passes ✓
- Health endpoint returns 200 ✓
- Biome lint clean ✓
- TypeScript strict mode, zero errors ✓

### Testing pattern established

This test establishes the pattern for all future tests:
1. Import `app` from `../app` (not from `index.ts`)
2. Use `app.request()` to make requests
3. Assert response status and body structure
4. No HTTP server needed

## File List

| File | Action | Description |
|------|--------|-------------|
| `packages/api/src/__tests__/health.test.ts` | CREATE | First passing test |
| `/package.json` | MODIFY | Add typecheck script |
