---
name: error-budget
description: Map and audit all error paths in code — flag silently swallowed errors, errors caught too far from where they arise, and defensive noise that drowns the happy path. Use when the user wants a failure-mode review of a file, module, service, or whole codebase; when they suspect bugs are being eaten by try/catch blocks, when production failures don't surface in logs, when error handling feels inconsistent, or when they ask to "audit error handling," "find swallowed errors," "review failure paths," or "check our retry/timeout/fallback logic." Trigger whenever code is suspiciously "robust" (lots of try/except with no logging), when on-call sees symptoms but no errors, when fallbacks hide upstream breakage, or before shipping anything that handles external I/O. Produces a ranked report of failure-mode problems — not edits.
---

# Error Budget

Failure paths are part of the interface. In joyful code, error handling is as legible as the happy path; in painful code, errors are swallowed, caught far from where they arise, or so heavily guarded the happy path drowns.

This skill audits whether the program **tells the truth about its failures**.

**Diagnoses and reports; does not edit.**

## The vocabulary

- **Surfaced** — error reaches someone who can act: user, operator (logs/alerts), or caller equipped to retry/fallback. The desired default.
- **Swallowed** — caught and discarded. No log, no rethrow, no fallback. The bug equivalent of muting an alarm.
- **Faked** — caught and replaced with a fake-success value (empty list, `null`, default). Correct only if the caller can tell the difference.
- **Caught too far** — caught many layers above origin; loses context needed to handle intelligently.
- **Caught too close** — caught and rethrown at the same layer; noise without decision.
- **Defensive noise** — guards around code that can't actually fail the way being guarded.
- **Retry/fallback as cover** — loops or fallbacks that mask upstream breakage; system "self-heals" into silent degradation.
- **Boundary** — layer where errors must be summarized/converted (HTTP handler, queue consumer, CLI entry, public API). Errors travel as exceptions/Results *within* a boundary, get translated *at* the boundary.

## The one test that matters

> **For every catch / except / `.catch` / `Result::Err` / `if err != nil`: imagine the caught error is a real production incident at 3am. Does this handler give on-call the information and signal to act? If the error vanishes, becomes an unactionable log line, or quietly degrades the system, it's a failure-mode bug.**

## The audit order (do not reorder)

Swallows first (worst-case-now), then boundaries (worst-case-future), then noise.

### 1. Enumerate every catch site and classify

List every `try/catch`, `try/except`, `.catch()`, `Result`/`Either` consumer, `if err != nil`, `rescue`, panic recovery, middleware handler. For each, classify the action:

- **Surface** — rethrow, return Err, propagate, log+alert.
- **Convert** — translate to domain error and rethrow at a boundary.
- **Recover** — handle locally with sensible value (must be intentional and documented).
- **Swallow** — caught and discarded.
- **Fake** — caught and replaced with default.
- **Log-only** — logged then swallowed. A swallow in disguise.

Clean code is mostly Surface and Convert with intentional Recover at chosen seams. Concentration of Swallow / Fake / Log-only is a finding.

### 2. Hunt swallowed errors first

Every swallow is potentially an invisible production bug. Patterns:

- `catch (e) {}`, `catch (e) { /* ignore */ }`, `pass`, `_ = err`, `if err != nil { return nil }` without logging.
- `try: ... except Exception: pass`, bare `except:`.
- `.catch(() => {})` or `.catch(() => null)` in promise chains.
- Background tasks / goroutines / workers / timers / event listeners whose errors have no destination.
- Promise chains missing terminal `.catch` or `await`.

For each: name what error gets swallowed, what state the system ends up in, how on-call would learn about the resulting symptom.

### 3. Hunt fakes and silent fallbacks

Faking is sometimes correct (`getCachedValueOr(default)`), sometimes catastrophic (`fetchPermissions().catch(() => [])` silently denies or grants access).

For every fake, ask:
- Can the caller distinguish the faked value from a real one? If not, what wrong decision results?
- Is the default *safe* (denies access, returns empty, surfaces "no data") or *unsafe* (returns stale data as fresh, defaults to permissive)?
- Does anything alert when the fallback fires repeatedly?
- Does the fallback have a budget — pages if it triggers > N%?

Retry loop with no max, no backoff, no alerting = fake-success machine waiting to be a thundering-herd machine.

### 4. Trace where errors travel

Pick top failure modes (network down, disk full, DB unreachable, malformed input, auth expired, dependency timeout). For each, trace origin → destination. Findings live in the gap:

- **Caught too far** — `db.query` error ends up in generic 500 handler with no context. Can't retry, can't degrade, can't even tell the user what failed.
- **Caught too close** — `db.query` wrapped in try/catch that rethrows unchanged. Pure noise.
- **Boundary-less travel** — domain exceptions leak through HTTP/API boundaries (stack traces in responses, internal codes exposed).
- **No translation at the boundary** — same domain error becomes 400/422/500 in three different handlers.

### 5. Spot defensive noise

Code that guards against impossible states taxes every reader:

- Null/undefined checks on typed non-nullable values.
- `try` around code that cannot throw.
- Belt-and-suspenders handlers checking the same condition in nested layers.
- Catch blocks that rethrow the same error unchanged — scaffolding around historical fear.

Deletion test: if you removed this guard, what actually breaks? If nothing, flag it.

### 6. Check the observability tax

Errors surfaced but practically unactionable:

- Log lines without context (`"failed to process"` — no IDs, no payload, no cause).
- `logger.error(e.message)` — loses the stack trace.
- Errors not tagged for alerting / not grouped in tracker.
- Metrics counting "errors" in one bucket — can't tell timeout from 4xx from panic.

The error reached the log, but the log can't be acted on. Softer swallow.

### 7. Check retry/timeout/circuit behavior

For every retry, timeout, circuit breaker, fallback:
- Is there a timeout at all? Anywhere external I/O happens without one is a hang waiting to happen.
- Retry budget bounded (max attempts, max total duration)?
- Backoff exists and jittered?
- Operation idempotent? If not, retries are bug factories.
- Anything alert when the circuit opens?
- Are timeouts shorter than the timeouts of upstream callers? If not, upstream gives up first and your retries are wasted work.

## Output

Always a **report** — ranked findings, never edits.

Rank:
1. **Swallowed errors** — potential silent production bugs.
2. **Unsafe fakes / silent fallbacks** — mask incidents.
3. **Boundary failures** — architectural; errors at wrong layer.
4. **Missing retry/timeout safety** — bites when dependency degrades.
5. **Defensive noise** — never a bug; tax on every reader. Cheap, last.

### Report structure

```
# Error-handling audit — <target>

**Verdict:** <one sentence — does this code tell the truth about its failures?>

## What's handled well ✨
<Patterns worth preserving as the convention.>

## Swallowed errors 🔇
<Ranked. For each: location, what error is swallowed, what symptom the
user/operator sees instead, proposed fix (surface / convert / recover-with-intent).>

## Unsafe fakes & silent fallbacks 🎭
<Ranked. For each: location, faked value, what wrong decision the caller makes,
whether anything alerts, proposed fix.>

## Boundary problems 🚧
<Ranked. For each: which boundary, what's leaking, what translation is missing,
where conversion should live.>

## Retry / timeout / circuit gaps ⏱
<Ranked. For each call: missing safety (timeout, bounded retry, backoff,
idempotency, alerting), worst-case incident, proposed fix.>

## Defensive noise 🛡
<Ranked or grouped. Cheap deletions that flatten the happy path.>

## Observability gaps 📉
<Errors surfaced but unactionable. For each: location, what context to add.>

## Where to start
<Numbered: swallows first (one PR per cluster) → unsafe fakes → boundaries
→ retry/timeout → noise. Each step gets a one-line payoff.>
```

Tie every finding to a real location and a concrete worst-case symptom. Errors are part of the interface — audit them with the same seriousness as the happy path.
