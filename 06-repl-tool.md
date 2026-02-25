# Deep Dive: The REPL Tool

**Phase**: 10 | **Date**: Nov 23, 2025

## Context

Before the REPL, the agent interacted with spreadsheets through multiple discrete tools or static views (TSV, SQL, HTML). Each tool call was isolated — no state persisted between calls, and composing operations required the LLM to manage state entirely in its reasoning context.

## Hypothesis

Instead of giving the LLM a fixed set of tools, give it a **persistent, programmable environment** — a REPL with access to a rich spreadsheet API. This mirrors how human analysts work: they explore interactively, store intermediate results, and compose operations fluidly.

## Architecture

### The REPL Tool (`xlsx_repl`)
```
Agent Loop ──(JavaScript code)──→ Node.js REPL ──(JSON-RPC)──→ .NET xlsx-serve
                                      ↑                              ↑
                                  State persists              Workbook state
                                  across calls                 persists
```

- **Single tool**: The agent has one tool (`xlsx_repl`) that accepts arbitrary JavaScript code
- **Persistent state**: Variables persist across tool calls — `const wb = await xlsx.openWorkbook("file.xlsx")` stays available
- **Rich API**: The `@witan/xlsx` library provides 30+ functions for reading, searching, tracing, computing, and writing
- **Sandbox**: No filesystem writes, no network, no subprocesses
- **Limits**: 90-second timeout per execution, 20K character output truncation

### The xlsx API Surface

**Reading**: `listSheets`, `getUsedRange`, `readCell`, `readRange`, `readRow`, `readColumn`, `readRangeTsv`, `getStyle`

**Searching**: `findCells`, `findRows`, `detectTables` (later `describeSheets`), `tableLookup`

**Tracing**: `getCellPrecedents`, `getCellDependents`, `traceToInputs`, `traceToOutputs`

**Computing**: `evaluateFormula`, `evaluateFormulas`

**Writing**: `setCells`, `scaleRange`, `insertRowAfter`, `deleteRows`, `insertColumnAfter`, `deleteColumns`, `addSheet`, `deleteSheet`, `setStyle`

**Rendering**: `previewStyles` (PNG screenshot of a range)

## Why This Was a Breakthrough

### 1. Composability
Before REPL:
```
Tool call 1: list_sheets() → ["Summary", "Data", "Inputs"]
Tool call 2: read_range("Summary!A1:D10") → cell values
Tool call 3: find_cells("Revenue") → [matches]
```
Each call requires an LLM round-trip.

After REPL:
```javascript
const sheets = await xlsx.listSheets(wb);
const summary = await xlsx.readRangeTsv(wb, `${sheets[0].name}!A1:D10`);
const revenue = await xlsx.findCells(wb, "Revenue", { context: 1 });
console.log(sheets, summary, revenue);
```
One tool call, three operations, all results visible together.

### 2. State Persistence
```javascript
// First tool call
const wb = await xlsx.openWorkbook("model.xlsx");
const tables = await xlsx.detectTables(wb);
const revenueTable = tables.find(t => t.headers.includes("Revenue"));

// Second tool call (variables persist)
const data = await xlsx.readRangeTsv(wb, revenueTable.range);
```

The agent can build up understanding incrementally without re-querying.

### 3. Flexible Exploration
The agent can try something, inspect the result, and adapt:
```javascript
// Try a search
let results = await xlsx.findCells(wb, "Net Revenue");
// No results? Try synonyms
if (!results.length) results = await xlsx.findCells(wb, ["Total Revenue", "Net Sales", "Revenue"]);
```

### 4. Single-Tool Simplicity
Instead of managing 15+ discrete tools with different schemas, the agent has one tool. The "tool documentation" is the API reference embedded in the system prompt.

## System Prompt Integration

The REPL API is documented in `tool-xlsx-repl.mustache`, embedded in the system prompt:

```
## REPL Tool (`xlsx_repl`)

### Hard rules
- Zero new formula errors: after every write, inspect calc.errors
- Use the model, not JavaScript math: read outputs or use evaluateFormula
- Always cite sheet + address in answers

### First-time exploration
xlsx = await import('@witan/xlsx');
wb = await xlsx.openWorkbook("filename.xlsx");
sheets = await xlsx.listSheets(wb);
tables = await xlsx.detectTables(wb);

### API Reference
{{{typedefs}}}    ← TypeScript type definitions injected here
```

The `{{{typedefs}}}` section is crucial: it injects the actual TypeScript type definitions for all API functions, giving the LLM precise signature information.

## Hard Rules

Three rules embedded in the prompt reflect hard-won lessons:

1. **"Zero new formula errors"** — After every write operation, the agent must check `calc.errors`. New formula errors indicate the write broke something.

2. **"Use the model, not JavaScript math"** — The agent must read calculated values from the workbook engine, never perform arithmetic in JavaScript. This prevents the agent from substituting its own (potentially wrong) calculations for the workbook's formula engine.

3. **"Always cite sheet + address"** — Every answer must reference the source cell (`Summary!C10`), making verification possible.

## Evolution After Initial Launch

### Nov 25-26: Search Improvements
- `findByLabels` replaced with general `lookup` function
- Fuzzy, case-insensitive text search
- Date-aware matching
- `in:` parameter for range scoping
- `context` parameter for surrounding cells

### Dec 2025: REPL Documentation
- Comprehensive `repl-docs.md` with variable persistence rules
- Timeout increased from 10s to 30s, later to 90s

### Feb 2026: Sub-Agent Namespacing
When sub-agents were added, each sub-agent got an isolated variable namespace (`varPrefix` system) to prevent collisions. The REPL remained the same; isolation was handled at the variable level.

### Feb 25, 2026: QuickJS Sandboxing
Replaced direct Node.js REPL execution with QuickJS-Emscripten for security. Agent code now runs in an isolated JavaScript engine with controlled access to the xlsx API.

## Comparison with Other Approaches

| Approach | Composability | State | API Growth | Complexity |
|----------|---------------|-------|------------|------------|
| Individual tools | Low (one per call) | None | Requires new tool definitions | High (many tools) |
| Batch dispatch | Medium (batch per call) | None | Requires new operation types | Medium |
| SQL queries | Medium (flexible queries) | Database | Schema changes | Medium |
| **REPL** | **High (arbitrary code)** | **Persistent** | **Just add functions** | **Low (one tool)** |

## What Worked

1. **Dramatically reduced tool calls** — complex explorations in 2-3 REPL calls instead of 10-15 individual tool calls.
2. **Natural programming patterns** — the agent could use loops, conditionals, and variables like a human programmer.
3. **Easy API extension** — adding a new function to `@witan/xlsx` immediately made it available to the agent.
4. **Comprehensive responses** — the agent could gather all needed data in one call and compose a thorough answer.

## What Needed Attention

1. **Output truncation** — 20K character limit sometimes cut off important results. The agent had to learn to be selective about what it logs.
2. **Timeout management** — complex operations could exceed the 90-second timeout. The agent needed to break up heavy work.
3. **Error recovery** — if the REPL process died, the agent needed to re-import and re-open the workbook. This was documented but still caused occasional failures.
4. **Prompt documentation quality** — the agent's effectiveness was directly proportional to how well the API was documented in the system prompt.

## Legacy

The REPL pattern became the foundational interaction paradigm for the entire product. Every subsequent feature — sub-agents, financial expert mindset, composable skills, Google Sheets support — built on top of the REPL. It was the single most impactful architectural decision in the project.

The pattern also informed the benchmark framework's skill design: the `witan-exec` skill documents the CLI's `exec` command, which is essentially an external REPL — run JavaScript against a workbook via the Witan API.
