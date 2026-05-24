---
name: blast-radius
description: Map what breaks if a function, module, migration, config change, or deploy ships wrong. Produce a ranked surface of callers, hidden coupling, irreversibility, and recovery cost. Use when the user is about to make a change and wants to understand its impact, when reviewing a PR for unintended reach, when planning a migration or refactor, when asked to "scope this change," "find everywhere this is used," "audit the impact," or "tell me what depends on this." Trigger before risky changes (renames, schema migrations, shared-library bumps, removed public APIs, config/env changes, infra moves) and whenever someone says "this is a small change" about code that isn't obviously local. Produces a ranked impact map and risk report — not edits.
---

# Blast Radius

Every change has a radius. Most engineering pain comes from a change whose true radius was larger than the author thought. This skill **maps the radius before the change ships** — every direct caller, every hidden coupling, every assumption that depends on the current behavior, and every place that won't easily revert if the change turns out wrong.

This skill **diagnoses and reports; it does not change code.** It produces a map and a risk-ranked impact report a human (or an agent acting for them) can read before deciding *how* to ship the change — atomically, behind a flag, in stages, or not at all.

A change is "small" only if its blast radius is small. Audit the radius before believing the change is small.

## The vocabulary

Use these terms in every finding. Don't drift into "this affects a lot of code" — vague.

- **Direct surface** — every callsite, importer, subscriber, or consumer that names the thing being changed by symbol. Greppable.
- **Indirect surface** — code that depends on the thing's *behavior* without naming the symbol: serialized payloads, database columns shaped by it, log formats parsed downstream, contracts with other services, cached values, generated code, schemas.
- **Hidden coupling** — dependents you can't grep for: dynamic dispatch, reflection, string-keyed lookups, framework auto-wiring, config-driven references, downstream consumers in another repo, dashboards/queries that key on the field, runbooks that mention it.
- **Invariant** — something callers assume that the change might break: ordering, idempotency, sort stability, default values, error types thrown, performance characteristics, atomicity.
- **Boundary** — a place the change crosses (process, service, repo, deployment unit, persisted-data layer, public API, cache, queue). Each boundary multiplies risk because the two sides deploy independently.
- **Reversibility** — cost to undo if wrong, measured in minutes / hours / days / weeks. Schema deletions and serialized-format breaks are weeks; flag flips are minutes.
- **Recovery posture** — what restores the system if the change is bad: revert, flag-off, migration rollback, manual cleanup, data backfill, restore-from-backup, public apology.

## The one test that matters

> **Imagine you ship this change at 17:00 on Friday. At 03:00 Saturday, the worst plausible thing has happened. What is the symptom, who notices, what restores the system, and how long does it take? If the answer is "revert and we're fine within minutes," the radius is small. If it involves backfills, schema rollbacks, coordinating with other teams, or apologizing to users, the radius is large — regardless of how small the diff looks.**

Everything below explains how to derive that answer concretely.

## The audit order (do not reorder)

Map the surface before judging risk. A risk verdict on a partial surface is worse than no verdict — it gives false confidence. Direct surface first (cheap), then indirect (harder), then hidden (requires guessing), then invariants (requires thinking), then reversibility (requires honesty).

### 1. Map the direct surface

Find everything that names the symbol. Be thorough; one missed callsite is a production bug.

- All callsites of the function/method.
- All imports of the module.
- All references to the type / interface / trait.
- All subscribers / handlers / consumers of the event or topic.
- All routes / endpoints that hit this code.
- All test files that exercise it (counts as surface — they tell you what behavior is pinned).

Tools: grep / ripgrep / IDE find-usages / language server / `git grep` across the repo. Across repos: search the org. For services: search consumer repos and gateway configs.

Report: count, list, and group by purpose (test vs. production vs. tooling).

### 2. Map the indirect surface

These are dependents that don't name the symbol but depend on its behavior:

- **Persisted shape.** Does this code write to a database column, file format, cache key, queue message, log line? Anyone who reads that shape is a dependent — including humans reading logs, ETL jobs, downstream analytics, BI dashboards.
- **Wire shape.** Public API responses, internal RPC payloads, webhook bodies. Each consumer is a dependent; consumers in other orgs/teams are dependents you can't see.
- **Configuration surface.** Env vars, feature flags, runtime config files. Anything that reads these is a dependent.
- **Generated artifacts.** Code generated from a schema, client SDKs, API docs. Regenerate and re-publish, or downstream breaks.
- **Filesystem / process contracts.** PID files, lock files, temp dirs, signals. Operators are dependents.

Report: which categories apply, which specific consumers per category, which you couldn't enumerate.

### 3. Hunt hidden coupling

Things you can't grep for. The dangerous ones:

- **Dynamic dispatch.** Reflection, `getattr`, `__getattribute__`, dynamic imports, registry-pattern lookups by string key. Search for the string form of the symbol, not just the symbol itself.
- **Framework magic.** ORM hooks, decorators, dependency injection by type, Spring/Nest-style auto-wiring, Django signals, Rails callbacks. The framework wires things you didn't write.
- **Config-driven references.** YAML/JSON/env that names the thing as a string. The change to code doesn't update the config; deployment finds out.
- **Cross-repo consumers.** Internal SDK clients, downstream services, sister-team services, public clients. Search the org. If you can't, ask who consumes this surface.
- **Out-of-band consumers.** Observability dashboards (Grafana panels keyed on a field), alerting rules, runbooks, on-call docs, customer-support macros. Renaming a metric breaks the dashboard silently.
- **Cached / serialized values.** If the change alters the shape of something currently cached or serialized, in-flight cached values become poison after deploy. Includes user-side caches, CDN, browser localStorage.
- **External monitoring of side effects.** Webhooks, email/SMS templates, exported events to data warehouse, audit logs scraped by compliance tooling.

Report: each suspected hidden coupling, how you'd verify it exists, and what the symptom of breakage looks like.

### 4. List the invariants the change might break

A change can preserve every callsite and still break callers — by breaking the invariants they assume. For each invariant, ask: does the change preserve it? If not, who depends on it?

- **Ordering and stability.** Sort order, iteration order, event order, idempotency of retries.
- **Type / shape.** Nullable becoming non-nullable (or vice versa). Optional fields becoming required. Wider type accepted, narrower type returned.
- **Error contract.** Same error type? Same error codes? Same conditions for throwing? A code path that previously threw and now returns `null` silently changes every catch block.
- **Performance.** Was this called in a hot loop? Did it amortize to O(1) and now amortizes to O(n)? Does it now do I/O it didn't before? Does it now block where it used to be async?
- **Concurrency / atomicity.** Was this safe to call concurrently? Did it hold a lock? Does the new version need ordering guarantees?
- **Defaults.** Default arg values, default config values, default SQL column values. Changing a default silently changes behavior for every caller that didn't override.
- **Side effects.** Did this previously read-only thing now write? Did this previously synchronous thing now schedule? Did this previously local thing now hit the network?

Report: invariants that change, how callers depend on each, how breakage would manifest.

### 5. Map boundaries crossed

Every boundary multiplies risk because the two sides deploy independently and may run mixed versions for some window.

For each boundary the change crosses, ask: can the two sides be deployed in either order? If not, you have a deployment-ordering hazard and need a multi-step rollout (expand → migrate → contract, or similar).

- Process boundary (worker vs API).
- Service boundary (microservice ↔ microservice).
- Repo boundary (library ↔ consumer).
- Deploy-unit boundary (frontend bundle ↔ backend).
- Persisted-data boundary (writer ↔ data ↔ reader, where "data" persists across deploys).
- Cache boundary (writer puts shape A, reader expects shape B during deploy window).
- Public API boundary (you deploy, external clients update on their schedule — i.e., never).

Report: which boundaries are crossed, deployment-ordering constraints, mixed-version compatibility for the window when both run.

### 6. Score reversibility and recovery posture

For each part of the change, score what it takes to undo if production is unhappy:

- **Trivially reversible** (minutes) — flag flip, config rollback, code revert with no persistent side effects.
- **Reversible with effort** (hours) — code revert plus cache invalidation, or rolling restart, or republishing a generated artifact.
- **Reversible with risk** (days) — data backfill, schema migration in reverse, coordinated multi-service deploy.
- **Effectively one-way** (weeks / never) — destroyed data, public API breakage seen by external clients, serialized format consumers can't be reached, secrets rotated.

A small diff with one-way consequences deserves more scrutiny than a large diff with one-button revert. Highlight the one-way doors.

### 7. Construct the 3am scenario

Synthesize a concrete failure narrative for the worst plausible outcome. Be specific:

- What's the first symptom and where does it appear (alert / customer report / silent degradation noticed days later)?
- Who's paged, with what context?
- What's the immediate mitigation (revert, flag-off, scale-out)?
- What's the full recovery (any data fixup, customer comms, post-incident review trigger)?
- Estimated time to mitigation and to full recovery.

If you can't construct this scenario, the audit isn't done. The act of writing it forces the missing pieces of the radius into view.

## Output

Always a **report** — ranked findings, never edits. Even if the user says "make this safe to ship," read it as "tell me what would make this safe to ship," not "do it." The decision is theirs.

Rank by what shapes the rollout plan:

1. **Hidden / out-of-org dependents** — you can't fix what you can't see; surface them first so the author can decide whether to coordinate.
2. **One-way doors** — irreversibility is the dominant risk factor; flag before discussing anything else.
3. **Boundary / deployment-ordering hazards** — these dictate rollout shape (atomic vs. expand-migrate-contract).
4. **Broken invariants** — drive whether callers need updating in lockstep.
5. **Direct surface volume** — drives whether the change is a one-PR or a campaign.
6. **Defensive noise / dead surface** — opportunities to *shrink* the radius before shipping (e.g., delete dead callsites first, narrow the public API, then do the real change).

### Report structure

```
# Blast radius — <change>

**Verdict:** <one sentence — small / medium / large radius, and the dominant risk factor.>

## The 3am scenario 🌙
<Concrete failure narrative for the worst plausible outcome: symptom, who pages,
mitigation, full recovery, time-to-recovery. Lead with this — it grounds everything below.>

## Direct surface
<Count and grouped list of callsites/importers/consumers. Distinguish test from
production from tooling. Link or path-line every site.>

## Indirect surface
<Persisted shapes, wire shapes, config, generated artifacts. For each: who reads
this shape, what breaks if the shape changes.>

## Hidden coupling ⚠
<Ranked. For each suspected dependency you can't grep for: how to verify it,
symptom of breakage. Out-of-org consumers go first.>

## Broken invariants
<Ranked. For each invariant the change alters: what callers assume, how breakage
manifests, which callers depend on it.>

## Boundaries crossed
<For each boundary: deployment-ordering constraint, mixed-version compatibility
during the deploy window, required rollout shape (atomic vs phased).>

## Reversibility map
<For each part of the change: reversibility tier (minutes/hours/days/never),
what recovery requires. Highlight one-way doors explicitly.>

## Recommended rollout
<Numbered. The shape of the safe rollout that falls out of the findings: e.g.,
"1. Delete dead callsites X, Y, Z (shrinks radius). 2. Add new field alongside
old (expand). 3. Migrate readers behind flag. 4. Remove old field (contract)."
Or "ship atomically — radius is small." Be specific.>
```

Tie every finding to a real location or a concrete dependent. The report's job is to make the author choose a rollout shape that matches the actual radius — atomically when safe, staged when not. A small diff with a large radius is the most dangerous thing a codebase ships; this skill exists to make that mismatch visible before the change goes out.
