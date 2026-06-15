---
name: plan-feature
description: Convert business requirements into implementation-ready beads for a coding agent. Uses grilling interview style to resolve design decisions, explores codebase for constraints, and outputs structured beads. Use when starting new features, converting specs/PRDs into tasks, or preparing work for Codex/coding agents. NEVER writes code.
effort: high
allowed-tools: Bash(bd create *) Bash(bd update *) Bash(bd close *) Bash(bd dep *) Bash(bd show *) Bash(bd list *) Bash(bd ready *) Bash(bd graph *) Bash(bd remember *) Bash(bd recall *) Bash(bd prime *) Bash(bd doctor *) Bash(bd stale *) Bash(bd orphans *) Bash(bd blocked *) Bash(bd find-duplicates *) Bash(bd duplicate *) Bash(bd upgrade *) Bash(bd status *) Bash(git add *) Bash(git commit *) Bash(git push *) Bash(git branch *) Bash(git status *)
---

# Plan

Convert business requirements into implementation-ready beads. Output is a feature bead (goal + acceptance criteria) plus task beads (implementation specs) for a coding agent (Codex). This skill NEVER writes code.

## Hard rules

1. **Git commits allowed on feature branches only.** Before any `git add` / `git commit` / `git push`, run `git branch --show-current` and verify the branch is NOT `main`, `master`, `dev`, `stg`, `qa`, or `prod`. If on a protected branch, stop and tell the user to create a feature branch first. On a feature branch: stage, commit, and push freely.
2. **Do NOT write code.** No implementations, no stubs, no prototypes, no method bodies. Bead descriptions specify interfaces (function names, parameter types, return types) and behavioral contracts ("given X, return Y") — never HOW the code works internally. A code block in a bead description is almost always a mistake.
3. **Do NOT enter plan mode.** Stay in this skill's workflow.
4. **Max 2 exploration attempts** per question. If codebase exploration doesn't answer it, ask the user.
5. **Use beads for ALL task tracking.** No TodoWrite, TaskCreate, or markdown TODOs.
6. **Relative paths only in beads.** All file paths in bead descriptions MUST be relative to the repo root (e.g. `src/auth/models.py`). NEVER absolute paths. Agents may work in git worktrees where absolute paths point to the wrong directory.

## Process

### 1. Understand the domain

Read `CONTEXT.md` for domain glossary and architecture. Read `docs/decisions/` for recorded design decisions. Run `bd ready` and `bd list --status=open` to see existing work.

If any term in the user's request conflicts with `CONTEXT.md`, call it out: _"Your glossary defines X as Y, but you seem to mean Z — which is it?"_

### 2. Grill the requirements

Interview the user **one question at a time**, walking down each branch of the design tree. For each question:

- Provide your recommended answer with reasoning
- If the question can be answered by exploring the codebase, explore instead of asking
- Cross-reference against `CONTEXT.md` vocabulary — sharpen fuzzy language into precise canonical terms
- Stress-test with concrete scenarios that probe edge cases

**Open questions.** When the user can't answer a question (needs stakeholder input, needs research, says "I don't know yet"), add it to an **open questions list** instead of blocking. Label each with a short ID (Q1, Q2, ...) and note which design decisions depend on it. Continue grilling other branches that don't depend on the unanswered question.

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

Use a heredoc for the description to avoid permission prompts from `#` characters in quoted strings:

```bash
bd create --type feature --priority <0-4> \
  --title "<Feature name>" \
  --description "$(cat <<'BEAD'
Goal
<1-2 sentences: what this achieves and why>

Context files
- CONTEXT.md — domain glossary and architecture
- <file1> — <why to read it>

Acceptance Criteria
<MANDATORY. Behavioral 'when X then Y' assertions. See BEAD-SPEC-FORMAT.md for rules.>
BEAD
)"
```

**Do NOT use `#` headers inside bead descriptions.** Use plain text section labels (e.g. `Goal` not `## Goal`). The `#` character inside quoted shell arguments triggers security validation prompts.

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

### 7. Record decisions

When a design decision during grilling is hard to reverse, surprising without context, and the result of a real trade-off — record it as a decision file in `docs/decisions/`. See [CONTEXT-FORMAT.md](../grill-with-docs/CONTEXT-FORMAT.md) for the file format and numbering.

## Final message

### If open questions remain

**Do NOT produce the handoff.** Implementation must not start while open questions exist — unanswered questions mean the spec is incomplete and the implementer will guess wrong. Only skip this gate if the user explicitly says they want the handoff despite open questions (e.g. "give me the handoff anyway", "I'll figure it out during implementation").

Create an open questions bead and present a compact summary the user can take to stakeholders.

**Create the open questions bead:**

```bash
bd create --type task --priority 0 \
  --title "Resolve open questions for <feature name>" \
  --description "$(cat <<'BEAD'
Open Questions

Q1: <question>
Blocks: <which design decisions / task beads depend on the answer>

Q2: <question>
Blocks: <which design decisions / task beads depend on the answer>

Return with answers to continue planning.
BEAD
)"
```

**Set it to block all task beads** so implementation cannot start until the questions are resolved:

```bash
bd dep add <open-questions-bead-id> <task-bead-id-1>
bd dep add <open-questions-bead-id> <task-bead-id-2>
...
```

Present the open questions list in conversation so the user can take it to stakeholders.

When the user returns with one or more answers, incorporate them into the design, re-grill if the answer opens new branches, and update the open questions bead description to remove answered questions. **When all questions are resolved, close the open questions bead** (`bd close <id>`) — this unblocks the task beads — and produce the handoff.

### Handoff (all questions resolved, or user explicitly overrides)

Planning is done. The user picks their next step — do not implement yourself.

Provide all handoff formats so the user can paste into their chosen agent. **Include the feature bead ID** so the receiving skill/agent knows which feature to work on.

**Test-first (recommended) — write tests before implementation:**
```
/write-tests <feature-bead-id>
```

**Skip to implementation (tests written by implementer):**
```
/implement <feature-bead-id>
```

**For other agents (no skills):**
```
The feature bead is <feature-bead-id>. Run `bd show <feature-bead-id>` for goal, context files, and acceptance criteria. Run `bd list --status=open` to see all task beads. For each, run `bd show <id>` for the full spec. Check `bd graph` for execution order. Read CONTEXT.md for domain vocabulary. Implement beads in dependency order. Run tests after each bead. Claim with `bd update <id> --claim`, close with `bd close <id>`.
```
