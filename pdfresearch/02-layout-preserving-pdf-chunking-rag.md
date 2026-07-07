# Research: Layout Preserving PDF Chunking for RAG

**Date:** 2026-07-06
**Keyword:** layout preserving PDF chunking RAG

## Executive Summary

Layout-preserving chunking focuses on maintaining the **spatial and structural integrity** of PDFs during chunking. This includes preserving:

- Page numbers
- Bounding boxes
- Reading order
- Table structure
- Heading hierarchy
- Figure captions and layout entities

The research strongly recommends **hybrid approaches** that combine layout-aware parsing with semantic refinement, rather than pure semantic or pure fixed-size chunking.

## Key Recommendations

### 1. Extraction Tools
- **Primary**: PyMuPDF or pdfplumber for layout + bounding boxes
- **Table extraction**: Camelot (lattice/stream modes)
- **OCR fallback**: Tesseract or EasyOCR for scanned documents
- **Advanced**: Learning-based layout models (LayoutParser, vision LMs like Donut)

### 2. Chunking Strategy (Recommended)
- **Default baseline**: Page-level chunks (surprisingly effective and stable for citation)
- **Refinement**: Split further by logical layout blocks (headings, columns, tables, figures)
- **Semantic layer**: Apply semantic clustering only after preserving layout structure
- **Overlap**: Use moderate, configurable overlap (especially for analytical content)

### 3. Metadata to Preserve (Critical for Lossless RAG)
- Document ID + page number
- Bounding box coordinates (x, y, w, h)
- Section/heading hierarchy
- Original raw text + normalized text
- OCR engine + confidence score
- Chunk generation method
- Deterministic chunk ID + checksum

## Important Insights

- Converting PDFs directly to Markdown is often **lossy** for spatial metadata.
- Tables that span multiple pages should be treated as single logical units.
- Layout-aware chunking significantly improves retrieval on scientific papers, reports, and multi-column documents.
- Provenance metadata (page + bbox) enables much better citation and grounding in the final LLM response.

## Recommended Pipeline

```
PDF → Layout-aware Parser (PyMuPDF/pdfplumber) 
    → OCR fallback (if scanned) 
    → Hybrid Chunking (Page + Layout blocks + Semantic) 
    → Rich Metadata Storage 
    → Embeddings 
    → Vector DB
```

## Evidence Gaps Identified
- Lack of standardized end-to-end benchmarks linking chunking parameters directly to QA accuracy.
- Limited public numbers on latency/storage cost trade-offs at scale.

## Sources
- LanceDB blog on LiteParse + layout-aware chunking
- PyMuPDF documentation
- pdfplumber GitHub
- arXiv papers on PDF structure recognition
- LlamaIndex chunking strategies guide
- Layout-aware RAG demo (lumozai)

---

**Next**: We can continue to the next keyword.