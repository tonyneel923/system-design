# Staff & Principal Frontend System Design Repository

> **High-Yield Interview Preparation & Technical Pattern Playbook**  
> Tailored for Staff / Principal Frontend Engineering System Design interviews (Roblox, Meta, Google).

---

## 🎯 System Design Mock Practice Guides (`practice/`)

Real 45-minute mock interview runs complete with candidate whiteboard sketch evaluations, Mermaid translations, comparative evaluation matrices, and production reference architectures:

1. **[01: Roblox Persistent Social Presence & Party System](./practice/01_roblox_persistent_presence.md)**
   - *Focus*: WebSockets singleton, reference-preserved array copies, `React.memo` bailout, REST delta reconnection.
2. **[02: Roblox Universal Chat & Messaging Panel](./practice/02_roblox_chat_messaging_panel.md)**
   - *Focus*: Hybrid transport (REST + WSS), AI moderation text filtering (`###`), dual-key indexing, zero-CLS in-place error badges, `requestAnimationFrame` micro-batching.
3. **[03: Roblox High-Scale Image Feed](./practice/03_roblox_image_feed.md)**
   - *Focus*: Virtualized dynamic height rendering, aspect-ratio skeletons, `useLayoutEffect` scroll position sync (`scrollTop += ΔH`), off-thread image decoding (`img.decoding = "async"`), `AbortController` cancellation on viewport exit, and tiered IndexedDB page-chunk caching.

![Roblox High-Scale Image Feed Whiteboard Sketch](./practice/assets/03_roblox_image_feed_sketch_v2.png)

---

## 📚 Technical Topic Deep-Dives & Patterns (`lessons/`)

1. **[00: Master Frontend System Design Patterns & Anti-Patterns](./lessons/00_frontend_system_design_patterns.md)**
   - Master rules, network layer singletons, dual-key state machines, ingestion micro-batching, anti-patterns & interviewer traps.
2. **[01: Real-Time Transports (WebSockets vs. SSE vs. Long-Polling)](./lessons/01_realtime_transports_websockets_sse.md)**
   - Protocol decision matrix, decision trees, exponential backoff with full jitter algorithm, and `requestAnimationFrame` micro-batch engine code.
3. **[02: Offline Storage, Service Workers & Outbox Sync Architecture](./lessons/02_offline_storage_outbox_sync.md)**
   - 4-tier browser storage matrix (JS Heap vs IndexedDB vs Cache API vs Web Storage), Service Worker caching strategies, and complete TypeScript `OutboxSyncEngine` implementation.
4. **[03: Infinite List Virtualization & Image Lifecycle Engineering](./lessons/03_infinite_list_virtualization_and_image_lifecycle.md)**
   - Virtualization layout comparison (padding vs `translate3d` vs recycled DOM pools), dynamic height prefix-sum offset math, zero-flicker scroll sync, and React code examples.
5. **[04: Memory-Bound Architecture, Idle Scheduling & Zero-Copy Web Workers](./lessons/04_client_memory_management_and_idle_scheduling.md)**
   - Strict <30MB heap ceilings, Safari `requestIdleCallback` polyfill (`MessageChannel` + 16.6ms budget check), $O(1)$ zero-copy Web Worker transfers, object pooling, and Blob URL leak prevention.

---


## 🏢 Industry Tech Blog Case Studies (`eng_blogs/`)

Categorized technical breakdowns of production engineering blogs:

- **[Roblox Tech Blog Breakdown Index](./eng_blogs/roblox/README.md)**
  1. **[Real-Time Chat & Presence System](./eng_blogs/roblox/01_realtime_chat_and_presence.md)** (TextChatService, hybrid transport, sequence reconciliation, subway recovery).
  2. **[Real-Time AI Moderation & Multilingual Translation](./eng_blogs/roblox/02_ai_moderation_and_translation.md)** (Sub-100ms transformer LLM, PII `###` filtering, zero-CLS layout stability).
  3. **[Declarative UI Frameworks & Virtualized Rendering](./eng_blogs/roblox/03_ui_frameworks_and_rendering.md)** (Roact/Rodux $\rightarrow$ Fusion/Vide/Reflex, fine-grained reactivity, layout virtualization).
  4. **[Client Memory Budgets & Frame Scheduling](./eng_blogs/roblox/04_client_performance_and_memory.md)** (Microprofiler, <30MB heap ceilings, GC pause elimination, frame scheduling).

---

## 📋 Interviewer Prompt Templates (`templates/`)

1. **[Master Principal Mock Interviewer Prompt](./templates/principal_mock_interviewer.md)**
2. **[02: Roblox Chat & Messaging Prompt](./templates/02_roblox_chat_messaging_prompt.md)**
3. **[03: Roblox Image Feed Prompt](./templates/03_roblox_image_feed_prompt.md)**

