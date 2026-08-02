# Deep Dive Study Guide: Offline Storage, Service Workers & Outbox Sync Architecture

> **Target Audience**: Staff & Principal Frontend System Design Candidates (Roblox, Meta, Google).  
> **Focus**: Master-class architectural guide on multi-tiered browser storage, Service Worker cache strategies, persistent outbox mutation queues, idempotency, and reconnection reconciliation.

---

## Part 1: Browser Storage Tiering Matrix

| Storage Tier | API Target | Capacity Ceiling | Synchronous / Async | Persistence Lifecycle | Ideal Use Cases |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Tier 1: JS Memory Heap** | JS Objects / Redux / Zustand | Strict 25–30MB budget | Synchronous | Volatile (Wiped on page reload / crash) | Active visible viewport state, normalized `byIds` map. |
| **Tier 2: IndexedDB** | `idb` / Custom `StorageController` | 50% of available disk space (> 1GB) | Async (Non-blocking) | Persistent (Survives browser restarts) | Feed page chunks, normalized entities, offline Outbox task queue. |
| **Tier 3: Cache API** | `window.caches` (Service Worker) | Hundreds of MBs (Disk quota bounded) | Async (Non-blocking) | Persistent (Survives browser restarts) | Static media assets (WebP/AVIF image blobs, JS/CSS bundles). |
| **Tier 4: Web Storage** | `localStorage` / `sessionStorage` | ~5MB per origin | **Synchronous (Main-thread blocking!)** | Persistent (`local`) or Session (`session`) | **Never use for heavy feeds!** Lightweight auth tokens, UI theme preference only. |

---

## Part 2: Three-Tier Offline Architecture Blueprint

```
+---------------------------------------------------------------------------------------------------+
|                                  CLIENT SYSTEM BOUNDARY                                           |
|                                                                                                   |
|   +-------------------+          +-------------------+          +------------------------------+  |
|   |  JS Heap Memory   | <------> | IndexedDB Engine  | <------> |  Service Worker Cache API    |  |
|   |  (Active Viewport |          | (Page Chunks &    |          |  (Image Asset Blobs          |  |
|   |   ~30MB Heap)     |          |  Outbox Queue)    |          |   100MB LRU Bound)           |  |
|   +-------------------+          +-------------------+          +------------------------------+  |
|             ^                              ^                                     ^                |
|             |                              |                                     |                |
|             +------------------------------+-------------------------------------+                |
|                                            |                                                      |
|                                   [ StorageController ]                                           |
+---------------------------------------------------------------------------------------------------+
```

---

## Part 3: Service Worker Caching Strategies

```
1. Cache-First (Media Images / Assets)
   Request ---> Match in Cache API? ---> YES ---> Return Blob
                     | NO
                     v
             Fetch from CDN ---> Save to Cache API ---> Return Blob

2. Stale-While-Revalidate (User Profiles / Badges)
   Request ---> Return Cached Data Immediately AND Fetch Network in Background ---> Update Cache API

3. Network-First with Cache Fallback (Feed Pages / Metadata)
   Request ---> Fetch REST API ---> SUCCESS ---> Return Data & Save to IndexedDB
                     | FAILURE (Offline)
                     v
             Query IndexedDB Page Chunk ---> Return Cached Data
```

---

## Part 4: Persistent Outbox Queue & Reconnection Sync Engine

When a user likes a post, writes a comment, or creates a post while offline, the system must **never reject the action**.

```mermaid
graph TD
    UserAction[User Click: Create Post / Like] --> OptState[1. Immediate Optimistic UI Dispatch]
    OptState --> OutboxTask[2. Create OutboxMutationTask + idempotencyKey + temp_id]
    OutboxTask --> IDB[3. Write Task to IndexedDB OutboxStore]

    IDB --> Listener{4. Network Connection Available?}
    Listener -->|No / Subway Tunnel| Wait[Keep Task in OutboxStore]
    Listener -->|Yes / Reconnected| SyncEngine[5. Trigger SyncEngine.flush()]

    SyncEngine --> POST[6. HTTP POST with X-Idempotency-Key Header]
    POST <--> Backend[Backend Service]

    Backend -->|201 Created ACK| Success[7. Swap temp_id -> real_id & Delete Task from OutboxStore]
    Backend -->|4xx / Moderation Failure| Rollback[8. Rollback Optimistic UI State & Render Error Badge]
```

### **1. Outbox Task TypeScript Schema**

```typescript
export interface OutboxMutationTask {
  mutationId: string;       // Unique UUID for the outbox task
  idempotencyKey: string;   // Client nonce to prevent duplicate server writes
  type: 'CREATE_POST' | 'LIKE_POST' | 'ADD_COMMENT';
  tempEntityId: string;     // Client-generated temporary ID (e.g. temp_uuid_123)
  payload: Record<string, any>;
  timestamp: number;        // Epoch timestamp
  retryCount: number;
  status: 'PENDING_OFFLINE' | 'SYNCING' | 'FAILED';
}
```

---

### **2. Production SyncEngine Reconnection Implementation**

```typescript
import { openDB, IDBPDatabase } from 'idb';

export class OutboxSyncEngine {
  private dbPromise: Promise<IDBPDatabase>;

  constructor() {
    this.dbPromise = openDB('app_offline_db', 1, {
      upgrade(db) {
        db.createObjectStore('outbox', { keyPath: 'mutationId' });
      },
    });

    // Listen for browser reconnection
    window.addEventListener('online', () => this.flushOutbox());
  }

  public async enqueueTask(task: OutboxMutationTask): Promise<void> {
    const db = await this.dbPromise;
    await db.put('outbox', task);

    if (navigator.onLine) {
      this.flushOutbox();
    }
  }

  public async flushOutbox(): Promise<void> {
    if (!navigator.onLine) return;

    const db = await this.dbPromise;
    const tasks: OutboxMutationTask[] = await db.getAll('outbox');

    // Replay tasks in chronological order
    tasks.sort((a, b) => a.timestamp - b.timestamp);

    for (const task of tasks) {
      try {
        await db.put('outbox', { ...task, status: 'SYNCING' });

        const response = await fetch('/api/v1/mutations', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Idempotency-Key': task.idempotencyKey,
          },
          body: JSON.stringify(task),
        });

        if (response.ok) {
          const result = await response.json();
          // Server returns real_id for client ID reconciliation
          this.reconcileIdSwap(task.tempEntityId, result.realEntityId);
          await db.delete('outbox', task.mutationId);
        } else if (response.status >= 400 && response.status < 500) {
          // Client/Moderation error: Rollback optimistic UI and drop task
          this.rollbackOptimisticState(task);
          await db.delete('outbox', task.mutationId);
        }
      } catch (err) {
        console.warn(`Sync failed for task ${task.mutationId}. Will retry on next reconnect.`, err);
        await db.put('outbox', { ...task, status: 'PENDING_OFFLINE', retryCount: task.retryCount + 1 });
        break; // Stop loop on network error to preserve temporal order
      }
    }
  }

  private reconcileIdSwap(tempId: string, realId: string): void {
    // Dispatch ID swap action to central Redux/Zustand store
    console.log(`Reconciled ID: ${tempId} -> ${realId}`);
  }

  private rollbackOptimisticState(task: OutboxMutationTask): void {
    // Rollback state in central store
    console.warn(`Rolled back optimistic task: ${task.mutationId}`);
  }
}
```
