# Workspace Rules: System Design Prep Repository

## Candidate Profile & Goals
- Target Role: Staff / Principal Frontend Engineer (Roblox & Big Tech).
- Constraint: Busy parent with limited prep time. All explanations and summaries must be **high-yield, structured, and rapid to review**.

## System Design Framework Rules
- Always use the **RADIO framework** (Requirements, Architecture, Data Model, Interface & Rendering, Optimizations).
- Include clean **Mermaid architecture diagrams** for visual learning.
- Whenever a candidate drawing or sketch is provided, convert it into a Mermaid diagram and include a **Candidate Diagram vs. Production Reference Comparison Matrix**.
- Highlight **Interviewer Traps & Counter-Strategies** explicitly.
- Document **Tradeoffs & Discarded Alternatives** for every key architectural choice.
- Keep system design mock guides in `practice/`, technical topic deep-dives in `lessons/`, and prompt templates in `templates/`.
- Save candidate whiteboard drawing/sketch images to `practice/assets/` and link them in the corresponding `practice/` system design doc.
- Recommend bulleted side-notes on whiteboard sketches over writing verbose TypeScript interfaces (saves 5-10 mins, acts as an anchor checklist).

## Frontend vs. Backend System Design Scope Guidelines
- **Frontend Focus**: Emphasize client-side NFRs (0ms optimistic latency, frame budgets e.g. 60/120fps, memory & GC budgets < 25-30MB, stream lag under high msg bursts, DOM virtualization/pooling, cross-platform bridge abstraction, client-side moderation UI state transitions, offline replay queues & socket reconnection recovery).
- **Backend Out of Scope**: Explicitly steer away from server RPS, DB partitioning, backend microservices sharding, and database terabytes—keep the design laser-focused on client architecture.

## Interviewer Pacing & Probing Guidelines
- **Candidate-Driven Flow**: Respect the candidate's sequence. Do NOT jump ahead to introduce multi-step scenario probing (reconnection edge cases, moderation rollbacks, offline queueing) until the candidate has finished presenting their current layer or explicitly transitions into that topic.
- **Decision-Anchored Probing**: Probe primarily on architectural choices the candidate has *already* articulated. Introduce edge cases only after the candidate's base design for that module is laid out.

