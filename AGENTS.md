# Readable ad-hoc shell — agent instructions

These are harness-neutral instructions for **any** coding agent. Drop them into
a system prompt, an `AGENTS.md`, a Cursor/Windsurf rules file, or load them as a
Claude Code skill (see [`readable-ad-hoc-shell/SKILL.md`](readable-ad-hoc-shell/SKILL.md)).
The goal: when you run non-trivial shell, format it so the human can grasp its
intent, steps, and blast radius **before** it executes — instead of a dense
one-liner they must reverse-engineer or approve blind.

## When this applies

Apply the rules below to any shell that is not a trivial single command:

- Multi-line scripts, or `&&` / `;` / `|` chains of 2+ meaningful steps
- Loops (`for`/`while`), conditionals, functions, `xargs`
- Here-docs (`<<EOF`) and multi-line string literals
- Non-obvious flags, subshells, process substitution, complex quoting/escaping
- Anything that **writes, deletes, moves, or overwrites** files or state

A bare `ls`, `git status`, or `cat file` needs none of this — just run it.

## The rules

1. **One step per line, top to bottom.** Break dense one-liners into a readable
   sequence the reader can scan. Prefer newlines over `;`.
2. **Comment the *why*, not the obvious.** A short `#` above each logical block
   explaining its purpose or effect. Skip comments that restate the command.
3. **Name things.** Pull magic values, paths, and repeated strings into named
   variables at the top (`readonly` for constants). A named variable is
   self-documenting.
4. **Announce progress for long scripts.** `echo` a short banner before each
   phase so the human can follow the output, not just the source.
5. **Fail loud and early.** Start real scripts with `set -euo pipefail` (or the
   equivalent for the language) so a failed step stops the run.
6. **Preview destructive actions.** Before anything that deletes/overwrites,
   echo exactly what will be affected or do a dry run first. Never bury an
   `rm -rf` mid-pipeline.
7. **Say what it does in prose, first.** Before running, give a one- to
   three-line plain-English summary of what the command does and what changes to
   expect. Then show the command.

## Transform: before → after

**Before** (typical machine-optimized one-liner):

```bash
find . -name "*.log" -mtime +7 -exec gzip {} \; && find . -name "*.gz" -mtime +30 -delete && du -sh .
```

**After** (readable):

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

Prose to precede it:

> This gzips any `.log` file older than 7 days, then deletes `.gz` archives
> older than 30 days, and finally prints the tree's total size. It **modifies
> and removes files** under the current directory.

## Anti-patterns

- **Wall-of-text one-liner** with five `&&` — split it.
- **Silent destruction** — `rm`, `>`, `truncate`, `git reset --hard`,
  `DROP TABLE` with no preview or summary.
- **Cleverness over clarity** — a nested `awk`/`sed`/`xargs` incantation when a
  short loop reads better. Optimize for the reader, not for line count.
- **Comment spam** — restating every command defeats the purpose; comment blocks
  and intent, not lines.

## Balance

Match the effort to the command. A two-command chain needs a one-line prose
summary and maybe a comment — not a `set -euo pipefail` header and banners.
Reserve the full treatment for genuinely complex or destructive scripts. The
goal is human understanding, not ceremony.

> **Not shell-only.** The same rules apply to any command language an agent
> emits ad hoc — PowerShell, Python `-c` snippets, `jq` programs, SQL. Adapt the
> mechanics (comment syntax, fail-fast idiom) to the language; keep the intent.
