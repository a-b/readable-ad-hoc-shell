---
name: readable-ad-hoc-commands
description: >
  Format the ad-hoc commands you run so the user reads their intent, steps, and
  blast radius before they execute — shell, `python -c`/`node -e`, `jq`, or SQL.
  Use whenever about to run a non-trivial command (multi-step, piped, looped,
  here-doc, or a dense inline snippet), or anything that writes, deletes, or
  overwrites state.
---

# Readable Ad-hoc Commands

Ad-hoc commands an agent writes are usually optimized for the machine: one dense
line, cryptic flags, no explanation. The user then has to reverse-engineer what
it does — or worse, approve it without understanding. Make the commands you run
**legible**: the user should grasp intent, steps, and **blast radius** before the
command executes. This is about throwaway *commands* in any terminal/REPL
language — not maintained application code, which is code review's job.

## When this applies

Apply the rules to any command that is not a trivial single invocation:

- Multi-line scripts, or `&&` / `;` / `|` chains of 2+ meaningful steps
- Loops, conditionals, functions, `xargs`
- Here-docs (`<<EOF`) and multi-line string literals
- Dense inline snippets: `python -c "..."`, `node -e "..."`, `jq '...'`, `psql -c "..."`
- Non-obvious flags, subshells, process substitution, complex quoting/escaping
- Anything that **writes, deletes, moves, or overwrites** files, tables, or state

A bare `ls`, `git status`, or `SELECT 1` needs none of this — just run it.

## The rules

Language-neutral; the *mechanics* (comment char, fail-fast idiom) differ per
language — see the table.

1. **One step per line, top to bottom.** Break dense one-liners into a readable
   sequence the reader can scan. Prefer newlines over `;`. For inline snippets,
   use a here-doc or real multi-line string instead of cramming everything onto
   one `-c` line.
2. **Comment the *why*, not the obvious.** A short comment above each logical
   block explaining its purpose or effect. Skip comments that restate the command.
3. **Name things.** Pull magic values, paths, and repeated strings into named
   variables/bindings at the top. A named value is self-documenting.
4. **Announce progress for long runs.** For multi-phase work, print a short
   banner before each phase so the user follows the output, not just the source.
5. **Fail loud and early.** Start real scripts with the language's fail-fast
   idiom so a failed step stops the run instead of barreling on.
6. **Preview destructive actions.** Before anything that deletes/overwrites,
   print exactly what will be affected, dry-run it, or wrap it in a transaction
   you can roll back. Never bury a destructive step mid-pipeline.
7. **Say what it does in prose, first.** Give the user a one- to three-line
   plain-English summary of what the command does and what changes to expect.
   Then show the command.

## Per-language mechanics

Same rules, adapted to each language's idioms:

| Language | Multi-line form | Comment | Fail-fast (rule 5) | Preview / undo (rule 6) |
|---|---|---|---|---|
| **bash/sh** | here-doc, `\` continuations | `#` | `set -euo pipefail` | dry-run, echo targets before `rm` |
| **PowerShell** | script block, backtick | `#` | `$ErrorActionPreference='Stop'` | `-WhatIf` / `-Confirm` |
| **python -c** | `python - <<'PY' … PY` here-doc | `#` | exceptions abort by default | print planned changes; commit at end |
| **node -e** | `node - <<'JS' … JS` here-doc | `//` | `'use strict'`; throw on error | log intended writes first |
| **jq** | `-f file.jq` or multi-line `'…'` | `#` | `-e` (exit non-zero on empty) | run without `-i`/output-only first |
| **sql (psql -c)** | here-doc or `-f file.sql` | `--` | `\set ON_ERROR_STOP on` | wrap in `BEGIN; … ` + review before `COMMIT` |

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

> Prose: *"Gzips `.log` files older than 7 days, deletes `.gz` archives older
> than 30 days, then prints the tree's total size. It **modifies and removes
> files** under the current directory."*

For the same treatment in other languages — a crammed `python -c` unfolded into a
here-doc, and a `psql` `DELETE` made safe with preview-and-transaction — see
[EXAMPLES.md](EXAMPLES.md).

## Anti-patterns

- **Silent destruction** — `rm`, `>`, `truncate`, `git reset --hard`,
  `DELETE`/`DROP`, `jq -i` in place, with no preview or summary (rule 6).
- **Cleverness over clarity** — a nested `awk`/`sed`/`xargs` incantation or a
  golfed comprehension when a short loop reads better. Optimize for the reader,
  not for line count.

## Balance

Match the effort to the command. A two-command chain needs a one-line prose
summary and maybe a comment — not a fail-fast header and banners. Reserve the
full treatment for genuinely complex or destructive commands: understanding, not
ceremony.
