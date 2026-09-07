# How to Read EXPLAIN (ANALYZE) and Fix a Slow Postgres Query

When a Postgres query feels slow, guessing—"maybe add an index?"—wastes time and sometimes makes writes worse. `EXPLAIN (ANALYZE, BUFFERS)` shows what the planner chose and what actually happened at runtime. This tutorial walks through reading a real-shaped plan, spotting the usual villains (sequential scans on large tables, bad row estimates, nested loops gone wrong), and applying a selective index plus a tighter filter.

You should be comfortable with basic SQL (`SELECT`, `JOIN`, `WHERE`). No extension beyond stock Postgres is required.

## The scenario

Imagine an orders table used by a support dashboard:

```sql
CREATE TABLE orders (
    id            bigserial PRIMARY KEY,
    customer_id   bigint NOT NULL,
    status        text NOT NULL,
    total_cents   integer NOT NULL,
    created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX orders_customer_id_idx ON orders (customer_id);
```

Load enough rows that plans matter (hundreds of thousands or more in a real system). For a local reproduction you can synthesize data:

```sql
INSERT INTO orders (customer_id, status, total_cents, created_at)
SELECT
    (random() * 50000)::bigint + 1,
    (ARRAY['pending', 'paid', 'shipped', 'cancelled'])[1 + floor(random() * 4)::int],
    (random() * 20000)::int + 100,
    now() - (random() * interval '365 days')
FROM generate_series(1, 500000);
```

The slow query the dashboard runs:

```sql
SELECT id, customer_id, total_cents, created_at
FROM orders
WHERE status = 'paid'
  AND created_at >= now() - interval '7 days'
ORDER BY created_at DESC
LIMIT 50;
```

Only customer lookups are indexed. Filtering by `status` and a recent time window still touches a large fraction of the heap—or the whole heap—depending on data distribution.

## Capture a useful plan

Always use `ANALYZE` so you see actual timings and row counts, not only estimates. `BUFFERS` adds I/O detail:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT id, customer_id, total_cents, created_at
FROM orders
WHERE status = 'paid'
  AND created_at >= now() - interval '7 days'
ORDER BY created_at DESC
LIMIT 50;
```

A representative (abbreviated) plan before tuning looks like this:

```text
Limit  (cost=... rows=50 width=...) (actual time=42.//..45.2 rows=50 loops=1)
  Buffers: shared hit=8120 read=3200
  ->  Sort  (cost=... rows=12400 width=...) (actual time=42.1..42.2 rows=50 loops=1)
        Sort Key: created_at DESC
        Sort Method: top-N heapsort  Memory: 27kB
        ->  Seq Scan on orders  (cost=0.00..17800 rows=12400 width=...)
              (actual time=0.03..38.9 rows=9612 loops=1)
              Filter: ((status = 'paid'::text) AND (created_at >= ...))
              Rows Removed by Filter: 490388
              Buffers: shared hit=8120 read=3200
Planning Time: 0.2 ms
Execution Time: 45.3 ms
```

On a cold cache or a multi-million-row table, that sequential scan becomes hundreds of milliseconds or seconds. The shape of the problem is already visible.

## How to read the plan top-down and bottom-up

### Nodes and arrows

Postgres prints a tree. Indentation means "child feeds parent." Data flows upward: the `Seq Scan` produces rows, `Sort` orders them, `Limit` keeps fifty.

### Cost vs actual

- **cost=startup..total** — planner estimates in abstract units (not milliseconds). Useful for comparing alternatives the planner considered.
- **actual time=start..total** — real milliseconds for that node, averaged per loop.
- **rows=** in the estimate vs **actual rows=** — when these diverge a lot, the planner may pick a bad strategy.

Here the filter estimate (`rows=12400`) is in the same ballpark as actual (`9612`). The issue is not a catastrophic mis-estimate; it is that a seq scan plus sort is still expensive.

### Rows Removed by Filter

Almost half a million rows were read and discarded. That is the smoking gun: you are paying to read data you do not need. Indexes exist to avoid that work when selectivity is good.

### Buffers

`shared hit` means pages already in `shared_buffers`. `read` means OS/disk reads. High `read` on a hot query path is a latency and cache-pressure problem even if CPU time looks modest.

### Sort Method

`top-N heapsort` is relatively cheap for `ORDER BY ... LIMIT N`. Still, sorting 9k+ rows that you only needed fifty of is wasted work if an index can return them already ordered.

## Fix 1: a selective composite index

For this query pattern—filter on `status` and a range on `created_at`, order by `created_at DESC`—a composite index matching filter + order is the usual win:

```sql
CREATE INDEX CONCURRENTLY orders_paid_created_at_idx
ON orders (created_at DESC)
WHERE status = 'paid';
```

This is a **partial index**: it only stores paid orders. Partial indexes stay smaller and more cache-friendly when one status (or tenant, or soft-delete flag) dominates a hot path.

If you need several statuses with the same shape, a non-partial composite index works:

```sql
CREATE INDEX CONCURRENTLY orders_status_created_at_idx
ON orders (status, created_at DESC);
```

Re-run `EXPLAIN (ANALYZE, BUFFERS)`:

```text
Limit  (actual time=0.04..0.08 rows=50 loops=1)
  Buffers: shared hit=4
  ->  Index Scan using orders_paid_created_at_idx on orders
        (actual time=0.03..0.07 rows=50 loops=1)
        Index Cond: (created_at >= ...)
        Buffers: shared hit=4
Execution Time: 0.10 ms
```

What changed:

1. **Seq Scan → Index Scan** (or Index Only Scan if the index covers selected columns).
2. **Rows Removed by Filter** disappears or shrinks dramatically.
3. **Buffers** drop from thousands of pages to a handful.
4. The explicit **Sort** may vanish because the index order satisfies `ORDER BY created_at DESC`.

## Fix 2: tighten the predicate (selectivity)

Indexes help when the predicate is selective. If `status = 'paid'` matches 80% of rows, a partial index on paid rows is still large, and a status-leading btree may not beat a seq scan. Check selectivity:

```sql
SELECT status, count(*) AS n,
       round(100.0 * count(*) / sum(count(*)) OVER (), 2) AS pct
FROM orders
GROUP BY status
ORDER BY n DESC;
```

And for the time window:

```sql
SELECT count(*) FROM orders
WHERE status = 'paid'
  AND created_at >= now() - interval '7 days';
```

If the dashboard only needs one week of data, keep that filter in the query (do not fetch a year in the app and filter in Python). Pushing selective filters into SQL is still one of the highest-leverage performance habits.

Sometimes the query is slow because of a **function-wrapped column** that prevents index use:

```sql
-- Bad: cannot use a plain index on created_at
WHERE date_trunc('day', created_at) = CURRENT_DATE

-- Better: range on the bare column
WHERE created_at >= CURRENT_DATE
  AND created_at < CURRENT_DATE + interval '1 day'
```

The Postgres planner documentation discusses sargable predicates and index usage extensively; the practical rule is: keep the indexed column alone on one side of the comparison when you can.

## Fix 3: when estimates lie

Suppose `ANALYZE` was never run after a bulk load. Estimates might say `rows=100` while actual is `rows=200000`, pushing the planner toward a nested loop that explodes.

```sql
ANALYZE orders;
```

For joins, compare estimated vs actual at each node. A nested loop with `loops=50000` and a slow inner scan often means "wrong join order / missing index on the inner side." Add the lookup index, or rewrite so the selective filter is applied first.

## A short checklist while reading plans

1. Find the node with the largest **actual time**.
2. Check **Rows Removed by Filter**—scanning to discard is a red flag.
3. Compare **estimated rows** vs **actual rows** (order-of-magnitude misses hurt).
4. Note **Buffers read** vs **hit** for cache behavior.
5. Ask whether an index can satisfy **filter + order** together.
6. Prefer `CREATE INDEX CONCURRENTLY` on production to avoid long write locks.
7. Re-`EXPLAIN (ANALYZE)` after each change; keep the winner, drop unused indexes later.

## Conclusion and takeaways

`EXPLAIN (ANALYZE, BUFFERS)` turns performance from folklore into evidence. In this walkthrough, a sequential scan filtered away nearly the entire table; a partial index on `created_at` for `status = 'paid'` turned the plan into a tight index scan with trivial I/O.

Takeaways:

- Always measure with `ANALYZE` (and `BUFFERS` when I/O matters).
- Optimize the node that actually burns time, not the one with a scary-looking cost label alone.
- Match indexes to **predicates and sort order**; consider partial indexes for hot subsets.
- Keep predicates sargable and selective; fix stale statistics with `ANALYZE`.
- Validate with a second explain—plans are the feedback loop.

Once you can read a plan fluently, "the database is slow" becomes a concrete engineering problem with a concrete next index, rewrite, or statistics fix.

