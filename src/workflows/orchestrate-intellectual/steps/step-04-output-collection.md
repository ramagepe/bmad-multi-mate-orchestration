---
name: step-04-output-collection
description: "Gather and validate document artifacts from all sub-agents"
nextStepFile: './step-05-cross-validation.md'
phase: sequential
phase_number: 3
executor: orchestrator
max_rerun_attempts: 2
---

# Step 4: Output Collection

**Progress: Step 4 of 8** — Next: Cross-Validation
**Phase:** 3 (SEQUENTIAL — Orchestrator Collects)

---

## STEP GOAL

Gather processing results from all sub-agents, validate that artifacts exist and acceptance criteria are covered, and classify each story's processing status. This feeds the cross-validation step and corrective loop.

---

## MANDATORY EXECUTION RULES

- 🛑 VERIFY artifacts exist on disk — don't trust reports alone
- 📖 Parse every sub-agent's processing_report
- 🚫 NEVER assume success — independently verify artifact existence
- 🎯 Check acceptance criteria coverage for each story

---

## EXECUTION SEQUENCE

### 1. Parse Sub-Agent Reports

For each entry in `{dispatch_results}`:

**IF sub-agent returned a valid `processing_report`:**
- Parse the structured report
- Extract: status, artifacts_produced, acceptance_criteria_coverage, issues

**IF sub-agent output is unstructured (no valid report):**
- Mark as `report_missing`
- Attempt to extract any useful information from raw output
- Fall back to filesystem verification (step 2)

### 2. Independent Artifact Verification

For EACH story's output path, independently verify the sub-agent's claims:

```bash
# Check if output directory exists and has content
ls -la {output_path}

# List all artifacts produced
find {output_path} -type f -name "*.md" -o -name "*.yaml" -o -name "*.json"

# Check file sizes (empty files = failed generation)
find {output_path} -type f -empty
```

**Build verified results:**

```yaml
verified_result:
  story_id: "{story-id}"
  reported_status: "{from report}"
  verified_status: "{from filesystem verification}"
  artifacts_exist: boolean
  artifact_count: int
  artifacts_verified:
    - path: "{file path}"
      size_bytes: int
      non_empty: boolean
  empty_artifacts: list[string]     # Files that exist but are empty
  missing_artifacts: list[string]   # Reported but not found on disk
  discrepancies: list[string]       # Differences between report and reality
```

### 3. Acceptance Criteria Coverage Check

For EACH story, compare the reported AC coverage against the original story:

1. Re-read the original story file to extract acceptance criteria
2. Check the sub-agent's `acceptance_criteria_coverage` report
3. For each AC marked as `addressed: true`, spot-check by reading the artifact to confirm

```yaml
ac_verification:
  story_id: "{story-id}"
  total_acs: int
  acs_addressed: int
  acs_missing: int
  acs_detail:
    - criterion: "AC description"
      reported_addressed: true/false
      verified_in_artifact: true/false  # Did spot-check confirm?
      artifact_path: "{path}"
```

**Classification impact:**
- All ACs addressed AND verified → trust status
- ACs reported addressed BUT not found in artifact → **NEEDS_CORRECTION** (inaccurate report)
- ACs explicitly missing → **NEEDS_CORRECTION**

### 4. Check for Original File Modifications

Verify that sub-agents did NOT modify original story files by comparing against
the `{baseline_timestamps}` recorded in step 3 before dispatch:

```bash
# Compare current modification time against pre-dispatch baseline
for each file in baseline_timestamps:
  current_mtime = stat -c '%Y' {file.path}
  IF current_mtime != file.mtime:
    → FILE WAS MODIFIED — contract violation
```

**IF original files were modified:**
```
🚨 CONTRACT VIOLATION DETECTED
═══════════════════════════════════════

Story file {story_file_path} was modified by sub-agent.
Sub-agents must NEVER modify original story files.

Options:
[R] Restore original — user restores the file manually (e.g., from version control)
[K] Keep modification (acknowledge violation)
[X] Abort workflow
```

**Note:** Intellectual mode does not perform git operations. If the user needs to
restore the original file, they must do so outside this workflow (e.g., via their
own version control or backup).

### 5. Classify Processing Results

For each story, determine the overall status:

| Condition | Classification |
|-----------|---------------|
| Report says completed + artifacts verified + ACs covered | **PROCESSED** |
| Report says completed but some ACs missing | **NEEDS_CORRECTION** |
| Report says completed but no artifacts found | **PROCESSING_INCOMPLETE** |
| Report says completed but artifacts are empty | **PROCESSING_INCOMPLETE** |
| Report says failed with details | **FAILED** |
| Report says blocked | **BLOCKED** |
| No report / timeout | **UNKNOWN** |

### 6. Build Collection Summary

```yaml
collection_summary:
  total_stories: int
  processed: list[{story_id, artifact_count, ac_coverage_pct}]
  needs_correction: list[{story_id, missing_acs, issues}]
  failed: list[{story_id, reason}]
  blocked: list[{story_id, blocker}]
  unknown: list[{story_id}]
```

### 7. Present Collection Report to User

```
📦 OUTPUT COLLECTION REPORT
═══════════════════════════════════════

Stories Processed: {total}

  Successfully Processed:
  ✅ {story-id-1} — {artifact_count} artifacts, {ac_coverage}% ACs covered
  ✅ {story-id-2} — {artifact_count} artifacts, {ac_coverage}% ACs covered

  Needs Correction:
  ⚠️ {story-id-3} — Missing ACs: {missing_ac_list}

  Failed:
  ❌ {story-id-4} — {failure_reason}

Artifacts written to: {output_folder}

Summary: {processed_count} processed,
         {correction_count} need correction,
         {failed_count} failed
```

### 8. User Decision on Problem Cases

If there are failed/blocked/unknown stories:

```
How would you like to handle problem cases?

[P] Proceed with processed stories only — validate them now
[R] Re-run failed stories (max {max_rerun_attempts} re-runs remaining)
    NOTE: Circuit breaker — after {max_rerun_attempts} re-runs, this option 
    is removed. You must choose [P], [M], or [A].
[M] Manual intervention — I'll fix these myself
[A] Abort — stop the workflow
```

**Re-run tracking:**
```yaml
rerun_tracking:
  rerun_count: 0
  max_reruns: "{max_rerun_attempts}"
```

After each re-run: increment `rerun_count`. IF `rerun_count >= max_reruns`, remove [R] option on next presentation.

- **P**: Continue with processed stories only
- **R**: Re-dispatch processors for failed stories (loops back to step 3 for those)
- **M**: User fixes manually, then orchestrator re-verifies
- **A**: Abort workflow

### 9. Store Collection State

```yaml
collection_state:
  collection_summary: {from above}
  stories_for_validation: list[string]  # Final list moving to cross-validation
  stories_deferred: list[string]        # Set aside for now
  correction_needed: list[string]       # Will enter corrective loop after validation
  artifacts_index:
    - story_id: "{story-id}"
      artifact_path: "{path}"
      summary: "{brief summary of artifact content}"
```

### 10. Proceed to Cross-Validation

```
✅ Output collection complete.
   {validation_count} stories ready for cross-validation.
   Loading Step 5: Cross-Validation...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Every sub-agent's output parsed and validated
- ✅ Artifact existence independently verified on disk
- ✅ Acceptance criteria coverage checked per story
- ✅ Discrepancies between reports and reality flagged
- ✅ User informed and decided on problem cases

## FAILURE MODES

- ❌ Trusting sub-agent reports without filesystem verification
- ❌ Not checking acceptance criteria coverage
- ❌ Not detecting empty artifacts (zero-byte files)
- ❌ Proceeding with failed stories without user decision
- ❌ Not detecting original file modifications by sub-agents
