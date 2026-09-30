# Frontend Architecture

This document describes the frontend architecture of Toremon App and the architectural
decisions behind it: which layers exist, what each one is responsible for, and where a given
piece of logic belongs.

It is written to be useful as an architecture-learning reference, so it explains the reasoning
behind each boundary rather than only stating the rule.

## Related documents

- [REQUIREMENTS.md](./REQUIREMENTS.md) — the product rules the application follows.
- [DATABASE.md](./DATABASE.md) — the database structure behind the application.
- [TECH_STACK.md](./TECH_STACK.md) — the technology choices.
- [API.md](./API.md) — the REST API contract.

This document does not repeat those. Business rules live in `REQUIREMENTS.md`, endpoints and
error formats live in `API.md`, and the reasoning behind React, TypeScript, Vite, Zustand, and
TanStack Query lives in `TECH_STACK.md`. Here we cover only how the frontend is organized.

---

## Architecture style

The frontend uses a **layer-based architecture**. Code is grouped by *what kind of job it
does* — UI, server communication, server state, client state, pure logic — rather than by
feature.

The practical benefit is that there is one obvious answer to "where does this go?" for any
piece of code, and each layer can be understood without reading the others.

### Planned structure

```text
src/
├── pages/                 → route-level screens
│
├── components/            → UI components
│   ├── common/
│   ├── auth/
│   ├── task-lists/
│   └── tasks/
│
├── hooks/                 → reusable React behavior
│
├── services/              → REST API communication
│
├── queries/               → TanStack Query logic
│
├── stores/                → Zustand / client state
│
├── types/                 → shared TypeScript types
│
├── utils/                 → pure helper functions
│
└── lib/                   → third-party library configuration
```

---

## Layer responsibilities

### `pages/`

Route-level page components — one per screen in the application.

A page's job is to **compose**: it arranges components and connects the screen to the
application logic it needs. It is the place where "this screen shows a task list and its
tasks" is expressed.

Pages are not a home for reusable UI. If a piece of UI could appear on another screen, it
belongs in `components/`. Letting reusable markup accumulate in `pages/` is the most common
way a layer-based structure erodes, because the next developer needing that UI either imports
from a page (creating an awkward dependency) or copies it.

> `TODO: Decision required` — no routing library has been chosen yet. See
> [Open decisions](#open-decisions). The `pages/` layer assumes route-level screens exist,
> but how routes are declared is not yet decided.

### `components/`

UI components, grouped into four folders:

```text
components/common/
components/auth/
components/task-lists/
components/tasks/
```

`common/` holds **genuinely shared** UI — pieces used across unrelated areas of the app. The
qualifier matters. A component belongs in `common/` because it is actually used in more than
one area, not because it looks generic or might be reused one day.

The other three folders hold components specific to their area: authentication UI, task list
UI, and task UI respectively. These names match the domains used throughout the rest of the
architecture, including `queries/`.

### `hooks/`

Reusable React behavior, as custom hooks.

A hook belongs here when it is **reusable independently of any one page** — behavior that is
React-specific (it uses state, effects, refs, or context) and is needed in more than one
place.

This folder is not a general dumping ground for application logic. Logic with no React
involvement belongs in `utils/`; server communication belongs in `services/`; server state
belongs in `queries/`. A `hooks/` folder that collects everything not obviously belonging
elsewhere quickly becomes the least understandable part of a codebase.

### `services/`

REST API communication with the Laravel backend. This is the only layer that knows how to
talk to the API.

Organized by domain, conceptually:

```text
task-lists.service.ts
tasks.service.ts
auth.service.ts
```

Services describe **operations**, named after what they do:

```text
getTaskLists()
createTaskList()
deleteTaskList()
```

A service function sends a request to the endpoint documented in [API.md](./API.md) and
returns the result. It attaches what the request requires, handles the response, and nothing
more.

**Services do not handle TanStack Query caching, invalidation, or any React-specific
behavior.** They contain no hooks and no component awareness. That restriction is what makes
them plain, testable functions: a service can be called from anywhere, and understanding it
requires no knowledge of React's rendering model.

> `TODO: Decision required` — the HTTP client used inside this layer (for example the native
> `fetch` API) has not been decided. See [Open decisions](#open-decisions).

### `queries/`

TanStack Query logic. This layer owns **server state**: data that lives on the backend and is
mirrored on the client.

Organized by domain:

```text
queries/
├── auth/
├── task-lists/
└── tasks/
```

It exposes query and mutation hooks:

```text
useTaskLists()
useCreateTaskList()
useDeleteTaskList()
```

These hooks are what components actually call. They provide the cached data along with its
loading and error state, and they invalidate the cache after a mutation so the UI reflects the
new server state.

#### The separation

```text
Component
    ↓
TanStack Query hook
    ↓
Service
    ↓
REST API
```

**The `queries/` layer calls functions from `services/` rather than making HTTP requests
itself.** Each layer has exactly one concern: the service knows *how to reach the API*, and
the query hook knows *how that data should be cached, refetched, and invalidated*.

Keeping them apart pays off in both directions. An endpoint change touches only the service.
A caching change touches only the query hook. And because services contain no React code,
the part of the system that talks to the network can be reasoned about on its own.

### `stores/`

Zustand stores, holding **client state**.

```text
TanStack Query → server state
Zustand        → client state
```

Client state is state the backend does not own. Examples include:

- selected UI state
- modal state
- sidebar state
- UI preferences

One store is already implied by an existing decision: [API.md](./API.md) specifies that the
access token is held in frontend memory / Zustand and never in `localStorage`.

**Do not put server-fetched task or task list data into Zustand just because several
components need it.** Being needed in multiple places does not make data client state. Task
lists and tasks come from the server, so they belong in TanStack Query, which is already
shared across every component that calls the same query hook — no store needed.

**Zustand is not a replacement for TanStack Query.** Copying server data into a store means
reimplementing caching, refetching, invalidation, and loading and error states by hand, and
taking on the risk that the copy drifts out of sync with the server.

### `types/`

Shared TypeScript types representing application and domain concepts.

A type belongs here when it is a **common concept** used across layers — the shape of a task
list or a task, for instance, which appears in services, queries, and components alike.
Defining it once means that when it changes, the compiler points at every place that needs
updating.

Avoid duplicating types across pages, components, services, and queries. Duplicated types
drift apart silently: two definitions of the same concept stop matching, and TypeScript cannot
warn you, because as far as it knows they were never the same type.

Types that are genuinely local — the props of a single component, for example — can stay with
that component. `types/` is for shared concepts, not for every type in the codebase.

### `utils/`

Pure helper functions: given the same input, always the same output, with no side effects.

**Utilities contain no React-specific behavior and no API communication.** No hooks, no
component state, no requests. A utility should be callable from anywhere and understandable in
isolation.

Progress calculation is the natural example, since [REQUIREMENTS.md](./REQUIREMENTS.md)
defines it as derived data computed from completed and total task counts. See
[Open decisions](#open-decisions) regarding where that calculation happens.

### `lib/`

Configuration for third-party libraries — setup code rather than application logic.

```text
lib/queryClient.ts
```

This holds the global TanStack Query `QueryClient` configuration.

**Do not create a separate `tanstack/` folder.** See the next section.

---

## TanStack Query architecture

**Decision: we will not create a generic `tanstack/` directory.**

TanStack Query work is split across three places according to the *kind* of work it is:

| Location | Responsibility |
| --- | --- |
| `lib/queryClient.ts` | Global `QueryClient` configuration |
| `queries/` | Query and mutation hooks, organized by domain |
| `services/` | The actual REST API communication |

### Why these responsibilities are separated

A `tanstack/` folder would group code by **which library it uses** rather than by what it
does. That sounds tidy, but it produces a folder containing three unrelated kinds of code —
one-time global setup, per-domain data-fetching hooks, and network calls — while splitting
related code apart. Task list query hooks would sit next to unrelated client configuration,
and away from everything else about task lists.

Organizing by responsibility instead gives each piece a single reason to change:

- **`lib/queryClient.ts`** is configuration. It is written once, changes rarely, and affects
  the whole app. Keeping it in `lib/` alongside other third-party setup reflects that.
- **`queries/`** is where caching decisions live — query keys, invalidation, and the shape of
  each hook. These change when data requirements change, per domain.
- **`services/`** is where the API contract lives. These change when [API.md](./API.md)
  changes, and they change for reasons that have nothing to do with caching.

There is a further benefit: the library boundary stays thin. TanStack Query appears in
`queries/` and in one configuration file, and nowhere else. Components call hooks, services
call the API, and neither layer is coupled to the caching library.

---

## State management decision

The central rule of this architecture:

```text
TanStack Query → server state
Zustand        → client state
```

### Server state

Use **TanStack Query** for anything that comes from the backend:

- task lists
- tasks
- authenticated-user data
- other API resources

TanStack Query handles:

- fetching
- caching
- loading states
- error states
- refetching
- mutations
- cache invalidation

Server state has properties client state does not: it is asynchronous, it is shared with other
clients, and it can become stale without anything happening locally. Those properties are what
the library exists to manage.

### Client state

Use **Zustand** for UI and application state the server does not own — the kinds of state
listed under [`stores/`](#stores).

### Do not duplicate

**Do not duplicate server state between TanStack Query and Zustand without a clear
architectural reason.** Two copies of the same data means two things that can disagree, and
nothing in the type system will catch it. When in doubt, ask where the data's authoritative
version lives: if the answer is the backend, it is server state.

---

## Dependency direction

Dependencies flow in one direction:

```text
Pages
  ↓
Components / Hooks
  ↓
Queries
  ↓
Services
  ↓
REST API
```

Each layer depends on the one below it and is unaware of the one above. Pages know about
components; components do not know which page renders them. Queries know about services;
services do not know they are being called by a query hook.

```text
Stores
```

Stores hold client state and are read by components, pages, and hooks as needed. **Stores are
not a replacement for the query layer** — server data flows through `queries/`, not through a
store.

`types/` and `utils/` sit outside this flow. Both are dependency-free leaves: any layer may
use them, and neither depends on anything above it.

**Avoid unnecessary circular dependencies.** A cycle means neither layer can be understood or
changed on its own, and it is usually a sign that something is in the wrong place. If a
service needs something from a query hook, the dependency is pointing the wrong way.

---

## Architecture principles

1. **Separation of concerns.** Each piece of code does one kind of job.
2. **Clear responsibility for each layer.** Every layer has a single answer to "what belongs
   here?"
3. **Server state and client state are handled separately.** TanStack Query for one, Zustand
   for the other.
4. **API communication is separated from TanStack Query.** Services talk to the API; queries
   decide how the results are cached.
5. **Reusable UI belongs in components**, not in pages.
6. **Pure logic belongs in utils** — no React, no network.
7. **Reusable React behavior belongs in hooks.**
8. **Pages compose the application** rather than containing all business logic.
9. **Avoid premature abstraction.** Build the abstraction when a second real use appears, not
   in anticipation of one. A wrong abstraction is harder to undo than a little duplication.
10. **Prefer simple architecture that is easy to understand and maintain.** These layers exist
    to make the code easier to navigate. If a rule ever makes the code harder to follow, that
    is worth revisiting rather than working around.

---

## Open decisions

These are genuinely unresolved in the current project documentation. They are recorded here so
they are decided deliberately rather than by accident.

| Decision | Status | Notes |
| --- | --- | --- |
| Routing library | `TODO: Decision required` | `pages/` holds route-level screens, but no router has been chosen in [TECH_STACK.md](./TECH_STACK.md) or elsewhere. |
| HTTP client for `services/` | `TODO: Decision required` | Whether services use the native `fetch` API or another client is undecided. No HTTP client appears in [TECH_STACK.md](./TECH_STACK.md). |
| Where progress is calculated | `TODO: Decision required` | [API.md](./API.md) states that progress is derived and not stored, but leaves open whether the API returns it or the frontend computes it from task data. This determines whether a progress helper lives in `utils/`. |
| Where the token-refresh retry lives | `TODO: Decision required` | [API.md](./API.md) defines the refresh flow — a `401` triggers `/api/auth/refresh`, then the original request is retried — but not which frontend layer implements that retry. |
| Styling approach | `TODO: Decision required` | No styling or component library is recorded in [TECH_STACK.md](./TECH_STACK.md). |

Anything not listed here and not documented in the related documents should be treated as
undecided rather than assumed.
