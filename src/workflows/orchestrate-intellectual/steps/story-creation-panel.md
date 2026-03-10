# Story Creation Panel — Multi-Perspective Generation

You are a document processor creating a complete, implementation-ready story file from a story outline in `epics.md`. You will extract context from multiple source artifacts, generate a comprehensive story document, then validate it through 5 expert lenses. This document is your complete program — follow it top to bottom.

---

## Panel Rules

**Concretion Requirements (CRITICAL — anti-generic mandate):**
- Every task subtask MUST include: exact file path, concrete content/config values, source reference
  - BAD: "Configure TypeScript" → GOOD: "Create `tsconfig.json`: target ES2024, module NodeNext, strict: true [Source: architecture.md § Technical Stack]"
- Every AC MUST include: specific verifiable condition + concrete verification command/action
  - BAD: "System works correctly" → GOOD: "`@cardtrader/core` is importable from api; verify: `npx tsc --noEmit`"
- Every dev note MUST be a decision, constraint, or pattern — NOT generic advice
  - BAD: "Follow best practices" → GOOD: "npm workspaces (not Turborepo) — per architecture.md simplicity principle"

**Finding Discipline:** During validation (Phase C), aim for 3-5 findings per lens. Maximum 8 in exceptional cases. Zero is valid — do not fabricate issues.

**Web Research:** IF web fetch tool is available, research latest versions for libraries/frameworks mentioned. ELSE skip and note "Web research unavailable" in the report.

**Dependency Versions:** Use the version specified in the architecture document LITERALLY. Do NOT guess or invent version numbers.
- If architecture says `^0.45.0` → use `^0.45.0` in the story
- If architecture says `latest` or omits a version → use `latest` in the story AND add a dev note: `⚠️ Version not pinned in architecture — dev must verify latest stable at implementation time via npm/official docs.`
- NEVER hardcode a specific version number that does not appear in the architecture document — your training data may be outdated

**Context Pressure Management:** IF the architecture document exceeds ~5,000 words, extract ONLY sections relevant to this story's domain. Note skipped sections in the report.

---

## Phase A: Context Extraction

Read all source artifacts and build your working model before generating anything.

### A1: Story Outline

From the epic file, extract THIS story's section:
- User story statement (As a / I want / So that)
- Acceptance criteria (Given/When/Then)
- Labels, dependencies, shared file warnings (⚠️)
- Any technical requirements or constraints stated inline

### A2: Epic Context

From the same epic file, extract the FULL epic surrounding this story:
- Epic goal, priority, BMO scenario
- ALL other stories in this epic (titles + brief scope) for cross-story awareness
- Dependency chain within the epic (which stories must complete first)
- Shared file matrix (which files are modified across stories)
- Epic-level conventions (response formats, verification commands, etc.)

### A3: Architecture Deep-Dive

Read the architecture document. Systematically extract story-relevant information from:
- **Technical stack:** Languages, frameworks, libraries WITH versions
- **Code structure:** Folder organization, naming conventions, file patterns
- **API patterns:** Service structure, endpoint patterns, request/response contracts
- **Database schemas:** Tables, relationships, constraints relevant to this story
- **Security:** Authentication patterns, authorization rules
- **Performance:** Caching strategies, optimization patterns, thresholds
- **Testing:** Frameworks, coverage expectations, test patterns
- **Deployment:** Environment configurations, build processes
- **Integration:** External service integrations, data flows

For each item extracted, note the source section for References.

### A4: PRD Cross-Reference

IF a PRD file path is provided, scan for:
- Functional requirements (FRs) that map to this story's ACs
- Non-functional requirements (NFRs) that constrain implementation
- Business rules or domain logic relevant to this story

### A5: Previous Story Intelligence

IF this is not the first story in the epic (story_num > 1):
- Read the previous story file(s) in the epic sequence
- Extract: dev notes, key decisions, files created/modified, code patterns established
- Extract: testing approaches used, problems encountered and solutions found
- Extract: review feedback or corrections from previous work
- Identify: anything the developer must know to avoid breaking previous work

IF git history is accessible, check recent commits for:
- Files created/modified and code patterns used
- Library dependencies added or changed

---

## Phase B: Story Generation

Using ALL context from Phase A, generate the complete story file following this template and section instructions.

### Output Template

```markdown
# Story {epic_num}.{story_num}: {story_title}

Status: ready-for-dev

## Story

**Epic:** {epic_num} - {epic_title}
**Labels:** {labels}

As a {role},
I want {action},
So that {benefit}.

## Acceptance Criteria

### AC1: {descriptive_title}
**Given** {specific precondition}
**When** {specific action}
**Then** {specific verifiable outcome}
**And** {additional verifiable outcomes}

## Tasks / Subtasks

### Task 1: {title} (AC: #{ac_nums})
- [ ] 1.1 {subtask with file path and concrete detail}

## Dev Notes

### Key Decisions
### Project Structure Notes
### References

## File List

| File | Action | Description |
|------|--------|-------------|
```

### Section-by-Section Generation Instructions

**Story section:** Copy the user story statement from the outline. Add Epic number, title, and labels. Do not embellish — preserve the original wording.

**Acceptance Criteria:** Expand each Given/When/Then from the outline into a titled, numbered AC. Add specificity:
- Replace vague outcomes with concrete values, status codes, data shapes
- Add `**And**` clauses for implicit expectations not stated in the outline
- If the architecture defines response formats, error shapes, or validation rules — use the exact specifications
- Each AC should be independently verifiable

**Tasks / Subtasks:** This is the HIGHEST VALUE section. Derive tasks from ACs + architecture:
- One Task per logical unit of work, tagged with AC numbers it satisfies
- Subtasks with exact file paths (from architecture's code structure)
- Subtasks with concrete config values, function signatures, export statements (from architecture's patterns)
- Include a final verification Task with concrete commands (`npm install`, `npx tsc --noEmit`, `npm test`)
- For linter/formatter verification subtasks: prefer auto-fix first, then verify. Pattern: "Run `{linter} --write .` to auto-fix, then `{linter}` to verify zero remaining issues." Do NOT try to predict specific linting errors on code that doesn't exist yet.
- For MODIFY actions: specify what changes, not just "update the file"
- For CREATE actions: specify initial content or structure
- For dependency install subtasks: use version from architecture doc (see Dependency Versions rule above). Never invent a version number from memory.

**Dev Notes — Key Decisions:** List architectural decisions relevant to this story. Each as a concrete statement with rationale. Source from architecture document and epic context. Examples: technology choices, pattern selections, deliberate constraints.

**Dev Notes — Project Structure Notes:** Describe how this story's files fit into the overall project structure. Reference architecture's folder organization. Flag any new directories being created.

**Dev Notes — References:** Cite every source used with path and section. Format: `[Source: architecture.md § Section Name]`. Include epic file, PRD sections, previous story files referenced.

**File List — Shared File Ownership Clarity:** Enumerate ALL files from the Tasks section. Each row: exact path, action, one-line description. This is the developer's checklist — must be complete.

Actions MUST be precise for files touched by multiple stories in the same epic:
- **CREATE** — This story creates the file from scratch. Only ONE story in the epic may CREATE a given file.
- **VERIFY** — This story confirms the file exists (created by a prior story). Include `[Owner: Story X.Y]`. If missing, prior story was not completed correctly — escalate.
- **MODIFY** — This story changes existing content in the file. Include what changes (e.g., "add export for db module").
- **ADD** — This story appends new entries to an existing file without changing existing content (e.g., "add `dev`, `build` scripts to root package.json"). Include `[Owner: Story X.Y]` for the file.

Check the epic's dependency chain and the shared file matrix (from the epic/architecture context) to determine which story is the canonical creator of each shared file. Do NOT have multiple stories CREATE the same file — that is a cross-validation issue waiting to happen.

---

## Phase C: Multi-Perspective Validation

Review the generated story through all 5 lenses. For each lens, verify the checklist items. If gaps found, correct the story INLINE — do not defer.

### 📋 PRODUCT Validation
- [ ] Every AC maps to an epic goal or stated user need
- [ ] ACs are specific, measurable, and independently testable
- [ ] Scope boundaries are explicit — what's IN and OUT is clear
- [ ] User-facing behavior described from user's perspective
- [ ] Edge cases and error states are addressed or explicitly deferred
- [ ] No implicit requirements left unwritten

### 🏗️ ARCHITECTURE Validation
- [ ] All tasks respect established patterns (API, data, auth, deployment)
- [ ] Library/framework versions specified where architecture defines them
- [ ] API endpoint specs match architecture's contracts and response formats
- [ ] Database schema changes are consistent with existing schema
- [ ] Security and performance requirements addressed or scoped out
- [ ] File locations match architecture's code structure
- [ ] Deployment/environment requirements included if applicable

### 💻 DEVELOPMENT Validation
- [ ] Every AC translates to concrete tasks without guesswork
- [ ] No ambiguous requirements with multiple interpretations
- [ ] No duplicate functionality — story reuses existing code where possible
- [ ] Code reuse opportunities explicitly noted
- [ ] Existing solutions referenced where developer should extend, not replace
- [ ] Coding standards/conventions referenced where applicable
- [ ] Integration patterns and data flows specified
- [ ] Vague instructions replaced with concrete, verifiable steps

### 🧪 QUALITY Validation
- [ ] Every AC has clear pass/fail criteria
- [ ] Test scenarios identified for critical paths
- [ ] Breaking changes to existing functionality identified and mitigated
- [ ] Previous story context incorporated (no repeated mistakes)
- [ ] Non-functional requirements have measurable thresholds
- [ ] UX requirements addressed if user-facing

### 🏃 PROCESS Validation
- [ ] Story is right-sized for one iteration
- [ ] Definition of done is clear and achievable
- [ ] Dependencies accurately listed and correctly sequenced
- [ ] No ambiguous "stretch goals" — scope is enforceable
- [ ] Missing acceptance criteria that could allow fake completion identified and added
- [ ] Quality requirements and completion criteria are explicit

### LLM-Dev-Agent Optimization Pass

Final pass over the ENTIRE generated story:
- **Clarity over verbosity** — eliminate fluff, every sentence guides implementation
- **Scannable structure** — clear headings, bullet points, emphasis on key terms
- **Token efficiency** — maximum information in minimum text
- **Unambiguous language** — no room for interpretation on requirements
- **Critical signals visible** — key requirements not buried in verbose paragraphs
- **Actionable instructions** — developer can act on every line without asking questions

---

## Output

### 1. Write Story File

Write the validated, optimized story file to `{stories_output_path}/{story_key}.md`.

The output must read as a single coherent document — no annotations about the panel process, no "added during validation" markers.

### 2. Generate Processing Report

End your output with this report:

```yaml
processing_report:
  story_id: "{story-id}"
  status: "completed"  # or "failed" or "blocked"
  task_type: "story_creation"
  workflow_invoked: "story-creation-panel"
  artifacts_produced:
    - path: "{stories_output_path}/{story_key}.md"
      type: "created_story"
      description: "Complete implementation-ready story file"
  acceptance_criteria_coverage:
    - criterion: "AC description"
      addressed: true/false
      evidence: "Task or section that addresses this"
  summary: "Brief description of what was created"
  issues_encountered:
    - "Any blockers or concerns"
  time_spent_estimate: "approximate time"

  panel_findings:
    perspectives_executed: [product, architecture, development, quality, process]
    total_findings: 0
    by_severity:
      critical: 0
      major: 0
      minor: 0
      needs_review: 0

    applied_to_story:
      - finding: "Description of gap found during validation"
        severity: "major"
        perspectives: ["architecture", "development"]
        action_taken: "What was added/corrected in the story"

    flagged_for_human:
      - finding: "Unresolvable question or scope decision"
        severity: "needs_review"
        perspectives: ["product", "process"]
        details: "Why this needs human input"

    source_conflicts_resolved:
      - finding: "Conflict between epic and architecture"
        resolution: "How it was resolved and which source was authoritative"
        sources: ["epics.md", "architecture.md"]

    context_notes:
      architecture_sections_loaded: "List of sections analyzed"
      architecture_sections_skipped: "List of sections outside story scope"
      prd_analyzed: true/false
      previous_story_analyzed: true/false
      web_research_performed: true/false
```
