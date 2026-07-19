# WaveServer Architecture: Client-Side vs Server-Side

**Date:** 2026-07-20
**Keyword:** waveserver, client-server, vcd, mcp, rtl-debugging, architecture
**Status:** Design Analysis

---

## Table of Contents

1. [How VCD Simulation Works](#1-how-vcd-simulation-works)
2. [What a WaveServer Does](#2-what-a-waveserver-does)
3. [Client-Side Architecture](#3-client-side-architecture)
4. [Server-Side Architecture](#4-server-side-architecture)
5. [Tradeoff Analysis: Head-to-Head](#5-tradeoff-analysis-head-to-head)
6. [Security Analysis: Moving VCD Files to a Server](#6-security-analysis-moving-vcd-files-to-a-server)
7. [The Indicator Pattern in Both Architectures](#7-the-indicator-pattern-in-both-architectures)
8. [Hybrid Architectures](#8-hybrid-architectures)
9. [Real-World EDA Context](#9-real-world-eda-context)
10. [Recommendations by Scenario](#10-recommendations-by-scenario)
11. [Implementation Roadmap](#11-implementation-roadmap)
12. [Agent Harness Integration: The CLI Orchestrator](#12-agent-harness-integration-the-cli-orchestrator)
13. [Deep Transaction Analysis](#13-deep-transaction-analysis)
14. [Conclusion: Final Recommendation & Rationale](#14-conclusion-final-recommendation--rationale)

---

## 1. How VCD Simulation Works

### The Simulation Pipeline

A VCD (Value Change Dump) file is **not** a simulation — it is a **sparse event log** produced *by* a simulator. Understanding this distinction is critical to designing the WaveServer.

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│   Testbench (SV)    │     │   Simulator Engine   │     │     VCD File        │
│                     │     │                      │     │                     │
│ $dumpfile("w.vcd")  │────►│ Registers hooks for  │────►│ $date               │
│ $dumpvars(0, tb)    │     │ tracked signals      │     │ $timescale 1ps      │
│                     │     │                      │     │ $var wire 1 ! clk   │
│ initial begin       │     │ Event-driven loop:   │     │ ...                 │
│   clk = 0;          │     │  1. Eval always blocks│    │ $enddefinitions     │
│   #5 clk = 1;       │     │  2. Update drivers   │     │ #0                  │
│   ...               │     │  3. If signal changed │     │ $dumpvars          │
│ end                 │     │     → WRITE TO VCD   │     │ 0!   (clk=0)       │
│                     │     │  4. Advance time     │     │ ...                 │
└─────────────────────┘     └─────────────────────┘     │ #5                  │
                                                        │ 1!   (clk=1)       │
                                                        │ ...                 │
                                                        └─────────────────────┘
```

### VCD File Format (IEEE 1364)

A VCD file has three logical sections:

| Section | Contents | Example |
|---------|----------|---------|
| **Header** | Metadata (date, version, timescale) | `$timescale 1ps $end` |
| **Variable Definitions** | Scope hierarchy + signal declarations with identifier codes | `$var wire 1 ! clk $end` |
| **Value Changes** | Time-stamped sparse transitions | `#105\n1!` (clk → 1 at time 105) |

**Critical property: Sparse recording.** If signal `clk` stays at `0` for 1000ps, the VCD writes *nothing* for `clk` during that interval. The viewer/server must *reconstruct* state by scanning backward to the most recent transition.

### What "Playing Back" a VCD Means

A WaveServer does **not** re-simulate. It:
1. Parses the header to build a signal registry (name → identifier code mapping)
2. Indexes all transitions in time order
3. For any query `(signal, time)`, performs a binary search to find the last transition ≤ time
4. Returns the value at that transition

### The wave.vcd Example

From the VCD in this repository (`wave.vcd`), here is what the parsed signal timeline looks like:

```
Time  clk  a    q_ref  q_dut  tb_mismatch   Notes
────  ───  ─    ─────  ─────  ───────────   ─────
  0    0   x     x      x        0          Initial dump
  5    1   0     x      x        0          a driven, clk↑
 15    1   0     1      x        1          q_ref=1, MISMATCH flagged
 45    1   0     0      0        0          All aligned
105    1   1     1      1        0          All aligned
185    1   1     1      0        1  ⚠️       q_dut fell EARLY (q_ref still 1)
195    1   1     1      1        0          q_dut caught up, mismatch cleared
205    1   1     1      0        1  ⚠️       Same pattern repeats
...
```

**Root cause visible in the waveform:** `q_dut` transitions one clock cycle **before** `q_ref`. This causes `tb_mismatch` to assert for exactly 1 cycle, then clear when `q_ref` catches up. The AI debug agent would form hypotheses: "DUT samples on wrong edge", "setup time violation", "reference model buffered by one cycle."

---

## 2. What a WaveServer Does

The WaveServer is a **tool** in the investigation toolbox (the `wave_signals` tool from the architecture docs). It answers queries from hypothesis investigators.

### Core API

```python
class WaveServer:
    # Open/close waveform sessions
    async def open_waveform(self, path: str) -> str        # → session_id
    async def close_waveform(self, session_id: str) -> None

    # Signal discovery
    async def list_signals(
        self, session_id: str, pattern: str = None,
        hierarchy: str = None, recursive: bool = False
    ) -> List[SignalInfo]

    # Value retrieval — the primary tool
    async def get_window(
        self, session_id: str,
        signals: List[str],
        start_time: int, end_time: int
    ) -> WindowResult

    # Anomaly detection — key for debugging
    async def find_mismatches(
        self, session_id: str,
        ref_signal: str, dut_signal: str,
        start_time: int = 0, end_time: int = -1
    ) -> List[MismatchWindow]

    # Conditional search — like waveform-mcp's find_conditional_events
    async def find_events(
        self, session_id: str,
        condition: str,
        start_time: int = 0, end_time: int = -1
    ) -> List[EventMatch]
```

### Output Format (LLM-Optimized)

Raw VCD dumps would blow up context windows. Instead, the WaveServer returns **run-length encoded (RLE)** JSON:

```json
{
  "window": [100, 350],
  "timescale": "1ps",
  "signals": {
    "tb.clk":     [{"t": 105, "v": "1"}, {"t": 110, "v": "0"}, {"t": 115, "v": "1"}],
    "tb.q_ref":   [{"t": 105, "v": "1"}, {"t": 115, "v": "0"}, {"t": 125, "v": "1"}],
    "tb.q_dut":   [{"t": 105, "v": "1"}, {"t": 115, "v": "0"}, {"t": 125, "v": "1"}]
  },
  "mismatches": [
    {"start": 185, "end": 195, "ref": "1", "dut": "0", "duration_ps": 10}
  ]
}
```

This is ~100x more compact than raw VCD text for the same information.

### Data Flow in the Investigation Pipeline

```
regex_log_search           wave_signals              sv_file_outline
        │                       │                          │
        ▼                       ▼                          ▼
  "FATAL @ 185ps"      get_window(q_ref,        "Show always block
   tb_mismatch"         q_dut, 150-220)           driving q_dut"
        │                       │                          │
        └───────────────────────┼──────────────────────────┘
                                │
                                ▼
                    HYPOTHESIS FORMED:
          "q_dut sampled on wrong clock edge at T=185"
```

---

## 3. Client-Side Architecture

### Overview

The WaveServer runs as a **local process** on the engineer's machine. It reads VCD files directly from the local filesystem. Communication happens via MCP stdio (stdin/stdout) or direct Python function calls.

```
┌──────────────────────────────────────────────────────┐
│                 Engineer's Machine                   │
│                                                      │
│  ┌──────────┐    MCP stdio     ┌──────────────┐     │
│  │  Claude   │◄──────────────►│  WaveServer   │     │
│  │ (Desktop) │   (stdin/out)  │  (local proc) │     │
│  └──────────┘                 │               │     │
│                               │  ┌─────────┐  │     │
│                               │  │ VCD     │  │     │
│                               │  │ Parser  │  │     │
│                               │  └────┬────┘  │     │
│                               │       │       │     │
│                               │  ┌────▼────┐  │     │
│                               │  │ In-Mem  │  │     │
│                               │  │ Cache   │  │     │
│                               │  └─────────┘  │     │
│                               └──────────────┘     │
│                                        │            │
│                              reads from│            │
│                                        ▼            │
│                              ┌─────────────────┐    │
│                              │  /sim/waves/     │    │
│                              │  design.vcd      │    │
│                              │  (50 GB)         │    │
│                              └─────────────────┘    │
└──────────────────────────────────────────────────────┘
```

### How It Works

1. Claude Desktop (MCP client) launches `waveserver` as a child process
2. Claude calls `open_waveform("/sim/waves/design.vcd")` via MCP JSON-RPC over stdio
3. WaveServer mmaps or streams the VCD file, parses header, builds signal index
4. Claude calls `get_window(session, ["tb.q_ref", "tb.q_dut"], 150, 220)`
5. WaveServer binary-searches the transition index, returns RLE JSON
6. All data stays on the local machine

### Existing Implementations

| Project | Language | Transport | Formats | Key Feature |
|---------|----------|-----------|---------|-------------|
| [key2/mcp-vcd](https://github.com/key2/mcp-vcd) | Python | MCP stdio | VCD | GTKWave `.gtkw` integration |
| [jiegec/waveform-mcp](https://github.com/jiegec/waveform-mcp) | Rust | stdio + HTTP | VCD, FST | `find_conditional_events` expression engine |
| [fvutils/pywellen-mcp](https://github.com/fvutils/pywellen-mcp) | Python | MCP stdio | VCD, FST, GHW, LXT | 35+ tools, multi-threaded, LRU cache |

### Pros

- **Zero network dependency** — works in air-gapped environments
- **No data leaves the machine** — satisfies the strictest security policies
- **Sub-millisecond latency** — stdio JSON-RPC is near-instant
- **Zero configuration** — `pip install waveserver`, add one line to `claude_desktop_config.json`
- **No file transfer** — reads VCD directly from disk, no upload step
- **Natural for individual use** — matches how engineers already use GTKWave/Verdi locally

### Cons

- **No shared cache** — if 5 investigators query the same VCD, the in-process singleton handles it, but across different Claude sessions, each parses independently
- **Limited by local compute** — parsing a 50GB VCD on a laptop may take minutes and consume significant RAM
- **Single-user** — no collaboration or shared waveform state across a team
- **Memory pressure** — large VCDs compete with other tools (simulators, synthesis) for RAM

---

## 4. Server-Side Architecture

### Overview

The WaveServer runs as an **HTTP service** on a shared server (on-prem or cloud). VCD files are either uploaded to the server or accessed via a shared filesystem (NFS mount). Communication happens via MCP Streamable HTTP.

```
┌──────────────────────┐          ┌──────────────────────────────────────┐
│  Engineer's Machine  │          │         Waveform Server              │
│                      │          │                                      │
│  ┌──────────┐  HTTP  │          │  ┌──────────────┐   ┌────────────┐  │
│  │  Claude   │───────┼──────────┼─►│  API Gateway  │──►│  Session   │  │
│  │ (Desktop) │       │          │  │  (auth/limit) │   │  Manager   │  │
│  └──────────┘       │          │  └──────────────┘   └─────┬──────┘  │
│                      │          │                           │         │
│  ┌──────────┐  HTTP  │          │  ┌────────────────────────▼──────┐  │
│  │  Claude   │───────┼──────────┼─►│       Shared VCD Cache        │  │
│  │ (Desktop) │       │          │  │  ┌─────────┐  ┌─────────┐     │  │
│  └──────────┘       │          │  │  │ Parsed  │  │ Parsed  │     │  │
│                      │          │  │  │ design1 │  │ design2 │     │  │
│  Multiple users      │          │  │  │ (in mem)│  │ (in mem)│     │  │
│  share one server ◄──┼──────────┼──│  └─────────┘  └─────────┘     │  │
│                      │          │  └───────────────────────────────┘  │
│                      │          │                                      │
│                      │          │  ┌────────────────────────────────┐  │
│                      │          │  │  /nfs/sim_results/             │  │
│                      │          │  │  design_regression_20260720/   │  │
│                      │          │  │  ├── test1.vcd  (50 GB)        │  │
│                      │          │  │  ├── test2.vcd  (45 GB)        │  │
│                      │          │  │  └── test3.vcd  (60 GB)        │  │
│                      │          │  └────────────────────────────────┘  │
└──────────────────────┘          └──────────────────────────────────────┘
```

### Two Sub-Patterns

#### 4a. Upload-Based (Cloud/Remote Server)

```
Engineer Machine                    Waveform Server
─────────────────                  ─────────────────
1. User uploads VCD ─────────────► 2. Server receives & stores
                                   3. Server parses & indexes
4. Claude queries signals ◄───────► 5. Server returns data
```

- **Pro:** Works from anywhere, no corporate network needed
- **Con:** Uploading 50GB VCD over VPN/WAN takes minutes to hours
- **Con:** Server stores proprietary waveform data — massive security concern

#### 4b. NFS-Mount Based (On-Prem Server)

```
Engineer Machine         Shared Storage          Waveform Server
─────────────────       ───────────────         ─────────────────
1. Sim writes VCD ────► /nfs/results/vcd  ◄─── 2. Server reads
3. Claude queries ────────────────────────────► 4. Server returns
```

- **Pro:** No upload needed — server reads directly via NFS
- **Pro:** Works within corporate firewall
- **Con:** Requires shared filesystem infrastructure
- **Con:** Server must be on the same network

### Pros

- **Shared cache** — one VCD parsed once, serves all investigators and all users
- **Powerful hardware** — server can have 256GB+ RAM, fast NVMe, many cores for parallel parsing
- **Team collaboration** — multiple engineers can work on the same failure simultaneously
- **Centralized management** — monitoring, auth, rate limiting, audit trails
- **Persistent sessions** — sessions survive client restarts

### Cons

- **Network latency** — every query is an HTTP round-trip (1-50ms)
- **Upload overhead** — for the upload model, transferring large VCDs is slow
- **Security risk** — waveform data contains the entire chip design; a server breach exposes everything
- **Infrastructure complexity** — Docker, Kubernetes, NFS, auth, monitoring
- **Offline unavailable** — doesn't work in air-gapped environments without an on-prem deployment
- **Single point of failure** — if the server is down, nobody can debug

---

## 5. Tradeoff Analysis: Head-to-Head

| Dimension | Client-Side | Server-Side (NFS) | Server-Side (Upload) |
|-----------|-------------|-------------------|---------------------|
| **VCD Read Speed** | Local SSD: 3-7 GB/s | NFS: 1-3 GB/s | Upload over WAN: 0.01-0.1 GB/s |
| **Parse Time (50GB VCD)** | 2-10 min (laptop) | 1-3 min (server, 32 cores) | 1-3 min + 10-100 min upload |
| **Query Latency** | <1ms (in-process) | 2-20ms (HTTP local) | 10-200ms (HTTP remote) |
| **5 Parallel Investigators** | Shared in-process cache, ~100MB | One parse, all share, ~100MB | Same as NFS |
| **Memory Per VCD** | 100MB-2GB (signal index) | 100MB-2GB | 100MB-2GB |
| **Security Posture** | 🟢 Data never leaves machine | 🟡 Data on corporate network | 🔴 Data on remote server |
| **Air-Gapped Support** | 🟢 Works fully offline | 🟡 Needs on-prem server | 🔴 Impossible |
| **Configuration** | `pip install` + 1 JSON line | Docker + NFS + auth | Docker + storage + auth |
| **Multi-User** | ❌ Each user parses independently | 🟢 Shared cache | 🟢 Shared cache |
| **Offline Resilience** | 🟢 Always available | 🔴 Server dependency | 🔴 Server + internet dependency |
| **Team Collaboration** | ❌ No shared state | 🟢 Multiple users, shared view | 🟢 Multiple users, shared view |
| **Audit Trail** | ❌ Local logs only | 🟢 Centralized logging | 🟢 Centralized logging |
| **Cost** | $0 (existing hardware) | Server + maintenance | Server + cloud storage + bandwidth |

### The Critical Insight

The client vs server decision is **not primarily about performance**. For a single user debugging a single failure, client-side is faster (no network). For a team running regression debug, server-side is more efficient (shared cache).

**The decision is primarily about security.** In semiconductor companies, waveform data is treated as highly confidential IP. The security team's answer to "Can we send VCD files to a cloud server?" is almost always **no**.

---

## 6. Security Analysis: Moving VCD Files to a Server

### What's Actually in a VCD File

A VCD file contains the **complete state of every tracked signal** in your design at every transition point. This includes:

- **Signal names** revealing design hierarchy and architecture
- **Signal values** at all points in time — effectively the full functional behavior
- **Module hierarchy** from `$scope` declarations
- **Clock frequencies, protocol timing, data patterns**
- **Failure signatures** that reveal design bugs

In a chip design context, this is **among the most sensitive data** in the company. A VCD file is effectively a complete blueprint of how the chip operates.

### Threat Model

| Threat | Client-Side | Server-Side |
|--------|-------------|-------------|
| **Data exfiltration via compromised server** | N/A — data stays local | 🔴 Server breach = all VCD data exposed |
| **Insider threat (server admin)** | N/A — no server | 🔴 Admin can read all uploaded waveforms |
| **Network interception** | N/A — no network transfer | 🟡 Mitigated by TLS, but data in transit |
| **Cloud provider access** | N/A | 🔴 Cloud provider has physical access |
| **Local machine compromise** | 🟡 Attacker gets VCD + RTL | 🟡 Attacker gets query results only |
| **Accidental exposure** (wrong bucket, public endpoint) | 🟢 Near impossible | 🔴 Misconfigured S3 bucket, exposed API |

### Mitigations for Server-Side

If server-side is chosen, these mitigations are **essential**:

1. **NFS-mount, never upload** — the server reads VCDs from the existing corporate filesystem; no copy is made
2. **On-premises deployment only** — never put waveform data in the cloud
3. **Read-only access** — the WaveServer only reads, never writes or copies VCD files
4. **No persistent storage of VCD data** — parsed in-memory cache only, flushed on session close
5. **Authentication at API level** — tie sessions to corporate SSO
6. **Audit logging** — log every query: who, when, which signals, which time window
7. **Network isolation** — WaveServer on the same VLAN as simulation compute, not exposed to internet

### The Winning Argument for Client-Side

> A client-side WaveServer eliminates the entire security attack surface. There is no server to compromise, no network to intercept, no admin to bribe, no cloud provider to subpoena. The VCD data never leaves the engineer's workstation — the same workstation that already has the RTL source code and the simulator.

---

## 7. The Indicator Pattern in Both Architectures

We previously designed an **async non-blocking execution model** where tool invocations run in parallel and an "indicator" notifies the AI agent when results are ready.

### Client-Side Indicator Pattern

```
Claude (MCP Client)              WaveServer (Local Process)
────────────────────            ────────────────────────────
│                                │
│ 1. open_waveform("big.vcd")───►│ 2. Start parsing in
│                                │    background thread
│ 3. ◄── {"status":"parsing",   │
│         "progress": "12%",     │
│         "retry_in_s": 5}       │
│                                │
│ 4. [Claude works on other      │ 5. Parsing continues...
│    hypotheses, queries         │
│    regex_log_search instead]   │
│                                │
│ 6. poll open_waveform ────────►│ 7. "done"
│ 8. ◄── {"session_id": "abc"}   │
│                                │
│ 9. get_window("abc", ...) ────►│ 10. Binary search index
│ 11. ◄── RLE JSON result        │     (near-instant)
│                                │
```

**Key advantage for client-side:** The indicator pattern is trivial to implement — the WaveServer is a local process with shared memory. A Python `threading.Event` or `asyncio.Future` signals completion. The AI agent (Claude) polls with zero network cost.

### Server-Side Indicator Pattern

```
Claude (MCP Client)              Waveform Server (HTTP)
────────────────────            ────────────────────────────
│                                │
│ 1. POST /open {file:"big.vcd"}►│
│                                │
│ 2. ◄── 202 Accepted            │
│        Location: /jobs/42      │
│        Retry-After: 5          │
│                                │
│ 3. [Work on other things]      │ 4. Parse in background
│                                │    worker
│ 5. GET /jobs/42 ──────────────►│
│ 6. ◄── {"status":"parsing",   │
│          "progress":"45%"}     │
│                                │
│ 7. GET /jobs/42 ──────────────►│
│ 8. ◄── 303 See Other           │
│        Location: /sessions/abc │
│                                │
│ 9. POST /sessions/abc/query ──►│
│ 10.◄── RLE JSON result         │
│                                │
```

**Key disadvantage for server-side:** Polling over HTTP adds latency to every check. WebSockets can mitigate this (server pushes completion), but add complexity.

---

## 8. Hybrid Architectures

Pure client-side and pure server-side are endpoints on a spectrum. Several hybrid approaches exist:

### Hybrid A: Local Parse + Remote AI (Recommended for Most Users)

```
┌──────────────────────────────────────────────────┐
│              Engineer's Machine                   │
│                                                   │
│  ┌──────────┐   MCP stdio    ┌──────────────┐    │
│  │  Claude   │◄──────────────►│  WaveServer   │    │
│  │ (Desktop) │               │  (local proc) │    │
│  └──────────┘                │               │    │
│         │                    │  Parses VCD   │    │
│         │  API (HTTPS)       │  locally       │    │
│         │                    └──────────────┘    │
│         │                                        │
└─────────┼────────────────────────────────────────┘
          │
          │  ONLY structured RLE data
          │  (not raw VCD) goes to cloud
          ▼
┌─────────────────────────────────┐
│     Claude API (Anthropic)      │
│  Receives context-efficient     │
│  JSON, not waveform files       │
└─────────────────────────────────┘
```

**This is the architecture we should build first.** VCD parsing is entirely local. Only the already-compressed RLE JSON ever leaves the machine (as part of the LLM context). The LLM provider sees "signal q_dut fell at 185ps" — not your chip design.

### Hybrid B: On-Prem Server with NFS Mount

```
┌──────────────────────────────────────────────────────┐
│                 Corporate Network                     │
│                                                       │
│  ┌──────────┐   HTTP (internal)   ┌──────────────┐   │
│  │  Claude   │◄──────────────────►│  WaveServer   │   │
│  │ (Desktop) │                    │  (on-prem)    │   │
│  └──────────┘                     │               │   │
│                                   │  Reads via    │   │
│                                   │  NFS mount    │   │
│                                   └───────┬───────┘   │
│                                           │           │
│                                   ┌───────▼───────┐   │
│                                   │  NFS Filer    │   │
│                                   │ /sim/results/ │   │
│                                   └───────────────┘   │
└──────────────────────────────────────────────────────┘
```

Best for large teams in a semiconductor company. Combines server-side shared cache with corporate network security.

### Hybrid C: Client-Side with Remote Fallback

The WaveServer first tries local parse. If the VCD is too large for local memory or the engineer is on a thin client, it falls back to an on-prem server. The MCP config points to both:

```json
{
  "mcpServers": {
    "waveserver": {
      "command": "waveserver",
      "args": ["--local", "--remote-fallback", "https://waveserver.internal.corp:8080"]
    }
  }
}
```

### Comparison of Hybrids

| Architecture | VCD Data Location | Security | Performance | Complexity |
|-------------|-------------------|----------|-------------|------------|
| **Hybrid A: Local Parse + Remote AI** | Local only | 🟢 Best | 🟢 Fast queries | 🟢 Simple |
| **Hybrid B: On-Prem NFS Server** | Corporate network | 🟡 Good | 🟢 Fast, shared cache | 🟡 Medium |
| **Hybrid C: Local + Remote Fallback** | Both (adaptive) | 🟡 Depends on fallback | 🟢 Adaptive | 🔴 Complex |
| **Pure Client-Side** | Local only | 🟢 Best | 🟢 Fast (single user) | 🟢 Simplest |
| **Pure Server-Side Upload** | Remote server | 🔴 Risky | 🟡 Upload overhead | 🟡 Medium |

---

## 9. Real-World EDA Context

### How Existing Tools Handle This

Every major waveform viewer in the EDA industry is **client-side / local**:

| Tool | Vendor | Architecture | VCD files stay... |
|------|--------|-------------|-------------------|
| **GTKWave** | Open source | Local GUI app | On local disk |
| **Verdi nWave** | Synopsys | Local GUI app | On local/NFS disk |
| **SimVision** | Cadence | Local GUI app | On local/NFS disk |
| **Visualizer** | Siemens | Local GUI app | On local/NFS disk |
| **Surfer** | Open source (Rust) | Local GUI + CLI | On local disk |

**None of them are server-side web apps.** The EDA industry has converged on local waveform viewing for a reason: VCD files are too large to move and too sensitive to expose.

### Where the Industry Is Going

- **Surfer** (Rust, open source) is the modern answer to GTKWave — fast, GPU-accelerated, handles multi-GB VCDs smoothly on a laptop
- **Verdi** supports remote display (X11 forwarding) but the viewer logic still runs locally
- **Cloud EDA** (AWS, Azure) runs simulators in the cloud, but waveform viewing is still done via remote desktop to cloud VMs — the VCD stays within the cloud environment

### What This Means for WaveServer

The WaveServer should follow the EDA industry pattern: **client-side first, server-side as an optimized option for teams that already have the infrastructure.** The default should be local. Server-side should be opt-in, on-prem, and NFS-based — never upload-to-cloud.

---

## 10. Recommendations by Scenario

### Scenario 1: Individual Researcher / Open-Source Developer

**Recommendation: Client-Side (Hybrid A)**

- You're debugging your own designs on your own machine
- `pip install waveserver` and add to Claude Desktop config
- Zero security concerns, zero configuration
- VCDs are typically <1GB for personal projects — laptop handles it easily

### Scenario 2: Startup (5-20 engineers)

**Recommendation: Client-Side with Optional On-Prem Server (Hybrid A → B)**

- Start with client-side for speed and simplicity
- If regression debugging becomes a bottleneck, deploy an on-prem server with NFS mount to the simulation results directory
- The MCP config supports both modes; users can switch seamlessly
- Keep VCD data on your existing NFS filer — no new storage system

### Scenario 3: Large Semiconductor Company (100+ engineers)

**Recommendation: On-Prem Server with NFS Mount (Hybrid B)**

- Simulation results already live on a shared NFS filer (Lustre, NetApp, etc.)
- Deploy WaveServer as a Kubernetes service on the same cluster as regression compute
- Shared cache means 100 engineers debugging the same regression failure parse the VCD once
- SSO authentication, audit logging, read-only NFS access
- **Never upload to cloud** — keep everything within the corporate firewall

### Scenario 4: Air-Gapped / Classified Environment

**Recommendation: Client-Side Only**

- No network connectivity whatsoever
- WaveServer runs as a local MCP server
- All tools (regex_log_search, wave_signals, code_fetcher) run locally
- The AI agent itself may run locally (open-weight model via Ollama) or the structured RLE data is the only thing manually shared

---

## 11. Implementation Roadmap

### Phase 1: Client-Side Core (Build First)

```
waveserver/
├── __init__.py          # Package exports
├── parser.py            # VCD/FST parser (pure Python, no heavy deps)
├── indexer.py           # Binary search index over transitions
├── anomaly.py           # Mismatch detection, glitch detection
├── server.py            # MCP stdio server (local)
├── indicator.py         # Async non-blocking execution pattern
└── cli.py               # Standalone CLI for testing without MCP
```

**Key design decisions:**
- Use `pyvcd` or a custom lightweight parser (the VCD format is simple enough to parse with a state machine)
- Store transitions in `numpy` arrays per signal for fast binary search
- LRU cache for parsed VCDs (max 2-3 in memory, oldest evicted)
- MCP stdio transport for Claude Desktop integration
- RLE output format for all query responses

### Phase 2: Server-Side Extension

```
waveserver/
├── ... (Phase 1 files)
├── http_server.py       # FastAPI/Starlette HTTP wrapper
├── session_manager.py   # Multi-user session management
├── auth.py              # SSO integration (OIDC)
├── audit.py             # Query logging for compliance
└── nfs_watcher.py       # Watch NFS directories for new VCDs
```

**Key design decisions:**
- Same `waveserver` package, different entrypoint (`waveserver --http` vs `waveserver --stdio`)
- Shared in-memory cache protected by `asyncio.Lock` for concurrent access
- MCP Streamable HTTP transport per the 2025 MCP spec
- NFS mount mode: server accepts a directory path, not a file upload

### Phase 3: Production Hardening

- **FST binary format support** — VCD is ASCII and bloated; FST is 10-50x smaller and faster to parse. Use `pywellen` or `wellen` Rust bindings
- **Parallel VCD parsing** — split the VCD file by time ranges and parse in parallel threads
- **Incremental indexing** — if a new regression run appends to a VCD, only index the new data
- **Waveform diff** — compare two VCD files (e.g., pre-fix vs post-fix) and highlight differences
- **Signal grouping import** — read `.gtkw` files to understand how engineers organize signals into buses and groups

---

## Summary Decision Matrix

| If you... | Choose... | Because... |
|-----------|-----------|------------|
| Work alone on your laptop | Client-side | No reason to add server complexity |
| Work in a small team with shared storage | Client-side first, add on-prem server later | Start simple, scale when needed |
| Work in a large company with NFS infrastructure | On-prem server with NFS mount | Shared cache saves real time across 100+ engineers |
| Work in defense/aerospace/classified | Client-side only | Security policy prohibits server-side |
| Have VCDs <5GB | Client-side | Parse time is negligible |
| Have VCDs >50GB regularly | On-prem server with beefy hardware | Server can have 256GB+ RAM for large indexes |
| Need audit trail for compliance | On-prem server | Centralized logging |
| Need offline capability | Client-side | Server doesn't work without network |

---

## 12. Agent Harness Integration: The CLI Orchestrator

### What Is the Agent Harness?

The **agent harness** is the CLI orchestrator — think Claude Code, Codex CLI, or our Viv agent. It runs a tool-calling loop:

```
┌─────────────────────────────────────────────────────────┐
│                    Agent Harness Loop                     │
│                                                          │
│  ┌──────────┐     ┌──────────┐     ┌──────────────────┐ │
│  │   LLM    │────►│  Choose  │────►│  Execute Tool    │ │
│  │ (Claude) │     │  Tool    │     │  (spawn / call)  │ │
│  └──────────┘     └──────────┘     └────────┬─────────┘ │
│       ▲                                      │           │
│       │         ┌──────────┐                 │           │
│       └─────────│  Format  │◄────────────────┘           │
│                 │  Result  │                             │
│                 └──────────┘                             │
│                                                          │
│   Tools: [regex_log_search, regex_code_search,           │
│           wave_signals, code_fetcher, log_fetcher,       │
│           hierarchy_descent, sv_references, ...]          │
└─────────────────────────────────────────────────────────┘
```

The harness manages **tool lifecycle**: spawn processes, send/receive MCP JSON-RPC, handle timeouts, retry on failure, and marshal results back to the LLM. The WaveServer is one tool among many — the harness treats it the same as `regex_log_search` or `code_fetcher`.

### Integration Model: How the Harness Connects to the WaveServer

#### Architecture B: Client-Side MCP (Recommended)

```
Agent Harness (Parent Process)
│
├── Spawns child: waveserver --stdio
│   │
│   ├── MCP stdio transport (stdin/stdout)
│   │   ┌──────────────────────────────────────────┐
│   │   │  → {"jsonrpc":"2.0","method":"tools/call",│
│   │   │     "params":{"name":"get_window",        │
│   │   │     "arguments":{"signals":["tb.q_ref",   │
│   │   │     "tb.q_dut"],"start":150,"end":220}}}  │
│   │   │                                           │
│   │   │  ← {"jsonrpc":"2.0","result":{...RLE...}} │
│   │   └──────────────────────────────────────────┘
│   │
│   └── On crash: SIGCHLD → harness restarts process
│
└── Config (~10 lines):
    {
      "mcpServers": {
        "waveserver": {
          "command": "python",
          "args": ["-m", "waveserver", "--stdio"]
        }
      }
    }
```

**Integration complexity: ~10 lines of JSON config.** The harness already speaks MCP stdio natively — it's the standard pattern for all MCP tools. Zero new code paths in the harness. The WaveServer process is just another child process alongside `ripgrep` and `ctags`.

#### Architecture A: Remote HTTP WaveServer

```
Agent Harness
│
├── HTTP Client Layer (CUSTOM CODE — ~500+ lines)
│   │
│   ├── Connection pool (aiohttp / httpx)
│   ├── Auth manager (token refresh, expiry detection)
│   ├── Upload manager (chunked transfer, progress tracking)
│   ├── Retry policy (exponential backoff, circuit breaker)
│   ├── Session tracker (job_id → session_id mapping)
│   └── Error classifier (4xx = user error, 5xx = retry, timeout = backoff)
│
├── Upload phase:
│   │   POST /api/v1/waveforms
│   │   Content-Type: multipart/form-data
│   │   Body: <20GB VCD file streamed in 64MB chunks>
│   │   ← 202 Accepted {job_id: "abc123", status_url: "/jobs/abc123"}
│   │
├── Poll phase:
│   │   GET /jobs/abc123
│   │   ← 200 {status: "parsing", progress: 0.45}
│   │   GET /jobs/abc123
│   │   ← 200 {status: "parsing", progress: 0.89}
│   │   GET /jobs/abc123
│   │   ← 200 {status: "ready", session_id: "sess_xyz"}
│   │
├── Query phase:
│   │   POST /api/v1/sessions/sess_xyz/query
│   │   Body: {"signals":["tb.q_ref","tb.q_dut"],"start":150,"end":220}
│   │   ← 200 {signals: {...RLE...}}
│   │
└── Cleanup:
        DELETE /api/v1/sessions/sess_xyz
        ← 204 No Content
```

**Integration complexity: ~500+ lines of new harness code.** The harness needs a full HTTP client stack: connection pooling, auth token lifecycle management, chunked upload with resume capability, async polling with backoff, circuit breaker for when the server is down, and proper cleanup on harness shutdown. Every one of these is a new failure mode.

### Tool Invocation Comparison

| Aspect | Client-Side MCP | Remote HTTP Server |
|--------|----------------|-------------------|
| **Harness code to add** | ~0 lines (native MCP) | ~500+ lines |
| **Tool discovery** | MCP `tools/list` (automatic) | Manual API spec import |
| **Error propagation** | JSON-RPC error codes (standard) | HTTP status codes (must map) |
| **Process lifecycle** | SIGCHLD on crash → auto-restart | Connection error → manual reconnect |
| **Timeout handling** | OS-level (dead process = broken pipe) | Network-level (TCP timeout = 30-120s default) |
| **Concurrent queries** | Shared in-process state | HTTP/2 multiplexing or connection pool |
| **Graceful shutdown** | SIGTERM → process exits cleanly | DELETE session + close connections |
| **Dry-run / testing** | `waveserver --stdio < test_commands.json` | Need running server or mock |

### What Happens During a Multi-Tool Investigation

The agent harness runs tools **sequentially** within a single hypothesis investigator, but **in parallel** across 5 different hypothesis investigators. Here's the trace for Investigator #2 ("Timing Hypothesis"):

```
Time   Harness Action                          Transport        Latency
────   ─────────────                          ─────────        ───────
0ms    LLM: "Call regex_log_search for 185ps"
1ms    spawn: rg -U "185[0-9]{2}" sim.log      local process    <1ms
15ms   ← result: "tb_mismatch asserted at 185"  stdout          ~15ms parse
16ms   LLM: "Interesting. Call wave_signals"
17ms   tools/call {get_window, [150,220]}        MCP stdio       <1ms
18ms   ← RLE JSON: q_dut fell, q_ref stayed       MCP stdio       <1ms
19ms   LLM: "This is a timing issue. Call code_fetcher for the flip-flop driving q_dut"
20ms   tools/call {code_fetcher, ibex_core.sv}   MCP stdio       <1ms
22ms   ← always @(posedge clk) q_dut <= a;       MCP stdio       ~2ms
25ms   LLM: "HYPOTHESIS CONFIRMED: q_dut samples on posedge, but reference model uses negedge"
```

**Total wall clock for one investigator: ~25ms of harness overhead + LLM inference time.** The tool execution itself (parsing, searching, fetching) is dominated by I/O, not harness latency. Client-side MCP stdio adds negligible overhead.

With the **Remote HTTP Server**, the same trace would have 20-150ms added to each tool call just from network round-trip, and the initial upload phase would add 15-45 minutes before the first `get_window` can even be called.

---

## 13. Deep Transaction Analysis

### 13.1 End-to-End Timing Breakdown (20GB VCD)

A typical debug session with a 20GB VCD file. Times are wall-clock, single user.

#### Architecture B: Client-Side MCP

```
Phase                    Duration    CPU Cores    Notes
─────                    ────────    ─────────    ─────
Harness spawns process   0.1s        1            fork + python import
Parse VCD header         0.5s        1            Read ~10KB header, build signal map
Stream & index body      60-120s     1-4          mmap or buffered read, state machine per line
Build binary search idx  5-15s       1            numpy array per signal, sort by time
─── READY ───────────────────────────────────────
get_window query         <0.001s     1            Binary search + reconstruct state
Format RLE JSON          <0.001s     1            Serialize to JSON
MCP stdio round-trip     <0.001s     -            Local pipe, no serialization overhead
─── TOTAL TO FIRST QUERY: ~1-2 minutes ──────────
─── PER-QUERY LATENCY: <1ms ─────────────────────
```

#### Architecture A: Remote HTTP (Upload Model)

```
Phase                    Duration       Notes
─────                    ────────       ─────
Compress (optional)      2-5 min        gzip -1 on VCD: ~2-3x reduction, CPU intensive
Upload 8-20GB             15-45 min      At 100 MB/s (1 Gbps link, realistic with overhead)
Server parse             30-90s          Server has 16-32 cores, parallel chunk parsing
Build index              5-10s           Fast NVMe + large RAM
─── READY ───────────────────────────────────────
get_window query         0.05-0.2s       HTTP round-trip (LAN: ~20ms, WAN: ~100ms)
─── TOTAL TO FIRST QUERY: ~20-50 minutes ───────
─── PER-QUERY LATENCY: 20-200ms ────────────────
```

#### Architecture A: Remote HTTP (NFS Mount — No Upload)

```
Phase                    Duration       Notes
─────                    ────────       ─────
Upload                   0s             Server reads directly from NFS
Server parse             30-90s          Shared array, parallel cores
Build index              5-10s           
─── READY ───────────────────────────────────────
get_window query         0.05-0.2s       HTTP round-trip
─── TOTAL TO FIRST QUERY: ~1-2 minutes ──────────
─── PER-QUERY LATENCY: 20-200ms ────────────────
```

**Key takeaway:** The NFS-mount server matches client-side parse time but adds 20-200ms per query. The upload model is categorically unviable for interactive debugging — nobody will wait 30 minutes to start investigating a failure.

### 13.2 Resource Consumption Profiles

| Resource | Client-Side MCP | Remote HTTP (Upload) | Remote HTTP (NFS) |
|----------|----------------|---------------------|-------------------|
| **Client CPU** | 🔴 1-4 cores @ 100% for 2 min (parse burst) | 🟢 Near-zero (just network I/O) | 🟢 Near-zero |
| **Client RAM** | 🟡 200MB-1GB (index) + 500MB (harness + LLM context) = ~1-2GB total | 🟢 ~500MB (harness only) | 🟢 ~500MB (harness only) |
| **Client Disk** | 🟢 Reads VCD from existing location. Optional: ~200MB temp for index cache | 🟡 Needs temp space for compressed VCD during upload | 🟢 No disk used |
| **Client Network** | 🟢 **0 bytes** | 🔴 **20-50GB outbound** (raw or compressed VCD) | 🟢 **~10KB per query** (JSON only) |
| **Server CPU** | N/A | 🟡 16-32 cores during parse (shared) | 🟡 16-32 cores during parse (shared) |
| **Server RAM** | N/A | 🟡 1-2GB per active VCD (shared across users) | 🟡 1-2GB per active VCD |
| **Server Disk** | N/A | 🔴 Must store uploaded VCDs (potentially TB-scale) | 🟢 No server-side storage (reads from NFS) |

**The resource tradeoff is simple:** Client-side burns local CPU for 2 minutes but uses zero network. Server-side saves local CPU but burns network (upload model) or requires NFS infrastructure (mount model).

**Mitigation for client CPU:** The VCD parse runs at lower I/O priority (`ionice -c 3`) and CPU priority (`nice -n 10`) so it doesn't starve the simulator or IDE. Most engineers have 8-16 core machines; 1-4 cores at 100% for 2 minutes is barely noticeable.

### 13.3 Failure Mode Catalog

Every possible failure mode, how each architecture handles it, and which handles it better.

| # | Failure Mode | Client-Side MCP | Remote HTTP Server | Winner |
|---|-------------|----------------|-------------------|--------|
| 1 | **VCD file not found / wrong path** | Immediate error: `FileNotFoundError`. Harness reports to LLM. | Upload fails immediately. Same UX. | Tie |
| 2 | **VCD corrupt / truncated** | Parser fails at corrupt line. Returns `{error: "truncated at line 1.2M", parsed_until: 350ps}`. Partial data may still be useful. | Upload succeeds, parser fails server-side. User waited 30 min to find out. | **Client** |
| 3 | **VCD too large for RAM** | Progressive memory pressure → OS may OOM-kill. Mitigation: streaming parser that never holds full VCD in memory, only the index (~1-5% of VCD size). If index > available RAM, evict to sqlite on disk. | Server has 256GB+ RAM — handles it. | **Server** (mitigated by streaming parser on client) |
| 4 | **Parse timeout** | Harness sets subprocess timeout (configurable, default 10 min). On timeout: SIGTERM → partial results returned. | Server returns 408 Request Timeout or 202 with failed status. Harness retries or reports. | **Client** (cleaner: local SIGTERM) |
| 5 | **Upload interrupted** | N/A — no upload. | Connection reset at 80%. Harness must implement resumable upload (Content-Range) or retry from start. 20GB retry is painful. | **Client** |
| 6 | **Server unreachable** | N/A — no server. | DNS failure, firewall block, maintenance window. Harness must implement circuit breaker: after 3 failures, tell LLM "wave_signals unavailable, use other tools". | **Client** |
| 7 | **Server returns 5xx** | N/A. | Harness retries with exponential backoff (1s, 2s, 4s, 8s). After N retries, reports error to LLM. | **Client** |
| 8 | **Auth token expired** | N/A — no auth. | Mid-session: queries start returning 401. Harness must refresh token (OAuth2 refresh flow) or fail. Token refresh can itself fail. | **Client** |
| 9 | **WaveServer process crash** | Child process exits with non-zero. Harness detects via SIGCHLD or broken pipe. Automatically respawns and re-opens waveform (session lost, but VCD is local → fast re-parse). | Server crash = all users blocked. Harness gets 502/504. Waits for server to recover (minutes? hours?). | **Client** |
| 10 | **Zombie processes** | If harness is killed (SIGKILL), child `waveserver` may become zombie. Harness should register `atexit` handler to SIGTERM children. OS eventually reaps zombies. | N/A — no local child processes. | **Server** (no zombies) |
| 11 | **Disk full (local)** | Can't write index cache. Fallback: in-memory only, no caching. Warn user. | N/A — server manages its own disk. | **Server** |
| 12 | **Signal not found** | `{error: "signal 'tb.bad_name' not found. Did you mean: 'tb.q_dut', 'tb.q_ref'?"}` — fuzzy match suggestions. | Same error, but over HTTP. | Tie |
| 13 | **Time range out of bounds** | `{error: "time 999999 exceeds VCD max time 615"}` | Same. | Tie |
| 14 | **LLM context overflow** | Too many signals / too large window → RLE JSON exceeds context budget. Mitigation: pagination (max 50 transitions per signal) and summarization ("48 transitions omitted"). | Same mitigation. | Tie |
| 15 | **Concurrent access race** | Two investigators in same harness process share in-memory index via asyncio.Lock. No race. | Two HTTP clients query same session. Server must use read-write lock. Minor. | Tie |
| 16 | **No network connectivity** | 🟢 Works perfectly offline. | 🔴 Dead in the water. Can't even start. | **Client** |

**Score: Client wins 8, Server wins 2, Tie 6.** The client-side architecture eliminates entire categories of failure: network, auth, upload, server availability. The only advantage server-side has is handling VCDs that genuinely don't fit in local RAM — and that's rare with modern workstations and streaming parsers.

### 13.4 State Machine Diagrams

#### Architecture B: Client-Side MCP WaveServer

```
                    ┌──────────────────────────┐
                    │         IDLE              │
                    │  WaveServer not running   │
                    └────────────┬─────────────┘
                                 │
                     harness spawns child process
                     (subprocess.Popen, ~100ms)
                                 │
                                 ▼
                    ┌──────────────────────────┐
           ┌───────►│        PARSING            │◄──────────────┐
           │        │  tool status: "parsing"   │               │
           │        │  progress: 0.0 → 1.0      │               │
           │        └────────────┬─────────────┘               │
           │                     │                              │
           │      parse complete │           parse timeout      │
           │      (MCP notification)         (harness timer)    │
           │                     │                              │
           │                     ▼                              │
           │        ┌──────────────────────────┐               │
           │        │         READY             │───────────────┘
           │        │  tool registered:         │  (partial results
           │        │  get_window, find_mismatch│   returned)
           │        └────────────┬─────────────┘
           │                     │
           │       LLM calls      │
           │       tools/call     │
           │                     ▼
           │        ┌──────────────────────────┐
           │        │        QUERYING           │
           │        │  binary search + RLE      │
           │        │  latency: <1ms            │
           │        └────────────┬─────────────┘
           │                     │
           │                     ▼
           │        ┌──────────────────────────┐
           │        │     RESULT RETURNED       │────► back to READY
           │        │  RLE JSON to harness      │     (wait for next query)
           │        └──────────────────────────┘
           │
           │   ┌──────────────────────────┐
           │   │        ERROR              │
           └───┤  process crash, OOM,      │
               │  file not found, corrupt  │
               └────────────┬─────────────┘
                            │
               harness detects, reports to LLM
               may respawn (re-parse needed)
                            │
                            ▼
                        back to IDLE

Timeout values:
  PARSING: default 10 min (configurable)
  QUERYING: default 30 sec (single query should be <1ms; 30s is extreme safety margin)
```

#### Architecture A: Remote HTTP WaveServer

```
                    ┌──────────────────────────┐
                    │         IDLE              │
                    │  No session active        │
                    └────────────┬─────────────┘
                                 │
              harness initiates upload
              POST /waveforms (20GB body)
                                 │
                                 ▼
                    ┌──────────────────────────┐
           ┌───────►│       UPLOADING           │◄──────────────┐
           │        │  progress: 0% → 100%      │               │
           │        │  duration: 15-45 min      │    retry from │
           │        └────────────┬─────────────┘    last chunk  │
           │                     │                   (Content-   │
           │      upload success │  upload fail       Range)     │
           │      ← 202 {job_id} │  (network blip)               │
           │                     │                              │
           │                     ▼                              │
           │        ┌──────────────────────────┐               │
           │        │        PARSING            │               │
           │        │  harness polls:           │               │
           │        │  GET /jobs/{job_id}       │               │
           │        │  interval: 5s             │               │
           │        │  max polls: 120           │               │
           │        └────────────┬─────────────┘               │
           │                     │                              │
           │       server done   │  server timeout/fail         │
           │       ← 303 /sessions/{id}                        │
           │                     │                              │
           │                     ▼                              │
           │        ┌──────────────────────────┐               │
           │        │         READY             │───────────────┘
           │        │  session_id stored        │
           │        └────────────┬─────────────┘
           │                     │
           │       LLM calls      │
           │       wave_signals   │
           │                     ▼
           │        ┌──────────────────────────┐
           │        │        QUERYING           │
           │        │  POST /sessions/{id}/query│
           │        │  latency: 20-200ms        │
           │        └────────────┬─────────────┘
           │                     │
           │       ← 200 OK      │  ← 401 (token expired)
           │                     │  ← 502 (server down)
           │                     │  ← 504 (gateway timeout)
           │                     ▼
           │        ┌──────────────────────────┐
           │        │     RESULT RETURNED       │────► READY
           │        └──────────────────────────┘
           │
           │   ┌──────────────────────────┐
           │   │        ERROR              │
           └───┤  circuit breaker opens    │
               │  after 3 consecutive      │
               │  failures                 │
               └────────────┬─────────────┘
                            │
               harness reports to LLM:
               "wave_signals unavailable"
               suggests alternative tools
                            │
                            ▼
                        back to IDLE
               (circuit breaker resets after 60s)

Additional states unique to server-side:
  UPLOADING: 15-45 min, must be resumable
  AUTH_REFRESHING: transparent token refresh mid-session
  CIRCUIT_OPEN: server unreachable, stop trying
```

### 13.5 Transaction Mechanics: How the Receiver and Sender Connect

#### Client-Side MCP: The Connection

The "connection" is a **local pipe** — the simplest possible IPC:

```python
# Agent harness (pseudocode)
import subprocess, json

class MCPTool:
    def __init__(self):
        # Spawn the WaveServer as a child process
        self.process = subprocess.Popen(
            ["python", "-m", "waveserver", "--stdio"],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE  # for logging, not data
        )

    async def call(self, tool_name: str, arguments: dict) -> dict:
        request = {
            "jsonrpc": "2.0",
            "id": self._next_id(),
            "method": "tools/call",
            "params": {"name": tool_name, "arguments": arguments}
        }
        # Write to child's stdin
        self.process.stdin.write(json.dumps(request).encode() + b"\n")
        self.process.stdin.flush()
        # Read response from child's stdout
        line = self.process.stdout.readline()
        return json.loads(line)
```

**There is no network. No serialization overhead. No authentication. No TLS handshake.** The OS kernel handles the pipe. Data moves at memory bandwidth (~50 GB/s), not network bandwidth (~1 Gbps).

#### Remote HTTP Server: The Connection

The connection is a **full HTTP stack** with multiple layers:

```
Agent Harness                  Network                    Remote Server
─────────────                  ───────                    ─────────────

1. DNS resolve ───────────────► DNS server ──────────────► returns IP
   (5-50ms)                                              

2. TCP handshake ─────────────► SYN ─────────────────────► SYN-ACK
   (1-100ms RTT)               ACK ◄───────────────────── 

3. TLS 1.3 handshake ─────────► ClientHello ─────────────► ServerHello
   (10-100ms)                   Certificate verify        Certificate
                                Finished                   Finished

4. HTTP request ───────────────► POST /wavesforms ────────►
   (chunked transfer)           Transfer-Encoding: chunked
                                64MB chunks streaming

5. Auth: Authorization header ─► Bearer <jwt_token> ─────► validate
   (token may expire mid-transfer!)

6. Response ◄────────────────── 202 Accepted ◄────────────
   {job_id: "..."}
```

**Every query repeats steps 1-5** (minus the upload). For a keep-alive connection, steps 1-3 are amortized, but step 4 (HTTP request/response) is always 1 RTT. At 20-100ms RTT, 100 queries = 2-10 seconds of pure network waiting — compared to <1ms total for 100 local MCP queries.

### 13.6 What Happens When a Service Fails

#### Scenario: Mid-Debug Server Crash

```
Time   Client-Side MCP                          Remote HTTP Server
────   ─────────────                            ──────────────────
0s     LLM: "Call wave_signals for 150-220"     LLM: "Call wave_signals for 150-220"
0.1s   ← RLE JSON result                        POST /sessions/abc/query → ... no response ...
0.2s   LLM: "Interesting. Call code_fetcher"     
1s                                              TCP timeout begins (30s default)
5s     Investigation continues normally          Harness: no response yet. LLM is waiting.
10s                                             Harness: still waiting.
15s                                             LLM context: "Calling wave_signals..." (spinner)
30s                                             TCP timeout. Harness retries.
30.1s                                           POST /sessions/abc/query → 502 Bad Gateway
30.2s                                           Harness: circuit breaker opens.
31s                                             Harness to LLM: "Error: wave_signals unavailable"
32s                                             LLM: "OK, I'll work with what I have from logs."
                                                LOST: 32 seconds of investigation time.
```

**Client-side recovers instantly** (process crash → harness respawns, re-opens VCD in ~1-2 min). **Server-side loses 30+ seconds per timeout**, and if the server is down for maintenance, the entire debug session is blocked indefinitely.

#### Scenario: Upload Failure at 80%

```
Remote HTTP Server (Upload Model) Only:

Time   Event
────   ─────
0min   Upload started: 20GB VCD
12min  80% complete (16GB uploaded)
12min  NETWORK BLIP: VPN drops for 3 seconds
12min  TCP connection reset
12min  Harness must decide: retry from 0% or resume from 80%?

       Option A: Retry from 0%
       24min  Upload completes (total: 36 min)
       
       Option B: Resume from 80% (requires Content-Range support)
       15min  Upload completes (total: 27 min)
       
       Option C: Fail and tell LLM
       12min  LLM: "Cannot access waveform data. Debug using logs only."
       
       NONE OF THESE ARE GOOD.
```

**Client-side has no upload phase.** There is nothing to fail. The VCD is already on disk.

---

## 14. Conclusion: Final Recommendation & Rationale

### The Recommendation

**Build the WaveServer as a client-side MCP stdio tool first. Offer an on-prem server with NFS mount as an optional deployment for large teams. Never build an upload-based cloud server.**

### Rationale (In Order of Importance)

#### 1. Security Is Non-Negotiable

VCD files contain the complete functional behavior of a chip design — clock frequencies, data patterns, protocol timing, failure signatures. In the semiconductor industry, this is crown-jewel IP. Sending it over a network to any server creates an unacceptable attack surface. Client-side eliminates this surface entirely.

#### 2. Upload Is Unviable for Interactive Debugging

Debugging is an interactive, iterative process. An engineer (or AI agent) asks a question, gets an answer, asks a follow-up, gets another answer — dozens of times per session. Waiting 15-45 minutes to upload a 20GB VCD before the first query can be made destroys this flow. The upload model turns a 5-minute debug session into a 50-minute ordeal.

#### 3. Failure Modes Are Drastically Fewer Client-Side

Of 16 failure modes analyzed, client-side architecture handles 8 better, ties on 6, and loses on only 2 (and those 2 are mitigated by streaming parsers and modern workstation hardware). Network failures, auth token expiry, server crashes, DNS issues, firewall blocks — all eliminated.

#### 4. The NFS-Mount Server Is a Valid Optimization for Teams

For large teams where VCDs are already on a shared filesystem, an on-prem server with read-only NFS access provides shared caching without the upload overhead. This is the one server-side pattern that makes sense — and it only works because the VCD never moves.

#### 5. The MCP Ecosystem Defaults to Client-Side

The Model Context Protocol was designed for local tools. Every major MCP server (filesystem, git, postgres, memory) runs as a local child process. Our WaveServer should follow this pattern. The integration is ~10 lines of config, not ~500 lines of custom HTTP client code.

### Decision Flowchart

```
Is your environment air-gapped?
  ├── YES → Client-Side MCP (only option)
  └── NO  → Continue ↓

Do you have a shared NFS filesystem for simulation results?
  ├── YES → Is your team > 20 engineers?
  │          ├── YES → On-Prem NFS Server + Client-Side fallback
  │          └── NO  → Client-Side MCP (simpler, same speed)
  └── NO  → Client-Side MCP (only option that makes sense)

Would your security team approve sending waveform data to a server?
  ├── YES → (They won't. But if they do:) On-Prem NFS Server
  └── NO  → Client-Side MCP
```

### What We Lose by Choosing Client-Side

Fairness requires acknowledging the tradeoffs:

| We lose... | Mitigation |
|-----------|------------|
| Shared cache across users | Acceptable: parse time is 1-2 min per user; negligible in a multi-hour debug session |
| Centralized audit logging | Mitigation: local logs can be aggregated. Or deploy NFS server for teams that need compliance. |
| Thin-client support | Mitigation: modern engineering laptops have 32-64GB RAM, 8-16 cores. 20GB VCD parsing uses ~2GB RAM. If truly constrained, fall back to on-prem NFS server. |
| Server-grade parallel parsing | Mitigation: VCD parsing is I/O-bound, not CPU-bound. A streaming parser on a local NVMe is as fast as a 32-core server over NFS. |

### What We Gain by Choosing Client-Side

| We gain... | Impact |
|-----------|--------|
| Zero network dependency | Works anywhere: office, home, airplane, air-gapped lab |
| Zero security attack surface | No server to compromise, no network to intercept, no auth to manage |
| Instant first query | 1-2 min parse, then <1ms per query — no 30-min upload wait |
| Zero configuration | `pip install waveserver` + one JSON config line |
| No infrastructure to maintain | No Docker, no Kubernetes, no NFS, no auth, no monitoring |
| Matches EDA industry pattern | GTKWave, Verdi, SimVision — all local |
| Matches MCP ecosystem pattern | All standard MCP servers are local child processes |

### The Hybrid Path Forward

1. **Phase 1 (now):** Build the client-side MCP WaveServer. Ship it. Let engineers use it.
2. **Phase 2 (when needed):** If a large team requests shared caching, add `--http` mode that reads from NFS. The same codebase, same tools, different transport.
3. **Phase 3 (never):** Do not build upload-to-cloud. The security and latency costs are insurmountable for chip design workflows.

---

## Sources

- IEEE 1364-2001 (Verilog HDL) — VCD format specification
- [jiegec/waveform-mcp](https://github.com/jiegec/waveform-mcp) — Rust MCP server for VCD/FST (MIT)
- [key2/mcp-vcd](https://github.com/key2/mcp-vcd) — Python MCP server for VCD + GTKWave (MIT)
- [fvutils/pywellen-mcp](https://github.com/fvutils/pywellen-mcp) — Python MCP server with 35+ EDA tools (Apache 2.0)
- [Surfer](https://gitlab.com/surfer-project/surfer) — Modern Rust waveform viewer
- `ideas/revised_parllel_investigation.md` — Multi-hypothesis RTL debug architecture
- `ideas/multi_hypothesis_rtl_debug_architecture.md` — Viv agent architecture reconstruction
- `wave.vcd` — Example Icarus Verilog VCD with cosimulation mismatch pattern

**Next:** Implement Phase 1 (client-side WaveServer core) with VCD parser, signal indexer, anomaly detector, and MCP stdio server.
