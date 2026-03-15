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

Copy the skill into your project's `.claude/skills/` directory:

```bash
# From your project root
mkdir -p .claude/skills/autoresearch
cp autoresearch-skill/.claude/skills/autoresearch/SKILL.md .claude/skills/autoresearch/SKILL.md
```

Or copy the entire `.claude/skills/autoresearch/` directory into your project.

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

## Real-World Results

Used in the [Sentinel Immortal](https://github.com/igorvagun) project (S110), 10 autoresearch runs optimized 7 metrics:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| soul.md tokens | 4,531 | 2,344 | -48% |
| Search latency | 2,635ms | 130ms | -95% (20x) |
| Bot startup | 19.6s | 4.0s | -80% (5x) |
| Plugin import | 104ms | 40ms | -62% |
| Sanitizer false positives | 2 | 0 | -100% |
| BM25 relevance (25 queries) | 17 | 21 | +24% |
| KG edges per fact | 0.64 | 0.89 | +39% |

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
