---
name: implement
description: Execute an implementation brief produced by the /plan-feature skill. Reads the brief, claims beads, implements changes layer by layer with test verification between each step, and closes beads when done. Use when you have a brief file (docs/briefs/*.md) and need to implement the specified changes.
model: sonnet
allowed-tools: Bash(bd *) Bash(git add *) Bash(git commit *) Bash(git push *) Bash(git branch *) Bash(git status *)
---

# Implement

Execute an implementation brief. You are the coding agent — read the brief, follow it precisely, and implement the changes.

## Hard rules

1. **Follow the brief exactly.** Design decisions in the brief are constraints, not suggestions. They were resolved during planning — do not re-litigate them.
2. **Git commits allowed on feature branches only.** Before any `git add` / `git commit` / `git push`, run `git branch --show-current` and verify the branch is NOT `main`, `master`, `dev`, `stg`, `qa`, or `prod`. If on a protected branch, stop and tell the user to create a feature branch first. On a feature branch: stage, commit, and push freely.
3. **Do NOT make design decisions.** If the brief is ambiguous, stop and describe the ambiguity — do not guess.
4. **Max 2 attempts** at any fix. If something isn't working after 2 tries, describe the problem and stop.
5. **Use beads for tracking.** `bd update <id> --claim` when starting a bead, `bd close <id>` when done.

## Process

### 1. Load the brief

Read the brief file passed as argument. If no file is passed, check `docs/briefs/` for the most recent brief.

Read `CONTEXT.md` for domain vocabulary and architecture context.

### 2. Verify preconditions

Before writing any code:

- Run `bd show <id>` for each bead in the brief — verify they exist and are open
- Check that files referenced in the brief still exist at the expected paths
- Run the test suite to establish a passing baseline (check `CONTEXT.md` or `bd recall test-command` for the exact command)
- Note the test count — you must not reduce it
- Check for a `## Test Harness` section in the brief. If present:
  - Tests were pre-written by `/write-tests`. Your job is to make them pass (red→green).
  - Read each test file listed. Understand what the tests expect.
  - Note the stub files — these define the interface you must implement.
  - Do NOT modify test assertions. If a test is wrong (tests the wrong behavior, not implementation preference), flag it and stop.

### 3. Execute bead by bead

Follow the execution order from the brief. For each bead:

**Claim:** `bd update <id> --claim`

**Read:** Re-read the target files. The brief has line numbers but they may have shifted if earlier beads modified the same files.

**Implement:** Follow the numbered steps in the brief. Use existing patterns — the brief points to examples in the codebase. Follow them exactly.

**Verify:** Run the project's test suite after each bead.
- If the brief has a Test Harness section: the target is making pre-written tests pass. Check which tests for THIS bead have turned green. Do NOT modify test assertions.
- If no Test Harness section: write your own tests as before.
- All previously passing tests must continue to pass.
- If tests fail unexpectedly (not the pre-written red tests turning green), fix within 2 attempts. If still failing, stop and report.

**Close:** `bd close <id>`

### 4. Layer-by-layer for cross-cutting changes

When a bead touches multiple architectural layers (models, scorers, routes, config, tests):

1. Domain models / ABCs first → run tests
2. Implementations (scorers, adapters) → run tests
3. Wiring (client config, route handler) → run tests
4. Config files (YAML, CacheConfig) → run tests

Do NOT implement all layers at once then test. Verify between each layer.

When a Test Harness exists, each layer should turn more pre-written tests green. Track progress — after each layer, note which pre-written tests now pass.

### 5. Final verification

After all beads are closed:

- Run the full test suite one final time
- Verify test count hasn't decreased
- Run `bd list --status=in_progress` — should be empty
- List all files modified for the user's review

## Constraints

These apply to every implementation:

- Use the project's configured runner (check `CONTEXT.md` or `bd recall runner`)
- Imports at top of files, never inside functions
- `__init__.py` files: only re-exports, never class/function definitions
- No comments unless the WHY is non-obvious
- Follow existing code patterns exactly — check similar implementations before creating new ones
- Do not over-engineer. Prefer the simplest approach that works.
- Do not remove hardcoded values or env vars without the brief explicitly saying to

## When stuck

If you hit a problem the brief doesn't cover:

1. Describe what you tried (max 2 attempts)
2. Show the error or unexpected behavior
3. State what you think the issue is
4. **Stop and wait** — do not keep trying

If the problem is bead-related (wrong dependency, stale bead, missing bead), use `/bead-review` instead of trying to fix it yourself.
