# Roblox Tech Blog Analysis: Client Performance & Memory Management

> **Source Material**: *Roblox Engineering — "Microprofiler Telemetry, Heap Budgets & GC Pause Elimination"*  
> **Relevance**: Memory Footprint (<30MB Heap), Frame Budget Scheduling, Ingest Batching, Object Pooling.

---

## Executive Summary (3-Minute Read)

Roblox runs on low-end mobile devices (2GB RAM, 3G networks) up to high-end multi-monitor gaming rigs. The client engine enforces strict **30MB JS/Lua memory budgets for UI subsystems** and relies on **Microprofiler** telemetry to catch frame drops over 16.6ms.

```
       [ Microprofiler Telemetry & Frame Budget Monitor (16.6ms Target) ]
                                       |
    +----------------------------------+----------------------------------+
    |                                                                     |
[ Dual-Layer Eviction Engine ]                          [ Image & Asset Pipeline ]
1. DOM Windowing (~15-20 rows mounted)                 1. img.decoding = "async" / Image.decode()
2. JS Store Truncation (Trim past 1,000 items)         2. AbortController cancel on viewport exit
3. IndexedDB Page-Chunk Offloading                     3. Scroll velocity prefetch throttling
```

---

## Key Engineering Insights & System Patterns

### 1. Dual-Layer Memory Eviction
- **Problem**: Virtualizing DOM nodes prevents DOM memory bloat, but retaining 10,000 items in central JS state memory bloats heap (>100MB), triggering severe Garbage Collection (GC) pauses.
- **Roblox Strategy**:
  - **DOM Layer**: Render only visible items (~15-20 rows) using windowing.
  - **Store Layer**: Enforce max 1,000 items ceiling in JS memory array. Evict older items to IndexedDB in **discrete page chunks**.
  - **Back-Scroll Recovery**: If user scrolls back up, lazily retrieve page chunks from IndexedDB without network refetches.

### 2. Off-Main-Thread Async Image Decoding
- Default image rasterization occurs synchronously on the main thread during layout/paint, dropping frames during fast scrolling.
- **Solution**: Use `img.decoding = "async"` or explicit `Image.prototype.decode()` promise before DOM append.

### 3. Rapid Scroll Fling Request Cancellation
- Fast scrolling (25 items/sec) enqueues dozens of concurrent image fetches, saturating HTTP/2 browser socket pools.
- **Solution**:
  - Each image component manages an `AbortController`.
  - When `IntersectionObserver` fires `isIntersecting === false` before image completes loading, call `abortController.abort()`.
  - If scroll velocity $v > 1,500\text{px/sec}$, pause prefetching until `scrollend`.

---

## Interviewer Probing Answers (Roblox Specific)

- **Q: How do you track down micro-stutters during scrolling?**  
  *A*: Use Roblox's **Microprofiler** (or Chrome DevTools Performance Timeline) to inspect main-thread task durations per frame. Ensure layout measurement, state dispatches, and image decoding stay under 16.6ms total.
- **Q: Why evict from store in page chunks instead of single items?**  
  *A*: Single-item eviction destroys cursor pagination index boundaries. Page-chunk eviction preserves server cursor continuity when paging back up from local storage.
