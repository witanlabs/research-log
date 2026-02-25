# Teaching Machines to Read Spreadsheets

### Four months of building an LLM-powered spreadsheet agent — what we tried, what broke, and what worked

---

### Lessons at a glance

1. **Let the agent write code, not call tools.** A programmable REPL where the agent composes operations in JavaScript beat every approach based on discrete tool calls. ([details](#the-repl-the-decision-that-changed-everything))
2. **Agent failures that look like reasoning failures are often data pipeline bugs in disguise.** A one-character extraction bug caused more wrong answers than any prompt issue. Our instinct was to blame the model — the actual problem was upstream. ([details](#the-first-attempt-turn-the-spreadsheet-into-a-database))
3. **Prompts are code.** Small wording changes in block discovery prompts measurably shifted performance. A SKILL.md with backwards quoting examples caused cascading agent failures. Prompt quality determined agent quality at every phase — version, test, and review them like source code. ([details](#block-discovery-teaching-the-agent-to-see-structure))
4. **Make the agent think before acting.** A five-step structured reasoning process (disambiguate, define end state, plan, execute, verify) was the highest-leverage intervention for editing tasks. ([details](#three-agents-not-one))
5. **Domain knowledge is as portable as tools — and often more valuable.** Financial expertise and QnA workflow skills improved results regardless of tool backend. ([details](#domain-knowledge-as-a-product))
6. **Benchmark relentlessly.** Every significant improvement was motivated by evaluation results — from the 50%→73% bug fix to the 74%→92% REPL trajectory to the humbling verify-vs-openpyxl comparison. ([details](#the-results-from-74-to-92-in-two-weeks))
7. **A failed test can be the most important one.** Testing the wrong workflow for the task led directly to the product architecture — exec for coding, verify as an add-on. ([details](#the-test-that-shaped-the-product))

---

Spreadsheets are deceptively hard for AI. A human glances at a financial model and instantly sees the structure: there's a revenue table here, assumptions in that yellow-shaded corner, a chart summarizing the P&L. They know that "Q4" means the fourth column in a quarterly layout, that parentheses mean negative numbers, that the cell labeled "EBITDA" is derived from the ones above it.

A large language model sees none of this. It sees text — thousands of cell values, cryptic formulas like `=SUMPRODUCT((B$3:B$50="NEW")*(G$3:G$50))`, and formatting metadata it has no intuition for. Give it a 20-sheet financial model and ask "what's the IRR?", and it has to figure out not just what IRR means, but which of the 10,000 cells contains the answer, which sheet it's on, and whether the number it finds is an input assumption or a calculated output.

This is the story of how we spent four months — from October 2025 through February 2026 — building an agent that can do this. It's a story of approaches tried and abandoned, breakthroughs that came from unexpected places, and the recurring discovery that simpler solutions often beat sophisticated ones.

---

## The first attempt: turn the spreadsheet into a database

> Detailed reference: [SQLite Knowledge Representation](https://github.com/witanlabs/research-log/blob/main/01-sqlite-knowledge-representation.md)

Our earliest instinct was to transform the problem. An LLM can't hold a 50,000-cell workbook in its context window, but it's quite good at writing SQL queries. So in late October 2025, we built a pipeline that converts Excel workbooks into SQLite databases — every cell, formula, merged region, and color block stored in relational tables.

The agent was a LangGraph-based system using GPT-5. It had a handful of tools: run a SQL query, write to cells, insert rows, delete ranges. Each tool call was a separate round-trip to the language model. The agent would query the database to understand the workbook, then use xlwings (a Python library for Excel automation) to make modifications.

We tested it against SpreadsheetBench, a public benchmark of 130 spreadsheet manipulation tasks. The first results were sobering: around 50% pass rate. But when we looked closer, roughly 15% of the questions were unfair — ambiguous instructions, incorrect answer keys, or tasks that required resolving genuinely ambiguous human language. Excluding those, we were at 73%.

Two things jumped out immediately. First, a one-character bug in our extraction code — a wrong function argument — was silently corrupting data for a large fraction of tasks. Fixing that single character moved the needle more than any prompt engineering we'd done. The agent had *looked* confused — giving wrong answers, failing to find data — and our instinct was to blame the model or iterate on the prompt. The actual problem was upstream: the agent was reasoning correctly over corrupted inputs.

Second, the agent struggled not with reasoning but with *navigation*. It could figure out what to do, but it couldn't efficiently find the right cells to work with.

**Lesson: when an agent behaves poorly, the instinct is to blame the model. But the root cause is often in the infrastructure feeding it data. Agent failures that look like reasoning failures are frequently data pipeline bugs in disguise.**

---

## Block discovery: teaching the agent to see structure

> Detailed reference: [Block Discovery](https://github.com/witanlabs/research-log/blob/main/02-block-discovery.md)

The navigation problem led to our first real architectural experiment. Humans look at a spreadsheet and see blocks — a header row here, a data table there, some key-value inputs in the corner. Our agent was reconstructing this structure from raw SQL results, one query at a time, and doing it poorly.

We built a "block discovery agent" — a separate, lightweight LLM call that runs before the main task agent. Its job: query the database for structural signals (colored regions, merged cells, contiguous data blocks) and produce a structured description of the workbook's layout. The main agent then receives this structural overview as context.

The concept worked. The main agent made fewer navigation errors when it started with a map of the territory. But the block discovery prompt was *extremely* sensitive to phrasing. Small wording changes measurably changed downstream task performance.

This was our first encounter with a theme that would recur throughout the project: **prompt quality determines agent quality, and prompt engineering is real engineering — fragile, hard to test, and high-leverage.**

---

## Three agents, not one

> Detailed reference: [Three-Agent Architecture](https://github.com/witanlabs/research-log/blob/main/03-three-agent-architecture.md)

By the end of October, we'd refactored from a monolithic agent into three specialists:

1. A **block discovery agent** (low reasoning effort) that identifies workbook structure.
2. An **edit agent** (high reasoning effort) that modifies workbooks following a five-step process: disambiguate the request, define the desired end state, plan the implementation, execute, and verify.
3. A **question agent** (read-only) that answers questions about workbook contents.

The separation of concerns helped. The edit agent's five-step process was particularly effective — by forcing the agent to define the end state *before* executing, it caught many errors during planning that would otherwise surface as incorrect outputs. And restricting the question agent to read-only access eliminated an entire class of accidental modification bugs.

But the pipeline was rigid. Discovery happened once at the start; the agent couldn't re-examine structure mid-task. And running three LLM calls per task was expensive.

**Lesson: structured reasoning processes (like the five-step edit) matter more than sophisticated tool design. Making the agent *think before acting* is the highest-leverage intervention.**

---

## A parallel track: understanding formulas from the ground up

> Detailed reference: [Rust Formula Pipeline](https://github.com/witanlabs/research-log/blob/main/04-rust-formula-pipeline.md)

While the agent work progressed in Python, a parallel workstream tackled a different question: can we build deep understanding of spreadsheet formulas? We explored this in Rust, building a ten-crate monorepo implementing a four-phase pipeline: parse formulas into an AST with source-location tracking, validate against a catalog of 554 Excel/Sheets functions, analyze dependency graphs with circular reference detection, and lint for common errors.

Along the way we tried and discarded several approaches — a Go parser (758 lines, 39,000-formula test corpus), an ABNF-to-PEG grammar converter, even AWK scripts against TSV exports (which, alarmingly, answered 9 out of 10 test questions).

This pipeline didn't directly power the agent. But it deeply informed the production tools. Tracing formula dependencies became `traceToInputs` in our API. Range normalization — a tile-based algebra engine for detecting overlapping references — became the foundation for our linting rules. The diagnostic taxonomy of 22 formula-level bugs shaped what our linter catches today.

**Lesson: deep domain understanding, even when it doesn't ship directly, pays dividends by shaping the tools you build.**

---

## The pivot: TypeScript, .NET, and a new foundation

By early November, we'd hit the ceiling of the Python approach. openpyxl and xlwings had fidelity gaps with complex workbooks. The LangGraph agent architecture was hard to deploy as a web service. And we wanted a product, not just a benchmark runner.

We started fresh with a new codebase: TypeScript for the agent loop and API, .NET for Excel manipulation. The .NET engine handles complex workbooks — charts, pivot tables, conditional formatting, formula evaluation — with much higher fidelity than Python libraries. The two halves communicate via JSON-RPC over stdio: the TypeScript agent sends commands, the .NET process manages workbook state.

The first few weeks were a period of exploration. We tried multiple ways to present workbook data to the LLM, and none felt right:

| Representation | Approach | Why abandoned |
|---|---|---|
| **TSV views** | Tab-separated values with address\|value\|formula format | Lost formatting and structural context as a standalone representation; later kept as one of the read methods inside the REPL |
| **SQL views** | Relational database (echoing our earlier approach) | Flat tables didn't capture visual structure |
| **HTML tables** | Formatted HTML rendering | Verbose, consumed too many tokens |
| **DOT graphs** | Graphviz visualization of formula dependencies | Useful for analysis but not for agent interaction |
| **XML/XSLT** | XML workbook representation with XSLT stylesheets for operations | Too verbose; LLM struggled with the syntax |

Each representation captured some aspects of the workbook but lost others. The breakthrough, two weeks later, would come from abandoning the search for the right static representation entirely.

We also built an exercises framework from day one — small, hand-crafted tasks with clear grading criteria. These gave us a tight feedback loop: make a change, run exercises, see results in minutes. This was faster and more controlled than the full benchmark suite, and we used it heavily throughout development.

---

## The REPL: the decision that changed everything

> Detailed reference: [The REPL Tool](https://github.com/witanlabs/research-log/blob/main/06-repl-tool.md)

On November 23rd, we made the single most impactful architectural decision of the project: we gave the agent a REPL.

Instead of multiple discrete tools ("read this cell", "search for this label", "write this value"), the agent got one tool: a persistent Node.js REPL with access to a rich spreadsheet API. The agent writes JavaScript code. Variables persist across calls. The API includes functions for listing sheets, searching for cells, reading ranges, tracing formula dependencies, evaluating formulas, and writing values.

This sounds simple, but it changed the agent's relationship with the workbook fundamentally. Before the REPL, exploring a workbook was like asking questions through a translator — each question required a round-trip to the LLM. With the REPL, the agent could *program*: chain operations, store intermediate results, try something and adjust based on the output, use loops and conditionals.

A typical exploration before the REPL took 10-15 tool calls. After the REPL, the same exploration took 2-3 calls. The agent could write:

```javascript
const sheets = await xlsx.listSheets(wb);
const tables = await xlsx.detectTables(wb);
const revenue = await xlsx.findCells(wb, "Revenue", { context: 1 });
console.log(sheets, tables, revenue);
```

One call, three operations, all results visible together.

The REPL also made API evolution trivial. Adding a new capability — say, `traceToInputs` to follow formula chains back to hardcoded assumptions — meant adding a function to the xlsx library. The agent could use it immediately through the REPL, with no changes to the tool layer.

Three hard rules, embedded in the system prompt, encoded lessons from earlier failures:

1. **"Zero new formula errors"**: after every write, check for errors. If your edit broke a formula, fix it before moving on.
2. **"Use the model, not JavaScript math"**: read calculated values from the workbook's formula engine. Never substitute your own arithmetic. The spreadsheet's formulas are the source of truth.
3. **"Always cite sheet + address"**: every answer must reference where the data came from (`Summary!C10`), making verification possible.

**Lesson: give the agent more autonomy, not more tools. A programmable environment beats a fixed tool set.**

---

## The results: from 74% to 92% in two weeks

With the REPL in place, we benchmarked our agent against 165 financial analysis questions — tasks like "What's the IRR of this business plan?", "What happens to EBITDA if revenue increases 10%?", and "Which segment is most profitable by operating margin?"

The results improved rapidly:

- **November 30**: 74% pass rate (77/104 tasks)
- **December 11**: 88% (147/167 tasks)
- **December 14**: 92.1% (152/165 tasks), zero timeouts, 50-second average runtime

The 18-point improvement in two weeks came from compounding small gains: better search (fuzzy matching, contextual results), new API functions (formula tracing, TSV range reads), improved system prompt documentation, and bug fixes in the .NET backend. No single change was dramatic. But each one removed a class of failures, and the effects multiplied.

---

## Building the evaluation machine

> Detailed reference: [Benchmark Evolution](https://github.com/witanlabs/research-log/blob/main/05-benchmark-evolution.md)

Running an agent once and checking the answer manually doesn't scale. We built a benchmark framework on LangSmith that grew to 568 commits and 29,000 lines of TypeScript — nearly as complex as the agent itself. It supported seven agent runners, five evaluation strategies, and a composable skills system for A/B testing tool approaches by changing a CLI flag.

The evaluation methodology taught us a hard lesson early. We started with an LLM comparing agent output to gold-standard workbooks. This was unreliable — the evaluator made inconsistent judgments that masked real progress. We switched to deterministic, programmatic comparison: cell values by set similarity, layout by sequence alignment, formatting by pixel-accurate comparison. LLM-based grading survived only for genuinely subjective assessments (open-ended text answers).

**Lesson: use LLMs for evaluation only when the judgment is genuinely subjective. For everything else, use deterministic comparison.**

---

## Domain knowledge as a product

Throughout this work, we increasingly recognized that the agent wasn't just a tool operator — it needed to be a **domain expert**. A question like "what's the profitability?" isn't asking for a cell value; it's asking for margins, trends, and context. "What if revenue increases 10%?" requires understanding that revenue changes cascade through COGS, taxes, working capital, and cash flow.

We embedded financial expertise directly into the system prompt: common metrics and their relationships, how to interpret financial statements, model type recognition (DCF, LBO, three-statement), and communication conventions (use $1.2M, not $1,234,567.89; parentheses for negatives; always provide year-over-year context).

The most surprising finding was that this domain knowledge was **more portable than the tools**. We structured it as a reusable skill component — applicable regardless of whether the agent uses openpyxl, the Witan CLI, or anything else. We did the same with QnA workflows: codified rules about disambiguation, temporal references, and guardrails ("never calculate in your head what the spreadsheet can calculate for you") that improved results across every tool backend we tested.

**Lesson: domain knowledge, structured as reusable prompt components, is as important as tool access — and more portable.**

---

## The test that shaped the product

> Detailed reference: [CLI vs. openpyxl](https://github.com/witanlabs/research-log/blob/main/07-cli-vs-openpyxl.md)

By this point, the REPL/exec approach had already proven itself — 92% on QnA tasks, consistently beating openpyxl. But we had a second Witan workflow: standalone CLI commands (`find`, `calc`, `render`, `lint`) that agents could invoke individually. We tested *that* workflow against openpyxl on the same kind of QnA exploration tasks. Same model (Claude Opus 4.6), same 20 tasks, same runner.

The result: **openpyxl won, 85% to 70%.**

This wasn't Witan losing to openpyxl — the exec approach had already beaten openpyxl months earlier. It was the wrong workflow for the task. The reasons were mundane:

- **Processing speed**: Each Witan CLI command took 50-130 seconds — not due to network latency (the API ran locally), but because of a degenerate performance bug in the recalculation engine that queried the dependency index per-cell instead of in batch. openpyxl avoids this entirely by reading cached values without recalculating. With a separate CLI process spawn per operation, these delays compounded quickly. Tasks timed out before the agent could finish exploring.
- **Shell escaping**: Sheet names with spaces (like "SW Selections") broke due to a three-layer quoting interaction between the Claude Code SDK, zsh, and the .NET parser. openpyxl just uses Python strings.
- **Documentation errors**: Our own SKILL.md file — the instructions telling the agent how to use the CLI — had the quoting examples backwards. The agent followed our wrong instructions faithfully.

This was sobering — but it was also a comparison of the wrong workflow for the task. Using separate CLI commands for data exploration meant spawning a new process for each query. For QnA tasks that require 20+ queries to navigate a workbook, this per-operation overhead was fatal.

But the test answered a question we hadn't quite asked yet: *should the REPL be the product?*

We'd built it for our own agent — an internal tool powering the system that hit 92% on QnA tasks. The verify-workflow comparison showed exactly why it worked: a single `exec` invocation runs an entire exploration script without spawning a process for each operation. The verify commands were good at what they were designed for — checking your work after edits. But they were the wrong tool for exploration.

This realization closed a loop. We took the REPL that had proven itself inside our agent and exposed it as `witan xlsx exec` — a complete spreadsheet coding environment in a single CLI command that any coding agent could use, not just our own.

The remaining commands — `render`, `calc`, and `lint` — found their own role as a standalone verification add-on. `find` was removed: it's a data exploration operation, not a verification gate, and it belongs inside exec where it can be composed with other operations. The verify skill became a tightly scoped trio: any agent that already uses openpyxl, pandas, or whatever else for spreadsheet work can add `render`, `calc`, and `lint` to gain the verification capabilities it's missing — eyes (render), a formula engine (calc), and a code reviewer (lint).

**Lesson: the most important test isn't the one that confirms your assumptions — it's the one that reveals your product.**

---

## What we learned

Four months of work across four codebases, hundreds of experiments, and tens of thousands of lines of code distilled into a few recurring themes:

**The REPL pattern won.** The evolution from individual tools to batch dispatch to a programmable REPL was the clearest architectural trajectory. Giving the agent a persistent, composable environment with a rich API was more effective than any fixed tool set. The REPL grew organically — adding a new function meant adding it to the library, with no tool-layer changes needed.

**Prompt quality is engineering quality.** Block discovery prompts sensitive to small wording changes. SKILL.md documentation with backwards quoting examples that caused cascading agent failures. QnA workflows that codified disambiguation rules. At every phase, the quality of the agent's instructions determined its performance. Prompts are code — they should be versioned, tested, and reviewed with the same rigor.

**Evaluation drives progress.** Every significant improvement was motivated by benchmark results. The one-character bug that moved us from 50% to 73%. The REPL agent trajectory from 74% to 92%. The verify-vs-openpyxl comparison that revealed our documentation was backwards — and that the REPL should be the product. Without rigorous, automated evaluation, we'd be guessing.

**Domain knowledge is as portable as tools — and sometimes more valuable.** The financial expert mindset and QnA workflow skills improve results regardless of the tool backend. They capture knowledge that took months to develop and can be applied to any future approach.

**Infrastructure bugs masquerade as reasoning failures.** Our most impactful "prompt engineering" was often fixing a bug in the extraction pipeline, the API, or the documentation. When the agent seems confused, suspect the plumbing.

---

## Where this goes

As of late February 2026, the system handles financial QnA tasks at 92% accuracy with sub-minute response times. The agent can explore complex workbooks, perform what-if analysis through ephemeral writes, trace formula dependencies, and communicate findings with financial domain expertise.

Open questions remain. Table detection — automatically identifying the boundaries and semantics of data tables in arbitrary spreadsheets — is still an active research problem. Search quality (fuzzy matching, semantic search, relevance ranking) has room to improve. And the Google Sheets integration, currently in progress, will test whether our architecture is truly spreadsheet-engine-agnostic.

But the core insight holds: the best way to help an LLM work with spreadsheets is not to give it more tools, but to give it a better environment — a programmable, persistent, well-documented workspace where it can explore and compose operations like the expert analyst it's learning to become.

---

## Appendix: Timeline

| Date | Key Event |
|------|-----------|
| Oct 15-20 | Rust parser experiments begin (Go, ABNF, PEG) |
| Oct 22 | SQLite knowledge representation, first agent |
| Oct 25 | Batch execution, block discovery agent |
| Oct 27 | **50% benchmark**, bug fixes |
| Oct 30 | Three-agent architecture |
| Oct 31 | TypeScript + .NET monorepo created |
| Nov 1 | Benchmark framework: first commit |
| Nov 4 | Multiple runners, **"poor vs openpyxl"** signal |
| Nov 9 | Analyzer v1, dependency graphs |
| Nov 17-19 | Evaluation strategy specialization |
| Nov 23 | **REPL tool breakthrough** |
| Nov 25-26 | Search & lookup evolution |
| Nov 30 | REPL agent benchmark: **74% on 104 QnA tasks** |
| Dec 5 | Benchmark orchestrator with multi-run support |
| Dec 14 | REPL agent benchmark: **92% on 165 QnA tasks** |
| Jan 23 | Financial expert mindset |
| Feb 10 | Opus 4.6 support with thinking |
| Feb 17 | **Verify-vs-openpyxl** comparison → catalyst for exec |
| Feb 24 | Composable skills system |
| Feb 25 | Google Sheets spike, QuickJS sandboxing |

## Deep Dives

For detailed analysis of individual experiments:

1. [SQLite Knowledge Representation](./01-sqlite-knowledge-representation.md) — the first approach: converting workbooks to queryable databases
2. [Block Discovery](./02-block-discovery.md) — teaching the agent to see spreadsheet structure
3. [Three-Agent Architecture](./03-three-agent-architecture.md) — separating discovery, editing, and question answering
4. [Rust Formula Pipeline](./04-rust-formula-pipeline.md) — deep formula analysis that shaped the production tools
5. [Benchmark Evolution](./05-benchmark-evolution.md) — how evaluation methodology matured alongside the agent
6. [The REPL Tool](./06-repl-tool.md) — the most impactful architectural decision
7. [Verify vs. openpyxl](./07-cli-vs-openpyxl.md) — the test that shaped the product
