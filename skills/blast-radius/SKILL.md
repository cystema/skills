---
name: blast-radius
description: Map what breaks if a function, module, migration, config change, or deploy ships wrong. Produce a ranked surface of callers, hidden coupling, irreversibility, and recovery cost. Use when the user is about to make a change and wants to understand its impact, when reviewing a PR for unintended reach, when planning a migration or refactor, when asked to "scope this change," "find everywhere this is used," "audit the impact," or "tell me what depends on this." Trigger before risky changes (renames, schema migrations, shared-library bumps, removed public APIs, config/env changes, infra moves) and whenever someone says "this is a small change" about code that isn't obviously local. Produces a ranked impact map and risk report — not edits.
---

# Blast Radius

Every change has a radius. Most engineering pain comes from a change whose true radius was larger than the author thought. This skill **maps the radius before the change ships** — direct callers, hidden coupling, broken invariants, and irreversibility.

A change is "small" only if its blast radius is small.

**Diagnoses and reports; does not change code.**

## The vocabulary

- **Direct surface** — callsites, importers, subscribers, consumers that name the symbol. Greppable.
- **Indirect surface** — code that depends on *behavior* without naming the symbol: serialized payloads, DB columns, log formats, contracts with other services, cached values, generated code.
- **Hidden coupling** — dependents you can't grep for: dynamic dispatch, reflection, string-keyed lookups, framework auto-wiring, config references, downstream consumers in other repos, dashboards/runbooks.
- **Invariant** — something callers assume that the change might break: ordering, idempotency, defaults, error types, performance, atomicity.
- **Boundary** — place the change crosses (process, service, repo, deploy unit, persisted-data layer, public API, cache, queue). Each multiplies risk because the two sides deploy independently.
- **Reversibility** — cost to undo if wrong: minutes / hours / days / weeks. Schema deletions and serialized-format breaks are weeks; flag flips are minutes.

## The one test that matters

> **Ship the change at 17:00 Friday. At 03:00 Saturday, the worst plausible thing happened. What's the symptom, who notices, what restores the system, how long does it take? If the answer is "revert and we're fine within minutes," radius is small. If it involves backfills, schema rollbacks, coordinating with other teams, or apologizing to users, radius is large — regardless of how small the diff looks.**

## The audit order (do not reorder)

Map the surface before judging risk. A risk verdict on a partial surface gives false confidence.

### 1. Map the direct surface

Find everything that names the symbol. One missed callsite = production bug.

- Callsites of the function/method.
- Imports of the module.
- References to the type/interface/trait.
- Subscribers/handlers/consumers of the event/topic.
- Routes/endpoints that hit this code.
- Tests that exercise it (surface — they pin behavior).

Tools: grep / ripgrep / find-usages / language server. Across repos: search the org. Services: search consumer repos and gateway configs.

Report: count, list, group by purpose (test vs production vs tooling).

### 2. Map the indirect surface

Dependents that don't name the symbol but depend on its behavior:

- **Persisted shape.** DB column, file format, cache key, queue message, log line. Anyone who reads that shape is a dependent — including humans reading logs, ETL jobs, BI dashboards.
- **Wire shape.** API responses, RPC payloads, webhook bodies. External consumers are dependents you can't see.
- **Configuration surface.** Env vars, feature flags, runtime config. Anything that reads these.
- **Generated artifacts.** Schema-generated code, client SDKs, API docs. Regenerate or downstream breaks.

### 3. Hunt hidden coupling

Things you can't grep for. The dangerous ones:

- **Dynamic dispatch.** Reflection, `getattr`, dynamic imports, registry-pattern lookups by string key. Search the string form, not just the symbol.
- **Framework magic.** ORM hooks, decorators, DI by type, Spring/Nest auto-wiring, Django signals, Rails callbacks.
- **Config-driven references.** YAML/JSON/env naming the thing as a string. Code change doesn't update config; deployment finds out.
- **Cross-repo consumers.** Internal SDK clients, downstream services, public clients. If you can't search, ask.
- **Out-of-band consumers.** Grafana panels, alerting rules, runbooks, support macros. Renaming a metric breaks the dashboard silently.
- **Cached/serialized values.** If the shape changes, in-flight cached values become poison after deploy. Includes browser localStorage, CDN.

For each suspected coupling: how to verify, what breakage looks like.

### 4. List invariants the change might break

A change can preserve every callsite and still break callers — by breaking assumptions:

- **Ordering / stability.** Sort order, iteration order, event order, retry idempotency.
- **Type / shape.** Nullable ↔ non-nullable. Optional becoming required.
- **Error contract.** Same error type? Same conditions? A code path that previously threw and now returns `null` silently breaks every catch block.
- **Performance.** Hot loop? O(1) → O(n)? Now does I/O? Now blocks where it was async?
- **Concurrency / atomicity.** Was it safe concurrently? Did it hold a lock?
- **Defaults.** Default args, default config values, default SQL column values. Silently changes behavior for callers that didn't override.
- **Side effects.** Previously read-only thing now writes? Previously local thing now hits the network?

### 5. Map boundaries crossed

Each boundary multiplies risk because the two sides deploy independently and may run mixed versions for some window.

For each boundary: can the two sides deploy in either order? If not, you have a deployment-ordering hazard and need expand → migrate → contract.

Boundaries to check: process, service, repo, deploy-unit (frontend ↔ backend), persisted-data (writer ↔ data ↔ reader), cache, public API.

### 6. Score reversibility

- **Minutes** — flag flip, config rollback, code revert with no persistent side effects.
- **Hours** — code revert + cache invalidation, rolling restart, republish generated artifact.
- **Days** — data backfill, reverse schema migration, coordinated multi-service deploy.
- **Weeks / never** — destroyed data, public API breakage seen by external clients, secrets rotated, unreachable consumers of changed serialized format.

A small diff with one-way consequences deserves more scrutiny than a large diff with one-button revert. **Highlight one-way doors explicitly.**

### 7. Construct the 3am scenario

Synthesize a concrete failure narrative. Be specific:
- First symptom and where it appears (alert, customer report, silent degradation noticed days later).
- Who's paged, with what context.
- Immediate mitigation (revert, flag-off, scale-out).
- Full recovery (data fixup, customer comms, post-incident review).
- Time to mitigation and full recovery.

If you can't construct this scenario, the audit isn't done.

## Output

Always a **report** — ranked findings, never edits.

Rank by what shapes the rollout plan:

1. **Hidden / out-of-org dependents** — can't fix what you can't see.
2. **One-way doors** — irreversibility is the dominant risk factor.
3. **Boundary / deployment-ordering hazards** — dictate rollout shape.
4. **Broken invariants** — drive whether callers need lockstep updates.
5. **Direct surface volume** — drives one-PR vs campaign.
6. **Dead surface to shrink first** — delete dead callsites before the real change.

### Report structure

```
# Blast radius — <change>

**Verdict:** <one sentence — small/medium/large radius, dominant risk factor.>

## The 3am scenario 🌙
<Concrete failure narrative: symptom, who pages, mitigation, full recovery,
time-to-recovery. Leads — grounds everything below.>

## Direct surface
<Count and grouped list. Distinguish test/production/tooling. Path-line every site.>

## Indirect surface
<Persisted shapes, wire shapes, config, generated artifacts. For each: who reads
this shape, what breaks if it changes.>

## Hidden coupling ⚠
<Ranked. For each suspected dependency: how to verify, breakage symptom.
Out-of-org consumers first.>

## Broken invariants
<Ranked. For each: what callers assume, how breakage manifests, which callers
depend on it.>

## Boundaries crossed
<For each: deployment-ordering constraint, mixed-version compatibility during
deploy window, required rollout shape (atomic vs phased).>

## Reversibility map
<For each part: tier (minutes/hours/days/never), what recovery requires.
Highlight one-way doors.>

## Recommended rollout
<Numbered. Shape that falls out of the findings: e.g., "1. Delete dead callsites
X, Y (shrinks radius). 2. Add new field alongside old. 3. Migrate readers behind
flag. 4. Remove old field." Or "ship atomically — radius small." Be specific.>
```

Tie every finding to a real location or a concrete dependent. The report's job is to make the author choose a rollout shape that matches the actual radius. A small diff with a large radius is the most dangerous thing a codebase ships.
