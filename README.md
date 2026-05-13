# Claude Code Skills

A collection of custom skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that implement a structured **plan → implement → validate** development workflow.

## Skills included

| Skill | What it does |
|-------|-------------|
| **init-project** | One-time project onboarding. Explores the codebase, generates `CONTEXT.md`, sets up beads tracking, and prepares the workflow. |
| **plan-feature** | Converts business requirements into implementation briefs and beads. Grills you on design decisions. Never writes code. |
| **implement** | Executes a brief produced by `/plan-feature`. Claims beads, implements layer by layer, runs tests between steps. |
| **validate** | Reviews completed implementation against the brief, beads, and `CONTEXT.md`. Reports issues — never fixes them directly. |
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

The core loop is **init → plan → implement → validate**. Here's what that looks like on a real project.

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
4. Output a **brief** in `docs/briefs/` with implementation steps
5. Create **beads** (trackable work items) for each step

The brief is a complete spec — a coding agent can follow it without asking questions.

### 3. Implement

```
/implement docs/briefs/email-verification.md
```

The skill reads the brief and implements it:
1. Claims each bead before starting work on it
2. Makes changes layer by layer (data model → business logic → API → UI)
3. Runs tests after each layer to catch regressions early
4. Closes beads as each step is verified

It follows the brief exactly. If something is ambiguous, it stops and asks rather than guessing.

### 4. Validate

```
/validate
```

The skill acts as an independent reviewer:
1. Reads the brief and checks the git diff against it
2. Runs the full test suite
3. Starts the local server and tests end-to-end behavior
4. Creates new beads for any gaps found
5. Gives a verdict: **SHIP**, **FIX**, or **SEND BACK**

It never fixes issues itself — it reports what's wrong and recommends next steps.

### 5. Iterate

Validation ends with one of three verdicts:

- **SHIP** — Feature works. Commit.
- **FIX** — Implementation bugs, but the design is sound. The validator auto-generates a fix brief at `docs/briefs/<feature>-fix.md`. Hand it back to implement:
  ```
  /implement docs/briefs/email-verification-fix.md
  ```
  Then run `/validate` again.
- **SEND BACK** — The design itself is wrong. Re-implementing won't help. The validator explains what's flawed and why. Take it back to the planner:
  ```
  /plan-feature <paste the design issues from the validation report>
  ```
  The planner produces a new brief, then restart from step 3.

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
