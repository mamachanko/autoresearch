# autoresearch

Autonomous experiment loop skill for AI coding agents. Inspired by [karpathy/autoresearch](https://github.com/karpathy/autoresearch) and [pi-autoresearch](https://github.com/davebcn87/pi-autoresearch).

Set up an optimization goal, then run experiment cycles. Each cycle proposes a change, benchmarks it, and keeps or discards the result automatically. Works with any measurable target: test speed, build time, model training loss, bundle size, latency, etc.

## Install

```bash
npx skills add mamachanko/autoresearch
```

Or manually copy the `skills/` directory into your project's `.claude/skills/` or `~/.claude/skills/`.

## Usage

### 1. Set up the experiment

```
/autoresearch optimize unit test runtime, monitor correctness
```

This creates the infrastructure files in your project:

| File | Purpose |
|------|---------|
| `autoresearch.sh` | Benchmark script — runs measurement, outputs `METRIC name=value` |
| `autoresearch.checks.sh` | Correctness gates — tests, types, linting (optional) |
| `autoresearch.jsonl` | Append-only experiment log (machine-readable) |
| `autoresearch.md` | Living research document (human-readable) |

**Cost: 1 agent request.**

### 2. Run experiment cycles

```
/autoresearch-cycle
```

Each invocation runs one full cycle:

1. Reads current state from `autoresearch.md` and `autoresearch.jsonl`
2. Proposes a hypothesis based on past results
3. Implements the change
4. Commits and benchmarks
5. Keeps the change if the metric improved (and checks pass), otherwise reverts
6. Logs the result

**Cost: 1 agent request per cycle.** Run as many as your quota allows.

### 3. Check progress

```
/autoresearch-status
```

Shows metric history, best result, confidence score, and remaining ideas.

## Examples

```
/autoresearch optimize unit test runtime, monitor correctness
/autoresearch model training, run 5 minutes of train.py and note the loss ratio as optimization target
/autoresearch improve build speed across the project
/autoresearch reduce bundle size of the frontend app
/autoresearch minimize API response latency for the /search endpoint
```

## Design Philosophy

**Request-efficient.** Each agent invocation does one complete unit of work. No back-and-forth needed.

**Stateful across sessions.** The `autoresearch.jsonl` and `autoresearch.md` files contain all state. A fresh agent can resume from where the last one left off.

**Git-integrated.** Every kept experiment is a git commit. Discarded experiments are reverted. The git log tells the story of what worked.

**Domain-agnostic.** The benchmark script (`autoresearch.sh`) is the only domain-specific piece. Everything else is generic infrastructure.

## Files

| File | Modified by agent? | Purpose |
|------|--------------------|---------|
| `autoresearch.sh` | Setup only | Runs benchmark, outputs `METRIC name=value` |
| `autoresearch.checks.sh` | Setup only | Correctness validation (tests, types) |
| `autoresearch.jsonl` | Every cycle | Append-only experiment log |
| `autoresearch.md` | Every cycle | Research context, wins, dead ends, ideas |

## How It Works

```
┌─────────────────────────────────────────┐
│          /autoresearch <goal>           │
│                                         │
│  Analyze codebase → Create scripts →    │
│  Run baseline → Write config → Commit   │
│                                         │
│  Cost: 1 request                        │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│        /autoresearch-cycle              │
│                                         │
│  Load state → Hypothesize → Implement → │
│  Commit → Benchmark → Evaluate →        │
│  Keep or Revert → Log → Report          │
│                                         │
│  Cost: 1 request per cycle              │
│  Run N times                            │
└─────────────────────────────────────────┘
```

## License

MIT
