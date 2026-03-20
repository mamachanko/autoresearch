---
name: autoresearch-cycle
description: Run one autonomous experiment cycle. Reads the current autoresearch state, proposes a code change, benchmarks it, and keeps or discards the result. Use this after /autoresearch has set up the experiment infrastructure.
---

# Autoresearch Cycle

You are running ONE experiment cycle in an autonomous optimization loop. Each cycle is precious — make it count.

## What You Must Do (all in this single request)

### Step 1: Load State

Read these files to understand the current state:

1. **`autoresearch.md`** — Understand the objective, metric, files in scope, what's been tried, dead ends, and wins.
2. **`autoresearch.jsonl`** — Parse all lines. The first line is the config. Subsequent lines are experiment results. Note:
   - What the current best metric value is.
   - Which direction is better (lower/higher).
   - What experiments have been tried and their outcomes.
   - What was recently kept vs. discarded.
3. **The files in scope** — Read the actual source code you'll be modifying.
4. **Git log** — Run `git log --oneline -20` to see the recent history of kept changes.

### Step 2: Hypothesize

Based on everything you've learned:

- What hasn't been tried yet?
- What worked before that could be extended?
- What patterns from discarded experiments suggest a different approach?
- Consult the "Ideas Backlog" in `autoresearch.md`.

Pick ONE focused change to try. Write down your hypothesis before coding.

**Good experiments are:**
- Focused on a single change (easy to attribute results).
- Informed by past successes and failures.
- Reversible (git will handle this).

### Step 3: Implement

Make the code change. Modify only files listed in "Files in Scope" in `autoresearch.md`.

Keep changes small and focused. One idea per cycle.

### Step 4: Commit

```bash
git add <modified files>
git commit -m "autoresearch: <short description of what you changed>"
```

Record the short commit hash: `git rev-parse --short HEAD`

### Step 5: Benchmark

Run the benchmark:

```bash
./autoresearch.sh
```

Parse the output for `METRIC <name>=<value>` lines. Extract the primary metric value.

If the script crashes (non-zero exit), this is a "crash" result — skip to Step 7.

### Step 6: Evaluate and Decide

Compare the new metric to the current best:

- **Read the config line** from `autoresearch.jsonl` to know `bestDirection`.
- **Find the current best**: scan all "keep" and "baseline" entries for the best metric value.

**Decision rules:**

- **If improved** (metric is better than current best):
  - If `autoresearch.checks.sh` exists, run it.
    - If checks pass → **KEEP**.
    - If checks fail → **DISCARD** (status: `checks_failed`).
  - If no checks file → **KEEP**.
- **If same or worse** → **DISCARD**.

### Step 7: Log Result

Determine the next run number (max run number from jsonl + 1).

Append one JSON line to `autoresearch.jsonl`:

```json
{"run":<N>,"commit":"<7-char-hash>","metric":<value>,"status":"<keep|discard|crash|checks_failed>","description":"<brief description>","timestamp":<unix_ms>}
```

### Step 8: Keep or Revert

- **If KEEP**: Leave the code as-is. The commit stays.
- **If DISCARD/CRASH/CHECKS_FAILED**: Revert the code change but preserve autoresearch files:
  ```bash
  git stash push -- autoresearch.jsonl autoresearch.md
  git reset --hard HEAD~1
  git stash pop
  ```

### Step 9: Update `autoresearch.md`

Update the "What's Been Tried" table with this experiment's results.

If the experiment was a KEEP:
- Add it to "Key Wins".
- Update the baseline metric in the header.

If the experiment revealed a dead end:
- Add it to "Dead Ends".

Add any new ideas that emerged to "Ideas Backlog". Remove the idea you just tried.

Commit the updated autoresearch files:
```bash
git add autoresearch.jsonl autoresearch.md
git commit -m "autoresearch: log run <N> — <keep|discard>: <description>"
```

### Step 10: Report

Print a concise summary:

```
## Autoresearch Cycle <N>

**Hypothesis**: <what you tried>
**Result**: <metric value> (<status>)
**Best so far**: <best metric value> (run <N>)
**Cycles completed**: <total runs>
**Kept improvements**: <count of keeps>

Run `/autoresearch-cycle` for the next experiment.
```

## Important Rules

- Do EVERYTHING above in this single request. Do not stop early or ask for input.
- Make exactly ONE code change per cycle. Keep it focused.
- ALWAYS revert on discard. Never leave failed experiments in the codebase.
- ALWAYS log to jsonl, even on crash. The history is critical for future cycles.
- ALWAYS update autoresearch.md. A fresh agent must be able to resume from these files.
- Think deeply before coding. Study what worked and what didn't. The best experiments come from understanding, not guessing.
- If you've exhausted obvious ideas, try creative approaches: algorithmic changes, caching, parallelism, different data structures, reducing allocations, etc.
- If the last 3+ experiments were all discards, step back and reconsider your strategy. Document this in autoresearch.md.
