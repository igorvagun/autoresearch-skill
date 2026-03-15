# Autoresearch Skill for Claude Code

An autonomous self-improving optimization loop for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Iterates a single measurable metric by making atomic changes, verifying mechanically, keeping improvements, and reverting failures.

Adapted from [Karpathy's autoresearch pattern](https://x.com/kaboromonkey/status/1896235023092035953). No git required — uses file backups for safe rollback.

## What It Does

Given a **goal**, a **scope** (files to modify), a **metric** (number to optimize), and a **verify command** (shell command that outputs the number), this skill runs an autonomous loop:

1. **Review** current state and past attempts
2. **Ideate** the next atomic change
3. **Backup** files before touching them
4. **Modify** one thing
5. **Verify** by running the metric command
6. **Keep or revert** based on whether the metric improved
7. **Log** results and repeat

It stops when: bounded iteration limit hit, metric reaches theoretical max, or 10 consecutive failed attempts (plateau).

## Installation

```bash
# From your project root
mkdir -p .claude/skills/autoresearch
curl -o .claude/skills/autoresearch/SKILL.md \
  https://raw.githubusercontent.com/igorvagun/autoresearch-skill/main/.claude/skills/autoresearch/SKILL.md
```

Or clone and copy:

```bash
git clone https://github.com/igorvagun/autoresearch-skill.git
cp -r autoresearch-skill/.claude/skills/autoresearch .claude/skills/
```

## Usage

In Claude Code, invoke with:

```
/autoresearch
Goal: Increase test coverage for the auth module
Scope: src/auth/*.py, tests/test_auth*.py
Metric: test coverage % (higher is better)
Verify: python -m pytest tests/test_auth*.py --cov=src/auth --cov-report=term 2>&1 | grep TOTAL | awk '{print $4}' | tr -d '%'
```

### Bounded mode

Limit to N iterations:

```
/autoresearch loop 10
Goal: Reduce bundle size
Scope: src/**/*.ts
Metric: bundle size KB (lower is better)
Verify: npm run build 2>&1 | grep 'Total size' | awk '{print $3}'
```

## Examples

### Optimize search latency

```
/autoresearch loop 15
Goal: Reduce search query latency
Scope: src/search/engine.py, src/search/indexer.py
Metric: avg query time ms (lower is better)
Verify: python benchmark_search.py 2>&1 | tail -1
```

### Increase test pass rate

```
/autoresearch
Goal: Fix failing tests
Scope: src/**/*.py, tests/**/*.py
Metric: test pass count (higher is better)
Verify: python -m pytest tests/ 2>&1 | grep 'passed' | grep -oP '\d+(?= passed)'
```

### Reduce memory usage

```
/autoresearch loop 20
Goal: Reduce peak memory usage
Scope: src/data_pipeline.py
Metric: peak RSS MB (lower is better)
Verify: python -c "import tracemalloc; tracemalloc.start(); exec(open('src/data_pipeline.py').read()); print(tracemalloc.get_traced_memory()[1] // 1024 // 1024)"
```

### Compress prompt/config file

```
/autoresearch loop 15
Goal: Reduce token count while preserving all rules
Scope: prompts/system.md
Metric: token count (lower is better)
Verify: python -c "from transformers import AutoTokenizer; t=AutoTokenizer.from_pretrained('gpt2'); print(len(t.encode(open('prompts/system.md').read())))"
```

## Real-World Results

88 iterations across 9 targets in ~3 hours. 7 produced measurable improvements:

| # | Metric | Before | After | Improvement | Iterations |
|---|--------|--------|-------|-------------|------------|
| 1 | Test coverage | 35% | 93% | +58pp | 20/20 keeps |
| 2 | Prompt tokens | 4,531 | 2,344 | -48% | 15/15 keeps |
| 3 | Bot startup | 19.6s | 4.0s | 5x faster | 3/10 keeps |
| 4 | Search latency | 2,635ms | 130ms | 20x faster | 7/15 keeps |
| 5 | Sanitizer FPs | 2 | 0 | -100% | 2/2 keeps |
| 6 | BM25 relevance | 17/25 | 21/25 | +24% | 4/15 keeps |
| 7 | KG extraction | 0.64 | 0.89 | +39% | 1/1 keep |

### Sample Iteration Logs

**Prompt token compression (15/15 keeps):**

| Iter | Tokens | Delta | Change |
|------|--------|-------|--------|
| 0 | 4,531 | — | Baseline |
| 1 | 4,428 | -103 | Merge Personality into Tone section |
| 2 | 4,200 | -228 | Condense 5 self-description subsections |
| 3 | 3,779 | -421 | Compress 20 tool descriptions |
| 4 | 3,527 | -252 | Tighten rules 9-16, remove examples |
| 5 | 3,364 | -163 | Tighten rules 1-8 |
| ... | ... | ... | ... |
| 14 | 2,459 | -161 | Compress memory rules: 11 → 5 bullets |
| 15 | 2,344 | -115 | Final pass on identity sections |

Every iteration found something to compress — the skill systematically worked through each section.

**Search latency (7/15 keeps, 20x speedup):**

| Iter | Latency | Delta | Status | Change |
|------|---------|-------|--------|--------|
| 0 | 2,635ms | — | baseline | Hybrid BM25 + cosine + BGE reranker |
| 2 | 2,538ms | -94ms | keep | Reduce rerank pool 30→15 |
| 5 | 2,484ms | -15ms | keep | BM25 candidates 120→60 |
| **7** | **157ms** | **-2,311ms** | **keep** | **Eager reranker init (cold-start fix)** |
| 11 | 130ms | -17ms | keep | Confidence pool 50→20 |
| 12-14 | — | — | discard | Diminishing returns |

Iteration 7 found a 2.3-second cold-start penalty in lazy reranker initialization — a bottleneck hiding in plain sight for months.

## When NOT to Use

- **Subjective quality** — "make the UI look better" has no number
- **Multi-dimensional** — optimizing 4 metrics simultaneously needs a different approach
- **Slow verify commands** — if measurement takes >5 minutes, expect agent stalls
- **Non-deterministic metrics** — if the number varies ±20% between runs, signal drowns in noise
- **Architecture changes** — autoresearch makes *atomic* changes, not system redesigns

## Lessons Learned

From running 88 iterations across 9 targets:

1. **Verify command speed matters** — under 60s works well. Over 5 min is risky. One target (reinforcement scoring, 30 min/iteration) stalled the agent.
2. **Perfect keep rate is possible** — the prompt compression achieved 15/15 keeps because every section had fat to trim. When the optimization surface is rich, the skill rarely misses.
3. **Breakthroughs are unpredictable** — the biggest win (2,311ms latency drop) came from a single line change on iteration 7. The skill found it by systematically exhausting smaller optimizations first.
4. **Plateau detection works** — after 3-5 consecutive discards, the skill correctly identifies diminishing returns and moves on.
5. **Simpler IS better** — several "keep" iterations removed code complexity while maintaining the same metric. The skill correctly prefers simplicity.

## How It Works

The core insight: **most optimization is iterative and mechanical**. You try something, measure, keep or revert. Claude Code can do this autonomously if you give it:

- A clear metric (single number)
- A mechanical way to measure (shell command)
- A bounded scope (which files to touch)
- A safe rollback mechanism (file backups)

The skill enforces discipline:
- **One atomic change per iteration** — no multi-file refactors
- **Always backup before modify** — the backup IS the undo
- **Always restore on failure** — no partial changes survive
- **Never modify the verify command** — the metric is immutable
- **Simpler is better** — removing complexity for equal metrics is a win

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- A project with a measurable metric
- A shell command that outputs a number

## License

MIT
