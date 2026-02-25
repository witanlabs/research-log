# Deep Dive: Benchmark Framework Evolution

**Phase**: 9, 12, 14, 15 | **Date**: Nov 1, 2025 - Feb 25, 2026 | **Repo**: benchmark-framework

## Context

The benchmark framework evolved from a simple test harness to a sophisticated multi-runner, multi-strategy evaluation platform. Its 568 commits over ~4 months tell the story of how evaluation methodology matured alongside the agent.

## Timeline

### Nov 1-4: Genesis — Compare Agent
- First commit: basic test infrastructure
- Initial evaluation used a "compare agent" — an LLM that compared agent output workbooks to gold-standard answers
- Multiple runners from the start: witan agent, Anthropic API, "Claude skill" (openpyxl baseline)

**Critical early signal (Nov 4)**: *"Very poor initial performance compared to claude skill."* The simple Python openpyxl-based approach outperformed the more complex witan agent. This foreshadowed a recurring theme: **simplicity often wins**.

### Nov 7: Unified Runner Interface
- Abstract `AgentBenchmarkRunner` base class with `invokeAgent()` method
- All runners now share a common interface
- Task types renamed for clarity: `manipulation→edit`, `question→QnA`

### Nov 9-10: Unified Evaluator (attempted)
- Consolidated multiple evaluation agents into a single unified evaluator
- Added configuration for controlling re-evaluation behavior

### Nov 13: Production Agent Runner
- Added a runner connecting to the production agent's API
- First runner connecting to the "current" production agent

### Nov 17-19: Evaluation Strategy Specialization
**Key insight**: The unified evaluator was too coarse. Split into specialized evaluators:

1. **Content evaluator** — cell value comparison
2. **Structure evaluator** — row-level structural alignment
3. **Visual evaluator** — formatting comparison
4. **Text evaluator** — LLM-based grading

**Nov 18 decision**: *"Calculate results programmatically to prevent LLM inconsistencies."* LLM-based evaluation was unreliable for objective comparisons. This was a critical realization — the evaluator needs to be more reliable than the agent it evaluates.

### Nov 20-27: Infrastructure Hardening
- OpenAI streaming support
- XLS-to-XLSX conversion for input files
- Configurable timeouts with process group cleanup (not just process kill)
- Explicit run status: `complete`, `error`, `timeout`
- Task vs. evaluation failure distinction

### Dec 5: Benchmark Orchestrator
Major architectural refactoring introducing:
- `BenchmarkOrchestrator` class for declarative benchmark control
- `BenchmarkHandlers` for modular workflow steps
- Async queue for concurrent task processing
- Retry utility for transient failures

This separated **orchestration** (how to run benchmarks) from **execution** (what each runner does).

### Dec 29: Multi-Run Statistical Analysis
- Run benchmarks multiple times to collect statistical distributions
- Outcome distribution tracking (pass rates, runtime variance)
- Parallel multi-run execution with shared resource pool

This was essential for distinguishing **real improvements** from **noise**. A 75% vs. 80% pass rate on 20 tasks isn't statistically significant; running 5x and computing confidence intervals is.

### Jan 16, 2026: Gemini Runner
Added Google Gemini API support, expanding the model comparison surface.

### Feb 17: CLI vs. openpyxl Critical Comparison
The most impactful evaluation result of the project. See [07-cli-vs-openpyxl.md](./07-cli-vs-openpyxl.md) for details.

### Feb 24: Composable Skills System
Replaced dedicated witan runners with composable `--skills` flag.

## Five Evaluation Strategies (Final Architecture)

### Content Evaluation
- **What it checks**: Are the right values present?
- **Method**: Extract all cell values from both workbooks, compare as sets
- **Metrics**: Jaccard similarity, precision, recall, F1
- **Default threshold**: Jaccard ≥ 70%
- **Position-agnostic**: Values can be in different cells

### Structure Evaluation
- **What it checks**: Is the layout correct?
- **Method**: Row-level sequence alignment using dynamic programming
- **Handles**: Inserted/deleted rows, reordered sections
- **Multi-sheet**: Weighted by cell count
- **Default threshold**: ≥ 70% weighted score

### Visual Evaluation
- **What it checks**: Does it look right?
- **Method**: Font, color, number format, and dimension comparison using ExcelJS
- **Strictest**: Default threshold ≥ 90%
- **Why strict**: Formatting errors are visually obvious and professionally unacceptable

### Scenarios Evaluation
- **What it checks**: Do formulas work correctly?
- **Method**: Set input values, recalculate, compare outputs
- **Label-based**: Find cells by label, not by address (robust to layout changes)
- **Ephemeral**: No disk modifications
- **Binary per scenario**: Pass or fail

### Text Evaluation
- **What it checks**: Is the answer correct?
- **Method**: LLM-based grading against `grading_criteria`
- **Used for**: QnA tasks where the output is text, not a workbook
- **Returns**: 0.0 (fail) or 1.0 (pass)

### Orchestrator
Combines strategies with configurable weights:
```typescript
evaluation: {
  pass_threshold: 0.8,
  evaluations: [
    { type: 'content', weight: 0.5 },
    { type: 'scenarios', weight: 0.5, config: { scenarios: [...] } }
  ]
}
```
Final score = weighted average. Pass/fail by threshold comparison.

## Runner Architecture (Final)

```
AgentBenchmarkRunner (abstract)
├── ClaudeStyleXLSXRunner (skill-driven local execution)
│   ├── ClaudeCodeRunner (Claude Code SDK)
│   └── DeepAgentsRunner (DeepAgents framework)
├── AnthropicBenchmarkRunner (Anthropic API)
├── OpenAIBenchmarkRunner (OpenAI API)
├── OpenAICLIBenchmarkRunner (OpenAI Codex sandbox + witan CLI)
├── ProductionAgentRunner (production agent API)
└── GeminiRunner (Google Gemini API)
```

## Analysis & Debugging Infrastructure

The framework accumulated sophisticated analysis tooling:
- **Log analysis prompts** (in `.prompts/`): Templates for investigating failures, comparing runs, extracting roadmap items from logs
- **Task log analyzer**: `analyze-task-log` CLI command for summarizing agent behavior
- **Error scanner**: `find-log-errors` for triaging noisy runs
- **Trace viewer**: Visualization tools for LangSmith traces
- **Multi-run summaries**: Aggregated statistics across benchmark runs

## Key Learnings

1. **Evaluation methodology evolves with agent capabilities.** As the agent improved, the evaluation strategies needed to become more nuanced.
2. **Programmatic evaluation > LLM evaluation for objective comparisons.** Use LLMs only for genuinely subjective assessments.
3. **Statistical rigor matters.** Single-run comparisons are noise; multi-run analysis reveals signal.
4. **Composability beats monolithic runners.** The skills system enabled systematic experimentation.
5. **The benchmark framework is as complex as the agent itself.** 568 commits, 29K lines of TypeScript — evaluation is a first-class engineering challenge.

## Known Issues (as of Feb 25, 2026)

- Memory leaks in DeepAgents runner (OOM at concurrency 10+)
- 455% CPU spike at batch startup (deepagents-specific)
- Per-task redundancies: dataset scanning, TypeScript parsing, API validation repeated per task
