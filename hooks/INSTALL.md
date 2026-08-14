# Installing Engramory

Engramory is a memory **protocol**, not a skill — its discipline must fire on
*every* task, so it loads primarily as **standing rules**, with `SKILL.md` as the
full reference (optionally registered as a skill) and a hook for the hard cap.

## 1. Load the discipline as standing rules (primary)

Paste [`rules-snippet.md`](../rules-snippet.md) into your host's **always-loaded**
rules so the protocol applies on every task (not just when a skill happens to load
by relevance):

- **Claude Code:** `%USERPROFILE%\.claude\CLAUDE.md` (Windows) / `~/.claude/CLAUDE.md`
  (macOS / Linux) for all projects, or the project's `CLAUDE.md`.
- **Codex:** `AGENTS.md` (global `~/.codex/AGENTS.md` or per-project).
- **Cursor / Cline / Windsurf:** `.cursor/rules` / `.clinerules` / `.windsurfrules`.

[`rules-snippet.md`](../rules-snippet.md) is the full two-tier standing discipline —
it tells the agent to read both indexes (`~/.engramory/MEMORY.md` and the project's
`memory/MEMORY.md`) and write to the right tier. Paste the whole thing into your
user-level `CLAUDE.md`. The project store itself is bootstrapped once per project
by the `memory-init` skill (which copies the project template, wires the project
`CLAUDE.md`, and adds `memory/` to `.gitignore`) — see §4 below.

## 2. (Optional) Register the full spec as a skill — Claude Code

The standing rules carry the trigger; [`SKILL.md`](../SKILL.md) is the complete
protocol. On Claude Code you can ALSO register it as an Agent Skill so the full
reference loads on demand — copy or symlink the folder so `SKILL.md` lands at:

- **Windows:** `%USERPROFILE%\.claude\skills\engramory\SKILL.md`
- **macOS / Linux:** `~/.claude/skills/engramory/SKILL.md`
- Project-only: `<project>/.claude/skills/engramory/SKILL.md`

Symlink keeps this repo as the single source of truth, e.g. on Windows
(PowerShell, admin):

```powershell
New-Item -ItemType SymbolicLink `
  -Path "$env:USERPROFILE\.claude\skills\engramory" `
  -Target "<ABSOLUTE_PATH_TO>\engramory"   # the folder that contains SKILL.md
```

Minimal `SKILL.md` frontmatter is just `name` + `description`; both are already
set. Claude reads the `description` to decide when to load the skill.

## 3. The hard-cap hook (deterministic enforcement)

The standing rules' 150/200 behavior is model-followed; the hook is the
*deterministic backstop* the model cannot skip. It's written and tested for Claude
Code only. Some other hosts expose a pre-write deny you can adapt the shim to (Hermes —
whose `pre_tool_call` has a reported non-firing bug in some worker contexts, #25204;
Cursor, though its is newer/flaky); OpenClaw blocks only via a `before_tool_call` plugin,
and some hosts have none — see PORTING.md (coverage varies by host and version — you verify it).

1. Open your settings file (`%USERPROFILE%\.claude\settings.json` for all
   projects, or `.claude/settings.json` for one project).
2. Merge in the `hooks` block from [`settings.snippet.json`](settings.snippet.json).
   It uses the **exec form** (`"command"` = the interpreter, `"args"` = its argument
   list): Claude Code spawns it directly with **no shell**, so the script path in `args`
   is passed literally — no backslash-escaping and no quoting, even under `Program Files`
   or on Windows.
3. Put the absolute path to `engramory_index_guard.py` in `args`.
4. Set `"command"` to the Python that should run it:
   - **macOS / Linux:** `python3` (a bare `python` frequently does not exist here — the
     old shell-form snippet failed silently on such systems).
   - **Windows:** `python`.
   - **Most robust on any OS:** the absolute interpreter path from
     `python -c "import sys; print(sys.executable)"` (e.g. `C:\Python312\python.exe`),
     which sidesteps the `python`-vs-`python3` question entirely.

The hook fires on every `Edit` / `Write` / `MultiEdit`, returns instantly for any
file that isn't the index, and only acts when the target's filename is the index
(default `MEMORY.md`).

### Hook configuration (environment variables, all optional)

| var | default | meaning |
|---|---|---|
| `ENGRAMORY_HARD` | `200` | hard line ceiling — edits that grow past it are denied |
| `ENGRAMORY_WARN` | `150` | soft line warning — model is nudged to compact |
| `ENGRAMORY_HARD_BYTES` | `25600` | hard byte ceiling (25 KB) — growth past it is denied |
| `ENGRAMORY_WARN_BYTES` | `20480` | soft byte warning (20 KB) |
| `ENGRAMORY_INDEX_NAME` | `MEMORY.md` | which filename counts as the index |
| `ENGRAMORY_INDEX_PATH` | — | absolute path of the one index to guard (use when several `MEMORY.md` files exist; overrides name matching) |
| `ENGRAMORY_INDEX_IGNORE` | — | comma-separated paths or bare basenames to **exempt** from the guard. A full path matches **only by resolved identity** (normcase + realpath) — exempting `.../templates/MEMORY.md` does NOT exempt the real `memory/MEMORY.md` index that shares the basename. A bare basename (e.g. `MEMORY.md`) exempts any file with that name in any directory, **including both tiers' indexes** — use sparingly. |

> ⚠️ **If you keep a file named `MEMORY.md` that is NOT an index** (project docs, this
> repo's own `templates/MEMORY.md`, …), the filename default would gate it too. Two
> opt-outs, pick by which is rarer:
>
> - **Exactly one real index, several non-index `MEMORY.md`s** → set
>   `ENGRAMORY_INDEX_PATH` to the real index's absolute path; the hook then gates
>   only that file.
> - **Both tiers' indexes plus one stray `MEMORY.md`** → keep the default (it guards
>   both tiers — including the project's `memory/MEMORY.md` and the global
>   `~/.engramory/MEMORY.md`) and add the stray file to `ENGRAMORY_INDEX_IGNORE` as a
>   **full path** (e.g. `.../templates/MEMORY.md`) so only it is exempted.
>
> Otherwise a legitimate edit that grows the unrelated `MEMORY.md` past the cap is
> denied (the deny message also reminds you of these opt-outs).

Requires **Python 3.9+** (for the hook and the `tools/` scripts; `python3` on most
systems).

## 4. Point `<MEMORY_ROOT>` at your memory directory

Tell the agent where memory lives (or reuse the host's native memory directory).
If it's inside a git repo, confirm it is `.gitignore`d — memories often hold
machine-local detail (server IPs, ssh paths, serial numbers). Never write a
secret's *value* into memory at all (keys, tokens, passwords) — see SKILL.md §5.

A common per-project layout uses a `memory/` directory at the project root, with a
project-level `CLAUDE.md` binding it. To bootstrap it on demand, a `memory-init`
scaffold (a small skill or script installed to the host's skills dir) creates
`memory/MEMORY.md` from the project template, wires the project `CLAUDE.md`, and
adds `memory/` to `.gitignore`.

With the optional global store, memory has **two tiers**: the project store above
plus a host-agnostic global store (`~/.engramory/`, created via
`python tools/engramory_init.py home`) for cross-project / cross-agent memory.
The hook matches the index by filename (`MEMORY.md`), so it guards **both** tiers'
indexes automatically — the project's `memory/MEMORY.md` AND the global
`~/.engramory/MEMORY.md` — and they must both stay under the caps. If some *other*
`MEMORY.md` that is NOT an index exists, keep the default and exempt it with a full
path in `ENGRAMORY_INDEX_IGNORE`, or (when exactly one real index exists) switch to
`ENGRAMORY_INDEX_PATH`.

## 5. Other agents (Cursor, Cline, Codex, OpenClaw, Windsurf, …)

Step 1 already covers them: paste [`rules-snippet.md`](../rules-snippet.md) (or the
body of [`SKILL.md`](../SKILL.md)) into the agent's always-loaded rules. The
150/200 guard then applies via the instructions; for the deterministic cap, adapt
the hook to the host's pre-write deny hook or run `tools/engramory_check.py` after
each index write. Full per-host wiring is in [PORTING.md](../PORTING.md).

For **Codex** and **OpenClaw**, prefer the init helper instead of manually pasting:

```sh
python tools/engramory_init.py codex    --project-root <repo> --install-skill
python tools/engramory_init.py openclaw                       --install-skill   # -> ~/.openclaw/workspace
```

Each creates the memory template, adds a marked block to `AGENTS.md`, optionally
copies the skill into `.agents/skills/engramory`, and keeps the Engramory store
separate from the host's own memory. See
[adapters/codex/README.md](../adapters/codex/README.md) and
[adapters/openclaw/README.md](../adapters/openclaw/README.md).

To let a host **recall** (read-only) from a store another agent owns and writes — e.g.
Claude Code's native memory — use a `<host>-reader` host (it creates no store and never
writes; `--memory-root` must be an existing store; it lands in that host's own rules file):

```sh
python tools/engramory_init.py codex-reader --project-root ~/.codex \
  --memory-root ~/.claude/projects/<project>/memory
```

Reader hosts: `codex-reader` (dogfooded) plus `claude-reader`, `cursor-reader`, `kiro-reader`,
`cline-reader`, `windsurf-reader`, `openclaw-reader`, `hermes-reader` (wired from documented
formats, printed with an "unverified" note). See
[adapters/reader/README.md](../adapters/reader/README.md).
