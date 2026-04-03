# Q10 — Explicit, developer-aware boundaries vs fully automatic dumps

## Expanded overview

Helios pairs orchestrator-managed checkpoints with explicit developer memory writes. This combination recognizes that automatic archiving can preserve conversation continuity, but durable structured notes are still required for robust recovery and long-term autonomy.

## Why this matters

- Not all information in a chat transcript is equally valuable for future work.
- Important findings need to be stored in stable, queryable locations such as /best or /observations.
- A predictable checkpoint policy is easier to trust than arbitrary internal dumps.

## Detailed answer

### Short answer

System checkpoints follow **token policy** and a **fixed sequence** (gist → memory → `resetHistory`). **Critical** facts belong in **memory tools** because **chat** can be archived anytime.

### Two ideas

1. **Orchestrator-controlled** checkpoint — predictable **when** and **how** history resets (`orchestrator.ts`).
2. **Developer responsibility** — write durable notes to **`/experiments`, `/best`, …** (`init.ts` prompts stress this).

### Why not “automatic everything”?

- Dumping arbitrary internal state at random points ≠ **useful experiment knowledge**.
- **Structured memory** + **explicit** checkpoint beats opaque auto-snapshots for **recovery**.

### Takeaway

**Explicit** boundaries + **memory writes** = recovery you can **reason about**; blind automatic checkpointing of everything would not.

### Source snippet(s)

```md
// helios/src/init.ts (Memory System)
You have a persistent virtual filesystem for storing knowledge across context checkpoints.
When the conversation gets too long, your context will be checkpointed: history is archived and you'll receive a briefing with your memory tree.

CRITICAL: Write to memory IMMEDIATELY and CONTINUOUSLY. Your context can be checkpointed at any time — anything not in memory will be LOST FOREVER.
```

```ts
// helios/src/core/orchestrator.ts
this.activeProvider.resetHistory(session, briefing);
this._lastInputTokens = 0;

yield {
  type: "text",
  text: "\n\n---\n*[Context checkpoint — conversation archived, continuing from memory]*\n\n",
  delta: "\n\n---\n*[Context checkpoint — conversation archived, continuing from memory]*\n\n",
};
```

## Practical design implications

- The agent can survive context resets without losing its most important conclusions.
- Researchers can inspect and reason about persisted knowledge directly.
- Good system behavior depends on both architecture and disciplined memory usage.

## Conclusion

Overall, Q10 highlights a deliberate architectural choice in Helios: the system favors explicit, durable, and operationally reliable mechanisms over brittle or purely implicit behavior.

## Architectural reasoning

Helios does not assume automatic checkpointing alone is enough. The design explicitly combines system-managed checkpoints with developer-managed memory entries so the most important research knowledge becomes durable and inspectable.

## Example scenario

If an experiment identifies a new best configuration, storing it in /best means that information remains available after a context reset. If it exists only in transient chat, it may be lost even if a checkpoint summary is generated.

## Trade-offs and limitations

- This approach requires discipline: the agent or developer must keep memory updated.
- It is more explicit than invisible automatic dumping.
- However, it produces a knowledge base that is far easier to recover, audit, and build upon.

## Source files referenced

- `helios/src/init.ts`
- `helios/src/core/orchestrator.ts`
- `helios/src/memory/context-gate.ts`

