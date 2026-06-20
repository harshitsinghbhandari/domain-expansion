---
name: frontend-architecture
description: Use when scaffolding a new web frontend or making any frontend architecture decision for a React web app (choosing a data-fetching layer, router, state management, styling system, sync/offline strategy, real-time transport like WebSocket/SSE, type generation, or deploy target). Triggers on greenfield "set up the app" work and mid-project structural choices alike (e.g. "how should we manage state", "what router", "add offline mode", "wire up the backend types").
---

# Frontend Architecture

## Overview

Opinionated, blessed-default guidance for architecting the frontend of a web app you intend to scale. The bet: a handful of decisions made correctly on day 1 cost almost nothing up front and save a future engineer from tedious, low-value rework later. You don't have to learn each tool deeply (you direct agents to apply these defaults), but the *decisions* still have to be made deliberately.

**Core principle:** Pick the blessed default below unless the user has a specific reason to deviate. These are not a menu of equal options; they are recommendations with a track record. State the pick, apply it, and only surface alternatives when the project's constraints genuinely break the default.

## When to Use

- Scaffolding a new React web app from scratch.
- Any structural decision mid-project: data fetching, routing, state, styling, real-time transport, deploy target, type generation, query-param state.
- The user asks "how should we do X" where X is a frontend architecture concern.
- Reviewing or correcting an existing frontend's architecture.

**When NOT to use:**
- Single-component or single-bug work that doesn't touch architecture.
- Non-React stacks (the specific tool picks here assume React; the *principles* still transfer, but flag that the picks are React-centric).
- Backend-only or infra-only tasks.

## The Blessed Stack (Quick Reference)

| Decision | Blessed default | One-line why |
|---|---|---|
| Backend types | Generate a client from an OpenAPI spec | Hand-typing backend types is banned. The spec is the source of truth. |
| Server state / data fetching | TanStack Query | Caching, invalidation, and async state, solved. Other libs look similar but this one is the standard. |
| Routing | TanStack Router, or React Router framework mode | Type-safe routes + data loaders beat hand-rolled route trees. |
| Route data | Route data loaders | Load in the route, not in component effects. |
| Query-param state | nuqs (or TanStack), type-safe | If state lives in the URL, make it a first-class, typed citizen. |
| Server state mgmt | TanStack Query (one solution, that's it) | Most apps need exactly one server-state layer and nothing more. |
| Bespoke client state | Zustand | Simple, unopinionated, scales for the rare local-state need. |
| Heavy interactive state | XState / xstate-store | Model the frontend as a state machine to escape useEffect hell. |
| Memoization | React Compiler (drop manual useMemo/useCallback) | The compiler handles it. Update your priors. |
| Loading / error UI | Suspense + ErrorBoundary | `isPending`/`isError` sprinkled everywhere is madness. |
| Real-time transport | One blessed WebSocket/SSE path, chosen day 1 | Especially for anything AI. Decide the pattern before you need it. |
| Styling | A design system / component library, not raw Tailwind sprawl | Tailwind is fun but hard to keep consistent at scale; you want an "agent-first" component layer. |
| Deploy | Cloudflare or Vercel | Other hosts have weird missing features. |
| SPA + Next.js | Don't | A pure SPA on Next.js makes no sense. |

## The Decisions In Detail

### 1. Generate types from the backend, never hand-type them
Make the server emit an OpenAPI spec, then codegen all client-side types and request code from it. Typing backend types by hand should be treated as banned: it drifts, it rots, and it produces subtle bugs at the boundary. The generated client is the contract.

### 2. Decide how the client talks to the backend, deliberately
For REST or GraphQL, use TanStack Query. It is the goated default for server-state: caching, background refetch, invalidation, and request dedup come for free. Competing libraries look superficially similar; pick TanStack Query unless there's a concrete reason not to.

### 3. If you want sync or offline, architect it on day 1
Linear-style local-first sync and offline mode are brutal to bolt on later. If the product needs them, design for them from the very first commit. This is the single decision most likely to force a painful rewrite if deferred. Think about it HARD before scaffolding.

### 4. Routing: modern router + data loaders
Plain React Router still works, but the ecosystem has moved on. Use TanStack Router or React Router's framework mode, and lean on route data loaders so data fetching is declared at the route, not buried in component effects.

### 5. Query-param state is real state, so make it typed
If you keep meaningful state in query params (filters, tabs, selections), treat it as a first-class, type-safe citizen with nuqs or TanStack. Stringly-typed URL parsing scattered through components is a maintenance trap.

### 6. One server-state solution; reach for more only when needed
Most apps need a single server-state layer (TanStack Query) and nothing else. For genuine bespoke local needs, Zustand is the clean default. For a highly interactive app (lots of transient UI state, media playing, things entering and leaving view) lock in and model it as a state machine with XState. Trust the state-machine framing here; it is how you keep sanity and stay out of useEffect hell.

### 7. React Compiler changes your defaults
The React Compiler is here. Stop reflexively wrapping things in `useMemo`/`useCallback`. Write straightforward code and let the compiler memoize. Update your priors accordingly.

### 8. Loading and error states: Suspense + ErrorBoundary
Threading `isPending` and `isError` through every component is madness. Lean into Suspense for loading and ErrorBoundary for failures so the UI states are structural, not manually plumbed.

### 9. Pick a blessed real-time path early
For anything AI-related (and most live apps), choose one canonical pattern for WebSockets and SSE on day 1 and route everything through it. Deciding this once, early, pays dividends; discovering you have three ad-hoc socket patterns later does not.

### 10. Styling: a design system, not Tailwind sprawl
Tailwind is easy and fun but makes consistent styling hard to maintain across a large app. Invest in a design system / component library (an "agent-first" component layer that agents can compose predictably) rather than scattering utility classes everywhere.

### 11. Bend the router to the product
Don't be afraid to hack your routing library to fit the product. Drawers showing extra info are everywhere; you should be able to say "here's a route, render it as a drawer" and have everything handled from there. The router serves the UX, not the other way around.

### 12. Deploy and framework choices
Deploy on Cloudflare or Vercel; other services tend to have weird missing features. And if you're building a pure SPA, do not use Next.js; it adds machinery that buys you nothing for that shape of app.

## The Meta-Principle: Build the Factory

Once you've built something people want, the next job is to build the factory that builds the thing efficiently. Every decision above is partly in service of that: generated types, typed routes, a real design system, and a state model an agent can reason about all make the codebase a place where agents (and humans) ship fast and safely. Architect for the factory, not just the first feature.

## Common Mistakes

- **Hand-typing backend types** "to move fast." It is slower within a week. Generate from the spec.
- **Deferring sync/offline** until users ask. By then it's a rewrite. Decide on day 1.
- **Sprinkling `isPending`/`isError`** instead of using Suspense + ErrorBoundary.
- **Reaching for Redux/extra state libs** when TanStack Query already covers server state.
- **Reflexive `useMemo`/`useCallback`** after React Compiler exists.
- **Raw Tailwind everywhere** with no component layer, then fighting inconsistency at scale.
- **Next.js for a pure SPA**, inheriting SSR machinery you never use.
- **Ad-hoc WebSocket/SSE code** per feature instead of one blessed transport.

## How To Apply

When this skill fires, don't dump the whole table. Identify which decision(s) the task touches, state the blessed default for those, apply it, and note the "why" in one line. Only present alternatives if the project's constraints actually break the default (e.g. a non-React stack, a hard requirement the default can't meet). Keep the opinions: these are recommendations with a track record, not a neutral survey.
