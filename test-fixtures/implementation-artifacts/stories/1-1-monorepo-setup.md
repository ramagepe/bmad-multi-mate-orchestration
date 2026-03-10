# Story 1.1: Monorepo Setup

Status: ready-for-dev

## Story

**Epic:** 1 - Project Foundation
**Labels:** `core` · `setup`

As a developer,
I want a monorepo with npm workspaces for core and api packages,
So that I can start building features in the correct structure.

## Acceptance Criteria

### AC1: npm install resolves all workspaces

**Given** a fresh repository
**When** I run `npm install` at root
**Then** packages/core and packages/api install correctly
**And** TypeScript compilation works across packages
**And** `@cardtrader/core` is importable from api

### AC2: Package structure matches architecture

**Given** the monorepo is initialized
**When** I inspect the directory structure
**Then** `packages/core/` exists with `src/` directory
**And** `packages/api/` exists with `src/` directory
**And** root `package.json` has workspaces: `["packages/*"]`

### AC3: Cross-package TypeScript imports work

**Given** TypeScript is configured
**When** I import from `@cardtrader/core` in the api package
**Then** TypeScript resolves the import without errors

## Tasks / Subtasks

### Task 1: Initialize Root Monorepo (AC: #1, #2)

- [ ] 1.1 Create root `package.json`
  - Name: `cardtrader`
  - `"type": "module"`
  - `"private": true`
  - `"engines": { "node": ">=22.0.0" }`
  - Workspaces: `["packages/*"]`
  - Scripts: `dev`, `build`, `typecheck`, `test`, `lint`
  - File: `/package.json`

- [ ] 1.2 Create root `tsconfig.json` (base config)
  - Target: `ES2024`
  - Module: `NodeNext`
  - ModuleResolution: `NodeNext`
  - `"strict": true`
  - `"esModuleInterop": true`
  - `"skipLibCheck": true`
  - `"forceConsistentCasingInFileNames": true`
  - `"resolveJsonModule": true`
  - `"declaration": true`
  - `"sourceMap": true`
  - `"isolatedModules": true`
  - File: `/tsconfig.json`

- [ ] 1.3 Create `.gitignore`
  - Add: `node_modules/`, `dist/`, `*.db`, `.env`, `coverage/`, `.DS_Store`, `*.log`, `data/`
  - File: `/.gitignore`

- [ ] 1.4 Create directory skeleton
  - `packages/core/src/`
  - `packages/api/src/`

### Task 2: Configure packages/core (AC: #1, #3)

- [ ] 2.1 Create `packages/core/package.json`
  - Name: `@cardtrader/core`
  - `"type": "module"`
  - `"main": "./src/index.ts"`
  - `"types": "./src/index.ts"`
  - `"exports": { ".": "./src/index.ts", "./*": "./src/*.ts" }`
  - DevDependencies: `typescript@~5.7`
  - File: `packages/core/package.json`

- [ ] 2.2 Create `packages/core/tsconfig.json`
  - Extends: `../../tsconfig.json`
  - Include: `["src/**/*.ts"]`
  - File: `packages/core/tsconfig.json`

- [ ] 2.3 Create `packages/core/src/index.ts`
  - Export: `export const CORE_VERSION = '0.0.1';`
  - File: `packages/core/src/index.ts`

- [ ] 2.4 Create placeholder directories
  - `packages/core/src/db/` with `index.ts`
  - `packages/core/src/db/schema/` with `index.ts`
  - `packages/core/src/types/` with `index.ts`
  - `packages/core/src/validators/` with `index.ts`
  - `packages/core/src/constants/` with `index.ts`

### Task 3: Configure packages/api (AC: #1, #3)

- [ ] 3.1 Create `packages/api/package.json`
  - Name: `@cardtrader/api`
  - `"type": "module"`
  - Dependencies: `hono@~4`, `@cardtrader/core` (`workspace:*`)
  - DevDependencies: `typescript@~5.7`, `@types/node@latest`
  - File: `packages/api/package.json`

- [ ] 3.2 Create `packages/api/tsconfig.json`
  - Extends: `../../tsconfig.json`
  - Include: `["src/**/*.ts"]`
  - File: `packages/api/tsconfig.json`

- [ ] 3.3 Create `packages/api/src/index.ts` placeholder
  - Placeholder: `console.log('CardTrader API');`
  - File: `packages/api/src/index.ts`

### Task 4: Verify (AC: #1, #3)

- [ ] 4.1 Run `npm install` at root — verify zero errors
- [ ] 4.2 Verify `@cardtrader/core` is importable from api package
- [ ] 4.3 Run `npx tsc --noEmit` from root — verify zero type errors

## Dev Notes

### This is a greenfield story — nothing exists yet

All directories and files are created from scratch. Follow architecture.md monorepo structure exactly.

### Key decisions

- npm workspaces (not Turborepo/Nx) for simplicity
- TypeScript strict mode everywhere
- `"type": "module"` in all packages
- Core package uses direct `.ts` exports (no build step needed for Vitest)

## File List

| File | Action | Description |
|------|--------|-------------|
| `/package.json` | CREATE | Root workspace config |
| `/tsconfig.json` | CREATE | Base TypeScript config |
| `/.gitignore` | CREATE | Git ignore rules |
| `packages/core/package.json` | CREATE | Core package config |
| `packages/core/tsconfig.json` | CREATE | Core TypeScript config |
| `packages/core/src/index.ts` | CREATE | Core barrel export |
| `packages/core/src/db/index.ts` | CREATE | DB placeholder |
| `packages/core/src/db/schema/index.ts` | CREATE | Schema barrel export |
| `packages/core/src/types/index.ts` | CREATE | Types barrel export |
| `packages/core/src/validators/index.ts` | CREATE | Validators barrel export |
| `packages/core/src/constants/index.ts` | CREATE | Constants barrel export |
| `packages/api/package.json` | CREATE | API package config |
| `packages/api/tsconfig.json` | CREATE | API TypeScript config |
| `packages/api/src/index.ts` | CREATE | API entry placeholder |
