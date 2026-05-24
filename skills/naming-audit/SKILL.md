---
name: naming-audit
description: Audit names across a codebase as one concern — flag drift, lies, and under-description. Use when the user wants a naming pass over a file, module, or whole codebase; when they suspect the same operation is called `fetch`/`load`/`get`/`retrieve` in different places; when they ask to "rename things," "make names consistent," "find naming drift," or "tell me what's poorly named." Trigger whenever code reads as confusing despite being correct, when grep-ability is suffering, when machine-generated placeholder names (`data`, `result`, `handleX`) leak into committed code, or when reviewing whether the domain vocabulary matches the code vocabulary. Produces a ranked report of naming problems with proposed names — not edits.
---

# Naming Audit

Names are the highest-bandwidth thing in code. A truthful name means the reader never opens the body; a lying or under-describing name forces them in every time. This skill reviews **names as a category** across the whole target — file, module, or codebase — and reports drift, lies, and weak descriptions.

This skill **diagnoses and reports; it does not rename.** It produces findings a human (or an agent acting for them) can read, trust, and apply.

A name isn't a label — it's a contract. The audit asks: does each name tell the truth about what the thing means, does it, and returns? And across the codebase: does the same idea wear the same name everywhere?

## The vocabulary

Use these terms in every finding. Don't drift into "bad name" or "unclear" — vague, no actionable shape.

- **Lies** — the name promises one thing, the code does another. `getUser` that also mutates state. `validate` that throws. `isReady` that triggers initialization.
- **Under-describes** — the name is technically true but tells the reader less than the code does. `check()` that returns a value. `process` for a specific transformation. `handle` for anything event-shaped.
- **Shape-named** — the name describes the value's *type* or *container*, not its *meaning*. `userArray`, `dataMap`, `responseObject`. Machine-generated code leans on these.
- **Drift** — same operation, different verbs in different places: `fetch` here, `load` there, `get` over there, `retrieve` in a fourth. Or same word for different operations: `update` meaning four unrelated things.
- **Altitude mismatch** — the name lives at the wrong level of abstraction for its caller. A domain-layer function named `incrementCounter` when it really `recordPurchase`s. An infra-layer thing named in business terms.
- **Domain vs. implementation** — the name reflects how the code is built (`userArray`, `tokenString`) instead of what it means in the product (`pendingInvites`, `bearerToken`).

## The one test that matters

> **Read the name without opening the body. Can you predict what the function/value does, returns, and side-effects? If yes, the name earns its place. If you'd guess wrong, or have to open the body to know, the name is the bug.**

Everything below explains *why* a name passes or fails that test.

## The audit order (do not reorder)

Sequence matters. Drift across the codebase outranks any single ugly name — fixing local names while leaving inconsistent verbs in place is polishing the surface of a deeper problem.

### 1. Survey the verbs

Before reading any single name carefully, list every verb used for create / read / update / delete / fetch-from-remote / persist / transform. Group by operation, not by location. Drift jumps out the moment you see `fetchUser`, `loadUser`, `getUser`, `retrieveUser` next to each other. Decide the canonical verb per operation; flag every deviation.

Common drift axes to check explicitly:
- `fetch` / `load` / `get` / `retrieve` / `read` for read-from-source
- `save` / `store` / `persist` / `write` / `commit` for write-to-source
- `make` / `create` / `build` / `new` / `construct` for instantiation
- `delete` / `remove` / `destroy` / `purge` / `clear` for removal
- `update` / `patch` / `modify` / `set` / `apply` for in-place change
- `validate` / `check` / `verify` / `ensure` / `assert` for predicates-with-side-effect
- `handle` / `process` / `do` / `run` — almost always under-describes; flag wholesale

### 2. Check the nouns against the domain

Open the project's domain vocabulary (CONTEXT.md, glossary, ADRs, or the product surface itself). Compare to the nouns in the code. Mismatches:

- Code says `account`, product says `workspace`. Pick one, flag the other.
- Code uses an obsolete name from a renamed feature. Thank it, flag for migration.
- Code invents a noun the domain doesn't have (`manager`, `helper`, `wrapper`, `service` with no concrete referent). Flag — either there's a real concept hiding inside, or the abstraction itself doesn't earn its place.

### 3. Flag lies

A lying name is worse than an ugly one — ugly slows the reader, lying actively misleads. For each lie, capture what the name promises vs what the code actually does. Categories:

- **Side-effect lies.** `getX` that writes. `isX` that mutates. Pure-looking name with hidden I/O.
- **Return lies.** `check` that returns nothing useful (under-describes) or returns something surprising. `validate` that returns the parsed value instead of just a bool.
- **Scope lies.** Function named for a narrow operation that does three more things. `sendEmail` that also logs an event and updates a counter — three commits in one name.
- **Tense / mood lies.** `isExpired` that triggers expiration. `willRetry` that retries immediately. Predicates must be pure-read.

### 4. Flag under-description and shape-naming

For each name that's technically true but weak, propose a stronger one in the project's domain vocabulary.

- `data`, `result`, `value`, `item`, `thing`, `obj` — never earns its place at the variable level (function-internal short-lived loop vars excepted; flag everything else).
- `userArray`, `tokenMap`, `responseObject` — strip the type, name the meaning: `users` → `pendingInvites`, `tokenMap` → `tokensBySession`.
- `handleClick`, `handleSubmit`, `handleChange` — passable for trivial delegation, but flag any `handle*` whose body is more than a few lines: there's a real verb hiding inside.
- `Manager`, `Helper`, `Util`, `Service` suffixes — flag and ask what concrete responsibility the class actually owns. Often two responsibilities masquerading as one.

### 5. Check predicates and booleans

Booleans and predicates have their own rules — call sites read as sentences only if the names are shaped right.

- Predicates read as questions or assertions: `isExpired`, `hasAccess`, `canPublish`, `shouldRetry`. Not `expired` alone (ambiguous with verb), not `accessCheck` (noun phrase, reads wrong at call site).
- Negation: prefer `isVisible` over `isNotHidden`. Avoid double negatives at call sites.
- Boolean flags as function params are smells; if you must name one, name it for the *true* meaning (`force: true`, not `safeMode: false`).

### 6. Check the boring places

Drift hides in places no one re-reads:

- File and directory names: same concern split across `utils/auth.ts` and `helpers/authentication.ts`?
- Test names: do they describe behavior (`returns null when user not found`) or just restate the function name (`testGetUser`)?
- Commit message / PR title nouns: do they match the code's vocabulary?
- Error messages and log lines — the user-facing names. If the code says `invitation` but the error says `request`, the user pays for the drift.

## Output

Always a **report** — ranked findings, never renames. Even if the user says "fix the names," read it as "tell me what to rename," not "rename." Renaming is a separate step they can ask for after reading the findings.

Rank by leverage:
- Drift across the codebase first — fixing one site doesn't help; fixing the convention does.
- Lies second — they actively mislead.
- Domain mismatches third — they widen the gap between code and product.
- Under-description and shape-naming last — local, cheap, but high volume.

For each finding, give: location(s), current name, what it promises vs delivers (or what it drifts with), proposed name, one-line payoff.

### Report structure

```
# Naming audit — <target>

**Verdict:** <one sentence — does the codebase's vocabulary tell the truth?>

## What's named well ✨
<What's genuinely good and why. Always include — names worth preserving and using
as the convention for the rest.>

## Drift across the codebase 🌀
<Ranked. For each operation: the verbs/nouns in use, where each appears,
the canonical choice, sites to migrate. Group by operation, not by file.>

## Lies 🚨
<Ranked. For each: name, what it promises, what it does, location, proposed
name OR proposed behavior change (sometimes the code is wrong, not the name).>

## Domain mismatches
<Ranked. For each: code name, domain name, where each lives, why they diverged
if known, proposed unification.>

## Under-described / shape-named
<Ranked or grouped. Current → proposed, with one-line rationale. Group large
classes of the same fix (e.g., "12 variables named `data` — see table").>

## Where to start
<Short numbered path: drift fixes first (one PR per operation), then lies
(highest-traffic first), then domain alignment, then local cleanup.>
```

Tie every finding to a real location and a concrete proposed name. The reader should be able to act on each finding without re-deriving the rationale. Be specific and kind: a good naming audit makes the codebase read like prose to whoever opens it next.
