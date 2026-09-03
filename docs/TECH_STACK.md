# Tech Stack

This document records the confirmed technology decisions for Toremon App, a personal
task management application (authentication, task lists, tasks, completion, progress tracking).

For each technology it describes what it is used for, what it is responsible for, and
why it was chosen. This stack is fixed; changes should be made by updating this document
first.

## Architecture at a Glance

The application is split into two independently deployable parts that live in this
repository:

- `frontend/` — a React single-page application served as static assets.
- `backend/` — a Laravel application exposing a REST API over HTTP.

The frontend never talks to the database directly. All reads and writes go through the
REST API, which owns authentication, authorization, validation, and persistence.

---



## Frontend



### React 19

**Used for:** Building the entire user interface — the login screen, task list overview, task
views, and the progress indicators.

**Responsibility:** Rendering the UI and reacting to user interaction. React owns the
component tree, local UI state (open dialogs, form inputs, in-progress edits), and the
composition of screens from reusable components.

**Why we chose it:** React is a mature component-based library that fits the application's
 interactive UI and gives us a strong ecosystem.

### TypeScript

**Used for:** All frontend source code — components, hooks, stores, and the typed models
that mirror the API's request and response payloads.

**Responsibility:** Enforcing a compile-time contract across the codebase. TypeScript is
the single source of truth for the shape of a `Task`, a `TaskList`, and a `User` on the client,
and it guards the boundaries where data enters the app from the REST API.

**Why we chose it:** A task app is full of small, similar-looking objects, and type errors
there are easy to make and annoying to debug at runtime. Static types catch mistakes such
as a renamed field or a missing null check during development rather than in the browser.
The editor support (autocomplete, safe renames, go-to-definition) also makes refactoring
cheap, which matters while the feature set is still growing.

### Vite

**Used for:** The frontend development server and the production build pipeline.

**Responsibility:** Serving modules during development with hot module replacement,
resolving imports, applying TypeScript and JSX transforms, and producing the optimized,
bundled, and hashed static assets that get deployed.

**Why we chose it:** Vite gives near-instant server start and fast HMR because it serves
native ES modules in development instead of bundling the whole app on every change. Its
defaults for a React and TypeScript project require almost no configuration, and it uses
Rollup for production builds, so we get good output (code splitting, tree shaking) without
maintaining a custom build setup.

### Zustand

**Used for:** Client-side state that is not owned by the server — UI preferences such as the active task list or current filter (all / active / completed), and any cross-component view state.

**Responsibility:** Holding global client state and exposing it to components through selector-based subscriptions, so that components subscribe only to the state they need.

**Why we chose it:** We need a small amount of shared client-side state, and a full Flux-style state management solution would add more ceremony than the problem warrants. Zustand is a small, unopinionated store with a hook-based API, no provider wrapper required, and minimal boilerplate. Its selector-based subscriptions help keep component updates focused, and stores can also be accessed outside React when needed.

### TanStack Query

**Used for:** Every interaction with the REST API — fetching task lists and tasks, creating and
updating them, and toggling completion.

**Responsibility:** Owning all server state on the client. It handles caching, request
deduplication, background refetching, stale-while-revalidate behavior, loading and error
states, cache invalidation after mutations, and optimistic updates for actions such as
checking off a task.

**Why we chose it:** Server data is fundamentally different from client state — it is
asynchronous, shared, and can go stale — and hand-writing that logic in effects leads to
duplicated fetch code, race conditions, and inconsistent loading states. TanStack Query
solves this as a dedicated layer, and it pairs deliberately with Zustand: TanStack Query
holds anything that came from the server, Zustand holds anything that did not. That split
keeps both stores small and keeps us from caching API responses by hand.

---



## Backend



### Laravel

**Used for:** The API application — routing, request validation, authentication, business
rules, and database access.

**Responsibility:** Being the authoritative layer of the system. Laravel defines the REST
endpoints, validates and authorizes every incoming request, enforces that users can only
reach their own task lists and tasks, applies domain rules, persists data through Eloquent, and
manages the database schema through migrations.

**Why we chose it:** Laravel ships with the pieces this application needs already built and
integrated — routing, validation, an ORM, migrations and seeding, authentication scaffolding,
queues, and a strong testing story. That means the work goes into the task domain rather than
into assembling infrastructure. Its conventions are well established, so the codebase stays
predictable, and its documentation and ecosystem are excellent.

### PHP 8.2+

**Used for:** The language the entire backend is written in.

**Responsibility:** Executing the server-side application — the runtime that Laravel and all
domain code run on.

**Why we chose it:** The backend runtime. The exact minimum version will follow
the selected Laravel version.

### MySQL 8+

**Used for:** Persistent storage of all application data — users, task lists, tasks, and their
completion state.

**Responsibility:** Durably storing data and guaranteeing its integrity. MySQL enforces
relationships between users, task lists, and tasks through foreign keys, applies uniqueness and
other constraints, and provides transactional guarantees so that multi-step writes either
fully succeed or fully roll back.

**Why we chose it:** The data here is clearly relational — a user has many task lists, and a
task list has many tasks — so a relational database is the natural fit, and progress tracking is expressed
directly as aggregate queries over tasks.

---



## API



### REST API

**Used for:** The contract between the frontend and the backend. All communication between
the React app and Laravel happens over REST endpoints exchanging JSON.

**Responsibility:** Defining the boundary of the system — resource-oriented URLs for task lists
and tasks, HTTP verbs mapped to operations (`GET` to read, `POST` to create, `PATCH`/`PUT` to update,
`DELETE` to remove), meaningful status codes, and a consistent JSON envelope for both successful
responses and errors.

**Why we chose it:** The data model is a small set of well-defined resources with predictable
access patterns, which is exactly what REST handles well; a query language such as GraphQL would
add a schema layer and client tooling we do not need at this size. REST also fits the rest of the
stack directly: Laravel's resource routing and API resources produce these endpoints idiomatically,
and TanStack Query's cache keys map cleanly onto resource URLs. Being plain HTTP and JSON, it is
trivial to inspect in browser devtools, exercise with curl, and consume from any future client.

---



## Summary


| Layer    | Technology     | Owns                                      |
| -------- | -------------- | ----------------------------------------- |
| Frontend | React 19       | UI rendering and component state          |
| Frontend | TypeScript     | Compile-time type safety                  |
| Frontend | Vite           | Dev server and production build           |
| Frontend | Zustand        | Client-only global state                  |
| Frontend | TanStack Query | Server state, caching, mutations          |
| Backend  | Laravel        | Routing, validation, auth, business rules |
| Backend  | PHP 8.2+       | Server-side runtime                       |
| Backend  | MySQL 8+       | Persistence and data integrity            |
| API      | REST           | Frontend–backend contract                 |


