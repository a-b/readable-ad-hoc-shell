# readable-ad-hoc-shell

A [Claude Code](https://claude.com/claude-code) **skill** that makes the ad-hoc
shell an agent runs *legible*. Instead of a dense one-liner you have to
reverse-engineer — or approve blind — the agent formats multi-step commands so
you can grasp their intent, steps, and blast radius **before** they execute.

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

## Install

Skills live in `~/.claude/skills/` (user-level, applies to every project) or
`.claude/skills/` inside a project.

```bash
# User-level install
git clone https://github.com/a-b/readable-ad-hoc-shell.git
mkdir -p ~/.claude/skills
cp -r readable-ad-hoc-shell/readable-ad-hoc-shell ~/.claude/skills/
```

Claude Code auto-discovers the skill on startup — no config needed. The skill's
*description* is loaded every session; its body loads on demand when a task
calls for non-trivial shell.

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
readable-ad-hoc-shell/
└── SKILL.md      # the skill (frontmatter + instructions)
```

## License

[MIT](LICENSE)
