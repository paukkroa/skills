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
7. **Favor the architecturally-correct design over the minimal diff.** When a requirement is awkward to satisfy within the current structure, the default answer is NOT the smallest change that makes it compile. Ask _"what is best for this codebase architecturally, even if it requires refactoring existing code?"_ and plan that. A plan whose central move is a symptom-level workaround (see the red flags in step 3) is a planning failure — even if it "works." Naming a larger refactor and letting the user decide beats silently shipping a patch that leaks debt.
8. **Verify the dependency graph before handoff.** A reversed dependency edge is what makes an implementer pick up the wrong bead and produce a partial implementation. After wiring dependencies you MUST run the verification in step 5 (`bd dep cycles`, `bd dep tree`, `bd ready`) and confirm the intended starting bead — and only it — is READY.

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
- **What's the RIGHT design, not just a working one?** Actively ask _"if I were building this part of the codebase from scratch today, how would it be shaped?"_ Then plan toward that shape. If the clean design requires refactoring existing code (widening a model, moving a seam, unifying two representations), name that refactor as a bead — do not route around it with a bolt-on. "Minimal" and "careful" are not the goal; correct locality is. A change that touches more files but leaves every module deep beats a one-line change that leaks a cross-cutting concern into a deep module.
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

**Workaround red flags — if the plan relies on any of these, stop and redesign.** Each is a symptom of forcing a requirement through the wrong seam. The right move is almost always to change the model or the seam, not to smuggle data past it:

- **Writing a field a library/model doesn't declare** (`object.__setattr__`, monkey-patching, `extra="forbid"` bypass, casting to `any` to attach a property). If a consumer needs a signal, the signal belongs in the interface the consumer actually reads — or the plan should model the thing properly. A field nobody reads is a silent no-op that _looks_ done.
- **A parallel mechanism that duplicates something the codebase already does.** Before inventing a "featured X" / "special-case Y" path, ask whether an existing first-class concept (a component, a rule, an existing seam) already carries that behavior. Reuse the concept; don't grow a second one beside it.
- **Modifying a shared contract/interface to carry one caller's bespoke need** — especially one crossing a repo/library boundary or one that future adapters (remote services, alternate implementations) would each have to re-support. Prefer expressing the need through the already-defined interface.
- **Leaking a cross-cutting concern (visibility, auth, tenancy) into a deep module** by adding flags every downstream stage must now check. Prefer set-aside/restore or a single seam over sprinkling `if is_visible` across the pipeline.
- **A change that satisfies the acceptance criterion literally but not functionally** — data flows in and is never consumed, or the "winner" is computed but nothing acts on it. Trace every new field to a real consumer before writing the bead.

For each red flag the plan can't avoid, write the proper refactor as a bead and record the trade-off in `docs/decisions/` (step 7). If a genuinely-correct design is blocked by an external constraint (e.g. a library field that doesn't exist yet), the bead must say so explicitly and specify a documented no-op plus a follow-up bead — never an invented API surface that appears to work.

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

**Wiring dependencies — get the direction right the first time.** A reversed edge makes the implementer claim the wrong bead and ship a partial implementation. `bd dep` has two equivalent forms; they take arguments in OPPOSITE orders, which is the single most common mistake:

```
bd dep <blocker> --blocks <blocked>     # blocker blocks blocked  ← prefer this form; it reads left-to-right
bd dep add <blocked> <blocker>          # blocked depends on blocker  ← note: REVERSED argument order
```

Both express the same edge: `<blocker>` must be done before `<blocked>` can start. **Always use the `--blocks` form** — its argument order matches the sentence "the foundation blocks the thing built on top of it," so it's far harder to invert.

Direction rule: the **foundation** (model changes, shared types, the thing everything else imports) is the blocker; it blocks the work that builds on it. The **feature bead** is blocked by everything (it's the leaf that "completes" last). Example, foundation → downstream → feature:

```bash
bd dep <model-bead> --blocks <rule-bead>       # model blocks the rule that uses it
bd dep <rule-bead>  --blocks <config-bead>     # rule blocks the config that wires it
bd dep <config-bead> --blocks <tests-bead>     # config blocks the e2e tests
bd dep <tests-bead> --blocks <feature-bead>    # everything blocks the feature bead
```

For bulk wiring, pass `--no-cycle-check` on each `bd dep` call, then verify once at the end (below).

**Verify the graph before handoff (mandatory — hard rule 8).** Reversed edges and cycles are silent until an implementer trips on them:

```bash
bd dep cycles                    # must report no cycles
bd dep tree <feature-bead-id>    # eyeball that foundation is at the bottom, feature at the top
bd ready                         # ONLY the intended starting bead(s) should appear as ready
```

If a bead you expected to be startable is missing from `bd ready`, or an unexpected one appears, an edge is reversed — fix it with `bd dep remove` and re-add before continuing. Do not produce the handoff until `bd ready` shows exactly the starting bead(s) you intend.

Present the dependency graph to the user (foundation at the leaves, feature at the root):
```
bead-xxx  Feature
  └── bead-aaa  Config/wiring
        └── bead-yyy  Rule/logic
              └── bead-zzz  Model/foundation  [READY]
```

**Handoff names the READY bead, not the feature bead, as the starting point.** The implementer starts from what `bd ready` returns (the foundation), not the feature bead (which is blocked). Telling the implementer to "start with the feature bead" is how they pick up work that isn't unblocked yet. State explicitly: "Start with `<ready-bead-id>`; run `bd ready` after each close to get the next unblocked bead."

For standalone bead review/cleanup without planning a new feature, use `/bead-review`.

### 6. Update CONTEXT.md

If new domain terms were resolved during grilling, update `CONTEXT.md` inline using the format in [CONTEXT-FORMAT.md](../grill-with-docs/CONTEXT-FORMAT.md). Only add terms specific to this project's domain — not general programming concepts.

### 7. Record decisions

When a design decision during grilling is hard to reverse, surprising without context, and the result of a real trade-off — record it as a decision file in `docs/decisions/`. See [CONTEXT-FORMAT.md](../grill-with-docs/CONTEXT-FORMAT.md) for the file format and numbering.

### 8. Commit and push documentation changes

If steps 6 or 7 created or modified files (CONTEXT.md, decision files in `docs/decisions/`), stage, commit, and push them:

```bash
git add CONTEXT.md docs/decisions/
git commit -m "docs: update domain glossary and decisions for <feature name>"
git push
```

If the push fails because no upstream is set, use `git push -u origin $(git branch --show-current)`.

Only commit documentation files — never code, stubs, or tests.

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

**Set it to block all task beads** so implementation cannot start until the questions are resolved. The open-questions bead is the BLOCKER (it must be resolved before any task starts), so it goes on the left of `--blocks`:

```bash
bd dep <open-questions-bead-id> --blocks <task-bead-id-1>
bd dep <open-questions-bead-id> --blocks <task-bead-id-2>
...
```

Verify with `bd ready` — while open questions exist, NO task bead should appear as ready. If a task bead is still ready, the edge is reversed.

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
