# Story 1.4: Vitest + Biome Configuration

Status: ready-for-dev

## Story

**Epic:** 1 - Project Foundation
**Labels:** `core` · `setup`

As a developer,
I want Vitest and Biome configured for the monorepo,
So that I can run tests and lint from the root.

## Acceptance Criteria

### AC1: Vitest runs from root

**Given** Vitest is configured for the monorepo
**When** I run `npm test` at root
**Then** Vitest discovers and runs tests across all packages
**And** test results are reported correctly

### AC2: Biome lint passes

**Given** Biome is configured
**When** I run `npm run lint` at root
**Then** all source files are checked
**And** zero warnings or errors

### AC3: Biome format is consistent

**Given** Biome formatter is configured
**When** I run `npm run format` at root
**Then** all files are formatted consistently
**And** tabs for indentation (Biome default)

## Tasks / Subtasks

### Task 1: Install Vitest (AC: #1)

- [ ] 1.1 Add to root `package.json` devDependencies: `vitest`
- [ ] 1.2 Create `vitest.workspace.ts` at root:
  ```typescript
  export default ['packages/*'];
  ```
- [ ] 1.3 Create `packages/api/vitest.config.ts`:
  ```typescript
  import { defineConfig } from 'vitest/config';
  export default defineConfig({
    test: {
      globals: true,
      environment: 'node',
    },
  });
  ```
- [ ] 1.4 Add `"test": "vitest run"` to root `package.json` scripts
- [ ] 1.5 Add `"test:watch": "vitest"` to root `package.json` scripts

### Task 2: Install Biome (AC: #2, #3)

- [ ] 2.1 Add to root `package.json` devDependencies: `@biomejs/biome`
- [ ] 2.2 Create `biome.json` at root:
  - Linter enabled with recommended rules
  - Formatter enabled (tab indentation, 100 line width)
  - Organize imports enabled
  - Ignore: `node_modules`, `dist`, `data`, `*.db`, `coverage`
- [ ] 2.3 Add `"lint": "biome check ."` to root `package.json` scripts
- [ ] 2.4 Add `"format": "biome format --write ."` to root `package.json` scripts

### Task 3: Verify (AC: #1, #2)

- [ ] 3.1 Run `npm test` — should report "no tests found" or pass (no failures)
- [ ] 3.2 Run `npm run lint` — should pass with zero issues
- [ ] 3.3 Run `npm run format` — should complete without errors

## Dev Notes

### Vitest workspace vs monorepo

Using `vitest.workspace.ts` at root allows running `npm test` once to test all packages. Each package can have its own `vitest.config.ts` for package-specific settings.

### Biome over ESLint

Biome replaces both ESLint and Prettier. Single tool, zero config, fast. Default rules are good enough.

## File List

| File | Action | Description |
|------|--------|-------------|
| `/package.json` | MODIFY | Add vitest, biome devDeps + scripts |
| `/vitest.workspace.ts` | CREATE | Vitest workspace config |
| `/biome.json` | CREATE | Biome linter/formatter config |
| `packages/api/vitest.config.ts` | CREATE | API test config |
