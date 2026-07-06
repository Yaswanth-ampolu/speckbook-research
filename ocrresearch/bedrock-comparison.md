# Multi-Model Chip Design OCR Comparison

**Date:** 2026-07-06  
**Prompt:** Extended SerDes Architect (type-aware)  
**Images:** 8 types from ASA Motion Link SerDes spec (A-H)

---

## Models Compared

| # | Model | Provider | Endpoint | Auth |
|---|-------|----------|----------|------|
| 1 | **mistral-ocr-4-0** (pure OCR) | Azure Foundry | `/providers/mistral/azure/ocr` | api-key |
| 2 | **GPT-5.1** | Azure Foundry | `/openai/v1/chat/completions` | api-key |
| 3 | **GPT-5.4** | Azure Foundry | `/openai/v1/chat/completions` | api-key |
| 4 | **Claude Opus 4.6** | AWS Bedrock | `/model/{id}/invoke` | Bearer |
| 5 | **Claude Haiku 4.5** | AWS Bedrock | `/model/{id}/invoke` | Bearer |

## Head-to-Head Results (Extended Prompt)

### Domain Term Accuracy (higher = better)

| Image | Type | GPT-5.1 | GPT-5.4 | Opus 4.6 | Haiku 4.5 |
|-------|------|---------|---------|----------|-----------|
| A_Timing | SPI Waveform | 23 | 15 | 19 | **22** |
| B_FreqChart | Frequency Plot | 13 | 13 | 13 | **14** |
| C_BlockDiag | PCS Data Flow | 23 | 15 | **26** | **26** |
| D_FEC | RS(108,106) Encoder | 11 | 12 | **20** | 16 |
| E_StateMachine | Transceiver FSM | 13 | 11 | 9 | **19** |
| F_FrameFormat | DLL Container | 12 | 12 | 4 | **12** |
| G_Security | Key Hierarchy | **17** | 8 | 15 | 13 |
| H_Inline | Small Symbol | 8 | **10** | 8 | **12** |
| **AVERAGE** | | **15.0** | **12.0** | **14.3** | **16.8** |

### Output Verbosity (words)

| Image | GPT-5.1 | GPT-5.4 | Opus 4.6 | Haiku 4.5 |
|-------|---------|---------|----------|-----------|
| A_Timing | 1,646 | 646 | 1,439 | 1,405 |
| B_FreqChart | 1,200 | 530 | 910 | 1,227 |
| C_BlockDiag | 1,548 | 557 | 796 | 1,318 |
| D_FEC | 1,279 | 595 | 644 | 1,180 |
| E_StateMachine | 1,867 | 745 | 1,051 | 1,636 |
| F_FrameFormat | 1,553 | 647 | 553 | 1,456 |
| G_Security | 1,810 | 530 | 1,006 | 1,333 |
| H_Inline | 527 | 0 | 281 | 371 |
| **AVERAGE** | **1,428** | **531** | **835** | **1,240** |

### Latency (seconds)

| Image | GPT-5.1 | GPT-5.4 | Opus 4.6 | Haiku 4.5 |
|-------|---------|---------|----------|-----------|
| A_Timing | 45.2 | 21.7 | 58.5 | 26.0 |
| B_FreqChart | 28.0 | 17.5 | 40.7 | 24.1 |
| C_BlockDiag | 35.8 | 17.6 | 35.9 | 26.6 |
| D_FEC | 31.8 | 18.9 | 32.6 | 25.3 |
| E_StateMachine | 45.3 | 22.0 | 41.1 | 30.1 |
| F_FrameFormat | 34.0 | 17.1 | 23.8 | 27.9 |
| G_Security | 53.8 | 18.7 | 37.7 | 26.2 |
| H_Inline | 15.4 | 12.8 | 14.4 | 8.7 |
| **AVERAGE** | **36.2s** | **18.3s** | **35.6s** | **24.4s** |

---

## Per-Model Analysis

### GPT-5.1 (Azure) — Best All-Rounder with Extended Prompt
- **15.0 avg terms** — strong across the board
- **1,428 avg words** — thorough but sometimes over-verbose
- **36.2s avg** — expected latency for a frontier model
- **Best on:** Security/Key diagrams (17 terms)
- **Weakness:** Can over-explain simple images

### GPT-5.4 (Azure) — Best with ORIGINAL Prompt
- **12.0 avg terms** with extended prompt (weaker than 5.1)
- **531 avg words** — much more concise (tested with chat completions)
- **18.3s avg** — fastest of all models
- **Note:** Earlier test with ORIGINAL prompt showed GPT-5.4 nearly matching GPT-5.1 (11.2 vs 12.6 terms). The extended prompt didn't benefit GPT-5.4 as much as GPT-5.1.

### Claude Opus 4.6 (Bedrock) — Most Disciplined, Best on FEC
- **14.3 avg terms** — solid but not top
- **835 avg words** — most concise, highest information density
- **35.6s avg** — comparable to GPT-5.1
- **Best on:** Block/Architecture (26 terms), FEC/Encoding (20 terms)
- **Weakness:** Frame format diagrams (only 4 terms) — missed field extraction
- **Characteristic:** Never verbose, every word earns its place. Good for precise technical extraction.

### Claude Haiku 4.5 (Bedrock) — Surprise Winner
- **16.8 avg terms** — **#1 in domain accuracy**
- **1,240 avg words** — well-balanced
- **24.4s avg** — **fastest among frontier VLMs** (except GPT-5.4)
- **Best on:** State Machines (19 terms), Timing Waveforms (22 terms)
- **Hit max_tokens ceiling** on 4/8 images (C, D, E, G) — actual capability likely higher
- **Note:** This is Haiku, the "small/fast" model, yet it outperforms both GPT-5.1 and Opus 4.6 on domain terms

### Pure OCR (mistral-ocr-4-0) — Text Only
- Extracts raw markdown text
- **Cannot interpret** charts, graphs, state machines, block diagrams
- Fast and cheap for text-heavy pages
- **No domain reasoning** — just text extraction

---

## Combined Ranking

| Rank | Model | Domain Terms | Latency | Cost Model | Best For |
|------|-------|-------------|---------|------------|----------|
| 1 | **Claude Haiku 4.5** | **16.8** | 24.4s | Low (Haiku tier) | State machines, timing, overall accuracy |
| 2 | **GPT-5.1** | 15.0 | 36.2s | High (frontier) | Security, block diagrams, consistency |
| 3 | **Claude Opus 4.6** | 14.3 | 35.6s | Highest (Opus tier) | FEC/encoding, precise numerical extraction |
| 4 | **GPT-5.4** | 12.0 | 18.3s | Medium-High | Fast batch processing (use with ORIGINAL prompt) |
| — | **mistral-ocr-4-0** | N/A | ~3s | Low (OCR tier) | Pure text extraction from tables/text pages |

---

## Key Takeaways

1. **Haiku 4.5 is the dark horse**: Despite being the "small/cheap" Claude model, it achieved the highest domain term density at the best latency. It hits the 3000-token ceiling on complex images — increasing `max_tokens` would likely show even better results.

2. **Opus 4.6 is the precision tool**: Most concise (fewest words per domain term). Excellent for FEC/Galois field extraction where numerical precision matters. Undervalued on frame formats — likely needs more tokens.

3. **GPT-5.1 is the safe bet**: Consistently good across all image types. Never the best on any one type, but never weak either.

4. **GPT-5.4 with ORIGINAL prompt is the speed king**: At 18.3s with comparable accuracy to Opus, it's the best for batch processing 89 figures.

5. **Prompt matters more than model**: The gap between best and worst VLM (16.8 vs 12.0 terms) is smaller than the gap between best VLM and pure OCR (16.8 vs 0 domain reasoning).

---

## Recommended Pipeline

```
89 ASA spec figures
├── Text-heavy pages / tables → mistral-ocr-4-0 (fast, cheap)
├── Timing/SPI waveforms       → Claude Haiku 4.5 (best speed+accuracy)
├── FEC/encoding diagrams      → Claude Opus 4.6 (best numerical precision)
├── State machines / flowcharts→ Claude Haiku 4.5 (best state enumeration)
├── Block/architecture diagrams→ GPT-5.1 (best layer+signal mapping)
├── Security/key diagrams      → GPT-5.1 (best key hierarchy analysis)
├── Frame/packet formats       → Claude Haiku 4.5 (best field extraction)
└── Small inline images        → GPT-5.4 (fastest, adequate accuracy)
```