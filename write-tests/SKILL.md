---
name: write-tests
description: Write failing tests from a brief BEFORE implementation exists. Creates interface stubs and tests that encode the behavioral contract. The implementer then makes these tests pass (red→green). Use after /plan-feature and before /implement to prevent confirmation bias in testing.
model: sonnet
allowed-tools: Bash(bd *) Bash(git add *) Bash(git commit *) Bash(git push *) Bash(git branch *) Bash(git status *)
---

# Write Tests

Read a brief produced by `/plan-feature` and write tests BEFORE any implementation exists. Output is failing tests (red) plus minimal interface stubs. The implementer makes them green.

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

## Process

### 1. Load context

Read the brief file passed as argument. If no file is passed, check `docs/briefs/` for the most recent brief.

Read `CONTEXT.md` for domain vocabulary and architecture context.

Run `bd show <id>` for each bead in the brief to confirm they exist and are open.

### 2. Discover test infrastructure

Before writing anything, understand how this project tests:

- **Test framework**: pytest, jest, vitest, go test, etc. Check `CONTEXT.md`, `bd recall test-command`, `package.json`, `pyproject.toml`, `Makefile`.
- **Test location**: `tests/`, `__tests__/`, co-located `*.test.*`, `*_test.go`, etc.
- **Test naming**: how existing tests name files, classes, functions.
- **Fixtures and helpers**: existing conftest.py, test factories, builders, shared setup. Reuse these — do not create parallel infrastructure.
- **Test runner command**: the exact command to run tests. Verify it works now by running the existing suite.

If the project has no test infrastructure at all, ask the user which framework to use (suggest the most common for the language/framework), create the minimal test config, and record the command in `bd remember test-command "<cmd>"`.

Record the current test count. You must not break existing tests.

### 3. Analyze the brief for test targets

For each bead in the brief, extract:

1. **The public interface** — function signatures, class methods, API endpoints, config shapes. From the "What to do" section.
2. **The behavioral contract** — what the interface must do. From "Test expectations" and "Acceptance Criteria."
3. **The boundary conditions** — edge cases, error handling, fallback behavior. From design decisions recorded during grilling.

Classify each test target by level. See [TEST-STRATEGY.md](TEST-STRATEGY.md) for detailed guidance.

| Level | What it tests | When to use |
|---|---|---|
| **Contract test** | A module's public interface with real inputs and expected outputs | Every bead that introduces or modifies a public interface. Primary level. |
| **Integration test** | Multiple modules wired together, exercising a real data path | Cross-bead interactions, data flow from input to output. At least one per brief. |
| **Acceptance test** | End-to-end behavior matching an Acceptance Criteria line | Each line in the brief's Acceptance Criteria section. |

Do NOT write tests for internal/private functions. Internal structure is the implementer's decision.

### 4. Create interface stubs

For each module that tests need to import, create a stub file:

- Place stubs at the EXACT file paths specified in the brief's "Files" sections.
- Export the EXACT names the brief specifies (function names, class names, constants).
- Include type signatures / parameter lists that match the brief's instructions.
- Bodies: `raise NotImplementedError("stub")` (Python), `throw new Error("not implemented")` (JS/TS), equivalent in other languages.
- For classes: stub all public methods mentioned in the brief. No private methods.
- For modules being MODIFIED (not created): do NOT overwrite existing code. Read the existing file first, then add only the new functions/methods with stub bodies after existing code.
- `__init__.py` files: only re-exports, never definitions (per project convention).

**What stubs must NOT contain:**
- Implementation logic, algorithms, or branching
- Private/internal methods
- Import of internal dependencies (the implementer decides internal structure)
- Docstrings describing HOW (describe WHAT the function does in one line, not the algorithm)

### 5. Write tests bead by bead

Follow the execution order from the brief. For each bead:

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
- Import the app/router from where the brief says it will be wired.
- Test request/response shapes, status codes, headers — the HTTP contract.
- The route handler stub returns 501 Not Implemented.

**For modifications to existing modules:**
- Read existing tests first. Understand what's already covered.
- Add new test functions for new behavior. Do NOT modify existing test assertions.
- If existing tests will break due to interface changes, note them in the Test Harness section but do NOT change them — the implementer handles test migration.

**Verify red:** Run the test suite. Every new test MUST fail. If any new test passes:
- The stub has logic it shouldn't → fix the stub
- The test isn't testing new behavior → rewrite the test

Record the failure output — it's the implementer's target.

### 6. Write acceptance tests

After all bead-level tests, write tests for each line in the brief's "Acceptance Criteria" section:

- End-to-end tests that exercise the full feature path.
- May overlap with integration tests — that's fine. Acceptance tests are the validation contract.
- Use the exact wording from Acceptance Criteria as test names or docstrings.

Verify they all fail (red).

### 7. Update the brief

Append a `## Test Harness` section to the brief file:

```md
## Test Harness

Written by `/write-tests` — these tests encode the behavioral contract.
The implementer makes them pass (red→green) without modifying test assertions.

### Stub files created
- `<path>` — <what it stubs>

### Test files created
- `<path>` — contract tests for <bead-id>
- `<path>` — integration tests
- `<path>` — acceptance tests

### Current test results
- Existing tests: X passing
- New tests: Y failing (expected — all red)
- Failure modes: <summary of NotImplementedError / not-implemented patterns>

### Rules for the implementer
1. Make all new tests pass without modifying test files.
2. If a test is wrong (tests the wrong behavior), flag it — do not silently change it.
3. Replace stub bodies with real implementations.
4. Existing tests must continue to pass.
5. If existing tests break due to interface changes, update them to match the new interface while preserving their behavioral assertions.
```

### 8. Final verification

- Run the full test suite one final time.
- Existing tests: all pass (count matches baseline from step 2).
- New tests: all fail with `NotImplementedError` or equivalent — not import errors, not syntax errors. Clean failures only.
- List all files created/modified for the user's review.

### 9. Handoff

Provide both handoff formats:

**For agents with the `/implement` skill:**
```
/implement docs/briefs/<feature-name>.md
```

**For other agents:**
```
Read the brief at docs/briefs/<feature-name>.md. A test harness already exists — see the "Test Harness" section. Your job is to make all failing tests pass (red→green) without modifying test assertions. Implement beads <id-1>, <id-2>, ... in that order. Run the test suite after each bead. Do not make design decisions — the brief has all constraints.
```

## Constraints

- Do NOT write tests for internal/private functions. Test the public interface only.
- Do NOT mock the module under test. Mocks are only for external dependencies.
- Do NOT create test infrastructure that duplicates existing project fixtures.
- Do NOT write tests that assert on implementation details (internal state, private method calls, execution order of internals).
- Prefer real objects over mocks. Use the dependency classification from [DEEPENING.md](../improve-codebase-architecture/DEEPENING.md): in-process (no mock), local-substitutable (use the stand-in), remote-owned (in-memory adapter), true-external (mock).
- If a bead's "Test expectations" are too vague to write a concrete test, stop and ask the user. Do not invent requirements.
- When the brief references a module from another bead in the same brief, use its stub. When it references an existing module NOT in the brief, import the real module.

## When stuck

If you hit a problem the brief doesn't cover:

1. Describe what you tried (max 2 attempts)
2. Show the error or unexpected behavior
3. State what you think the issue is
4. **Stop and wait** — do not keep trying
