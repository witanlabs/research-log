# Deep Dive: Block Discovery

**Phase**: 3, 5 | **Date**: Oct 25-30, 2025 | **Repo**: python-utils

## Context

After building the SQLite knowledge layer, the agent could query individual cells and metadata. But it struggled with a fundamental problem: understanding the **visual and semantic structure** of a spreadsheet. Humans look at a spreadsheet and instantly see "there's a header row here, a data table there, some KV pairs in the corner." The LLM had to reconstruct this from flat SQL results.

## Hypothesis

If a specialized agent pre-processes the workbook to identify "blocks" — semantic/visual groupings of cells — the main task agent can reason at a higher level of abstraction. Instead of "cell A1 has value 'Revenue', cell B1 has value 'Q1'...", the agent gets "there's a financial data table spanning A1:F20 with quarterly revenue data."

## Block Taxonomy

The block discovery system identified these block types:

| Type | Description | Example |
|------|-------------|---------|
| **banner** | Title or header spanning the full width | "Financial Summary FY2024" merged across columns |
| **band** | Colored row or column acting as a section divider | Blue-shaded row separating income from expenses |
| **group** | Related cells without strict table structure | A cluster of input assumptions |
| **table** | Structured data with headers and rows | Revenue by product and quarter |
| **kv** | Key-value pairs | "Currency: USD", "Fiscal Year: 2024" |
| **note** | Annotations or comments | Footnotes explaining methodology |

## Implementation

### Block Discovery Agent

A separate LLM agent (GPT-5, low reasoning effort) with read-only SQL access. The agent:

1. Queries the SQLite database for structural metadata:
   - `color_blocks` — contiguous same-color regions
   - `merged_blocks` — merged cell ranges
   - `compute_rectangles` — contiguous data regions
   - `text_rectangles` — contiguous text regions
2. Samples actual cell values to understand content
3. Produces a structured markdown/YAML description of blocks

### Input to Block Discovery

The agent receives a workbook summary extracted from the database — a condensed view of what sheets exist, what metadata tables contain, and sample values.

### Output Format

Structured markdown describing each block with:
- Sheet name and cell range
- Block type
- Description of content
- Sample values
- Relationships to other blocks (e.g., "this KV block contains inputs used by the data table below")

## Prompt Sensitivity

The block discovery prompt was **highly sensitive** to phrasing. Evidence:

- A git commit message from this period reads: *"gonna revert to another block discovery prompt"* — indicating the current prompt wasn't performing well enough.
- The original prompt was heavy on efficiency constraints and block pattern detection rules.
- The revised prompt added a "markdown output contract" with structured format requirements.
- It included guidance on safety and uncertainty handling — what to do when block boundaries are ambiguous.

This was an early lesson in **prompt engineering as a critical engineering discipline**. Small changes in how the block discovery task was described had measurable downstream impact on the main agent's task performance.

## Integration with Main Agent

The block discovery output was passed as context to the main agent (Excel Edit Agent or Question Agent). The main agent could then:
- Reference blocks by name: "modify the revenue table" instead of "change cells A5:F20"
- Understand relationships: "the input assumptions in the KV block drive the projections in the data table"
- Navigate efficiently: jump to the relevant block instead of searching the entire workbook

## What Worked

1. **Higher-level reasoning** — the main agent made fewer navigation errors when it understood the workbook structure.
2. **Reduced exploration turns** — instead of querying many cells to understand structure, the agent started with a structural overview.
3. **Better disambiguation** — knowing that "Revenue" appears in both a summary table and a detailed table helped the agent choose correctly.

## What Didn't Work

1. **Prompt fragility** — small prompt changes could significantly degrade block quality.
2. **Cost of two-agent pipeline** — running block discovery added latency and cost (another LLM call) to every task.
3. **Fixed taxonomy** — the 6 block types didn't always capture real-world spreadsheet structures well.
4. **One-shot discovery** — blocks were discovered once at the start; if the agent needed to re-examine structure mid-task, it couldn't easily re-run discovery.

## Evolution

The block discovery concept evolved through several incarnations:

1. **python-utils (Phase 3)**: Separate LLM agent with SQL queries, producing markdown.
2. **the production system (Phase 8-10)**: Replaced with `detectTables` function in the xlsx library — algorithmic detection rather than LLM-based.
3. **the production system (Phase 18)**: `detectTables` replaced by `describeSheets` — broader structural description.

The trend was from **LLM-based structure identification** to **algorithmic structure detection**. The LLM-based approach was creative and flexible but unreliable; the algorithmic approach was more consistent but less nuanced.

## Legacy

The block discovery experiment established that **workbook structure preprocessing is essential**. Even when the specific mechanism changed (LLM agent → algorithm → richer algorithm), the principle remained: give the agent a structural overview before asking it to work on details.

The concept also influenced the `describeSheets` design in the production system and the table detection improvements that remain on the roadmap (multi-table detection, date header detection, etc.).
