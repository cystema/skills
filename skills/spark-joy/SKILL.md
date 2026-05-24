---
name: spark-joy
description: Review code through KonMari (Marie Kondo) principles and report whether it sparks joy. Use when the user wants a diagnostic review of whether code is clean, intuitive, has high glance value, is testable, and isn't shallow; when they ask you to "Marie Kondo," "spark-joy," "tidy-review," or quality-assess a codebase, file, module, or PR; or when they want a felt-sense review that goes beyond a lint pass. Trigger this whenever someone asks whether code "feels right," wants findings on what fails to earn its place, or wants a review oriented around joy, clarity, and keeping only what belongs — even if they don't say "KonMari." Produces a report with ranked findings, not edits.
---

# Spark Joy

Review code the way Marie Kondo tidies a home: keep only what earns its place, give everything a home, discard the rest with gratitude. The goal is code that **sparks joy** — clean, intuitive, instantly legible, testable, deep rather than shallow.

This skill **diagnoses and reports; it does not edit.** It produces findings a human (or an agent acting for them) can read, trust, and act on.

This is a *felt* review, not a lint pass — a linter checks rules; this asks whether someone opening the file feels relief or dread. The question matters because most of a codebase's cost is paid later, by whoever changes it next. "Sparks joy" is shorthand for "kind to the next person." Use the vocabulary below exactly; the shared language lets a reader compare findings across files.

## The vocabulary

Use these five terms in every finding. Don't drift into "code smell," "tech debt," or "clean code" — vague, no felt referent.

- **Sparks joy** — the felt test. Open the code cold: does its shape make the next change obvious, or does your stomach drop? Not prettiness — the confidence that this code will be kind to whoever touches it next.
- **Earns its place** — every module, abstraction, parameter, and dependency must justify existing. Code kept "just in case" does not.
- **A home** — the one obvious, predictable place for a piece of logic. Without a home, the same concern gets handled three ways in three files.
- **Category** — one concern reviewed across the whole codebase at once (all error handling, all data fetching), not file-by-file. This is how you see drift that file-by-file review hides.
- **Thank it and let it go** — before flagging code for removal, name what it was for. Understanding what you're removing means flagging the right things and sparing the load-bearing ones.

## The one test that matters

> **Open the file cold, as if you'd never seen it. Within ten seconds, do you know what it does, where to change it, and what would break if you did? If yes, it sparks joy. If your stomach drops, it doesn't.**

Everything below explains *why* code passes or fails that test. Lead with the felt verdict, then the reason.

## Names carry the most weight

Look at names first — they're the highest-bandwidth thing in the code and the most common reason the ten-second test fails. A truthful name means the reader never opens the body; a name that lies or under-describes forces them in every time. This hits machine-generated code hardest: it leans on placeholder names (`data`, `result`, `handleX`) that describe a value's *shape*, not its *meaning*.

- Name for what a thing *means* in the domain, not how it's built: `pendingInvites`, not `userArray`; `daysUntilExpiry`, not `diff`.
- Promise exactly what the code delivers: `getUser` that also mutates state lies; `check` that returns a value under-describes.
- Booleans/predicates read as questions or assertions (`isExpired`, `hasAccess`) so call sites read like sentences.
- Consistent verbs: if it's `fetch` in one place, it isn't `load`, `get`, and `retrieve` elsewhere for the same operation. Naming drift is meaning drift.

## The KonMari order (do not reorder)

Order matters: review by category, flag removals before structure, never tidy clutter into place. You're not editing — but the sequence sets what to look at first and how to rank findings, so the reader fixes things in an order that doesn't waste effort. No point recommending an abstraction for code that should be deleted; report the deletion first.

### 1. Look by category, not by file

Trace one concern through the whole codebase before forming a verdict — error handling, data fetching, config, date/money formatting, logging. Seeing every instance at once is the only way to catch that it's done five inconsistent ways; that drift is invisible to a file-by-file read. Gather the category into view, *then* assess. Group findings by category too, so the reader sees the pattern you saw.

### 2. Flag clutter before suggesting structure

You can't tidy clutter into joy, and shouldn't recommend organizing it. The highest-leverage thing a review surfaces is what shouldn't exist — deleting it is cheap and erases whole classes of problem. Look for, and report first:

- **The deletion test.** Imagine the module/function/parameter/branch deleted. Does complexity vanish, or reappear across callers? If it vanishes, it was clutter in the costume of structure — flag it. If it reappears, it earned its keep; say so.
- Dead code, commented-out code, unreachable branches, unused exports, `TODO`s older than the feature they describe.
- Speculative generality — config no one sets, hook points with one caller, interfaces with one implementation added "for flexibility." A seam earns its place at the second adapter; one adapter is hypothetical, two is real. Flag the hypotheticals.
- Wrapper functions that only forward arguments. Parameters always passed the same value.
- Comments narrating *what* obvious code does (keep the ones explaining *why*).

### 3. Note what lacks a home

Every concern should have one obvious home. If a new engineer needed to change date formatting, would they know within seconds where to go? If "it depends," it lacks a home — report where it's scattered now and where it should live. A home means:

- One canonical place per concern, named in the project's domain language.
- Predictable neighbors — things that change together live together; things that change for different reasons don't.
- No "miscellaneous" drawers. `utils.ts`, `helpers/`, `common/`, `misc/` are where homeless code piles up; name the actual concerns hiding inside.

### 4. Judge depth, not surface

Shallow code is the opposite of joy: a big interface over a small implementation, so the caller carries complexity the module should absorb. Deep code is a small, honest interface over a substantial implementation that takes real work off the caller.

- **The interface is everything the caller must know** — not just the signature, but invariants, ordering, error modes, required setup. A "clean signature" hiding three undocumented invariants is shallow.
- A module sparks joy when you can use it correctly without reading its body; it fails when you must open the implementation to know what it'll do. Report what leaks onto callers and what a deeper interface would absorb.
- Failure paths are part of the interface — in joyful code, error handling is as legible as the happy path. Flag errors swallowed silently, caught far from where they arise, or so heavily guarded the happy path drowns in defensive noise.

### 5. Thank it before recommending removal

For anything you flag for deletion, name what it was for — the bug it patched, the feature it served — in the finding. This keeps the review honest: it forces you to confirm the code is genuinely idle, not load-bearing in some non-obvious way, before telling someone to delete it. If you *can't* say what it was for, that's its own finding: nobody knows what this does, so the call is "investigate before removing," not "delete."

## The felt qualities

These are the dimensions to judge code against — use them to *form* your verdicts and decide what to report. Weigh naming heavily; most reading friction lives there.

Render them as an explicit ✨ (sparks joy) / 〜 (could go either way) / 🗑 (doesn't) scorecard **only for multi-file or whole-codebase reviews**, where an at-a-glance grid earns its place by letting the reader compare across modules. For a single file or function, skip the grid — fold these judgments into prose findings instead, since a one-target scorecard is more ceremony than signal. Either way, every verdict leads with the felt sense and says why in one sentence.

| Quality | The question | Fails when |
|---|---|---|
| **Naming** | Does each name tell the truth, so you never need the body to know? | `data2`, `handleThing`, `check()`; a name describing the implementation (`userArray`) not the meaning (`pendingInvites`); under-promising what the code does. |
| **Glance value** | Can you grasp it in ten seconds without holding other files in your head? | You must open three more files, or scroll past 40 lines of setup, before the point arrives. |
| **Narrative order** | Does it read top-to-bottom like prose — intent first, details below? | Real logic buried under a wall of guards; helpers defined far above their use; you have to read bottom-up to follow it. |
| **Intuitiveness** | Does it behave as its name promises, at one altitude of abstraction? | `getUser` also writes the cache and sends mail; one function mixes "validate email" with "increment byte offset"; five levels of nesting where early returns would flatten it. |
| **Honesty / depth** | Does the interface tell the whole truth, including how it fails? | Hidden invariants, surprise side effects, lies of omission; errors swallowed or handled far away; happy path buried in defensive noise. |
| **Testability** | Can you test it without elaborate scaffolding? | Testing one behavior means mocking the universe. The seam is in the wrong place. |
| **Earned existence** | Does every part justify being here? | Abstractions, options, indirection, comments restating code — things nothing uses. |
| **A home & consistency** | Does each concern have one obvious place, handled the same way everywhere? | The same logic, slightly different, in three files; each module inventing its own conventions so you re-learn the patterns every time. |

## Output

Always a **report** — ranked findings, never edits. Even if the user says "tidy" or "make it spark joy," read it as "tell me what to change," not "change it." Performing edits is a separate step they can ask for after reading the findings.

Rank everything so the reader knows where to start:

- Across the report, follow the KonMari order: removals first (cheapest, highest-leverage), then homes, then depth. A reader fixing top-to-bottom never polishes something a later finding says to delete.
- Within each section, order by leverage — most confusion or risk removed per unit of effort goes first.
- Make each suggestion actionable without being an edit: name the location, the change, the payoff. The reader should act on it without re-deriving your reasoning.

### Report structure

Use this shape. Keep the verdict, "what sparks joy," and "where to start" always; among the middle sections, drop any that would be empty and fold one into another where the findings genuinely overlap (a naming problem that's also a homeless concern needn't be reported twice). Don't pad to fill every heading.

```
# Does it spark joy? — <target>

**Verdict:** <one felt sentence — relief or dread on opening this?>

## What sparks joy ✨
<What's genuinely good and why. Always include — honors what works and tells
the reader what not to break while fixing the rest.>

## What to thank and let go 🗑
<Ranked. Each: what it is, what it was for (thank it), why it goes, where it lives.
Biggest, cheapest wins first.>

## What lacks a home 〜
<Ranked. Scattered concerns; for each, where it's spread now and the one place
it should live.>

## What's shallow
<Ranked. Interfaces hiding too little or lying by omission; for each, what leaks
onto callers and what a deeper interface would absorb.>

## What's hard to read 〜
<Ranked. Line-level friction not covered above: misleading/generic names, logic
buried under setup, mixed altitudes, deep nesting. For each, the felt cost and
the lighter shape it wants.>

## Where to start
<Short numbered path through the findings in KonMari order (remove → home →
deepen), so the reader has an obvious first move.>
```

Tie every finding to a real location. Lead with the felt sense, then the reason — that pairing is what makes the report trustworthy to a human and actionable for an agent. Be specific and kind: the aim is code someone is glad to open, and a review someone is glad to receive.
