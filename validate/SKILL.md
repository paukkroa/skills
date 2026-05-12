---
name: validate
description: Validate a completed implementation against business requirements, beads, and CONTEXT.md. Runs tests, checks end-to-end behavior with the local server, reviews architectural alignment, and identifies gaps. Use when a coding agent (Codex) has finished work and you want to verify it before committing.
---

# Validate

Review completed implementation against the original spec and beads. Focus is **functional correctness and end-to-end behavior**, not code style or test coverage metrics.

## Hard rules

1. **Do NOT fix bugs yourself.** Report them. The user decides whether to fix manually or send back to the coding agent.
2. **Max 2 debug attempts** when investigating an issue. If root cause isn't clear, describe what you see and ask the user.
3. **Use beads for tracking.** Create new beads for gaps found. Close beads that are verified complete.
4. **Functional focus.** Does the feature work as specified? Does anything break downstream? Not: is the code pretty?

## Process

### 1. Gather context

Read the brief file if it exists (`docs/briefs/<feature-name>.md`).
Run `bd list --status=in_progress` and `bd list --status=open` to see what was worked on.
Run `bd show <id>` for each relevant bead to get the original spec.

Read `CONTEXT.md` for domain vocabulary.

### 2. Check what changed

Run `git diff --name-only` to see modified files.
Run `git diff` on key files to understand the actual changes.

Verify against bead descriptions:
- Were all specified changes made?
- Were any beads only partially implemented?
- Were changes made that weren't in any bead? (scope creep)

### 3. Run tests

```bash
uv run pytest src/tests/ -x -q
```

If tests fail:
- Read the failure output
- Trace to root cause (max 2 attempts)
- Report the issue — do NOT fix it

If tests pass, note the count and move on.

### 4. Functional verification

This is the core of validation. Start the local server and test the feature end-to-end:

**Start server:**
```bash
make local-run
```

**Test each scenario from the spec:**
- Happy path with expected inputs
- Edge cases identified during planning (anonymous users, opt-out, missing data)
- Fallback/degradation scenarios
- Verify logs show correct state (scoring_fallback, x-request-id, timing)

**Check downstream impact:**
- Do existing scoring methods still work? (personalized, pass_through, random)
- Do business rules still execute?
- Does the page scorer still run when it should?
- Any new warnings or errors in logs that weren't there before?

Report findings as a table:

| Scenario | Expected | Actual | Status |
|---|---|---|---|
| Anonymous cache hit | 0ms scorer, scoring_fallback=anonymous | ... | PASS/FAIL |

### 5. Architectural review

Apply the [deep module lens](../improve-codebase-architecture/LANGUAGE.md):

- **Depth check:** Do new modules earn their interface cost? Apply the deletion test.
- **Seam check:** Are new seams real (2+ adapters) or hypothetical (1 adapter)?
- **Locality check:** Is related logic concentrated, or scattered across files?
- **CONTEXT.md alignment:** Do new terms match the glossary? Are new concepts documented?

Flag issues but don't block on them — architectural improvements are separate beads, not blockers for the current feature.

### 6. Report

Present a structured report:

```
## Validation Report: <feature name>

### Tests
- X passed, Y failed
- Failures: <list if any>

### Functional Verification
| Scenario | Expected | Actual | Status |
|---|---|---|---|

### Bead Status
- <bead-id>: COMPLETE / PARTIAL / NOT STARTED
  - <what's done / what's missing>

### Issues Found
1. <issue description> — severity: critical/minor
   - <root cause if identified>
   - <suggested fix direction>

### Architectural Notes
- <observations, not blockers>

### CONTEXT.md Updates Needed
- <new terms or corrections>

### Recommendation
SHIP / FIX THEN SHIP / SEND BACK TO CODING AGENT
```

### 7. Create beads for gaps

For any issue found:
- Critical: create bead with priority 1
- Minor: create bead with priority 3
- Architectural: create bead with priority 4

Close beads that are fully verified. Update partially-complete beads with notes on what's remaining.

### 8. Auto-generate fix brief (if SEND BACK)

When the recommendation is **SEND BACK TO CODING AGENT**, automatically generate a fix brief:

- Create/update beads for each issue found (step 7)
- Read the current codebase state (files changed, current line numbers)
- Write a new brief to `docs/briefs/<feature-name>-fix.md` following the same format as `/implement` ([BRIEF-FORMAT](../plan-feature/BRIEF-FORMAT.md))
- The fix brief includes:
  - Only the failing beads — not the whole feature
  - What was attempted and why it's wrong (from the validation report)
  - Exact file paths and line numbers in the CURRENT state (post-implementation, not pre)
  - Test commands to verify each fix
- Show the brief path in conversation so the user can hand it to Codex immediately

This closes the loop: `/validate` → fix brief → Codex → `/validate` again.

### 9. Update CONTEXT.md

If the implementation introduced new domain concepts or changed existing ones, update `CONTEXT.md` using [CONTEXT-FORMAT.md](../grill-with-docs/CONTEXT-FORMAT.md).
