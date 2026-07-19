---
name: trace-component-flow
description: Produce a low-fidelity "wireframe" trace report for a single root UI component, following its full downstream flow — component → event → handler → backend logic → response → state change → affected components. Use when the user wants to trace one component's end-to-end flow, map what a specific component touches downstream, or document a single component's runtime path with evidence. Always traces exactly one root; never fans out to sibling roots.
---

# Trace Component Flow

Produce a **single-root trace report**: a low-fidelity, evidence-backed walk of
everything one UI component touches downstream. Faithful to one root only —
shared modules are logged, not followed.

## Mental model

Think of this like a UI wireframe of *behavior*: strip away everything that
isn't on the load-bearing path from this component to its downstream effects.
Boxes, labels, arrows, file:line evidence. No runtime instrumentation, no
architecture maps, no sibling roots.

## Procedure

### 1. Lock the root

Get the user to name exactly one root component (file path or symbol). If they
give a feature or a page, narrow to the single component that owns the entry
surface. Refuse to start until one root is named — multi-root traces are out of
scope by design.

Record: component name, file path, one-line responsibility, props, local state.

### 2. Bound the trace

State the boundary aloud in the report header:

- **In scope:** every hop reachable from the root and only the root's concern.
- **Out of scope:** sibling roots, shared utilities' other consumers, third-party
  internals, pure helpers.

When a shared store/util is touched, log it in the **Out-of-Scope annex** and
resume tracing only the branch that returns to this root's concern.

### 3. Walk the hops, in order

For each hop, capture the symbol, the file:line evidence, and what the next hop
is. Use the [hop-detection cheat-sheet](references/hop-detection.md) to identify
each hop across stacks.

```
root component
  → events it emits / binds          (§1)
  → handlers                         (§2)
  → backend / service boundary       (§3)
  → response                         (§3)
  → state mutation                   (§4)
  → components that re-render        (§5)
```

Tooling is read-only: `rg`, `read`, grep, type lookups, reading tests. No
mutation, no runtime instrumentation, no code execution. Tests are fair game —
they often name the exact handler/response contracts.

### 4. Evidence every hop

Every claim cites `path:line`. If a hop cannot be confirmed statically, say so
in **Open questions / assumptions** rather than guessing. A traced-but-unverified
hop is marked `(unverified)` inline.

### 5. Render the report

Copy [assets/trace-report.md](assets/trace-report.md) as the skeleton. Fill it
in order; do not reorder or skip sections. The adjacency sketch is ASCII only —
boxes, labels, arrows. The end-to-end path is a one-line condensation.

### 6. Faithfulness check

Before finishing, for every row in §5 (affected components), walk the chain
backward: can you get from that component back to the root through §4→§3→§2→§1?
If not, move it to the Out-of-Scope annex. A trace that admits a second root is
a failed trace.

## When to stop early

- The root has no outbound events → report §0 and "no downstream flow" in §1;
  stop.
- The root is a pure presentational leaf reading props only → trace props upward
  one level to the provider, note it, stop. Do not turn this into a parent trace.
- The user asks to add a second root → decline; suggest a separate trace.

## Output

Deliver the filled report inline in the conversation. Offer to save it to a file
(e.g. `docs/traces/<root>.md`) only if asked.
