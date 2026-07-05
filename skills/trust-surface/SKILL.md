---
name: trust-surface
description: Review where untrusted data enters and whether authorization is enforced — input crossing trust boundaries unvalidated, tainted data reaching injection sinks (SQL, shell, HTML, path, template), implicit-allow authorization, and missing resource-ownership checks (IDOR / confused deputy). Use when the user wants a defensive security review of input handling and access control on their own code; when reviewing routes/handlers for who-can-do-what, when validating that user input is checked at the boundary, or before shipping endpoints that touch non-public data. Trigger when asked about "trust boundary," "input validation," "injection," "authorization," "auth check," "IDOR," "access control," "denied by default," or "can users touch each other's data." Defensive audit that flags structural holes and recommends a deeper security review for hot findings — not a penetration test. Produces a ranked report — not edits.
---

# Trust Surface

Two questions decide whether a feature is safe: can untrusted data get somewhere dangerous, and can a caller act on something that isn't theirs? This skill audits both — **input trust** (validation at the boundary, taint to sinks) and **authorization** (denied-by-default, resource ownership). It is a defensive review of your own code: find the holes, recommend closing them.

**Diagnoses and reports; does not edit. A review lens, not a penetration test — flag structurally and recommend a deeper security review for anything hot.**

## The vocabulary

- **Trust boundary** — the line where data crosses from untrusted (user, network, third-party, uploaded file) into trusted (your logic, DB, shell). Validation belongs *at* the boundary, once, before the data spreads inward.
- **Tainted data** — untrusted input not yet validated/sanitized. The danger is tainted data reaching a sink without being escaped or checked.
- **Sink** — a place where a structured artifact is built from strings: SQL query, shell command, HTML, file path, redirect URL, template, log line.
- **Authentication vs authorization** — authn = *who are you*; authz = *what may you do*. This skill audits authz — per route, per resource.
- **Denied by default** — the safe posture: no explicit grant means no access. **Implicit-allow** (open by default, deny by exception) is the bug pattern.
- **Confused deputy / IDOR** — code acting for a caller on a resource the caller doesn't own, because it trusts a caller-supplied id without an ownership check (`GET /invoice/:id` returning anyone's invoice).

## The one test that matters

> **Trace one piece of user-controlled input from where it enters to every place it's used, and for every route that reads or writes non-public data, ask what stops a logged-in user from acting on a resource that isn't theirs. If input reaches a sink untrusted, or a route trusts the caller's own claim about what they may access, it's a hole.**

## The audit order (do not reorder)

Injection first (untrusted-in to dangerous-sink is the highest-severity, most mechanical to spot), then validation placement, then authorization posture, then ownership.

### 1. Map trust boundaries and trace taint to sinks

Find where untrusted data enters: request params, query strings, headers, body, cookies, uploaded files, third-party API responses. Trace each to whether it reaches a sink unescaped:
- **SQL** — string-built queries / string interpolation instead of parameterized statements.
- **Shell / OS** — user data in `exec`/`system`/subprocess strings.
- **HTML / DOM** — unescaped input rendered (XSS), `innerHTML`, `dangerouslySetInnerHTML`.
- **Path** — user input in file paths (traversal, `../`).
- **Redirect / SSRF** — user-controlled URLs followed or redirected to.
- **Template / log** — user input into template engines or log lines (log injection, template injection).

For each: the entry point, the sink, the vector, the fix (parameterize, escape-at-sink, allow-list, safe API).

### 2. Check validation placement

- Is input validated once at the boundary, or trusted implicitly and re-checked haphazardly deep inside?
- Boundary-less input — data flowing straight from request into logic with no validation layer.
- Allow-list vs deny-list — deny-lists miss cases; flag them.

For each: where validation is missing or misplaced, where it should live (one boundary layer), the shape (schema/allow-list).

### 3. Enumerate the authenticated surface and check denied-by-default

List every route/handler/RPC that reads or writes non-public data. For each, what does it authorize?
- Implicit-allow: default-open with deny-by-exception, so a new route is unprotected until someone remembers.
- Missing checks: a handler with no authz at all.
- Disabled/commented guards, `// TODO: add auth`, test bypasses left in.

For each: the route, the current posture, the denied-by-default fix.

### 4. Hunt confused-deputy / IDOR

Routes that take a resource id and act on it:
- Does the handler check the caller *owns* (or is permitted on) that resource, or does it trust the id alone?
- Sequential/guessable ids amplify the risk.
- Mass-assignment cousins — a caller setting fields (role, ownerId) they shouldn't control.

For each: the route, the id it trusts, the missing ownership/scope check, the fix (scope the query to the caller, verify ownership before acting).

## Output

Always a **report** — ranked findings, never edits. Flag structurally; for any hot finding, recommend a focused security review or test rather than asserting exploitability.

Rank by severity × reachability: injection vectors first, then missing/implicit authz, then ownership gaps, then validation placement.

### Report structure

```
# Trust surface — <target>

**Verdict:** <one sentence — can untrusted data reach danger, or callers reach
what isn't theirs?>

## What's solid ✨
<Boundaries validated once, parameterized sinks, denied-by-default routes,
ownership-scoped queries. The convention to keep.>

## Injection vectors 💉
<Ranked. For each: entry point, sink, vector, fix (parameterize/escape/allow-list).>

## Authorization gaps 🔓
<Ranked. For each: route, current posture (implicit-allow / missing / disabled),
denied-by-default fix.>

## Ownership gaps (IDOR) 🪪
<Ranked. For each: route, the caller-supplied id it trusts, missing ownership
check, scope-to-caller fix.>

## Validation placement
<Ranked. For each: input trusted implicitly or validated too deep, where the
boundary layer should live, allow-list shape.>

## Where to start
<Numbered: injection first, then close implicit-allow routes, then ownership
checks, then centralize validation. Recommend a deeper security review for the
hottest findings.>
```

Tie every finding to a real route or data flow. This is defensive work — the goal is to find and close holes in your own code, and to know which findings are hot enough to deserve a dedicated security pass.
