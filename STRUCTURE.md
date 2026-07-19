# Structure

This document records the structural pattern of the codebase. It establishes the pattern rather than exploring it. It builds on the shared engineering vocabulary (**module**, **interface**, **seam**, **adapter**, **depth**) and adds one distinction on top: **seam-type as an addressable property**.

Read it alongside [`CONTEXT-MAP.md`](CONTEXT-MAP.md), which points to each package's domain language. CONTEXT-MAP names *what* each context is; this document names *how the contexts are glued together, and how to reason over that glue*.

## The model

The aim is to document the seams in a way that's structured and non-vague, so that structure becomes a query rather than git-archaeology. Doing that takes **addressable seams**: every seam has an address you can point at, and ADRs attach to the seam they decide. A module is scale-agnostic (it might be a function, a class, or a package), but every module has exactly one interface, and that interface lives at a seam.

A blank address, one that just says "this seam is between module A and module B," carries no real information about the joint. What makes an address carry information is a layer tag on each seam, recording *which kind of seam* a module's interface lives at:

- **Package**: a module whose seam is **external**. "Package" is the container word the repo already uses; the convention below fuses package-boundaries with seam-boundaries so a single boundary system runs.
- **Logical section**: a module whose seam is **internal** (private to its package's implementation). This is the one coined term; "package" is existing vocabulary reused as the carrier for the external case.

That binary is the addressing scheme, not the root. Everything below follows from *addressable seams*; the binary is what makes them addressable rather than vague. It tells you, per seam, whether the two-adapter gate applies, where an ADR's home is, and which traversal assumption holds at each hop.

## Two seam-types

### Packages (external seams)

A package is a module whose interface lives at an external seam. Four properties follow from that:

- **Clear boundaries**: the seam is real, not hypothetical. Two adapters justify it (typically production plus a test stand-in); a single-adapter external seam is indirection, not a package.
- **Owns its data**: the implementation behind the seam is the authority for its data. Other packages reference it by identity rather than reaching in.
- **Depth**: it passes the deletion test. Delete it and complexity reappears across callers, which means it was earning its keep; it does not simply vanish.
- **Extractable**: it could be pulled out as an independent service or package, because the seam already exists.

**Convention: the external-seam module is realized as a package.** One external seam per package. The CONTEXT-MAP dependency diagram (`doc-reg ← pms ← core ← user-interface`) is the package diagram, and its one-way edge rule is the seam-direction rule. This is a deliberate choice, not a coincidence. Only one term is coined in this model ("logical section"); the external case reuses "package," the container word the repo already has, so package-boundaries and seam-boundaries fuse into a single boundary system instead of running as two drift-prone axes.

### Logical sections (internal seams)

A logical section is a module whose interface lives at an internal seam, private to its package's implementation. It has three properties:

- **Readability, not extractability**: a section exists to make a package legible, not to be pulled out. It still passes the deletion test: delete it and complexity reappears across the rest of the package, so it was earning its keep. What separates a section from a package is not depth but seam type. The internal seam is the signal that this is a section, not a package.
- **Shares freely**: it may share internal state and helpers with sibling sections. The seam is private to the package's implementation. Placement is justified by legibility (the deletion test), not by the two-adapter rule. Adapter count is orthogonal to placement: an internal seam may be hypothetical (0–1 adapters) or real (≥2). Do not require two adapters to justify an internal seam, and do not read that as "no adapters wanted."
- **Named in CONTEXT.md**: each package's CONTEXT.md already lists its sections as the named concepts in its Language section (e.g. core's `Scope`, `WorkspaceStateManager`, `OrgManager`, `PmsAdapter`, `FsAdapter`; user-interface's `Navigation`, `Board`, `scope-endpoint`, `content-endpoint`, `session-store`). The declaration home for sections is the package's CONTEXT.md.

## Interfaces and seams

- **Interface**: the contract *at* a glue point, covering the type signature plus invariants, ordering, error modes, required configuration, performance characteristics, and everything a caller must know. Interfaces tell us *what fits together*.
- **Seam**: *where* the glue goes, a place where you can alter behaviour without editing in that place; the location at which a module's interface lives. Seams are the glue of the system.

The two are co-determined. You never place a seam without defining the interface that crosses it, and you never define an interface without implicitly placing a seam. The distinction between them is in what each is *expensive* to reverse. Interface *shape* is cheap to reverse: reshape the contract and callers update locally. Seam *placement* is expensive to reverse, because moving a seam ripples across every caller. That asymmetry is why the two earn different documentation treatment.

## ADRs are recorded seam decisions

ADRs cluster on seam placement, not on interface shape. The three criteria for offering an ADR (hard to reverse, surprising without context, the result of a real trade-off) are precisely the properties of a seam-placement decision:

- **Hard to reverse**: moving a seam ripples across every caller (e.g. docs/adr/0001-slug.md accepts "two seams instead of one" because unifying them later is expensive).
- **Surprising without context**: why does storage not know about field names? Why are reads and writes on separate seams? A future reader will ask, and the ADR answers.
- **Real trade-off**: the rejected alternative is always *a different seam configuration* (one seam vs two; coupled vs decoupled; shared root vs split root).

ADRs that decide an **external seam** attach to a **package edge** (e.g. pms↔doc-reg, core↔pms). ADRs that decide an **internal seam** attach to a **section edge** within a package (e.g. Scope↔WorkspaceStateManager inside core, recorded in docs/adr/0001-slug.md). The attachment is what makes a seam *addressable*: an ADR is not a free-floating decision, it is a decision about a specific joint, and the joint has a location in the structure.

## Addressability

The model gives three things an address:

- **Seams**: by layer (external = package edge, internal = section edge) and by the two modules they join.
- **ADRs**: by the seam they decide. An ADR's home is the edge it shaped.
- **Code-threads**: by traversal, below.

Addressability is the property that makes "reason over the code as it grows" tractable. Without it, structure is reconstructable only by reading git history and prose references. With it, structure is a query.

## Code-threads

The code-thread concept emerged from `trace-component-flow`, which operationalizes the forward, single-root case for UI components. That skill is scoped to UI by necessity: it can't generalize. This document generalizes the concept to any named module over addressable seams, in both directions below.

A code-thread is **a traversal over addressable seams**, not a definition of a new container. Point at a named module and pick up the rope by following seams:

- **Within a package**: follow internal seams between sections (Board → content-endpoint → session-store inside user-interface).
- **Across packages**: cross the external seam at the package edge (session-store → PmsAdapter, crossing user-interface→core; PmsAdapter → Registry, crossing core→pms→doc-reg).

The layer tag is what makes the traversal safe at higher abstraction. Within a package you may assume shared state and helpers, because the seam is internal; across a package edge you assume identity-only references, because the seam is external. Each hop's address carries the assumption that holds there, so the trace stays anchored to the structure rather than drifting into a paraphrase of it.

Two traversal directions off the same root, both over the same addressable seams:

- **Forward (what it triggers)**: the path from a module through the events and actions it dispatches to downstream effects. `trace-component-flow` is the existing operationalization of this direction, scoped to a single UI root.
- **Set (what it needs / what belongs to it)**: the backward closure of every line of code that exists to make the root observable. This is the reasoning view: "the Board's thread" is the set of sections and packages it depends on to function.

The thread is not a third layer in the model. It is a *view* over the two seam-types, the thing you reason over, computed from addressable seams rather than re-derived from scratch each time.

## Drift

The payoff of the model is what it makes visible as the code grows, and what it *prevents* while you reason. Addressability works two ways off the same mechanism:

- **Prevention (ex-ante)**: while you trace a code-thread, the addresses anchor the trace to where the seams actually are, so reasoning in natural language at higher abstraction drifts less than it would unanchored. Each hop's address is the thing you're reasoning over, not a paraphrase of it.
- **Detection (ex-post)**: when the code changes, drift becomes structural and visible cheaply, because seams and ADRs are addressable:

  - **A seam that moved but its ADR didn't follow**: the implementation crossed a new boundary, or relocated an existing one, and the recorded decision no longer describes where the glue is.
  - **An ADR whose seam no longer exists**: the joint it decided was refactored away; the ADR is now a record of a decision about nothing.
  - **A seam with no ADR**: a joint that became expensive-to-reverse without anyone recording why. This is exactly the place a future reader will wonder "why on earth is it this way?" with no answer.

These are not two benefits. They are the same addressability (seams you can point at, ADRs attached to them) working before and after the change. Fusing them is the point: the addresses that let you trace at higher abstraction are the addresses that make drift cheap to catch.

The cycle's existing documentation reconciliation (`tdd` step 6, `to-changelog`) sharpens under this model. "Did this slice move a seam?" is the question that determines whether an ADR must move with it. A slice that crosses a package edge without an ADR for that edge is the signal to record one; a slice that relocates an internal section seam is the signal to check whether the section's CONTEXT.md entry still describes it.

## Conventions

- **One external seam per package.** The external-seam module *is* the package. The CONTEXT-MAP dependency diagram is the package diagram. One boundary system, not two — package-boundaries and seam-boundaries are the same axis.
- **Sections are declared in CONTEXT.md.** The named concepts in a package's Language section *are* its logical sections. Adding a section is a CONTEXT.md edit; removing one is too.
- **Every line of code belongs to a module.** Sections are the *named, addressable subset* of modules, the ones you can point at. Shared helpers within a package are modules that are not sections; they are covered by "belongs to a module," not orphaned, and not forced into a section they don't deserve. The invariant is "every LOC belongs to a module," not "every LOC belongs to a section."
- **ADRs attach to seams.** An ADR records a decision about a specific joint. When a slice moves that joint, the ADR moves with it or is marked superseded.
