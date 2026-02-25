# Deep Dive: SQLite as Knowledge Representation

**Phase**: 1 | **Date**: Oct 22, 2025 | **Repo**: python-utils

## Context

The fundamental challenge: Excel workbooks are dense, multi-dimensional artifacts. A single workbook might contain 50,000+ cells across 20 sheets, with complex formatting, formulas referencing other sheets, charts, pivot tables, and merged regions. An LLM's context window cannot hold all of this, and even if it could, the flat text representation would be overwhelming.

## Hypothesis

By converting the workbook into a SQLite database with a rich relational schema, the agent can:

1. Selectively query only the data it needs
2. Use SQL's expressive power to find patterns and relationships
3. Understand structure through metadata tables rather than scanning raw cells
4. Operate on a consistent, queryable representation regardless of workbook complexity

## Implementation

### The Schema

The extraction pipeline converts an Excel workbook into a SQLite database with these key tables:

**Core Data**:

- `cells` — every cell with value, type (date, number, text, boolean, error), and sheet reference
- `formulas` — formula text with coordinate mapping (Excel-style references like A1:B5)
- `array_formulas` — multi-cell array formulas

**Structure**:

- `merged_blocks` — merged cell ranges (common in headers and titles)
- `color_blocks` — contiguous regions with the same background color (visual structure indicator)
- `compute_rectangles` — contiguous data regions detected by value presence
- `text_rectangles` — contiguous text regions

**Formatting**:

- `cell_formatting` — font styles, colors, borders, number formats per cell

**Charts** (added later in Phase 6):

- `charts` — chart metadata (type, position, dimensions)
- `chart_series` — data series within charts

**Pivot Tables** (added later):

- `pivot_tables`, `pivot_caches`, `pivot_fields`

### The Agent

The first agent used DeepAgents (LangGraph) with ChatOpenAI (GPT-5). Tools:

```python
execute_sql(query: str) -> str           # Read-only SQL against the database
write_cells(cells: list[CellWrite]) -> str  # Write values (str, int, float)
add_sheet(name: str) -> str              # Add a new sheet
copy_values(src: str, dst: str) -> str   # Copy cell ranges
delete_ranges(ranges: list[str]) -> str  # Delete cell contents
insert_rows(sheet: str, row: int) -> str # Insert rows
insert_cols(sheet: str, col: int) -> str # Insert columns
```

Each tool was called individually — one tool call per LLM turn.

### The SpreadsheetBench Dataset

130 questions from the SpreadsheetBench benchmark, stored in `spreadsheetbench-questions.json` (640KB). Each question includes:

- Natural language instructions
- Input spreadsheet reference
- Expected answer position (cell range to compare)
- Gold-standard answer spreadsheet

### Evaluation

A sheet comparison utility performed cell-level comparison:

- Value normalization (numbers, dates, strings)
- Cell-level difference reporting
- Handles Excel epoch conversions for date comparison

## What Worked

1. **SQL for exploration** — queries like `SELECT * FROM cells WHERE sheet='Summary' AND value LIKE '%revenue%'` efficiently found relevant data.
2. **Metadata tables as structure indicators** — `color_blocks` and `compute_rectangles` gave the agent clues about visual groupings without scanning every cell.
3. **Decoupling source of truth from working copy** — the database was read-only; modifications went through xlwings to the actual Excel file.

## What Didn't Work

1. **Sequential tool calls** — each SQL query required a full LLM round-trip. Exploring a workbook took many turns.
2. **Flat relational representation** — SQL tables don't naturally represent the hierarchical, visual structure humans see in spreadsheets.
3. **Missing data types** — initial extraction didn't capture charts, formatting details, or pivot tables, causing failures on tasks that depended on these.
4. **Value type limitations** — `write_cells` only supported str/int/float, not dates, formulas, or complex types.

## Legacy

The SQLite approach established several patterns that persisted:

- **Preprocessing the workbook for the LLM** — don't just hand the agent raw data; transform it into a representation optimized for LLM reasoning.
- **Selective access** — let the agent query for what it needs rather than dumping everything into context.
- **Rich metadata extraction** — capturing structure (not just values) is essential.

The SQLite approach was eventually superseded by the REPL-based approach (Phase 10), which gave the agent more interactive, programmatic access. But the insight that workbooks need a "knowledge layer" between raw Excel and the LLM persisted.

## What This Led To

- **Phase 2**: Batch execution (addressing the sequential tool call problem)
- **Phase 3**: Block discovery (addressing the flat representation problem)
- **Phase 6**: Data fidelity expansion (addressing the missing data types problem)
- **Phase 8**: The TypeScript+.NET pivot (addressing fidelity and productionization)
