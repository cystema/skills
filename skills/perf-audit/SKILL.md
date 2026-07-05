---
name: perf-audit
description: Review code for structural performance problems — N+1 fan-out (DB, cache, HTTP, RPC), per-request work that should be per-startup, movable/cacheable work, allocation hotspots, and unbounded or over-fetching queries. Use when the user wants a review of where cost scales with traffic and data, rather than a micro-benchmark; when a request does I/O inside a loop, when latency grows with result-set size, when work is recomputed every request that could be precomputed, or before shipping code on a hot path. Trigger when asked about "N+1," "performance review," "why is this slow," "hot path," "too many queries," "over-fetching," or scaling concerns. Flags structural patterns and says what to profile — not a substitute for measurement. Produces a ranked report — not edits.
---

# Perf Audit

Most performance problems aren't slow lines — they're cheap work repeated at scale: one query per item, config parsed every request, a payload ten times bigger than needed. This skill finds **structural** cost — the patterns where spend grows with traffic × data — and says what to profile to confirm.

**Diagnoses and reports; does not edit. Flags structure; recommends measuring before optimizing.**

## The vocabulary

- **N+1** — one call to fetch a list, then one call per item. The canonical fan-out. Applies to DB, cache, HTTP, RPC — any I/O in a loop over results.
- **Hot path vs cold path** — hot = runs per-request or per-item; cold = runs per-startup or once. Cost on the hot path multiplies by traffic; the same cost on the cold path is paid once.
- **Fan-out** — one operation spawning many downstream calls. N+1 is fan-out in a loop; there's also fan-out across services in a single request.
- **Amortized cost** — the real per-operation cost once repetition is counted. O(1) inside a hot loop over N items is O(n) you didn't budget for.
- **Movable work** — computed per-request but effectively constant → precompute/cache; computed eagerly but rarely used → make it lazy.

## The one test that matters

> **Count the I/O and the allocations for one unit of user work — one request, one item processed. Does anything inside a loop touch the DB, cache, network, or filesystem? Does the per-unit cost grow with data you don't control? If cost scales with traffic × data, it's a hot-path problem waiting for load — regardless of how fast it feels on a dev machine with ten rows.**

## The audit order (do not reorder)

N+1 first (usually the biggest, most mechanical win), then hot/cold placement, then movable work, then allocations, then payload.

### 1. Hunt N+1 and fan-out

Look for I/O inside a loop over results:
- ORM lazy-loading in a `for` — accessing `order.customer.name` per order, each a query.
- `await` inside `.map`/`.forEach`, sequential per-item HTTP/RPC.
- Cache-get per item where a multi-get exists.
- Nested fan-out — N items each triggering M sub-calls (N×M).

For each: the loop, the per-item I/O, the count at realistic N, the fix (join/eager-load, batch endpoint, `IN` query, dataloader, multi-get).

### 2. Map hot vs cold placement

Work sitting on the per-request path that belongs at startup:
- Config/JSON parsing, regex compilation, template compilation inside the handler.
- Connection/client construction per request instead of a reused pool.
- Reflection, schema building, DI resolution repeated per call.

For each: the work, how often it truly changes, where to hoist it (module init, memoized singleton, startup).

### 3. Find movable work

- Per-request compute that's constant or slow-changing → cache/precompute (with an invalidation story).
- Eager work whose result is often unused → make it lazy.
- Synchronous work on an async path that blocks the event loop / ties up the worker → offload or await properly.

For each: the work, why it's movable, where it should move (cache, lazy, background job).

### 4. Check allocations in hot loops

- Per-item object churn, defensive copies, boxing inside tight loops.
- String concatenation in a loop instead of a builder/join.
- Large intermediate collections built only to be reduced.

Flag only the ones on a structurally hot path — don't chase allocations on cold code.

### 5. Check payload and result size

- Over-fetching — selecting all columns/fields when few are used; loading whole objects to read one attribute.
- Unbounded result sets — queries with no limit/pagination that grow with the table.
- Deep serialization — N-deep object graphs serialized per response.

For each: the over-fetch, how it grows, the bound/projection that fixes it.

## Output

Always a **report** — ranked findings, never edits. Every finding names the pattern and says **what to profile to confirm** — structure points to the suspect; measurement convicts.

Rank: N+1/fan-out first (biggest structural wins), then hot/cold misplacement, then movable work, then allocations, then payload.

### Report structure

```
# Perf audit — <target>

**Verdict:** <one sentence — where does cost scale with traffic × data?>

## What's efficient ✨
<Patterns that stay flat under load — batched I/O, hoisted setup, bounded
queries. The convention to keep.>

## N+1 & fan-out 🔁
<Ranked. For each: the loop, per-item I/O, count at realistic N, the batch/join
fix, what to profile to confirm.>

## Hot-path work that should be cold 🔥
<Ranked. For each: the per-request work, how often it truly changes, where to
hoist it.>

## Movable work
<Ranked. For each: the work, why movable, target (cache/lazy/background).>

## Allocation hotspots
<Ranked. For each: the loop, the churn, the leaner shape. Hot paths only.>

## Payload & over-fetch
<Ranked. For each: what's over-fetched/unbounded, how it grows, the projection
or limit that bounds it.>

## Where to start
<Numbered: N+1 first (measure query counts), then hoist setup, then cache/lazy,
then allocations/payload. Note what to profile at each step.>
```

Tie every finding to a real location and a concrete cost-at-scale. The report's job is to point profiling at the structurally suspect spots — the cheap work repeated until it isn't — not to guess at micro-optimizations.
