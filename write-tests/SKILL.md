---
name: write-tests
description: Write failing tests from beads BEFORE implementation exists. Creates interface stubs and tests that encode the behavioral contract. The implementer then makes these tests pass (red->green). Use after /plan-feature and before /implement to prevent confirmation bias in testing.
model: sonnet
allowed-tools: Bash(bd create *) Bash(bd update *) Bash(bd close *) Bash(bd dep *) Bash(bd show *) Bash(bd list *) Bash(bd ready *) Bash(bd graph *) Bash(bd remember *) Bash(bd recall *) Bash(bd prime *) Bash(bd doctor *) Bash(bd stale *) Bash(bd orphans *) Bash(bd blocked *) Bash(bd find-duplicates *) Bash(bd duplicate *) Bash(bd upgrade *) Bash(bd status *) Bash(git add *) Bash(git commit *) Bash(git push *) Bash(git branch *) Bash(git status *)
---

# Write Tests

Read beads produced by `/plan-feature` and write tests BEFORE any implementation exists. Output is failing tests (red) plus minimal interface stubs. The implementer makes them green.

## Hard rules

1. **Tests MUST fail.** Every test you write must fail when run. If a test passes against a stub, it tests nothing — delete and rewrite it.
2. **Stubs define interface, not implementation.** Stubs contain function signatures, type definitions, class shells, and `raise NotImplementedError("stub")` / `throw new Error("not implemented")` bodies. No logic, no algorithms, no branching.
3. **Do NOT implement features.** If you catch yourself writing logic inside a stub, stop. Extract the signature and raise/throw.
4. **Do NOT enter plan mode.** Stay in this skill's workflow.
5. **Do NOT modify test assertions in existing tests.** Only add new test functions.
6. **Max 2 exploration attempts** per question. If codebase exploration doesn't answer it, ask the user.
7. **Use beads for context only.** Read beads with `bd show <id>` but do NOT claim or close them — they stay open for the implementer.
8. **Git commits allowed on feature branches only.** Before any `git add` / `git commit` / `git push`, run `git branch --show-current` and verify the branch is NOT `main`, `master`, `dev`, `stg`, `qa`, or `prod`. If on a protected branch, stop and tell the user to create a feature branch first. On a feature branch: stage, commit, and push freely.
9. **Follow existing test patterns exactly.** Use the project's test framework, directory structure, naming conventions, and fixture patterns. Explore before writing.
10. **Worktree awareness.** You may be running in a git worktree. All file paths are relative to repo root. Create stubs and tests at repo-relative paths from `pwd`. Run all commands from the current working directory. See "Worktree setup" in the Process section.

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
- **If `bd recall test-command` or `CONTEXT.md` returns a command with an absolute path to the main repo, IGNORE the absolute path.** Run the command from `$WORK_ROOT` instead.
- **Create stubs and test files relative to `$WORK_ROOT`.** When a bead says `src/auth/tokens.py`, write to `$WORK_ROOT/src/auth/tokens.py`.
- **Run the test suite from `$WORK_ROOT`.**
- **Beads database.** If `bd list` fails (database not found), symlink: `ln -s "$(git rev-parse --git-dir | sed 's|/\.git/worktrees/.*||')/.dolt" .dolt`

If the output is `.git`, you are in the main repo. No special handling needed — `$WORK_ROOT` is the repo root.

### 2. Load context

The feature bead ID is passed as argument (e.g. `/write-tests bead-42`). If no argument, run `bd list --type=feature --status=open` to find it.

Read the feature bead with `bd show <feature-bead-id>` for the goal, context files, and acceptance criteria.

Run `bd list --status=open` to find all task beads. Read each with `bd show <id>` to get the implementation spec.

Read `CONTEXT.md` for domain vocabulary and architecture context.

### 3. Discover test infrastructure

Before writing anything, understand how this project tests:

- **Test framework**: pytest, jest, vitest, go test, etc. Check `CONTEXT.md`, `bd recall test-command`, `package.json`, `pyproject.toml`, `Makefile`.
- **Test location**: `tests/`, `__tests__/`, co-located `*.test.*`, `*_test.go`, etc.
- **Test naming**: how existing tests name files, classes, functions.
- **Fixtures and helpers**: existing conftest.py, test factories, builders, shared setup. Reuse these — do not create parallel infrastructure.
- **Test runner command**: the exact command to run tests. Verify it works now by running the existing suite **from `$WORK_ROOT`** (strip any absolute paths from the stored command).

If the project has no test infrastructure at all, ask the user which framework to use (suggest the most common for the language/framework), create the minimal test config, and record the command in `bd remember test-command "<cmd>"`.

Record the current test count. You must not break existing tests.

### 4. Analyze beads for test targets

For each task bead, extract from its description:

1. **The public interface** — function signatures, class methods, API endpoints, config shapes. From the "What to do" section.
2. **The behavioral contract** — what the interface must do. From "Test expectations" and the feature bead's "Acceptance Criteria."
3. **The boundary conditions** — edge cases, error handling, fallback behavior. From design decisions recorded during grilling.

Classify each test target by level. See [TEST-STRATEGY.md](TEST-STRATEGY.md) for detailed guidance.

| Level | What it tests | When to use |
|---|---|---|
| **Contract test** | A module's public interface with real inputs and expected outputs | Every bead that introduces or modifies a public interface. Primary level. |
| **Integration test** | Multiple modules wired together, exercising a real data path | Cross-bead interactions, data flow from input to output. At least one per feature. |
| **Acceptance test** | End-to-end behavior matching an Acceptance Criteria line | Each line in the feature bead's Acceptance Criteria section. |

Do NOT write tests for internal/private functions. Internal structure is the implementer's decision.

### 5. Create interface stubs

For each module that tests need to import, create a stub file:

- Place stubs at the EXACT file paths specified in the bead's "Files" sections.
- Export the EXACT names the bead specifies (function names, class names, constants).
- Include type signatures / parameter lists that match the bead's instructions.
- Bodies: `raise NotImplementedError("stub")` (Python), `throw new Error("not implemented")` (JS/TS), equivalent in other languages.
- For classes: stub all public methods mentioned in the bead. No private methods.
- For modules being MODIFIED (not created): do NOT overwrite existing code. Read the existing file first, then add only the new functions/methods with stub bodies after existing code.
- `__init__.py` files: only re-exports, never definitions (per project convention).

**What stubs must NOT contain:**
- Implementation logic, algorithms, or branching
- Private/internal methods
- Import of internal dependencies (the implementer decides internal structure)
- Docstrings describing HOW (describe WHAT the function does in one line, not the algorithm)

### 6. Write tests bead by bead

Follow the execution order from `bd graph`. For each task bead:

**Write contract tests** for the bead's public interface:
- Import from the stub files.
- Test the behavioral contract from "Test expectations" — not the implementation.
- Use concrete, domain-meaningful test data. Use `CONTEXT.md` vocabulary in variable names and assertions.
- Each test function tests ONE behavior. Name it after the behavior: `test_expired_token_returns_401` not `test_validate_token`.
- Include at least: one happy path, one error/edge case, one boundary condition per public function or method.

**Write integration tests** where the bead touches other beads:
- Test data flow across module boundaries.
- Use real objects where possible. Stubs only for external dependencies (databases, third-party APIs). See [TEST-STRATEGY.md](TEST-STRATEGY.md) for fixture guidance by dependency category.

**For API endpoints that don't exist yet:**
- Write tests using the project's HTTP test client (e.g., FastAPI's TestClient, supertest, httptest).
- Import the app/router from where the bead says it will be wired.
- Test request/response shapes, status codes, headers — the HTTP contract.
- The route handler stub returns 501 Not Implemented.

**For modifications to existing modules:**
- Read existing tests first. Understand what's already covered.
- Add new test functions for new behavior. Do NOT modify existing test assertions.
- If existing tests will break due to interface changes, note them in the test harness metadata but do NOT change them — the implementer handles test migration.

**Verify red:** Run the test suite **from `$WORK_ROOT`**. Every new test MUST fail. If any new test passes:
- The stub has logic it shouldn't -> fix the stub
- The test isn't testing new behavior -> rewrite the test

Record the failure output — it's the implementer's target.

### 7. Write acceptance tests

After all bead-level tests, write tests for each line in the feature bead's "Acceptance Criteria" section:

- End-to-end tests that exercise the full feature path.
- May overlap with integration tests — that's fine. Acceptance tests are the validation contract.
- Use the exact wording from Acceptance Criteria as test names or docstrings.

Verify they all fail (red).

### 8. Create test harness bead

Create a task bead that captures everything the implementer needs to know about the test harness. Use `--type task` (not `--type test` — that is not a valid bd type). Use heredoc and plain text labels (no `#` headers) to avoid shell security prompts:

```bash
bd create --type task --priority 1 \
  --title "Test harness for <feature name>" \
  --description "$(cat <<'BEAD'
Test Harness

Feature bead: <feature-bead-id>
Baseline: <N> tests passing before this feature

Stub files
- <path/to/stub1.py>
- <path/to/stub2.py>

Test files
- <path/to/test_file1.py>
- <path/to/test_file2.py>

Breaking tests
<test files the implementer needs to update, or 'none'>

Notes
<any context the implementer needs — e.g. fixture choices, why certain mocks were used>
BEAD
)"
```

Then wire dependencies so the implementer sees it:
```bash
bd dep add <feature-bead-id> <test-harness-bead-id>
```

### 9. Final verification

- Run the full test suite **from `$WORK_ROOT`** one final time.
- Existing tests: all pass (count matches baseline from step 3).
- New tests: all fail with `NotImplementedError` or equivalent — not import errors, not syntax errors. Clean failures only.
- List all files created/modified for the user's review.

### 10. Handoff

Provide both handoff formats:

**For agents with the `/implement` skill:**
```
/implement <feature-bead-id>
```

**For other agents:**
```
The feature bead is <feature-bead-id>. Run `bd show <feature-bead-id>` for goal and acceptance criteria. A test harness bead exists — find it via `bd list --status=open` (titled "Test harness for ...") and run `bd show <id>` for stub and test file paths. Your job is to make all failing tests pass (red->green) without modifying test assertions. Implement beads in dependency order (check `bd graph`). Run the test suite after each bead. Do not make design decisions — the bead specs have all constraints.
```

## Constraints

- Do NOT write tests for internal/private functions. Test the public interface only.
- Do NOT mock the module under test. Mocks are only for external dependencies.
- Do NOT create test infrastructure that duplicates existing project fixtures.
- Do NOT write tests that assert on implementation details (internal state, private method calls, execution order of internals).
- Prefer real objects over mocks. Use the dependency classification from [DEEPENING.md](../improve-codebase-architecture/DEEPENING.md): in-process (no mock), local-substitutable (use the stand-in), remote-owned (in-memory adapter), true-external (mock).
- If a bead's "Test expectations" are too vague to write a concrete test, stop and ask the user. Do not invent requirements.
- When a bead references a module from another bead in the same feature, use its stub. When it references an existing module NOT in the feature, import the real module.

## When stuck

If you hit a problem the bead specs don't cover:

1. Describe what you tried (max 2 attempts)
2. Show the error or unexpected behavior
3. State what you think the issue is
4. **Stop and wait** — do not keep trying
