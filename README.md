# Engram – Domain State Extension for A2A

> **Engram** is an A2A extension for *durable domain state*.
>
> It gives agents a clean way to read, write, and subscribe to **keyed JSON records** backed by a real store, while still using A2A Messages, Tasks, and Artifacts as the interop surface.

---

## Status

* **Spec:** v0.1 (pre-1.0, evolving; feedback welcome)
* **Extension URI:** `https://github.com/EmberAGI/a2a-engram/tree/v0.1`
* **Repo:** `https://github.com/EmberAGI/a2a-engram`

Engram is designed to evolve independently of the core A2A spec. New breaking versions will use new URIs (`.../tree/v0.2`, `.../tree/v1`, etc.).

---

## Motivation

A2A gives you:

* **Contexts** – long‑lived sessions / conversations
* **Tasks** – stateful units of work inside a context
* **Messages & Artifacts** – immutable snapshots of exchanges and outputs

This is perfect for **task‑ and context‑local history**, but A2A deliberately does **not** define a model for:

* Workflow configs
* Strategy performance metrics
* User / agent / project profiles
* Global or per‑tenant settings

…i.e. long‑lived **domain records** you’d usually keep in a DB.

Engram fills that gap as an **extension**, not a core change:

* It gives agents and services a shared, durable store for domain records.

* It plays nicely with **AG-UI shared state**, giving you a clear split between:

  * AG-UI as the UI- and interaction-facing state model, and
  * Engram as the versioned, queryable backing store that survives across sessions, agents, and contexts.

* A **keyed record store**: `key -> { value, version, timestamps, tags }`

* Backed by a **real database** or memory system

* Exposed via **JSON‑RPC methods**

* Streamed via **A2A Tasks & Artifacts**

Engram lets you keep domain truth *out of* the protocol, while still making it easy for A2A clients, agents, and AG-UI-based UIs to read, write, and subscribe to that truth.

---

## Design Overview

Engram has three main pieces:

1. **Keyed Records**
2. **JSON‑RPC API** (get / list / set / patch / delete / subscribe / resubscribe)
3. **Streaming via A2A Tasks & Artifacts**

### 1. Keyed Records

Engram stores **versioned JSON records** keyed by a minimal, domain‑agnostic identifier.

```ts
// Canonical identifier for a record
interface EngramKey {
  key: string;                        // unique per record within an Engram store
  labels?: Record<string, string>;    // optional metadata; semantics are app-defined
}

// A single Engram record
interface EngramRecord {
  key: EngramKey;
  value: unknown;                     // arbitrary JSON, app-defined
  version: number;                    // monotonically increasing per key
  createdAt: string;                  // ISO-8601
  updatedAt: string;                  // ISO-8601
  tags?: string[];                    // optional labels for querying/search
}
```

Notes:

* The **spec** only cares that `key` is unique and stable. Its structure is up to you.
* You can encode your own conventions into `key` and/or `labels`, e.g.

  * `key: "config/workflow/wf:123/settings"`
  * `key: "metrics/strategy/ETH-USDC/performance"`
  * `labels: { space: "config", ownerType: "workflow", ownerId: "wf:123" }`

### 2. JSON Patch (RFC 6902) for Deltas

Engram uses **JSON Patch** as the standard format for partial updates.

```ts
// RFC 6902
type JsonPatch = Array<{
  op: "add" | "remove" | "replace" | "move" | "copy" | "test";
  path: string;
  from?: string;
  value?: unknown;
}>;
```

Patches are always applied to `record.value` for a given key. Engram automatically bumps `version` and `updatedAt` when a patch or full set succeeds.

---

## Engram JSON‑RPC API

Engram exposes a small set of JSON‑RPC methods as an A2A extension.

> **Note:** Method names here use a simple `engram/*` prefix. In a real AgentCard you may choose a more namespaced prefix (e.g. `extensions/engram/get`).

### `engram/get`

**Purpose:** Fetch one or more full record snapshots.

**Params:**

```ts
interface EngramFilter {
  keyPrefix?: string;
  tagsAny?: string[];
  tagsAll?: string[];
  updatedAfter?: string;              // ISO-8601
  labelEquals?: Record<string, string>;
}

interface EngramGetParams {
  key?: EngramKey;                    // exact key
  keys?: EngramKey[];                 // multiple exact keys
  filter?: EngramFilter;              // broader match
  includeHistory?: boolean;           // optional
}
```

**Result:**

```ts
interface EngramRecordHistoryEntry {
  version: number;
  value: unknown;
  updatedAt: string;
}

interface EngramRecordHistory {
  key: EngramKey;
  entries: EngramRecordHistoryEntry[];
}

interface EngramGetResult {
  records: EngramRecord[];
  history?: EngramRecordHistory[];    // only if includeHistory=true
}
```

This is a **one‑shot RPC**. It does **not** create a Task or subscription by default.

---

### `engram/list`

**Purpose:** List records matching a filter with pagination.

**Params:**

```ts
interface EngramListParams {
  filter?: EngramFilter;
  pageSize?: number;
  pageToken?: string;
}
```

**Result:**

```ts
interface EngramListResult {
  records: EngramRecord[];
  nextPageToken?: string;
}
```

---

### `engram/set`

**Purpose:** Create or replace the entire `value` for a key (upsert).

**Params:**

```ts
interface EngramSetParams {
  key: EngramKey;
  value: unknown;
  expectedVersion?: number;           // optional CAS
  tags?: string[];                    // optional tags
}
```

**Result:**

```ts
interface EngramSetResult {
  record: EngramRecord;               // new snapshot with incremented version
}
```

If `expectedVersion` is provided and does not match the current version, the call MUST fail with a version conflict error.

---

### `engram/patch`

**Purpose:** Apply JSON Patch to an existing record.

**Params:**

```ts
interface EngramPatchParams {
  key: EngramKey;
  patch: JsonPatch;
  expectedVersion?: number;
}
```

**Result:**

```ts
interface EngramPatchResult {
  record: EngramRecord;               // snapshot after patch, new version
}
```

If the record does not exist, or `expectedVersion` mismatches, the call MUST fail.

---

### `engram/delete`

**Purpose:** Delete a record.

**Params:**

```ts
interface EngramDeleteParams {
  key: EngramKey;
  expectedVersion?: number;
}
```

**Result:**

```ts
interface EngramDeleteResult {
  deleted: boolean;
  previousVersion?: number;
}
```

---

## Streaming: `engram/subscribe` & `engram/resubscribe`

Engram uses **A2A Tasks and Artifacts** as the streaming surface.

### `engram/subscribe`

**Purpose:** Open a subscription that streams record changes as Engram events, wrapped in Artifacts of a dedicated Task.

**Params:**

```ts
interface EngramSubscribeParams {
  filter: EngramFilter;               // which records to watch
  includeSnapshot?: boolean;          // send initial snapshots
  contextId?: string;                 // A2A context for the subscription Task
  fromSequence?: string;              // optional resume point
}
```

**Result:**

```ts
interface EngramSubscribeResult {
  subscriptionId: string;
  taskId: string;                     // underlying A2A Task
}
```

**Behavior:**

* The server creates a dedicated subscription Task (or uses an implementation-defined Task type) in the given `contextId`.
* That Task emits `TaskArtifactUpdateEvent`s over the normal A2A streaming channel.
* Each artifact includes one or more **EngramEvent** payloads in its `DataPart`s.

```ts
interface EngramEvent {
  kind: "snapshot" | "delta" | "delete";
  key: EngramKey;
  record?: EngramRecord;              // for snapshot
  patch?: JsonPatch;                  // for delta
  version: number;
  sequence: string;                   // monotonically increasing per subscription
  updatedAt: string;                  // ISO-8601
}
```

On the wire, an event is represented as:

```json
{
  "kind": "data",
  "data": {
    "type": "engram/event",
    "event": { /* EngramEvent */ }
  }
}
```

If `includeSnapshot=true`, the first artifact(s) SHOULD contain `kind: "snapshot"` events for all matching records.

### `engram/resubscribe`

**Purpose:** Resume a subscription after disconnection, mirroring `tasks/resubscribe` semantics.

**Params:**

```ts
interface EngramResubscribeParams {
  subscriptionId: string;
  fromSequence?: string;              // resume from this EngramEvent sequence
}
```

**Result:**

```ts
interface EngramResubscribeResult {
  subscriptionId: string;
  taskId: string;
}
```

**Notes:**

* Implementations MAY map `subscriptionId` → `taskId` internally and call `tasks/resubscribe(taskId, ...)` under the hood.
* Clients that already manage `taskId`s can call `tasks/resubscribe` directly on the underlying Task if they prefer.

---

## Dual Integration Modes

Engram is designed to work in **two modes**:

1. **RPC Mode (canonical)**
2. **Message‑Embedded Mode (ergonomic)**

### 1. RPC Mode

The RPC methods (`engram/get`, `engram/set`, etc.) are the **canonical interface**.

* They are advertised in the AgentCard's `extensions` list using the Engram URI.
* They are ideal for:

  * other agents and tools,
  * backends and orchestrators,
  * tests and scripts.

### 2. Message‑Embedded Mode

Some A2A clients only speak in terms of `message/send` / `message/stream`. For those, Engram defines a simple **operation envelope** that can be embedded in a `data` part:

```ts
type EngramOpKind = "get" | "set" | "patch" | "delete";

interface EngramOperation {
  type: "engram/op";
  op: EngramOpKind;
  key?: EngramKey;                    // required for set/patch/delete
  value?: unknown;                    // for set
  patch?: JsonPatch;                  // for patch
  filter?: EngramFilter;              // for get/list-style ops
  requestId?: string;                 // correlates responses
}
```

Example message:

```json
{
  "role": "user",
  "parts": [
    {
      "kind": "data",
      "data": {
        "type": "engram/op",
        "op": "set",
        "key": { "key": "config/workflow/wf:123/settings" },
        "value": { "maxRisk": 0.01, "rebalanceInterval": "1h" },
        "requestId": "req-123"
      }
    }
  ]
}
```

Implementation guidance:

* Message‑embedded ops SHOULD be treated as a **thin wrapper** that internally calls the corresponding RPC methods.
* Responses back to the client MAY:

  * include EngramEvents via the subscription Task,
  * or attach snapshots as Artifacts/messages as appropriate.

---

## Relationship to Tasks, Artifacts, and State

Engram is intentionally **layered on top of A2A**:

* **Tasks, messages, artifacts, metadata** remain the canonical way to:

  * represent workflows,
  * track interactions,
  * share outputs between agents.

* **Engram** provides a separate, durable **domain store**:

  * Long‑lived records (configs, performance metrics, profiles, etc.) live in the Engram store.
  * Tasks read from and write to that store as needed.
  * Artifacts serve as snapshots and projections for interop.

### Recommended mental model

* Use **Tasks + Artifacts** as the **event log / workflow history** layer.
* Use **Engram** as the **materialized domain model** layer.
* Let UIs and other agents:

  * call `engram/get` / `engram/set` for one‑shot operations,
  * use `engram/subscribe` to keep dashboards and shared state in sync.

---

## Example: Workflow Settings & Performance

Imagine a trading workflow with:

* Workflow settings: `config/workflow/wf:123/settings`
* Workflow performance metrics: `metrics/workflow/wf:123/performance`

**Change settings via RPC:**

```json
{
  "jsonrpc": "2.0",
  "id": "1",
  "method": "engram/set",
  "params": {
    "key": { "key": "config/workflow/wf:123/settings" },
    "value": {
      "maxRisk": 0.01,
      "rebalanceInterval": "1h"
    },
    "expectedVersion": 3
  }
}
```

**Subscribe to performance updates:**

```json
{
  "jsonrpc": "2.0",
  "id": "2",
  "method": "engram/subscribe",
  "params": {
    "filter": {
      "keyPrefix": "metrics/workflow/wf:123/"
    },
    "includeSnapshot": true,
    "contextId": "ctx-trading-dashboard"
  }
}
```

The client then listens for artifact updates on the returned `taskId` and applies `EngramEvent`s to its local state store.

---

## AG-UI Shared State Compatibility

Engram is designed to complement, not replace, **AG-UI shared state**. If you are already using AG-UI, Engram can act as the **backing store and subscription layer** for that state.

### Engram as a backing store for shared state

A common pattern is:

* Treat one or more Engram records as the **source of truth** for AG-UI shared state.

  * e.g. `key: "ui/agent:trader/state"` or more granular keys like `"ui/agent:trader/layout"`, `"ui/agent:trader/filters"`.
* When your backend agent updates AG-UI state, it also:

  * writes the new state (or slice) to Engram via `engram/set` or `engram/patch`, and
  * optionally emits EngramEvents via `engram/subscribe` for any interested dashboards.
* On startup or reconnect, AG-UI can be hydrated by:

  * calling `engram/get` to fetch current records, and
  * mapping them into the initial AG-UI shared state object.

This keeps AG-UI state **durable and cross-session** while preserving the AG-UI mental model for frontend code.

### Mapping Engram events to AG-UI `STATE_SNAPSHOT` / `STATE_DELTA`

If you already use AG-UI's `STATE_SNAPSHOT` and `STATE_DELTA` events, Engram can be treated as the **underlying store** that those events mirror:

* **Snapshots:**

  * An Engram `EngramRecord` (or a set of them) can be serialized into a single AG-UI `STATE_SNAPSHOT` payload.
  * This is ideal for initial hydration or full resync when patch application fails.

* **Deltas:**

  * Engram `EngramEvent` with `kind: "delta"` already carries a JSON Patch.
  * That patch can be applied directly to:

    * the Engram store (backing DB), and
    * the AG-UI shared state object as a `STATE_DELTA` event.

In other words, you can treat **Engram JSON Patch** as the same diff format that powers AG-UI state deltas, letting one stream drive both storage and UI.

### AG-UI as a client of Engram

From a frontend perspective:

* AG-UI remains the **primary interface** for rendering, user interaction, and stateful components.
* The AG-UI bridge / backend is responsible for:

  * calling `engram/get` / `engram/set` / `engram/patch` as needed, and
  * wiring `engram/subscribe` events into AG-UI `STATE_DELTA` events.

This keeps your React / UI code talking in AG-UI terms, while Engram provides a **consistent, versioned domain store** that multiple agents and tools can share.

---

## Roadmap & Non‑Goals

### Roadmap

* Reference TypeScript and/or Python Engram store implementation
* Helpers for mapping Engram records into:

  * LangGraph state / long‑term stores
  * AG‑UI shared state / `STATE_SNAPSHOT` / `STATE_DELTA` events
* Example A2A agents using Engram for:

  * workflow configs
  * strategy performance dashboards
  * user preference storage

### Non‑Goals

* Replacing A2A Task state or history
* Defining global domain concepts like "user", "strategy", or "workflow" in the core Engram spec
* Mandating any particular backing database implementation

---

## Contributing

Engram is designed to be:

* **Spec‑first**: the extension URI and JSON types are the source of truth.
* **Implementation‑agnostic**: you can implement the Engram store with any DB / runtime.

If you'd like to:

* Propose new methods or filters
* Add reference implementations (TS, Python, etc.)
* Add example agents/UIs that use Engram

…please open an issue or PR in this repo.

---

## License

Licensed under the [Apache License, Version 2.0](./LICENSE).

