---
name: naming-audit
description: Audit names across a codebase as one concern — flag drift, lies, and under-description. Use when the user wants a naming pass over a file, module, or whole codebase; when they suspect the same operation is called `fetch`/`load`/`get`/`retrieve` in different places; when they ask to "rename things," "make names consistent," "find naming drift," or "tell me what's poorly named." Trigger whenever code reads as confusing despite being correct, when grep-ability is suffering, when machine-generated placeholder names (`data`, `result`, `handleX`) leak into committed code, or when reviewing whether the domain vocabulary matches the code vocabulary. Produces a ranked report of naming problems with proposed names — not edits.
---

# Naming Audit

Names are the highest-bandwidth thing in code. A truthful name means the reader never opens the body; a lying or under-describing name forces them in every time. This skill audits **names as a category** across the target and reports drift, lies, and weak descriptions.

**Diagnoses and reports; does not rename.**

## The vocabulary

Use these in every finding:

- **Lies** — name promises one thing, code does another. `getUser` that mutates. `isReady` that initializes.
- **Under-describes** — technically true, weaker than the code. `check()` that returns. `handle`/`process`/`do` for anything.
- **Shape-named** — describes type/container, not meaning. `userArray`, `dataMap`, `responseObject`.
- **Drift** — same operation, different verbs across the codebase (`fetch`/`load`/`get`). Or same verb for different operations.
- **Domain vs. implementation** — name reflects how built (`tokenString`) instead of what it means (`bearerToken`).

## The one test that matters

> **Read the name without opening the body. Can you predict what it does, returns, and side-effects? If you'd guess wrong, the name is the bug.**

## The audit order (do not reorder)

Drift across the codebase outranks any single ugly name. Fix conventions before polishing locals.

### 1. Survey the verbs

Before reading any single name, list every verb used per operation. Group by operation, not by location. Drift jumps out when `fetchUser`/`loadUser`/`getUser`/`retrieveUser` sit side by side. Pick the canonical verb per operation; flag deviations.

Common drift axes — check explicitly:
- read-from-source: `fetch` / `load` / `get` / `retrieve` / `read`
- write-to-source: `save` / `store` / `persist` / `write` / `commit`
- instantiate: `make` / `create` / `build` / `new`
- remove: `delete` / `remove` / `destroy` / `purge`
- mutate: `update` / `patch` / `modify` / `set`
- predicate-with-effect: `validate` / `check` / `verify` / `ensure`
- `handle` / `process` / `do` / `run` — almost always under-describes; flag wholesale

### 2. Check nouns against the domain

Open CONTEXT.md, glossary, ADRs, or the product surface. Compare to nouns in code. Mismatches:
- Code says `account`, product says `workspace` — pick one.
- Obsolete names from renamed features — flag for migration.
- Invented nouns the domain doesn't have (`Manager`, `Helper`, `Service` with no concrete referent) — either a real concept hides inside, or the abstraction doesn't earn its place.

### 3. Flag lies

Lies actively mislead; worse than ugly. For each, capture promise vs delivery:

- **Side-effect lies.** `getX` that writes. `isX` that mutates. Pure-looking name with hidden I/O.
- **Return lies.** `check` that returns nothing useful. `validate` that returns the parsed value instead of bool.
- **Scope lies.** `sendEmail` that also logs and increments a counter — three commits in one name.
- **Tense / mood lies.** `isExpired` that triggers expiration. `willRetry` that retries now. Predicates must be pure-read.

### 4. Flag under-description and shape-naming

Propose a stronger domain name for each:

- `data`, `result`, `value`, `item`, `thing`, `obj` — never earns its place outside short loop vars.
- `userArray` → `pendingInvites`. Strip the type, name the meaning.
- `handle*` whose body is more than a few lines — real verb hiding inside.
- `Manager` / `Helper` / `Util` / `Service` — ask what concrete responsibility lives there. Often two responsibilities masquerading as one.

### 5. Predicates and booleans

Call sites should read as sentences:
- Predicates as questions or assertions: `isExpired`, `hasAccess`, `canPublish`, `shouldRetry`.
- Prefer `isVisible` over `isNotHidden`. Avoid double negatives.
- Boolean params are smells; if you must name one, name it for the *true* meaning (`force: true`, not `safeMode: false`).

### 6. Check the boring places

Drift hides where no one re-reads:
- File/dir names: same concern in `utils/auth.ts` and `helpers/authentication.ts`?
- Test names: behavior (`returns null when user not found`) or restated symbol (`testGetUser`)?
- Error messages and log lines — if code says `invitation` and the error says `request`, the user pays for the drift.

## Output

Always a **report** — ranked findings, never renames.

Rank: drift (codebase-wide) → lies (mislead) → domain mismatches → under-description (local, cheap, high volume).

For each finding: location(s), current name, promise vs delivery (or drift partners), proposed name, one-line payoff.

### Report structure

```
# Naming audit — <target>

**Verdict:** <one sentence — does the codebase's vocabulary tell the truth?>

## What's named well ✨
<Names worth preserving and using as the convention.>

## Drift across the codebase 🌀
<Ranked by operation. For each: verbs/nouns in use, where each appears,
canonical choice, sites to migrate.>

## Lies 🚨
<Ranked. For each: name, what it promises, what it does, location,
proposed name (or proposed behavior change — sometimes the code is wrong).>

## Domain mismatches
<Ranked. For each: code name, domain name, where each lives, proposed unification.>

## Under-described / shape-named
<Ranked or grouped. Current → proposed, one-line rationale. Group large
classes (e.g., "12 vars named `data` — see table").>

## Where to start
<Numbered path: drift first (one PR per operation) → lies (highest-traffic first)
→ domain alignment → local cleanup.>
```

Tie every finding to a real location and a concrete proposed name.
