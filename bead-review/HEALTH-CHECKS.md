# Bead Health Checks

Reference for bead hygiene. Each check, what it finds, and what to do.

## Checks

### Installation health: `bd doctor`

What: Verifies beads database is healthy, schema is current.
When unhealthy: Run `bd migrate` or `bd init` as recommended.

### Overview: `bd list`

What: All beads grouped by status.
Use for: Quick pulse check on project state.

### Ready work: `bd ready`

What: Open beads with no active blockers.
Use for: Identifying what can be worked on next via `/implement`.

### Stale beads: `bd stale`

What: Beads not updated in a configurable window.
Action: Review each — close if done, update if still relevant, ask user if unclear.

### Orphan beads: `bd orphans`

What: Referenced in git commits but still open.
Action: Check if the referenced work was completed. If yes, close. If partial, update the bead description with what remains.

### Blocked beads: `bd blocked`

What: Beads waiting on unresolved dependencies.
Action: Check the blocking bead. If it's closed, remove the stale dependency with `bd dep remove`. If it's open, verify it's still the right dependency.

### In-progress beads: `bd list --status=in_progress`

What: Beads someone claimed but hasn't closed.
Action: Check if work was completed (git log, file changes). If done, close. If abandoned, unclaim.

### Dependency graph: `bd graph`

What: Visual dependency tree.
Action: Look for cycles, excessive depth (>3 levels), orphans.

### Duplicates: `bd find-duplicates`

What: Semantically similar beads.
Action: Merge with `bd duplicate <source> <target>` or clarify the distinction in bead descriptions.
