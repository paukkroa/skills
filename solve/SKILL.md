---
name: solve
description: >
  Single-session skill for narrow bugs and small tasks. Explores the issue, grills the user
  only when decisions are needed, creates beads, implements, and verifies — all in one pass.
  Use when the full plan → write-tests → implement → validate cycle is overkill.
model: sonnet
allowed-tools: Bash(bd create *) Bash(bd update *) Bash(bd close *) Bash(bd dep *) Bash(bd show *) Bash(bd list *) Bash(bd ready *) Bash(bd graph *) Bash(bd remember *) Bash(bd recall *) Bash(bd prime *) Bash(bd doctor *) Bash(bd stale *) Bash(bd orphans *) Bash(bd blocked *) Bash(bd find-duplicates *) Bash(bd duplicate *) Bash(bd upgrade *) Bash(bd status *) Bash(git add *) Bash(git commit *) Bash(git push *) Bash(git branch *) Bash(git status *)
---

# Solve

Single-session skill for narrow, well-scoped work. Explore the issue, make decisions, create beads, implement, and verify — all in one pass. For bugs, small features, targeted refactors, or anything where the full plan/implement/validate cycle is overkill.

## When to use this vs. the full cycle

Use `/solve` when:
- The scope is narrow — one bug, one small feature, one focused change
- You can understand the problem AND fix it in one session
- The change touches a small number of files (roughly 1-5)
- No stakeholder input is needed beyond the user in front of you

Use `/plan-feature` → `/implement` when:
- The scope is broad — multiple subsystems, new architecture, multi-day work
- Multiple people need to review the plan before implementation starts
- The work will be handed off to a different agent or session
- You need extensive grilling to resolve many design branches

## Hard rules

1. **Git commits allowed on feature branches only.** Before any `git add` / `git commit` / `git push`, run `git branch --show-current` and verify the branch is NOT `main`, `master`, `dev`, `stg`, `qa`, or `prod`. If on a protected branch, stop and tell the user to create a feature branch first. On a feature branch: stage, commit, and push freely.
2. **Beads before code.** Create ALL beads (feature + tasks) BEFORE writing or modifying any code. No edits, no new files, no test changes until beads exist. Beads define the scope; implementation follows from them.
3. **Grill only when blocked.** Do not interview the user for the sake of thoroughness. Explore the codebase first. Only ask when you face a genuine fork where the user's intent is ambiguous and the codebase does not resolve it.
4. **Max 2 attempts** at any fix. If something is not working after 2 tries, describe the problem and stop.
5. **Relative paths only in beads.** All file paths in bead descriptions MUST be relative to the repo root. NEVER absolute paths.
6. **Worktree awareness.** You may be running in a git worktree. See "Worktree setup" in the Process section.

## Process

### 1. Worktree setup

Determine your working root — all subsequent steps use this directory:

```bash
WORK_ROOT="$(pwd)"
git rev-parse --git-dir
```

If the output is an absolute path containing `/worktrees/`, you are in a git worktree. The worktree root IS your `$WORK_ROOT`. The main repo is a DIFFERENT directory — do NOT run commands there.

Critical rules when in a worktree:

- NEVER `cd` to the main repo. All commands run from `$WORK_ROOT`.
- If `bd recall test-command` or `CONTEXT.md` returns a command with an absolute path to the main repo, IGNORE the absolute path. Run from `$WORK_ROOT` instead.
- Read and edit files from `$WORK_ROOT`. When a bead says `src/auth/models.py`, that means `$WORK_ROOT/src/auth/models.py`.
- Beads database: if `bd list` fails, symlink: `ln -s "$(git rev-parse --git-dir | sed 's|/\.git/worktrees/.*||')/.dolt" .dolt`

If the output is `.git`, you are in the main repo. No special handling needed.

### 2. Explore the issue

Read `CONTEXT.md` for domain vocabulary. Run `bd list --status=open` to see existing work.

Then dig into the problem:

- Read the relevant files. Follow the code path end to end.
- If the user reported a bug: reproduce it first. Run the failing scenario. Capture the exact error or wrong behavior. If you cannot reproduce, say so and stop.
- If the user requested a feature or change: understand the current behavior, find the seam where the change belongs, and identify existing patterns to follow.
- Check for related beads that might overlap or conflict.

Spend enough time here to understand the full picture. Rushing to a fix without understanding the problem produces wrong fixes.

### 3. Grill if needed

Most narrow tasks need zero grilling. But if you hit a fork:

- You found two valid approaches and the codebase does not favor one
- The user's request is ambiguous in a way that changes the implementation
- A design decision has downstream consequences the user should know about

Then ask. One question at a time. Provide your recommended answer. Move on when resolved.

Do NOT grill for:
- Decisions the codebase already answers (follow existing patterns)
- Trivial implementation details (pick the simpler option and go)
- Theoretical edge cases unlikely to matter for a narrow fix

### 4. Create beads BEFORE any code changes

Create a feature bead and task bead(s) for what you are about to do. **No code may be written or modified until all beads for the work exist.** This is a hard gate — beads define the scope, and implementation follows from them.

Keep beads proportional to the work.

For a single-fix bug, one feature bead + one task bead is fine. For a small feature with 2-3 parts, one feature bead + 2-3 task beads.

Feature bead:

```bash
bd create --type feature --priority <0-4> \
  --title "<Short description>" \
  --description "$(cat <<'BEAD'
Goal
<1-2 sentences: what this fixes/adds and why>

Context files
- <file1> — <why relevant>

Acceptance Criteria
- When <condition>, <expected outcome>
BEAD
)"
```

Task bead(s):

```bash
bd create --type task --priority <0-4> \
  --title "<What to do>" \
  --description "$(cat <<'BEAD'
Files
<file> (<new/modify>)

What to do
1. <step>

Design decisions
- <key decision and why, if any>

Test expectations
- <what to test>
BEAD
)"
```

Set dependencies if order matters: `bd dep add <blocker-id> <blocked-id>`.

Do NOT use `#` headers inside bead descriptions. Use plain text section labels.

### 5. Implement

Run the test suite to establish a passing baseline. Note the test count.

Then implement bead by bead:

- `bd update <id> --claim`
- Make the changes. Follow existing patterns in the codebase.
- Run tests after each bead. All previously passing tests must continue to pass.
- Write or update tests for the new behavior.

Do NOT close beads during implementation. Beads stay in-progress until validation passes.

If a bead touches multiple architectural layers, go layer by layer with test runs between each.

### 6. Verify and close

After all task beads are implemented:

- Run the full test suite. Verify test count has not decreased.
- If the issue was a bug: re-run the original failing scenario and confirm it now passes.
- If the issue was a feature: verify the new behavior works as specified in the acceptance criteria.
- Close all task beads: `bd close <id>` for each completed task bead.
- Close the feature bead: `bd close <feature-id>`.
- Run `bd list --status=in_progress` — should be empty.
- List all modified files for the user.

### 7. Commit

Stage and commit on the feature branch. Write a clear commit message that explains the WHY, not just the WHAT. Reference the feature bead ID.

## Constraints

- Use the project's configured runner (check `CONTEXT.md` or `bd recall runner`)
- Imports at top of files, never inside functions
- `__init__.py` files: only re-exports, never definitions
- No comments unless the WHY is non-obvious
- Follow existing code patterns — check similar implementations before inventing new ones
- Do not over-engineer. Simplest approach that works.

## When stuck

If you hit a problem after 2 attempts:

1. Describe what you tried
2. Show the error or unexpected behavior
3. State what you think the issue is
4. Stop and wait — do not keep trying

## Scope creep check

If during exploration you discover the issue is larger than expected — touches many subsystems, needs stakeholder decisions, would benefit from test-first development — say so. Recommend switching to the full cycle:

```
This is bigger than a single-session solve. Recommend:
/plan-feature <description>
```

Do not try to force a large change through a single-session skill.
