---
name: trace-thread
description: Trace a code-thread — an evidence-backed walk over addressable seams from a named module, in either direction. Use when the user can't articulate a bug and needs to locate the feature, wants to understand a module's full reach (what it triggers / what it needs), wants to find where a seam lives or whether it has drifted from its ADR, or is re-entering a codebase that feels alien. The generalization of trace-component-flow to any root, both directions.
---

# Trace Thread

A **thread** is a traversal over addressable seams from a named module — the
generalization of `trace-component-flow` to any root, in both directions. Point
at a module, pick up the rope, and follow seams: inside a package you follow
internal seams between sections; at the package edge you cross the external seam
to the next package. Each hop is tagged with its seam address and the rule that
holds there, so the trace stays anchored to the structure instead of drifting
into a paraphrase of it.

The motivating task is **feature location**: "I can't articulate the bug, so I
trace the path." When you can't yet name where to point a feedback loop, a
thread locates the feature for you.

This skill is **read-only**: `rg`, `read`, grep, type lookups, reading tests. No
mutation, no runtime instrumentation, no code execution. Tests are fair game —
they often name the exact interface contracts.

## Vocabulary

This skill speaks the `/codebase-design` vocabulary exactly — **module**,
**interface**, **seam**, **adapter**, **depth** — and does not substitute
"component," "service," "API," or "boundary." It adds **one axis** and **one
term** on top. Don't reach for this skill to *design* a module's shape — that's
`/codebase-design`. This skill *navigates* structure that already exists.

**Seam type** — the axis this skill runs on. Where a module's interface lives is
its own property, distinct from what sits behind it. A seam is either:

- **External** — the interface lives at a package edge. Cross it and you assume
  **identity-only references**: the two sides don't share state; they cross by
  model identity and message protocol. The external-seam module is realized as a
  **package** — the container word the repo already uses, now the carrier for an
  external seam, so package-boundaries and seam-boundaries fuse into one system
  instead of two.
- **Internal** — the interface lives private to its package's implementation.
  Inside it you may assume **shared implementation context**: siblings share
  state and helpers freely.

Seam type is **placement**, not container shape. It is orthogonal to adapter
count (Feathers' "one adapter = hypothetical seam, two = real"): an internal
seam may be hypothetical (`utils/` helpers) or real (`Models` ↔ `Provider` with
dozens of adapters). Don't require two adapters to justify an internal seam, and
don't read a 0–1 adapter internal seam as "no adapters wanted."

**Logical section** — the one coined term. A module whose seam is internal: a
named cluster of modules inside a package, sharing a purpose, presenting no
collective external interface. The discriminator is one question: *does the
group present a collective interface, or a bag of independent exports that only
share a label?* Collective interface → **module** (say **package** when its seam
is external). Shared label only → **logical section**. Sections are declared in
the package's `CONTEXT.md` — the named concepts in its Language section *are* its
sections.

**Thread** — a traversal over addressable seams, not a new container. Two
directions off the same root:

- **Forward (what it triggers)** — the path from a module through the events and
  actions it dispatches to downstream effects. `trace-component-flow` is the
  existing forward operationalization, scoped to one UI root.
- **Backward (what it needs)** — the closure of every line of code that exists to
  make the root observable. The reasoning view: "the Board's thread" is the set
  of sections and packages it depends on to function.

> **Lineage.** The traversal is **program slicing** (Weiser, 1981): a backward
> slice is every statement that affects a point; a forward slice is every
> statement it affects. The root is the *slicing criterion*; the thread is the
> slice. The motivating task is **feature location** / the **concept assignment
> problem** (Biggerstaff, 1993). The addressable structure is what **concern
> graphs** (Robillard & Murphy, 2002) and **intensional views** (Snelting)
> already gave the field. The mechanism is a rediscovery, arrived at from a
> scoped forward-only skill. What's added here is repurposing Feathers' seam
> into the structural axis with a rule per hop, aimed at the case where an agent
> and a human navigate the same addressable structure with a shared vocabulary.

## When exploring the codebase

Read `CONTEXT.md` (and `CONTEXT-MAP.md` if it exists) first, so section names
and package edges match the project's declared structure, and check ADRs in the
area you're touching. If the repo carries a `STRUCTURE.md` that states the
seam-type model, read it — it's the address book this skill walks.

## Procedure

### 1. Lock the root

Get the user to name exactly one root module — a symbol, a file path, or a
package. If they name a feature or a page, narrow to the single module that owns
the entry surface. Refuse to start until one root is named — a second root is a
second thread.

Record: module name, file path, one-line responsibility, and its **seam type**
(external → package, or internal → section).

### 2. Pick a direction

- **Forward** when the question is "what does this trigger?" — the user has a
  root and wants its downstream effects. (This is the `trace-component-flow`
  case, generalized beyond UI.)
- **Backward** when the question is "what does this need?" / "what makes this
  observable?" / "where is this bug?" — the user has a root or a symptom and
  wants the closure of what produces it.

If the user can't say, ask which side of the root the symptom is on: symptom at
the root → forward (trace what it triggers); symptom produced *into* the root →
backward (trace what feeds it). The feature-location case — "I can't articulate
the bug" — is usually a forward trace from the component where the symptom
appears, or a backward trace from the observable symptom itself.

### 3. Bound the thread

State the boundary aloud in the report header:

- **In scope:** every hop reachable from the root in the chosen direction, over
  addressable seams.
- **Out of scope:** the opposite direction's branches beyond the root, sibling
  roots, third-party internals, pure helpers not on the load-bearing path.

When a shared helper is touched, log it in the **Out-of-Scope annex** and resume
tracing only the branch that returns to this root's concern.

### 4. Walk the hops, tagging each with its seam address

For each hop, capture: the symbol, `path:line` evidence, the next hop, and the
**seam address** — `(external, A ↔ B)` or `(internal, A ↔ B, within package P)`.
The address is the load-bearing part: it records *which kind of seam* you
crossed and the rule that holds there.

- **Within a package:** follow internal seams between sections. Assumption at
  each hop: **shared state and helpers**.
- **Across a package edge:** cross the external seam. Assumption at each hop:
  **identity-only references** — no shared state across the joint.

The layer tag is what makes the traversal safe at higher abstraction: each hop's
address carries the assumption that holds there, so reasoning in natural
language drifts less than it would unanchored. If a hop's seam type is unclear,
say so — don't tag it to keep the walk tidy.

Use the hop-detection cheat-sheet from `trace-component-flow` (if installed) to
identify hops across stacks.

### 5. Evidence every hop

Every claim cites `path:line`. If a hop can't be confirmed statically, say so in
**Open questions / assumptions** rather than guessing. A traced-but-unverified
hop is marked `(unverified)` inline. No `path:line`, no claim.

### 6. Render the thread

Deliver the thread inline as an ordered list of hops, each with its seam address
and the rule that holds there, plus a one-line condensation of the end-to-end
path. An ASCII adjacency sketch is optional. Offer to save it to
`docs/threads/<root>.<direction>.md` only if asked.

### 7. Faithfulness + drift check

Before finishing, run both checks:

- **Faithfulness.** For the last hop, walk the chain back to the root through the
  recorded seams. If you can't, move it to the Out-of-Scope annex. A thread that
  admits a second root is a failed thread.
- **Drift.** The addresses that made the walk safe are the addresses that make
  drift cheap to catch. For each seam the thread crossed, check:
  - **Seam moved, ADR didn't follow** — the implementation crossed or relocated a
    boundary; the recorded decision no longer describes where the glue is.
  - **ADR whose seam no longer exists** — the joint it decided was refactored
    away; the ADR is now a decision about nothing.
  - **Seam with no ADR** — a joint that became expensive-to-reverse with no
    recorded why. This is exactly where a future reader will wonder "why on
    earth is it this way?"

Surface drift as a section of the report, not silently. Don't fix it here —
this skill is read-only.

## When to stop early

- The root has no outbound seams in the chosen direction → report the root and
  "no downstream/upstream seams" and stop.
- The root is a pure leaf → trace one hop in the opposite direction to its
  provider, note it, stop. Don't turn it into a parent trace.
- The user asks to add a second root → decline; suggest a separate thread.

## Hand-offs

This skill is read-only and produces a thread. It does not fix, test, or
restructure. Hand off when the thread surfaces the next move:

- **The thread located the bug's neighborhood but you still need a pass/fail
  signal** → `/diagnosing-bugs`. You now know where to point the feedback loop;
  let it build the tight loop and refuse to theorize until it has one. The
  thread is the comprehension step; `diagnosing-bugs` is the loop step. They
  pair: feature location → feedback loop.
- **The drift check found a seam that moved without its ADR, an orphaned ADR, or
  an expensive seam with no ADR** → `/improve-codebase-architecture` with the
  specific joint. A mis-placed seam is a deepening opportunity; the thread is the
  survey that found it.
- **The walk surfaced a section that isn't named in `CONTEXT.md`, or a
  seam-placement decision that's hard to reverse and surprising without
  context** → `/domain-modeling` to name the section or attach an ADR to the
  seam. ADRs attach to the seam they decided; the thread gives you the seam's
  address.
- **The question turned out to be "what should this look like?" or "does this
  state model feel right?"** → `/prototype`. A thread answers *what exists*; a
  prototype answers *what should be*.

## What this skill is not

- **Not a container.** A thread is a *view* over the two seam types, computed
  from addressable seams — not a third layer in the model.
- **Not a design tool.** It navigates structure that exists. Designing a
  module's shape is `/codebase-design`; surfacing candidates to deepen is
  `/improve-codebase-architecture`.
- **Not a substitute for `trace-component-flow`.** That skill is the forward,
  single-UI-root operationalization and stays in scope there. This is the
  generalization: any named module, both directions, over addressable seams. If
  the root is a single UI component and the question is forward-only,
  `trace-component-flow` is the tighter tool.

## Going deeper

- [THREAD-EXHIBIT.md](THREAD-EXHIBIT.md) — a worked thread through a real
  monorepo (user prompt → agent loop → LLM stream), each hop tagged with its
  seam address. The walk that shows the tags holding *is* a thread; the thread
  is the proof.
