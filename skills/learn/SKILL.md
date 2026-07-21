---
name: learn
description: Teach a topic at a requested level from 1 (first encounter) to 5 (expert), grounded in the current workspace and trustworthy sources. Use when the user invokes /learn or $learn, asks to learn or understand a concept at a numbered level, wants code in a file explained at an appropriate depth, or wants an ongoing learning path with retrieval practice and learning records.
---

# Learn

Teach one tightly scoped thing at the requested depth. Optimize for what the learner can understand and use, not for how much information can fit in the answer.

## Parse the request

Interpret requests as:

```text
/learn <level 1-5> [-- <topic, question, file, or selection>]
```

Accept natural variations such as `/learn 1 - Temporal SDK usage in this file`, `level 2`, `L3`, or `$learn 4`. Treat either `-` or `--` after the level as an optional separator. If no level is supplied, infer the lowest useful level from the user's language and workspace evidence; state the assumption in one short sentence. Ask only when choosing incorrectly would materially change the lesson.

Treat `this file`, `this function`, or `this selection` as the code context supplied by the client. If that context is unavailable, inspect the most likely workspace target before asking for a path.

Teach at the selected level only. Do not explain the topic five times unless the user asks to compare levels.

## The five levels

Levels measure depth of mental model, not the learner's intelligence.

### Level 1 — Orientation

Assume no prior knowledge. Give the purpose, one concrete mental model, and the smallest useful example. Define every necessary term in plain language. Focus on recognition and a first success; omit machinery that is not needed yet.

### Level 2 — Working use

Assume the learner knows the basic vocabulary. Teach the normal workflow, the important API or components, and how to make a small change safely. Explain common inputs, outputs, and one likely mistake.

### Level 3 — System understanding

Explain how the parts cooperate: lifecycle, control flow, state, boundaries, and important invariants. Connect the local example to the broader system and show why the code is shaped this way. Include debugging cues and meaningful alternatives.

### Level 4 — Advanced practice

Cover production concerns and non-obvious behavior: failure modes, concurrency, retries, performance, testing strategy, operational tradeoffs, and sharp edges. Compare viable designs and explain when each wins.

### Level 5 — Expert inquiry

Engage at peer depth. Examine implementation details, protocol or runtime semantics, historical/design constraints, unresolved tradeoffs, and where conventional guidance breaks down. Distinguish documented facts, code-backed inference, and opinion. Prefer primary sources and source code.

## Ground the lesson

For workspace code:

1. Read the target before explaining it.
2. Trace only enough nearby definitions, call sites, tests, and configuration to make the explanation accurate.
3. Explain the concept through the target's real names and control flow.
4. Separate what the local code does from what the library or framework guarantees.

Never trust memory for unstable technical details. Prefer sources in this order:

1. The target code, tests, lockfiles, and installed version in the workspace.
2. Official documentation and specifications for that exact version when available.
3. Primary source code, maintainers' material, and papers.
4. High-quality secondary explanations only when primary material is insufficient.

Cite external claims with direct links. Do not browse when the code and local documentation fully answer the question; browse when current or version-specific behavior matters.

## Teach through a tight loop

Default to an in-conversation micro-lesson with this shape:

1. **Outcome** — what the learner will be able to recognize or do.
2. **Mental model** — the smallest accurate model at the selected level.
3. **Walkthrough** — connect the model to a concrete example or the target code.
4. **Try it** — one small prediction, retrieval question, or edit for the learner.
5. **Check** — give feedback after the learner responds; do not reveal a quiz answer in advance.
6. **Next rung** — one sentence describing what the next level would add.

Keep the lesson within working memory. Prefer one tangible win over a survey. Match the amount of terminology, examples, and caveats to the selected level.

For knowledge, reduce incidental difficulty. For skills, add desirable difficulty through retrieval, spacing, and interleaving. Do not mistake a fluent explanation for evidence that the learner can use the idea.

## Preserve learning state when useful

Treat learning as stateful when the workspace already contains learning files or the user asks for an ongoing course. Read existing `MISSION.md`, `RESOURCES.md`, `NOTES.md`, `learning-records/`, `reference/`, and `lessons/` before continuing the path.

Create or update learning artifacts only when they add durable value:

- `MISSION.md` records the concrete outcome the learner is pursuing. Ask why before creating it if the mission is unclear.
- `RESOURCES.md` curates annotated, high-trust knowledge sources and practitioner communities.
- `NOTES.md` records stable teaching preferences.
- `learning-records/NNNN-<slug>.md` records demonstrated understanding, disclosed prior knowledge, corrected misconceptions, or a changed mission. Do not record mere exposure.
- `reference/*.html` stores compact, printable material worth revisiting.
- `lessons/NNNN-<slug>.html` stores a self-contained lesson when the user asks for a durable artifact or an ongoing course benefits from one.
- `assets/` stores reusable styles and interactive components shared by durable lessons.

Do not force course scaffolding for a one-off `/learn` request. Never edit application code during a lesson unless the user explicitly asks to practice through an edit.

## Calibrate from evidence

Use the requested number as the ceiling for conceptual depth, then adapt examples to the learner's demonstrated knowledge. If they answer the check easily, offer a harder application at the same level or the next rung. If they struggle, repair the missing prerequisite without labeling them or silently changing the requested level.

End by inviting the learner to answer the check or ask about the part that remains unclear.
