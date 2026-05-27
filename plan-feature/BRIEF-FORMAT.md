# Coding Agent Brief Format

The brief is a self-contained document that a coding agent (Codex) can execute without conversation context.

## Structure

```md
# Brief: <Feature Name>

## Goal

<1-2 sentences: what this achieves and why>

## Context

<Read these files first, in order>

- `CONTEXT.md` — domain glossary and architecture
- `<file1>` — <why to read it>
- `<file2>` — <why to read it>

## Bead dependency graph

<paste the graph from planning>

## Per-bead instructions

### <bead-id> — <title>

**Files:** <exact paths>

**What to do:**
<Numbered steps with specifics — file paths, class names, method signatures (name + params + return type), behavioral description of each function. No method bodies, no code blocks, no algorithms. Describe WHAT each function does, not HOW.>

**Design decisions:**
<Key decisions from grilling, quoted. These are constraints, not suggestions.>

**Test expectations:**
<What tests to write or update. What behavior to verify. Flag existing tests that will break due to these changes (e.g. hardcoded registries, expected counts, fixture assumptions).>

---

<repeat for each bead>

## Execution order

<bead-id> → <bead-id> → (<bead-id> + <bead-id> parallel) → <bead-id>

## Acceptance Criteria

<MANDATORY. Behavioral assertions independent of code. These are the "Validation Contract" — what `/write-tests` turns into failing tests and `/validate` checks against.>

<Every user-visible outcome gets one criterion. Each must be:>
<- Testable without reading the implementation>
<- Specific enough to write a pass/fail assertion>
<- Independent of internal structure (no "function X calls Y")>

- "When a user signs up with a valid email, a verification email is sent within 30 seconds"
- "When the scorer times out after 950ms, the response uses cached anonymous scores and includes fallback_used: cached_anonymous in the latency report"
- "When a filmstrip contains custom_card items with a game reference, item_id equals game.gameName lowercased"
- "Existing block types must produce identical output to before this change"

<NOT acceptable:>
<- "Verify it works" — not testable>
<- "Check the response" — no expected value>
<- "Run make unit-test" — that's a verification step, not a criterion>

## Constraints

- Do NOT run git add or git commit — user handles git
- Use the project's configured runner (check `CONTEXT.md` or `bd recall runner`)
- Imports at top of files, never inside functions
- `__init__.py` files: only re-exports, never definitions
- Run tests using the project's test command (check `CONTEXT.md` or `bd recall test-command`)
- Read `bd show <id>` before starting each bead
- Mark `bd update <id> --claim` when starting, `bd close <id>` when done
```

## Rules

- **Be concrete.** File paths, class names, method signatures. Not "update the service" but "add `calculate_discount(order: Order, rules: DiscountRules) -> Discount` to `OrderProcessor` in `src/orders/processor.py`."
- **No code bodies.** Specify function signatures (name, params, return type) and behavioral contracts ("given X input, returns Y"). Never write method implementations, class bodies, or algorithm steps in code blocks. The brief defines the WHAT and the interface — the implementer and test writer decide the HOW. A brief with code bodies pre-commits everyone to one implementation and creates confirmation bias in testing.
- **Include current state.** If building on prior work, say what exists and what's remaining.
- **Quote design decisions.** The coding agent wasn't in the grilling session. It needs the WHY, not just the WHAT.
- **Layer the execution.** Group beads by dependency — what can run in parallel, what's sequential.
- **No ambiguity.** If there are two valid approaches, the planner already picked one. The brief states which.
- **Acceptance Criteria are mandatory.** Every brief must have behavioral "when X then Y" assertions. These are what `/write-tests` converts to failing tests and `/validate` checks against. Without them, testing and validation have no contract.
- **Test Harness section is optional.** If `/write-tests` runs before `/implement`, it appends a `## Test Harness` section with pre-written test files and stubs. The implementer uses these as red→green targets. If no Test Harness section exists, the implementer writes its own tests.
