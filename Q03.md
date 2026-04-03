# Q3 — Metric parsing and observability without invasive instrumentation

## Expanded overview

Observability is essential for autonomous experimentation, but forcing users to integrate a custom SDK into every training loop would raise friction and increase coupling. Helios instead extracts metrics from ordinary stdout or log output.

## Why this matters

- Users can keep their normal training scripts and logging style.
- Metric capture becomes configuration-driven rather than code-invasive.
- The same mechanism works across frameworks and custom scripts.

## Detailed answer

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

## Practical design implications

- Lower adoption cost for new projects.
- Metrics remain available for dashboards, comparisons, and trend analysis.
- Changing tracked metrics often requires only config updates, not script rewrites.

## Conclusion

Overall, Q3 highlights a deliberate architectural choice in Helios: the system favors explicit, durable, and operationally reliable mechanisms over brittle or purely implicit behavior.

## Architectural reasoning

Helios favors observability by convention. If a training job already prints useful values, the platform can interpret those values without forcing the user to embed a custom agent API inside the training loop.

## Example scenario

If a script prints lines such as loss=0.82 acc=0.74 lr=1e-4, Helios can extract those numbers automatically and turn them into tracked metrics. The user does not need to rewrite the core training logic.

## Trade-offs and limitations

- Parsing logs depends on output consistency.
- It may be less explicit than direct instrumentation APIs.
- However, it dramatically lowers adoption friction and works across diverse training stacks.

## Source files referenced

- `helios/src/metrics/parser.ts`
- `helios/src/metrics/collector.ts`
- `helios/README.md`

