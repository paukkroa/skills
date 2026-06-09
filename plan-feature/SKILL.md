---
name: plan-feature
description: Convert business requirements into implementation-ready beads for a coding agent. Uses grilling interview style to resolve design decisions, explores codebase for constraints, and outputs structured beads. Use when starting new features, converting specs/PRDs into tasks, or preparing work for Codex/coding agents. NEVER writes code.
effort: high
allowed-tools: Bash(bd *) Bash(git add *) Bash(git commit *) Bash(git push *) Bash(git branch *) Bash(git status *)
---

# Plan

Convert business requirements into implementation-ready beads. Output is a feature bead (goal + acceptance criteria) plus task beads (implementation specs) for a coding agent (Codex). This skill NEVER writes code.

## Hard rules

1. **Git commits allowed on feature branches only.** Before any `git add` / `git commit` / `git push`, run `git branch --show-current` and verify the branch is NOT `main`, `master`, `dev`, `stg`, `qa`, or `prod`. If on a protected branch, stop and tell the user to create a feature branch first. On a feature branch: stage, commit, and push freely.
2. **Do NOT write code.** No implementations, no stubs, no prototypes, no method bodies. Bead descriptions specify interfaces (function names, parameter types, return types) and behavioral contracts ("given X, return Y") — never HOW the code works internally. A code block in a bead description is almost always a mistake.
3. **Do NOT enter plan mode.** Stay in this skill's workflow.
4. **Max 2 exploration attempts** per question. If codebase exploration doesn't answer it, ask the user.
5. **Use beads for ALL task tracking.** No TodoWrite, TaskCreate, or markdown TODOs.

## Process

### 1. Understand the domain

Read `CONTEXT.md` for domain glossary, architecture, and recorded decisions. Run `bd ready` and `bd list --status=open` to see existing work.

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

### 4. Create the feature bead

Create one bead of `--type feature` that captures the high-level spec. This is the root — all task beads relate back to it.

```bash
bd create --type feature --priority <0-4> \
  --title "<Feature name>" \
  --description "## Goal
<1-2 sentences: what this achieves and why>

## Context files
- CONTEXT.md — domain glossary and architecture
- <file1> — <why to read it>

## Acceptance Criteria
<MANDATORY. Behavioral 'when X then Y' assertions. See BEAD-SPEC-FORMAT.md for rules.>"
```

**Acceptance Criteria are mandatory.** Every feature bead must include behavioral "when X then Y" assertions that are testable, implementation-independent, and cover every user-visible outcome. These are what `/write-tests` turns into failing tests and what `/validate` checks against. Vague checklists ("verify it works") are not acceptance criteria.

### 5. Create task beads with dependency graph

Create task beads using `bd create` with rich descriptions. See [BEAD-SPEC-FORMAT.md](BEAD-SPEC-FORMAT.md) for the full format.

Each task bead includes:
- `--title` — imperative, specific
- `--description` — structured with sections:
  - **Files** — exact paths to touch
  - **What to do** — numbered steps with file paths, class names, method signatures (name + params + return type), behavioral description. No method bodies, no code blocks, no algorithms. Describe WHAT each function does, not HOW.
  - **Design decisions** — key decisions from grilling, quoted. These are constraints, not suggestions.
  - **Test expectations** — what tests to write or update. What behavior to verify. Flag existing tests that will break.
- `--type` — task/bug
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

### 6. Update CONTEXT.md

If new domain terms were resolved during grilling, update `CONTEXT.md` inline using the format in [CONTEXT-FORMAT.md](../grill-with-docs/CONTEXT-FORMAT.md). Only add terms specific to this project's domain — not general programming concepts.

### 7. Record decisions in CONTEXT.md

When a design decision during grilling is hard to reverse, surprising without context, and the result of a real trade-off — record it in `CONTEXT.md` under the Decisions section. This keeps all project knowledge in one place.

## Final message: copy-paste handoff

Planning is done. The user picks their next step — do not implement yourself.

Provide all handoff formats so the user can paste into their chosen agent:

**Test-first (recommended) — write tests before implementation:**
```
/write-tests
```

**Skip to implementation (tests written by implementer):**
```
/implement
```

**For other agents (no skills):**
```
Run `bd list --status=open` to see all work. For each bead, run `bd show <id>` to get the full spec. Check `bd graph` for execution order. Read CONTEXT.md for domain vocabulary. Implement beads in dependency order. Run tests after each bead. Claim with `bd update <id> --claim`, close with `bd close <id>`.
```
