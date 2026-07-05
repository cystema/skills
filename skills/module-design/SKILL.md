---
name: module-design
description: Review the shape of a module, class, or package against Ousterhout/Parnas design principles — depth, information hiding, cohesion, and temporal coupling. Use when the user wants a design review of a module's boundary rather than its line-level style; when an interface feels awkward to call, a class seems to do too much, callers keep reaching into internals, or an API has must-call-X-before-Y ordering traps. Trigger when asked to review "module design," "class responsibilities," "is this abstraction right," "interface depth," "does this leak," "SRP," or whether a component "earns its boundary." Complements spark-joy (which reads line-level) by judging whole-module shape. Produces a ranked report — not edits.
---

# Module Design

A module's job is to take complexity off its callers. It earns its boundary when a caller can use it correctly — including how it fails and in what order to call it — without reading its body. This skill reviews **module shape**: is the interface deep, does it keep its secrets, does it change as one unit, does it hide ordering traps?

**Diagnoses and reports; does not edit.**

## The vocabulary

- **Deep vs shallow** (Ousterhout) — deep = small interface over big implementation; caller gets a lot, must know little. Shallow = interface roughly as complex as what it hides; the caller carries the weight anyway.
- **The interface is everything the caller must know** — not just the signature: invariants, ordering, error modes, required setup. A "clean signature" over three undocumented invariants is shallow.
- **Secret** (Parnas) — what the module conceals so it can change without breaking callers. A leaked secret is a coupling: callers now depend on how it's built.
- **Cohesion** — do the pieces change together? The **split seam** is the line along which responsibilities diverge — different actors, different change cadence.
- **Temporal coupling** — must-call-X-before-Y. An ordering requirement the type system doesn't enforce, so callers learn it by breaking it.
- **Leak** — an implementation detail that escapes into the interface: an internal type returned, an internal exception thrown across the boundary, a mutable internal handed out by reference, config that only makes sense if you know the internals.

## The one test that matters

> **Can the caller use this module correctly — including its failure behavior and call ordering — without reading its body? If they must open the implementation to know what it does, or discover the required call order by breaking it, the module is shallow or leaky.**

## The audit order (do not reorder)

Depth and leaks first (they shape everything callers touch), then cohesion (the split), then ordering traps.

### 1. Measure depth

Compare interface size to implementation size. Shallow flags:
- Pass-through wrappers — methods that only forward arguments to one other call.
- Classes that are bags of getters/setters with no behavior — the caller orchestrates; the class just holds.
- Thin interfaces with one implementation added "for flexibility" — a seam earns its place at the second adapter, not the first.
- A big surface (many methods/params) over a small job — the caller must learn a lot to do a little.

Deep, by contrast: one honest method that absorbs real work. Note what's deep — it's what not to break.

### 2. Find the leaks (information hiding)

For each module, name its secret, then check what escapes:
- Internal types in public signatures (a DB row struct returned from a service method).
- Internal exceptions thrown across the boundary (caller must catch `SqlError` from a domain API).
- Mutable internals returned by reference — caller can mutate your state behind your back.
- Config/flags that only make sense if you know how it's built.

Each leak is a coupling: the thing you wanted to change freely is now pinned by callers.

### 3. Check cohesion

Does everything in the module change together? The split-seam test: if forced to split this in two, where's the line?
- Pieces that change for different reasons (different actors, different cadence) want different homes — SRP violation.
- A `Manager`/`Service`/`Util` that owns three unrelated concerns — name the concerns; each is a candidate module.
- Deletion test: if merged into its one caller, does complexity vanish (it wasn't a real boundary) or spread (it earned its keep)?

### 4. Hunt temporal coupling

Ordering requirements the type system doesn't enforce:
- must-`init`-before-use, must-`open`-before-`read`, must-`close`-after.
- must-set-X-before-Y (set a field, then call a method that assumes it).
- Two calls that must happen together but can be called apart.

One test per trap: **what's the symptom if called in the wrong order?** Silent corruption is worse than a loud error. Prefer designs that make wrong order unrepresentable (builder returns the next type, `open` returns the handle you `read` from) — recommend the shape, don't just warn.

## Output

Always a **report** — ranked findings, never edits.

Rank: shallow interfaces & leaks first (every caller pays), then cohesion splits (structural), then temporal traps (sharp but local).

### Report structure

```
# Module design — <target>

**Verdict:** <one sentence — do these modules take complexity off their callers?>

## What's deep ✨
<Modules with small, honest interfaces over real work. What not to break.>

## Shallow interfaces
<Ranked. For each: the module, why the interface ≈ the implementation, what
the caller is forced to carry, what a deeper interface would absorb.>

## Leaks
<Ranked. For each: the module, its secret, what escapes (type/exception/mutable
ref/config), the coupling it creates, what to hide instead.>

## Weak cohesion
<Ranked. For each: the module, the responsibilities that diverge, the split seam,
proposed homes.>

## Temporal traps
<Ranked. For each: the required order, the symptom when violated, a shape that
makes wrong order unrepresentable.>

## Where to start
<Numbered: deepen/hide first (callers benefit immediately), then split, then
redesign ordering traps.>
```

Tie every finding to a real module and a concrete caller cost. The report's job is to make each module kinder to call: deep, secret-keeping, single-purpose, order-safe.
