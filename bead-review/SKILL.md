---
name: bead-review
description: Review, update, and manage beads without planning or implementing. Standalone bead housekeeping — status review, dependency management, gap analysis, stale/orphan cleanup, and spec-vs-codebase comparison. Use when user wants to check bead status, update beads, manage dependencies, find stale or orphan beads, or do bead housekeeping. NEVER plans features or writes code.
allowed-tools: Bash(bd *)
---

# Bead Review

Standalone bead management. Review status, update beads, manage dependencies, find gaps. This skill NEVER plans features and NEVER writes code.

## Hard rules

1. **Do NOT plan features.** If the user needs a new feature planned, hand off to `/plan-feature`.
2. **Do NOT write code.** If beads need implementation, hand off to `/implement`.
3. **Do NOT create briefs.** Briefs are `/plan-feature`'s output.
4. **Max 2 exploration attempts** per question. If codebase exploration doesn't answer it, ask the user.
5. **All bead operations use `bd` CLI.** No TodoWrite, TaskCreate, or markdown TODOs.

## Process

### 1. Load context

Read `CONTEXT.md` for domain vocabulary and recorded decisions.
Run `bd prime` if available to recover prior session context.

### 2. Bead health check

Run the checks from [HEALTH-CHECKS.md](HEALTH-CHECKS.md):

- `bd doctor` — installation health
- `bd list` — overview of all beads by status
- `bd ready` — beads with no active blockers
- `bd stale` — beads not updated recently
- `bd orphans` — referenced in commits but still open
- `bd list --status=in_progress` — claimed but not closed
- `bd blocked` — blocked by unresolved dependencies

Present findings as a structured report (see step 6).

### 3. Spec-vs-codebase comparison

For beads that reference specific files or behaviors:

- Check whether referenced files still exist at expected paths
- Check whether the described behavior already exists in the codebase (the bead may be stale / already done)
- Check whether the bead's description matches what `CONTEXT.md` says

Flag:
- **Already done:** bead describes work that exists in code but bead is still open
- **Stale reference:** bead references files/lines that have moved or been deleted
- **Vocabulary drift:** bead uses terms that conflict with `CONTEXT.md`

### 4. Dependency audit

Run `bd graph` to visualize the dependency tree.

Check for:
- Circular dependencies
- Orphan beads (no parents, no children, no epic)
- Dependency chains longer than 3 levels (potential over-planning)
- Beads blocked by closed beads (stale dependency)

### 5. Act on findings

For each finding, take the appropriate action:

| Finding | Action |
|---|---|
| Already done | `bd close <id>` with note explaining why |
| Stale reference | `bd update <id>` with corrected file paths/lines |
| Vocabulary drift | `bd update <id>` with corrected terminology |
| Stale dependency | `bd dep remove` the resolved dependency |
| Circular dep | Flag to user — requires manual resolution |
| Long-unclaimed | Ask user: still relevant? Close or re-prioritize |
| Discovered gap | `bd create` new bead (see below) |

For discovered gaps, create beads with:
- `--title` — imperative, specific
- `--description` — WHY the gap exists, WHAT needs to be done, which files are involved
- `--type` — task/bug/feature
- `--priority` — 0-4

Set up dependencies with `bd dep add`.

### 6. Report

Present a structured report:

```
## Bead Review Report

### Health
- Total: X open, Y in-progress, Z closed
- Stale: N beads not updated in >7 days
- Orphans: N beads

### Findings
| Bead | Finding | Action Taken |
|---|---|---|
| <id> | Already done | Closed |
| <id> | Stale reference | Updated paths |

### Dependency Graph
<current graph from bd graph>

### New Beads Created
- <id>: <title> (priority N)

### Recommendations
- <actionable next steps>

### Handoffs
- Ready for `/implement`: <bead-ids>
- Needs `/plan-feature` first: <bead-ids>
- Needs `/diagnose` first: <bead-ids with bugs>
```
