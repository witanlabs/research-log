# Deep Dive: Three-Agent Architecture

**Phase**: 5 | **Date**: Oct 30, 2025 | **Repo**: python-utils

## Context

After achieving 50% on the benchmark with the initial approach and fixing critical bugs, the codebase had grown organically into a single agent trying to do everything: understand structure, answer questions, and edit workbooks. The agent's system prompt was becoming unwieldy, and the tool set mixed read-only and write operations.

## Hypothesis

Separating concerns into specialized agents — each with a focused prompt, specific tool access, and an appropriate reasoning effort level — should improve both accuracy and debuggability.

## The Three Agents

### 1. Block Discovery Agent
- **Purpose**: Identify workbook visual/semantic structure
- **Access**: Read-only SQL queries (`read_only_batch_tool`)
- **Reasoning effort**: Low (structure identification doesn't require deep reasoning)
- **Output**: Structured block descriptions (YAML/markdown)

### 2. Excel Edit Agent
- **Purpose**: Modify workbooks to match instructions
- **Access**: Full read-write (batch tool dispatcher with SQL queries and xlwings scripts)
- **Reasoning effort**: High (editing requires planning and verification)
- **Process**: 5-step structured workflow

### 3. Question Agent
- **Purpose**: Answer questions about workbook contents
- **Access**: Read-only SQL queries
- **Output**: Natural language answers

## The 5-Step Edit Process

The Excel Edit Agent followed a structured workflow:

1. **Disambiguate the request** — Clarify what the instruction means, especially for ambiguous terms like "delete the cells" (values vs. rows).
2. **Define the desired end state** — Describe what the workbook should look like after the edit, with SQL validation queries to verify.
3. **Plan the implementation** — Break the edit into concrete steps (which cells to modify, in what order).
4. **Execute the changes** — Use xlwings + batch tools to make the modifications.
5. **Verify completion** — Run the validation queries from step 2 to confirm the edit was applied correctly.

## Tool Architecture

The batch tool dispatcher used typed Pydantic operation schemas:

```python
class ExecuteSqlQueryOperation:
    id: str
    query: str

class ExecXlwingsScriptOperation:
    id: str
    script: str  # Python code executed via xlwings
```

Operations were dispatched in a batch — the agent specified a list of operations, and the harness executed them sequentially, returning all results.

## Pipeline

```
Input Spreadsheet
      ↓
[ExcelToSqlite] → SQLite DB + working Excel copy
      ↓
[Block Discovery Agent] → Block structure document
      ↓
[Excel Edit Agent] or [Question Agent]
  - Receives: blocks, database access, instructions
  - Edit Agent: 5-step process with xlwings
  - Question Agent: read-only SQL queries
      ↓
[Compare Sheets] → Pass/Fail vs gold standard
      ↓
Logged results
```

## What Worked

1. **Separation of concerns** — each agent could be optimized independently. Block discovery prompts were tuned without affecting the edit agent.
2. **The 5-step process** — structured reasoning reduced errors, especially for ambiguous instructions. Defining the desired end state before executing forced the agent to think ahead.
3. **Reasoning effort calibration** — using low effort for structure discovery and high effort for editing was a cost-effective tradeoff.
4. **Typed operations** — Pydantic schemas caught malformed tool calls before execution.
5. **Debuggability** — failures could be traced to specific agents, making root cause analysis easier.

## What Didn't Work

1. **Pipeline rigidity** — the sequential pipeline (discover → edit) couldn't handle cases where the agent needed to re-examine structure mid-edit.
2. **Inter-agent communication** — blocks were passed as flat text context, losing some structural information.
3. **Cost** — running three agents (two LLM calls + one task execution) per task was expensive.
4. **xlwings limitations** — xlwings required a running Excel instance, which was unreliable in automated settings (dialog boxes, timeouts).

## Key Learnings

1. **Structured reasoning processes (like the 5-step edit) are more valuable than sophisticated tool design.** The 5-step process forced the agent to plan, and the planning step caught many errors before execution.

2. **Reasoning effort calibration is a real lever.** Not every agent call needs maximum reasoning. Block discovery (pattern recognition) is fundamentally different from workbook editing (planning and execution).

3. **Read-only vs. read-write separation is a good safety pattern.** Preventing the question agent from accidentally modifying the workbook eliminated an entire class of bugs.

## What This Led To

- The concept of multiple specialized agents was carried forward into the production system's sub-agent delegation (Phase 16).
- The 5-step edit process influenced the QnA skill's structured workflows (value lookups, what-if analysis).
- The typed operation pattern evolved into the REPL's function API.
- The separation of read-only and read-write access persisted in sub-agent design (sub-agents are research-only).
