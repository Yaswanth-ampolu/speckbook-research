# regex_log_search — Dynamic Pattern-Driven Log Search Tool

**Date:** 2026-07-20
**Keyword:** regex-log-search, tool-design, log-filtering, viv, rtl-debugging
**Status:** Design Specification

---

## Table of Contents

1. [What Is regex_log_search?](#1-what-is-regex_log_search)
2. [Why Not Just Read the File?](#2-why-not-just-read-the-file)
3. [How Viv v0.2.3 Used It — Observed Behavior](#3-how-viv-v023-used-it--observed-behavior)
4. [The Pattern Derivation Engine](#4-the-pattern-derivation-engine)
5. [Dynamic & Suggestive Design](#5-dynamic--suggestive-design)
6. [Progressive Refinement Strategy](#6-progressive-refinement-strategy)
7. [Tool API Specification](#7-tool-api-specification)
8. [Implementation Design](#8-implementation-design)
9. [Integration with the Investigation Pipeline](#9-integration-with-the-investigation-pipeline)
10. [Resource & Performance Considerations](#10-resource--performance-considerations)

---

## 1. What Is regex_log_search?

`regex_log_search` is the **first tool called in any RTL debug investigation**. It searches simulation log files using dynamically-derived regex patterns to find the failure signature without loading entire log files into the LLM's context window.

It is **not** a grep wrapper. It is a **pattern intelligence system** — the LLM derives *what* to search for from context, the tool executes the search efficiently, and the results feed back into pattern refinement.

### Relationship to regex_code_search

`regex_code_search` is the sister tool — identical API but applied to source files (`.sv`, `.svh`, `.v`, `.S`, `.py`) instead of log files. They are often called in pairs:

```
regex_log_search(rtl_sim.log, "lui.*x26")   → "Found: lui x26, 0x40001 at time 1856200"
regex_code_search(test.S, "lui.*x26")       → "Found: lui x26, 0x40001 at line 42 of test.S"
```

---

## 2. Why Not Just Read the File?

### The Problem with "Read the Log File"

A typical chip design simulation produces log files that are **enormous**:

| Simulation Type | Log File Size | Lines | Time to Read |
|----------------|--------------|-------|-------------|
| Unit test (small IP) | 10-100 MB | 100K-1M | 1-10 sec |
| Block-level regression | 1-10 GB | 10M-100M | 1-10 min |
| Full-chip regression | 50-500 GB | 500M-5B | 1-10 hours |
| Gate-level simulation | 100 GB - 1 TB | 1B-10B | 10+ hours |

Reading a 10GB log file into the LLM's context is:
1. **Impossible** — context windows are 200K-2M tokens, not 100M
2. **Expensive** — dumping the whole file would cost hundreds of dollars in API calls
3. **Useless** — 99.9% of lines are normal operation, not the failure

### What regex_log_search Does Instead

```
                    10 GB log file (100 million lines)
                    ─────────────────────────────────
                    
Full read:          [EVERY SINGLE LINE] → LLM
                    Context: exploded. Cost: $500. Value: zero.
                    
regex_log_search:   pattern: "UVM_FATAL|1856300|x26"
                    ─────────────────────────────────
                    ↓ ripgrep (0.5 sec, scans 10GB)
                    ↓
                    47 matching lines + 6 context lines each = ~330 lines
                    ─────────────────────────────────
                    → LLM
                    Context: ~8K tokens. Cost: $0.02. Value: failure located.
```

**The compression ratio:** 10GB → ~20KB of targeted results. That's 500,000:1 reduction while preserving 100% of the relevant information.

---

## 3. How Viv v0.2.3 Used It — Observed Behavior

From the 39 OCR'd frames of Viv v0.2.3 debugging an Ibex RISC-V cosimulation failure, I traced every `regex_log_search` and `regex_code_search` invocation. Here is the complete pattern evolution:

### Phase A: Initial Broad Search (t=2.00s, frame 0004)

The `deep_investigator` receives a natural language question:

> *"Summarize the failing test scenario and what the cosim mismatch at ibex_cosim_scoreboard.sv line 168 means. Include likely instruction context around time 1856300 from available logs."*

From this single question, the LLM derives:

```
regex_log_search(
    path: rtl_sim.log,
    pattern: Cosim mismatch|1856300|x26|commit|pc|instr,
    context: 6
)
```

**How was this pattern derived?** The LLM extracted these seeds from the question:
| Seed from question | Pattern element | Reasoning |
|-------------------|----------------|-----------|
| "cosim mismatch" | `Cosim mismatch` | The failure type |
| "time 1856300" | `1856300` | The time anchor |
| "ibex_cosim_scoreboard.sv line 168" | `x26`, `commit`, `pc`, `instr` | Inferred: scoreboard line 168 is a register mismatch → likely involves a register (x26), commit logic, PC, and instruction |

### Phase A: Refined Search (t=2.50s, frame 0005)

After the broad search returned results, the LLM constructed a more precise second pass:

```
regex_log_search(
    path: rtl_sim.log,
    pattern: 1856300|cosim|mismatch|UVM_FATAL|ibex_cosim_scoreboard|rvfi|spike|fatal,
    case_sensitive: false,
    context: 3
)
```

**What changed?**
- Added `UVM_FATAL` (learned from first results that the failure is a UVM fatal)
- Added `rvfi` (learned that RVFI trace interface is involved)
- Added `spike` (learned that Spike is the reference model)
- Added `fatal` (broader than UVM_FATAL)
- Reduced context from 6 to 3 (more targeted now)
- Made case-insensitive

### Phase A: Cross-File Correlation (t=3.00s, frame 0006)

From the first two searches, the LLM extracted specific values and constructed cross-file patterns:

```
# Search generator log for specific instruction encodings
regex_log_search(
    pattern: 80000080|1d37|4000d137|0000001d37|x26|s10,
    path: gen.log,
    context: 2,
    case_sensitive: false
)

# Search command output for test-level context
regex_log_search(
    pattern: riscv_arithmetic_basic_test|FAILED|cosim|trace_core|spike|mismatch|1856300|lui|x26,
    path: command.out,
    context: 3,
    case_sensitive: false
)
```

**What changed?**
- Now searching specific hex values: `80000080` (PC), `4000d137` (Spike's expected instruction), `0000001d37` (DUT's actual instruction)
- Added register alias: `s10` (x26 = s10 in RISC-V ABI)
- Added test name: `riscv_arithmetic_basic_test`
- Added `lui` (identified the failing instruction type)
- Cross-file: searching gen.log AND command.out for the same evidence

### Phase A: Code Search (t=4.00s, frame 0008)

```
# Search assembly test for the failing instruction
regex_code_search(
    path: test.S,
    pattern: \blui\s+x26,
    context: 2,
    case_sensitive: false
)

# Also search the log of the assembly test
regex_log_search(
    path: test.S,
    pattern: \blui\s+x26,
    context: 2,
    case_sensitive: false
)
```

**What changed?**
- Now using word-boundary anchors (`\b`) for precise matching
- Searching source code (test.S) with the same pattern as log files
- The pattern is now a specific instruction: `lui x26`

### Phase A: Scoreboard Code Search (t=4.50s, frame 0009)

```
regex_code_search(
    path: dv/uvm/core_ibex/common/ibex_cosim_agent/ibex_cosim_scoreboard.sv,
    pattern: get_cosim_error_str|riscv_cosim_step|run_cosim_rvfi|rvfi_instr,
    context: 2
)
```

**What changed?**
- Now searching the scoreboard implementation for specific function names
- Learned from earlier results: `riscv_cosim_step`, `rvfi_instr` are key functions

### Phase B: Per-Hypothesis Targeted Searches (t=8.00s, frame 0016)

In the Swarm phase, each hypothesis investigator uses patterns specific to its theory:

| Hypothesis | Likely search patterns | Why |
|-----------|----------------------|-----|
| #1 IF Bit-30 Corruption | `rvfi_insn|instr_rdata|if_stage|fetch|0x4000|bit.*30` | Looking for where bit 30 of the instruction gets corrupted |
| #3 Spurious Semantics | `spurious|inject.*fault|no.spurious|spurious.*enable` | Looking for spurious response configuration and behavior |
| #5 Cosim Step Misalignment | `riscv_cosim_step|sync|align|trace.*diverg|commit.*order` | Looking for trace/commit synchronization issues |

### The Complete Pattern Evolution Summary

```
Phase     Pattern Type              Example Pattern                           Source of Seeds
─────     ────────────              ───────────────                           ───────────────
A1        Broad failure cast        Cosim mismatch|1856300|x26|pc|instr      Natural language question
A2        Refined with learned      UVM_FATAL|rvfi|spike|fatal               Results from A1
A3        Specific values           80000080|4000d137|0000001d37|x26|s10     Results from A1+A2
A4        Instruction mnemonic      \blui\s+x26                              Results from A3
A5        Function names            get_cosim_error_str|riscv_cosim_step     Results from A1-A4
B1        Hypothesis #1 (RTL)       rvfi_insn|instr_rdata|bit.*30            Hypothesis description
B3        Hypothesis #3 (Spec)      spurious|inject|no-spurious              Hypothesis description
B5        Hypothesis #5 (TB)        riscv_cosim_step|sync|trace.*diverg      Hypothesis description
```

**Key insight:** Every single pattern was **dynamically derived**, not hardcoded. The LLM used the failure description, prior search results, and hypothesis statements as seeds to construct each pattern.

---

## 4. The Pattern Derivation Engine

### How the LLM Knows WHAT to Search For

The LLM doesn't magically know — it follows a structured reasoning chain. Here's the mental model:

```
                    ┌─────────────────────────────┐
                    │    FAILURE SIGNATURE         │
                    │  (from user or prior tool)   │
                    │  - assertion, time, register,│
                    │    PC, instruction, module   │
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────▼───────────────┐
                    │   SEED EXTRACTION (LLM)      │
                    │  "What tokens in this        │
                    │   failure signature could     │
                    │   appear in a log file?"      │
                    │                               │
                    │  → numeric values (time, PC)  │
                    │  → register names (x26, s10)  │
                    │  → error types (UVM_FATAL)    │
                    │  → signal names (rvfi_insn)   │
                    │  → module names (scoreboard)  │
                    │  → instruction mnemonics      │
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────▼───────────────┐
                    │   PATTERN CONSTRUCTION       │
                    │  pipe-separated OR clauses    │
                    │  "seed1|seed2|seed3|..."      │
                    │                               │
                    │  Three tiers:                 │
                    │  Tier 1: Exact values         │
                    │    (1856300, 80000080)        │
                    │  Tier 2: Semantic equivalents │
                    │    (UVM_FATAL, fatal, error)  │
                    │  Tier 3: Related concepts     │
                    │    (commit, pc, instr)        │
                    └─────────────┬───────────────┘
                                  │
                    ┌─────────────▼───────────────┐
                    │   EXECUTION & REFINEMENT     │
                    │  Run search → get results →   │
                    │  extract NEW seeds →          │
                    │  construct MORE PRECISE       │
                    │  pattern → repeat             │
                    └─────────────────────────────┘
```

### The Three-Tier Pattern Construction

Every pattern the LLM builds has three tiers of seeds, ordered from most specific to most general:

| Tier | Type | Example | Match Likelihood | False Positive Risk |
|------|------|---------|-----------------|-------------------|
| **Tier 1: Exact Values** | Numeric constants, specific strings | `1856300`, `80000080`, `0x40001d37` | Very high (appears in exactly the right places) | Very low |
| **Tier 2: Semantic Equivalents** | Domain synonyms for the same thing | `UVM_FATAL`, `fatal`, `FATAL` | High (different loggers use different formatting) | Low |
| **Tier 3: Related Concepts** | Broader context that might be nearby | `commit`, `pc`, `instr`, `rvfi` | Medium (contextual breadcrumbs) | Medium (might match unrelated lines) |

**Example construction:**

```
Failure: "Register write data mismatch to x26 at time 1856300, PC 0x80000080"

Tier 1: 1856300|0x80000080|x26|0x00001000|0x40001000
Tier 2: mismatch|UVM_FATAL|fatal|assertion
Tier 3: commit|pc|instr|rvfi|register.*write|scoreboard

Combined: 1856300|0x80000080|x26|mismatch|UVM_FATAL|commit|pc|instr
```

The LLM naturally prioritizes Tier 1 → Tier 2 → Tier 3, and can adjust the balance based on how many results come back. Too few results? Add more Tier 3 seeds. Too many? Drop Tier 3 and use only Tier 1+2.

### Regex Complexity: Keep It Simple

**Observed in Viv: patterns are always pipe-separated keyword alternatives, never complex regexes.**

Good patterns (used by Viv):
```
Cosim mismatch|1856300|x26|commit|pc|instr
\blui\s+x26
get_cosim_error_str|riscv_cosim_step|run_cosim_rvfi
```

Bad patterns (over-engineered, never observed):
```
(Cosim mismatch).*?(1856300).*?(x26)          ← too specific, misses variations
(?<=UVM_FATAL).*x26                           ← lookbehind, fragile
^.*(mismatch|fatal).*(x26|s10).*(0x[0-9a-f]+) ← over-constrained
```

**Why simple OR patterns work better:**
1. **Log formats vary** — different tools format the same information differently. `UVM_FATAL` vs `UVM_FATAL :` vs `[UVM_FATAL]` — a simple `UVM_FATAL` matches all of them.
2. **LLMs are bad at regex syntax** — they frequently produce invalid regex when trying to be clever.
3. **ripgrep is fast at OR patterns** — it compiles them into a DFA.
4. **The LLM does the semantic filtering** — get all lines containing ANY of the keywords, then let the LLM decide which are relevant.

---

## 5. Dynamic & Suggestive Design

### Dynamic: No Hardcoded Keywords

The `regex_log_search` tool itself has **zero knowledge** of RTL, UVM, Verilog, or any domain. It is a pure regex search engine. All intelligence lives in the LLM's pattern derivation.

The tool's signature is deliberately generic:

```python
def regex_log_search(
    path: str,              # File to search
    pattern: str,           # Regex pattern (LLM-generated)
    context: int = 3,       # Lines of surrounding context
    case_sensitive: bool = False,
    max_results: int = 100,  # Cap on match count
    max_chars: int = 50000   # Cap on output size (per-result limit)
) -> SearchResult
```

**The tool doesn't know what "UVM_FATAL" or "x26" means.** It just executes the regex. The LLM provides the intelligence.

### Suggestive: Guiding the Investigation Forward

After `regex_log_search` returns results, the tool (or the agent harness) should provide **suggestions** for next searches. This is what makes the tool "suggestive":

```json
{
  "matches": [...],
  "stats": {
    "files_searched": 1,
    "lines_scanned": 245000000,
    "matches_found": 47,
    "time_ms": 523
  },
  "suggestions": {
    "related_patterns": [
      "UVM_ERROR|UVM_WARNING|assertion",
      "1856[0-9]{3}",
      "x26|s10|x10"
    ],
    "related_files": [
      "rtl_sim_stdstreams.log",
      "trace_core_00000000.log",
      "spike_cosim_trace_core_00000000.log"
    ],
    "next_tools": [
      "log_fetcher — fetch surrounding lines of match #3 for more context",
      "code_fetcher — fetch ibex_cosim_scoreboard.sv around line 168",
      "wave_signals — query waveform signals at time 1856300"
    ]
  }
}
```

**How suggestions are generated:**

| Suggestion Type | Generation Method |
|----------------|------------------|
| **Related patterns** | Extract the most frequent tokens from match lines; suggest variations (e.g., if `UVM_FATAL` appears, suggest `UVM_ERROR`, `UVM_WARNING`) |
| **Related files** | If the search path was `rtl_sim.log`, suggest sibling files in the same directory (discovered via earlier `ls_tool` call) |
| **Next tools** | If matches contain line numbers and file references, suggest `code_fetcher` for those files. If matches contain timestamps, suggest `wave_signals`. |

The suggestions are **heuristic, not LLM-generated** — they're cheap to compute and give the LLM immediate options for the next step. The LLM can accept, modify, or ignore them.

---

## 6. Progressive Refinement Strategy

### The Full Pattern Refinement Lifecycle

```
Iteration 1: BROAD CAST
  Pattern: "Cosim mismatch|1856300|x26|commit|pc|instr"
  Purpose: Find ANY mention of the failure anywhere in the logs
  Expected results: 10-100 matches
  └──→ LLM reads results, extracts: exact PC, exact instruction, register

Iteration 2: REFINED SEARCH
  Pattern: "UVM_FATAL|rvfi|spike|1856300|x26"
  Purpose: Zoom in on the exact failure mechanism
  Expected results: 3-15 matches
  └──→ LLM reads results, extracts: function names, file paths, signal names

Iteration 3: SPECIFIC VALUES
  Pattern: "80000080|4000d137|0000001d37|get_cosim_error_str"
  Purpose: Find every occurrence of these exact values
  Expected results: 5-20 matches across multiple files
  └──→ LLM reads results, maps the data flow across files

Iteration 4: CROSS-FILE CORRELATION
  Pattern: (same pattern, different files)
  Files: gen.log, command.out, test.S, rtl_trace.csv
  Purpose: Trace the failure through the entire toolchain
  └──→ LLM now has enough evidence to form hypotheses

Iteration 5: HYPOTHESIS-TARGETED
  Per-investigator patterns specific to each hypothesis
  Investigator #1: "rvfi_insn|instr_rdata|bit.*30"
  Investigator #4: "spurious|inject.*fault|vif_spurious"
  └──→ Each investigator gathers evidence for/against their theory
```

### When to Stop Refining

The refinement loop stops when:
1. **Sufficient evidence** — the LLM has enough context to form or eliminate a hypothesis
2. **Diminishing returns** — the last search returned zero new matches
3. **Context budget reached** — the LLM has accumulated enough tokens from search results
4. **Time to pivot** — the evidence suggests a different tool would be better (waveform, code fetch, hierarchy)

---

## 7. Tool API Specification

### regex_log_search

```yaml
tool_name: regex_log_search
description: >
  Search simulation log files using a regex pattern. Returns matching lines
  with surrounding context. The pattern should be pipe-separated (|) keyword
  alternatives for best performance. This tool scans the file directly —
  it does NOT load it into memory. Use this instead of reading large log files.

parameters:
  path:
    type: string
    description: Path to the log file, relative to the artifacts directory
    required: true
    example: "rtl_sim.log"

  pattern:
    type: string
    description: >
      Regex pattern to search for. Prefer pipe-separated (|) keyword alternatives
      rather than complex regex. Example: "UVM_FATAL|mismatch|1856300|x26"
      Use \b for word boundaries when needed: "\blui\s+x26"
    required: true
    example: "Cosim mismatch|1856300|x26|commit|pc|instr"

  context:
    type: integer
    description: Number of lines of context to show before and after each match
    default: 3
    minimum: 0
    maximum: 20
    example: 6

  case_sensitive:
    type: boolean
    description: Whether the search is case-sensitive
    default: false
    example: false

  max_matches:
    type: integer
    description: Maximum number of matching lines to return
    default: 100
    maximum: 500

  max_chars:
    type: integer
    description: Maximum characters in the result (prevent context overflow)
    default: 50000
    maximum: 100000

returns:
  type: object
  properties:
    matches:
      type: array
      items:
        type: object
        properties:
          line_number: {type: integer}
          line_text: {type: string}
          context_before: {type: array, items: {type: string}}
          context_after: {type: array, items: {type: string}}
    stats:
      type: object
      properties:
        file_path: {type: string}
        file_size_bytes: {type: integer}
        lines_scanned: {type: integer}
        matches_found: {type: integer}
        matches_returned: {type: integer}
        time_ms: {type: integer}
        truncated: {type: boolean}
    suggestions:
      type: object
      properties:
        related_patterns: {type: array, items: {type: string}}
        related_files: {type: array, items: {type: string}}
        next_tools: {type: array, items: {type: string}}
```

### regex_code_search

```yaml
tool_name: regex_code_search
description: >
  Identical to regex_log_search, but searches source code files
  (.sv, .svh, .v, .S, .py) instead of log files. Use this to find
  signal declarations, always blocks, module definitions, and
  assertion locations in RTL source code.

parameters:
  # Same as regex_log_search, plus:
  file_types:
    type: array
    items: {type: string}
    description: File extensions to search (default: all known source types)
    default: [".sv", ".svh", ".v", ".vh", ".S", ".py", ".cpp", ".h"]

returns:
  # Same as regex_log_search
```

### Example: Full Invocation and Response

**Request:**
```json
{
  "tool": "regex_log_search",
  "arguments": {
    "path": "rtl_sim.log",
    "pattern": "Cosim mismatch|1856300|x26|commit|pc|instr",
    "context": 6,
    "case_sensitive": false
  }
}
```

**Response:**
```json
{
  "matches": [
    {
      "line_number": 48210,
      "line_text": "UVM_FATAL @ 1856300: Register write data mismatch to x26 DUT: 0x00001000 expected: 0x40001000",
      "context_before": [
        "[1856200] RVFI: commit PC=0x80000078 instr=0x...",
        "[1856250] RVFI: commit PC=0x8000007c instr=0x...",
        "[1856290] RVFI: order=5 insn=0x00001d37 rd=x26",
        "[1856295] Spike: expected insn=0x40001d37",
        "[1856298] Cosim: comparing x26",
        "[1856300] Cosim: MISMATCH DETECTED"
      ],
      "context_after": [
        "[1856300] ibex_cosim_scoreboard.sv:168: cosim mismatch",
        "[1856305] DUT x26=0x00001000, Spike x26=0x40001000",
        "[1856310] Test: riscv_arithmetic_basic_test FAILED",
        "[1856310] UVM_FATAL : simulation stopping",
        "",
        ""
      ]
    }
  ],
  "stats": {
    "file_path": "/build/sim_17_2026041217143230/logs/rtl_sim.log",
    "file_size_bytes": 2450000000,
    "lines_scanned": 52000000,
    "matches_found": 12,
    "matches_returned": 12,
    "time_ms": 523,
    "truncated": false
  },
  "suggestions": {
    "related_patterns": [
      "UVM_ERROR|UVM_WARNING|1856[0-9]{3}",
      "0x00001000|0x40001000",
      "ibex_cosim_scoreboard|rvfi_insn"
    ],
    "related_files": [
      "rtl_sim_stdstreams.log",
      "trace_core_00000000.log",
      "spike_cosim_trace_core_00000000.log"
    ],
    "next_tools": [
      "log_fetcher: fetch lines 48200-48220 of rtl_sim.log for full UVM report context",
      "code_fetcher: fetch ibex_cosim_scoreboard.sv around line 168",
      "wave_signals: query rvfi_if signals at time 1856200-1856400"
    ]
  }
}
```

---

## 8. Implementation Design

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    regex_log_search                      │
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────┐  │
│  │ Pattern  │    │  Search  │    │  Result Builder  │  │
│  │ Validator│───►│  Engine  │───►│  + Suggester     │  │
│  └──────────┘    └──────────┘    └──────────────────┘  │
│       │               │                    │            │
│       │          ┌────▼────┐               │            │
│       │          │ ripgrep │               │            │
│       │          │ (rg)    │               │            │
│       │          └─────────┘               │            │
│       │               │                    │            │
│  Validates:      Scans files          Formats output    │
│  - pattern is    in sub-ms using      - matches with    │
│    valid regex   multi-threaded        context          │
│  - path exists   search               - stats           │
│  - context in    - handles GB+        - suggestions     │
│    range         files natively       - truncation      │
└─────────────────────────────────────────────────────────┘
```

### Why ripgrep?

`ripgrep` (`rg`) is the ideal backend because:
- **Sub-millisecond search** on GB-scale files (multi-threaded, memory-mapped)
- **Respects .gitignore** (won't search build artifacts unless told to)
- **Built-in context lines** (`-C N`)
- **Built-in line numbers** (`-n`)
- **Built-in stats** (`--stats` for lines scanned, matches found, time)
- **Handles binary files** gracefully
- **Already installed** on most developer machines; easily installable via `apt`, `brew`, `cargo`

### Python Wrapper (pseudocode)

```python
import subprocess
import json
import re
import os
from pathlib import Path
from typing import Optional

class RegexLogSearch:
    """Dynamic pattern-driven log search tool."""
    
    MAX_CHARS = 50000      # Per-result character cap
    MAX_MATCHES = 100      # Default match limit
    CONTEXT_DEFAULT = 3
    
    def __init__(self, artifacts_dir: str):
        self.artifacts_dir = Path(artifacts_dir)
    
    def search(
        self,
        path: str,
        pattern: str,
        context: int = CONTEXT_DEFAULT,
        case_sensitive: bool = False,
        max_matches: int = MAX_MATCHES,
        max_chars: int = MAX_CHARS
    ) -> dict:
        """
        Execute a regex search over a log file.
        
        Args:
            path: Relative path within artifacts_dir
            pattern: Regex pattern (pipe-separated OR clauses preferred)
            context: Lines of context before/after each match
            case_sensitive: Case-sensitive search
            max_matches: Cap on number of matching lines
            max_chars: Cap on total output characters
        """
        # Resolve and validate path
        full_path = self._resolve_path(path)
        self._validate_pattern(pattern)
        
        # Build ripgrep command
        cmd = ["rg", "--json"]  # JSON output for parsing
        if not case_sensitive:
            cmd.append("-i")
        cmd.extend(["-C", str(context)])     # Context lines
        cmd.extend(["-m", str(max_matches)]) # Match limit
        cmd.extend(["-e", pattern])          # Pattern
        cmd.append(str(full_path))
        
        # Execute
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=30)
        
        # Parse ripgrep JSON output
        matches = self._parse_rg_output(result.stdout, max_chars)
        
        # Build suggestions
        suggestions = self._build_suggestions(
            path, pattern, matches, self.artifacts_dir
        )
        
        # Build stats
        stats = self._build_stats(full_path, result, matches)
        
        return {
            "matches": matches,
            "stats": stats,
            "suggestions": suggestions
        }
    
    def _resolve_path(self, path: str) -> Path:
        """Resolve and validate the file path."""
        full_path = self.artifacts_dir / path
        full_path = full_path.resolve()
        
        # Security: ensure path is within artifacts_dir
        if not str(full_path).startswith(str(self.artifacts_dir.resolve())):
            raise ValueError(f"Path escapes artifacts directory: {path}")
        
        if not full_path.exists():
            raise FileNotFoundError(f"File not found: {path}")
        
        return full_path
    
    def _validate_pattern(self, pattern: str):
        """Validate that the pattern is a valid regex."""
        try:
            re.compile(pattern)
        except re.error as e:
            raise ValueError(f"Invalid regex pattern: {e}")
    
    def _parse_rg_output(self, stdout: str, max_chars: int) -> list:
        """Parse ripgrep JSON output into structured matches."""
        matches = []
        total_chars = 0
        truncated = False
        
        for line in stdout.strip().split("\n"):
            if not line:
                continue
            try:
                entry = json.loads(line)
            except json.JSONDecodeError:
                continue
            
            if entry.get("type") == "match":
                match_data = entry["data"]
                match_text = match_data["lines"]["text"].rstrip("\n")
                
                # Build context
                context_before = []
                context_after = []
                # ... parse context from surrounding entries
                
                match_obj = {
                    "line_number": match_data["line_number"],
                    "line_text": match_text,
                    "context_before": context_before,
                    "context_after": context_after
                }
                
                # Check char limit
                match_chars = len(json.dumps(match_obj))
                if total_chars + match_chars > max_chars:
                    truncated = True
                    break
                
                matches.append(match_obj)
                total_chars += match_chars
        
        return matches
    
    def _build_suggestions(
        self, path: str, pattern: str, matches: list, artifacts_dir: Path
    ) -> dict:
        """Build heuristic suggestions for next search / next tool."""
        suggestions = {
            "related_patterns": [],
            "related_files": [],
            "next_tools": []
        }
        
        if not matches:
            suggestions["related_patterns"].append(
                "Try broadening the pattern: remove specific values, keep general terms"
            )
            return suggestions
        
        # Related patterns: extract frequent tokens from matches
        tokens = self._extract_frequent_tokens(matches)
        if tokens:
            suggestions["related_patterns"].append("|".join(tokens[:5]))
        
        # Related files: suggest sibling log files
        current_dir = (artifacts_dir / path).parent
        if current_dir.exists():
            siblings = [
                f.name for f in current_dir.iterdir()
                if f.is_file() and f.name != Path(path).name
                and f.suffix in ('.log', '.txt', '.csv', '.yaml', '.out')
            ]
            suggestions["related_files"] = siblings[:5]
        
        # Next tools: suggest based on match content
        match_texts = " ".join(m["line_text"] for m in matches[:10])
        if re.search(r'\d{7,}', match_texts):  # Timestamps
            suggestions["next_tools"].append(
                "wave_signals: query waveform signals near the timestamps found"
            )
        if re.search(r'\.sv:\d+|line \d+', match_texts):  # File references
            suggestions["next_tools"].append(
                "code_fetcher: fetch the referenced source files"
            )
        suggestions["next_tools"].append(
            "log_fetcher: fetch full context around key match lines"
        )
        
        return suggestions
    
    def _extract_frequent_tokens(self, matches: list) -> list:
        """Extract frequently occurring tokens from match lines."""
        from collections import Counter
        
        all_text = " ".join(m["line_text"] for m in matches)
        # Simple tokenization: split on whitespace and punctuation
        tokens = re.findall(r'[A-Za-z_][A-Za-z0-9_]*', all_text)
        # Filter out very common words
        stopwords = {'the', 'and', 'for', 'with', 'from', 'this', 'that'}
        tokens = [t for t in tokens if t.lower() not in stopwords and len(t) > 2]
        
        counter = Counter(tokens)
        # Return tokens that appear multiple times
        return [t for t, count in counter.most_common(10) if count > 1]
    
    def _build_stats(self, full_path: Path, result, matches: list) -> dict:
        """Build statistics about the search."""
        file_size = full_path.stat().st_size if full_path.exists() else 0
        
        return {
            "file_path": str(full_path),
            "file_size_bytes": file_size,
            "matches_found": len(matches),
            "matches_returned": len(matches),
            "time_ms": 0,  # Would parse from ripgrep stats
            "truncated": False
        }
```

### The Suggester: Heuristic, Not LLM

The suggestion engine uses **simple heuristics**, not LLM calls:

| Trigger | Suggestion |
|---------|-----------|
| Matches contain timestamps (7+ digit numbers) | Suggest `wave_signals` at those timestamps |
| Matches contain file references (`.sv:168`) | Suggest `code_fetcher` for those files |
| Matches contain register names (`x26`, `s10`) | Suggest searching for related registers |
| Match count > 50 (too many) | Suggest narrowing the pattern |
| Match count == 0 | Suggest broadening the pattern or checking a different file |
| Multiple files in same directory | Suggest searching those files with the same pattern |
| Pattern contains specific hex values | Suggest searching for variant representations (`0x40001` vs `40001` vs `0x00040001`) |

---

## 9. Integration with the Investigation Pipeline

### Position in the Tool Call Order

`regex_log_search` is **always the first tool called** because:

1. The failure signature originates in log files
2. Logs contain the time anchor that all other tools depend on
3. Logs contain file/line references that guide code searches
4. Logs contain signal names that guide waveform queries

```
Tool Call Order (Phase A — Initial Investigation):
─────────────────────────────────────────────────
1. regex_log_search(rtl_sim.log, broad_pattern)     ← FIRST: find the failure
2. regex_log_search(rtl_sim_stdstreams.log, same)   ← Cross-file same pattern
3. ls_tool(artifacts_dir)                            ← Discover available files
4. code_file_finder(**/scoreboard.sv)                ← Locate failing source
5. code_fetcher(scoreboard.sv, around line 168)      ← Read failing code
6. regex_log_search(rtl_sim.log, refined_pattern)    ← Refined second pass
7. log_fetcher(trace_core_00000000.log, 1, 200)      ← Read trace headers
8. regex_code_search(test.S, instruction_pattern)    ← Search test source
9. regex_code_search(scoreboard.sv, function_names)  ← Search scoreboard
10. hierarchy_descent(rvfi_if)                        ← Explore hierarchy
11. wave_signals(rvfi_if.signals, 1856200-1856400)   ← Query waveforms
```

### How regex_log_search Feeds Other Tools

```
regex_log_search finds:                           This enables:
───────────────────────                           ─────────────
"UVM_FATAL @ 1856300"                    ────────► wave_signals at time 1856200-1856400
"ibex_cosim_scoreboard.sv:168"           ────────► code_fetcher lines 150-190
"PC=0x80000080"                          ────────► regex_code_search for 0x80000080 in test.S
"x26 DUT:0x1000 expected:0x40001000"    ────────► regex_log_search for 0x00001000|0x40001000
"rvfi_insn=0x00001d37"                   ────────► regex_code_search for rvfi_insn in scoreboard
"ibex_mem_intf_response_seq_lib"         ────────► code_file_finder for the sequence library
```

### Per-Hypothesis Usage in the Swarm

In Phase B, each hypothesis investigator uses `regex_log_search` independently with patterns tailored to its specific theory:

```
Investigator #1 (RTL: IF Bit-30 Corruption):
  regex_log_search(rtl_sim.log, "rvfi_insn|instr_rdata|if_stage|fetch|0x4000")
  Purpose: Where does the instruction value enter the pipeline?

Investigator #3 (Spec: Spurious Semantics):
  regex_log_search(command.out, "spurious|inject|enable|no.spurious|spurious_response")
  Purpose: How is spurious injection configured?

Investigator #5 (TB: Cosim Step Misalignment):
  regex_log_search(rtl_trace.csv, "80000080|80000084|commit|retire")
  regex_log_search(spike_cosim_trace.log, "80000080|80000084|commit")
  Purpose: Align DUT and Spike commit traces at the divergence point
```

---

## 10. Resource & Performance Considerations

### Performance Profile

| Log File Size | ripgrep Scan Time | Typical Matches | Output Size |
|--------------|-------------------|-----------------|-------------|
| 10 MB | <10 ms | 5-20 | ~2 KB |
| 100 MB | ~50 ms | 10-50 | ~5 KB |
| 1 GB | ~500 ms | 20-100 | ~10 KB |
| 10 GB | 2-5 sec | 30-200 | ~20 KB |
| 50 GB | 10-30 sec | 50-500 | ~50 KB |

**Key insight:** Even for a 50GB log file, `regex_log_search` returns in under 30 seconds and produces ~50KB of output. Reading the entire file would take hours and produce gigabytes.

### Context Window Budget

A typical debug session with Viv used:
- 5-10 `regex_log_search` calls at ~5-20KB each = 25-200KB total
- Plus code fetches, waveform queries, hierarchy dumps
- Total: ~200-500KB of tool output across the entire investigation
- LLM context usage: 10-50K tokens for tool results vs 500K-5M tokens if reading entire files

### The 50K Character Cap

Observed in frame 0022: `hierarchy_descent: 51,686 chars -> 50,000 (per-result limit)`. The same cap applies to regex searches. This is critical for:
1. **Predictable context usage** — the LLM knows each result is ≤50K chars
2. **Fair resource allocation** — one runaway search can't consume the entire context window
3. **Encourages refinement** — if the cap is hit, the LLM must narrow its pattern

### ripgrep vs Python re: Why Not Pure Python?

| Approach | 1GB File Scan | 10GB File Scan | Memory Usage |
|----------|--------------|----------------|-------------|
| **ripgrep** | 0.5 sec | 5 sec | ~20 MB (mmap) |
| **Python `re`** (line-by-line) | 30 sec | 5 min | ~100 MB (buffered read) |
| **Python `mmap` + `re`** | 15 sec | 2.5 min | ~10 MB (mmap) |

ripgrep is 30-60x faster because it:
- Uses memory-mapped I/O (no copies)
- Is multi-threaded (scans file chunks in parallel)
- Uses SIMD-accelerated string matching (SSE4.2 / AVX2)
- Compiles regex to a DFA (not backtracking)

---

## Summary

`regex_log_search` is the **entry point** for all RTL debug investigations. Its design principles:

1. **Dynamic, not hardcoded** — every pattern is LLM-derived from failure context
2. **Progressive refinement** — broad → refined → specific → hypothesis-targeted
3. **Suggestive** — heuristic suggestions guide the next search without LLM overhead
4. **Context-efficient** — 500,000:1 compression ratio vs full file reads
5. **ripgrep-powered** — sub-second search over GB-scale files
6. **Simple patterns** — pipe-separated keywords, never complex regex
7. **Cross-file** — same pattern applied across logs, traces, command output, test files

---

## Sources

- `video_frames/investigation_report.md` — Full Viv v0.2.3 execution trace
- `ocr_output/frame_0004.txt` through `frame_0032.txt` — OCR'd tool invocations
- `ideas/revised_parllel_investigation.md` — Multi-hypothesis architecture
- `ideas/multi_hypothesis_rtl_debug_architecture.md` — Viv agent architecture
- [ripgrep](https://github.com/BurntSushi/ripgrep) — The search backend

**Next:** Implement `regex_log_search` as a Python MCP tool with ripgrep backend, pattern validator, and heuristic suggestion engine.
