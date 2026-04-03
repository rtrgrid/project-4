# Q8 — Checkpoint recovery and migration across hosts

## Expanded overview

Because Helios stores core state on disk under its data directory, the agent's working memory can be transferred across hosts by moving that persisted state. This is much more practical than trying to reconstruct the session from raw chat alone.

## Why this matters

- Portability matters when users change machines or recover from host failure.
- Durable local state gives a concrete artifact that can be copied or backed up.
- The distinction between agent state and external running jobs must stay explicit.

## Detailed answer

### Short answer

Agent state lives under **`~/.helios/`** (especially **`helios.db`**). **Copy/sync** that tree to another machine → same **sessions, memory, metrics** from Helios’s perspective.

### What transfers

- SQLite + local config → **resume** narrative and **memory** on a new host.

### What does not magically move

- Remote jobs are **`machine_id:pid`** — you still need correct **SSH / machines config** on the new setup.
- **Transparency** here means **Helios persisted state**, not relocating arbitrary GPU jobs by magic.

### Takeaway

**Disk-backed** recovery is what makes **porting the agent’s brain** (DB) feasible across machines.

### Source snippet(s)

```md
// helios/README.md (Data)
Everything is stored locally in `~/.helios/`:

~/.helios/
  helios.db          SQLite database (sessions, metrics, memory)
  machines.json      Remote machine configs
  auth/
    auth.json        OAuth tokens and API keys
  skills/            Skill templates (editable)
  preferences.json   Last provider, claude mode
```

```ts
// helios/src/scheduler/state-store.ts
const TRIGGERS_FILE = "triggers.json";

save(sessions: SleepSession[]): void {
  const dir = getHeliosDir();
  mkdirSync(dir, { recursive: true });

  // Atomic write
  const tmpPath = this.filePath + ".tmp";
  writeFileSync(tmpPath, JSON.stringify(state, null, 2));
  renameSync(tmpPath, this.filePath);
}
```

## Practical design implications

- Migration of the agent's memory is feasible.
- External machine configuration may still need to be recreated or validated.
- Users should not confuse state portability with transparent migration of active compute jobs.

## Conclusion

Overall, Q8 highlights a deliberate architectural choice in Helios: the system favors explicit, durable, and operationally reliable mechanisms over brittle or purely implicit behavior.

## Architectural reasoning

Helios can migrate its persisted brain more easily than its live compute. The important distinction is between transferring stored agent state and transferring running jobs or external machine connectivity.

## Example scenario

Copying ~/.helios to another host may restore memory, metrics, and session artifacts. But if the original system referenced remote machines, those credentials and compute resources still need to exist and be valid on the new host.

## Trade-offs and limitations

- Migration is practical for persisted state but not magical for external dependencies.
- Users may still need to re-establish environment, auth, or machine config details.
- Even so, portable persisted state is a major advantage over chat-only systems.

## Source files referenced

- `helios/README.md`
- `helios/src/scheduler/state-store.ts`
- `helios/src/memory/memory-store.ts`

