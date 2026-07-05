# skills

[![skills.sh](https://skills.sh/b/cystema/skills)](https://skills.sh/cystema/skills)

Diagnostic Claude Code skills. Each produces a ranked report — no edits.

## Skills

Each shares one shape: a vocabulary, a single "test that matters", ordered checks, and a ranked report.

**Review & readability**
- **[spark-joy](skills/spark-joy)** — KonMari felt-review. Keep only what earns its place; flag shallow, untestable, low-glance-value code.
- **[naming-audit](skills/naming-audit)** — Names as one concern. Flag drift, lies, and under-description across the codebase.

**Design & data**
- **[module-design](skills/module-design)** — Whole-module shape (Ousterhout/Parnas): interface depth, information hiding, cohesion, temporal coupling.
- **[state-shape](skills/state-shape)** — Data models that let bad states exist: illegal states, primitive obsession, drifting invariants, mutation, value/identity.

**Reliability & failure**
- **[reliability-audit](skills/reliability-audit)** — Behavior under concurrency and retry: races, idempotency, lock scope, timeout budgets.
- **[error-budget](skills/error-budget)** — Error paths: swallowed errors, catches far from source, defensive noise drowning the happy path.
- **[observability-audit](skills/observability-audit)** — Telemetry coverage: invisible operations, missing context, golden signals, correlation gaps.

**Change & cost**
- **[blast-radius](skills/blast-radius)** — What breaks if a change ships wrong: callers, hidden coupling, irreversibility, recovery cost.
- **[perf-audit](skills/perf-audit)** — Structural cost: N+1/fan-out, hot-path work, movable work, over-fetch.

**Security**
- **[trust-surface](skills/trust-surface)** — Input trust and authorization: injection, validation placement, denied-by-default, IDOR.

## Install

See [skills.sh](https://skills.sh/cystema/skills) for install instructions.
