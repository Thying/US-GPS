# US-GPS Architecture

**US-GPS** is an architectural approach for realtime applications with clear separation of concerns and unidirectional data movement.

The scheme consists of two independent parts:
- **US** — client side (UI + Store)
- **GPS** — server side (Gate + Pipeline + Services)

Each part can be used separately. GPS is a full-fledged server architecture that works without a US client.

### General Scheme

![US-GPS Architecture](https://github.com/Thying/US-GPS/blob/main/architecture.png)

---

## Key Principles

Layers are isolated and form a unidirectional dependency structure:

```
Gate → Pipeline → Services
```

The upper layer knows nothing about the internal structure of the lower layer. This allows changing the implementation of any layer without affecting the others.

---

## GPS (Server)

GPS is a standalone server architecture. It can be applied independently of US: for example, for API services, webhook handlers, background workers, microservices. The client can be anything — web, mobile, CLI, another service.

### Gate

- **Why:** To separate the protocol (WebSocket/HTTP/CLI) from business logic.
- **What it does:** Accepts requests, checks authorization, validates input data, calls `Pipeline`.
- **What it does NOT do:** Does not contain business logic, does not work with the database, does not send events.

### Pipeline

- **Why:** To orchestrate complex business processes.
- **What it does:** Manages the sequence of steps, calls `Services` to perform work, manages transactions.
- **What it does NOT do:** Does not contain business rules, does not import `Gate`.

### Services

- **Why:** To provide independent, reusable capabilities for working with data, logic, and events.
- **What it does:** `Pipeline` calls these services to perform specific tasks. They do not depend on each other and exchange data only through `Pipeline`.
- **What it does NOT do:** Does not import `Gate` and `Pipeline`. Services do not depend on each other.

#### Service Composition

The composition of services is **not fixed**. `Core`, `DB`, `Emit` are just typical examples that suit most projects. `Services` can contain anything your application needs:

| Service Example | Purpose |
|---|---|
| **Core** | Pure business logic: rules, calculations, validation. No side effects. |
| **DB** | Database operations: reading and writing. |
| **Emit** | Sending events to clients (WebSocket, SSE, bus). |
| **Cache** | Caching (Redis, in-memory). |
| **Queue** | Enqueuing and processing background tasks. |
| **Mailer** | Sending emails. |
| **Storage** | File operations (S3, local storage). |
| **Search** | Full-text search (Elasticsearch, Meilisearch). |
| **Auth** | Authentication and authorization. |
| **Payment** | Integration with payment systems. |
| **Logger** | Structured logging. |

The rules for services remain unchanged:
1. A service does not depend on other services.
2. A service knows nothing about `Pipeline` and `Gate`.
3. All coordination between services is done through `Pipeline`.

This approach makes `Services` extensible: adding a new service does not require changes in other services and does not break existing code.

---

## US (Client)

The client part is responsible for displaying data and synchronizing state with the server.

### UI

- **Why:** To separate presentation from data and logic.
- **What it does:** Displays data from `Store`, accepts user input, passes commands to `Store`.
- **What it does NOT do:** Does not contain business logic, does not work with the server directly, does not store state.

Composition:

- **Elements** — atomic components without logic (buttons, cards, input fields).
- **View** — read data from `Store`. Display only.
- **Edit** — modify data through `Store` (call `Invoke`).
- **Page** — assemble `View`, `Edit`, and `Elements` into a page.

### Store

- **Why:** To manage state and synchronize it with the server.
- **What it does:** Stores data, processes requests, updates state upon server responses.
- **What it does NOT do:** Does not contain business logic, knows nothing about `UI`.

Composition:

- **State** — data and reducers for modifying it.
- **Entity** — combines `State`, initialization (data loading), and subscriptions to server updates. Manages the data lifecycle (init/clean).
- **Invoke** — sending requests to the server. Receives response, updates `State` via reducers.

---

## Data Movement

### Request

1. **UI → Store** (command).
2. **Store → Gate** (network request via Socket.IO v4).
3. **Gate → Pipeline** (call).
4. **Pipeline → Services** (Core, DB, Emit, or any other service).
5. **Response:** `Pipeline → Gate → Store → UI`.

### Event

1. Server event → **Pipeline**.
2. **Pipeline → Emit** (or another publishing service).
3. **Emit → Entity** (on the client) via Socket.IO v4.
4. **Entity → State** (update).
5. **State → UI** (re-render).

---

## Why GPS Can Be Used Separately

GPS does not depend on US. It is a full-fledged server architecture that solves three tasks:

1. **Receiving external requests** — via `Gate` (HTTP, WebSocket, CLI, cron).
2. **Orchestration of business processes** — via `Pipeline`.
3. **Performing work** — via `Services`.

The client can be anything:

- A web application on React/Vue/Svelte.
- A mobile application.
- A CLI utility.
- Another service (in a microservices architecture).
- A webhook from an external system (e.g., Directus).

GPS does not know who exactly is contacting it through `Gate`. This makes the architecture universal: the same core serves different types of clients, and adding a new client does not require changes in the server part.

---

## Summary

**US-GPS** divides the application into two independent parts:

- **US** — client: `UI → Store`
- **GPS** — server: `Gate → Pipeline → Services`

**Key properties of the architecture:**

- Unidirectional dependency structure.
- Layers are isolated and testable.
- `Services` is an extensible set of independent services. `Core`, `DB`, `Emit` are just examples.
- GPS can be used separately from US for any type of clients and tasks.
- Changes in one layer do not affect others.