# Roblox Engineering Blog Breakdown & Frontend System Design Case Studies

> **High-Yield Summaries of Roblox Tech Blog & SIGGRAPH Engineering Publications**

---

## 📚 Articles Breakdown

### 1. [Real-Time Chat & Presence System](./01_realtime_chat_and_presence.md)
- **Topics**: `TextChatService`, Hybrid Transport Model, Sequence Reconciliation, Subway Tunnel Recovery.
- **Key Takeaway**: Outbound messages send over REST POST for CDN edge micro-batching & moderation routing; inbound stream received over WebSockets.

### 2. [Real-Time AI Moderation & Chat Translation](./02_ai_moderation_and_translation.md)
- **Topics**: 100ms Multilingual Transformer LLM, PII Filter Redaction (`###`), Zero-CLS In-Place Error Badges.
- **Key Takeaway**: Sub-100ms translation at edge; server moderation updates item status asynchronously without triggering layout shifts.

### 3. [Declarative UI Engines & Virtualized Rendering](./03_ui_frameworks_and_rendering.md)
- **Topics**: Roact / Rodux $\rightarrow$ Fusion / Vide / Reflex, Fine-Grained Signals, Absolute Positioning (`translate3d`), Dynamic Height Math.
- **Key Takeaway**: Shifting from full virtual-DOM diffing to fine-grained state signals eliminates unneeded component re-render cascades.

### 4. [Client Memory Budgets & Frame Scheduling](./04_client_performance_and_memory.md)
- **Topics**: Microprofiler Telemetry, <30MB JS/Lua Heap Ceilings, GC Pause Elimination, `requestAnimationFrame` / `task.defer`.
- **Key Takeaway**: Dual-layer eviction (pruning DOM viewport + trimming state store) prevents mobile OOM crashes.
