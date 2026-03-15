---
name: autoresearch
description: Autonomous self-improving optimization loop. Iterates a single measurable metric by modifying scoped files, verifying mechanically, keeping improvements and reverting failures. Adapted from Karpathy's autoresearch pattern. No git required — uses file backups.
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent
argument-hint: "Goal: <goal> Scope: <file globs> Metric: <what to measure, direction> Verify: <shell command that outputs a number>"
---

# Autoresearch — Autonomous Optimization Loop

Iterates ONE measurable metric by making atomic changes, verifying mechanically, and keeping only improvements. Based on Karpathy's autoresearch pattern, adapted for any project — git optional.

## When to Use

Any task where you can express success as a **single number** extracted from a **shell command**:
- Test coverage %, latency ms, file size KB, score /10, pass count, error count
- NOT subjective quality ("looks better"), NOT multi-dimensional

## Invocation

```
/autoresearch
Goal: Reduce API response latency
Scope: src/api/*.py
Metric: p95 latency ms (lower is better)
Verify: curl -s http://127.0.0.1:8000/benchmark | jq '.p95_ms'
```

Bounded mode (max N iterations):
```
/autoresearch loop 10
Goal: ...
```

## Instructions

### Phase 0: PARSE & VALIDATE

Extract from `$ARGUMENTS`:
- **Goal**: What we're optimizing (plain language)
- **Scope**: File glob patterns that are editable (e.g., `src/**/*.py`)
- **Metric**: What number + direction (higher/lower is better)
- **Verify**: Shell command that outputs a parseable number
- **Loop count**: From `loop N` if specified, otherwise unlimited

**Validation gates — STOP if any fail:**

1. Scope resolves to at least 1 existing file:
```bash
ls $SCOPE 2>/dev/null | head -5
```

2. Verify command runs and outputs a parseable number:
```bash
$VERIFY_COMMAND
```
Extract the number. If not parseable, ask user to fix the verify command.

3. Record **baseline metric** value.

4. Create results log in the project root:
```bash
echo -e "iteration\tmetric\tdelta\tstatus\tdescription" > autoresearch-results.tsv
echo -e "0\t$BASELINE\t0\tbaseline\tInitial measurement" >> autoresearch-results.tsv
```

Print summary and confirm:
```
Autoresearch Configuration:
  Goal:     [goal]
  Scope:    [N files matched]
  Metric:   [name] ([direction])
  Baseline: [value]
  Loop:     [N iterations | unlimited]

Starting autonomous loop...
```

### Phase 1: REVIEW (each iteration)

Read the current state:
```bash
# Read results history
cat autoresearch-results.tsv

# Read all scoped files (use Read tool on each)
```

Analyze patterns:
- What changes were kept? (status=keep)
- What changes were discarded? (status=discard)
- What hasn't been tried yet?
- Are we plateauing? (last 3 iterations all discard)

### Phase 2: IDEATE

Based on review, select the next change. Rules:
- **ONE atomic change** per iteration, explainable in one sentence
- **Never modify the verify command** or results file
- **Balance exploration vs exploitation**: try new approaches, but lean toward what's worked
- If last 3 iterations were all discards, try a fundamentally different approach
- If bounded mode with <3 iterations left, exploit only (refine what worked)
- **Simplicity preference**: equal results with simpler code is a WIN

### Phase 3: BACKUP

Before modifying any file, create a backup:
```bash
# For each file being modified
cp "$FILE" "$FILE.autoresearch.bak"
```

### Phase 4: MODIFY

Make the ONE atomic change using Edit tool. Keep it focused.

### Phase 5: VERIFY

Run the mechanical metric:
```bash
$VERIFY_COMMAND
```

Parse the output number. If command crashes:
- Check if the crash is from our change (syntax error, import error)
- If fixable in <30 seconds, fix and re-verify
- If not, treat as crash and restore

### Phase 6: DECIDE

Compare new metric to previous best:

**If IMPROVED** (metric moved in desired direction):
- Status: `keep`
- Delete backup files: `rm "$FILE.autoresearch.bak"`
- Record new best value

**If EQUAL or WORSE**:
- Status: `discard`
- Restore from backup: `cp "$FILE.autoresearch.bak" "$FILE"`
- Delete backup: `rm "$FILE.autoresearch.bak"`

**If CRASHED**:
- Status: `crash`
- Restore from backup: `cp "$FILE.autoresearch.bak" "$FILE"`
- Delete backup: `rm "$FILE.autoresearch.bak"`

### Phase 7: LOG

Append result to TSV:
```bash
echo -e "$ITERATION\t$METRIC_VALUE\t$DELTA\t$STATUS\t$DESCRIPTION" >> autoresearch-results.tsv
```

Print iteration summary:
```
[Iteration N] STATUS | Metric: X.XX (delta: +/-Y.YY) | "one-sentence description"
```

Every 5 iterations, print progress report:
```
=== Progress Report (iteration N) ===
Baseline: X.XX -> Current Best: Y.YY (improvement: +Z.ZZ%)
Keeps: K | Discards: D | Crashes: C
Last 5: [keep, discard, keep, discard, keep]
```

### Phase 8: REPEAT

Go back to Phase 1. **Never stop. Never ask "should I continue?"**

Only stop when:
- Bounded mode: iteration count reached
- Metric hits theoretical maximum (e.g., 100% coverage)
- 10 consecutive discards (plateau detected — report and stop)

### Final Summary

When loop ends (bounded limit, plateau, or maximum reached):
```
=== Autoresearch Complete (N iterations) ===
Baseline: X.XX -> Final: Y.YY (improvement: +Z.ZZ%)
Keeps: K | Discards: D | Crashes: C
Best change: Iteration #M — "description"
Worst attempt: Iteration #W — "description"

Results saved: autoresearch-results.tsv
```

## Critical Rules

1. **NEVER modify the verify command** — the metric is immutable
2. **NEVER modify the results file** (except appending new rows)
3. **ONE change per iteration** — atomic, reversible, explainable
4. **ALWAYS backup before modify** — the backup IS the rollback mechanism
5. **ALWAYS restore on failure** — no partial changes survive a discard
6. **NEVER ask for permission** to continue the loop — just iterate
7. **Simpler is better** — removing complexity for equal results is a keep

## Adaptation Notes

This skill works in any project. Adjust for your environment:
- **Python path**: Use your venv's python (e.g., `.venv/bin/python`, `.venv/Scripts/python.exe`)
- **HTTP calls**: Use `127.0.0.1` instead of `localhost` on Windows (IPv6 resolution issues)
- **Git projects**: You can use git stash/restore instead of file backups if preferred
- **Results log**: `autoresearch-results.tsv` is created in the working directory, one per run
