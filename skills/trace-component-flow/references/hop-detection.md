# Hop-Detection Cheat-Sheet

Stack-agnostic heuristics for recognizing each hop in the trace chain so the
report stays faithful without assuming a specific framework.

## Table of contents
1. [Component → event](#component--event)
2. [Event → handler](#event--handler)
3. [Handler → backend boundary](#handler--backend-boundary)
4. [Backend → response](#backend--response)
5. [Response → state mutation](#response--state-mutation)
6. [State → affected components](#state--affected-components)
7. [When to stop](#when-to-stop)

---

## Component → event

Look for the component's outbound surface — what it emits or binds.

- DOM/JSX/Vue/Svelte: `onClick`, `@click`, `on:click`, `onTap`, inline `dispatch`
- Web Components: `CustomEvent` dispatch, `this.dispatchEvent`
- Angular: `(click)` template binding, `@Output()`
- Imperative widgets: `element.addEventListener`, jQuery `.on()`
- Form submits: `<form onSubmit>`, `submit` listeners

Record: event name, source element, the handler symbol it points to.

## Event → handler

Resolve the bound symbol to its definition. Watch for:

- Inline arrow → follow the body
- Method reference (`this.handleSubmit`) → class method
- Curried/propagated handler (`onSubmit={props.onSubmit}`) → follow the prop
  upward one level; mark "prop-drilled from {{parent}}" and continue from there
- Action dispatch (`dispatch({ type })`) → jump to the reducer/action creator
  (this is a handler-to-state shortcut; record both the dispatch site and the
  reducer)

## Handler → backend boundary

A hop crosses the boundary when control leaves the rendering process. Signals:

- `fetch`, `axios`, `XHR`, `$.ajax`, `apolloClient.mutate`, `useQuery`/`useMutation`
- `trpc.thing.mutate`, `rpc.thing(...)`, gRPC client stubs
- WebSocket/SSE send
- Server actions (`"use server"`), Remix `action`, loaders
- Store thunk/saga that calls an API
- Direct DB/ORM calls if the component lives server-side (SSR component, RSC)

Record transport, payload, and the endpoint/resolver symbol.

## Backend → response

Follow the endpoint to its return. Signals for "this is the response":

- Controller returns / resolver returns / loader returns
- `res.json(...)`, `reply.send(...)`, `return data`
- Error path: thrown exception, rejection, error response code

Record both the success shape and the error shape; both mutate state downstream.

## Response → state mutation

Where the response is consumed and written to state. Signals:

- `setState` / `useState` setter, `this.setState`, `useState` updater
- `dispatch` with response payload → reducer/action
- Store mutation: Pinia `state.x =`, Zustand `set`, Redux reducer, Vuex commit
- Context value replacement
- Cache writes: SWR `mutate`, React Query `setQueryData`, Apollo cache `writeQuery`
- Optimistic updates (record separately — they precede the response)

Record: store, field(s), mechanism, evidence.

## State → affected components

Find components that read the mutated slice. Signals:

- `useSelector`, `mapStateToProps`, `connect`, `useStore`
- `useContext`, `inject`, `provide`/`inject` consumers
- `useQuery`/`useSWR` keyed on the same cache key
- Prop-drilled state: follow the provider that passes the state down
- Subscribe patterns: `store.subscribe`, `watch`, `computed`

Record: component, file, slice read, subscription mechanism, evidence.

## When to stop

Stop tracing a branch when you hit any of:

- A **pure helper** with no state, IO, or side effects (formatDate, pick, memo).
- A **shared/global module** whose other consumers belong to a different root.
  Log it in Out-of-Scope and resume only this root's branch.
- A **rendering leaf** that reads state but emits nothing back into the trace.
- An **external boundary** (third-party SDK, browser API) — record the call,
  do not trace inside.

Faithfulness rule: never follow a branch whose origin is a different root. If
the root's handler calls a shared service that 5 other roots also call, you
record "shared service touched" and trace only the response as it returns to
this root's state — not the service's other callers.
