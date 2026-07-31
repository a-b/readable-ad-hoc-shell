---
name: readable-ad-hoc-shell
description: >
  Format ad-hoc shell scripts and multi-step bash commands so the user can
  read them and understand what is happening before and while they run. Use
  whenever you are about to run a non-trivial shell command — a multi-line
  script, a pipeline, a loop, a here-doc, chained `&&`/`;` commands, or
  anything with quoting/escaping the user would struggle to parse at a glance.
---

# Readable Shell

Ad-hoc shell that an agent writes is usually optimized for the machine: one
dense line, cryptic flags, no explanation. The user then has to reverse-engineer
what it does — or worse, approve it without understanding. Make the shell you run
**legible**: the user should grasp intent, steps, and blast radius before the
command executes.

## When this applies

Trigger on any shell that is not a trivial single command. Concretely:

- Multi-line scripts, or `&&` / `;` / `|` chains of 2+ meaningful steps
- Loops (`for`/`while`), conditionals, functions, `xargs`
- Here-docs (`<<EOF`) and multi-line string literals
- Non-obvious flags, subshells, process substitution, complex quoting/escaping
- Anything that **writes, deletes, moves, or overwrites** files or state

A bare `ls`, `git status`, or `cat file` needs none of this — just run it.

## The rules

### 1. One step per line, top to bottom
Break dense one-liners into a readable sequence. The reader should be able to
scan line-by-line and follow the logic. Prefer newlines over `;`.

### 2. Comment the *why*, not the obvious
Add a short `#` comment above each logical block explaining its purpose and, if
relevant, its effect. Skip comments that just restate the command
(`# list files` above `ls` is noise).

### 3. Name things
Pull magic values, paths, and repeated strings into named variables at the top.
`readonly` for constants. A named variable is self-documenting.

### 4. Announce progress for long scripts
For scripts with several phases, `echo` a short banner before each phase so the
user can follow along in the output, not just the source.

### 5. Fail loud and early
Start real scripts with `set -euo pipefail` so a failed step stops the run
instead of silently barreling on. Mention it exists; don't over-explain it.

### 6. Preview destructive actions
Before anything that deletes/overwrites, either echo exactly what will be
affected, or do a dry run first. Never bury an `rm -rf` mid-pipeline.

### 7. Say what it does in prose, first
Before running, give the user a one- to three-line plain-English summary of what
the command will do and what changes to expect. Then show the command.

## Transform: before → after

**Before** (what an agent typically emits):

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
goal is user understanding, not ceremony.
