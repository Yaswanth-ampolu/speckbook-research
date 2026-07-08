# AGENTS.md — pdfresearch

## Project Overview

Research repository for building AI-powered chip design investigation tools:

1. **PDF-to-RTL pipeline** — Converting chip specification PDFs (ASA Motion Link SerDes spec) into structured data that can drive RTL module generation
2. **Multi-hypothesis RTL debugging** — Architecture for an AI agent that debugs RTL failures using parallel competing hypotheses, modeled after experienced verification engineers
3. **OCR/VLM benchmarking** — Evaluating models (GPT-5.1, GPT-5.4, Claude Opus 4.6, Claude Haiku 4.5, Mistral OCR) for extracting technical content from SerDes spec diagrams

## Repository Structure

```
ideas/              Architecture docs for RTL debug agent (multi-hypothesis, parallel investigation)
ocrresearch/        OCR/VLM model comparison results, benchmark data (JSON + Markdown)
pdfresearch/        8 research docs on PDF parsing, chunking, embedding, RAG best practices
images/             89 named figures + 127 inline images extracted from ASA Motion Link spec
```

## Repository Status

This is a research/documentation repository. There is no code currently. All content is research documents, benchmark data, and architecture specifications.

**Security Note:** Never use, commit, or reference AWS Bedrock credentials, API keys, or access tokens in this repository. All OCR/VLM benchmarking should use Azure endpoints or other approved services only.

## Document Conventions

### Markdown (Research Documents)

- Follow the existing format: `## Heading`, tables with `|---|` separators, code blocks with language tags
- Date-stamp all research documents: `**Date:** YYYY-MM-DD`
- Include keyword/topic in header: `**Keyword:** <search term>`
- End each document with `## Sources` and `**Next keyword:**` or `**Research Series Complete**`
- Use YAML blocks for structured data examples (hypothesis reports, investigation results)

### JSON (Benchmark Data)

- Use 2-space indentation
- Keys should be descriptive and consistent (e.g., `{Model}_{ImageType}`)
- Include units in key names where applicable (`elapsed` in seconds, `words` as count)

## Domain Conventions

### ASA Motion Link SerDes Terminology

Use standard SerDes/PHY terminology consistently:

- **Layers:** PMD, PMA, PCS, DLL (not "data layer" or "physical layer" generically)
- **Speed Grades:** SG1, SG2, SG3, SG4, SG5
- **FEC codes:** RS(108,106), RS(216,214), RS(240,214) — always include parameters
- **Signal naming:** Follow spec conventions (M_CSn, M_SCK, M_MOSI, M_MISO, d_plp_tx, tx_phy_block)
- **Key types:** BK (binding key), DK (device key), LK (link key)
- **Startup phases:** Phase1G, PhaseSGA, PhaseSGB, PhaseSGC

### OCR/VLM Prompt Conventions

- Always use the **SERDES_ARCHITECT** role-priming prompt for diagram analysis (2.5x more domain-accurate than generic prompts)
- Use the **Extended type-aware prompt** for timing waveforms, state machines, and frame formats
- Use the **Original 5-section prompt** for FEC/encoding diagrams and batch processing
- Pipeline routing by image type:
  - Timing/SPI waveforms → Claude Haiku 4.5
  - FEC/encoding → Claude Opus 4.6
  - Block/architecture → GPT-5.1
  - Security/key → GPT-5.1
  - State machines → Claude Haiku 4.5
  - Small inline → GPT-5.4

### RTL Debug Investigation Report Schema

All investigation reports must follow this YAML structure:

```yaml
hypothesis: <name>
status: supported | rejected | inconclusive
confidence: 0.0-1.0
supporting_evidence: [...]
contradicting_evidence: [...]
remaining_unknowns: [...]
root_cause: <description>
recommended_next_step: <action>
```

### PDF Pipeline Conventions

- Always preserve provenance metadata: document ID, page number, bounding box (x, y, w, h)
- Use layout-aware chunking (not fixed-size) for technical documents
- Chunk sizes: leaf 100-300 tokens, mid 200-500 tokens, parent = full section
- Overlap: 10-20% moderate overlap for boundary context
- Retrieval: hybrid (BM25 + Dense) + cross-encoder reranking
- Store hierarchical parent-child chunk relationships

## Key Research Findings (for context)

- Role-priming prompts are 2.5x more domain-accurate than generic description prompts
- Claude Haiku 4.5 achieved highest domain term density (16.8 avg) despite being the "small" model
- Docling is the strongest open-source PDF parser for technical documents
- Multimodal embeddings significantly outperform text-only for diagram-heavy specs
- Layout-preserving chunking is critical for technical specification RAG

## Files to Be Aware Of

- `ideas/multi_hypothesis_rtl_debug_architecture.md` — Core architecture for the RTL debug agent
- `ideas/revised_parllel_investigation.md` — Refined parallel investigation workflow
- `ocrresearch/comparison.md` — Full OCR/VLM benchmark results with recommendations
- `ocrresearch/bedrock-comparison.md` — Multi-model comparison including Bedrock models
- `pdfresearch/08-best-tools-for-pdf-to-structured-data-2026.md` — Final synthesis of PDF tooling recommendations
