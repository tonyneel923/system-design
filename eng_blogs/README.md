# Industry Engineering Blogs & Case Studies

> **High-Yield Architecture Summaries of Big Tech Engineering Blogs**  
> Structured for rapid review during Staff & Principal Frontend System Design prep.

---

## 🏢 Company Folders

### 🔴 [Roblox](./roblox/README.md)
- **[01. Real-Time Chat & Presence System](./roblox/01_realtime_chat_and_presence.md)** (TextChatService, hybrid transport, sequence reconciliation, subway recovery).
- **[02. Real-Time AI Moderation & Chat Translation](./roblox/02_ai_moderation_and_translation.md)** (100ms transformer LLM translation, PII `###` filtering, zero-CLS layout stability).
- **[03. Declarative UI Engines & Virtualized Rendering](./roblox/03_ui_frameworks_and_rendering.md)** (Roact/Rodux $\rightarrow$ Fusion/Vide, fine-grained reactivity, layout virtualization).
- **[04. Client Memory Budgets & Frame Scheduling](./roblox/04_client_performance_and_memory.md)** (Microprofiler, <30MB heap ceilings, GC pause elimination, `task.defer` scheduling).

---

## 📌 Categorized Quick Reference Index

| Core Category | Roblox Case Study | Key System Design Takeaway |
| :--- | :--- | :--- |
| **Real-Time Networking** | [Real-Time Chat & Presence](./roblox/01_realtime_chat_and_presence.md) | Hybrid HTTP/2 REST (outbound) + WSS/SSE (inbound fanout). |
| **Safety & Moderation** | [AI Moderation & Translation](./roblox/02_ai_moderation_and_translation.md) | Sub-100ms edge transformer pipeline; in-place `FAILED` / `FILTERED` UI badges. |
| **UI & Virtualization** | [UI Engines & Rendering](./roblox/03_ui_frameworks_and_rendering.md) | Fine-grained signals over full Virtual DOM diffing; absolute node positioning (`translate3d`). |
| **Performance & Memory** | [Client Memory & Scheduling](./roblox/04_client_performance_and_memory.md) | Dual-layer eviction (DOM + Store); frame-budget scheduled ingestion (`rAF`). |
