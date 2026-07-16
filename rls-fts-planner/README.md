# Row-Level Security Silently Disabled My GIN Index

**Date:** 2026-07-16
**Author:** deutschmark

---

## Abstract

A public job board with 87,000 rows started returning `57014` (`statement_timeout`) on anonymous text search, while the identical query ran in milliseconds from a psql session and every `EXPLAIN` I looked at showed a clean Bitmap Index Scan. The cause was not bloat, not statistics, not connection pooling, and not the PostgREST schema cache. It was that PostgreSQL will not push a non-`LEAKPROOF` operator below a row-security barrier, `tsvector`'s `@@` is not leakproof, and so the GIN index was unusable **for the `anon` role only**. Every anonymous search degraded to a sequential scan measured at 10,032 ms against a 3 s statement timeout. The fix was to serve public reads through definer-semantics views with exposure identical to the policy they replaced. The general lesson is narrower and more useful than "RLS is slow": a plan you did not produce under the actual querying role is not evidence.

---

## 1. The symptom

The board is a Postgres table, `job_postings`, behind Supabase's PostgREST. Anonymous visitors search it. Logged-in users search it. Both hit the same table, the same index, the same query shape.

Anonymous search 500'd. Authenticated search did not.

```
ERROR:  canceling statement due to statement timeout
SQLSTATE: 57014
```

The table had a partial GIN index on a `search_vector` column and a public-read RLS policy that amounted to a single predicate:

```sql
create policy job_postings_public_read on public.job_postings
  for select to anon, authenticated
  using (is_active);
```

That is about as simple as a policy gets. It reads one boolean column. It calls no functions. It performs no subquery. There is nothing in it that looks like it could cost ten seconds.

The index was, if anything, better matched to the policy than usual:

```sql
create index job_postings_search_vector_idx on public.job_postings
  using gin (search_vector) where is_active;
```

Its partial predicate is the policy's predicate. The index exists to serve exactly the rows the policy exposes. It was still not used.

## 2. Four things that were not the problem

Worth recording, because each one is the obvious suspect and each one cost real time.

**Index bloat.** The board ingests continuously and rewrites rows on every crawl, so bloat was the first theory. `REINDEX` changed nothing.

**Stale statistics.** The table had recently grown from 16k to 87k rows. `ANALYZE` changed nothing.

**The PostgREST schema cache.** After out-of-band DDL, PostgREST genuinely does need `NOTIFY pgrst, 'reload schema'`, and role GUC changes need `'reload config'`. Both are real gotchas. Neither was this one.

**Exact counts.** The first *visible* symptom was a count query timing out, so I moved counts to PostgREST's `count: 'estimated'` and shipped "About N results" copy. That helped, and I kept it for cold-cache resilience, but it was treating a symptom: the row fetch was timing out too.

Each of these is a reasonable thing to chase. All four share a property: I could investigate them without reproducing the failure. That is exactly what made them attractive and exactly what made them a waste.

## 3. The measurement that mattered

Every `EXPLAIN ANALYZE` I ran looked perfect, because I ran all of them as the owner.

```sql
-- What I kept doing: a plan for a role that never queries this table.
explain analyze
select id, title from job_postings
where search_vector @@ websearch_to_tsquery('english', 'analyst');
--  Bitmap Heap Scan ... Bitmap Index Scan on job_postings_search_vector_idx
--  Execution Time: 4.1 ms
```

The owner bypasses row security. The plan above is a true plan, for a role that does not exist on this code path.

```sql
-- What I should have done first.
set local role anon;
explain analyze
select id, title from job_postings
where search_vector @@ websearch_to_tsquery('english', 'analyst');
--  Seq Scan on job_postings  (rows=87k)
--    Filter: (is_active AND (search_vector @@ ...))
--  Execution Time: 10032.4 ms
```

Same query. Same index. Same table. A different role, and a 2,400x difference.

The `anon` role's `statement_timeout` was 3 s. 10,032 ms against a 3 s budget is not a close call, which is why it presented as a hard, total failure rather than as slowness.

## 4. Root cause

When row security is active, the planner treats the policy as a **security barrier**. Qualifiers from your query cannot be evaluated before the policy's qualifiers, because a hostile qualifier could otherwise observe rows the policy is supposed to hide. The classic exploit is a `WHERE` clause calling a function that raises an error containing the row it saw, leaking data the user was never allowed to read.

To make this safe, Postgres will only push a user qualifier below a security barrier if the operators involved are marked `LEAKPROOF`: guaranteed to reveal nothing about their arguments, not through errors, not through side channels.

`tsvector @@ tsquery` is not marked leakproof.

```sql
select p.proname, p.proleakproof
from pg_proc p
join pg_operator o on o.oprcode = p.oid
where o.oprname = '@@';
--  ts_match_vq | f
```

So under RLS, `@@` gets hoisted **above** the barrier. It becomes a post-filter applied to rows the policy already returned, rather than an index qualifier used to find rows in the first place. And an operator that cannot be an index qualifier cannot use the GIN index. The planner does not warn. It does not error. It quietly produces the only plan it is permitted to produce, which is a sequential scan of all 87,000 rows.

The privilege that made my `EXPLAIN` fast is the same privilege that made it worthless: the owner has no barrier, so `@@` stays where it belongs, so the index gets used, so the plan looks great.

## 5. The fix

The policy's entire content was `is_active`. So the security requirement was: *anonymous readers may see active rows, all columns*. A view with definer semantics can state that requirement directly and let the planner see an unobstructed query underneath.

```sql
create or replace view public.board_jobs
with (security_invoker = off) as
  select * from public.job_postings where is_active;

grant select on public.board_jobs to anon, authenticated, service_role;
```

`security_invoker = off` means the view executes with the *definer's* rights, so the underlying table's RLS is not applied to the view's own scan, and the barrier disappears. What replaces it is the view's own `WHERE is_active`, which is not a weaker rule. It is character-for-character the rule the policy enforced.

That distinction is the whole of the argument for this being safe rather than a hole:

- Exposure is unchanged. Same rows (`is_active`), same columns (`*`).
- RLS stays **enabled** on `job_postings`. Every direct-table path, including every authenticated and service-role path, is still governed by policy. Nothing was disabled.
- The board's anonymous read path goes through the view. `useBoardData.js` queries `board_jobs` for search, listing, and detail. Other anon-visible surfaces still read `job_postings` directly and are unaffected, because a plain `is_active` filter never needed a non-leakproof operator in the first place. This is the point: the change is scoped to the queries that were actually broken, not applied table-wide as a reflex.

An aggregate view over the same table, `board_active_roles`, was failing the same way for the same reason and took the same change.

The result: anon text search returns from a Bitmap Index Scan in single-digit milliseconds.

I also raised `anon`'s `statement_timeout` from 3 s to 8 s to match `authenticated`. That is worth being clear about: it is headroom, not a fix. A 10 s sequential scan under an 8 s timeout is still a bug. If raising a timeout resolves your incident, you have not resolved your incident.

## 6. What generalizes

The narrow lesson is a one-liner: **if a table has RLS and serves an indexed operator to a public role, verify the plan as that role before you believe it.**

```sql
set local role anon;   -- or authenticated, or whatever actually queries
explain analyze <the real query>;
reset role;
```

The broader one is that this is not really about `@@`. It is about every non-leakproof operator that you expect to be index-backed:

- `pg_trgm`'s `%` similarity operator, for fuzzy name matching.
- PostGIS operators, for proximity search.
- Anything you find with `select proname from pg_proc where not proleakproof`.

Each one behaves identically: perfect from a privileged session, sequential scan under RLS, no warning at any point.

The failure mode this belongs to is worth naming on its own. It is not "slow query." It is **a system that behaves differently depending on who is asking, where the diagnostic tools default to asking as someone privileged**. Postgres RLS has that shape. So do most authorization layers. The reflex it should build is not "remember the leakproof rule," which is trivia you will forget. It is *reproduce as the role that failed, before forming a theory*. I had four theories before I had one measurement taken under the conditions of the bug, and every one of those theories was wrong. The measurement took thirty seconds and ended the investigation.

## 7. Notes

- Measured on Postgres 17.6 (Supabase), `job_postings` at ~87.8k rows, partial GIN index on a `tsvector` column.
- The estimated-count change (`count: 'estimated'`, "About N results") was kept after the view fix. Exact counts on a text search remain expensive on a cold cache, and an approximate number is honest as long as the UI says "About".
- PostgREST embeds through the view (`board_companies(...)`) resolve via base-table foreign keys and required no additional work.
