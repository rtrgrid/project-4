# Helios: Architecture & design Q&A

Grounded in the Helios codebase. **Sources** (examples):

| Area | Paths |
|------|--------|
| Docs & overview | `helios/README.md` |
| Context checkpoints | `helios/src/memory/context-gate.ts`, `helios/src/core/orchestrator.ts` |
| Metrics | `helios/src/metrics/parser.ts`, `helios/src/tools/compare-runs.ts` |
| Sleep / triggers | `helios/src/scheduler/` |

Some answers mix **documented behavior** with normal **design rationale**.

---

## Q1 — Filesystem checkpointing vs in-memory snapshots

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

## Q2 — Composable sleep triggers vs hard-coded timing

### Short answer

Hard-coded timing assumes a **fixed delay**; real ML jobs vary in length and outcome. **Composable triggers** express “wake when *any* of these real conditions holds.”

### What hard-coded timing misses

- Training can finish **early** or **late**.
- You care about **process exit**, **metric thresholds**, **files**, not just the clock.

### What composable triggers add

- Combine conditions: e.g. **process exit**, **metric** (loss below X), **timer**, **file / resource** (`helios/README.md` *Sleep & Wake*; `helios/src/scheduler/triggers/types.ts`).
- Replace *“sleep 2 hours”* with *“wake when training ends **or** loss &lt; X **or** 2h cap.”*

### Takeaway

One **declarative** `sleep(..., triggers: [...])` matches messy real workflows better than a single baked-in sleep policy.

### Source snippet(s)

```md
// helios/README.md (Sleep & Wake)
sleep(reason: "waiting for training to finish", triggers: [
  { type: "process_exit", machine_id: "gpu1", pid: 4521 },
  { type: "metric", metric_name: "loss", threshold: 1.0, direction: "below" },
  { type: "timer", minutes: 120 }
])
```

```ts
// helios/src/scheduler/triggers/types.ts
export interface TimerCondition {
  kind: "timer";
  wakeAt: number; // Unix timestamp ms
}

export interface ProcessExitCondition {
  kind: "process_exit";
  machineId: string;
  pid?: number;
  processPattern?: string;
  expectedExitCode?: number;
}

export interface CompositeTrigger {
  op: "and" | "or";
  children: TriggerExpression[];
}

export type TriggerExpression = TriggerCondition | CompositeTrigger;

export const DEFAULT_POLL_INTERVALS: Record<TriggerCondition["kind"], number> =
  {
    timer: 1000,
    process_exit: 15_000,
    file: 10_000,
    metric: 30_000,
    resource: 10_000,
    user_message: 0, // event-driven
  };
```

## Q3 — Metric parsing and observability without invasive instrumentation

### Short answer

Helios **parses logs/stdout** with patterns; training code stays normal **`print` / logging**—no Helios SDK inside the training loop.

### How it works

1. **Config** names metrics: `metricNames`, `metricPatterns` in `helios.json` (see `helios/README.md`).
2. **`parser.ts`** applies regexes / `key=value` style rules to **lines of output**.
3. **`collector.ts`** feeds remote log tails into the same parsing pipeline.

### Why “non-invasive”

- No per-step API calls from user training scripts.
- **Declarative** patterns turn **existing** prints into structured series for UI + tools.

### Takeaway

Observable behavior comes from **conventions + parsing**, not from instrumenting every framework hook.

### Source snippet(s)

```ts
// helios/src/metrics/parser.ts
export function patternsFromNames(names: string[]): MetricPatterns {
  const patterns: Record<string, RegExp> = {};
  for (const name of names) {
    // Match name=value or name: value
    patterns[name] = new RegExp(
      `(?:^|\\s|,)${escapeRegex(name)}\\s*[=:]\\s*([+-]?\\d+\\.?\\d*(?:e[+-]?\\d+)?)`,
      "i",
    );
  }
  return { patterns };
}

export function parseWithPatterns(
  output: string,
  mp: MetricPatterns,
): MetricPoint[] {
  const points: MetricPoint[] = [];
  const now = Date.now();
  const lines = output.split("\\n");

  for (const line of lines) {
    for (const [name, re] of Object.entries(mp.patterns)) {
      const match = re.exec(line);
      if (match && match[1]) {
        const value = parseFloat(match[1]);
        if (isFinite(value)) {
          points.push({
            metricName: name,
            value,
            timestamp: now,
          });
        }
      }
    }
  }

  return points;
}
```

## Q4 — Explicit checkpoint boundaries vs arbitrary code points

### Short answer

Checkpoints run at a **well-defined point in the orchestrator** (after a full model turn), not at random places in “user code.”

### Sequence (orchestrator)

After `send` completes → **`maybeCheckpoint`** → roughly:

1. **Gist** generation  
2. **Persist** gist + active tasks into **memory**  
3. **`resetHistory`** with a **briefing**  

See `helios/src/core/orchestrator.ts`.

### Why not “anywhere”?

- Mid-turn / mid–**tool-call** checkpoint → **inconsistent** conversation + tool state.
- A **single boundary** makes archive + memory write + history reset **coherent**.

### Takeaway

**Explicit** boundaries avoid corrupting the provider session; **implicit** arbitrary checkpoints would.

### Source snippet(s)

```ts
// helios/src/core/orchestrator.ts
private async *maybeCheckpoint(session: Session): AsyncGenerator<AgentEvent> {
  if (!this._contextGate || !this.activeProvider) return;

  const model = this.activeProvider.currentModel;
  const threshold = this._contextGate.checkThreshold(model, this._lastInputTokens);
  if (!threshold) return;

  const gist = await this.generateCheckpointGist(session);
  const briefing = this._contextGate.performCheckpointWithGist(gist);

  this.activeProvider.resetHistory(session, briefing);
  this._lastInputTokens = 0;

  yield {
    type: "text",
    text: "\\n\\n---\\n*[Context checkpoint — conversation archived, continuing from memory]*\\n\\n",
    delta: "\\n\\n---\\n*[Context checkpoint — conversation archived, continuing from memory]*\\n\\n",
  };
}
```

## Q5 — Separating state checkpointing from control flow

### Short answer

**Durable state** (memory DB, summaries) and **the chat loop** (`send` / `resetHistory`) are different layers. You can reset **context** without throwing away **experiment knowledge**.

### Two layers

| Layer | What it is | Where |
|-------|------------|--------|
| **State** | Memory tree, active tasks, metric hooks | `MemoryStore` / SQLite, `context-gate.ts` |
| **Control flow** | Autonomous loop, provider turns | `orchestrator.ts` |

### Advantage

- **Token budget** → replace long chat with a **briefing**; **memory** still holds `/experiments`, `/best`, etc.
- The **agent keeps running**; only the **serialized conversation buffer** is replaced.

### Takeaway

Separating **what we persist** from **how we drive the model this step** avoids conflating “checkpoint chat” with “lose all research state.”

### Source snippet(s)

```ts
// helios/src/memory/context-gate.ts
performCheckpointWithGist(gist: string): string {
  // Save the model's gist
  this.memory.write(
    "/context/gist",
    "Model-generated summary of conversation before checkpoint",
    gist,
  );

  // Save active tasks to /context/active-tasks
  this.saveActiveTasks();

  return this.buildBriefing(gist);
}

buildBriefing(gist: string | null): string {
  const tree = this.memory.formatTree("/");

  const parts = [
    "=== CONTEXT CHECKPOINT ===",
    "You are Helios, continuing an autonomous ML research session.",
    "Your previous conversation has been archived.",
  ];

  if (gist) {
    parts.push(
      "\\n## Your gist (written by you before the checkpoint):\\n",
      gist,
    );
  }

  parts.push(
    "\\n## Memory tree:\\n",
    tree,
    "\\nUse memory_read(path) for details on any item above.",
    "Use memory_write to store new findings as you work.",
    "Check /sources/ for prior work you've built on — cite these in any writeups.",
    "Continue working toward your goal.",
  );

  return parts.join("\\n");
}
```

## Q6 — Parsing metrics and regression-style comparison

### Short answer

You **configure** names/patterns once; parsing **fills** the store. **`compare_runs`** then diffs **stored** metrics between runs—no per-step manual reporting in scripts.

### Pipeline

- **Declare** metrics (names + optional regex) → **parse** logs continuously → **store** time series.
- **`compare_runs`** (`helios/src/tools/compare-runs.ts`): baseline vs experiment, **latest / min / max**, **deltas** on shared names.
- **`analyzer.ts`**: trend-style stats on **stored** points.

### “Without manual metric definitions” means

- **Not** hand-calling a reporting API on every training step.
- You still **name** what to parse in config—that is lightweight vs embedding calls everywhere.

### Takeaway

Regression detection is **tool-driven over parsed series**, not ad-hoc copy-paste of numbers from logs.

### Source snippet(s)

```ts
// helios/src/tools/compare-runs.ts
const comparisons = names.map((name) => {
  const a = summaryA[name];
  const b = summaryB[name];
  const delta = a && b ? b.latest - a.latest : null;
  const direction =
    delta === null
      ? "n/a"
      : delta < -0.0001
        ? "decreased"
        : delta > 0.0001
          ? "increased"
          : "unchanged";

  return {
    metric: name,
    baseline: a
      ? { latest: a.latest, min: a.min, max: a.max, samples: a.count }
      : null,
    experiment: b
      ? { latest: b.latest, min: b.min, max: b.max, samples: b.count }
      : null,
    delta,
    direction,
  };
});

return JSON.stringify({
  task_a: taskA,
  task_b: taskB,
  comparisons,
});
```

## Q7 — Composable triggers vs a pure event-driven design

### Short answer

Not every wake condition is a **single OS event**. Composition lets one **`sleep`** describe **timer + process + metric + file + …** under one abstraction.

### Why not “only events”?

- **Metric thresholds** and **file stability** often need **polling** or layered checks, not one kernel event.
- A **global event bus** still does not spell out **OR/AND** of heterogeneous conditions.

### What Helios does

- **Composable expressions** (`helios/src/scheduler/triggers/types.ts`).
- **`TriggerScheduler`** + **`EventEmitter`** for **wake** (`trigger-scheduler.ts`, `sleep-manager.ts`).
- **User message** can wake sessions in an **event-like** path.

### Takeaway

**Composition** describes *what* should wake the agent; the runtime may still use **events + polling** underneath.

### Source snippet(s)

```ts
// helios/src/scheduler/triggers/types.ts
export type TriggerCondition =
  | TimerCondition
  | ProcessExitCondition
  | FileCondition
  | MetricCondition
  | ResourceCondition
  | UserMessageCondition;

export interface CompositeTrigger {
  op: "and" | "or";
  children: TriggerExpression[];
}

export type TriggerExpression = TriggerCondition | CompositeTrigger;

export const DEFAULT_POLL_INTERVALS: Record<TriggerCondition["kind"], number> =
  {
    timer: 1000,
    process_exit: 15_000,
    file: 10_000,
    metric: 30_000,
    resource: 10_000,
    user_message: 0, // event-driven
  };
```

```ts
// helios/src/scheduler/trigger-scheduler.ts
onUserMessage(): void {
  for (const [id, session] of this.sessions) {
    session.wakeReason = "user_interrupt";
    session.wokeAt = Date.now();
    session.trigger.status = "satisfied";
    this.sessions.delete(id);
    this.emit("wake", session, "User sent a message");
  }
}

private async evaluate(
  expr: TriggerExpression,
  path: string,
  trigger: Trigger,
): Promise<boolean> {
  if ("op" in expr) {
    // Composite trigger — short-circuit to avoid unnecessary evaluations
    const children = expr.children ?? [];
    if (children.length === 0) return expr.op === "and";
    if (expr.op === "or") {
      for (let i = 0; i < children.length; i++) {
        if (await this.evaluate(children[i], `${path}.${i}`, trigger)) return true;
      }
      return false;
    } else {
      // AND
      for (let i = 0; i < children.length; i++) {
        if (!(await this.evaluate(children[i], `${path}.${i}`, trigger))) return false;
      }
      return true;
    }
  }

  // Check if already satisfied (latching for AND)
  if (trigger.satisfiedLeaves.has(path)) return true;

  const satisfied = await this.evaluateCondition(expr);
  if (satisfied) {
    trigger.satisfiedLeaves.add(path);
  }
  return satisfied;
}
```

## Q8 — Checkpoint recovery and migration across hosts

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

## Q9 — Parsing output vs explicit metric reporting APIs

### Short answer

Training stacks differ (PyTorch, Lightning, custom). **Parsing stdout** avoids a **Helios-specific reporting SDK** inside every script.

### Benefits

- Keep **normal** `print` / logger output.
- Add or adjust **patterns** in config instead of editing training code.
- **Lower coupling** between experiment code and the agent runtime.

### References

- `helios/src/metrics/parser.ts`  
- `helios/README.md` — *Metric Tracking*

### Takeaway

**Convention + parsing** scales across projects better than mandatory explicit reporting hooks everywhere.

### Source snippet(s)

```md
// helios/README.md (Metric Tracking)
Training scripts print metrics to stdout. Helios parses them automatically:

# key=value format (detected via metric_names)
print(f"loss={loss:.4f} acc={acc:.4f} lr={lr:.6f}")

# Custom patterns (detected via metric_patterns)
print(f"Step {step}: Loss {loss:.4f}")
```

```ts
// helios/src/metrics/parser.ts
export function parseWithPatterns(
  output: string,
  mp: MetricPatterns,
): MetricPoint[] {
  const points: MetricPoint[] = [];
  const now = Date.now();
  const lines = output.split("\n");

  for (const line of lines) {
    for (const [name, re] of Object.entries(mp.patterns)) {
      const match = re.exec(line);
      if (match && match[1]) {
        const value = parseFloat(match[1]);
        if (isFinite(value)) {
          points.push({
            metricName: name,
            value,
            timestamp: now,
          });
        }
      }
    }
  }

  return points;
}
```

## Q10 — Explicit, developer-aware boundaries vs fully automatic dumps

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

## Code snippets and file references

Below are small excerpts from the Helios codebase that back up each answer.

### Snippet for Q1 – Persistent memory and triggers state

```ts
// helios/src/memory/memory-store.ts
export class MemoryStore {
  private sessionId: string;
  // ...
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
}
```

```ts
// helios/src/scheduler/state-store.ts
const TRIGGERS_FILE = "triggers.json";

export class TriggerStateStore {
  save(sessions: SleepSession[]): void {
    const state: PersistedState = { /* ... */ };
    const dir = getHeliosDir();
    mkdirSync(dir, { recursive: true });
    const tmpPath = this.filePath + ".tmp";
    writeFileSync(tmpPath, JSON.stringify(state, null, 2));
    renameSync(tmpPath, this.filePath);
  }
}
```

### Snippet for Q2 – Sleep with composable triggers

```md
<!-- helios/README.md (Sleep & Wake) -->
The agent can put itself to sleep with composable triggers:

sleep(reason: "waiting for training to finish", triggers: [
  { type: "process_exit", machine_id: "gpu1", pid: 4521 },
  { type: "metric", metric_name: "loss", threshold: 1.0, direction: "below" },
  { type: "timer", minutes: 120 }
])
```

```ts
// helios/src/scheduler/triggers/types.ts
export type TriggerCondition =
  | { kind: "timer"; /* ... */ }
  | { kind: "process_exit"; /* ... */ }
  | { kind: "metric_threshold"; /* ... */ }
  | { kind: "file_change"; /* ... */ }
  | { kind: "user_message"; /* ... */ };

export interface CompositeTrigger {
  kind: "composite";
  op: "and" | "or";
  children: TriggerExpression[];
}
```

### Snippet for Q3 – Metric parsing from stdout/logs

```ts
// helios/src/metrics/parser.ts
export function parseWithPatterns(
  output: string,
  mp: MetricPatterns,
): MetricPoint[] {
  const points: MetricPoint[] = [];
  const now = Date.now();
  const lines = output.split("\n");

  for (const line of lines) {
    for (const [name, re] of Object.entries(mp.patterns)) {
      const match = re.exec(line);
      if (match && match[1]) {
        const value = parseFloat(match[1]);
        if (isFinite(value)) {
          points.push({ metricName: name, value, timestamp: now });
        }
      }
    }
  }
  return points;
}
```

### Snippet for Q4 – Orchestrator checkpoint boundary

```ts
// helios/src/core/orchestrator.ts
private async *maybeCheckpoint(session: Session): AsyncGenerator<AgentEvent> {
  if (!this._contextGate || !this.activeProvider) return;

  const model = this.activeProvider.currentModel;
  const threshold = this._contextGate.checkThreshold(model, this._lastInputTokens);
  if (!threshold) return;

  const gist = await this.generateCheckpointGist(session);
  const briefing = this._contextGate.performCheckpointWithGist(gist);

  this.activeProvider.resetHistory(session, briefing);
  this._lastInputTokens = 0;

  yield {
    type: "text",
    text: "\n\n---\n*[Context checkpoint — conversation archived, continuing from memory]*\n\n",
    delta: "\n\n---\n*[Context checkpoint — conversation archived, continuing from memory]*\n\n",
  };
}
```

### Snippet for Q5 – ContextGate and memory-based briefing

```ts
// helios/src/memory/context-gate.ts
export class ContextGate {
  // ...
  performCheckpointWithGist(gist: string): string {
    this.memory.write(
      "/context/gist",
      "Model-generated summary of conversation before checkpoint",
      gist,
    );
    this.saveActiveTasks();
    return this.buildBriefing(gist);
  }

  buildBriefing(gist: string | null): string {
    const tree = this.memory.formatTree("/");
    const parts = [
      "=== CONTEXT CHECKPOINT ===",
      "You are Helios, continuing an autonomous ML research session.",
      "Your previous conversation has been archived.",
    ];
    // ... include gist and tree ...
    return parts.join("\n");
  }
}
```

### Snippet for Q6 – Comparing runs using stored metrics

```ts
// helios/src/tools/compare-runs.ts
export function createCompareRunsTool(
  metricStore: MetricStore,
): ToolDefinition {
  return {
    name: "compare_runs",
    // ...
    execute: async (args) => {
      const taskA = args.task_a as string;
      const taskB = args.task_b as string;

      const summaryA = metricStore.getTaskSummary(taskA);
      const summaryB = metricStore.getTaskSummary(taskB);
      // ...
      const comparisons = names.map((name) => {
        const a = summaryA[name];
        const b = summaryB[name];
        const delta = a && b ? b.latest - a.latest : null;
        // ... direction logic ...
        return { metric: name, baseline: a, experiment: b, delta, direction };
      });
      return { taskA, taskB, comparisons };
    },
  };
}
```

### Snippet for Q7 – Trigger scheduler using expressions and events

```ts
// helios/src/scheduler/trigger-scheduler.ts
export class TriggerScheduler extends EventEmitter<TriggerSchedulerEvents> {
  // ...
  async start(session: SleepSession): Promise<void> {
    const trigger = session.trigger;
    // schedule periodic evaluation of trigger.expression
  }

  onUserMessage(): void {
    // Handle user input event — wakes any sleeping session with user_message condition
  }
}
```

### Snippet for Q8 – Data layout under ~/.helios

```md
<!-- helios/README.md (Data section) -->
~/.helios/
  helios.db          SQLite database (sessions, metrics, memory)
  machines.json      Remote machine configs
  auth/
    auth.json        OAuth tokens and API keys
  skills/            Skill templates (editable)
  preferences.json   Last provider, claude mode
```

### Snippet for Q9 – Metric tracking from prints

```md
<!-- helios/README.md (Metric Tracking) -->
# key=value format (detected via metric_names)
print(f"loss={loss:.4f} acc={acc:.4f} lr={lr:.6f}")

# Custom patterns (detected via metric_patterns)
print(f"Step {step}: Loss {loss:.4f}")
```

### Snippet for Q10 – System prompt about memory & checkpoints

```ts
// helios/src/init.ts (excerpt)
const SYSTEM_PROMPT = `You have a persistent virtual filesystem for storing knowledge across context checkpoints.
