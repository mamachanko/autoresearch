---
name: autoresearch
description: Set up an autonomous experiment loop for any optimization goal. Use when the user wants to iteratively optimize something measurable — test speed, build time, model training loss, bundle size, latency, or any other metric. Creates all infrastructure files for running repeated experiment cycles.
argument-hint: <optimization goal and metric description>
---

# Autoresearch Setup

You are setting up an autonomous experiment loop. The user wants to iteratively optimize a measurable target. Your job is to create all the infrastructure so that each subsequent experiment cycle can run independently in a single agent request.

## User's Goal

$ARGUMENTS

## What You Must Do (all in this single request)

### Step 1: Understand the Codebase and Goal

- Read the user's goal above carefully.
- Explore the relevant parts of the codebase to understand what you're optimizing.
- Identify:
  - **The primary metric** to optimize (e.g., test runtime in seconds, build time, loss value, bundle size in KB).
  - **The optimization direction**: is lower better or higher better?
  - **The benchmark command**: what shell command measures the metric?
  - **Files in scope**: which files can be modified to improve the metric?
  - **Correctness constraints**: what must remain correct (tests pass, types check, etc.)?

### Step 2: Create `autoresearch.sh`

Create an executable bash script at `./autoresearch.sh` that:

1. Runs the benchmark/measurement.
2. Outputs the primary metric as: `METRIC <name>=<value>` (e.g., `METRIC test_runtime_s=12.5`).
3. Can optionally output secondary metrics the same way (e.g., `METRIC memory_mb=256`).
4. Exits 0 on success, non-zero on failure.
5. For fast benchmarks (under 5 seconds), runs multiple iterations and reports the median for stability.

The metric name should be descriptive and use underscores (e.g., `build_time_s`, `val_loss`, `bundle_size_kb`).

Make it executable: `chmod +x autoresearch.sh`.

Example:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Run benchmark 3 times, take median
times=()
for i in 1 2 3; do
  start=$(date +%s%N)
  npm test -- --silent 2>/dev/null
  end=$(date +%s%N)
  elapsed=$(( (end - start) / 1000000 ))
  times+=("$elapsed")
done

sorted=($(printf '%s\n' "${times[@]}" | sort -n))
median=${sorted[1]}
echo "METRIC test_runtime_ms=$median"
```

### Step 3: Create `autoresearch.checks.sh` (if correctness constraints exist)

Create an executable bash script at `./autoresearch.checks.sh` that validates correctness. This runs AFTER a successful benchmark only when the metric improved. It gates whether changes are kept.

Examples of checks:
- `npm test`
- `tsc --noEmit`
- `cargo test`
- `python -m pytest`

If no correctness constraints are relevant, skip this file.

Make it executable: `chmod +x autoresearch.checks.sh`.

### Step 4: Run Baseline Measurement

Run `./autoresearch.sh` to get the baseline metric value. This is the starting point.

If it fails, debug and fix the script until it works.

### Step 5: Create `autoresearch.jsonl`

Create the experiment log file with a config header line followed by the baseline result:

```
{"type":"config","metricName":"<name>","metricUnit":"<unit>","bestDirection":"<lower|higher>","goal":"<user's goal summary>","filesInScope":["<file1>","<file2>"]}
{"run":0,"commit":"baseline","metric":<baseline_value>,"status":"baseline","description":"Initial baseline measurement","timestamp":<unix_ms>}
```

### Step 6: Create `autoresearch.md`

Create a living research document at `./autoresearch.md`:

```markdown
# Autoresearch: <Goal Summary>

## Objective
<Specific description of what we're optimizing>

## Primary Metric
- **Name**: <metric_name>
- **Unit**: <unit>
- **Direction**: <lower|higher> is better
- **Baseline**: <baseline_value>

## Benchmark
`./autoresearch.sh`

## Correctness Checks
`./autoresearch.checks.sh` (or "None")

## Files in Scope
<List of files that can be modified>

## Constraints
<Any constraints the user mentioned>

## What's Been Tried
| Run | Metric | Status | Description |
|-----|--------|--------|-------------|
| 0   | <baseline> | baseline | Initial measurement |

## Key Wins
(None yet)

## Dead Ends
(None yet)

## Ideas Backlog
<List 3-5 initial optimization ideas based on your codebase analysis>
```

### Step 7: Git Commit

Stage and commit all autoresearch files:
```
git add autoresearch.sh autoresearch.checks.sh autoresearch.jsonl autoresearch.md
git commit -m "autoresearch: initialize experiment loop for <goal>"
```

Do NOT commit any other changes.

### Step 8: Report to User

Print a summary:
- The goal and metric
- The baseline value
- Files in scope
- Initial ideas from the backlog
- Instructions: "Run `/autoresearch-cycle` to execute the next experiment. Each invocation runs one full experiment cycle."

## Important Rules

- Do ALL of the above in this single request. Do not ask the user for clarification — make reasonable choices based on the codebase.
- If something is ambiguous, choose the most sensible default and document it in `autoresearch.md`.
- The benchmark script must be deterministic and fast enough to run repeatedly. Prefer wall-clock time measurements.
- Keep `autoresearch.sh` simple and reliable. Complex parsing is fragile.
- Every file you create must be self-contained and well-documented so a fresh agent can understand the setup from these files alone.
