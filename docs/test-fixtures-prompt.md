# BMO Test Fixtures — Generation Prompt

> **Instruccion**: Abri una ventana nueva de Claude Code apuntando a este repo (`bmad-multi-mate-orchestration`) y pega este prompt completo.

---

## Contexto

Este repo es **BMO** (BMAD Multi-Mate Orchestration) — un modulo externo para el framework BMAD Method. Lee `docs/product-brief.md` para el contexto completo del modulo.

BMO orquesta workflows BMAD en paralelo. Para testearlo end-to-end, necesitamos **artefactos de prueba** que sigan el formato exacto que produce el framework BMAD (modulo BMM).

## Tu Mision

Crear un **proyecto ficticio completo** con artefactos BMAD para testear BMO. El proyecto ficticio es una **API REST de Task Manager** (Node.js/Express/TypeScript/PostgreSQL) — simple, realista, con 3 stories disenadas para cubrir escenarios especificos de orquestacion.

## Primer Paso: Leer el Product Brief

Lee `docs/product-brief.md` completo para entender:
- Que es BMO y que problema resuelve
- Los input/output contracts de los workflows
- Los 3 modos de orquestacion
- Las limitaciones v1

## Donde Crear los Archivos

Todo va en `test-fixtures/` en la raiz del repo:

```
test-fixtures/
├── project/                         # El "proyecto ficticio" simulado
│   ├── package.json                 # Metadata del proyecto
│   ├── tsconfig.json                # TypeScript config
│   ├── prisma/
│   │   └── schema.prisma            # Prisma schema (tasks table only)
│   ├── src/
│   │   ├── index.ts                 # Entry point Express
│   │   ├── routes/
│   │   │   └── tasks.ts             # Task routes (CRUD basico)
│   │   ├── models/
│   │   │   └── task.ts              # Task interface
│   │   ├── middleware/
│   │   │   └── auth.ts              # Auth middleware stub
│   │   └── config/
│   │       └── database.ts          # DB config
│   └── tests/
│       └── tasks.test.ts            # Test placeholder
├── planning-artifacts/
│   ├── prd.md                       # Product Requirements Document
│   ├── architecture.md              # Architecture doc
│   └── epics.md                     # Epic breakdown con stories inline
└── implementation-artifacts/
    └── stories/
        ├── story-1-1.md             # Task categories (happy path)
        ├── story-1-2.md             # Task priority (shared files)
        └── story-1-3.md             # Task search (ambiguous AC)
```

## Formato de Cada Artefacto

### PRD (prd.md)

```markdown
# TaskManager API — Product Requirements Document

**Author:** Test Fixture
**Date:** 2026-03-09
**Version:** 1.0

## Overview
[1-2 parrafos: API REST para gestion de tareas. TypeScript, Express, PostgreSQL via Prisma.]

## Target Users
Developers building task management into their applications.

## Functional Requirements
- **FR1:** Users can create tasks with title and description *(implemented)*
- **FR2:** Users can list all tasks with pagination *(implemented)*
- **FR3:** Users can update task status (pending/in_progress/completed) *(implemented)*
- **FR4:** Users can delete tasks *(implemented)*
- **FR5:** (Epic 1) Users can assign categories to tasks
- **FR6:** (Epic 1) Users can set priority levels (low/medium/high/critical) on tasks
- **FR7:** (Epic 1) Users can search tasks by keyword across title and description

## Non-Functional Requirements
- **NFR1:** API response time < 200ms for list operations
- **NFR2:** PostgreSQL 16 for persistence via Prisma ORM
- **NFR3:** TypeScript strict mode, ESLint enforced
- **NFR4:** Jest for testing, minimum 80% coverage on new code

## Technical Constraints
- Node.js 20+, Express 4, TypeScript 5, PostgreSQL 16
- Prisma ORM for database access
- Existing codebase has CRUD for tasks (FR1-FR4 implemented)
- New work: FR5-FR7 covered by Epic 1
```

### Architecture (architecture.md)

```markdown
# TaskManager API — Architecture

**Date:** 2026-03-09

## Stack
- **Runtime:** Node.js 20
- **Framework:** Express 4
- **Language:** TypeScript 5 (strict mode)
- **Database:** PostgreSQL 16
- **ORM:** Prisma 5

## Project Structure
[describir la estructura de test-fixtures/project/]

## API Endpoints (Existing)
- `GET    /api/tasks`      — List tasks (paginated)
- `POST   /api/tasks`      — Create task
- `PUT    /api/tasks/:id`  — Update task
- `DELETE /api/tasks/:id`  — Delete task

## API Endpoints (Epic 1 — New)
- `GET /api/tasks?category=X`     — Filter by category (Story 1.1)
- `GET /api/tasks?priority=high`  — Filter by priority (Story 1.2)
- `GET /api/tasks/search?q=keyword` — Full-text search (Story 1.3)

## Database Schema (Current)
```sql
CREATE TABLE tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR(255) NOT NULL,
  description TEXT,
  status VARCHAR(20) DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

## Shared Files Warning (Critical for BMO Testing)
The following files are modified by multiple stories and will trigger BMO's shared file mutation detection:
- `src/models/task.ts` — Story 1.1 adds `category`, Story 1.2 adds `priority`
- `src/routes/tasks.ts` — Story 1.1 adds category filter, Story 1.2 adds priority filter, Story 1.3 adds search endpoint
- `prisma/schema.prisma` — Story 1.1 adds category field, Story 1.2 adds priority enum + field
```

### Epic (epics.md)

CRITICO: Seguir este formato exactamente. Las stories van INLINE dentro del epic (resumen con Given/When/Then), y ADEMAS como archivos individuales expandidos en `implementation-artifacts/stories/`.

```markdown
# TaskManager API — Epic Breakdown

**Author:** Test Fixture
**Date:** 2026-03-09
**Version:** 1.0

---

## Overview

This epic adds task enrichment features to the existing TaskManager API: categories, priority levels, and search.

### Epic Summary

| Order | Epic | Title | Stories |
|-------|------|-------|---------|
| 1 | 1 | Task Enrichment | 3 |

### Shared File Matrix

| File | Story 1.1 | Story 1.2 | Story 1.3 |
|------|-----------|-----------|-----------|
| src/models/task.ts | adds category | adds priority | reads only |
| src/routes/tasks.ts | adds category filter | adds priority filter | adds search endpoint |
| prisma/schema.prisma | adds category field | adds priority enum + field | no changes |

---

## Epic 1: Task Enrichment

**Goal:** Add categories, priority levels, and search capabilities to the task API.
**User Value:** Users can organize, prioritize, and find tasks efficiently.
**Prerequisites:** Existing CRUD API (FR1-FR4) operational.

---

### Story 1.1: Task Categories

As a **developer**,
I want to assign categories to tasks,
So that tasks can be organized by topic.

**Acceptance Criteria:**

**Given** a task is being created
**When** the user provides a `category` field (string, optional)
**Then** the task is saved with the category value

**Given** tasks exist with various categories
**When** the user calls `GET /api/tasks?category=bug`
**Then** only tasks with category "bug" are returned

**Given** a task exists without a category
**When** the user calls `GET /api/tasks?category=bug`
**Then** that task is NOT included in the results

**Prerequisites:** None
**Story Points:** 3

---

### Story 1.2: Task Priority

As a **developer**,
I want to set priority levels on tasks,
So that high-priority items can be identified and addressed first.

**Acceptance Criteria:**

**Given** a task is being created or updated
**When** the user provides a `priority` field
**Then** the value must be one of: low, medium, high, critical

**Given** an invalid priority value is provided
**When** the API processes the request
**Then** it returns a 400 error with message "Invalid priority. Must be one of: low, medium, high, critical"

**Given** tasks exist with various priorities
**When** the user calls `GET /api/tasks?priority=high`
**Then** only tasks with priority "high" are returned

**Given** a task is created without specifying priority
**When** the task is saved
**Then** the default priority is "medium"

**Prerequisites:** None (can run in parallel with Story 1.1)
**Story Points:** 3

---

### Story 1.3: Task Search

As a **developer**,
I want to search tasks by keyword,
So that users can quickly find relevant tasks.

**Acceptance Criteria:**

**Given** tasks exist with various titles and descriptions
**When** the user calls `GET /api/tasks/search?q=deploy`
**Then** tasks containing "deploy" in title OR description are returned

**Given** a search query with no matches
**When** the user calls `GET /api/tasks/search?q=xyznonexistent`
**Then** an empty array is returned with 200 status

**Given** a search query is executed
**When** results are returned
**Then** the response should be fast and relevant

**Prerequisites:** None
**Story Points:** 5
```

### Stories Individuales (story-1-X.md)

CADA story sigue este formato EXACTO:

```markdown
# Story 1.X: [Titulo]

Status: ready-for-dev

## Story

As a **[role]**,
I want [capability],
So that [benefit].

## Acceptance Criteria

1. **Given** [precondition], **When** [action], **Then** [result].
2. **Given** [precondition], **When** [action], **Then** [result].
3. [minimo 3 ACs]

## Tasks / Subtasks

- [ ] **Task 1: [titulo]** (AC: #1, #2)
  - [ ] 1.1 [subtask con path de archivo]
  - [ ] 1.2 [subtask especifico]
- [ ] **Task 2: [titulo]** (AC: #3)
  - [ ] 2.1 [subtask]

## Dev Notes

### What Already Exists (DO NOT recreate)
- `src/models/task.ts` — Task interface with id, title, description, status, timestamps
- `src/routes/tasks.ts` — Express router with GET/POST/PUT/DELETE endpoints
- [etc.]

### Implementation Details
[codigo de ejemplo mostrando que cambiar]

### Shared Files Warning (si aplica)
[listar archivos que otras stories tambien modifican]
```

## Diseno Intencional de las 3 Stories

### Story 1.1: Task Categories — HAPPY PATH
- Proposito: validar el flujo basico de orquestacion sin complicaciones
- Archivos: `task.ts`, `tasks.ts`, `schema.prisma`
- 3 ACs claros, 3 tasks con subtasks
- NO comparte archivos con stories que causen conflictos graves

### Story 1.2: Task Priority — SHARED FILE MUTATION
- Proposito: testear **shared file mutation detection** de BMO (step-09 de implementation)
- Modifica los MISMOS archivos que Story 1.1: `task.ts`, `tasks.ts`, `schema.prisma`
- Marcar en Dev Notes: `SHARED FILES: task.ts, tasks.ts, and schema.prisma are also modified by Story 1.1`
- 4 ACs, 3-4 tasks
- Agrega un archivo nuevo: `src/middleware/validation.ts`

### Story 1.3: Task Search — AMBIGUOUS AC (corrective loop test)
- Proposito: testear el **corrective loop** de intellectual mode (cross-validation debe flaguear el AC ambiguo)
- El AC #3 es INTENCIONALMENTE AMBIGUO: "the response should be fast and relevant" — no define que es "fast" (ms?) ni "relevant" (ranking? threshold?)
- Modifica `tasks.ts` (COMPARTIDO) y crea `src/services/search.ts` (nuevo)
- 3 ACs (el #3 ambiguo), 2-3 tasks

## Proyecto Ficticio (test-fixtures/project/)

No necesita funcionar — es scaffold para dar contexto a las stories. Crear archivos minimos pero realistas. El codigo existente debe reflejar SOLO FR1-FR4 (CRUD basico). Las features de FR5-FR7 (categories, priority, search) NO existen aun — eso es lo que las stories implementan.

## Que NO Hacer

- NO instalar dependencias ni correr codigo
- NO crear archivos fuera de `test-fixtures/`
- NO usar placeholders — todo el contenido debe ser realista y completo
- NO omitir Dev Notes con "What Already Exists" + snippets — son criticos para sub-agent dispatch
- NO hacer el AC #3 de Story 1.3 claro — DEBE ser ambiguo a proposito

## Verificacion Final

Despues de crear todos los archivos, lista la estructura completa y verifica:
- [ ] Archivos de proyecto en `test-fixtures/project/` (package.json, tsconfig, prisma schema, src/, tests/)
- [ ] 3 archivos en `test-fixtures/planning-artifacts/` (prd.md, architecture.md, epics.md)
- [ ] 3 archivos en `test-fixtures/implementation-artifacts/stories/` (story-1-1.md, story-1-2.md, story-1-3.md)
- [ ] Todas las stories tienen `Status: ready-for-dev`
- [ ] Story 1.1 y 1.2 comparten archivos (task.ts, tasks.ts, schema.prisma) y esta documentado
- [ ] Story 1.3 AC #3 es intencionalmente ambiguo
- [ ] Dev Notes con "What Already Exists" + code snippets en las 3 stories
- [ ] epics.md tiene epic summary table + stories inline con Given/When/Then
- [ ] PRD cross-referencia el epic
- [ ] Architecture cross-referencia shared files
