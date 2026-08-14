---
name: memory-init
description: >-
  Initialize (or verify) a project-tier Engramory memory store for the current
  project. Use this when starting work in a new project, when you are unsure
  whether the current project has its own memory directory, or when the user asks
  to "set up memory", "init project memory", or "create a memory store here". It
  checks whether the project already has a `memory/` directory with a `MEMORY.md`
  index; if absent, it creates the store from the template, wires a project-level
  `CLAUDE.md` that binds the store path and connects the global store
  (`~/.engramory/`), and adds `memory/` to the project's `.gitignore`. It never
  creates a `user` note in the project store (that type lives only in the global
  tier). Safe to run repeatedly — it skips silently if the store already exists.
---

# memory-init — project memory setup

Scaffolds the **project tier** of Engramory memory for the current project,
without touching the user's global store.

## When to run

Run this when any of these is true:

- You just started working in a **new project** and no memory store exists yet.
- You're unsure whether the current project has a project memory directory.
- The user says something like *"set up memory for this project"*,
  *"create a memory store here"*, or *"initialize project memory"*.

Do NOT run it to *recall* or *save* a memory — that is the normal Engramory
discipline (read the two `MEMORY.md` indexes, write to the right tier). This skill
only **bootstraps the store**.

## What it does

1. **Detect the project root.** Use the current working directory (the directory
   the agent was started in — this is the project root for a single-project
   session). If a git repository, use its top-level directory.

2. **Check for an existing store.** Look for a `memory/` directory containing a
   `MEMORY.md` index at the project root:
   - If it exists → **stop, do nothing.** The store is already set up. Report
     `already set up` and return. (Idempotent — safe to run repeatedly.)
   - If it does not exist → continue.

3. **Create the store.** Make `memory/` under the project root, plus the
   three per-type subfolders `memory/feedback/`, `memory/project/`,
   `memory/reference/`. Copy `~/.engramory/templates/MEMORY_project_template.md`
   (the user's home directory) to `memory/MEMORY.md`. (If the template is missing,
   create a minimal three-section index instead: `## feedback`, `## project`,
   `## reference` — never a `user` section here.)

4. **Wire the project-level `CLAUDE.md`.** Ensure the project root has a
   `CLAUDE.md`. If one already exists, **append** an Engramory block (marked
   with `<!-- BEGIN ENGRAMORY MEMORY -->` … `<!-- END ENGRAMORY MEMORY -->`)
   that binds the store. If none exists, create it with just that block. The
   block must say:

   ```markdown
   ## Memory (Engramory — project tier)

   This project's memory store is at `memory/` (index: `memory/MEMORY.md`).

   - At the start of a task, read `memory/MEMORY.md` (project index) and
     `~/.engramory/MEMORY.md` (global index, loaded from the user-level CLAUDE.md).
   - Save project-specific facts here — `memory/` — one file = one fact, add a
     pointer line to `memory/MEMORY.md`. Never create `user` notes here (they go
     in the global store only).
   - This store is git-ignored (see `.gitignore`).
   ```

5. **Update `.gitignore`.** Ensure the project's `.gitignore` contains a line
   ignoring `memory/` (and, if it exists, the store directory). Add the line if
   absent, without touching existing content.

6. **Report.** Report **briefly**, in the user's preferred language (Chinese for
   this user). Just confirm completion in one short sentence — e.g. **"已完成项目
   记忆库搭建"** — with no detail list. If the store already existed, say so in one
   short line (e.g. "项目记忆库已存在,无需搭建").

## Rules

- **Never** create or touch anything in `~/.engramory/` — that is the global
  store, owned by the user.
- **Never** create a `user` note or `## user` section in the project store.
- **Idempotent**: running twice on the same project changes nothing the second
  time.
- If `memory/` exists but has no `MEMORY.md`, treat it as missing and create the
  index.
- Do not overwrite an existing `CLAUDE.md` block — if the markers already exist,
  leave them; just verify the store is present.

## Where detail files go

Each note is a **one-fact file** placed in the per-type subfolder, and its index
pointer line carries the subfolder path:

- `feedback` notes → `memory/feedback/<slug>.md`, pointer `- [title](feedback/<slug>.md) — hook`
- `project` notes → `memory/project/<slug>.md`, pointer `- [title](project/<slug>.md) — hook`
- `reference` notes → `memory/reference/<slug>.md`, pointer `- [title](reference/<slug>.md) — hook`

Never write a detail file loose in `memory/` — always into its type subfolder.
Archive/retired notes go to `memory/archive/`.
