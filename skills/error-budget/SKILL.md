---
name: error-budget
description: Map and audit all error paths in code — flag silently swallowed errors, errors caught too far from where they arise, and defensive noise that drowns the happy path. Use when the user wants a failure-mode review of a file, module, service, or whole codebase; when they suspect bugs are being eaten by try/catch blocks, when production failures don't surface in logs, when error handling feels inconsistent, or when they ask to "audit error handling," "find swallowed errors," "review failure paths," or "check our retry/timeout/fallback logic." Trigger whenever code is suspiciously "robust" (lots of try/except with no logging), when on-call sees symptoms but no errors, when fallbacks hide upstream breakage, or before shipping anything that handles external I/O. Produces a ranked report of failure-mode problems — not edits.
---

# Error Budget

Failure paths are part of the interface. In joyful code, error handling is as legible as the happy path; in painful code, errors are swallowed, caught far from where they arise, or so heavily guarded that the happy path drowns in defensive noise.

This skill **diagnoses and reports; it does not edit.** It produces a ranked map of how the target handles failure — what gets surfaced, what gets eaten, what gets retried, what gets faked, and where the boundaries between those choices live.

You are not auditing whether errors are *handled*. You are auditing whether the program tells the truth about them.

## The vocabulary

Use these terms in every finding. Don't drift into "missing error handling" — too vague.

- **Surfaced** — error reaches a place that can act on it: user, operator (logs/alerts), or upstream caller equipped to retry/fallback. The desired default.
- **Swallowed** — error is caught and discarded. No log, no rethrow, no fallback. The bug equivalent of muting an alarm. Almost always wrong.
- **Faked** — error caught and replaced with a fake-success value (empty list, `null`, default object). May be correct, may be hiding a bug; depends entirely on whether the caller can tell the difference.
- **Caught too far** — error is caught many layers above where it arose, losing the context needed to handle it intelligently. The catcher knows *that* something broke, not *what*.
- **Caught too close** — error is caught and rethrown at the same layer that raised it, adding noise without adding decision.
- **Defensive noise** — guards, null-checks, try/catches around code that can't actually fail in the way being guarded. The happy path drowns.
- **Retry / fallback as cover** — retry loops or fallbacks that mask upstream breakage instead of reporting it. The system "self-heals" into a silently degraded state.
- **Boundary** — a layer where errors must be summarized and converted: HTTP handler, queue consumer, CLI entry point, public API. Errors should travel as exceptions/Results *within* a boundary and be translated *at* the boundary.

## The one test that matters

> **For every catch / except / `.catch` / `Result::Err` / `if err != nil`: imagine the caught error is a real production incident at 3am. Does this handler give on-call the information and signal they need to act? If yes, it earns its place. If the error vanishes into the void, becomes an unactionable log line, or quietly degrades the system, it's a failure-mode bug.**

Everything below explains *why* a handler passes or fails that test.

## The audit order (do not reorder)

You're hunting silent bugs and load-bearing fallbacks in the same pass. Order matters: find the swallows first (worst-case-now bugs), then the boundaries (worst-case-future bugs), then the noise.

### 1. Find every catch site and classify

Enumerate every error-handling construct in the target: `try/catch`, `try/except`, `.catch()`, `Result`/`Either` consumers, `if err != nil`, `rescue`, panic recoveries, global handlers, framework error middleware. For each, classify the action taken:

- **Surface** — rethrow, return Err, propagate, log + alert.
- **Convert** — translate to a domain error and rethrow at a boundary.
- **Recover** — handle locally with a sensible value and continue (must be intentional and documented).
- **Swallow** — caught and discarded.
- **Fake** — caught and replaced with default.
- **Log-only** — logged and then swallowed. Usually a swallow in disguise.

A clean codebase is mostly Surface and Convert, with intentional Recover at well-chosen seams. Any concentration of Swallow / Fake / Log-only is a finding.

### 2. Hunt swallowed errors first

These are the highest-leverage findings — every swallow is potentially an invisible production bug. Look for:

- `catch (e) {}` and equivalents — empty handlers.
- `catch (e) { /* ignore */ }`, `pass`, `_ = err`, `if err != nil { return nil }` with no logging.
- `try: ... except Exception: pass` (Python) or bare `except:` clauses.
- `.catch(() => {})` or `.catch(() => null)` in JS/TS promise chains.
- `await x.catch(() => defaultValue)` where `defaultValue` is indistinguishable from a real success.
- Background tasks / timers / goroutines / workers whose errors have no destination — a panic in a goroutine with no recover, an unhandled rejection in a `setInterval` callback, a thrown error in an event listener.
- Promise chains missing a terminal `.catch` or `await` (errors get printed to stderr at best, lost at worst).
- Logger calls that themselves can fail and are wrapped in their own swallow.

For each: name what error is being swallowed, what happens to the system state when it is, how on-call would learn about the resulting symptom.

### 3. Hunt fakes and silent fallbacks

Faking an error with a default value is sometimes correct (`getCachedValueOr(default)`) and sometimes catastrophic (`fetchUserPermissions().catch(() => [])` → user silently denied access OR silently granted, depending on which way the default points).

Ask, for every fake:
- Can the caller distinguish the faked value from a real one? If not, what wrong decision will it make?
- Is the default *safe* (denies access, returns empty, surfaces a clear "no data" state) or *unsafe* (grants access, returns stale data as if fresh)?
- Is the fallback masking an upstream incident? Does anything alert when the fallback fires repeatedly?
- Does the fallback have a budget — does it page someone if it triggers more than N% of the time?

A retry loop with no max attempts, no backoff, and no alerting is a fake-success machine waiting to be a thundering-herd machine.

### 4. Trace where errors travel

Pick the top failure modes you can think of for the target (network down, disk full, DB unreachable, malformed input, auth expired, dependency timeout). For each, trace where the error originates and where it ends up. Findings come from the gap:

- Caught too far — error from `db.query` ends up in a generic 500 handler with no context. The handler can't retry, can't degrade gracefully, can't even tell the user what failed.
- Caught too close — `db.query` wrapped in try/catch that just rethrows the same error with no added context or conversion. Pure noise.
- Boundary-less travel — domain exceptions leak through HTTP/API boundaries unconverted (stack traces in API responses, internal error codes exposed to users).
- No translation at the boundary — every HTTP handler invents its own way to turn an error into a status code; same domain error becomes 400/422/500 in three different places.

### 5. Spot the defensive noise

Code that guards against impossible states pays a tax on every read. Look for:

- Null/undefined checks on values that are typed non-nullable.
- `try` blocks wrapping code that cannot throw (constructors of value types, pure functions over already-validated input).
- "Belt and suspenders" handlers that check the same condition twice in nested layers.
- Defaulted parameters whose defaults are immediately re-validated.
- Catch blocks that re-throw the same error with no transformation — they exist only as scaffolding around historical fear.

The deletion test applies: if you removed this guard, what would actually break? If nothing, flag it.

### 6. Check the observability tax

Errors that are technically surfaced but practically unactionable:

- Log lines without context (`"failed to process"` with no IDs, no payload, no upstream cause).
- `logger.error(e.message)` instead of `logger.error({ err: e, ...context })` — loses the stack trace.
- Errors logged but not tagged for alerting / not grouped in the error tracker.
- Metrics that count "errors" as one bucket — can't tell a timeout from a 4xx from a panic.
- Sentry/Rollbar/etc. capture with stripped breadcrumbs or no user/request context.

The error reached the log, but the log can't be acted on. Treat as a softer swallow.

### 7. Check retry / timeout / circuit behavior

For every retry, timeout, circuit breaker, or fallback path:

- Is there a timeout at all? Anywhere an external call is made without one is a hang waiting to happen.
- Is the retry budget bounded (max attempts, max total duration)?
- Does retry backoff exist, and is it jittered?
- Is the operation idempotent? If not, retries are bug factories.
- Does the circuit breaker have any visibility — does anything alert when it opens?
- Are timeouts shorter than the timeouts of the things calling this code? If not, the upstream gives up first and your retries are wasted work.

## Output

Always a **report** — ranked findings, never edits. Even if the user says "fix the error handling," read it as "tell me what to fix," not "fix." Edits are a separate step they can ask for after reading.

Rank everything so the reader fixes the worst-bug-now first:

1. **Swallowed errors** — every one is a potential silent production bug. Fix first.
2. **Unsafe fakes / silent fallbacks** — mask incidents and degrade silently. Fix next.
3. **Boundary failures** — errors caught at the wrong layer, untranslated leaks. Architectural.
4. **Missing retry/timeout safety** — won't bite until the dependency degrades, but will bite hard.
5. **Defensive noise** — never a bug, but tax on every reader. Cheap, last.

### Report structure

```
# Error-handling audit — <target>

**Verdict:** <one sentence — does this code tell the truth about its failures?>

## What's handled well ✨
<Patterns worth preserving and propagating as the convention.>

## Swallowed errors 🔇
<Ranked. For each: location, what error is being swallowed, what symptom
the user/operator sees instead, proposed fix (surface / convert / recover-with-intent).>

## Unsafe fakes & silent fallbacks 🎭
<Ranked. For each: location, faked value, what wrong decision the caller makes,
whether anything alerts when the fallback fires, proposed fix.>

## Boundary problems 🚧
<Ranked. For each: which boundary, what's leaking through it, what translation
is missing, where the conversion should live.>

## Retry / timeout / circuit gaps ⏱
<Ranked. For each external call or background operation: what safety is missing
(timeout, bounded retry, backoff, idempotency, alerting), what the worst-case
incident looks like, proposed fix.>

## Defensive noise 🛡
<Ranked or grouped. Code that guards against the impossible. Cheap deletions
that flatten the happy path.>

## Observability gaps 📉
<Errors surfaced but unactionable: missing context, stripped traces, ungrouped
buckets. For each: location, what context to add.>

## Where to start
<Numbered path: swallows first (one PR per cluster), then unsafe fakes,
then boundaries, then retry/timeout, then noise. For each step, the one-line payoff.>
```

Tie every finding to a real location and a concrete worst-case symptom. The reader should be able to act on each finding without re-deriving the rationale. Errors are part of the interface — audit them with the same seriousness as the happy path.
