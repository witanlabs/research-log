# Deep Dive: CLI vs. openpyxl — The Critical Comparison

**Phase**: 15 | **Date**: Feb 17, 2026 | **Repo**: benchmark-framework

## Context

By February 2026, the project had two primary tool approaches for LLM-spreadsheet interaction:

1. **openpyxl** (`claude-xlsx` skill): The agent uses Python with openpyxl library to directly read/write Excel files. In-process file access, instant response times, but limited Excel fidelity (notably: no formula recalculation).

2. **Witan CLI standalone commands**: The agent calls separate `witan` CLI commands — `xlsx find`, `xlsx calc`, `xlsx render`, `xlsx lint` — each as an individual shell command communicating with a local Witan API server (backed by a .NET Excel engine). Higher fidelity and verification capabilities, but each operation involves a CLI process spawn and a local HTTP call to the API server.

**Important**: the CLI side used the **verify-style workflow** (separate commands per operation), not the **exec/REPL approach** (JavaScript code execution via `witan xlsx exec`). The exec approach — which became the core of the production agent — was not the subject of this comparison.

Both approaches ran against a local API server — there was no network latency involved. The question: which approach produces better results?

## The Test

- **Runner**: `claude-code` (Claude Code SDK) for both
- **Model**: claude-opus-4-6 for both
- **Dataset**: 20 QnA tasks from the `qna` benchmark dataset
- **Difference**: Only the skill — `claude-xlsx` (Python openpyxl) vs. `witan-verify` (separate CLI commands: find, calc, render, lint)

## Results

| Metric | openpyxl | Witan CLI |
|--------|----------|-----------|
| **Pass rate** | **17/20 (85%)** | **14/20 (70%)** |
| CLI-only failures | — | 4 tasks |
| Shared failures | 3 tasks | 3 tasks |
| Avg tool calls | 21 spans | 42 spans |

**The simpler approach won by 15 percentage points.**

## Root Cause Analysis

### 1. Server-Side Processing Slowness (caused 3 timeouts)
Each witan CLI operation took 50-130 seconds — despite running against a local API server with no network latency. The root cause was a degenerate performance issue in the .NET backend: full recalculation mode was repeatedly querying the dependency index for individual cells instead of doing a single batch query covering all cells. For tasks requiring 10+ operations, total time exceeded the 600-second timeout.

This was later fixed by restructuring the recalculation to perform a single dependency index query covering all affected cells, eliminating the per-cell overhead.

**openpyxl equivalent**: File operations complete in milliseconds. openpyxl doesn't recalculate formulas at all (it reads cached values), so it never hits this codepath.

### 2. Shell Escaping Bug (compounded 2 timeouts)
Sheet names with spaces (e.g., "SW Selections") triggered a three-layer interaction:

```
Layer 1: Claude Code SDK's shell-quote wraps in double quotes
         → escapes ! to \!
Layer 2: zsh on macOS treats \! as literal two-char sequence
         → passes \! to the CLI
Layer 3: .NET AddressParser fails on \!
         → ADDRESS_MISSING_SHEET error
```

The SKILL.md documented the wrong quoting pattern, compounding the issue.

**openpyxl equivalent**: No shell quoting needed — sheet names are Python strings.

### 3. Model Hallucination (1 failure)
Task `rec3o77Tl55cx9s5M` (FCA steel shipments): Claude fabricated a "Fleet Readiness Assessment" about ships. Made zero tool calls, never opened the spreadsheet. The word "shipments" triggered an association with "ships."

**This affected both approaches** — it's a model-level issue, not a tool issue. But it was caught in the CLI run.

### 4. 2x Tool Call Overhead
The CLI agent averaged 42 tool calls vs. 21 for openpyxl:

| Operation | openpyxl | Witan CLI |
|-----------|----------|-----------|
| Open workbook | Python import (instant) | CLI process spawn + local API call |
| Read cell | Direct memory access | CLI → local HTTP → .NET → response |
| Search | In-process | CLI → local HTTP → .NET → response |
| Write | Direct | CLI → local HTTP → .NET → response |

Every operation requires a separate CLI process invocation and local API call. While there's no network latency, the per-operation overhead of process spawning and request handling adds up — and each slow operation (see #1) compounds the effect.

### 5. Intermittent API Failures
The witan API intermittently returned empty results from `find` and `calc` commands — even on valid files. Success rates as low as 11-40% on some tasks. What looked like agent confusion was actually the agent legitimately retrying failed commands.

### 6. SKILL.md Documentation Errors
The `witan-exec` SKILL.md instructed:
```bash
# DOCUMENTED (BROKEN):
./witan xlsx exec file.xlsx --expr 'xlsx.readRangeTsv(wb, "'Sheet Name'!A1:J50")'

# CORRECT:
./witan xlsx exec file.xlsx --expr 'xlsx.readRangeTsv(wb, "Sheet Name!A1:J50")'
```
The inner single quotes break bash quoting. The API auto-quotes sheet names.

### 7. Render Verification Loops
The CLI agent over-used PNG rendering for data extraction:
```
render → download PNG → read PNG → extract data → verify
```
Each cycle: 3-4 tool calls + multimodal token cost. The agent should have used `find -C` or `calc -r` instead.

### 8. No .xls Auto-Conversion
The runner copied `.xls` files verbatim, but the API expects `.xlsx`. An `ensureXlsxFormat()` utility existed but was never called.

## The Fundamental Tension

This comparison crystallized a tension that ran throughout the project:

**Sophistication** (Witan CLI verify workflow):
- Higher Excel fidelity (.NET engine > openpyxl)
- Formula recalculation engine
- Semantic linting and visual rendering
- But: separate shell command per operation, each with process spawn overhead
- And: the verify-style commands (find, calc, render) were not designed for rapid iterative exploration — they're verification gates

**Simplicity** (openpyxl):
- Instant in-process file access
- Zero infrastructure (no API server)
- Zero quoting issues (Python strings)
- Fewer failure modes
- But: no formula recalculation, limited formatting fidelity

On the QnA benchmark, the simpler approach won because **speed and reliability matter more than feature richness** for question-answering tasks. The CLI's per-operation overhead — compounded by the recalculation performance bug — meant the agent simply couldn't explore workbooks fast enough before timing out. The bottleneck was not Excel fidelity — it was being able to read and search data quickly and reliably.

**Note**: This comparison tested the **verify-style workflow** (separate CLI commands), not the **exec/REPL approach** (JavaScript code execution via `witan xlsx exec`). The exec approach addresses many of the issues found here — it batches multiple operations into a single CLI invocation, eliminating per-operation process spawn overhead, and provides a richer API surface designed for interactive exploration rather than post-hoc verification.

## What This Led To: The `exec` Command

This comparison was the direct catalyst for exposing the REPL as a CLI command. We already had the exec/REPL approach working well inside our agent (92% on QnA tasks by December). The verify-workflow comparison showed exactly why: the per-operation overhead of spawning a CLI process for each `find`, `calc`, or `render` call made standalone commands fundamentally uncompetitive for exploration-heavy tasks.

The solution was to bring the REPL's power to the CLI. `witan xlsx exec` runs a JavaScript script against a workbook in a single invocation — the agent can call `listSheets`, `findCells`, `readRange`, `setCells`, and `traceToInputs` all in one execution, with no per-operation overhead. This became the centerpiece of the `witan-exec` skill — a complete Excel coding agent in a single tool.

The verify commands (`render`, `calc`, `lint`) found their own role: not as the other half of exec, but as a standalone add-on for *any* spreadsheet agent. An agent that already uses openpyxl, pandas, or any other library for reading and writing can add the `witan-verify` skill to gain the capabilities it's missing — visual verification via render, formula correctness via calc, and semantic bug detection via lint. The verify skill doesn't replace the agent's existing tools; it complements them with a verification loop.

Two products emerged from one failed test:
1. **Exec** — a full spreadsheet coding environment exposed as a single CLI command, born from the REPL that proved itself inside our agent. Data exploration operations like `find` belong here, composed with other operations in a single invocation.
2. **Verify** — a tightly scoped verification toolkit (`render`, `calc`, `lint`) that any existing spreadsheet agent can adopt, regardless of what tools it already uses for reading and writing. `find` was removed from this set — it's a data exploration operation, not a verification gate.

In effect, this test closed a loop: the REPL approach that had proven itself inside our agent (Phase 10) was now externalized as a CLI command that any coding agent could use — not just our own.

## Priority Fix List

Ordered by effort/impact ratio:

1. **Fix SKILL.md quoting** — low effort, eliminates shell escaping failures
2. **Auto-convert .xls in runner** — low effort, eliminates extension errors
3. **Clarify calc -r behavior** — low effort, reduces wasted turns
4. **Add zero-tool-call retry** — low effort, catches hallucinations
5. **Batch query prompt improvements** — low effort, reduces turn count
6. **Reduce render usage** — low effort, saves tokens
7. **Controlled Python fallback** — low effort, prevents timeout deaths
8. **Fix API slowness** — high effort, highest impact
9. **Fix intermittent API failures** — high effort, high impact
10. **Fix PostToolUse tracing bug** — medium effort, improves debugging

## Key Takeaways

1. **Test your assumptions empirically.** The team expected the more sophisticated tool to perform better. It didn't, for well-defined reasons.

2. **Per-operation speed is a first-order concern.** 50-130s operations (caused by a recalculation performance bug, not network latency) killed the CLI approach. In agent systems, each tool call is on the critical path of reasoning.

3. **Documentation errors are agent errors.** Wrong SKILL.md instructions directly caused failures. The agent trusts its documentation.

4. **The right tool depends on the task.** QnA tasks (read-heavy, no editing) favored the simple, fast approach. Editing tasks with complex formatting might favor the higher-fidelity approach.

5. **Measure, don't assume.** This comparison would not have happened without the composable skills system making it a one-flag change.
