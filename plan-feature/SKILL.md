---
name: plan-feature
description: Convert business requirements into implementation specs and beads for a coding agent. Uses grilling interview style to resolve design decisions, explores codebase for constraints, and outputs structured briefs. Use when starting new features, converting specs/PRDs into tasks, or preparing work for Codex/coding agents. NEVER writes code.
---

# Plan

Convert business requirements into implementation-ready specs. Output is beads + a structured brief for a coding agent (Codex). This skill NEVER writes code.

## Hard rules

1. **Do NOT write code.** No implementations, no stubs, no prototypes. Only specs and beads.
2. **Do NOT enter plan mode.** Stay in this skill's workflow.
3. **Max 2 exploration attempts** per question. If codebase exploration doesn't answer it, ask the user.
4. **Use beads for ALL task tracking.** No TodoWrite, TaskCreate, or markdown TODOs.

## Process

### 1. Understand the domain

Read `CONTEXT.md` for domain glossary and architecture. Read relevant ADRs in `docs/adr/`. Run `bd ready` and `bd list --status=open` to see existing work.

If any term in the user's request conflicts with `CONTEXT.md`, call it out: _"Your glossary defines X as Y, but you seem to mean Z — which is it?"_

### 2. Grill the requirements

Interview the user **one question at a time**, walking down each branch of the design tree. For each question:

- Provide your recommended answer with reasoning
- If the question can be answered by exploring the codebase, explore instead of asking
- Cross-reference against `CONTEXT.md` vocabulary — sharpen fuzzy language into precise canonical terms
- Stress-test with concrete scenarios that probe edge cases

Focus areas:
- **Where does this live architecturally?** Use the [deep module lens](../improve-codebase-architecture/LANGUAGE.md): depth, seams, locality, leverage. Apply the deletion test to proposed new modules.
- **What's the fallback chain?** Every external call needs a degradation path.
- **What changes for each user state?** Anonymous, opt-out, personalized, error.
- **What's the cache/performance story?** TTLs, warming, stale-while-revalidate.
- **What breaks downstream?** Trace new inputs through the full processing pipeline. Each stage may have assumptions (filters, validators, serializers) that silently drop or corrupt data the pipeline hasn't seen before.

### 3. Assess architectural impact

Before creating beads, evaluate the structural impact using the [deep module principles](../improve-codebase-architecture/LANGUAGE.md):

- Does this create shallow modules? (Interface nearly as complex as implementation)
- Does it split tightly-coupled logic across seams?
- Could existing modules absorb this without interface growth?
- Apply the deletion test to any new module being proposed.

### 4. Create beads with dependency graph

Create beads using `bd create` with:
- `--title` — imperative, specific
- `--description` — WHY the issue exists and WHAT needs to be done. Include:
  - Files to touch (with current line numbers where relevant)
  - Design decisions made during grilling (quote the user's answers)
  - Architectural constraints from CONTEXT.md
  - What's already done vs remaining (if building on prior work)
- `--type` — task/bug/feature
- `--priority` — 0-4 (not high/medium/low)

Set up dependencies with `bd dep add`.

Present the dependency graph:
```
bead-xxx  Description
  ├── bead-yyy  Description
  │     └── bead-zzz  Description
  └── bead-aaa  Description
```

For standalone bead review/cleanup without planning a new feature, use `/bead-review`.

### 5. Update CONTEXT.md

If new domain terms were resolved during grilling, update `CONTEXT.md` inline using the format in [CONTEXT-FORMAT.md](../grill-with-docs/CONTEXT-FORMAT.md). Only add terms specific to this project's domain — not general programming concepts.

### 6. Offer ADRs sparingly

Only offer an ADR when ALL three conditions are met:
1. Hard to reverse
2. Surprising without context
3. Result of a real trade-off with genuine alternatives

Use [ADR-FORMAT.md](../grill-with-docs/ADR-FORMAT.md).

## Output: coding agent brief

After beads are created and user approves, generate a **coding agent brief** — a self-contained document that a coding agent (Codex) can execute without this conversation's context.

Write to `docs/briefs/<feature-name>.md` AND show a summary in conversation.

Brief format — see [BRIEF-FORMAT.md](BRIEF-FORMAT.md).

## Final message: copy-paste handoff

Planning is done. The user picks ONE implementation agent — do not implement yourself.

Provide both handoff formats so the user can paste into their chosen agent:

**For agents with the `/implement` skill:**
```
/implement docs/briefs/<feature-name>.md
```

**For other agents:**
```
Read the brief at docs/briefs/<feature-name>.md and implement beads <id-1>, <id-2>, ... in that order. Run the test suite after each bead. Do not make design decisions — the brief has all constraints.
```
