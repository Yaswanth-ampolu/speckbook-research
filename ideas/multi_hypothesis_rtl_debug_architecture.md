# Reconstructed Architecture of a Multi-Hypothesis RTL Debugging Agent

> **Note**
>
> This document is a reconstruction based on publicly available
> documentation, observed UI screenshots, and standard RTL debugging
> practices. It is **not** an official description of Viv's internal
> implementation.

------------------------------------------------------------------------

# High-Level Architecture

``` text
                             Failing Test
                                  │
             ┌────────────────────┴────────────────────┐
             │                                         │
      Load Artifacts                           Load Code / Specs
 (logs, traces, waves, configs)         (RTL, TB, docs, hierarchy)
             │                                         │
             └────────────────────┬────────────────────┘
                                  │
                                  ▼
                     Initial Evidence Extraction
          (failure signature, PC, instruction, timestamp,
             assertion, mismatch, failing component)
                                  │
                                  ▼
                        Hypothesis Generator
         "What are the most plausible explanations?"
                                  │
        Generates 3–5 competing, intentionally diverse hypotheses
                                  │
     ┌─────────────┬──────────────┬──────────────┬──────────────┐
     ▼             ▼              ▼              ▼              ▼
 RTL Investigator  TB Investigator Trace Investigator Timing Investigator ...
     │             │              │              │
     │         Independent reasoning loops       │
     ▼             ▼              ▼              ▼
  Decide next evidence required for THIS hypothesis
     │
 ┌───┼───────────────┬──────────────┬──────────────┬─────────────┐
 ▼   ▼               ▼              ▼              ▼             ▼
Regex Search   Code Fetcher   Log Fetcher   Wave Server   Hierarchy   Spec Lookup
                               (VCD/FST/FSDB)
 └──────────────┬──────────────┴──────────────┬──────────────┘
                ▼                             ▼
          Structured Evidence         Contradicting Evidence
                │
                ▼
      Refine / Reject / Strengthen Hypothesis
                │
          Repeat until sufficient confidence
                │
                ▼
          Structured Investigation Report
     └──────────────────┬──────────────────┘
                         ▼
              Synthesizer / Evidence Merger
                         │
      Cluster similar reports, compare evidence,
      eliminate contradictions, rank explanations
                         ▼
         Root Cause + Evidence + Fix Narrative
```

------------------------------------------------------------------------

# Inspiration from a Real Chip Design Debug Flow

An experienced verification engineer rarely jumps to "the bug."

Instead, they naturally form multiple mental models.

Given:

    PC: 0x80000080
    RTL retired:   0x00001d37
    Spike expects: 0x40001d37

Possible explanations include:

-   Instruction memory corruption
-   Fetch alignment issue
-   Decoder bug
-   Pipeline corruption
-   Scoreboard bug
-   Reference model issue
-   Trace synchronization issue
-   Testbench sampling error

Rather than committing early, the engineer tests several explanations
until evidence rules them in or out.

The architecture above mirrors that process.

------------------------------------------------------------------------

# What Each Investigator Does

Each investigator starts with a different working assumption.

Examples:

-   **RTL Investigator**
    -   "Assume the RTL produced the wrong instruction."
-   **Trace Investigator**
    -   "Assume RTL is correct but trace capture is wrong."
-   **Testbench Investigator**
    -   "Assume the scoreboard sampled the wrong cycle."
-   **Timing Investigator**
    -   "Assume valid/ready timing caused the mismatch."

Each investigator asks one question repeatedly:

> **"Can I prove or disprove my current hypothesis?"**

This bias is intentional. It encourages broad exploration rather than
one agent chasing its first intuition.

------------------------------------------------------------------------

# Evidence Collection Loop

Every investigator follows a ReAct-style loop:

``` text
Observe failure
      │
      ▼
Think
      │
      ▼
Choose next tool
      │
      ▼
Collect evidence
      │
      ▼
Update belief
      │
      ▼
Need more evidence?
      │
   Yes ─────────────┐
      │             │
      ▼             │
Choose another tool │
      │             │
      └─────────────┘
      │
      ▼
Produce report
```

------------------------------------------------------------------------

# Example Structured Report

``` yaml
hypothesis: Instruction Fetch Corruption

status: supported

confidence: 0.91

supporting_evidence:
  - PC 0x80000080
  - rvfi_insn = 0x00001d37
  - Spike expected 0x40001d37
  - Bit 30 differs
  - Waveform confirms DUT value

contradicting_evidence:
  - Decoder output matches fetched instruction
  - No scoreboard mismatch before retirement

remaining_unknowns:
  - Cause of bit flip before decode

root_cause:
  Instruction appears corrupted before entering decode.

recommended_next_step:
  Inspect instruction memory interface and fetch path.
```

------------------------------------------------------------------------

# What the Synthesizer Should Do

Rather than "vote" on reports, a robust synthesizer should:

1.  Merge all reports.
2.  Cluster reports describing the same underlying cause.
3.  Measure agreement across independent investigators.
4.  Weigh evidence quality, not only confidence scores.
5.  Highlight contradictions.
6.  Identify remaining unknowns.
7.  Produce a single engineering narrative explaining:
    -   why one hypothesis is preferred,
    -   which evidence supports it,
    -   which competing hypotheses were rejected,
    -   and what should be investigated next.

This resembles a design review meeting more than a majority vote.

------------------------------------------------------------------------

# Why Multi-Hypothesis Investigation Matters

A single investigator can fall into confirmation bias:

    "I think it's a fetch bug."

    ↓

    Everything searched reinforces that assumption.

Independent investigators reduce that risk.

When different investigators, following different assumptions, converge
on the same conclusion, confidence comes from **independent agreement**,
not repetition.

This is very similar to how experienced hardware teams converge on root
cause during silicon bring-up or verification debugging.
