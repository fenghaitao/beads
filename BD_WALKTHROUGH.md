# bd (beads) Walkthrough — Dogfooding for the beads Project

This document records a real, end-to-end workflow using `bd` to track issues
for the beads project itself. Every command shown was executed and its output
captured.

## Prerequisites

- Go 1.26.2 (matching `go.mod`)
- `bd` binary installed at `~/.local/bin/bd`

Build and install:
```bash
make install
```

---

## Step 1: Initialize bd in the Project

```bash
bd init
```

This creates:
- `.beads/dolt/` — the Dolt-backed version-controlled database
- `.beads/hooks/` — lifecycle hooks (pre-commit, pre-push, etc.)
- Agent integration files (CLAUDE.md, AGENTS.md, skill files)

Output confirms the configuration:
```
Backend: dolt
Mode: embedded
Database: beads
Issue prefix: beads
Issues will be named: beads-<hash> (e.g., beads-a3f2dd)
```

---

## Step 2: Create Issues of Different Types

### Bug (P1 — High priority)
```bash
bd create "Memory leak in bd graph --watch mode" \
  -t bug -p 1 \
  --description="When running 'bd graph --watch' for extended periods (>1 hour), \
resident memory grows linearly without bound. Profiling suggests goroutine leaks \
in the graph refresh loop." --json
```
Created: **beads-lwf** (status: open, priority: 1)

### Feature (P2 — Medium priority)
```bash
bd create "Add JSON Schema validation for issue metadata" \
  -t feature -p 2 \
  --description="Allow users to define a JSON Schema for the metadata field. \
On create/update, validate metadata against the schema and reject invalid \
payloads with clear error messages." --json
```
Created: **beads-770** (status: open, priority: 2)

### Task (P2 — Medium priority)
```bash
bd create "Add test coverage for dependency cycle detection" \
  -t task -p 2 \
  --description="The cycle detection in internal/storage/domain/dependency.go \
has limited test coverage. Add table-driven tests covering: self-referencing \
deps, 2-node cycles, 3-node cycles, and diamond patterns." --json
```
Created: **beads-6hn** (status: open, priority: 2)

### Epic (P1 — High priority)
```bash
bd create "Multi-repo workspace support" \
  -t epic -p 1 \
  --description="Support tracking issues across multiple repositories in a \
single beads workspace. Includes: repo-aware issue IDs, cross-repo dependency \
graphs, per-repo filters, and federation push/pull." --json
```
Created: **beads-soe** (status: open, priority: 1)

### Child Issue (P2, under the epic)
```bash
bd create "Cross-repo dependency graph visualization" \
  -t feature -p 2 \
  --description="Extend bd graph to color-code nodes by source repo and draw \
repo boundaries." \
  --deps parent-child:beads-soe --json
```
Created: **beads-715** (status: open, priority: 2, parent: beads-soe)

---

## Step 3: Wire Up Dependencies

### Epic depends on bug fix
```bash
bd dep add beads-soe beads-lwf -t depends-on --json
```
The epic "Multi-repo workspace support" depends on the memory leak bug being
fixed first.

### Feature related to task
```bash
bd dep add beads-770 beads-6hn -t related --json
```
JSON Schema validation work is related to test coverage improvements.

### Resulting dependency graph
```
beads-soe (epic) ──depends-on──▶ beads-lwf (bug, in_progress)
beads-soe (epic) ──parent──▶ beads-715 (feature)
beads-770 (feature) ──related──▶ beads-6hn (task, closed)
```

---

## Step 4: Query and Filter

### List all issues
```bash
bd list --json
```

### Filter by type
```bash
bd list -t bug --json
```

### Show ready (unblocked) work
```bash
bd ready --json
```

### Visual dependency graph
```bash
bd graph beads-soe
```

Shows a layered graph with color-coded statuses:
- `○` open
- `◐` in_progress
- `●` blocked
- `✓` closed
- `❄` deferred

---

## Step 5: Update Status and Close Issues

### Claim and start work
```bash
bd update beads-lwf --claim --json
```
Sets status to `in_progress`, records `started_at`, assigns to current user.

### Close completed work
```bash
bd close beads-6hn \
  --reason "Completed: Added table-driven tests for all cycle patterns" --json
```
Sets status to `closed`, records `closed_at` and `close_reason`.

---

## Step 6: Sync to Remote

```bash
bd dolt push
```

Pushes the entire issue database to the configured Dolt remote
(`fenghaitao/beads` on DoltHub). Other collaborators can `bd dolt pull`
to get the latest state.

---

## Final State Summary

| ID | Title | Type | Priority | Status |
|---|---|---|---|---|
| beads-lwf | Memory leak in bd graph --watch mode | bug | P1 | in_progress |
| beads-soe | Multi-repo workspace support | epic | P1 | open |
| beads-715 | Cross-repo dependency graph visualization | feature | P2 | open |
| beads-770 | Add JSON Schema validation for issue metadata | feature | P2 | open |
| beads-6hn | Add test coverage for dependency cycle detection | task | P2 | closed |

**Dashboard**: 1 closed, 1 in_progress, 3 open · 3 ready · 0 blocked

---

## Key Takeaways

1. **Hash-based IDs** (`beads-lwf`, `beads-6hn`) prevent collisions across
   teams — no more "who took issue #42?"
2. **Dependencies are first-class** — `blocks`, `depends-on`, `related`,
   `parent-child`, `discovered-from` all carry semantic meaning for the
   readiness algorithm.
3. **JSON output everywhere** — every command supports `--json` for
   programmatic consumption by AI agents and scripts.
4. **Version-controlled database** — Dolt commits every write automatically,
   so issue history is never lost and syncs like git.
5. **Dogfooding** — the beads project tracks its own issues in beads.
