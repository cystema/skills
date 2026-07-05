---
name: state-shape
description: Review a data model for shapes that let bad states exist — illegal states representable, primitive obsession, denormalized invariants that can drift, unexpected mutation, and value-versus-identity confusion. Use when the user wants a review of a type, struct, schema, or state container rather than control flow; when a bug traces back to "the data was in a state that shouldn't be possible," when null/optional handling is tangled, when the same fact lives in two places and disagrees, or when shared mutable state causes spooky action at a distance. Trigger when asked to review "data model," "state shape," "type design," "make illegal states unrepresentable," "why is this null," or shared-mutable/identity bugs. Produces a ranked report — not edits.
---

# State Shape

Most stubborn bugs are a data model letting a state exist that never should. This skill reviews **the shape of state**: can the type represent nonsense, does the same fact drift between two homes, does something mutate under you, is a value confused with an identity? Fix the shape and whole classes of bug become unrepresentable.

**Diagnoses and reports; does not edit.**

## The vocabulary

- **Illegal state representable** — the model permits combinations that can never be valid: `loading && error && data` all set at once; a status enum plus three booleans that can contradict it. If the shape allows it, someone eventually creates it.
- **Primitive obsession** — a domain concept modeled as a raw `string`/`int`/`bool` (`userId: string`, `cents: int`), so nothing stops you passing the wrong one. The type says `string`; the meaning is lost.
- **Drifting invariant** — the same fact in two places that must agree, with nothing enforcing it: parallel arrays, a denormalized copy, a cached derived value that can fall out of sync with its source.
- **Mutation surface** — what mutates, who can mutate it, and whether it's shared across threads/requests. Wide surface = spooky action at a distance.
- **Value vs identity** (Hickey) — a **value** is an immutable fact (`{x:1,y:2}`); an **identity** is a thing that changes over time (an account, referenced by id). Confusing them yields shared mutable references and "snapshots" that are really live pointers.

## The one test that matters

> **Name the nastiest invalid state you can imagine for this data — two fields that contradict, a null that should be impossible, a reference someone mutates behind your back. Can the shape represent it? If yes, it will occur. The fix is a shape where it can't.**

## The audit order (do not reorder)

Illegal states first (they cause the impossible-bug reports), then primitive obsession and drift (they let bad data in and spread), then mutation and identity (the spooky ones).

### 1. Hunt illegal states

Enumerate the invalid combinations the shape permits:
- Booleans that co-vary and should be one enum: `isLoading`/`isError`/`isSuccess` → a single `state: Loading | Error(e) | Success(data)` sum type.
- Optionals that are only valid together (`startDate` set iff `endDate` set) → group them so "one without the other" can't be built.
- Enum-plus-flags that can contradict (status = `shipped` but `deliveredAt` null and `cancelled` true).

For each: the illegal combo, the bug it causes, the sum type / tagged union / narrower shape that forbids it.

### 2. Find primitive obsession

Domain concepts modeled as raw primitives:
- `string` ids interchangeable across entities (pass a `userId` where an `orderId` goes — compiles fine).
- Money as `float`/`int` with no currency, no rounding contract.
- Stringly-typed enums (`status: string` accepting any string).

For each: the concept, the raw type, the wrong value that type-checks today, the domain type (newtype/branded type/value object) that would catch it.

### 3. Find drifting invariants

Same fact in two places:
- Parallel arrays indexed in lockstep (`names[i]` / `ages[i]`) — one reorder desyncs them.
- Denormalized copies that must be updated together.
- Derived/cached values that can drift from their source (a `total` field alongside the `items` it should be computed from).

For each: the duplicated fact, how they desync, the single-source-of-truth shape (store one, derive the rest).

### 4. Map the mutation surface

What mutates that shouldn't, and what's shared:
- Shared mutable defaults (Python `def f(x=[])`, a shared config object handed to many owners).
- Module-level / global mutable state read and written from many places.
- A mutable object passed to multiple owners who each assume they own it.
- "Looks pure but isn't" — a function that reads clean but mutates an argument or captured state.

For each: what mutates, who's surprised, whether immutability or a defensive copy or clearer ownership fixes it.

### 5. Check value/identity confusion

- Mutable "value" objects that get shared and then mutated — callers holding a stale or surprise-changed reference.
- A "snapshot" that's actually a reference to live state (you saved it to compare later; it changed under you).
- Caches keyed by an identity whose contents change, or equality-by-reference where you meant equality-by-value.

For each: the confusion, the bug it enables, whether the thing wants to be an immutable value or an explicit identity with controlled updates.

## Output

Always a **report** — ranked findings, never edits.

Rank: illegal states first (highest-leverage — a shape fix kills a bug class), then primitive obsession & drift, then mutation & identity.

### Report structure

```
# State shape — <target>

**Verdict:** <one sentence — can this data represent states that should be impossible?>

## What's well-shaped ✨
<Types that make illegal states unrepresentable. The pattern to copy.>

## Illegal states representable
<Ranked. For each: the invalid combination, the bug it causes, the sum type /
narrower shape that forbids it.>

## Primitive obsession
<Ranked or grouped. For each: concept, raw type, wrong value that type-checks,
proposed domain type.>

## Drifting invariants
<Ranked. For each: the duplicated fact, how it desyncs, the single-source shape.>

## Mutation hazards
<Ranked. For each: what mutates, who's surprised, the fix (immutability /
ownership / defensive copy).>

## Value–identity confusion
<Ranked. For each: the confusion, the bug it enables, value-or-identity fix.>

## Where to start
<Numbered: illegal-state shapes first (kills bug classes), then domain types,
then single-source-of-truth, then mutation/identity.>
```

Tie every finding to a real type and a concrete bad state. The goal is a data model where the invalid states you fear simply cannot be written down.
