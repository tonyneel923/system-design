# Master Study Guide: Common Frontend System Design Patterns & Anti-Patterns

> **High-Yield Cross-System Design Playbook for Roblox & Big Tech**
> 
> This document synthesizes recurring architectural patterns, state reconciliation strategies, network transport paradigms, memory management protocols, and interviewer counter-strategies derived across all system design modules in this repository.

---

## I. Universal Architectural Patterns

### 1. Network Layer Decoupling (The Singleton Pattern)
- **The Core Rule**: Never instantiate WebSocket listeners, socket heartbeats, or transport managers inside UI component lifecycles (e.g. React `useEffect` or Vue `onMounted`).
- **Why It Matters**: UI component unmounts during navigation (e.g. switching tabs or entering a 3D game canvas) tear down sockets, trigger duplicate handshake storms, and lose in-flight message queues.
- **The Pattern**:
  ```
  +-------------------------------------------------------------+
  |              Singleton Connection Manager                   |
  |  (Pure TypeScript / Lua Module outside UI Component Tree)   |
  |                                                             |
  |  +-------------------+  +----------------+  +------------+  |
  |  |  WSS Controller   |  | Event Pub/Sub  |  | Action TTL |  |
  |  | (Heartbeat/Retry) |  |   Dispatcher   |  |   Queue    |  |
  |  +-------------------+  +----------------+  +------------+  |
  +-------------------------------------------------------------+
                                 | (Pub/Sub Events)
                                 v
  +-------------------------------------------------------------+
  |                Decoupled Central State Store                |
  +-------------------------------------------------------------+
                                 | (Subscribed Selectors)
                                 v
  +-------------------------------------------------------------+
  |            UI Components (React / Native / Engine)          |
  +-------------------------------------------------------------+
  ```

---

### 2. Hybrid Transport Paradigm (Outbound REST + Inbound WSS)
- **The Core Rule**: Use **HTTP/2 REST** for outbound user actions (sending messages, joining games, updating profiles) and **WebSockets (WSS)** for inbound event broadcasting (chat streams, presence updates, notifications).
- **Why It Matters**: Full-duplex WebSockets suffer from write queue head-of-line blocking during heavy traffic and prevent edge proxies (CDNs, Cloudflare, API Gateways) from running low-latency validation and AI moderation workers on POST payloads.

---

### 3. Dual-Key Indexing & Optimistic UI State Transitions
- **The Core Rule**: Always pair a frontend-generated `client_nonce` (UUIDv4) with a backend-assigned monotonic `sequenceId` ($S_{id}$).
- **The State Machine**:
  ```
  [User Action] --> Insert Local Item (clientId, status = OPTIMISTIC)
                            |
           +----------------+----------------+
           | (Server ACK)                    | (Network / Policy Failure)
           v                                 v
  Update to PENDING_MOD               Update to FAILED_SEND
  Map clientId -> messageId           Retain Node Slot (In-Place Error Badge)
           |                                 |
     +-----+-----+                     [User Clicks Retry]
     |           |                           |
     v           v                           v
  APPROVED   FILTERED (###)          Re-enter OPTIMISTIC Pipeline
  ```
- **UX Guardrail (Zero Cumulative Layout Shift)**: Never delete a failed optimistic item from the array when newer items exist below it. Retain the node in the layout with a `FAILED_SEND` badge to prevent layout jumps.

---

### 4. Dual-Layer Memory & GC Protection
- **The Core Rule**: High-scale frontend streams (1,000+ friend updates, 10,000+ chat messages) require **both** DOM Virtualization **and** Central Store Historical Truncation.
- **The Blueprint**:
  1. **DOM Virtualization**: Mount only visible rows (~15–20 items) inside the viewport using fixed container heights and dynamic row height caching.
  2. **Store Window Truncation**: Enforce a strict array ceiling in memory (e.g. max 1,000 items in store). When new items push the store over capacity, purge oldest items from memory. If the user scrolls back up, lazily re-fetch historical deltas via paginated REST endpoints (`GET /history?before_seq=N`).
  3. **Heap Goal**: Keep client JS/Lua heap allocations < 25–30MB to eliminate Garbage Collection (GC) frame stutters on low-end mobile devices.

---

### 5. Ingestion Micro-Batching (`requestAnimationFrame`)
- **The Core Rule**: Never dispatch a store update or trigger a UI re-render on every individual WebSocket frame during high-frequency bursts (e.g., 50+ events/sec).
- **The Pattern**:
  ```typescript
  class MicroBatchScheduler<T> {
    private buffer: T[] = [];
    private frameScheduled = false;

    public enqueue(event: T, onFlush: (items: T[]) => void): void {
      this.buffer.push(event);
      if (!this.frameScheduled) {
        this.frameScheduled = true;
        requestAnimationFrame(() => {
          const batch = this.buffer;
          this.buffer = [];
          this.frameScheduled = false;
          onFlush(batch);
        });
      }
    }

    public flushImmediately(onFlush: (items: T[]) => void): void {
      if (this.buffer.length > 0) {
        const batch = this.buffer;
        this.buffer = [];
        this.frameScheduled = false;
        onFlush(batch);
      }
    }
  }
  ```
- **Local Send Guarantee**: Whenever the local user triggers an action (e.g., sends a message), immediately invoke `flushImmediately()` to drain pending incoming frames before appending the user's optimistic action. This guarantees strict local temporal ordering.

---

## II. Reconnection & Resilience Patterns (The Subway Tunnel Scenario)

```
[Mobile Device Enters Subway Tunnel]
                 |
        (Connection Drops)
                 |
   1. Local actions appended to Offline Replay Queue / IndexedDB OutboxStore.
   2. WS Connection Manager / NetworkObserver enters Exponential Backoff Retry loop.
                 |
      (Reconnection Established)
                 |
   3. Purge expired actions from Replay Queue (TTL < Date.now()).
   4. Issue REST Delta Sync / Flush Outbox Queue with X-Idempotency-Key.
   5. Reconcile Delta Payload & Temp IDs (temp_id -> server_id) into Store.
   6. Resume real-time WSS / SSE stream listening.
```

### **IndexedDB Outbox Queue + Service Worker Sync Architecture**
- **The Core Rule**: Never lose user writes (likes, comments, post creations) during network drops. All user mutations pass through a persistent **Outbox Store in IndexedDB** before hitting the network.
- **The Pipeline**:
  1. **Optimistic Dispatch**: UI immediately updates local state (`status: 'PENDING_OFFLINE'`).
  2. **Disk Persistence**: Mutation payload + `idempotencyKey` + `client_nonce` written to IndexedDB.
  3. **Sync Engine Replay**: When `navigator.onLine` fires (or Service Worker `sync` event triggers), `SyncEngine` reads Outbox tasks in FIFO order and POSTs to server with `X-Idempotency-Key`.
  4. **ID Swap & Eviction**: Server responds with `201 Created` + `server_id`. Store swaps `temp_id -> server_id` and deletes task from Outbox Store.

---


## III. Common Candidate Anti-Patterns & Interviewer Traps

| Anti-Pattern / Trap | Root Cause / Vulnerability | Principal Counter-Strategy |
| :--- | :--- | :--- |
| **`useWebSocket` Hook Anti-Pattern** | Placing socket instantiation inside React hooks. Causes connection teardowns on component unmounts and duplicate socket handshakes. | Decouple transport layer into a pure TypeScript/Lua Singleton outside React. UI components subscribe via Pub/Sub or Store selectors. |
| **Timestamp-Based Reconciliation** | Relying on client or server timestamps for message sorting and deduplication. Suffer from global client clock skew and microsecond timestamp collisions. | Enforce monotonic integer sequence numbers ($S_{id}$) generated by backend storage. Treat timestamps purely as UI display strings. |
| **Deleting Failed Optimistic Items** | Removing a failed optimistic message node from the stream array. Triggers jarring Cumulative Layout Shift (CLS) when newer items have rendered below it. | Retain node slot in array and render an in-place `FAILED_SEND` badge with inline retry trigger. |
| **Unbounded Store Accumulation** | Virtualizing DOM nodes while allowing the central JS state store to store 10,000+ items indefinitely. | Implement Dual-Layer Memory Truncation: trim store array when exceeding 1,000 items and re-fetch on upward scroll. |
| **Full WebSocket Outbound POSTs** | Sending all outbound API calls through WebSockets. Prevents CDN edge verification and leads to socket head-of-line blocking. | Implement Hybrid Transport: HTTP/2 REST for outbound actions, WSS for inbound fanout streaming. |
| **Main-Thread Image Decoding** | Synchronous image rasterization during scrolling causes dropped frames (jank). | Set `img.decoding = "async"` or use `Image.prototype.decode()` to move decompression to background threads. |
| **Blind Fling Image Requests** | Rapid scrolling triggers 30+ image fetches that saturate HTTP/2 sockets and block API calls. | Attach `AbortController` to `IntersectionObserver` exit callbacks and pause prefetching when scroll velocity $v > \text{threshold}$. |
| **Layout Shift on Upstream Resize** | Items above the visible viewport resizing cause jarring visual scroll jumps for the user. | Synchronously adjust `container.scrollTop += ΔH` inside `useLayoutEffect` before browser paint. |
| **Random Item Store Eviction** | Evicting single items from memory breaks cursor pagination continuity. | Evict in discrete **page chunks** to IndexedDB via a `StorageController` abstraction layer. |
| **Un-polyfilled `requestIdleCallback`** | Calling bare `requestIdleCallback` crashes or fails silently in Safari/WebKit. | Use `CrossBrowserIdleScheduler` with `MessageChannel` macro-task fallback and 16.6ms frame budget tracking. |
| **Worker Structured Clone Overhead** | `worker.postMessage(data)` deep-copies objects, doubling memory usage and causing CPU copy lag. | Use Transferable Objects (`ArrayBuffer`, `ImageBitmap`) for $O(1)$ zero-copy memory ownership transfers. |
| **Un-revoked Blob URLs** | `URL.createObjectURL(blob)` holds strong references in memory indefinitely. | Invoke `URL.revokeObjectURL(blobUrl)` immediately upon image load or component unmount. |

---

## IV. Quick Reference Architecture Matrix across Topics

| System Design Module | Core Network Strategy | Key State Mechanism | Primary Memory Safeguard |
| :--- | :--- | :--- | :--- |
| **Roblox Persistent Presence** | WSS Singleton + REST Delta Catch-up | Reference-Preserved Array Copies (`newArr[i] = oldArr[i]`) | `React.memo` + Shallow Prop Equality ($O(1)$ re-renders) |
| **Roblox Chat & Messaging Panel** | Hybrid HTTP/2 REST + WSS Fanout | Dual-Key (`clientId` + `sequenceId`) State Machine | DOM Virtualization + Central Store Window Eviction (<25MB Heap) |
| **Roblox High-Scale Image Feed** | REST Cursor + SSE Unread Notifications | Normalized Store (`byIds`/`feedOrder`) + Tiered `StorageController` (IndexedDB) | DOM Virtualization + `AbortController` Cancellation + `scrollTop` Offset Sync (<30MB Heap) |

