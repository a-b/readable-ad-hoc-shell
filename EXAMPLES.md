# Worked transforms — non-shell languages

Companion to [SKILL.md](SKILL.md). The shell transform lives inline there; these
cover the other command languages. Same rules, adapted mechanics (see the
per-language table in SKILL.md).

## python -c → here-doc

A crammed `-c` is unreadable; a here-doc reads far better and lets you comment
the *why*.

**Before:**

```bash
python -c "import json,sys,glob;print(sum(len(json.load(open(f))['items']) for f in glob.glob('*.json')))"
```

**After:**

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

Rules shown: one step per line via here-doc (#1), comment the why (#2), prose
summary (#7).

## SQL — preview before a destructive write

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

Rules shown: preview destructive actions (#6), fail-fast via
`\set ON_ERROR_STOP on` (#5), prose summary (#7).
