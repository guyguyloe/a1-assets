# Trace Report: {{ROOT_COMPONENT}}

> Low-fidelity, single-root trace. Every hop below is reachable from the root and
> only from the root's concern. Shared utilities touched en route are logged in
> the Out-of-Scope annex, not traced further.

**Repository:** {{repo path / ref}}  **Generated:** {{date}}
**Root file:** {{path:line}}

---

## 0. Root component

- **Identity:** {{component name}}  ({{path}})
- **Responsibility (one line):** {{what it owns}}
- **Rendered shape:** {{box sketch — see adjacency below}}
- **Inputs (props / params):** {{list}}
- **Local state:** {{keys + types}}
- **Outbound channels:** {{events dispatched / callbacks invoked / actions dispatched}}

### Adjacency sketch

```
{{ASCII wireframe of the root's own surface — its child boxes and labeled
  event arrows. Low fidelity: boxes, labels, arrows only.}}
```

---

## 1. Events originating from the root

For each event the root emits or binds:

| # | Event / trigger | Source element | Bound handler | Evidence |
|---|---|---|---|---|
| 1.1 | {{onClick "submit"}} | {{<button>}} | {{handleSubmit}} | {{path:line}} |
| 1.2 | … | | | |

---

## 2. Handlers

For each handler reached in §1:

- **Handler:** {{name}} — {{path:line}}
- **Called by:** {{events from §1}}
- **Responsibility:** {{one line}}
- **Calls (downstream):** {{functions / actions / API fns}}
- **Side effects:** {{navigation, focus, logging, etc.}}
- **Branched traces:** → {{link to §3 / §4 section for each call}}

---

## 3. Backend logic

For each call that crosses the process boundary (or enters a store/service layer):

- **Boundary fn:** {{name}} — {{path:line}}
- **Transport:** {{REST / GraphQL / tRPC / RPC / local service}}
- **Request shape:** {{payload}}
- **Server endpoint / resolver:** {{path:line}}
- **Server logic chain:** {{fn → fn → fn, with path:line each}}
- **Persistence touched:** {{table / collection / cache key}}
- **Branched traces:** → {{link to §4 for response handling}}

---

## 4. Response → state change

For each response from §3:

- **Handler of response:** {{name}} — {{path:line}}
- **State store:** {{global store / context / local state / cache}}
- **Mutation:** {{field(s) changed, before → after}}
- **Mechanism:** {{dispatch / setter / updater / reducer action}}
- **Evidence:** {{path:line}}
- **Branched traces:** → {{link to §5 for each affected component}}

---

## 5. Affected components

Components that read the mutated state and re-render as a consequence:

| # | Component | File | State slice read | Re-render trigger | Evidence |
|---|---|---|---|---|---|
| 5.1 | {{name}} | {{path}} | {{selector / prop}} | {{subscription / hook / prop drill}} | {{path:line}} |
| 5.2 | … | | | | |

> Each row must be traceable back to the root via §0→§1→§2→§3→§4. If a row
> cannot be chained back to the root, move it to Out-of-Scope.

---

## End-to-end path (condensed)

```
{{root}} --(event)--> {{handler}} --(call)--> {{endpoint}}
  --> {{mutation}} --> re-renders: {{A, B, C}}
```

---

## Out-of-scope / shared (annex)

Shared utilities, global modules, or components touched but **not** owned by
this trace's root. Recorded for honesty, not traced:

| Item | Touched at | Why not traced |
|---|---|---|
| {{util formatDate}} | {{path:line}} | pure helper, no downstream state |
| {{<SharedModal>}} | {{path:line}} | mounted by sibling root, not this one |

---

## Open questions / assumptions

- {{any hop that could not be confirmed statically; e.g. "handler is dynamic dispatch — assumed the default reducer branch"}}
