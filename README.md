# Session Continuity Framework

**Production-tested continuity system for AI agents with no persistent memory**

A 4-layer state management architecture that gives session-based AI agents operational continuity — solving the fundamental problem that your AI operator forgets everything the moment a conversation ends. In production since October 2024.

---

## The Problem

Session-based AI agents have amnesia. Every conversation starts from zero. This creates a cascade of operational failures:

- **Context loss:** The agent doesn't know what happened yesterday, what's in progress, or what failed last time. Every session burns tokens re-explaining state.
- **Silent degradation:** Without memory, the agent can't detect patterns across sessions — recurring bugs, drifting priorities, accumulating technical debt.
- **Catastrophic cutoffs:** Token limits are invisible to both parties. The agent says "plenty of capacity left" and then hits the wall mid-task, losing all unsaved work.
- **Ramp-up tax:** Each session spends 10-20% of its token budget just getting the agent back to where the last session ended.

The standard solution — "just give the AI more context" — doesn't scale. Dumping every document into the context window burns your token budget before the agent does any actual work. You need selective retrieval, not brute force.

---

## Architecture

```
┌─────────────────────────────────────────────┐
│         SESSION CONTINUITY SYSTEM            │
├─────────────────────────────────────────────┤
│                                             │
│  Layer 1: Living State Document             │
│  ┌─────────────────────────────────────┐    │
│  │ session-context.md                  │    │
│  │ Updated 350+ times                  │    │
│  │ Current state, priorities, flags    │    │
│  │ Recent session history (10 sessions)│    │
│  └─────────────────────────────────────┘    │
│                    ↓                        │
│  Layer 2: Session Protocols                 │
│  ┌─────────────────────────────────────┐    │
│  │ startup-skill → load state → route  │    │
│  │ end-skill → capture → update → save │    │
│  │ Standardized handoff contracts      │    │
│  └─────────────────────────────────────┘    │
│                    ↓                        │
│  Layer 3: Semantic Retrieval                │
│  ┌─────────────────────────────────────┐    │
│  │ 17,428 embedded chunks              │    │
│  │ Sub-second retrieval via pgvector   │    │
│  │ Search before load (token savings)  │    │
│  └─────────────────────────────────────┘    │
│                    ↓                        │
│  Layer 4: Token Management                  │
│  ┌─────────────────────────────────────┐    │
│  │ Proactive checkpoints every 25k     │    │
│  │ Budget estimation before loading    │    │
│  │ 75% warning → checkpoint prompt     │    │
│  │ Decision points at 70%, 80%         │    │
│  └─────────────────────────────────────┘    │
│                                             │
└─────────────────────────────────────────────┘
```

### Layer 1: Living State Document

A single `session-context.md` file serves as the system's ground truth. Updated at the end of every session with current state, active priorities, recent session history, and caution flags. The agent reads this first — always — and gets zero-ramp-up orientation in seconds.

This isn't a log. It's a living document that gets rewritten, not appended to. Old sessions roll off. Current state stays current. The document answers three questions: *What's happening now? What's next? What could go wrong?*

The schema has 5 enforced sections:

| Section | Purpose |
|---------|----------|
| **Next Session** | Focus area + context for the next agent to pick up cold |
| **Active State** | Current priorities, work in progress, patterns discovered |
| **Recent Sessions** | Rolling table of last 10: number, date, focus, key output |
| **Flags** | Urgent items surfaced for immediate attention (deadlines, blockers) |
| **Structure Notes** | Database version, embedding stats, workflow IDs the agent needs |

Every section has a defined purpose. An agent that reads this document knows what to do next without asking. The schema is the reason a new session is productive within one message — not the document's existence, but its structure.

### Layer 2: Session Protocols

Standardized startup and shutdown skills enforce consistency across every session:

- **Startup:** Read state → announce session number → route to focus area. No summarizing, no guessing — just load and ask.
- **End:** Capture decisions → update session-context → flag incomplete work → save journey notes. The agent never "just stops" — it always hands off cleanly.
- **Handoff contracts:** Standardized schema (action / files changed / state / errors / next steps) across all 8 production skills. Prevents the most common failure mode: the agent did work but nobody recorded what or why.

### Layer 3: Semantic Retrieval

When the living state document isn't enough — when the agent needs context from 50 sessions ago — semantic search retrieves it without loading entire files:

- 17,428 embedded chunks (14,335 conversation + 3,093 document)
- Sub-second retrieval via Supabase pgvector
- Hash-based incremental updates (<10s vs 5+ min full rebuilds)
- "Search before load" policy: find relevant chunks first, load full files only when necessary

This is the difference between burning 50k tokens loading "everything that might be relevant" and spending 2k tokens on precisely what you need.

### Layer 4: Token Management

Born from a production incident where the AI agent exhausted its context window mid-session with no warning. Formal incident analysis revealed the root cause: zero visibility into token consumption for either party. The agent auto-loaded 51k tokens of context without any budget check.

The fix — a 3-phase protocol:

1. **Planning:** Estimate token cost before loading documents. Budget-check before action.
2. **Monitoring:** Proactive announcements every 25k tokens. No surprises.
3. **Checkpoints:** Decision points at 70% and 80% capacity. Mandatory save at 75%. The agent asks "checkpoint now?" instead of silently running out.

---

## Key Insight

**The hardest continuity problem isn't technical — it's behavioral.**

Building the state document was straightforward. The hard part was making the agent *actually use it consistently*. Without enforced protocols, the agent would skip the startup read "to save tokens," forget to update at shutdown "because the conversation ended abruptly," or load too much context "just to be thorough."

Every layer exists because a behavioral failure happened first:
- Layer 1 exists because the agent kept re-asking questions it already had answers to.
- Layer 2 exists because sessions ended without recording what happened.
- Layer 3 exists because brute-force context loading was unsustainable.
- Layer 4 exists because a mid-session crash lost an entire session's work.

The system doesn't trust the agent to remember to do the right thing. It makes the right thing the default path.

---

## Results

| Metric | Before | After |
|--------|--------|-------|
| Unexpected session cutoffs | Recurring | **Zero in 200+ sessions** |
| Session ramp-up time | 10-20% of token budget | **Seconds** (read state doc) |
| Context retrieval | Load everything or guess | **Sub-second semantic search** |
| Cross-session pattern detection | None | **17,428 searchable chunks** |
| Session state updates | Ad hoc, often skipped | **Continuous since October 2024** |
| Token waste from over-loading | ~51k tokens/incident | **Budget-checked before every load** |

---

## Failure Catalog

Real failures that shaped the system:

**Token exhaustion incident (Session 48).** Agent said "plenty of capacity left." Hit the wall 3 minutes later. Lost all unsaved work. Root cause: neither party had visibility into consumption. Fix: proactive management protocol with checkpoints. Zero recurrence in 200+ sessions.

**Journey capture overwrite.** A skill used `write` instead of `append`, destroying ~150 sessions of organizational learning history. Partially recovered from embeddings. Fix: updated skill logic. Lesson codified: "ambiguous skill instructions default to destructive behavior."

**Context rot.** Agent loaded a stale version of session-context.md and made decisions based on outdated priorities. Fix: startup skill validates recency. Lesson: state documents that aren't actively maintained become liability, not asset.

---

## Why It Matters

Incident response applied to AI systems. The token exhaustion incident followed the same arc as any production outage: something failed, neither party had visibility into why, the root cause turned out to be an observability gap, and the fix was a monitoring protocol with defined checkpoints. The session overwrite incident was a classic destructive-default bug — ambiguous instructions, no write-mode validation, irreversible action. Both failures produced permanent system changes. Both are now documented anti-patterns.

The broader pattern: any system operating autonomously needs state management, handoff contracts, and behavioral enforcement — not just capability. An agent that can do the work but can't hand off cleanly, or can't detect when it's about to run out of resources, is a reliability problem. The architecture here is the same pattern behind incident management runbooks, operational handoff protocols, and change management systems. The constraint (one party forgets everything overnight) makes the pattern harder to build, not different in kind.

---

## Built With

- **State management:** Markdown living documents (session-context.md)
- **Protocols:** 8 versioned skills with standardized handoff contracts
- **Retrieval:** Supabase pgvector, sentence-transformers (all-MiniLM-L6-v2), n8n webhooks
- **Token tracking:** Custom 3-phase protocol (plan → monitor → checkpoint)
- **Production scale:** 350+ sessions, 17,428 embedded chunks, 50,000+ words of documentation

---

## License

MIT

## Author

**Jordan Waxman** — [mrminor-dev.github.io](https://mrminor-dev.github.io)

14 years operations leadership — building production AI infrastructure since 2025. This framework was developed operating an AI-as-COO for a real business, where session continuity isn't a nice-to-have — it's the difference between an agent that compounds knowledge and one that starts from scratch every day.
