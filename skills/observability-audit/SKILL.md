---
name: observability-audit
description: Review whether operations leave enough trace in logs, metrics, and traces to tell what happened after the fact — invisible operations, context-free signals, missing golden signals, broken correlation, trace blind spots, and over-logging noise. Use when the user wants a review of telemetry coverage on the happy path, not just error handling; when an incident couldn't be diagnosed from existing logs, when "it's slow" can't be pinned to a cause, when log lines lack the ids to answer "for whom," or before shipping an operation that matters. Trigger when asked about "observability," "logging review," "metrics coverage," "tracing," "can we debug this in prod," "golden signals," or "why can't we tell what happened." Sibling to error-budget (which covers failure-path observability). Produces a ranked report — not edits.
---

# Observability Audit

When something breaks or slows in production, you can only see what the code bothered to record. This skill reviews **telemetry coverage on the happy path**: does each operation leave a trace that answers *what happened, to whom, and why* — after the fact, without adding code and waiting for it to recur? It is the sibling of [error-budget], which covers the failure path; this one asks whether even successful, slow, or surprising operations are visible.

**Diagnoses and reports; does not edit.**

## The vocabulary

- **Invisible work** — an operation that leaves no log, metric, or span. When it breaks or slows, nothing recorded says it even ran. Silent success blinds as badly as silent failure.
- **Actionable signal** — telemetry that lets an operator answer "what happened, to whom, why" without editing code and redeploying.
- **Cardinality / context** — the identifiers attached to a signal (request id, user id, resource id). Without them a line says "failed" but not "for whom," and you can't find the one bad case.
- **The three pillars** — logs (discrete: what happened), metrics (aggregate: how much / how often), traces (causal: where the time went). Different questions; gaps in each.
- **Golden signals** — latency, traffic, errors, saturation. If you can't chart these per operation, you're flying blind under load.
- **Correlation** — the ability to tie a log line to a trace to a metric spike for one request, via a shared request/trace id. Missing it leaves three disconnected views.

## The one test that matters

> **Something you care about just happened — a user completed a purchase, a request took eight seconds. Sitting in the logs, metrics, and traces afterward, can you tell that it happened, to whom, and why it was slow — without adding code and waiting for it to recur? If the operation is invisible until it throws, it fails the audit.**

## The audit order (do not reorder)

Invisible operations first (nothing to debug with at all), then context (can't find the case), then golden signals and traces (can't see load or latency), then noise (signal drowned).

### 1. Find invisible operations

Meaningful state changes and external calls with no telemetry:
- Writes/mutations that succeed silently — no log, no metric, no span.
- External calls (DB, HTTP, cache, queue publish) not wrapped in a span or counted.
- Branches that matter (fallback taken, cache miss, feature-flag path) with no signal that they were hit.

For each: the operation, what you couldn't reconstruct after an incident, the minimal signal to add (log line / counter / span).

### 2. Check signal context (cardinality)

Do logs and spans carry the ids to answer "for whom" and "which one"?
- Context-free lines: `"processing..."`, `"done"`, `"failed to update"` — no request id, user id, resource id.
- `logger.error(e.message)` that drops the stack and the surrounding context.
- Metrics with no dimensions where you'll need to slice (per-endpoint, per-tenant).

For each: the signal, the missing ids, what question it currently can't answer.

### 3. Check golden signals per operation

For the operations that matter:
- **Latency** — measured (and as a distribution, not just a mean that hides the p99)?
- **Traffic** — throughput counted?
- **Errors** — tagged by cause, or lumped in one bucket that can't tell a timeout from a 4xx from a panic?
- **Saturation** — pool/queue/worker utilization visible before it hits the ceiling?

For each gap: the operation, the missing signal, the incident it would leave undiagnosable.

### 4. Check correlation and trace coverage

- Is a request/trace id threaded through logs and propagated across service hops? Or are logs, metrics, and traces three views that can't be joined for one request?
- Do external calls sit inside spans, or is their time invisible — "eight seconds somewhere" instead of "six seconds in the payments call"?

For each: the break in the chain, what it costs during triage, where to thread/propagate the id or add the span.

### 5. Check the altitude (noise)

Over-observability drowns the signal:
- Log-per-loop-iteration, debug logs left at info, the same event logged at three layers.
- High-cardinality labels on metrics that blow up the backend.

For each: what to cut or lower, so the operations that matter aren't buried.

## Output

Always a **report** — ranked findings, never edits.

Rank: invisible operations first (no debugging possible), then missing context, then golden-signal and trace gaps, then noise.

### Report structure

```
# Observability audit — <target>

**Verdict:** <one sentence — can you tell what happened in prod from telemetry alone?>

## What's observable ✨
<Operations that record enough — contextual logs, span coverage, golden-signal
metrics. The convention to spread.>

## Invisible operations 🕳
<Ranked. For each: the operation, what you couldn't reconstruct after an incident,
the minimal signal to add.>

## Context-free signals
<Ranked. For each: the signal, the missing ids, the question it can't answer.>

## Golden-signal gaps 📉
<Ranked. For each operation: missing latency/traffic/errors/saturation, the
undiagnosable incident it leaves.>

## Correlation & trace blind spots 🔗
<Ranked. For each: the break in the id chain or the untraced call, its triage cost,
the propagate/span fix.>

## Noise (over-logging)
<Ranked or grouped. What to cut or lower so real signal surfaces.>

## Where to start
<Numbered: make invisible operations visible, then add context ids, then golden
signals and trace propagation, then trim noise. Cross-reference error-budget for
failure-path observability.>
```

Tie every finding to a real operation and a concrete question triage would need to answer. The goal is code you can debug from production telemetry alone — where the things that matter are visible before they break, not only after.
