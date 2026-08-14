# Engramory — always-on pointer

Paste this into your host's **always-loaded** rules (Claude Code: `CLAUDE.md` or
`~/.claude/CLAUDE.md`; Codex/OpenClaw: `AGENTS.md`; Cursor: `.cursor/rules`; …) so
the memory discipline applies even on tasks where the engramory skill isn't loaded by
relevance. Keep it short — the full protocol lives in the engramory `SKILL.md`.

---

## Memory (Engramory)

You have a curated, file-based memory in **two tiers**:

- **Global tier** — `~/.engramory/` (index: `MEMORY.md`). Holds the **cross-project**
  four-type memory: who the user is (`user`), cross-project work habits / general
  procedures (`feedback`), cross-project resource pointers (`reference`). It is shared
  by every agent and loaded every session. Skip it if the directory doesn't exist.
- **Project tier** — `memory/` at the project root (index: `memory/MEMORY.md`).
  Holds **this project's** state and decisions (`project`), project-specific habits
  (`feedback`) and pointers (`reference`). It never holds `user` notes — who the user
  is is always cross-project, so that type belongs only in the global tier. Create it
  once per project with the `memory-init` skill; save project memory here, never in
  the host's native auto-memory.

- **At the start of a task**, read **both** indexes: `~/.engramory/MEMORY.md`
  (global) and `memory/MEMORY.md` (this project's, if it exists) — one line per
  memory — and open only the detail files whose hooks look relevant. Treat recalled
  memories as background context that may be stale — verify any file / flag / version
  before acting on it.
- **When you learn something durable** worth a future session: confirm it isn't already
  in the repo / git / `CLAUDE.md` (don't duplicate the source of truth) and isn't a
  secret *value*; **pick the tier** — cross-project / about the user → global, about
  this project → project (when unsure, prefer the project tier). `user` notes live only
  in the global tier; a project store never creates a `user` note. Search that tier's
  index and **update an existing note** rather than duplicate; otherwise write one atomic
  markdown file (one fact) with frontmatter `name` / `description` (a sharp one-line hook)
  / `type` (`user | feedback | project | reference`) / `created` + `updated`
  (`YYYY-MM-DD`). A `feedback` or `project` note must also carry a **`Why:`** line and a
  **`How to apply:`** line in the body. Add one pointer line to that tier's `MEMORY.md`.
  **Delete** memories that turn out wrong.
- **Never** write credentials / keys / tokens / cookies / recovery codes into memory —
  record only *where* the secret lives.
- Keep **each** `MEMORY.md` small. Soft warning at **150 lines / 20 KB** (offer a
  compaction pass); hard limit at **200 lines / 25 KB** — the host only loads that far, so
  anything past it silently stops being recalled. Once past the soft line, compact:
  pointer-ify over-long lines, merge duplicates, archive cold notes.

Full protocol & rationale: the engramory `SKILL.md`.
