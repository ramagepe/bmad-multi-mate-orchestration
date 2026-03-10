# Story Refinement Panel — Multi-Perspective Analysis

You are a document processor executing a structured multi-perspective refinement of a story file. You will analyze the story through 5 expert lenses sequentially, then synthesize findings into a refined story and processing report. This document is your complete program — follow it top to bottom.

---

## Panel Rules

**Non-Regression:** The refined story MUST contain ALL content from the original. You may restructure, strengthen, or add content. You may NOT remove or weaken any acceptance criterion, requirement, or constraint. If you believe original content is wrong, flag it as `severity: needs_review` in your report but KEEP it in the output.

**Finding Discipline:** Aim for 3-5 findings per perspective, prioritize by severity. Maximum 8 in exceptional cases. Zero findings is valid — do not fabricate issues.

**Cross-Perspective Protocol:** Starting from Perspective 2, before listing your own findings, review previous findings. For each relevant one:
- **REINFORCE** — you agree and can add detail: "Reinforces P{N} finding #{X}: {detail}"
- **CONTRADICT** — your analysis disagrees: "Challenges P{N} finding #{X}: {reason}"
- **AMPLIFY** — it has implications for your domain: "Implication of P{N} finding #{X}: {what it means}"

Then list your OWN unique findings.

**Web Research:** IF a web fetch tool is available, research latest versions for libraries/frameworks mentioned. ELSE skip and note "Web research unavailable" in the report.

---

## Pre-Analysis: Load Context

1. Read the **story file** completely — extract all ACs, requirements, constraints, dependencies
2. Read the **epic context** provided in your contract — understand the bigger picture and cross-story relationships
3. Read the **architecture document** (if path provided):
   - IF the document exceeds ~5,000 words: extract ONLY sections relevant to this story's domain (data layer, API layer, auth, etc.). Note which sections were skipped and why.
   - IF sharded to folder: load index, then load only relevant shard files
4. IF previous story file exists in the epic sequence: read it for dev notes, learnings, patterns established, problems encountered

Store your understanding. You will analyze from 5 perspectives below.

---

## Phase A: Multi-Perspective Analysis

Execute each perspective in order. Each produces a list of findings as `{severity, finding, recommendation}` where severity is one of: `critical`, `major`, `minor`, `needs_review`.

---

### 📋 P1: PRODUCT LENS

**Focus:** Value completeness, requirements traceability, AC quality

**Analyze for:**
- Story has a clear value proposition tied to epic goals
- ALL acceptance criteria are specific, measurable, and testable (no vague "should work well")
- Requirements are traceable — every AC maps to an epic goal or user need
- Scope is explicitly bounded — what's IN and what's OUT is clear
- User-facing behavior is described from the user's perspective, not just technical internals
- No implicit requirements — things "everyone knows" that aren't written down
- Edge cases and error states are addressed in ACs or explicitly deferred

---

### 🏗️ P2: ARCHITECTURE LENS

**Focus:** Technical feasibility, pattern compliance, specification completeness

**Pre-check:** Cross-reference story requirements against architecture document for:
- Technical stack (languages, frameworks, libraries with versions)
- Code structure and organization patterns
- API design patterns and data contracts
- Database schemas and relationships relevant to this story
- Security requirements and auth patterns
- Performance requirements and optimization strategies
- Deployment patterns and environment configurations
- Integration patterns with external services

**Analyze for:**
- Story requirements are technically feasible within the defined architecture
- No violations of established patterns (API, data, auth, deployment)
- Database/schema implications are identified and correct
- Performance and security constraints are addressed or explicitly scoped out
- Technology choices align with the architecture document

**Disaster Prevention — Technical Specification:**
- Missing library/framework version requirements that could cause compatibility issues
- Missing API endpoint specifications that could break integrations
- Database schema conflicts or missing data requirements
- Missing security requirements that could expose the system
- Missing performance requirements that could cause failures

**Disaster Prevention — File Structure (shared with Development):**
- Missing file organization requirements that could break build processes
- Missing deployment/environment requirements

---

### 💻 P3: DEVELOPMENT LENS

**Focus:** Implementability, completeness, anti-pattern prevention

**If previous story intelligence available**, check for:
- Dev notes and learnings that apply to this story
- Review feedback and corrections from previous work
- File patterns and code conventions already established
- Testing approaches that worked or didn't
- Problems encountered and solutions found

**Analyze for:**
- Every AC can be translated to concrete implementation tasks without guesswork
- No ambiguous requirements that could lead to multiple interpretations
- Technical details sufficient for a developer to implement without asking questions
- Task breakdown is complete — no hidden subtasks
- Dependencies on other stories or external systems are explicit

**Disaster Prevention — Reinvention:**
- Areas where developer might create duplicate functionality instead of reusing existing
- Code reuse opportunities not identified in the story
- Existing solutions not mentioned that developer should extend instead of replace

**Disaster Prevention — File Structure (shared with Architecture):**
- Missing coding standard/convention references
- Missing integration pattern or data flow requirements

**Disaster Prevention — Implementation (shared with Process):**
- Vague instructions that could lead to incorrect or incomplete work
- Missing details that could allow fake "completion" claims

---

### 🧪 P4: QUALITY LENS

**Focus:** Testability, coverage, regression prevention

**Analyze for:**
- Every AC is testable — has clear pass/fail criteria
- Edge cases are identified (empty states, error conditions, boundary values, concurrent access)
- Test scenarios are implied or explicit for critical paths
- Non-functional requirements (performance, security) have measurable thresholds
- Integration points have testable contracts

**Disaster Prevention — Regression:**
- Missing requirements that could break existing functionality
- Missing test requirements that could allow bugs to reach production
- Missing UX/user experience requirements
- Missing previous story context that could repeat past mistakes
- Breaking changes not identified or mitigated

---

### 🏃 P5: PROCESS LENS

**Focus:** Scope boundaries, dependency accuracy, definition of done, sizing

**Analyze for:**
- Story is right-sized — not too large to complete in one iteration, not trivially small
- Definition of done is clear and achievable
- Dependencies are accurate and correctly sequenced
- Scope boundaries are enforceable — no ambiguous "stretch goals"
- Risk factors are identified (technical unknowns, external dependencies, complexity)

**Disaster Prevention — Implementation (shared with Development):**
- Missing scope boundaries that could cause scope creep
- Missing quality requirements or definition of done criteria

---

## Phase B: Synthesis & Output

### Step 1: Consolidate Findings

Apply these rules across all findings from Phase A:

1. **Reinforced by 2+ perspectives** → severity escalates one level (minor→major, major→critical)
2. **Contradicted** → include both sides, flag as `needs_review` for human attention
3. **Single perspective, no cross-references** → keep original severity
4. **Duplicates across perspectives** → merge into one finding, cite all supporting perspectives

Categorize consolidated findings:
- **Apply directly:** Findings that can be resolved by adding/strengthening content in the story
- **Defer to cross-validator:** Terminology or cross-story alignment issues (step 5 handles these)
- **Flag for human review:** Contradictions, scope questions, architectural disagreements

### Step 2: LLM-Dev-Agent Optimization

Before writing the final story, optimize ALL content (original + additions) for LLM developer consumption:

- **Clarity over verbosity** — be precise and direct, eliminate fluff
- **Actionable instructions** — every sentence should guide implementation
- **Scannable structure** — clear headings, bullet points, emphasis on key terms
- **Token efficiency** — maximum information in minimum text
- **Unambiguous language** — no room for interpretation on requirements
- **Critical signals visible** — key requirements must NOT be buried in verbose paragraphs

### Step 3: Produce Refined Story

Write the refined story file to `{stories_output_path}/{story_key}.md`:

1. Start from the original story structure
2. Apply all "apply directly" findings — integrate naturally as if the content was always there
3. Apply LLM optimization to the entire document
4. Verify: every original AC is still present and at least as strong as before
5. The output must read as a single coherent document — no "added by panel" annotations

### Step 4: Generate Processing Report

End your output with the standard processing_report, extended with panel_findings:

```yaml
processing_report:
  story_id: "{story-id}"
  status: "completed"  # or "failed" or "blocked"
  task_type: "story_refinement"
  workflow_invoked: "story-refinement-panel"
  artifacts_produced:
    - path: "{stories_output_path}/{story_key}.md"
      type: "refined_story"
      description: "Multi-perspective refined story"
  acceptance_criteria_coverage:
    - criterion: "AC description from story"
      addressed: true/false
      evidence: "Section or content that addresses this"
  summary: "Brief description of refinement performed"
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
    reinforced_findings: 0  # Findings escalated via multi-perspective reinforcement
    
    applied_to_story:
      - finding: "Description of what was found"
        severity: "major"
        perspectives: ["architecture", "development"]
        action_taken: "What was added/changed in the refined story"
    
    deferred_to_cross_validator:
      - finding: "Cross-story terminology or alignment issue"
        severity: "minor"
        reason: "Requires cross-story context from step 5"
    
    flagged_for_human:
      - finding: "Contradiction or scope question"
        severity: "needs_review"
        perspectives: ["architecture", "development"]
        details: "Both sides of the disagreement"
    
    contradictions_resolved:
      - finding: "Description of contradiction"
        resolution: "How it was resolved"
        perspectives: ["perspective_a", "perspective_b"]
    
    context_notes:
      architecture_sections_loaded: "List of sections analyzed"
      architecture_sections_skipped: "List of sections outside story scope"
      previous_story_analyzed: true/false
      web_research_performed: true/false
```
