---
name: readable-ad-hoc-commands
description: >
  Format ad-hoc terminal commands so the user can read them and understand what
  is happening before and while they run — shell scripts, but also inline
  interpreter snippets (`python -c`, `node -e`), `jq` programs, and `psql`/`sql`
  one-liners. Use whenever you are about to run a non-trivial command: a
  multi-line script, a pipeline, a loop, a here-doc, chained `&&`/`;` commands,
  a dense `-c`/`-e` snippet, or anything with quoting/escaping the user would
  struggle to parse at a glance.
---

# Readable Ad-hoc Commands

Ad-hoc commands an agent writes are usually optimized for the machine: one dense
line, cryptic flags, no explanation. The user then has to reverse-engineer what
it does — or worse, approve it without understanding. Make the commands you run
**legible**: the user should grasp intent, steps, and blast radius before the
command executes.

This applies to **any command language you fire at a terminal or REPL**, not
just bash — `python -c`, `node -e`, `jq`, `psql -c`, `awk`. It is about
throwaway *commands*, not maintained programs; writing clean application code is
a different concern (that's code review's job).

## When this applies

Apply the rules to any command that is not a trivial single invocation:

- Multi-line scripts, or `&&` / `;` / `|` chains of 2+ meaningful steps
- Loops, conditionals, functions, `xargs`
- Here-docs (`<<EOF`) and multi-line string literals
- Dense inline snippets: `python -c "..."`, `node -e "..."`, `jq '...'`,
  `psql -c "..."`
- Non-obvious flags, subshells, process substitution, complex quoting/escaping
- Anything that **writes, deletes, moves, or overwrites** files, tables, or state

A bare `ls`, `git status`, or `SELECT 1` needs none of this — just run it.

## The rules

These are language-neutral. The *mechanics* (comment char, fail-fast idiom)
differ per language — see the table below.

### 1. One step per line, top to bottom
Break dense one-liners into a readable sequence. The reader should be able to
scan line-by-line and follow the logic. Prefer newlines over `;`. For inline
snippets, use a here-doc or a real multi-line string instead of cramming
everything onto one `-c` line.

### 2. Comment the *why*, not the obvious
Add a short comment above each logical block explaining its purpose and, if
relevant, its effect. Skip comments that just restate the command.

### 3. Name things
Pull magic values, paths, and repeated strings into named variables/bindings at
the top. A named value is self-documenting.

### 4. Announce progress for long runs
For multi-phase work, print a short banner before each phase so the user can
follow along in the output, not just the source.

### 5. Fail loud and early
Start real scripts with the language's fail-fast idiom so a failed step stops
the run instead of silently barreling on. Mention it exists; don't over-explain.

### 6. Preview destructive actions
Before anything that deletes/overwrites, print exactly what will be affected, or
do a dry run first (or wrap it in a transaction you can roll back). Never bury a
destructive step mid-pipeline.

### 7. Say what it does in prose, first
Before running, give the user a one- to three-line plain-English summary of what
the command does and what changes to expect. Then show the command.

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

### Shell

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

> Prose: *"Gzips `.log` files older than 7 days, deletes `.gz` archives older
> than 30 days, then prints the tree's total size. It **modifies and removes
> files** under the current directory."*

### python -c

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

> Prose: *"Read-only: counts total `items` across all `*.json` files in the
> current directory and prints the sum."*

### SQL — preview before a destructive write

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

> Prose: *"Deletes `sessions` rows not seen in 90+ days. It first prints how
> many rows match, then deletes them inside a transaction. This **removes
> database rows**."*

## Anti-patterns

- **Wall-of-text one-liner** with five `&&`, or a 200-char `python -c` — split
  it into lines / a here-doc.
- **Silent destruction** — `rm`, `>`, `truncate`, `git reset --hard`,
  `DELETE`/`DROP`, `jq -i` in place, with no preview or summary.
- **Cleverness over clarity** — a nested `awk`/`sed`/`xargs` incantation (or a
  golfed comprehension) when a short loop reads better. Optimize for the reader,
  not for line count.
- **Comment spam** — restating every line defeats the purpose; comment blocks
  and intent, not lines.

## Balance

Match the effort to the command. A two-command chain needs a one-line prose
summary and maybe a comment — not a fail-fast header and banners. Reserve the
full treatment for genuinely complex or destructive commands. The goal is user
understanding, not ceremony.
