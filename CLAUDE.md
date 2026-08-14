<!-- BEGIN ENGRAMORY MEMORY -->
## Memory (Engramory — project tier)

This project's memory store is at `memory/` (index: `memory/MEMORY.md`).

- At the start of a task, read `memory/MEMORY.md` (project index) and
  `~/.engramory/MEMORY.md` (global index, loaded from the user-level CLAUDE.md).
- Save project-specific facts here — `memory/` — one file = one fact, add a
  pointer line to `memory/MEMORY.md`. Never create `user` notes here (they go
  in the global store only).
- This store is git-ignored (see `.gitignore`).
<!-- END ENGRAMORY MEMORY -->
