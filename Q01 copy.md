# Q1 — Why does Helios use filesystem checkpointing for agent state instead of in-memory snapshots?

## Expanded overview

Helios is designed for long-running autonomous work. In that setting, the failure mode to avoid is not just a crashed Python object but total loss of the agent's accumulated research state. Disk-backed checkpointing solves that by storing state in durable storage rather than relying on the current process memory.

## Why this matters

- A research run can span many model turns, machine operations, and checkpoint cycles.
- If the process exits or the context window is reset, memory held only in RAM disappears instantly.
- Persisting to SQLite and JSON gives Helios a stable source of truth that survives interruptions.

## Detailed answer

### Short answer

Persisting state to **disk** (SQLite + config files) survives **crashes, restarts, and context-window resets**; pure in-memory snapshots do not.

### What gets persisted

- **`~/.helios/helios.db`** — sessions, **memory tree** (`memory_nodes`), metrics, etc.
- **`triggers.json`** — sleep trigger state (`helios/src/scheduler/state-store.ts`)

### Why not only RAM?

- **Process exit** → in-memory state is gone.
- **Context checkpoint** → Helios **archives provider history** but continues from **stored memory + briefing** (`context-gate.ts`, `orchestrator.ts`). That only works if memory was durable **before** the reset.

### Takeaway

Filesystem / DB checkpointing makes **resume** and **long autonomous runs** reliable; snapshots of heap-only state would not.

### Source snippet(s)

```ts
// helios/src/memory/memory-store.ts
read(path: string): MemoryNode | null {
  path = validatePath(path);
  const row = this.stmt(
    `SELECT path, gist, content, is_dir, created_at, updated_at
     FROM memory_nodes
     WHERE session_id = ? AND path = ?`,
  )
  .get(this.sessionId, path) as Record<string, unknown> | undefined;

  return row ? rowToNode(row) : null;
}

write(path: string, gist: string, content?: string | null): void {
  path = validatePath(path);
  const now = Date.now();
  const isDir = content === undefined || content === null;

  // Auto-create parent directories
  this.ensureParents(path);

  this.stmt(
    `INSERT INTO memory_nodes (session_id, path, gist, content, is_dir, created_at, updated_at)
     VALUES (?, ?, ?, ?, ?, ?, ?)
     ON CONFLICT(session_id, path) DO UPDATE SET
       gist = excluded.gist,
       content = excluded.content,
       is_dir = excluded.is_dir,
       updated_at = excluded.updated_at`,
  ).run(this.sessionId, path, gist, isDir ? null : content, isDir ? 1 : 0, now, now);
}
```

```ts
// helios/src/scheduler/state-store.ts
const TRIGGERS_FILE = "triggers.json";

save(sessions: SleepSession[]): void {
  const state: PersistedState = {
    version: 1,
    sessions: sessions.map((s) => ({
      session: {
        ...s,
        trigger: {
          ...s.trigger,
          satisfiedLeaves: Array.from(s.trigger.satisfiedLeaves),
        },
      },
    })),
  };

  const dir = getHeliosDir();
  mkdirSync(dir, { recursive: true });

  // Atomic write
  const tmpPath = this.filePath + ".tmp";
  writeFileSync(tmpPath, JSON.stringify(state, null, 2));
  renameSync(tmpPath, this.filePath);
}
```

## Practical design implications

- Reliable resume after crashes or restarts.
- Context checkpointing can safely archive chat history while preserving working state.
- The agent can behave more like a persistent system than a one-shot chat session.

## Conclusion

Overall, Q1 highlights a deliberate architectural choice in Helios: the system favors explicit, durable, and operationally reliable mechanisms over brittle or purely implicit behavior.

## Architectural reasoning

Filesystem checkpointing gives Helios durability across failures. In-memory snapshots are fast but fundamentally tied to the lifetime of the current process. Helios is meant to operate over long horizons, so the system chooses persistence over ephemeral convenience.

## Example scenario

For example, if an autonomous training session runs overnight and the process is interrupted, a memory tree stored in SQLite can still be reloaded the next morning. A purely in-memory design would require reconstructing the state from scratch or from fragile chat history.

## Trade-offs and limitations

- Disk-backed persistence is slightly more complex than simple in-memory objects.
- Consistency and atomic writes become important design concerns.
- The payoff is much stronger recovery and continuity guarantees.

## Source files referenced

- `helios/src/memory/memory-store.ts`
- `helios/src/scheduler/state-store.ts`
- `helios/src/memory/context-gate.ts`
- `helios/src/core/orchestrator.ts`

