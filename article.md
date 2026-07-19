# I Stopped Organizing Code by Folders. Here's What Replaced It.

AI makes shipping faster than ever, but it also makes a codebase feel more alien the moment you step away from it. Time away makes the parts I never fully understood a little more daunting, and plants doubt on the parts I thought I did. Docs help, but they easily descend into "markdown hell."

After struggling to explain a UI bug to my agent, I ended up building a small tracing skill — and accidentally landed on a cleaner way to think about code organization. This is that story.

## Background

I asked @platput how he handles it. He documents his code in sections: "I write/generate docs for each section or module or class." The idea of "sections" caught my attention, so I went back and forth with my Pi exploring the idea in my own setup.

How I organize my repos is inspired by the @pidotdev repo — from a glance it just looked right, so I adopted the structure without much thought. That was after I'd already been through the generic advice of organizing into folders, which neglects a nuance: a folder is a label, and a label doesn't do much for code organization unless it's well thought out... which I couldn't really do, being relatively new to SWE, with AI essentially writing all of my code.

## The concept I already had

I've had this loose concept of a "code-thread" for a while: a path through the code from a component to the backend and back. A while back I had a UI bug I couldn't quite articulate to the agent, so I had Pi write a skill for it: `trace-component-flow`.

It follows a single UI root's full downstream path (component → event → handler → backend → response → state → affected components). That was a thread. I resumed that session and landed on two abstractions: **subsystems** and **logical sections**.

## The initial picture

I framed a subsystem as essentially a claim about coupling. Using the pi repo as an example, `pi-ai/` can be thought of as a subsystem of the whole pi monorepo, but `providers/` inside `pi-ai` is itself a subsystem (each vendor is swappable behind a common interface). That connects back to the interfaces-and-seams idea (picked up from @mattpocockuk), and the picture started becoming clearer:

- Interfaces and seams exist to connect subsystems.
- Within each subsystem we can have **logical sections**: groupings of code with no interface and possibly some shared state.

What's interesting is how this helps with "vibe-debugging" — it generalizes the `trace-component-flow` skill from earlier. Within code we can trace **threads**, and a thread can be described as "starting section + the interface crossings out to the next subsystem's section."

## Reality check against the skills

The skills my Pi runs on carry a fixed vocabulary (adopted from the skills repo by @mattpocockuk). Its canonical definitions:

- **Module**: anything with an interface and an implementation. Deliberately scale-agnostic — it applies equally to a function, class, package, or tier-spanning slice.
- **Interface**: everything a caller must know to use the module correctly, from the type signature to invariants, ordering, error modes, config, and performance.
- **Seam**: where an interface lives. A place behavior can be altered without editing in place. (From Michael Feathers.)
- **Adapter**: a concrete thing that satisfies an interface at a seam.

And one discriminator that matters here: one adapter means a hypothetical seam. Two adapters means a real one.

Checking my two coinages against that vocabulary surfaced an uncomfortable fact.

**A subsystem is a module.** The skills froze **module** as "anything with an interface and an implementation, scale-agnostic: function, class, package, or tier-spanning slice." My "subsystem" doesn't sit beside that definition — it sits inside it. `pi-ai` is a module. `providers/` is a module. `shortHash` is a module.

The skills are explicit that substituting synonyms fights the whole point: *"Use these terms exactly — don't substitute 'component,' 'service,' 'API,' or 'boundary.' Consistent language is the whole point."*

## The rename wasn't the answer

So "subsystem" as a bare rename of "module" is exactly the move the vocabulary was designed to refuse. The honest move was to drop "subsystem" and say **module** — but I was attached to the two-layer model I was building and couldn't just let it go.

After some thought I realized what "module" doesn't express is a *relationship*: **containment**. The flat word has no notion of "this module groups smaller modules into named clusters" — which is a useful anchor when directing the code (i.e. vibe coding).

## The one term that earns its keep

A **logical section** is a named cluster of modules *inside* a module, sharing a purpose, presenting no collective external interface. It's a label over a bag of independent modules — not itself a module.

The discriminator collapses to one question: **does the group present a collective interface, or is it a bag of independent exports that only share a label?**

- Collective interface → **module**.
- Shared label only → **logical section**.

The pi repo shows the difference. `pi-ai`'s public surface (`packages/ai/src/index.ts`) surfaces the two cases differently:

- **`utils/` is surfaced per-file**: `export * from "./utils/diagnostics.ts"`, `…/json-parse.ts`, `…/retry.ts`. Each helper cherry-picked individually, no `export * from "./utils"`. The folder is a label over independent modules — a clear logical section.
- **`providers/` is surfaced as a collective subpath**: the `index.ts` comment declares *"Provider factories live under `@earendil-works/pi-ai/providers/*`,"* and `providers/all.ts` is the registry of every vendor adapter behind one `Provider` interface (`createProvider(...)` in `models.ts`, with every vendor returning `Provider<"...">`). Collective interface → a module.

And `utils/`'s helpers are mostly pure (`shortHash`, `repairJson`, `headersToRecord`), so this particular section doesn't exhibit shared state. The "possibly some shared state" caveat belongs to the general definition, not to this example.

So the reframe:

- Say **module** where I had "subsystem." Scale-agnostic, already covers the package- and folder-sized cases.
- Keep **logical section** as the one genuinely new term: for the clusters that don't cross a collective interface.

## The unease that stayed

And still, confirming the discriminator didn't resolve the unease. It was correct, but it wasn't the axis that explains *why* a well-organized codebase like pi's feels well-organized. The rename killed the bad justification — "subsystem" as a container-shape category distinct from module — but it left my two-layer model with no foundation.

Both a module and a logical section pass the deletion test. Both earn their keep. So what actually separates a cluster with a collective interface from a cluster with a shared label, if "module" is scale-agnostic and both hide complexity?

The vocabulary had the discriminator and no reason for it. That was the dig.

## The real axis: seam type

I went back to the skills one more time and surfaced the half of the seam definition I'd missed: a seam is where an interface lives, and placing it is its own design decision, distinct from what goes behind it.

The skills already noted that a module can have **internal seams** (private to its implementation) as well as the **external seam** at its interface — but they carried that as a testing-discipline note: don't expose internal seams through the interface just because tests use them. Not as an axis.

This re-grounding promotes it to the axis. The binary isn't container shape, and it isn't containment. It's **seam type**:

- A module whose interface lives at an **external seam** is one thing.
- A module whose interface lives at an **internal seam** — private to its package's implementation — is another.

Here "package" isn't a loose synonym for "module"; it earns a precise seat. The external-seam module is realized as a package, so package boundaries and seam boundaries fuse into one system instead of two. This isn't seam type escaping container shape — it's reusing the container word the repo already has as the carrier for the external value, so a single boundary system runs. And **logical section** is the one coined term: a module whose seam is internal.

That's the whole distinction, and it's the one the rename couldn't produce: the rename nullified a bad justification, but it couldn't produce an addressing scheme. Seam type did. It also retired "subsystem" for good — the external case already had a word.

What makes seam type more than a label is that it's **addressable**. Tag every seam with its layer and structure becomes a query rather than git archaeology. Each seam has an address you can point at, and ADRs can attach to the seam they decided. A decision about a joint is no longer a free-floating paragraph — it has a home at the edge it shaped. And each hop carries its rule:

- Across an **external seam** you assume identity-only references: the two sides don't share state; they cross by model identity and message protocol.
- Inside an **internal seam** you may assume shared implementation context, because siblings can share state and helpers freely.

The layer tag is what lets you reason over the code at higher abstraction without drifting into a paraphrase of it: each hop's address carries the assumption that holds there.

## Threads

This is where the model pays off.

A **thread** is a traversal over addressable seams, not a new container. Point at a named module and pick up the rope by following seams: inside a package you follow internal seams between sections, and at the package edge you cross the external seam to the next one. The two-layer structure makes the whole path addressable:

> starting module + the interface crossings out to the next module's section

That's the same shape my Pi's skills already think in:

- `write-tests` maps "seams where behavior is observable" and hunts for the gap where a test sits at the *wrong* seam.
- `trace-component-flow` follows a single UI root's full downstream path.

Both are threads, already shipped. Both are scoped — `trace-component-flow` to one UI root, forward only, by necessity, because it couldn't generalize.

What I was reaching for was the generalization: any named module over addressable seams, in both directions.

- **Forward** is what it triggers: the path from a module through the events and actions it dispatches to downstream effects.
- **Backward** is the closure of what it needs: every line of code that exists to make the root observable.

The layer tag makes the traversal safe at a higher abstraction, because each hop's address carries the assumption that holds there: shared state inside, identity-only across. [link to full exploration at the end, STRUCTURE.md]

## My confidence inverted

That generalization is **program slicing**, studied since the early 80s. Mark Weiser introduced it in 1981 as a debugging and comprehension aid — my exact use case.

- A *backward slice* of a point is every statement that affects it.
- A *forward slice* is every statement it affects.

My "backward is the closure of what it needs" *is* the backward slice; my "forward is what it triggers" *is* the forward slice. The named module I start from is the *slicing criterion*; the thread is the slice. And the addressable structure I traverse (structure-as-a-query, mappable back to source) is what concern graphs (Robillard & Murphy, 2002) and intensional views (Snelting) already gave the field.

The task motivating it — "I can't articulate the bug so I trace the path" — is *feature location*, originally framed as the *concept assignment problem* (Biggerstaff, 1993).

So the mechanism isn't mine. I re-derived it from first principles, which I'll take as a sign the axis underneath is sound — but the traversal is a rediscovery.

What I think *is* mine is narrower, and it's the synthesis the previous section built to. Three things I'm standing on, none of them mine:

- The **binary** — external seam vs internal seam — is Parnas' information hiding (1972): the interface is the public surface, the implementation hides the secret; from outside, the internal resource is invisible.
- The **seam** itself is Michael Feathers' — a place behavior can be altered without editing in place, coined as a testing and refactoring concept in *Working Effectively with Legacy Code*.
- The **vocabulary** is Matt Pocock's, taught to me through the skills Pi runs on — module, interface, seam, adapter — frozen into a fixed language the agent and I share.

Collectively those give you a public/private binary (Parnas), a named place to alter behavior (Feathers), and a shared language to point at them (Pocock).

What I'm adding is repurposing Feathers' seam from a testing concept into the primary structural axis — making *where the seam sits* the organizer rather than container shape, with a rule attached to each hop (shared state inside a package, identity-only references across one), so the layer tag lets you reason over the code at higher abstraction without paraphrasing it. And aiming the whole scheme at the vibe-coding case, where agents and humans navigate the same addressable structure with that shared vocabulary.

To prove the axis held, I re-entered pi-mono and walked a real thread: a user prompt → the agent loop → the LLM stream, tagging each hop with its seam address. The walk that shows the tags holding *is* a thread. The thread is the proof. [full example on github, PI-MONO.md.]

## Net

- Say **package** for the external case — the container word the repo already uses, now the carrier for an external seam. One boundary system, not two.
- **Logical section** is the one coined term: a module whose seam is internal. The binary is seam type, not container shape; "package" is existing vocabulary reused as the external value of that binary.
- The discriminator (collective interface vs shared label) is the surface check. Seam type is the foundation beneath it. Both a package and a section pass the deletion test, so what separates them is where the seam sits.
- **Threads** traverse addressable seams, each hop carrying its rule — but the traversal is program slicing (Weiser, 1981), and the addressable structure is concern graphs (Robillard & Murphy, 2002) / intensional views (Snelting). I arrived at the general form independently from a scoped, forward-only skill; the mechanism was already there.
- The binary the axis runs on is Parnas' information hiding (1972); the *seam* is Feathers', via @mattpocockuk's skills. What's mine is the repurposing — seam-as-structural-axis, a rule per hop, and the aim: a shared navigation substrate for an AI agent and a human. **Logical section** is the one coined term it leaves behind.
- Minimal docs stay useful, down to the same CONTEXT.md files the agents read and maintain. The named concepts in a package's Language section *are* its logical sections, and ADRs attach to the seams they decided.

What started as friction with context switching served as a learning opportunity to improve how I understand and reason over code in general (WIP). Hopefully this helps someone out there.

How do you currently organize code so your AI agent can navigate it?

Special thanks to @platput, @mattpocockuk, and the @pidotdev team.
