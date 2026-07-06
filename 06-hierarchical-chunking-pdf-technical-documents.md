# Research: Hierarchical Chunking for Technical PDFs

**Date:** 2026-07-06
**Keyword:** hierarchical chunking PDF technical documents

## Executive Summary

Hierarchical chunking is currently one of the strongest approaches for technical documents (specifications, manuals, research papers, reports). It creates a **multi-level structure** (Document → Section → Subsection → Paragraph → Atomic element) while preserving relationships between text, tables, figures, and equations.

This approach significantly outperforms flat chunking for complex technical content.

## Key Principles of Hierarchical Chunking

### 1. Multi-Level Structure
- **Parent chunks**: Larger context (e.g., full section or page)
- **Child chunks**: Fine-grained retrieval units (paragraphs, tables, figures)
- Maintain explicit **parent → child** relationships for reassembly during generation.

### 2. Boundary Detection Signals
Effective hierarchical chunking combines multiple signals:
- Visual/layout cues (font size, whitespace, columns)
- Structural cues (TOC, headings, numbering)
- Linguistic cues (discourse markers, sentence boundaries)
- Visual instance segmentation (for figures/tables)

### 3. Handling Different PDF Types
- **Born-digital PDFs**: Use layout parsers (pdfplumber, PyMuPDF) for speed and structure.
- **Scanned PDFs**: Require OCR + vision-language models.
- **Mixed PDFs**: Combine native text with OCR output.

### 4. Chunk Sizing Recommendations
- Leaf level: 100–300 tokens (for precision)
- Mid-level: 200–500 tokens
- Parent level: Larger sections for context reconstruction
- Use **moderate overlap** (10–20%) to preserve boundary context.

## Recommended Metadata to Store

For every chunk:
- Document ID + hierarchical ID
- Page number(s) and character/byte offsets
- Bounding box coordinates
- Element type (paragraph, table, figure, equation, code, etc.)
- Parent ID (for hierarchical linking)
- OCR confidence (if applicable)
- Provenance information

## Retrieval Strategy

Best practice for hierarchical chunking:
- Index both **parent** and **child** chunks.
- Use **hybrid retrieval** (BM25 + Dense).
- Apply **reranking** on top candidates.
- At generation time: Retrieve fine-grained children + their parent context.

This balances precision (small chunks) with coherence (larger context).

## Tools & Approaches Mentioned

- **Layout parsers**: pdfplumber, PyMuPDF, pdfalto (for ALTO XML)
- **Hierarchical models**: HiPS, MultiDocFusion, H-RAG
- **Vector stores**: Qdrant, Milvus, Weaviate, Pinecone
- **Metadata standards**: METS/ALTO for preserving structure

## Best Practices

- Treat tables/figures that span pages as single logical units.
- Maintain bidirectional parent-child mappings.
- Use deterministic, hierarchical identifiers.
- Combine layout signals with semantic signals for boundary detection.
- Evaluate using both retrieval metrics (R@k) and downstream QA accuracy.

## Evidence Gaps
- Lack of standardized benchmarks specifically for hierarchical chunking on technical documents.
- Limited public data on optimal parent/child chunk size ratios across domains.

## Sources
- arXiv papers on MultiDocFusion and HiPS
- Firecrawl chunking strategies guide
- Various vector database comparison articles
- Layout-aware RAG research

---

**Next keyword:** `OCR + layout PDF embedding pipeline`