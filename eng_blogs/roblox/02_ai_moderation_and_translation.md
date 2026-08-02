# Roblox Tech Blog Analysis: Real-Time AI Moderation & Multilingual Translation

> **Source Material**: *Roblox Tech Blog — "Scaling Real-Time Multilingual Translation & Safety Infrastructure"*  
> **Relevance**: Trust & Safety Systems, Edge Inference (<100ms), Asynchronous State Machines, Zero Cumulative Layout Shift (CLS).

---

## Executive Summary (3-Minute Read)

Roblox operates an automated, real-time AI translation and moderation pipeline serving over 16 languages. The system processes chat text and media in **under 100 milliseconds**, balancing strict COPPA/safety compliance with zero perceived UI delay.

```
[User Types Message] ---> Insert Optimistic UI (status: OPTIMISTIC, text: "Hello friend!")
                                 |
                        (HTTP POST /v1/chat)
                                 v
               +-----------------------------------+
               |  Roblox AI Transformer Pipeline   |  <-- Sub-100ms Inference
               +-----------------------------------+
                                 |
            +--------------------+--------------------+
            | (Safe / Translated)                     | (PII / Violation)
            v                                         v
   WS Push: APPROVED                         WS Push: FILTERED
   text: "¡Hola amigo!"                      text: "### ####!"
   (UI Status -> APPROVED)                   (UI Status -> FILTERED)
```

---

## Key Engineering Insights & System Patterns

### 1. Multi-State Moderation Lifecycle
Messages pass through a non-blocking asynchronous state machine:
- `OPTIMISTIC`: Inserted locally in 0ms with local user text.
- `PENDING_MOD`: Server ACK received; AI moderation / translation worker is processing.
- `APPROVED`: Passed safety filter; text translated to viewer's native language.
- `FILTERED`: Contains PII or policy violation; text replaced with `###` hashes.
- `FAILED_SEND`: Network failure or unrecoverable error.

### 2. In-Place Redaction & Zero Cumulative Layout Shift (CLS)
- **Anti-Pattern**: Deleting a flagged message or shifting the layout after moderation rejects text.
- **Roblox Solution**: Retain the exact DOM viewport height slot. If text is filtered, replace characters with `###` or render an inline `FAILED_SEND` badge with a retry button. This prevents visual jumps when 5 newer messages are already rendered below it.

### 3. Sub-100ms Edge Transformer Translation
- Roblox deploys custom lightweight Transformer LLM models at edge micro-data centers.
- Understands platform-specific slang (e.g. "obby" $\rightarrow$ obstacle course) to prevent mistranslating game-specific terms.

---

## Interviewer Probing Answers (Roblox Specific)

- **Q: Why not perform moderation on the client device to save server costs?**  
  *A*: Client-side moderation is easily bypassed via API inspection or memory injection. Heuristics can pre-check basic static lists on the client, but authoritative safety filtering **must** run on isolated server AI pipelines.
- **Q: How do you prevent global clock drift from scrambling message order during async translation?**  
  *A*: Never sort by timestamps. Order messages strictly by the server-assigned monotonic `sequence_id` ($S_{\text{id}}$).
