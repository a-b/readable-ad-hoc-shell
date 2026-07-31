# readable-ad-hoc-commands

**Stop approving shell commands you can't read.** When an AI coding agent fires
a dense one-liner at your terminal, you either reverse-engineer it or rubber-stamp
it blind. This makes the agent format those commands so you can see their intent,
steps, and **blast radius** *before* they run — and preview anything destructive
first.

Works with any agent, in any command language (shell, `python -c`, `node -e`,
`jq`, SQL).

## The payoff

**Before** — what an agent normally emits:

```bash
find . -name "*.log" -mtime +7 -exec gzip {} \; && find . -name "*.gz" -mtime +30 -delete && du -sh .
```

**After** — what this produces:

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

…plus a one-line plain-English summary in chat before it runs. Seven rules do
the work — name values, comment the *why*, banner each phase, fail loud, preview
deletions, prose first — and the agent knows to skip all of it for a bare `ls`.
Full rules and per-language mechanics: [`AGENTS.md`](AGENTS.md). More transforms
(`python -c`, SQL): [`EXAMPLES.md`](EXAMPLES.md).

## Install

**Claude Code** (bundled skill, auto-discovered on startup):

```bash
git clone https://github.com/a-b/readable-ad-hoc-commands.git
cp -r readable-ad-hoc-commands/readable-ad-hoc-commands ~/.claude/skills/   # all projects
```

**Any other agent** — the rules are plain Markdown in [`AGENTS.md`](AGENTS.md);
drop them where your tool reads instructions:

| Agent | Where |
|---|---|
| Codex, Amp, Zed, Jules ([AGENTS.md](https://agents.md) convention) | `AGENTS.md` at repo root |
| Cursor | `.cursor/rules/readable-commands.mdc` |
| Windsurf | `.windsurf/rules/` |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Generic / API | Prepend `AGENTS.md` to the system prompt |

## Layout

```
AGENTS.md                       # harness-neutral rules (portable source of truth)
EXAMPLES.md                     # worked transforms for python -c and SQL
readable-ad-hoc-commands/
└── SKILL.md                    # Claude Code skill adapter (same rules + frontmatter)
```

## License

[MIT](LICENSE)
