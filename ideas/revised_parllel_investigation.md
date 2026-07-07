# Reverse-Engineered Multi-Hypothesis RTL Debugging Architecture

---

# Core Philosophy

Traditional AI debugging asks:

> What is the bug?

This architecture asks:

> What are the most plausible explanations, and how can we eliminate them systematically?

Instead of producing one answer immediately, it creates several competing theories and investigates them independently until evidence converges.

This closely mirrors how experienced RTL verification engineers debug difficult failures.

---

# High-Level Workflow

```text
                    Failing Test
                         │
                         ▼
              Failure Discovery Engine
                         │
     Collect enough information to characterize
             the failure, not solve it.
                         │
                         ▼
              Failure Signature Builder
                         │
        Structured Failure Representation
                         │
       ┌─────────────────┴────────────────┐
       │                                  │
  Execution Context                 Design Context
       │                                  │
       ▼                                  ▼
Logs  Waves  Traces               RTL  TB  Specs
       └─────────────────┬────────────────┘
                         ▼
               Hypothesis Generator
                         │
     Produce diverse competing explanations
                         │
 ┌──────────┬──────────┬──────────┬──────────┬──────────┐
 ▼          ▼          ▼          ▼          ▼
Hyp A     Hyp B      Hyp C      Hyp D      Hyp E
 │          │          │          │          │
 ▼          ▼          ▼          ▼          ▼
Independent Investigation Engines
 │
 ▼
Observe
Think
Choose Tool
Gather Evidence
Update Belief
Repeat
 │
 ▼
Structured Investigation Report
 └──────────────┬────────────────┘
                ▼
      Evidence Synthesis Engine
                │
 Merge facts
 Remove duplicates
 Resolve contradictions
 Build causal chain
                │
                ▼
     Root Cause + Engineering Report
```

---

# Stage 1 — Failure Discovery

## Objective

Do not debug.

Instead:

Build a structured description of the failure.

Typical outputs:

- failing assertion
- simulation timestamp
- failing register
- PC
- instruction
- failing module
- failing test
- relevant files

Think of this as opening a new bug ticket.

---

# Stage 2 — Hypothesis Generation

Instead of selecting one explanation:

Generate several intentionally different explanations.

Example:

Observed:

Register mismatch

Possible explanations:

- Instruction corruption
- Decoder bug
- Pipeline timing
- Scoreboard bug
- Reference model divergence
- Trace synchronization
- Testbench corruption

The objective is diversity.

Not correctness.

---

# Stage 3 — Parallel Investigation

Every hypothesis receives an independent investigator.

Each investigator assumes:

"My hypothesis is true."

Then repeatedly asks:

"What evidence would support this?"

"What evidence would disprove this?"

This naturally prevents confirmation bias because different investigators begin with different assumptions.

---

# Investigation Loop

```text
Current Hypothesis
        │
        ▼
Reason
        │
        ▼
Choose Tool
        │
        ▼
Collect Evidence
        │
        ▼
Update Confidence
        │
        ▼
Need More Evidence?
        │
  Yes ─────────────┐
        │          │
        └──────────┘
        │
        ▼
Produce Report
```

---

# Investigation Toolbox

Observed tools include:

- regex_log_search
- regex_code_search
- log_fetcher
- code_fetcher
- hierarchy_descent
- wave_signals
- sv_references
- sv_file_outline
- planner
- find_signals

These tools provide targeted retrieval rather than loading the entire repository.

---

# Investigation Report

Each investigator should produce structured findings rather than prose.

Example:

```yaml
hypothesis:

status:

confidence:

supporting_evidence:

contradicting_evidence:

remaining_unknowns:

root_cause:

recommended_next_step:
```

---

# Stage 4 — Evidence Synthesis

This stage should not simply choose the report with the highest confidence.

Instead:

Merge evidence.

Cluster reports describing the same cause.

Identify contradictions.

Determine which explanation best fits every observation.

The result is an engineering explanation rather than a majority vote.

---

# Why This Works

Suppose five investigators exist.

Four independently eliminate:

- Scoreboard bug
- Timing issue
- Trace alignment
- Reference model

One finds overwhelming evidence for instruction corruption.

Confidence comes from:

Independent agreement.

Not from one agent being more persuasive.

---

# Observed Characteristics

Confirmed:

✓ Initial investigation phase

✓ Parallel hypothesis investigation

✓ Tool-based evidence gathering

✓ Waveform-driven debugging

✓ Structured synthesis

Strong inference:

✓ Investigators update beliefs based on evidence

✓ Different investigators intentionally explore different causal paths

Unknown:

- Exact hypothesis generation algorithm

- Exact synthesizer ranking logic

- Internal report schema

---

# Proposed Improvements

(Not observed. Design ideas.)

## Evidence Graph

Normalize observations into reusable engineering facts.

Example:

PC

Instruction

Waveform

Register Write

Trace

Assertion

become connected nodes.

The synthesizer reasons over facts rather than free-form reports.

---

## Memory

Cache:

- previous investigations
- module summaries
- specification mappings
- waveform signatures

to avoid repeated work.

---

## Adaptive Investigator Allocation

Instead of always spawning five investigators:

Spawn additional investigators only when confidence remains low.

---

# Key Insight

The most valuable idea in this architecture is not multiple LLMs.

It is separating:

Discovery

↓

Hypothesis Formation

↓

Independent Evidence Gathering

↓

Evidence Synthesis

This is remarkably similar to how experienced RTL verification engineers debug difficult failures.