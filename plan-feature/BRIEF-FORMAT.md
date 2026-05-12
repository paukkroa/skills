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
<Numbered steps with specifics — file paths, class names, method signatures, line references>

**Design decisions:**
<Key decisions from grilling, quoted. These are constraints, not suggestions.>

**Test expectations:**
<What tests to write or update. What behavior to verify.>

---

<repeat for each bead>

## Execution order

<bead-id> → <bead-id> → (<bead-id> + <bead-id> parallel) → <bead-id>

## Constraints

- Do NOT run git add or git commit — user handles git
- Use `uv run` instead of `python`
- Imports at top of files, never inside functions
- __init__.py files: only re-exports, never definitions
- Run tests with `uv run pytest src/tests/` from the api/ directory
- Read `bd show <id>` before starting each bead
- Mark `bd update <id> --claim` when starting, `bd close <id>` when done
```

## Rules

- **Be concrete.** File paths, class names, method signatures. Not "update the scorer" but "add `anonymous_user_id` property to `LLBBatchItemScorer` in `api/src/app/clients/veikkaus/scorers/llb_scorers.py`."
- **Include current state.** If building on prior work, say what exists and what's remaining.
- **Quote design decisions.** The coding agent wasn't in the grilling session. It needs the WHY, not just the WHAT.
- **Layer the execution.** Group beads by dependency — what can run in parallel, what's sequential.
- **No ambiguity.** If there are two valid approaches, the planner already picked one. The brief states which.
