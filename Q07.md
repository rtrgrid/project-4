# Q7 — Composable triggers vs a pure event-driven design

## Expanded overview

A pure event-driven model is not sufficient for all wake conditions in ML workflows. Some conditions naturally arise from events, but others require polling, aggregation, or threshold evaluation. Helios therefore uses a composable trigger abstraction above the runtime mechanism.

## Why this matters

- Not every meaningful condition maps to a single operating-system event.
- Logical composition such as OR and AND must work across heterogeneous trigger types.
- The user-facing API should describe intent rather than implementation detail.

## Detailed answer

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

## Practical design implications

- Helios can mix event-driven and polling-based wake checks behind one interface.
- The system can short-circuit composite expressions efficiently.
- The abstraction remains flexible as new trigger types are added.

## Conclusion

Overall, Q7 highlights a deliberate architectural choice in Helios: the system favors explicit, durable, and operationally reliable mechanisms over brittle or purely implicit behavior.

## Architectural reasoning

Helios treats trigger composition as the user-facing abstraction and leaves implementation details to the runtime. Some triggers are naturally event-driven, others need polling, and the abstraction must cover both without exposing that complexity everywhere.

## Example scenario

A user message may wake the agent immediately as an event, while a metric threshold may require repeated checks over time. Helios hides that difference behind a common trigger expression model.

## Trade-offs and limitations

- The runtime is more complex because it must support heterogeneous evaluation strategies.
- There may be tuning considerations for poll intervals and trigger latency.
- Still, the design is more realistic for ML infrastructure than a purely event-only model.

## Source files referenced

- `helios/src/scheduler/triggers/types.ts`
- `helios/src/scheduler/trigger-scheduler.ts`
- `helios/src/scheduler/sleep-manager.ts`

