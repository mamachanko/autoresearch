---
name: autoresearch-status
description: Show the current status of an autoresearch experiment loop — metric history, best result, recent experiments, and confidence score. Use when the user asks about autoresearch progress.
---

# Autoresearch Status

Show the user the current state of their autoresearch experiment loop.

## What You Must Do

### Step 1: Read State

1. Read `autoresearch.jsonl` and parse all lines.
2. Read `autoresearch.md` for context.

If these files don't exist, tell the user to run `/autoresearch <goal>` first.

### Step 2: Compute Statistics

From the jsonl data:

- **Config**: metric name, unit, direction.
- **Total runs**: count of non-config lines (excluding baseline).
- **Baseline**: the run 0 metric value.
- **Current best**: best metric value among "keep" and "baseline" entries.
- **Improvement**: percentage change from baseline to best.
- **Keeps**: count of "keep" status entries.
- **Discards**: count of "discard" status entries.
- **Crashes**: count of "crash" status entries.
- **Check failures**: count of "checks_failed" status entries.
- **Last 5 experiments**: recent run summaries.

### Step 3: Compute Confidence Score

If there are 3+ metric values (from keeps and baseline):

1. Collect all metric values from "keep" and "baseline" entries.
2. Compute the Median Absolute Deviation (MAD): median of `|value - median(values)|`.
3. Compute the best improvement: `|best_metric - baseline_metric|`.
4. Confidence = `best_improvement / MAD`.

Interpret:
- **>= 2.0x**: High confidence — improvement is likely real (not noise).
- **1.0-2.0x**: Moderate — above noise but marginal.
- **< 1.0x**: Low — improvement may be within noise floor.

### Step 4: Display

```
# Autoresearch Status

**Goal**: <from config>
**Metric**: <name> (<unit>, <direction> is better)

## Progress
- **Baseline**: <value>
- **Current best**: <value> (run <N>)
- **Improvement**: <percentage>%
- **Confidence**: <score>x (<high|moderate|low>)

## Experiment Summary
- Total cycles: <N>
- Kept: <N> | Discarded: <N> | Crashed: <N> | Check failures: <N>

## Recent Experiments
| Run | Metric | Status | Description |
|-----|--------|--------|-------------|
| ... | ...    | ...    | ...         |

## Key Wins
<from autoresearch.md>

## Ideas Backlog
<from autoresearch.md>

Run `/autoresearch-cycle` for the next experiment.
```
