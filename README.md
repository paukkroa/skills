# Claude Code Skills

A collection of custom skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that implement a structured **plan → write-tests → implement → validate** development workflow.

## Skills included

| Skill | What it does |
|-------|-------------|
| **init-project** | One-time project onboarding. Explores the codebase, generates `CONTEXT.md`, sets up beads tracking, and prepares the workflow. |
| **plan-feature** | Converts business requirements into implementation-ready beads. Grills you on design decisions. Never writes code. |
| **write-tests** | Writes failing tests from beads BEFORE implementation. Creates interface stubs and a test harness. Prevents confirmation bias. |
| **implement** | Makes pre-written tests pass (red→green), or writes its own if no test harness exists. Claims beads, implements layer by layer. |
| **validate** | Reviews completed implementation against beads and `CONTEXT.md`. Reports issues — never fixes them directly. |
| **bead-review** | Standalone bead housekeeping — status review, dependency management, stale/orphan cleanup. |
| **grill-with-docs** | Stress-tests a plan against your domain model (`CONTEXT.md`). Sharpens terminology inline. |
| **improve-codebase-architecture** | Finds deepening opportunities — refactors that turn shallow modules into deep ones for testability and AI-navigability. |

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and/or [Claude Code SDK (Codex)](https://docs.anthropic.com/en/docs/claude-code) installed
- Skills directory exists (created automatically by each tool):
  - Claude Code: `~/.claude/skills/`
  - Codex: `~/.agents/skills/`

### Clone the repo

```bash
git clone <repo-url> ~/Documents/skills
```

### Create symlinks

Each skill folder needs a symlink in the skills directory. Link the ones you want:

**Claude Code** (`~/.claude/skills/`):

```bash
# Link all skills at once
for skill in ~/Documents/skills/*/; do
  name=$(basename "$skill")
  ln -sf "$skill" ~/.claude/skills/"$name"
done
```

**Codex** (`~/.agents/skills/`):

```bash
# Link all skills at once
for skill in ~/Documents/skills/*/; do
  name=$(basename "$skill")
  ln -sf "$skill" ~/.agents/skills/"$name"
done
```

Or link individual skills (substitute the target directory as needed):

```bash
ln -sf ~/Documents/skills/init-project     ~/.claude/skills/init-project
ln -sf ~/Documents/skills/plan-feature     ~/.claude/skills/plan-feature
ln -sf ~/Documents/skills/write-tests      ~/.claude/skills/write-tests
ln -sf ~/Documents/skills/implement        ~/.claude/skills/implement
ln -sf ~/Documents/skills/validate         ~/.claude/skills/validate
ln -sf ~/Documents/skills/bead-review      ~/.claude/skills/bead-review
ln -sf ~/Documents/skills/grill-with-docs  ~/.claude/skills/grill-with-docs
ln -sf ~/Documents/skills/improve-codebase-architecture ~/.claude/skills/improve-codebase-architecture
```

### Verify

```bash
ls -la ~/.claude/skills/   # Claude Code
ls -la ~/.agents/skills/   # Codex
```

Symlinks should point back to cloned repo. Skills available immediately — no restart needed.

### Uninstall

Remove symlinks (not the repo):

```bash
# Claude Code
for skill in ~/Documents/skills/*/; do
  rm -f ~/.claude/skills/"$(basename "$skill")"
done

# Codex
for skill in ~/Documents/skills/*/; do
  rm -f ~/.agents/skills/"$(basename "$skill")"
done
```

## Example workflow

The core loop is **init → plan → write-tests → implement → validate**. Here's what that looks like on a real project.

### 1. Initialize the project (one-time)

Open Claude Code in your project directory and run:

```
/init-project
```

This explores your codebase and creates:
- **`CONTEXT.md`** — domain glossary, architecture overview, key conventions, and recorded decisions
- **Beads tracking** — lightweight task tracking via the `bd` CLI

You only run this once per project. It's read-only — no code changes.

### 2. Plan a feature

```
/plan-feature Add user email verification on signup
```

The skill will:
1. Read your `CONTEXT.md` for domain context and recorded decisions
2. **Grill you** on design decisions one question at a time (email provider? verification link expiry? retry behavior?)
3. Explore your codebase to find constraints and existing patterns
4. Create a **feature bead** with the goal, context, and acceptance criteria
5. Create **task beads** with implementation-ready specs for each unit of work

Beads are the single source of truth — a coding agent reads them via `bd show <id>` to get the full spec.

### 3. Write tests (recommended)

```
/write-tests
```

The skill reads the beads and writes tests BEFORE any implementation exists:
1. Discovers the project's test infrastructure (framework, fixtures, conventions)
2. Creates interface stubs at the file paths from the bead specs (function signatures with `NotImplementedError` bodies)
3. Writes contract tests, integration tests, and acceptance tests — all failing (red)
4. Stores test harness metadata in beads (`bd remember`) for the implementer

This prevents confirmation bias — tests encode the requirement, not the implementation.

### 4. Implement

```
/implement
```

The skill reads the beads and implements them:
1. Claims each bead before starting work on it
2. If a test harness exists, makes pre-written tests pass (red→green) without modifying test assertions
3. Makes changes layer by layer (data model → business logic → API → UI)
4. Runs tests after each layer to catch regressions early
5. Closes beads as each step is verified

It follows the bead specs exactly. If something is ambiguous, it stops and asks rather than guessing.

### 5. Validate

```
/validate
```

The skill acts as an independent reviewer:
1. Reads the beads and checks the git diff against them
2. Runs the full test suite
3. Starts the local server and tests end-to-end behavior
4. Creates new beads for any gaps found
5. Gives a verdict: **SHIP**, **FIX**, or **SEND BACK**

It never fixes issues itself — it reports what's wrong and recommends next steps.

### 6. Iterate

Validation ends with one of three verdicts:

- **SHIP** — Feature works. Commit.
- **FIX** — Implementation bugs, but the design is sound. The validator auto-generates fix beads with implementation-ready specs. Hand back to implement:
  ```
  /implement
  ```
  Then run `/validate` again.
- **SEND BACK** — The design itself is wrong. Re-implementing won't help. The validator explains what's flawed and why. Take it back to the planner:
  ```
  /plan-feature <paste the design issues from the validation report>
  ```
  The planner produces new beads, then restart from step 3 (`/write-tests`).

Repeat until you get **SHIP**, then commit.

## Supporting skills

These skills complement the core loop:

- **`/bead-review`** — Check bead health between sessions. Find stale beads, orphaned work, dependency issues.
- **`/grill-with-docs`** — Stress-test any plan against your `CONTEXT.md` before committing to it.
- **`/improve-codebase-architecture`** — Find refactoring opportunities. Run periodically to keep the codebase AI-navigable.

## Skill structure

Each skill is a folder containing:

```
skill-name/
├── SKILL.md          # Main skill definition (required)
├── RESOURCE.md       # Bundled resources loaded with the skill (optional)
└── ...
```

`SKILL.md` has YAML frontmatter with `name` and `description`, followed by the skill's instructions. Additional `.md` files in the folder are bundled resources available to the skill at runtime.

## Attribution

The following skills are derived from [Matt Pocock's skills collection](https://github.com/mattpocock/skills):

- **grill-with-docs**
- **improve-codebase-architecture**

All functional credit for these skills goes to Matt Pocock.
