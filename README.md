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

## Example: before → after

**Before** — what an agent typically emits:

```bash
find . -name "*.log" -mtime +7 -exec gzip {} \; && find . -name "*.gz" -mtime +30 -delete && du -sh .
```

**After** — what the skill produces:

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

…preceded in chat by a one-line summary: *"This gzips logs older than 7 days,
deletes archives older than 30 days, then prints total size. It modifies and
removes files under the current directory."*

## Layout

```
readable-ad-hoc-shell/
└── SKILL.md      # the skill (frontmatter + instructions)
```

## License

[MIT](LICENSE)
