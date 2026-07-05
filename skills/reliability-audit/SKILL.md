---
name: reliability-audit
description: Review code for what goes wrong under concurrency, retries, and slow dependencies — races, non-idempotent operations reachable from retrying callers, lock-scope problems, and missing or misordered timeouts. Use when the user wants a review of behavior under load and failure conditions rather than logic correctness; when a bug only appears under concurrency, when a queue/webhook/client retry double-charges or double-sends, when requests hang on a slow dependency, or before shipping code that handles concurrent access or external I/O. Trigger when asked about "race conditions," "idempotency," "retry safety," "thread safety," "timeouts," "locks," or "what happens under load." Pairs with error-budget (which reviews how errors are handled after they occur). Produces a ranked report — not edits.
---

# Reliability Audit

Code that passes every single-threaded test can still corrupt state under concurrency, double-charge under retry, or hang on a slow dependency. This skill reviews the **pre-failure** conditions — what *causes* those incidents. It pairs with [error-budget], which reviews how errors are *handled* once they occur; this one asks what breaks under load, retry, and concurrency in the first place.

**Diagnoses and reports; does not edit.**

## The vocabulary

- **Race** — the outcome depends on interleaving. Lost update, double-fire, check-then-act (TOCTOU), read-modify-write without atomicity.
- **Idempotent** — applying the operation twice leaves the same state as applying it once. Required for anything a retry can re-deliver.
- **At-least-once** — the delivery guarantee most queues, webhooks, and retrying clients actually provide. It means downstream *must* tolerate duplicates; "exactly once" is usually a comforting lie.
- **Timeout budget** — every external call bounded, and bounded *shorter* than the caller's own timeout — otherwise upstream gives up first and your wait is wasted work.
- **Lock scope** — too wide serializes everything (throughput death); too narrow doesn't actually cover the invariant it's meant to protect.
- **Critical section** — the span that must execute atomically. Bugs live where atomicity is assumed but not enforced.

## The one test that matters

> **Run this operation twice at the same instant, and twice in a row after a retry. Does the system end in the same correct state both times? If concurrent runs corrupt each other, or a retry double-charges / double-sends / double-inserts, it's a reliability bug — no matter how green the single-threaded tests are.**

## The audit order (do not reorder)

Races and retry-safety first (silent data corruption), then locks, then timeouts (hangs under dependency slowness).

### 1. Hunt races

Look for state whose correctness depends on interleaving:
- **Check-then-act / TOCTOU** — `if (!exists) create()`, `if (balance >= x) debit()` without atomicity between check and act.
- **Read-modify-write** — load a counter/row, change it in app code, write it back; two runners clobber each other.
- **Unguarded shared state** — module-level vars, singletons, in-memory caches mutated from concurrent requests/threads/goroutines.
- **Non-atomic multi-step writes** — two rows/two systems updated with no transaction; a concurrent reader sees the half-done state.

For each: the concurrent-double-run scenario, the corrupted outcome, the fix (atomic op, transaction, compare-and-swap, DB constraint, serialize the section).

### 2. Check idempotency at every retry source

Enumerate handlers reachable from something that retries: queue consumers, webhook endpoints, cron-with-retry, client calls with retry, at-least-once event streams. For each, ask: **what happens if this runs twice?**
- Non-idempotent write (insert without a unique key, `balance += x`, "send email") on a retry path = duplicate bug.
- For each: the retry source, the duplicate effect (double charge / double row / double notification), the fix (idempotency key, upsert, unique constraint, dedup table, natural-key `INSERT ... ON CONFLICT`).

### 3. Audit lock scope

- **Too wide** — a global lock around a whole handler where only one line touches shared state; throughput collapses under load.
- **Too narrow** — the lock doesn't span the full invariant (guards the read but not the dependent write).
- **Missing** — shared mutable state with no guard at all.
- **Deadlock shape** — two locks acquired in different orders on different paths.

For each: the invariant, what the current scope actually protects vs needs to, the corrected boundary.

### 4. Check timeout budgets

For every external I/O (DB, HTTP, RPC, cache, lock acquisition):
- Is there a timeout at all? Anywhere without one is a hang waiting for the dependency to stall.
- Is it *shorter* than the timeout of whoever calls this code? If not, upstream abandons the request first and your retries burn resources for nothing.
- Is retry bounded (max attempts **and** max total duration) with jittered backoff? Unbounded retry without jitter is a thundering-herd generator.

For each: the call, the missing bound, the worst-case hang/amplification. (Error-budget owns how the resulting timeout error is surfaced; here, flag only the missing bound.)

### 5. Re-examine "safe-looking" concurrency

Code written as if single-threaded that isn't:
- Lazy init / memoization / singletons built without synchronization, initialized twice under a race.
- Connection pools, buffers, or clients shared across workers assuming exclusive access.
- Iterators/collections mutated while another path reads them.

For each: the hidden concurrency assumption, the symptom when it's violated.

## Output

Always a **report** — ranked findings, never edits.

Rank: races (silent corruption) → non-idempotent retry paths (duplicate side effects) → lock problems → timeout/retry gaps (hangs & amplification).

### Report structure

```
# Reliability audit — <target>

**Verdict:** <one sentence — does this hold up under concurrency, retry, and slow dependencies?>

## What's reliable ✨
<Patterns that are correct under load — atomic ops, idempotency keys, bounded
timeouts. The convention to spread.>

## Races 🏁
<Ranked. For each: the interleaving, the corrupted outcome, the fix
(atomic/transaction/CAS/constraint/serialize).>

## Non-idempotent retry paths 🔁
<Ranked. For each: the retry source, the duplicate effect, the fix
(idempotency key/upsert/unique constraint/dedup).>

## Lock problems 🔒
<Ranked. For each: the invariant, what scope protects vs needs to, deadlock risk,
corrected boundary.>

## Timeout & retry gaps ⏱
<Ranked. For each external call: missing bound, ordering vs upstream timeout,
retry/backoff gaps, worst-case hang or amplification.>

## Where to start
<Numbered: races first, then retry idempotency, then locks, then timeouts.
Cross-reference error-budget for how the resulting errors get surfaced.>
```

Tie every finding to a real location and a concrete concurrent-or-retry scenario. The report's job is to make the code correct when the world is hostile — running things twice at once, retrying blindly, and stalling at the worst moment.
