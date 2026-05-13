---
name: validate
description: Validate a completed implementation against business requirements, beads, and CONTEXT.md. Runs tests, checks end-to-end behavior with the local server, reviews architectural alignment, and identifies gaps. Use when a coding agent (Codex) has finished work and you want to verify it before committing.
---

# Validate

Review completed implementation against the original spec and beads. Focus is **functional correctness and end-to-end behavior**, not code style or test coverage metrics.

## Hard rules

1. **Do NOT fix bugs yourself.** No code edits, no config changes, no YAML tweaks, no "temporary" workarounds. Report what's wrong and how to fix it. The user decides whether to fix manually or send back to the coding agent. Even obvious one-line fixes go through the process — create a bead, describe the fix direction, recommend FIX or SEND BACK.
   - **FIX** = implementation bugs, minor gaps. The design is sound, the code just needs fixes. Auto-generate a fix brief (step 8).
   - **SEND BACK** = design is wrong, scope is wrong, approach is wrong. The brief itself was flawed. Explain what's wrong with the design so the user can re-plan with `/plan-feature` (step 9).
2. **Max 2 debug attempts** when investigating an issue. If root cause isn't clear, describe what you see and ask the user.
3. **Use beads for tracking.** Create new beads for gaps found. Close beads that are verified complete.
4. **Functional focus.** Does the feature work as specified? Does anything break downstream? Not: is the code pretty?
5. **Fresh context.** Work from the brief, beads, and git diff only. Do not rely on the implementation conversation — you are a separate verifier, not the same agent that wrote the code.

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

Run the project's test suite (check `CONTEXT.md` or `bd recall test-command` for the exact command).

If tests fail:
- Read the failure output
- Trace to root cause (max 2 attempts)
- Report the issue — do NOT fix it

If tests pass, note the count and move on.

### 4. Functional verification

This is the core of validation — do NOT skip it. Unit tests verify code correctness, not feature correctness. Start the local server and test the feature end-to-end:

**Start the local server** using the project's run command (check `CONTEXT.md` or `bd recall run-command`).

**Check against Acceptance Criteria:** If the brief has an Acceptance Criteria section, verify each assertion. These are the primary pass/fail criteria.

**Test each scenario from the spec:**
- Happy path with expected inputs
- Edge cases identified during planning
- Fallback/degradation scenarios
- Error handling and boundary conditions

**Check downstream impact:**
- Do existing features still work as before?
- Any new warnings or errors in logs that weren't there before?
- Does the change introduce regressions in related modules?

**When external data is stale or unavailable:** If cached/upstream data doesn't include the new content needed for testing, verify code correctness through alternative means (direct API calls, different endpoints, log inspection). Note the limitation in the report — don't modify code or config to work around it.

Report findings as a table:

| Scenario | Expected | Actual | Status |
|---|---|---|---|
| Happy path | <from spec> | ... | PASS/FAIL |

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
   - <failed approaches, if any — what was tried and why it didn't work>

### Architectural Notes
- <observations, not blockers>

### CONTEXT.md Updates Needed
- <new terms or corrections>

### Recommendation
SHIP / FIX / SEND BACK

- **SHIP**: Feature works as specified. Commit.
- **FIX**: Implementation bugs or minor gaps. Design is sound. Fix brief auto-generated at `docs/briefs/<feature>-fix.md` — hand to `/implement`.
- **SEND BACK**: Design, scope, or approach is fundamentally wrong. Re-plan needed. Hand the Design Issues section below to `/plan-feature`.

### Design Issues (SEND BACK only)
- What's wrong with the current design
- Why re-implementing won't fix it
- What needs to change at the planning level
```

### 7. Create beads for gaps

For any issue found:
- Critical: create bead with priority 1
- Minor: create bead with priority 3
- Architectural: create bead with priority 4

Close beads that are fully verified. Update partially-complete beads with notes on what's remaining.

For larger bead hygiene issues (many stale beads, dependency tangles), hand off to `/bead-review`.

### 8. Auto-generate fix brief (if FIX)

When the recommendation is **FIX**, automatically generate a fix brief:

- Create/update beads for each issue found (step 7)
- Read the current codebase state (files changed, current line numbers)
- Write a new brief to `docs/briefs/<feature-name>-fix.md` following the same format as `/implement` ([BRIEF-FORMAT](../plan-feature/BRIEF-FORMAT.md))
- The fix brief includes:
  - Only the failing beads — not the whole feature
  - What was attempted and why it's wrong (from the validation report)
  - Exact file paths and line numbers in the CURRENT state (post-implementation, not pre)
  - Test commands to verify each fix
- Show the brief path in conversation so the user can hand it to `/implement` immediately

This closes the loop: `/validate` → fix brief → `/implement` → `/validate` again.

### 9. Explain design issues (if SEND BACK)

When the recommendation is **SEND BACK**, the problem is at the design level — re-implementing the same brief won't fix it. Do NOT generate a fix brief. Instead:

- Explain what's wrong with the current design or approach
- Explain why more implementation won't solve it (wrong abstraction, missing requirement, incorrect assumption, scope mismatch)
- Reference the specific brief sections or bead specs that need rethinking
- Suggest what questions `/plan-feature` should resolve in the next round

The user takes this back to `/plan-feature` for re-scoping. The planner produces a new brief, then the cycle restarts.

### 10. Update CONTEXT.md

If the implementation introduced new domain concepts or changed existing ones, update `CONTEXT.md` using [CONTEXT-FORMAT.md](../grill-with-docs/CONTEXT-FORMAT.md).
