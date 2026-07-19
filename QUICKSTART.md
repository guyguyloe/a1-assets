# Quickstart

This repo holds an article on organizing code by seam type, the model behind it, and a drop-in skill that puts the model to work.

This guide is the shortest path from landing here to running your first thread on your own code — about ten minutes.

## What's in this repo

| File | What it is | Read it when |
|---|---|---|
| `final-draft.md` | The article — the narrative and the argument. | You want the why. |
| `STRUCTURE.md` | The model stated flat: seam-type as an addressable axis, the one coined term (`logical section`), the conventions. | You want the rules. |
| `PI-MONO.md` | A worked thread through a real monorepo (`pi-mono`), every hop tagged with its seam address. The walk is the proof. | You want to see one done. |
| `skills/trace-thread/` | The drop-in skill that operationalizes the model. **This is the thing you install.** | You want to run one. |
| `skills/trace-component-flow/` | The older, scoped skill `trace-thread` builds on (forward, single UI root). Optional companion. | Your root is a UI component. |
| `skills/write-tests/` | A sibling skill that already thinks in seams. Referenced, not a dependency. | You're hardening test coverage. |

## The one-paragraph model

Every module has one interface, and that interface lives at a **seam**. The load-bearing property is **where the seam sits**, not what the container looks like:

- **External seam** → the module is a **package**. Cross it and you assume *identity-only references* — no shared state across the joint.
- **Internal seam** (private to a package's implementation) → the module is a **logical section**. Inside it you assume *shared implementation context*. Siblings share state and helpers freely.

`logical section` is the only coined term. "Package" is the container word you already use, reused as the carrier for the external case so package-boundaries and seam-boundaries fuse into one system.

A **thread** is a traversal over those addressable seams from a named root, each hop tagged with its seam address and the rule that holds there. Full statement: [`STRUCTURE.md`](STRUCTURE.md).

## Prerequisites

1. **An agent harness that loads skills from markdown files** (Pi, or anything with the same convention). The skills here are plain prompt markdown — no code, no runtime.
2. **Matt Pocock's `/codebase-design` skill vocabulary.** `trace-thread` speaks the frozen terms (*module, interface, seam, adapter, depth*) exactly. Install those first if your harness lacks them, because this skill builds on them rather than shipping them.
3. **A repo you can point `rg` at.** The skill is read-only: `rg`, reads, grep, type lookups, reading tests. No mutation, no runtime instrumentation.

## Step 1 — Install the skill

Copy `skills/trace-thread/` into your harness's skills directory, preserving the subfolder (it carries `THREAD-EXHIBIT.md`, which the skill links to).

Pi example (skills live under `~/.pi/agent/skills/<category>/<name>/`):

```bash
cp -r skills/trace-thread ~/.pi/agent/skills/engineering/trace-thread
```

That's the whole install. The skill modifies none of Pocock's skills. It only builds on them.

**Optional companion:** if you trace UI components, also install `skills/trace-component-flow/`. `trace-thread` borrows its hop-detection cheat-sheet when present.

For a single forward UI root, `trace-component-flow` is the tighter tool. `trace-thread` is the generalization to any root, both directions.

```bash
cp -r skills/trace-component-flow ~/.pi/agent/skills/engineering/trace-component-flow
```

## Step 2 — Apply the model to your repo (optional but recommended)

The skill works on any repo, but it's anchored when the repo declares its seams. You don't need all of this on day one — start with the first two and grow into the rest.

1. **`CONTEXT.md` per package**, with a **Language** section that names the package's logical sections. The named concepts in that section *are* the sections, which makes that file the declaration home.
2. **`CONTEXT-MAP.md`** at the root, with a one-way package dependency diagram. This is the package diagram. Its edge rule is the seam-direction rule.
3. **`STRUCTURE.md`** at the root, stating the seam-type convention for *your* repo. Adapt [`STRUCTURE.md`](STRUCTURE.md) here — keep the abstract rule (internal = private placement; adapter count orthogonal), drop the `pi-mono` names.
4. **`docs/adr/`** with one ADR per expensive-to-reverse seam-placement decision. ADRs attach to the **seam** they decided (a package edge, or a section edge within a package), not to interface shape, and not as a free-floating paragraph.

The invariant to hold: **every line of code belongs to a module.** Sections are the *named, addressable subset*. Shared helpers inside a package are modules too, just not sections.

Don't force a helper into a section it doesn't deserve.

## Step 3 — Run your first thread

In your harness, invoke `trace-thread` and give it **exactly one root module** (a symbol, a file path, or a package) and a direction. If you name a feature or a page, the skill narrows you to the single module that owns the entry surface.

A second root is a second thread, so the skill refuses to start until one root is locked.

Pick the direction by where the symptom sits:

- **Forward** answers "what does this trigger?" Use it when you have a root and want its downstream effects, with the symptom *at* the root.
- **Backward** answers "what does this need?" or "where is this bug?" You want the closure of what produces a root or symptom, with the symptom *into* the root. The "I can't articulate the bug" case is usually this one.

Example prompt shape:

```
trace-thread: root = Board (src/ui/Board.svelte), direction = backward.
Symptom: board renders stale scope after a department switch.
```

The skill will:

1. Lock the root and record its seam type (external → package, or internal → section).
2. State the boundary (in scope / out of scope) in the report header.
3. Walk the hops, tagging each with its seam address — `(external, A ↔ B)` or `(internal, A ↔ B, within package P)` — and the rule that holds there (identity-only across a package edge; shared state inside one).
4. Cite `path:line` for every claim. No `path:line`, no claim.
5. Run a **faithfulness check** (every hop chains back to the root) and a **drift check** (seam moved without its ADR / orphaned ADR / expensive seam with no ADR).

You'll get an ordered list of hops with addresses, a one-line condensation of the end-to-end path, and any drift surfaced as its own section.

It's read-only, so it won't fix anything. It hands off to `/diagnosing-bugs`, `/improve-codebase-architecture`, or `/domain-modeling` when the thread surfaces the next move.

## Reading order if you want to understand it first

1. [`final-draft.md`](final-draft.md): the story and the argument.
2. [`STRUCTURE.md`](STRUCTURE.md): the model, flat.
3. [`PI-MONO.md`](PI-MONO.md): a real thread, tagged hop by hop.
4. [`skills/trace-thread/SKILL.md`](skills/trace-thread/SKILL.md): the procedure you'll run.
5. [`skills/trace-thread/THREAD-EXHIBIT.md`](skills/trace-thread/THREAD-EXHIBIT.md): the shape to aim for when your own thread hits two joints that both sit behind collective interfaces.

## What this guide is not

- It's not the model. That's `STRUCTURE.md`.
- It's not the proof. That's `PI-MONO.md`.
- It's not the skill. That's `skills/trace-thread/SKILL.md`.
- It's the shortest path from "I just landed here" to "I just ran a thread on my own code."
