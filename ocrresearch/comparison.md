# OCR Model Comparison: ASA Motion Link SerDes Chip Design Documents

**Date:** 2026-07-06  
**Endpoint:** `https://yaswanth-resource.services.ai.azure.com`  
**Source PDF:** ASA Technical Specification v2.0 (SerDes / high-speed serial link)

---

## Models Tested

| Model | Type | Deployment | Auth |
|-------|------|------------|------|
| `mistral-ocr-4-0` | Pure OCR (text extraction) | GlobalStandard serverless | `api-key` |
| `gpt-5.1` | Vision-Language Model | OpenAI-compatible | `api-key` |
| `gpt-5.4` | Vision-Language Model | OpenAI-compatible + Responses API | `api-key` |

---

## Endpoint Paths (Critical — this was the bug)

| Model | Correct URL Path | Wrong Paths Tried |
|-------|-----------------|-------------------|
| `mistral-ocr-4-0` | `/providers/mistral/azure/ocr` | `/v1/ocr`, `/ocr`, `/chat/completions`, `/models/ocr` |
| `gpt-5.1` / `gpt-5.4` | `/openai/v1/chat/completions` | N/A (standard path) |
| `gpt-5.4` (Responses API) | `/openai/v1/responses` | N/A |

**Key finding:** Mistral OCR on Azure Foundry uses a non-obvious path (`/providers/mistral/azure/ocr`), not the standard `/v1/ocr` or `/models/ocr`. This was the root cause of the user's original 404 errors.

---

## Image Types Tested (from ASA spec)

| Type | Example | Content |
|------|---------|---------|
| **Waveform/Timing** | `05_SPI_extended_timing_multi_frame.png` | SPI multi-frame timing diagram with signal traces, numbered annotations |
| **Block Diagram** | `34_Figure_2-1_ASA_layers_and_interfaces.png` | ASA protocol stack layers (PCS/PMA/DLL) |
| **PCS Data Flow** | `36_Figure_4-1_Normal_Mode_PCS_transmit_data_flow.png` | Transmit pipeline: scrambler → FEC → PAM mapping |
| **RS-FEC Encoder** | `44_Figure_4-9_RS_108_106_parity_generation.png` | Reed-Solomon LFSR with GF(256) arithmetic |
| **RS-FEC Encoder** | `45_Figure_4-10_RS_216_214_parity_generation.png` | Another RS code variant |
| **State Machine** | `33_Figure_2-3_ASA_transceiver_state_diagram.png` | 9-state transceiver FSM with transitions |
| **Security Keys** | `68_Figure_6-1_Cryptographic_Key_Overview.png` | BK/DK/LK key hierarchy diagram |
| **Frequency Chart** | `23_Speed_Grade_4_frequency.png` | Frequency domain graph (0-6000 MHz, dBm/Hz) |
| **Small Icon** | `img-140.png` | Inline numeric label ("3.6") |

---

## Prompt Strategies Evaluated

### A. SERDES_ARCHITECT (Best — domain role priming)
```
You are a senior SerDes / high-speed serial link architect reviewing
an ASA Motion Link technical specification. Analyze this diagram:
1. Domain Context (PCS, PMA, DLL, PHY, Security, OAM)
2. Technical Breakdown (data flow, encoding, states, parameters)
3. Numerical Extraction (bit widths, ratios, clock rates, polynomials, register sizes)
4. Design Intent (why this block exists in a SerDes)
5. Potential Issues (unusual design choices)
Be concise, use standard SerDes/PHY terminology.
```

### B. BASIC_DESCRIBE (Control baseline)
```
Describe what you see in this image in detail.
```

### C. STRUCTURED_EXTRACT (Extraction-only, no interpretation)
```
Extract from this technical diagram:
- All text labels, headings, and captions
- All numerical values, bit widths, and parameters
- All signal names, block names, and state names
- The data flow direction and connections between blocks
- Any table data or register definitions
Format the output as structured bullet points under clear headings.
```

---

## Results Summary

### 1. Plain OCR (mistral-ocr-4-0) vs VLMs

| Capability | mistral-ocr-4-0 | gpt-5.1 / gpt-5.4 |
|------------|-----------------|-------------------|
| Text extraction | Good | Good |
| Table extraction | Markdown tables | Markdown tables + interpretation |
| Graphical interpretation | **None** | **Excellent** |
| Frequency chart reading | Only title text extracted | Full axes, ticks, labels, values |
| Domain reasoning | None | Identifies PCS/PMA/DLL layering |
| Design intent | None | Explains WHY each block exists |
| Numerical extraction | Raw text from image | Parsed with context + meaning |
| Output format | Raw markdown | Structured analysis with sections |

**Example: Frequency Chart (`23_Speed_Grade_4_frequency.png`)**

- **OCR output:** `"Speed Grade 4"` (only the title — missed all axes/graph data)
- **GPT-5.1 output:** Read x-axis (Frequency 0-6000 MHz), y-axis (dBm/Hz -88 to -106), all tick marks, grid, and identified it as a Cartesian line graph

### 2. GPT-5.1 vs GPT-5.4 (with SERDES_ARCHITECT prompt)

| Metric | gpt-5.1 | gpt-5.4 |
|--------|---------|---------|
| Avg words per image | 610 | 632 |
| Avg domain terms hit | **12.6 / 25** | 11.2 / 25 |
| Avg latency | **18.1s** | 19.5s |
| Best single score | 18 terms (PCS data flow) | 15 terms (PCS data flow) |
| Consistency | Very consistent | Slightly more variable |

**Verdict:** gpt-5.1 has a marginal edge (+1.4 terms, -1.4s latency). Functionally equivalent.

### 3. Prompt Strategy Impact (gpt-5.1)

| Strategy | Avg Words | Avg Domain Terms | Quality |
|----------|-----------|-----------------|---------|
| **SERDES_ARCHITECT** | 681 | **10 / 10** | Structured 5-section analysis, design reasoning |
| BASIC_DESCRIBE | 569 | 4 / 10 | Good descriptions but generic, no domain depth |
| STRUCTURED_EXTRACT | 377 | 4 / 10 | Lists labels only, zero interpretation |

**The role-priming prompt (SERDES_ARCHITECT) is 2.5x more domain-accurate** than a generic description prompt, despite using the same model.

### 4. Per-Image Deep Dive (SERDES_ARCHITECT with gpt-5.1)

| Image | Key Extractions | Accuracy |
|-------|----------------|----------|
| PCS TX Data Flow | Full pipeline (DLL→scrambler→RS-FEC→resync→PAM2/PAM4→PMA), PTB clock, S0/S1 seeds, down/up separation | **Flawless** |
| RS(108,106) FEC | Generator polynomial, GF primitive `x⁸+x⁴+x³+x²+1`, g0=0x02/g1=0x03/g2=0x01, LFSR data flow, systematic codeword ordering | **Flawless** |
| RS(216,214) FEC | Same precision, correctly distinguished from RS(108,106), 214 message / 2 parity symbols | **Flawless** |
| Transceiver FSM | All 9 states (Power Off→Init→Startup→OAM Config→Normal/Test/Light Sleep/Deep Sleep/Fail), transitions, link training flow | **Flawless** |
| Crypto Keys | BK/DK/LK hierarchy, UUID-based derivation, Root ECU→Device distribution, link key pairing per node | **Flawless** |

---

## Recommendations

### For text-only extraction (tables, text-heavy pages)
- Use **mistral-ocr-4-0** — faster, cheaper, purpose-built

### For diagrams requiring interpretation
- Use **gpt-5.1** with the **SERDES_ARCHITECT** role prompt
- gpt-5.4 is an acceptable substitute if cost/latency differs

### For frequency/waveform charts
- Must use a **VLM** (GPT-5.1/5.4) — OCR alone cannot read graphical axes
- Add a chart-specific prompt: "Read all axis labels, tick values, and describe the waveform envelope"

### Prompt best practices
1. Always assign a **domain role** (SerDes architect, PHY engineer, etc.)
2. Request **structured sections** (context, breakdown, numbers, intent, issues)
3. Specify terminology expectations ("use standard SerDes/PHY terminology")
4. For chart images, explicitly ask for axis/scale/tick extraction

---

## Technical Notes

- Auth header for `services.ai.azure.com`: `api-key` (not `Bearer`)
- gpt-5.x `max_tokens` → must use `max_completion_tokens`
- OCR payload uses `document` object, not `messages`
- VLM payload uses standard `messages` with `image_url` content blocks
- Mistral OCR on Azure: discovered via liteLLM source code analysis (`/providers/mistral/azure/ocr`)

---

## Image Inventory (89 named figures, 127 inline images)

### By Document Chapter

| Chapter | Topic | Figures |
|---------|-------|---------|
| **2** | ASA Overview & Transceiver | 3 (layers, half-duplex overview, transceiver FSM) |
| **3** | SPI/GPI/I2S/PTB Protocols | 15 (SPI×7, GPI×2, PTB×3, I2S×1, misc×2) |
| **4** | PCS / PHY / Startup | 32 (data flow, FEC, scrambling, startup phases, PRBS, signal integrity) |
| **5** | Data Link Layer (DLL) & OAM | 13 (OAM frames×6, containers×2, mapper, forwarding, light sleep×3) |
| **6** | Security | 5 (key overview, container security, key install×2, key rotation) |
| **7** | ASEP (Stream Encapsulation) | 7 (packet format, payload, mapping, streams, composition, test) |
| **8** | MLE (Motion Link Ethernet) | 3 (PHY overview, PCS transmit, payload mapping) |
| — | Signal Integrity / Channel | 8 (return loss, insertion loss, NEXT, FEXT, response time, cable RL) |
| — | Speed Grade Frequency Charts | 5 (SG1–SG5) |
| — | Inline img-* | 127 (formula fragments, small symbols, inline labels) |

### By Image Type

| Category | Count | Examples |
|----------|-------|----------|
| **Timing Waveforms** | 9 | SPI write/read, GPI data format, multi-frame timing (bit-level annotations, CSn/SCK/MOSI/MISO traces) |
| **Frequency/S-Param Charts** | 13 | Speed Grade 1–5, insertion loss, return loss, NEXT, FEXT, cable RL (Cartesian XY plots, dB vs MHz) |
| **Block/Architecture Diagrams** | 12 | PCS TX flow, DLL container mapping, layers/interfaces, half-duplex overview, MLE PHY, resync header |
| **FEC/Encoding Diagrams** | 10 | RS(108,106), RS(216,214), RS(240,214) parity, PRBS9/PRBS11 LFSR, scramblers, PLP FEC mapping, CRC32 |
| **State Machines/Sequences** | 23 | Transceiver FSM, startup Phase1G/SGA/SGB/SGC, light sleep enter/exit, OAM sequences (root/nonroot/multidevice), PTB clock follower |
| **Frame/Packet Formats** | 12 | OAM frames (tx/rx), DLL container headers, ASEP packets/payload/mapping, DP frame composition |
| **Security/Key Management** | 5 | Key hierarchy, exchange sequence, link key install, container security, key rotation IV |
| **Inline Images (img-*)** | 127 | Small formula fragments, register values, inline symbols (200–3600 bytes each) |

---

## Extended SerDes-Aware Prompt (Chip Design Optimized)

This prompt is designed for GPT-5.1/5.4 analyzing ASA Motion Link technical specification images. It auto-detects the image type and provides type-specific analysis instructions.

```
You are a senior SerDes / high-speed serial link architect reviewing
an ASA (Automotive SerDes Alliance) Motion Link technical specification.
This document covers chapters 2-8: PHY/PCS/PMA/DLL, OAM, Security,
ASEP, and MLE. You understand all standard SerDes concepts including
clock-data recovery, channel equalization, FEC, scrambling, state
machines, key management, and automotive link budget analysis.

Analyze this diagram and first identify which type it is, then follow
the corresponding analysis framework:

---
### TYPE A: TIMING WAVEFORM (SPI, GPI, multi-frame)
Signal traces: M_CSn, M_SCK, M_MOSI, M_MISO, S_CS, S_SCK, S_MOSI, S_MISO
- Extract all numbered timing annotations in parentheses (e.g., (18), (28))
- Identify clock polarity/phase (CPOL/CPHA) and SPI mode (Motorola/TI)
- Map the read command, dummy bytes, and data bytes on MOSI/MISO
- Note buffer timing, coded CS windows, and inter-frame gaps
- Identify which speed grade (SG1–SG5) the timing applies to
- Flag any timing parameters: t_setup, t_hold, t_buffer, t_frame

---
### TYPE B: FREQUENCY / S-PARAMETER CHART
Cartesian plot with Frequency (MHz) on X-axis, dBm/Hz or dB on Y-axis
- Extract ALL axis labels, units, and scale ranges
- Read every major tick value on both axes
- Describe the mask envelope / limit line shape (piecewise linear segments)
- Identify which Speed Grade (SG1-5) the plot applies to
- For S-parameter plots: identify SDD11, SDD21, etc. and the channel type
- For crosstalk: distinguish NEXT vs FEXT, identify victim/aggressor pairs
- State the pass/fail criteria based on the limit lines shown

---
### TYPE C: BLOCK / ARCHITECTURE DIAGRAM
Arrows, boxes, dataflow paths through protocol layers
- Map the layer hierarchy: PMD → PMA → PCS → DLL → Application
- Trace data flow direction (TX downstream vs RX upstream)
- Identify all intermediate signals: d_plp_tx, tx_phy_block, tx_phy_block_scr
- Extract block labels: scrambler, gearbox, RS encoder, mapper, framer
- Note clock domains: m_ptb, tx_clk, rx_clk, recovered clock
- Identify any mux/demux ratios, parallel bus widths, gearbox ratios
- Distinguish between Root Node and Leaf Node paths where applicable

---
### TYPE D: FEC / ENCODING DIAGRAM
Reed-Solomon, PRBS, scrambler, CRC LFSR with Galois field arithmetic
- Extract the generator polynomial g(x) and primitive field polynomial
- Read all coefficient values (g0, g1, g2...) in hex
- Identify RS code parameters: RS(n,k), t = (n-k)/2 error correction capability
- For scramblers: extract seed values (S0, S1), polynomial, direction (up/down)
- For PRBS: extract tap positions and sequence length (2^n - 1)
- For CRC: extract polynomial and initial value
- Map the LFSR shift register data flow (feedback taps, XOR positions)
- Identify codeword format: systematic (data first then parity) or non-systematic

---
### TYPE E: STATE MACHINE / SEQUENCE DIAGRAM
Ovals (states), arrows (transitions), sequence lifelines
- Enumerate every state with its exact name as shown
- List all transitions with trigger conditions
- Identify initial state, normal operating state, and error/fail states
- For startup sequences: distinguish Phase1G, PhaseSGA, PhaseSGB, PhaseSGC
- Note timer references (t1, t2...) and timeout values
- For OAM: identify root vs nonroot device behavior differences
- For light sleep: map entry conditions, retained state, exit triggers
- Identify any loopback or test mode paths

---
### TYPE F: FRAME / PACKET FORMAT DIAGRAM
Bit/byte field layouts with field names and widths
- Extract every field name, bit position, and width in bits/bytes
- Identify header, payload, and trailer sections
- Note any CRC, parity, or checksum fields and their coverage
- Map reserved bits, version fields, and type/control flags
- For DLL containers: extract container header fields, payload structure
- For OAM frames: identify command/response fields, addressing
- For ASEP: map packet format → DLL payload → logical streams

---
### TYPE G: SECURITY / KEY DIAGRAM
Key hierarchies, exchange sequences, cryptographic flows
- Identify key types: BK (binding/base key), DK (device key), LK (link key)
- Map the trust hierarchy: Root ECU → Device → Node → Link
- Extract UUID references and key derivation relationships
- For exchange sequences: enumerate each step with actor (initiator/responder)
- Note any nonce, counter, or IV fields and their sizes
- Identify which AES-GCM or other cipher is implicit in the flow

---
### TYPE H: SMALL INLINE IMAGE (img-*)
Small formula fragment, register bitfield, or inline symbol
- Extract the exact text/number/symbol shown
- If it's a formula: state what physical quantity it represents
- If it's a register value: interpret in hex/decimal/binary
- Identify font styling (bold, italic, subscript, superscript)
- If it's a symbol in context (e.g., from an equation), suggest the likely meaning

---
After completing the type-specific analysis, provide:
1. **Figure Reference**: Which chapter/section/figure number this belongs to
2. **Cross-References**: Which other figures in the spec this diagram relates to
3. **Standards Compliance**: Any IEEE, MIPI, or ASA-specific conventions visible
4. **Implementation Notes**: What an RTL/verification engineer would need from this
```

**Usage:** This single prompt handles all 89 figures + 127 inline images. The model auto-detects the type from visual content and applies the right analysis framework. No need to switch prompts between image types.

---

## Extended vs Original Prompt: Head-to-Head with gpt-5.1

Tested on 8 images (one per type A-H) comparing the original 5-section SerDes Architect prompt against the extended type-aware prompt.

### Quantitative Comparison

| Type | Image | Orig Words | Ext Words | Orig Terms | Ext Terms | Δ Terms |
|------|-------|-----------|----------|-----------|----------|---------|
| A Timing | SPI multi-frame | 719 | 1,646 | 18 | **23** | **+5** |
| B FreqChart | Speed Grade 4 | 674 | 1,200 | 13 | 13 | =0 |
| C BlockDiag | PCS TX flow | 770 | 1,548 | 22 | **23** | +1 |
| D FEC | RS(108,106) | 466 | 1,279 | **15** | 11 | **-4** |
| E StateMachine | Transceiver FSM | 805 | 1,867 | 12 | **13** | +1 |
| F FrameFormat | DLL Container | 719 | 1,553 | 11 | **12** | +1 |
| G Security | Key Overview | 739 | 1,810 | 17 | 17 | =0 |
| H Inline | img-140 (3.6) | 508 | 527 | **10** | 8 | -2 |
| **AVERAGE** | | **675** | **1,428** | **14.8** | **15.0** | +0.3 |

### Latency

| Metric | Original | Extended |
|--------|----------|----------|
| Avg response time | **18.0s** | 36.2s |
| Slowest | 22.7s (BlockDiag) | 53.8s (Security) |
| Structure | Flat numbered list | 11-section markdown with headers |

### Key Findings

| Image Type | Winner | Why |
|------------|--------|-----|
| **Timing Waveforms** | **Extended** (+5 terms) | Auto-detected SPI mode, extracted CPOL/CPHA, enumerated every timing annotation |
| **State Machines** | **Extended** (+1 term) | Listed 30 individual states/transitions vs 5 grouped items; added PHASE detection |
| **Frame/Packet Formats** | **Extended** (+1 term) | Extracted 21 individual field names; original glossed over bit-level detail |
| **Block Diagrams** | **Extended** (+1 term) | Better layer hierarchy mapping; traced individual signal paths |
| **Frequency Charts** | Tie | Same domain terms; extended added pass/fail criteria but was verbose |
| **Security/Key** | Tie | Extended was 2.4x longer but same domain density |
| **FEC/Encoding** | **Original** (-4 terms) | Original was more precise on polynomial extraction; extended over-explained the general concept |
| **Small Inline** | **Original** (-2 terms) | Extended over-analyzed simple text; original was concise and accurate |

### Recommendations

| Scenario | Use |
|----------|-----|
| **Batch processing (all 89 figures)** | Original prompt — 2x faster, same domain accuracy |
| **Timing waveforms / state machines** | Extended prompt — type-specific extraction is worth the latency |
| **FEC / encoding diagrams** | Original prompt — more numerically precise |
| **Frame/packet formats** | Extended prompt — better field-by-field extraction |
| **Cost-sensitive pipeline** | Original prompt — same domain terms in half the tokens |
| **Single detailed analysis** | Extended prompt — richer output with cross-references |