# Seams on pi-mono

This document is the dense exhibit.

[`chronology.md`](chronology.md) narrates the arrival. [`STRUCTURE.md`](STRUCTURE.md) states the model. Neither needs the tree underfoot. This one does.

The exhibit re-enters the same repo that raised the step-8 unease. It does so with the final tool (seam-type), not the intermediate one (collective interface vs shared label).

## Method

The proof is a code-thread. It traces a real path through the packages and tags each hop with a seam address.

That is not a separate method from the claim. The claim is that threads become addressable when seams carry a layer tag. So the argument walks a thread and shows the tags holding.

## Two joints

### A. `pi-agent` → `pi-ai` (external)

**Address:** external · `pi-agent-core` ↔ `pi-ai`

**Contract:** `StreamFn` — `(Model, Context, SimpleStreamOptions?) → AssistantMessageEventStream`, plus the value types that hop with it.

What the tree shows:

- **Package edge.** `@earendil-works/pi-agent-core` depends on `@earendil-works/pi-ai`. Already separate packages.
- **Identity-only.** Agent-loop keeps `AgentMessage[]` inside agent. Conversion to LLM `Message[]` happens only at the call boundary ("Transforms to Message[] only at the LLM call boundary").
- **Real seam.** Default `streamSimple` (compat), harness `models.streamSimple`, `streamProxy`, and injectable `AgentOptions.streamFn`.
- **Separate ownership.** Agent owns state, tools, queues, `AgentMessage`. pi-ai owns models, provider streams, wire protocols. Cross by model identity and message protocol.
- **Core loop does not import `providers/*`.** Production agent code uses `@earendil-works/pi-ai` and `/compat`. Harness takes a `Models` and calls `streamSimple`.

Traversal assumption at this hop: identity-only. No shared agent state across the joint.

Collective-interface alone says "both modules." It does not tag the hop with a rule.

### B. `Models` ↔ `Provider` (internal, real)

**Address:** internal · `Models` ↔ `Provider` adapters

**Contract:** `Provider` (`id`, `auth`, `getModels`, `stream` / `streamSimple`). Defined in `models.ts`. Implemented in `providers/*` via `createProvider`.

What the tree shows:

- **Same package as the client.** `ModelsImpl` and the vendor factories live in `packages/ai`.
- **Shares freely.** Factories pull `api/*`, `auth/*`, `utils/oauth`, `utils/event-stream`, shared types.
- **Many adapters.** Dozens of vendors plus `faux`. Substitutability is exercised.
- **Composition is published.** `exports["./providers/*"]`, `setProvider`, `createModels`. Publishing the composition API publishes the *interface shape*. It does not move the *joint* out of the package.

Outsiders configure adapters behind an internal joint. They still depend on `Models.streamSimple` working, not on the Models↔Provider split being drawn this particular way.

Traversal assumption at this hop: shared implementation context.

Old axis: collective interface → module. Correct and incomplete. Seam-type adds placement and the hop rule.

### Contrast: `utils/` (internal, hypothetical)

No collective interface. Cherry-picked helpers, mostly pure. Folder is a label over independent modules.

Under seam-type this is still internal (private, free sharing). On the adapter-count axis it is mostly hypothetical (0–1).

STRUCTURE.md already allows non-section modules inside a package: every line belongs to a module; not every line belongs to a section.

## One thread, addresses attached

Forward path from a user prompt to an LLM stream:

1. **(internal, agent)** `Agent` → `agent-loop`  
   Shared `AgentMessage` and agent state.

2. **(external)** `convertToLlm` + `streamFn` · agent ↔ pi-ai  
   Identity-only. `AgentMessage[]` becomes `Message[]`. `StreamFn` is the joint.

3. **(internal, pi-ai)** `Models.streamSimple` → `Provider.streamSimple` → `api/*`  
   Shared package implementation. Vendor choice varies behind `Provider`.

Steps 2 and 3 both sit behind collective interfaces. Only seam-type marks them differently. That difference is the hop rule the intermediate discriminator could not give.

## What the exhibit proves

| | real (≥2 adapters) | hypothetical (0–1) |
|---|---|---|
| **external** | `pi-agent` ↔ `pi-ai` | — (indirection, not a package) |
| **internal** | `Models` ↔ `Provider` | `utils/` helpers |

Two axes, not one:

- **Seam-type (placement):** external / internal. This is the addressing scheme. External = identity-only; internal = shared implementation.
- **Seam-reality (adapter count):** real / hypothetical. Feathers' rule. Orthogonal to placement.

External seams *use* the two-adapter rule as justification. A single-adapter external edge is indirection.

Internal seams are justified by legibility. Adapter count is free to be 0, 1, or many.

`Models` ↔ `Provider` is the existence proof of the internal+real cell. Variety is real. The boundary stays private to pi-ai.

That is why step 8's check felt incomplete. The intermediate discriminator classified folders correctly and still could not walk the repo with hop rules. Seam-type can. The walk is a thread. The thread is the proof.

## Package convention

One external seam per package — the external-seam module *is* the package.

Multi-export on pi-ai (`.`, `./compat`, `./providers/*`, `./api/*`) is one external interface with facets, not multiple package edges. Matches "every module has exactly one interface."

## What this document is not

It does not replace STRUCTURE.md.

The abstract rule (internal = private placement; adapter count orthogonal) lives there, without pi-mono names. This document is the tree underfoot for that rule, and the place the step-8 unease is paid off by re-entry.
