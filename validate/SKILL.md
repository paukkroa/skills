---
name: validate
description: Validate a completed implementation against business requirements, beads, and CONTEXT.md. Runs tests, checks end-to-end behavior with the local server, reviews architectural alignment, and identifies gaps. Use when a coding agent (Codex) has finished work and you want to verify it before committing.
model: sonnet
effort: high
allowed-tools: Bash(bd create *) Bash(bd update *) Bash(bd close *) Bash(bd dep *) Bash(bd show *) Bash(bd list *) Bash(bd ready *) Bash(bd graph *) Bash(bd remember *) Bash(bd recall *) Bash(bd prime *) Bash(bd doctor *) Bash(bd stale *) Bash(bd orphans *) Bash(bd blocked *) Bash(bd find-duplicates *) Bash(bd duplicate *) Bash(bd upgrade *) Bash(bd status *) Bash(git diff *) Bash(git log *) Bash(git branch *) Bash(git status *)
---

# Validate

Review completed implementation against the original bead specs. Focus is **functional correctness and end-to-end behavior**, not code style or test coverage metrics.

## Hard rules

1. **Git commits allowed on feature branches only.** Before any `git add` / `git commit` / `git push`, run `git branch --show-current` and verify the branch is NOT `main`, `master`, `dev`, `stg`, `qa`, or `prod`. If on a protected branch, stop and tell the user to create a feature branch first. On a feature branch: stage, commit, and push freely.
2. **Worktree awareness.** You may be running in a git worktree. All file paths are relative to repo root. Run tests, server, and all commands from the current working directory (`pwd`). Check logs relative to `pwd`. See "Worktree setup" in the Process section.
3. **Do NOT fix bugs yourself.** No code edits, no config changes, no YAML tweaks, no "temporary" workarounds. Report what's wrong and how to fix it. The user decides whether to fix manually or send back to the coding agent. Even obvious one-line fixes go through the process — create a bead, describe the fix direction, recommend SHIP, FIX, or SEND BACK.
   - **FIX** = implementation bugs, minor gaps. The design is sound, the code just needs fixes. Auto-generate fix beads (step 9).
   - **SEND BACK** = design is wrong, scope is wrong, approach is wrong. The bead specs themselves were flawed. Explain what's wrong with the design so the user can re-plan with `/plan-feature` (step 10).
4. **Max 2 debug attempts** when investigating an issue. If root cause isn't clear, describe what you see and ask the user.
5. **Use beads for tracking.** Create new beads for gaps found. Close beads that are verified complete.
6. **Functional focus.** Does the feature work as specified? Does anything break downstream? Not: is the code pretty?
7. **Fresh context.** Work from the beads, CONTEXT.md, and git diff only. Do not rely on the implementation conversation — you are a separate verifier, not the same agent that wrote the code.
8. **Runtime evidence required.** At least the happy path MUST be verified by starting the server and making a real request. "Math verified" or "code looks correct" is not sufficient for a SHIP recommendation. If you cannot start the server, say so explicitly — never silently substitute code reading for runtime testing.
9. **Trace the full data path.** Before verifying individual pieces, map the data flow from source (config/input) through every intermediary function to the final consumer (API response/side effect). Verify each handoff — a correct function that never receives its input is a broken feature.

## Process

### 1. Worktree setup

Check if you are in a git worktree:

```bash
git rev-parse --git-dir
```

If the output is an absolute path containing `/worktrees/` (e.g. `/path/to/main/.git/worktrees/feature-x`), you are in a worktree. In that case:

- **All file paths are relative to `pwd`.** Bead specs use repo-relative paths. These resolve correctly from the worktree root.
- **Run tests, server, and log checks from `pwd`.** The worktree has the full working tree.
- **Use `git diff` normally.** Git commands work correctly in worktrees.
- **Beads database.** If `bd list` fails (database not found), the `.dolt/` directory is in the main repo. Symlink it: `ln -s "$(git rev-parse --git-dir | sed 's|/\.git/worktrees/.*||')/.dolt" .dolt` — or ask the user.

If the output is `.git`, you are in the main repo. No special handling needed.

### 2. Gather context

Run `bd list` to see all beads and their status.
Run `bd show <id>` for each relevant bead — both the feature bead (goal + acceptance criteria) and task beads (implementation specs).

Read `CONTEXT.md` for domain vocabulary.

### 3. Check what changed

Run `git diff --name-only` to see modified files.
Run `git diff` on key files to understand the actual changes.

Verify against bead descriptions:
- Were all specified changes made?
- Were any beads only partially implemented?
- Were changes made that weren't in any bead? (scope creep)

### 4. Run tests

Run the project's test suite (check `CONTEXT.md` or `bd recall test-command` for the exact command).

If tests fail:
- Read the failure output
- Trace to root cause (max 2 attempts)
- Report the issue — do NOT fix it

If tests pass, note the count and move on.

### 5. Trace the data path

Before testing individual scenarios, map the complete data flow for the new feature. This catches wiring bugs that unit tests miss (tests often construct inputs directly, bypassing real loading/routing).

1. **Identify the path**: From the source of truth (config file, API input, database) through every function/module to the final consumer (API response, side effect, UI).
2. **Verify each handoff**: At every boundary where data passes from one function/module to another, confirm the receiving function actually gets the data. Read the intermediary code — don't assume "it's wired up" because the bead says so.
3. **Question test fixtures**: If unit tests construct their own input data (hand-built dicts, mock configs), flag this. Tests that bypass real loading paths give false confidence. Note which integration paths are NOT covered by tests.

Report the data path in the verification table with a dedicated row, e.g.:

| Scenario | Expected | Actual | Status |
|---|---|---|---|
| Data flows from YAML config -> get_config() -> route -> compute() | field present at each step | get_config() drops the key | FAIL |

### 6. Functional verification

This is the core of validation — do NOT skip it. Unit tests verify code correctness, not feature correctness.

**Start the local server.** This is mandatory, not optional. Use the project's run command (check `CONTEXT.md` or `bd recall run-command`). If the server won't start, that's a finding — report it.

**Make real requests.** Hit the actual endpoint with inputs that exercise the new feature. Check the response for the new fields/behavior. At minimum:
- One happy-path request with expected inputs — verify the response contains the new data
- One edge-case request (missing config, boundary values) — verify graceful handling

**Check against Acceptance Criteria:** Read the feature bead's Acceptance Criteria section and verify each assertion. These are the primary pass/fail criteria.

**Inspect logs.** After making requests, check server logs for:
- Expected log entries from the new code path
- Unexpected errors or warnings
- Signs that the new code path wasn't actually exercised (missing expected logs)

**Test each scenario from the spec:**
- Happy path with expected inputs
- Edge cases identified during planning
- Fallback/degradation scenarios
- Error handling and boundary conditions

**Check downstream impact:**
- Do existing features still work as before?
- Any new warnings or errors in logs that weren't there before?
- Does the change introduce regressions in related modules?

**When the server can't be started or external data is unavailable:** Note this prominently in the report as a limitation. Verify code correctness through alternative means (direct function calls in a REPL, log inspection, reading intermediary code). But never give a SHIP recommendation without runtime evidence — downgrade to "SHIP (conditional — runtime untested)" and explain what the user should verify manually.

Report findings as a table. At least one row must come from an actual HTTP response or runtime observation, not code reading:

| Scenario | Expected | Actual | Status |
|---|---|---|---|
| Happy path (live request) | field = 5.0 | HTTP response shows field = 5.0 | PASS |
| Data path integrity | config -> route -> compute | verified at each handoff | PASS |

### 7. Architectural review

Apply the [deep module lens](../improve-codebase-architecture/LANGUAGE.md):

- **Depth check:** Do new modules earn their interface cost? Apply the deletion test.
- **Seam check:** Are new seams real (2+ adapters) or hypothetical (1 adapter)?
- **Locality check:** Is related logic concentrated, or scattered across files?
- **CONTEXT.md alignment:** Do new terms match the glossary? Are new concepts documented?

Flag issues but don't block on them — architectural improvements are separate beads, not blockers for the current feature.

### 8. Report

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
- **FIX**: Implementation bugs or minor gaps. Design is sound. Fix beads auto-generated — hand to `/implement`.
- **SEND BACK**: Design, scope, or approach is fundamentally wrong. Re-plan needed. Hand the Design Issues section below to `/plan-feature`.

### Design Issues (SEND BACK only)
- What's wrong with the current design
- Why re-implementing won't fix it
- What needs to change at the planning level
```

### 9. Create beads for gaps

For any issue found:
- Critical: create bead with priority 1
- Minor: create bead with priority 3
- Architectural: create bead with priority 4

Close beads that are fully verified. Update partially-complete beads with notes on what's remaining.

For larger bead hygiene issues (many stale beads, dependency tangles), hand off to `/bead-review`.

### 10. Auto-generate fix beads (if FIX)

When the recommendation is **FIX**, automatically generate fix beads:

- Create beads for each issue found (step 8) with implementation-ready descriptions following the [BEAD-SPEC-FORMAT](../plan-feature/BEAD-SPEC-FORMAT.md)
- Each fix bead includes:
  - Only the failing aspect — not the whole feature
  - What was attempted and why it's wrong (from the validation report)
  - Exact file paths and line numbers in the CURRENT state (post-implementation, not pre)
  - Test commands to verify each fix
- Set up dependencies between fix beads if needed
- Show a summary in conversation so the user can hand off to `/implement`

This closes the loop: `/validate` -> fix beads -> `/implement` -> `/validate` again.

### 11. Explain design issues (if SEND BACK)

When the recommendation is **SEND BACK**, the problem is at the design level — re-implementing won't fix it. Do NOT generate fix beads. Instead:

- Explain what's wrong with the current design or approach
- Explain why more implementation won't solve it (wrong abstraction, missing requirement, incorrect assumption, scope mismatch)
- Reference the specific bead specs that need rethinking
- Suggest what questions `/plan-feature` should resolve in the next round

The user takes this back to `/plan-feature` for re-scoping. The planner produces new beads, then the cycle restarts.

### 12. Update CONTEXT.md

If the implementation introduced new domain concepts or changed existing ones, update `CONTEXT.md` using [CONTEXT-FORMAT.md](../grill-with-docs/CONTEXT-FORMAT.md).
