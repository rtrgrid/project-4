# Q2 — Composable sleep triggers vs hard-coded timing

## Expanded overview

Sleep in an ML research agent is not just about waiting. It is about expressing the exact conditions under which the agent should resume work. Helios therefore models wake-up logic as composable conditions instead of a single fixed timeout.

## Why this matters

- Training jobs do not finish on a predictable wall-clock schedule.
- Useful wake conditions are heterogeneous: process exits, metric thresholds, file changes, or a hard timeout.
- Composition allows one abstraction to cover real-world orchestration cases.

## Detailed answer

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

## Practical design implications

- Less wasted waiting when a job finishes early.
- Fewer missed events when a run stalls or exceeds an expected duration.
- A single declarative sleep call can capture operational intent cleanly.

## Conclusion

Overall, Q2 highlights a deliberate architectural choice in Helios: the system favors explicit, durable, and operationally reliable mechanisms over brittle or purely implicit behavior.

## Architectural reasoning

Composable triggers let Helios express intent in operational terms. Instead of assuming time is the only meaningful signal, the scheduler can react to the actual state of a running experiment and resume exactly when it matters.

## Example scenario

A researcher might want the agent to resume when a run finishes, when validation loss drops below a threshold, or after two hours if neither happens. A single hard-coded sleep cannot represent that cleanly, but a composite trigger can.

## Trade-offs and limitations

- Composable logic is more powerful but requires a richer scheduler implementation.
- Different trigger kinds may need different polling intervals or event sources.
- The added flexibility is valuable for messy real-world ML workflows.

## Source files referenced

- `helios/src/scheduler/triggers/types.ts`
- `helios/src/scheduler/trigger-scheduler.ts`
- `helios/README.md`

