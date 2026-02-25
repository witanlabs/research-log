# Deep Dive: Rust Formula Analysis Pipeline

**Phase**: 7 | **Date**: Oct 15 - Nov 9, 2025 | **Repo**: witan-agent

## Context

In parallel with the Python agent work, a separate workstream explored a fundamentally different question: can deep understanding of spreadsheet formulas — their syntax, semantics, and dependency structure — provide a foundation for more sophisticated agent tooling?

This work was complementary to (not a replacement for) the agent development. The hypothesis was that formula analysis infrastructure would eventually power diagnostics, linting, and intelligent suggestions.

## Hypothesis

A 4-phase formula analysis pipeline (Parse → Validate → Analyze → Lint) built in Rust would provide:
1. High-performance formula processing for large workbooks
2. Precise diagnostics with source-location tracking
3. A foundation for advanced features (type inference, error detection, dependency tracing)
4. Cross-dialect support (Excel and Google Sheets)

## The Multi-Language Experiment (Oct 15-21)

### ABNF-to-PEG Converter
**Idea**: Excel publishes a formal grammar in ABNF notation. Derive the parser automatically.
**Result**: Abandoned. The official grammar is descriptive, not prescriptive — it doesn't match actual parser behavior. Empirical testing against real formulas matters more than formal correctness.
**Lesson**: Formal specifications and actual implementations diverge. Trust real-world behavior.

### Go Parser (Oct 20-21)
**Scope**: Full Excel formula parser — 758-line parser, 473-line lexer, 106-line AST, 39K+ test corpus snapshot.
**Features**: 224 function signatures, comprehensive test coverage, snapshot-based regression testing.
**Result**: Abandoned in favor of Rust on Oct 21.
**Reason**: Rust integrates better into the planned IDE/LSP ecosystem, and maintaining two parsers creates overhead.
**Lesson**: Pick one primary language early. The Go parser worked, but maintaining parity across two languages wasn't worth it.

### AWK-Based Analysis (experimental branch)
**Idea**: Convert XLSX to TSV, query with AWK scripts. Got 9/10 on test questions.
**Result**: Not pursued. Awk scripts are one-off; structured diagnostics scale better.
**Lesson**: Quick hacks can be seductively effective on small test sets but don't generalize.

## The Final Architecture: 10 Rust Crates

### Pipeline Design
```
Formula Text → Parse → Validate → Analyze → [Lint]
                ↓          ↓          ↓
           Diagnostics Diagnostics Diagnostics
```

Each phase has a single responsibility and emits its own diagnostics. Isolation enables reuse across tools (IDE, CLI, CI) and incremental enhancement.

### Crate Overview

| Crate | LOC | Purpose |
|-------|-----|---------|
| `formula_parser` | ~900 | Pest grammar → AST with span tracking |
| `formula_validator` | 542 | Function existence, arity, dialect checks |
| `spreadsheet_analyzer` | 3,537 | Dependency graph, circular detection, range normalization |
| `function_data` | 819 | 554 function signatures across Excel/Sheets |
| `diagnostic_core` | ~800 | Unified diagnostic infrastructure |
| `spreadsheet_model` | 1,113 | In-memory workbook snapshot |
| `spreadsheet_excel` | 295 | XLSX loading via calamine |
| `spreadsheet_cli` | 398 | Command-line interface |
| `function_docs_fetcher` | 1,145 | Excel/Sheets function doc scraper |

### Key Technical Decisions

**Span tracking**: Every AST node carries byte-offset spans, enabling precise error location. When the analyzer detects a reference to a nonexistent sheet, the diagnostic underlines the exact sheet name in the formula — not the whole expression.

**Dialect abstraction**: Excel and Google Sheets have different function availability, syntax features (implicit intersection, structured references), and grid sizes. The pipeline parameterizes these differences through `SyntaxOptions` and `Dialect` enums.

**Dependency graph (Tarjan SCC)**: The analyzer builds a bidirectional dependency graph with forward (precedent) and reverse (dependent) edges. Circular references are detected using Tarjan's Strongly Connected Components algorithm.

**Range normalization**: A dyadic tile-based algebra engine handles overlapping ranges, structured references with qualifiers (#Headers, #Totals, #Data, #ThisRow), and cross-sheet references. This enables precise diagnostics like "cells A50:A100 contribute with net weight 1.5."

**Faceted function metadata**: 554 functions have metadata split across facet files (CSV + JSON): argument names, return types, value type constraints, enumeration values. This separation lets contributors add metadata without understanding the full system.

## Planned Diagnostics (22 Rules)

We designed 22 diagnostics ranked by impact:

| ID | Rule | Category |
|----|------|----------|
| D01 | Double counting / multi-inclusion | Logic |
| D02 | Approximate match on unsorted array | Logic |
| D03 | Empty cell coercion | Type |
| D04 | Fill/reference pattern mistake | Pattern |
| D05 | Numeric range ignores text/booleans | Type |
| D06-D22 | (additional rules covering stale refs, inconsistent units, etc.) | Various |

These diagnostics were designed but not fully implemented in the Rust codebase — they represent the aspirational end state.

## Connection to Agent Work

While the Rust pipeline didn't directly power the agent, it influenced several aspects:

1. **Formula understanding**: The dependency graph concept informed how the agent traces formula chains (later implemented as `getCellPrecedents` / `traceToInputs` in the production API).
2. **Error detection**: The diagnostic taxonomy influenced the linting rules built later in the .NET backend.
3. **SQLite schema**: A comprehensive database schema design for full-fidelity workbook representation, drawing on lessons from both the Python SQLite extraction and the Rust analysis.
4. **Dialect awareness**: The Excel vs. Google Sheets distinction carried into the later Google Sheets integration work.

## Project Methodology

The project demonstrated disciplined engineering practices:
- **Execution plans**: Every significant feature started with a written plan.
- **Decision logs**: Each crate maintained a chronological record of key decisions.
- **50 completed plans**: Documenting rationale, alternatives considered, and decisions made.
- **CI discipline**: Build, test, lint, and format checks required before every commit.
- **Snapshot testing**: 39K+ formula corpus for regression testing.

## Status

As of Nov 9, 2025, the pipeline had:
- Complete parser with Excel/Sheets dialect support
- Formula validation (function, arity, dialect)
- Dependency graph with circular detection
- Range normalization
- Diagnostic core infrastructure
- CLI tool for interactive analysis

Remaining: type inference, the 22 diagnostic rules, incremental persistence, and deeper integration with the agent tooling.
