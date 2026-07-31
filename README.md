# readable-ad-hoc-commands

Instructions that make the ad-hoc *commands* an AI coding agent runs *legible* —
not just shell, but inline interpreter snippets (`python -c`, `node -e`), `jq`
programs, and SQL one-liners too. Instead of a dense command you have to
reverse-engineer — or approve blind — the agent formats multi-step commands so
you can grasp their intent, steps, and blast radius **before** they execute.

**Harness-neutral.** The rules live in [`AGENTS.md`](AGENTS.md) and work with any
agent that reads a project instructions file or system prompt. A ready-to-drop
[Claude Code](https://claude.com/claude-code) skill is bundled in
[`readable-ad-hoc-commands/SKILL.md`](readable-ad-hoc-commands/SKILL.md). See
[Use with any agent](#use-with-any-agent).

## Why

Agent-authored commands are usually optimized for the machine: one dense line,
cryptic flags, no explanation. This skill flips that priority toward the human
reader:

- One step per line, top to bottom
- Comments explain the **why**, not the obvious
- Magic values pulled into named variables/bindings
- Progress banners for multi-phase runs
- Fail-fast idiom so it stops on error instead of barreling on
- **Destructive actions are previewed** (dry run / listing / transaction) before they run
- A plain-English summary precedes the command
- Applies across command languages — shell, PowerShell, `python -c`, `node -e`, `jq`, SQL — not just bash

It also knows when *not* to bother: a bare `ls` or `git status` runs as-is —
the full treatment is reserved for genuinely complex or destructive commands.

## Use with any agent

The rules are plain Markdown in [`AGENTS.md`](AGENTS.md). Point your agent at
that file however it consumes instructions:

| Agent / tool | Where the guidance goes |
|---|---|
| **Any agent following the [AGENTS.md](https://agents.md) convention** (Codex, Amp, Zed, Jules, …) | Copy `AGENTS.md` to your repo root, or `cat` its contents into your existing one. |
| **Claude Code** | Use the bundled skill — see below. Or add the rules to `CLAUDE.md`. |
| **Cursor** | Paste the rules into `.cursor/rules/readable-commands.mdc` (or legacy `.cursorrules`). |
| **Windsurf** | Add to `.windsurf/rules/` or the global rules. |
| **GitHub Copilot** | Add to `.github/copilot-instructions.md`. |
| **Generic / API-driven agent** | Prepend `AGENTS.md`'s body to the system prompt. |

The guidance is intentionally short and self-contained so it drops cleanly into
any of these without editing.

### Claude Code skill

```bash
git clone https://github.com/a-b/readable-ad-hoc-commands.git
mkdir -p ~/.claude/skills                                   # user-level: all projects
cp -r readable-ad-hoc-commands/readable-ad-hoc-commands ~/.claude/skills/
```

Claude Code auto-discovers the skill on startup — no config needed. Its
*description* is loaded every session; its body loads on demand when a task
calls for a non-trivial command. For a single project, copy into
`.claude/skills/` instead.

## Examples

Each example pairs the dense command an agent would normally emit with what the
skill turns it into, and calls out the rules it applies.

### 1. Shell — log housekeeping (naming, banners, previewing deletion)

**Before:**

```bash
find . -name "*.log" -mtime +7 -exec gzip {} \; && find . -name "*.gz" -mtime +30 -delete && du -sh .
```

**After:**

```bash
#!/usr/bin/env bash
set -euo pipefail

# Housekeeping for log files in the current tree.
root="."
compress_after_days=7      # gzip logs older than this
delete_after_days=30       # remove archives older than this

echo "==> Compressing .log files older than ${compress_after_days}d"
find "$root" -name "*.log" -mtime "+${compress_after_days}" -exec gzip {} \;

echo "==> Deleting .gz archives older than ${delete_after_days}d"
find "$root" -name "*.gz" -mtime "+${delete_after_days}" -delete

echo "==> Disk usage after cleanup"
du -sh "$root"
```

> Prose first: *"Gzips `.log` files older than 7 days, deletes `.gz` archives
> older than 30 days, then prints the tree's total size. It **modifies and
> removes files** under the current directory."*

Rules shown: named variables (#3), progress banners (#4), fail-fast header (#5),
prose summary (#7).

### 2. `python -c` → here-doc — one step per line, prose

A crammed `-c` is unreadable; a here-doc reads far better.

**Before:**

```bash
python -c "import json,sys,glob;print(sum(len(json.load(open(f))['items']) for f in glob.glob('*.json')))"
```

**After** — a here-doc reads far better than a crammed `-c`:

```bash
python - <<'PY'
import glob, json

# Sum the number of items across every JSON report in the cwd. Read-only.
total = 0
for path in glob.glob("*.json"):
    with open(path) as fh:
        total += len(json.load(fh)["items"])

print(f"{total} items across all reports")
PY
```

> Prose first: *"Read-only: counts total `items` across all `*.json` files in
> the current directory and prints the sum."*

Rules shown: one step per line via here-doc (#1), comment the why (#2), prose
summary (#7).

### 3. SQL — preview before a destructive DELETE

**Before:**

```bash
psql -c "DELETE FROM sessions WHERE last_seen < now() - interval '90 days'"
```

**After** — count first, delete inside a transaction:

```bash
psql <<'SQL'
\set ON_ERROR_STOP on

-- Preview: how many rows WOULD be deleted (nothing changed yet).
SELECT count(*) AS to_delete
FROM sessions
WHERE last_seen < now() - interval '90 days';

BEGIN;
  DELETE FROM sessions
  WHERE last_seen < now() - interval '90 days';
  -- Review the DELETE row count above, then COMMIT (or ROLLBACK to abort).
COMMIT;
SQL
```

> Prose first: *"Deletes `sessions` rows not seen in 90+ days. It first prints
> how many rows match, then deletes them inside a transaction. This **removes
> database rows**."*

Rules shown: preview destructive actions (#6), fail-fast via
`\set ON_ERROR_STOP on` (#5), prose summary (#7).

### 4. When *not* to format

A single, obviously-safe command needs none of this — the skill runs it as-is:

```bash
git status
```

No header, no banners, no ceremony. The full treatment is reserved for
multi-step, quoting-heavy, or destructive commands.

## Layout

```
AGENTS.md                       # harness-neutral rules (portable source of truth)
readable-ad-hoc-commands/
└── SKILL.md                    # Claude Code skill adapter (same rules + frontmatter)
```

`AGENTS.md` and `SKILL.md` carry the same guidance; the skill just adds the
frontmatter Claude Code needs to auto-discover and trigger it.

## License

[MIT](LICENSE)
