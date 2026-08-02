# Deep Dive Study Guide: Real-Time Network Transports (WebSockets, SSE & Long-Polling)

> **Target Audience**: Staff & Principal Frontend System Design Candidates (Roblox, Meta, Google).  
> **Focus**: Master-class architectural guide on selecting, scaling, and engineering real-time communication protocols, connection lifecycles, reconnection resilience, and frame-rate ingestion batching.

---

## Part 1: Protocol Comparison & Decision Matrix

| Dimension | WebSockets (WSS) | Server-Sent Events (SSE) | HTTP/2 Long-Polling | WebRTC (DataChannels) |
| :--- | :--- | :--- | :--- | :--- |
| **Directionality** | Full-Duplex (Client <-> Server) | Unidirectional (Server -> Client) | Half-Duplex (Request/Response) | Full-Duplex Peer-to-Peer / Client-Server |
| **Transport Protocol** | Custom TCP Frame Protocol (`ws://` / `wss://`) | Standard HTTP/1.1 or HTTP/2 (`text/event-stream`) | Standard HTTP | UDP / SCTP |
| **Multiplexing** | Requires single TCP connection per socket (no HTTP/2 multiplexing) | Multiplexed natively over HTTP/2 | Multiplexed natively over HTTP/2 | Peer Connection Channels |
| **Native Auto-Reconnect** | Manual (requires custom client connection manager) | Built-in browser native auto-reconnect (`EventSource`) | Manual loop | Manual / ICE Restarts |
| **Firewall & Proxy Behavior** | Can be blocked by strict corporate proxies/firewalls | Traverses all proxies/firewalls cleanly (standard HTTP) | Standard HTTP | Requires STUN/TURN traversal servers |
| **Primary Use Cases** | Live Chat, Multiplayer Games, Collaborative Canvases, 3D Presence | Unread Feed Badges, Live Feeds, Stock Tickers, AI Stream Outputs | Legacy fallback environments | Peer-to-Peer Audio/Video, Ultra low-latency gaming |

---

## Part 2: Architectural Decision Tree

```
                          What is the direction of data flow?
                                         |
               +-------------------------+-------------------------+
               |                                                   |
      Unidirectional (Server -> Client)                   Bi-directional (Client <-> Server)
               |                                                   |
    Is HTTP/2 Available?                                Is latency < 50ms required for P2P/3D?
    +----------+----------+                                +----------+----------+
    |                     |                                |                     |
   YES                   NO                               YES                   NO
    |                     |                                |                     |
[Server-Sent Events]  [HTTP Long-Polling]              [WebRTC]       [Hybrid Model: HTTP REST Outbound
 (SSE text/event-stream)                                               + WSS Fanout Inbound]
```

> **The Hybrid Transport Paradigm**:  
> In production at scale (Roblox, Meta), **do not use WebSockets for outbound user writes**. Outbound POST requests sent over WebSockets cause write head-of-line blocking and bypass CDN edge API gateways (Cloudflare, AWS CloudFront) that perform low-latency auth, rate limiting, and AI moderation.  
> **Use HTTP/2 REST for Outbound Writes + WSS/SSE for Inbound Streaming**.

---

## Part 3: Connection Lifecycle, Heartbeats & Exponential Backoff

### **1. Connection Singleton Pattern**
Never instantiate WebSockets or SSE listeners inside React components (`useEffect`) or UI views. UI unmounts during navigation tear down connections, trigger socket handshake storms, and leak event listeners.

```
+-------------------------------------------------------------------------------+
|                    ConnectionManager (Pure TS Singleton)                       |
|                                                                               |
|   - Holds WSS / SSE Instance                                                  |
|   - Reconnection Backoff & Heartbeat Ping/Pong Loop                           |
|   - Offline Queue Buffer                                                      |
|   - Pub/Sub Event Emitter                                                     |
+-------------------------------------------------------------------------------+
                                     |
                         (Pub/Sub Event Dispatch)
                                     v
+-------------------------------------------------------------------------------+
|                   Centralized State Store (Redux / Zustand)                   |
+-------------------------------------------------------------------------------+
                                     |
                           (Subscribed Selectors)
                                     v
+-------------------------------------------------------------------------------+
|                   UI Layer (<ChatPanel />, <PresenceBar />)                   |
+-------------------------------------------------------------------------------+
```

---

### **2. Exponential Backoff with Jitter Algorithm**
When connectivity drops (e.g. entering a tunnel), aggressive instant retries will crash servers during thundering herd reconnects. Reconnection must use **Exponential Backoff with Full Jitter**:

$$T_{\text{wait}} = \text{random}(0, \min(T_{\text{max}}, T_{\text{base}} \cdot 2^k))$$

```typescript
class ReconnectionManager {
  private baseDelayMs = 1000;
  private maxDelayMs = 30000;
  private attempt = 0;

  public getNextBackoffDelay(): number {
    // Exponential calculation: 1s, 2s, 4s, 8s, 16s, 30s...
    const temp = Math.min(this.maxDelayMs, this.baseDelayMs * Math.pow(2, this.attempt));
    // Apply full jitter (random value between 0 and temp) to prevent thundering herd
    const sleep = Math.floor(Math.random() * temp);
    this.attempt++;
    return sleep;
  }

  public reset(): void {
    this.attempt = 0;
  }
}
```

---

## Part 4: Frame-Rate Ingestion Engine (`requestAnimationFrame`)

When receiving high-burst streams (e.g. 100 events/sec during live game events or chat bursts), dispatching store updates on every frame causes severe main-thread UI jank and React render cascades.

```typescript
export class MicroBatchIngestionEngine<T> {
  private queue: T[] = [];
  private isScheduled = false;

  constructor(private onFlush: (batch: T[]) => void) {}

  public push(item: T): void {
    this.queue.push(item);
    if (!this.isScheduled) {
      this.isScheduled = true;
      // Schedule batch processing right before the next browser repaint (16.6ms)
      requestAnimationFrame(() => this.flush());
    }
  }

  // Local send guarantee: Flush queue immediately on user action to preserve local temporal order
  public flushImmediately(): void {
    if (this.queue.length > 0) {
      this.flush();
    }
  }

  private flush(): void {
    const batch = this.queue;
    this.queue = [];
    this.isScheduled = false;
    this.onFlush(batch);
  }
}
```

---

## Part 5: Code Example: Production SSE vs WebSocket Client

### **Server-Sent Events (SSE) Client Singleton**

```typescript
export class SSEFeedStreamManager {
  private static instance: SSEFeedStreamManager;
  private eventSource: EventSource | null = null;

  private constructor() {}

  public static getInstance(): SSEFeedStreamManager {
    if (!SSEFeedStreamManager.instance) {
      SSEFeedStreamManager.instance = new SSEFeedStreamManager();
    }
    return SSEFeedStreamManager.instance;
  }

  public connect(url: string, onNewPostsCount: (count: number) => void): void {
    if (this.eventSource) return;

    this.eventSource = new EventSource(url, { withCredentials: true });

    this.eventSource.addEventListener('unread_update', (event: MessageEvent) => {
      const data = JSON.parse(event.data);
      onNewPostsCount(data.count);
    });

    this.eventSource.onerror = (err) => {
      console.warn('SSE connection lost. Browser native auto-reconnecting...', err);
    };
  }

  public disconnect(): void {
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = null;
    }
  }
}
```
