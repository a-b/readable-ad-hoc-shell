# readable-ad-hoc-shell

Instructions that make the ad-hoc shell an AI coding agent runs *legible*.
Instead of a dense one-liner you have to reverse-engineer — or approve blind —
the agent formats multi-step commands so you can grasp their intent, steps, and
blast radius **before** they execute.

**Harness-neutral.** The rules live in [`AGENTS.md`](AGENTS.md) and work with any
agent that reads a project instructions file or system prompt. A ready-to-drop
[Claude Code](https://claude.com/claude-code) skill is bundled in
[`readable-ad-hoc-shell/SKILL.md`](readable-ad-hoc-shell/SKILL.md). See
[Use with any agent](#use-with-any-agent).

## Why

Agent-authored shell is usually optimized for the machine: one line, cryptic
flags, no explanation. This skill flips that priority toward the human reader:

- One step per line, top to bottom
- Comments explain the **why**, not the obvious
- Magic values pulled into named variables
- Progress banners (`echo "==> ..."`) for multi-phase scripts
- `set -euo pipefail` so scripts fail loud and early
- **Destructive actions are previewed** (dry run / listing) before they run
- A plain-English summary precedes the command

It also knows when *not* to bother: a bare `ls` or `git status` runs as-is —
the full treatment is reserved for genuinely complex or destructive scripts.

## Use with any agent

The rules are plain Markdown in [`AGENTS.md`](AGENTS.md). Point your agent at
that file however it consumes instructions:

| Agent / tool | Where the guidance goes |
|---|---|
| **Any agent following the [AGENTS.md](https://agents.md) convention** (Codex, Amp, Zed, Jules, …) | Copy `AGENTS.md` to your repo root, or `cat` its contents into your existing one. |
| **Claude Code** | Use the bundled skill — see below. Or add the rules to `CLAUDE.md`. |
| **Cursor** | Paste the rules into `.cursor/rules/readable-shell.mdc` (or legacy `.cursorrules`). |
| **Windsurf** | Add to `.windsurf/rules/` or the global rules. |
| **GitHub Copilot** | Add to `.github/copilot-instructions.md`. |
| **Generic / API-driven agent** | Prepend `AGENTS.md`'s body to the system prompt. |

The guidance is intentionally short and self-contained so it drops cleanly into
any of these without editing.

### Claude Code skill

```bash
git clone https://github.com/a-b/readable-ad-hoc-shell.git
mkdir -p ~/.claude/skills                                   # user-level: all projects
cp -r readable-ad-hoc-shell/readable-ad-hoc-shell ~/.claude/skills/
```

Claude Code auto-discovers the skill on startup — no config needed. Its
*description* is loaded every session; its body loads on demand when a task
calls for non-trivial shell. For a single project, copy into `.claude/skills/`
instead.

## Examples

Each example pairs the dense command an agent would normally emit with what the
skill turns it into, and calls out the rules it applies.

### 1. Log housekeeping — naming, banners, previewing deletion

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

> Prose first: *"This gzips logs older than 7 days, deletes archives older than
> 30 days, then prints total size. It modifies and removes files under the
> current directory."*

Rules shown: named variables (#3), progress banners (#4), `set -euo pipefail`
(#5), prose summary (#7).

### 2. Bulk rename — preview before touching anything

The riskiest kind of one-liner: it renames files in place with no way to see
what it will do first.

**Before:**

```bash
for f in *.jpeg; do mv "$f" "${f%.jpeg}.jpg"; done
```

**After:**

```bash
#!/usr/bin/env bash
set -euo pipefail

# Normalize .jpeg extensions to .jpg in the current directory.

# Dry run first — show every rename WITHOUT doing it.
echo "==> Planned renames:"
for f in *.jpeg; do
  echo "    $f -> ${f%.jpeg}.jpg"
done

# Then perform them.
echo "==> Renaming"
for f in *.jpeg; do
  mv -v -- "$f" "${f%.jpeg}.jpg"
done
```

> Prose first: *"This renames every `.jpeg` in the current directory to `.jpg`.
> It first prints the full list of planned renames so you can check them, then
> performs them."*

Rules shown: preview destructive actions (#6), one step per line (#1),
comment the why (#2).

### 3. Cluster snapshot — replace a clever pipeline with a readable loop

A nested `awk`/`xargs` incantation reads better as a short, labelled loop.

**Before:**

```bash
kubectl get pods -A --no-headers | awk '$4>5{print $1,$2,$4}' | xargs -n3 sh -c 'echo "ns=$0 pod=$1 restarts=$2"'
```

**After:**

```bash
#!/usr/bin/env bash
set -euo pipefail

restart_threshold=5   # flag pods that have restarted more than this

echo "==> Pods with more than ${restart_threshold} restarts"
# Columns: NAMESPACE NAME READY STATUS RESTARTS ...
kubectl get pods -A --no-headers | while read -r ns name _ _ restarts _; do
  if (( restarts > restart_threshold )); then
    echo "    ns=${ns} pod=${name} restarts=${restarts}"
  fi
done
```

> Prose first: *"Read-only: lists every pod across all namespaces that has
> restarted more than 5 times. Changes nothing."*

Rules shown: clarity over cleverness (#3 anti-pattern), named threshold (#3),
comment the why (#2), read-only blast-radius note (#7).

### 4. When *not* to format

A single, obviously-safe command needs none of this — the skill runs it as-is:

```bash
git status
```

No header, no banners, no ceremony. The full treatment is reserved for
multi-step, quoting-heavy, or destructive scripts.

## Layout

```
AGENTS.md                    # harness-neutral rules (the portable source of truth)
readable-ad-hoc-shell/
└── SKILL.md                 # Claude Code skill adapter (frontmatter + same rules)
```

`AGENTS.md` and `SKILL.md` carry the same guidance; the skill just adds the
frontmatter Claude Code needs to auto-discover and trigger it.

## License

[MIT](LICENSE)
