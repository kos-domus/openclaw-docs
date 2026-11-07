---
title: "PostgreSQL Trigram Indexes and Function-Call Pitfalls in Batch Pipelines"
slug: postgres-trigram-function-pitfalls
category: troubleshooting
tags: ["postgresql", "pg-trgm", "performance", "gist-index", "batch-pipeline"]
sources:
- sessions/2026-11-06-decisore-locale-spesabot-pipeline-fase-v.md
last_updated: "2026-11-07"
version: 1
---

# PostgreSQL Trigram Indexes and Function-Call Pitfalls in Batch Pipelines

A weekly image-backfill pipeline over ~13,760 SKUs ran for hours and was killed twice by systemd timeouts. Root cause was not one bug but three compounding PostgreSQL usage mistakes. Fixing all three took the step from over 2 hours to 13 minutes, then to 1 min 50 s.

**Observed symptom:** the process logged `Images healed (canonical)` and then sat with near-zero CPU until the timer killed it. Near-zero CPU on a DB-bound step means the database is doing the work — profile the queries, not the script.

## Pitfall 1: `similarity(a, b) > k` bypasses the GiST trigram index

The `%` operator uses a GiST trigram index; calling `similarity()` and comparing the result manually does **not**. A trigram fallback branch written as:

```sql
WHERE similarity(name_norm, x) > 0.3
```

degrades to a sequential-scan-grade similarity computation per row (~0.4 s per call in the observed case).

**Fix:** use the operator, and set the threshold at runtime inside the function:

```sql
PERFORM set_config('pg_trgm.similarity_threshold', '0.3', true);
...
WHERE name_norm % x
```

Keep the explicit `similarity()` filters too if the query needs them — the operator handles the index path, the explicit filters keep semantics identical. Results were verified identical on 420 SKUs, including deliberately mangled names forced down the trigram branch.

Note: `ALTER FUNCTION ... SET pg_trgm.similarity_threshold` requires superuser because it is a placeholder (custom) parameter; `set_config(..., true)` (transaction-local) inside the function works without it.

## Pitfall 2: `UPDATE ... WHERE f(x) IS NOT NULL SET col = f(x)` calls f twice

Checking a function's result in the `WHERE` clause and then assigning it in `SET` evaluates the function **two times per row**. In a per-row lookup over thousands of rows this doubles the dominant cost silently.

**Fix:** compute once in a CTE and update from it:

```sql
WITH hit AS (
  SELECT id, lookup(x) AS v FROM candidates
)
UPDATE t SET col = hit.v FROM hit WHERE t.id = hit.id AND hit.v IS NOT NULL;
```

## Pitfall 3: Re-scanning the full backlog every run

The backfill step reprocessed *every* SKU without an image on each run, even though a run only touches a small slice of them.

**Fix:** scope work to rows touched by the current pass — e.g. only SKUs with `updated_at >= <run start>` — via a small `_since(ts)` helper. Backlog accumulation then stops mattering.

## Belt-and-braces: server-side timeout

Wrap the batch with a server-side statement timeout so a regression fails loudly instead of hanging until systemd kills it:

```bash
PGOPTIONS='-c statement_timeout=25min' psql ...
```

## Outcome

| Stage | Wall time |
|-------|-----------|
| Original (double lookup, no index, full backlog) | > 2 h (timeout) |
| Fix 1 + scoped scan (CTE, `_since`) | 13 min |
| Fix 3 (`%` operator + runtime threshold) | 1 min 50 s |

Related: the same run showed that a systemd `TERM` can be recorded as *success* by a bash wrapper — see [Cron Jobs, Sub-Agents, and Automation](../guides/cron-jobs-and-automation.md) for the trap-exit pitfall.
