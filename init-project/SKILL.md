---
name: init-project
description: One-time project onboarding that explores the codebase, generates CONTEXT.md, sets up beads tracking, reads existing specs/docs, creates a gap analysis, and prepares the plan/implement/validate workflow. Use when joining a new project, starting work on an unfamiliar repo, or bootstrapping the AI-assisted workflow for the first time.
allowed-tools: Bash(bd create *) Bash(bd update *) Bash(bd close *) Bash(bd dep *) Bash(bd show *) Bash(bd list *) Bash(bd ready *) Bash(bd graph *) Bash(bd remember *) Bash(bd recall *) Bash(bd prime *) Bash(bd doctor *) Bash(bd stale *) Bash(bd orphans *) Bash(bd blocked *) Bash(bd find-duplicates *) Bash(bd duplicate *) Bash(bd upgrade *) Bash(bd status *) Bash(bd init *)
---

# Init Project

One-time project onboarding. Explores everything, documents what exists, identifies what's missing, and sets up the plan → implement → validate pipeline.

## Hard rules

1. **Do NOT make code changes.** This is exploration and documentation only.
2. **Ask before creating files.** Show what you'd write, get approval, then write.
3. **Max 2 exploration attempts** per question. If structure is unclear, ask.
4. **Use domain expert language.** If the user corrects a term, adopt it immediately.

## Process

### 1. Explore the codebase

Use the Explore agent for broad discovery:

- Project structure (languages, frameworks, package managers, entry points)
- Build/test/run commands (Makefile, package.json scripts, pyproject.toml)
- Existing documentation (README, docs/, specs)
- Configuration patterns (env vars, YAML configs, settings files)
- Test structure (framework, location, how to run, current count)
- CI/CD pipeline (if visible)

Record findings — don't summarize yet.

### 2. Generate CONTEXT.md

If `CONTEXT.md` doesn't exist, create it using [CONTEXT-FORMAT.md](../grill-with-docs/CONTEXT-FORMAT.md):

- Extract domain terms from code (class names, module names, config keys)
- Identify the bounded context(s)
- Map relationships between domain concepts
- Document the architecture (project structure, request flow, key modules)
- Note infrastructure (runtime, package manager, caching, deployment)

If `CONTEXT.md` already exists, read it and verify it matches the current codebase. Flag stale entries.

**Present the draft to the user before writing.** They know the domain — you're guessing from code.

### 3. Set up beads

First, ensure beads is up to date:
- Run `bd upgrade status` to check if a newer version was installed
- If upgraded, run `bd upgrade review` to see what changed, then `bd upgrade ack`

Run `bd doctor` to check beads health. If beads isn't initialized, ask the user:

**Database location:**
- **Local** (default) → `bd init`
- **Remote server** (when a tunnel exists to the actual database server) →
  ```bash
  bd init --server --external --database remote_beads_project --server-host 127.0.0.1 --server-port 13306
  ```
  Ask the user for the database name, host, and port if they differ from defaults.

After init, verify `bd ready`, `bd list`, `bd create` work.

If beads already exists:
- Run `bd upgrade status` to check for version changes
- Run `bd status` for project health overview
- Run `bd ready` to see available work
- Run `bd stale` and `bd orphans` for hygiene issues

### 4. Read existing specs and docs

Look for:
- Product specs, PRDs, requirement docs (docs/, specs/, any markdown)
- API documentation (OpenAPI, Swagger)
- Architecture docs, diagrams

Summarize what exists and what each document covers.

### 5. Gap analysis

Compare the codebase against any specs/docs found:

| Area | Spec says | Code has | Status |
|---|---|---|---|
| Feature X | Required | Implemented | DONE |
| Feature Y | Required | Partial | GAP |
| Feature Z | Required | Missing | GAP |

For each GAP, create a bead with:
- `--title` — what needs to be done
- `--description` — what the spec requires vs what exists
- `--type` — feature/task/bug
- `--priority` — based on spec priority/timeline

Set up dependencies between beads.

### 6. Verify workflow tooling

Check that the plan → implement → validate pipeline is ready:

- [ ] `CONTEXT.md` exists and is current
- [ ] Beads tracking is healthy (`bd doctor`)
- [ ] Build command works (identify and record it)
- [ ] Test command works (identify, record, note current pass count)
- [ ] Local run command works (identify and record it)
- [ ] Skills available: `/plan-feature`, `/implement`, `/validate`, `/bead-review`

Record build/test/run commands for future reference.

### 7. Present onboarding report

```
## Project Onboarding Report

### Project Overview
- Language/framework: ...
- Entry point: ...
- Package manager: ...

### Commands
- Build: `...`
- Test: `... (X tests passing)`
- Run locally: `...`
- Lint: `...`

### Documentation Found
- CONTEXT.md: created/updated/verified
- Specs: <list>

### Beads Status
- Open: X
- In progress: X
- Ready (no blockers): X

### Gap Analysis
- X gaps found, Y beads created
- Critical gaps: <list>

### Workflow Ready
- [x/] CONTEXT.md
- [x/] Beads
- [x/] Test suite
- [x/] Local run

### Recommended First Steps
1. ...
2. ...
3. ...
```

### 8. Configure project integration

Ask the user about their setup preferences:

**Skills location:**
- Do you have a shared skills repo? (e.g. `~/Documents/skills/`)
  - Yes → symlink `plan`, `implement`, `validate`, `init-project`, `grill-with-docs`, `improve-codebase-architecture` into `.claude/skills/`
  - No → copy skills into `.claude/skills/` directly
- Should `.claude/skills/` be git-ignored? (Yes if symlinked, maybe not if copied)

**Beads persistence:**
- Local only (default `bd init`) or remote server?
  - If remote server (tunnel exists):
    ```bash
    bd init --server --external --database <db_name> --server-host 127.0.0.1 --server-port 13306
    ```
    Ask for database name, host, and port.
  - If team uses shared beads via Dolt remote: `bd clone <url>` instead of `bd init`
- Should `.beads/` / `.dolt/` be git-ignored? (Usually yes — beads has its own versioning)

**What to add to .gitignore:**
Present a checklist and let the user confirm before writing:
```
[ ] .claude/skills/     (symlinked from shared repo)
[ ] .dolt/              (beads database)
[ ] .beads-credential-key
[ ] *.db
```

Only add entries that don't already exist in `.gitignore`.

### 9. Save onboarding context

Store key findings using `bd remember`:
- Build/test/run commands
- Key architectural patterns
- Important conventions discovered
- Spec locations
- Skills repo location (if shared)
- Beads remote (if configured)

This ensures future sessions (even after `/clear`) can recover context with `bd prime`.
