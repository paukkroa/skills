---
name: implement
description: Execute implementation from beads produced by /plan-feature. Reads bead specs, claims beads, implements changes layer by layer with test verification between each step, and closes beads when done.
model: sonnet
allowed-tools: Bash(bd create *) Bash(bd update *) Bash(bd close *) Bash(bd dep *) Bash(bd show *) Bash(bd list *) Bash(bd ready *) Bash(bd graph *) Bash(bd remember *) Bash(bd recall *) Bash(bd prime *) Bash(bd doctor *) Bash(bd stale *) Bash(bd orphans *) Bash(bd blocked *) Bash(bd find-duplicates *) Bash(bd duplicate *) Bash(bd upgrade *) Bash(bd status *) Bash(git add *) Bash(git commit *) Bash(git push *) Bash(git branch *) Bash(git status *)
---

# Implement

Execute an implementation from beads. You are the coding agent — read the bead specs, follow them precisely, and implement the changes.

## Hard rules

1. **Follow the bead specs exactly.** Design decisions in bead descriptions are constraints, not suggestions. They were resolved during planning — do not re-litigate them.
2. **Git commits allowed on feature branches only.** Before any `git add` / `git commit` / `git push`, run `git branch --show-current` and verify the branch is NOT `main`, `master`, `dev`, `stg`, `qa`, or `prod`. If on a protected branch, stop and tell the user to create a feature branch first. On a feature branch: stage, commit, and push freely.
3. **Do NOT make design decisions.** If a bead spec is ambiguous, stop and describe the ambiguity — do not guess.
4. **Max 2 attempts** at any fix. If something isn't working after 2 tries, describe the problem and stop.
5. **Use beads for tracking.** `bd update <id> --claim` when starting a bead, `bd close <id>` when done.
6. **Worktree awareness.** You may be running in a git worktree. All file paths are relative to repo root. Run tests and all commands from the current working directory (`pwd`). See "Worktree setup" in the Process section.

## Process

### 1. Worktree setup

Determine your working root — all subsequent steps use this directory:

```bash
WORK_ROOT="$(pwd)"
git rev-parse --git-dir
```

If the output is an absolute path containing `/worktrees/`, you are in a git worktree. The worktree root IS your `$WORK_ROOT`. The main repo is a DIFFERENT directory — do NOT run commands there.

**Critical rules when in a worktree:**

- **NEVER `cd` to the main repo.** All commands run from `$WORK_ROOT`.
- **If `bd recall test-command` or `CONTEXT.md` returns a command with an absolute path to the main repo, IGNORE the absolute path.** Run the command from `$WORK_ROOT` instead. For example, if the stored command is `cd /Users/joe/project && pytest`, just run `pytest` from `$WORK_ROOT`.
- **Read and edit files from `$WORK_ROOT`.** When a bead says `src/auth/models.py`, that means `$WORK_ROOT/src/auth/models.py`.
- **Beads database.** If `bd list` fails (database not found), symlink: `ln -s "$(git rev-parse --git-dir | sed 's|/\.git/worktrees/.*||')/.dolt" .dolt`

If the output is `.git`, you are in the main repo. No special handling needed — `$WORK_ROOT` is the repo root.

### 2. Load beads

The feature bead ID is passed as argument (e.g. `/implement bead-42`). If no argument, run `bd list --type=feature --status=open` to find it.

Read the feature bead with `bd show <feature-bead-id>` for the goal, context files, and acceptance criteria.

Run `bd list --status=open` and `bd ready` to find task beads. Read `bd show <id>` for each to get the implementation spec.

Read `CONTEXT.md` for domain vocabulary and architecture context.

### 3. Verify preconditions

Before writing any code:

- Run `bd show <id>` for each task bead — verify they exist and are open
- Check that files referenced in bead descriptions still exist at the expected paths (relative to `$WORK_ROOT`)
- Run the test suite **from `$WORK_ROOT`** to establish a passing baseline (check `CONTEXT.md` or `bd recall test-command` for the command name — strip any absolute paths and run from `$WORK_ROOT`)
- Note the test count — you must not reduce it
- Check for a test harness bead: look for a bead titled "Test harness for ..." in `bd list --status=open`. If one exists:
  - Tests were pre-written by `/write-tests`. Your job is to make them pass (red->green).
  - Run `bd show <test-harness-bead-id>` to get stub file paths and test file paths.
  - Read each test file listed. Understand what the tests expect.
  - Note the stub files — these define the interface you must implement.
  - Do NOT modify test assertions. If a test is wrong (tests the wrong behavior, not implementation preference), flag it and stop.

### 4. Execute bead by bead

Follow the execution order from `bd graph`. For each task bead:

**Claim:** `bd update <id> --claim`

**Read:** Run `bd show <id>` to get the full spec. Re-read the target files. The bead has line numbers but they may have shifted if earlier beads modified the same files.

**Implement:** Follow the numbered steps in the bead's "What to do" section. Use existing patterns — the bead points to examples in the codebase. Follow them exactly.

**Verify:** Run the project's test suite **from `$WORK_ROOT`** after each bead.
- If a test harness exists: the target is making pre-written tests pass. Check which tests for THIS bead have turned green. Do NOT modify test assertions.
- If no test harness exists: write your own tests as before.
- All previously passing tests must continue to pass.
- If tests fail unexpectedly (not the pre-written red tests turning green), fix within 2 attempts. If still failing, stop and report.

**Close:** `bd close <id>`

### 5. Layer-by-layer for cross-cutting changes

When a bead touches multiple architectural layers (models, scorers, routes, config, tests):

1. Domain models / ABCs first -> run tests
2. Implementations (scorers, adapters) -> run tests
3. Wiring (client config, route handler) -> run tests
4. Config files (YAML, CacheConfig) -> run tests

Do NOT implement all layers at once then test. Verify between each layer.

When a test harness exists, each layer should turn more pre-written tests green. Track progress — after each layer, note which pre-written tests now pass.

### 6. Final verification

After all beads are closed:

- Run the full test suite **from `$WORK_ROOT`** one final time
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
- Do not remove hardcoded values or env vars without the bead spec explicitly saying to

## When stuck

If you hit a problem the bead spec doesn't cover:

1. Describe what you tried (max 2 attempts)
2. Show the error or unexpected behavior
3. State what you think the issue is
4. **Stop and wait** — do not keep trying

If the problem is bead-related (wrong dependency, stale bead, missing bead), use `/bead-review` instead of trying to fix it yourself.
