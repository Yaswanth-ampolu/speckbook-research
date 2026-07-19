# TypeScript Regex Search Tools for CLI Agent Harnesses

**Date:** 2026-07-20
**Keyword:** typescript, regex-search, cli-harness, ripgrep, mcp, ast-grep
**Status:** Reference & Implementation Guide

---

## Table of Contents

1. [What the Major CLI Harnesses Use](#1-what-the-major-cli-harnesses-use)
2. [TypeScript Implementation Options](#2-typescript-implementation-options)
3. [Lightweight Search Backends Compared](#3-lightweight-search-backends-compared)
4. [Building a TypeScript Grep Tool](#4-building-a-typescript-grep-tool)
5. [MCP Server Integration](#5-mcp-server-integration)
6. [AST-Aware Search](#6-ast-aware-search)
7. [Full Implementation Reference](#7-full-implementation-reference)

---

## 1. What the Major CLI Harnesses Use

### Claude Code (Anthropic)

**Backend:** `ripgrep` (`rg`) invoked via child process.

**Tool Schema:**
```typescript
interface GrepToolInput {
  pattern: string;           // Required: regex pattern
  path?: string;             // File or directory to search (default: cwd)
  glob?: string;             // File filter, e.g. "*.ts", "**/*.tsx"
  output_mode?: "content" | "files_with_matches" | "count";
  multiline?: boolean;       // Allow matching across lines
  head_limit?: number;       // Cap on output lines
}
```

**Key design decisions:**
- `output_mode: "files_with_matches"` is used when the agent just needs to locate files, not read content — saves massive context
- `head_limit` prevents context overflow from broad searches
- Multiline mode uses `rg -U` for cross-line patterns like `struct \{[\s\S]*?field`
- Claude Code explicitly **forbids** using `grep`/`rg`/`find` via the Bash tool — all search must go through the dedicated `Grep` tool to maintain sandboxing and result formatting

### Aider

**Backend:** `ripgrep` for text search + **tree-sitter** via `grep-ast` for structured context.

**Two-tier search:**
1. Raw text search via `ripgrep` — fast, broad
2. `grep-ast` — a Python utility that uses tree-sitter to show AST context around regex matches: "here's the function/class/method that contains this match"

**Why the second tier matters:** When `rg` finds `encoding` at line 42, `grep-ast` also shows you that line 42 is inside `def parse_header(self):` at line 38 and inside `class VCDParser:` at line 20. This context is invaluable for an LLM.

### Cursor / Continue

**Backend:** VS Code's internal search APIs (`vscode.workspace.findFiles`, `vscode.workspace.findTextInFiles`) when running in-IDE. When running as a standalone agent, falls back to `ripgrep`.

### Viv v0.2.3 (Our Subject System)

**Backend:** Observed using some grep-like tool (likely `ripgrep` based on performance and pattern syntax). Patterns are dynamically constructed by the LLM as pipe-separated keyword alternatives:
```
pattern: "Cosim mismatch|1856300|x26|commit|pc|instr"
```

---

## 2. TypeScript Implementation Options

### Option A: ripgrep Binary Wrapper (Recommended)

**What it is:** Spawn `rg` as a child process from Node.js. The `ripgrep` npm package bundles precompiled binaries for all platforms.

**Install:**
```bash
npm install ripgrep
# or use system-installed rg:
# brew install ripgrep / apt install ripgrep
```

**Pros:**
- Identical performance to command-line `rg` (sub-second on GB files)
- Multi-threaded, SIMD-accelerated, memory-mapped I/O
- Respects `.gitignore` automatically
- Handles binary files gracefully
- Battle-tested by Claude Code, Aider, and every major AI coding tool

**Cons:**
- Native binary dependency (~5MB)
- Must be spawned as child process (IPC overhead is negligible)

### Option B: Pure TypeScript — `fast-glob` + Node.js `fs`

**What it is:** Use `fast-glob` to find files matching patterns, then read and regex-match them in the Node.js event loop.

```typescript
import fg from "fast-glob";
import fs from "fs/promises";
import readline from "readline";

async function searchFiles(pattern: RegExp, glob: string): Promise<Match[]> {
  const files = await fg(glob, { dot: true, ignore: ["node_modules"] });
  const results: Match[] = [];
  
  for (const file of files) {
    const stream = fs.createReadStream(file);
    const rl = readline.createInterface({ input: stream });
    let lineNum = 0;
    for await (const line of rl) {
      lineNum++;
      if (pattern.test(line)) {
        results.push({ file, line: lineNum, text: line });
      }
    }
  }
  return results;
}
```

**Pros:**
- No native dependencies
- Works in any Node.js runtime (including restricted environments)

**Cons:**
- **50-100x slower** than ripgrep for large files
- Single-threaded (blocking the event loop for large scans)
- Doesn't respect `.gitignore` by default
- No built-in parallel directory traversal

### Option C: WASM ripgrep (Experimental)

The `@ast-grep/cli` and some experimental packages compile ripgrep's regex engine to WASM. This gives near-native speed without native binaries, but the ecosystem is immature.

---

## 3. Lightweight Search Backends Compared

| Tool | Type | Speed (1GB scan) | Dependency Size | .gitignore | Multi-threaded |
|------|------|-----------------|-----------------|------------|---------------|
| **ripgrep (rg)** | Rust binary | 0.5 sec | ~5MB binary | ✅ | ✅ |
| **fast-glob** | Pure JS glob | N/A (file listing only) | ~50KB npm | ❌ | ❌ |
| **find-in-files** | Pure JS regex | 30-60 sec | ~20KB npm | ❌ | ❌ |
| **grep-ast** | Python + tree-sitter | 2-5 sec | ~50MB (tree-sitter grammars) | ✅ (via rg) | ❌ |
| **ast-grep** | Rust binary | 0.5 sec text / 2 sec structural | ~10MB binary | ✅ | ✅ |
| **Node.js `fs` + `readline`** | Pure JS | 2-5 min | 0 (stdlib) | ❌ | ❌ |

**Recommendation:** For a CLI agent harness, **ripgrep is the only viable choice** for production use. Pure TypeScript alternatives are 50-100x slower and lack critical features like `.gitignore` respect and multithreading. Every major AI coding tool (Claude Code, Aider, Cursor, Continue) has converged on ripgrep.

---

## 4. Building a TypeScript Grep Tool

### The Production Pattern (Used by Claude Code)

```typescript
import { spawn } from "child_process";
import path from "path";

interface GrepMatch {
  file: string;
  line_number: number;
  line_text: string;
  column_start: number;
  column_end: number;
  context_before?: string[];
  context_after?: string[];
}

interface GrepOptions {
  pattern: string;
  path?: string;
  glob?: string;
  outputMode?: "content" | "files_with_matches" | "count";
  contextLines?: number;
  maxMatches?: number;
  maxChars?: number;
  caseSensitive?: boolean;
  multiline?: boolean;
}

interface GrepResult {
  matches: GrepMatch[];
  stats: {
    files_searched: number;
    matches_found: number;
    matches_returned: number;
    time_ms: number;
    truncated: boolean;
  };
  suggestions?: {
    narrower_patterns?: string[];
    broader_patterns?: string[];
    related_files?: string[];
  };
}

/**
 * Execute a ripgrep search and return structured results.
 * 
 * Uses `rg --json` for machine-parseable output with full metadata.
 */
export async function grep(options: GrepOptions): Promise<GrepResult> {
  const startTime = Date.now();
  const {
    pattern,
    path: searchPath = ".",
    glob,
    outputMode = "content",
    contextLines = outputMode === "content" ? 2 : 0,
    maxMatches = 100,
    maxChars = 50000,
    caseSensitive = false,
    multiline = false,
  } = options;

  const args: string[] = ["--json"];  // Machine-parseable JSON output

  // Case sensitivity
  if (!caseSensitive) args.push("-i");
  
  // Multiline mode (for patterns spanning multiple lines)
  if (multiline) args.push("-U", "--multiline-dotall");
  
  // Context lines
  if (contextLines > 0) args.push("-C", String(contextLines));
  
  // Match limit
  args.push("-m", String(maxMatches));
  
  // File type filter
  if (glob) args.push("-g", glob);
  
  // Mode flags
  if (outputMode === "files_with_matches") args.push("-l");
  if (outputMode === "count") args.push("-c");
  
  // Pattern and path
  args.push("-e", pattern, searchPath);

  return new Promise((resolve, reject) => {
    const rg = spawn("rg", args, {
      stdio: ["ignore", "pipe", "pipe"],
      cwd: process.cwd(),
    });

    let stdout = "";
    let stderr = "";
    let totalChars = 0;
    let truncated = false;

    rg.stdout.on("data", (chunk: Buffer) => {
      totalChars += chunk.length;
      if (totalChars > maxChars) {
        truncated = true;
        rg.kill();  // Stop if we're over the char limit
        return;
      }
      stdout += chunk.toString();
    });

    rg.stderr.on("data", (chunk: Buffer) => {
      stderr += chunk.toString();
    });

    rg.on("close", (code) => {
      // ripgrep exits with 0 on matches, 1 on no matches, 2 on error
      if (code === 2) {
        reject(new Error(`ripgrep error: ${stderr}`));
        return;
      }

      const matches = parseRipgrepJson(stdout, outputMode);
      const stats = {
        files_searched: countUniqueFiles(matches),
        matches_found: matches.length,
        matches_returned: truncated ? matches.length : matches.length,
        time_ms: Date.now() - startTime,
        truncated,
      };

      const result: GrepResult = { matches, stats };

      // Add heuristic suggestions
      if (matches.length === 0) {
        result.suggestions = {
          broader_patterns: [suggestBroadening(pattern)],
        };
      } else if (matches.length >= maxMatches) {
        result.suggestions = {
          narrower_patterns: [suggestNarrowing(pattern, matches)],
        };
      }

      resolve(result);
    });

    rg.on("error", (err) => {
      reject(new Error(`Failed to spawn ripgrep: ${err.message}`));
    });
  });
}
```

### Parsing ripgrep's JSON Output

```typescript
/**
 * Parse ripgrep --json output into structured match objects.
 * 
 * ripgrep --json emits one JSON object per line:
 *   {"type":"begin","data":{"path":{"text":"src/foo.ts"}}}
 *   {"type":"match","data":{"path":{"text":"src/foo.ts"},"lines":{"text":"match line\\n"},"line_number":42,...}}
 *   {"type":"end","data":{"path":{"text":"src/foo.ts"},"stats":{"matches":3,"elapsed":...}}}
 */
function parseRipgrepJson(
  stdout: string,
  outputMode: string
): GrepMatch[] {
  const matches: GrepMatch[] = [];
  let currentFile = "";
  const contextBefore: string[] = [];
  let collectingContext = false;

  for (const line of stdout.trim().split("\n")) {
    if (!line) continue;

    try {
      const entry = JSON.parse(line);
      
      if (entry.type === "begin") {
        currentFile = entry.data.path.text;
        contextBefore.length = 0;
        collectingContext = false;
      }
      
      if (entry.type === "match") {
        const data = entry.data;
        const match: GrepMatch = {
          file: data.path.text,
          line_number: data.line_number,
          line_text: data.lines.text.trimEnd(),
          column_start: data.submatches?.[0]?.start ?? 0,
          column_end: data.submatches?.[0]?.end ?? data.lines.text.length,
          context_before: [...contextBefore],
          context_after: [],
        };
        matches.push(match);
        collectingContext = true;
        contextBefore.length = 0;
      }
      
      if (entry.type === "context" && !collectingContext) {
        contextBefore.push(entry.data.lines.text.trimEnd());
      }
      
      if (entry.type === "context" && collectingContext) {
        // Attach to the most recent match
        const lastMatch = matches[matches.length - 1];
        if (lastMatch) {
          lastMatch.context_after = lastMatch.context_after || [];
          lastMatch.context_after.push(entry.data.lines.text.trimEnd());
        }
      }
      
      if (entry.type === "end") {
        collectingContext = false;
      }
    } catch {
      // Skip malformed JSON lines
    }
  }

  return matches;
}

function countUniqueFiles(matches: GrepMatch[]): number {
  return new Set(matches.map((m) => m.file)).size;
}
```

### Heuristic Suggestion Engine

```typescript
/**
 * Generate suggestions for next search without an LLM call.
 * These are cheap heuristics based on match count and content.
 */
function suggestBroadening(pattern: string): string {
  // If pattern has many specific values, suggest dropping some
  const parts = pattern.split("|");
  if (parts.length > 4) {
    // Keep the first 2-3 most general parts
    return parts.slice(0, 3).join("|");
  }
  // Suggest removing anchors
  if (pattern.includes("\\b") || pattern.includes("^") || pattern.includes("$")) {
    return pattern.replace(/\\b|\^|\$/g, "");
  }
  return pattern.replace(/[0-9]+/g, "[0-9]+");  // Generalize numbers
}

function suggestNarrowing(pattern: string, matches: GrepMatch[]): string {
  // Extract frequent tokens from match lines and add them
  const tokens = extractFrequentTokens(matches);
  if (tokens.length > 0) {
    return `${pattern}|${tokens.slice(0, 3).join("|")}`;
  }
  return pattern;
}

function extractFrequentTokens(matches: GrepMatch[]): string[] {
  const counts = new Map<string, number>();
  const stopWords = new Set([
    "the", "and", "for", "with", "from", "this", "that",
    "not", "are", "was", "has", "had", "will", "can"
  ]);
  
  for (const match of matches.slice(0, 50)) {
    const words = match.line_text.match(/[A-Za-z_][A-Za-z0-9_]*/g) || [];
    for (const word of words) {
      if (word.length > 3 && !stopWords.has(word.toLowerCase())) {
        counts.set(word, (counts.get(word) || 0) + 1);
      }
    }
  }
  
  return [...counts.entries()]
    .filter(([, count]) => count > 2)
    .sort(([, a], [, b]) => b - a)
    .slice(0, 10)
    .map(([token]) => token);
}
```

---

## 5. MCP Server Integration

### Building a Grep MCP Server in TypeScript

Using the `@modelcontextprotocol/sdk` package:

```bash
npm install @modelcontextprotocol/sdk ripgrep
```

```typescript
// mcp-grep-server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import { grep, type GrepOptions } from "./grep-tool.js";

const server = new Server(
  { name: "mcp-grep", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// Register available tools
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "regex_log_search",
      description: 
        "Search log files using regex patterns. " +
        "Prefer pipe-separated (|) keyword alternatives for best performance. " +
        "Use this instead of reading large log files directly.",
      inputSchema: {
        type: "object",
        properties: {
          path: {
            type: "string",
            description: "Path to search, relative to artifacts directory",
          },
          pattern: {
            type: "string",
            description: "Regex pattern. Pipe-separated keywords preferred.",
          },
          context: {
            type: "number",
            description: "Lines of context before/after each match",
            default: 3,
          },
          case_sensitive: {
            type: "boolean",
            default: false,
          },
          max_matches: {
            type: "number",
            default: 100,
          },
        },
        required: ["path", "pattern"],
      },
    },
    {
      name: "regex_code_search",
      description:
        "Search source code files using regex patterns. " +
        "Same as regex_log_search but for .sv, .v, .svh, .S, .py files.",
      inputSchema: {
        type: "object",
        properties: {
          path: { type: "string" },
          pattern: { type: "string" },
          context: { type: "number", default: 3 },
          case_sensitive: { type: "boolean", default: false },
          max_matches: { type: "number", default: 100 },
          file_types: {
            type: "array",
            items: { type: "string" },
            description: "File extensions to search",
            default: [".sv", ".svh", ".v", ".S", ".py"],
          },
        },
        required: ["path", "pattern"],
      },
    },
  ],
}));

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    if (name === "regex_log_search" || name === "regex_code_search") {
      const result = await grep({
        pattern: args.pattern,
        path: args.path,
        contextLines: args.context ?? 3,
        caseSensitive: args.case_sensitive ?? false,
        maxMatches: args.max_matches ?? 100,
        maxChars: 50000,
        glob: name === "regex_code_search"
          ? `*.{${(args.file_types ?? [".sv", ".svh", ".v", ".S", ".py"])
              .map((t: string) => t.replace(".", ""))
              .join(",")}}`
          : undefined,
      });

      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(result, null, 2),
          },
        ],
      };
    }

    throw new Error(`Unknown tool: ${name}`);
  } catch (error: unknown) {
    return {
      content: [
        {
          type: "text",
          text: JSON.stringify({
            error: error instanceof Error ? error.message : "Unknown error",
          }),
        },
      ],
      isError: true,
    };
  }
});

// Start server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch(console.error);
```

### MCP Config for the Agent Harness

```json
{
  "mcpServers": {
    "regex-search": {
      "command": "npx",
      "args": ["tsx", "mcp-grep-server.ts"]
    }
  }
}
```

---

## 6. AST-Aware Search

### When to Add AST Search

Regex finds text. AST search finds **code structures**. The difference:

| Query | Regex Result | AST Result |
|-------|-------------|------------|
| "Find x26" | Matches `x26` in comments, strings, variable names, everywhere | Matches only variable references to `x26` in executable code |
| "Find function calls" | Impossible with regex alone | `pattern: $FUNC($$$ARGS)` matches any function call |
| "Find all always_ff blocks" | `always_ff` — matches comments too | `kind: always_ff` — only actual always_ff blocks |

### ast-grep Integration

```typescript
// Add to your MCP server
import { spawn } from "child_process";

async function astGrepSearch(
  pattern: string,
  language: string,
  searchPath: string
): Promise<string> {
  return new Promise((resolve, reject) => {
    // ast-grep supports inline patterns without YAML:
    // sg -p 'pattern' -l lang path
    const sg = spawn("sg", [
      "-p", pattern,
      "-l", language,
      "--json=compact",
      searchPath,
    ]);

    let output = "";
    sg.stdout.on("data", (chunk: Buffer) => { output += chunk.toString(); });
    sg.on("close", () => resolve(output));
    sg.on("error", reject);
  });
}

// Usage: find all always_ff blocks in SystemVerilog
const result = await astGrepSearch(
  "always_ff @(posedge $$$CLK) begin $$$ end",
  "verilog",  // ast-grep uses tree-sitter-verilog
  "/home/fedora/dev/ibex"
);
```

### When ast-grep Is Worth the Complexity

- **RTL codebases** (Verilog/SystemVerilog): Regex struggles with `always @(*)` vs `always_ff`, generate blocks, parameterized modules
- **Large refactoring tasks**: "Find all module instantiations of `ibex_core`" — regex would match comments; ast-grep only matches actual instantiations
- **Cross-language search**: Same structural pattern across `.sv`, `.v`, `.svh`

**But:** For the initial `regex_log_search` tool in an RTL debug agent, **ast-grep is overkill.** The failure signature search is pure text-based — you're looking for `"UVM_FATAL @ 1856300"` in a log file, not structural patterns in RTL. Add ast-grep later for the `sv_file_outline` and `sv_references` tools, not for regex search.

---

## 7. Full Implementation Reference

### Minimal TypeScript Grep Tool (50 lines)

If you just need the core functionality without MCP:

```typescript
// grep.ts — minimal ripgrep wrapper
import { execFile } from "child_process";
import { promisify } from "util";

const execFileAsync = promisify(execFile);

export async function grep(
  pattern: string,
  filePath: string,
  contextLines = 3
): Promise<string> {
  try {
    const { stdout } = await execFileAsync("rg", [
      "-n",                          // Line numbers
      "-C", String(contextLines),    // Context
      "-e", pattern,                 // Pattern
      filePath,                      // File
    ], {
      maxBuffer: 500 * 1024,         // 500KB output max
      timeout: 30000,                // 30 second timeout
    });
    return stdout;
  } catch (error: any) {
    // rg exits with code 1 on "no matches" — this is normal
    if (error.code === 1) return "";
    throw error;
  }
}
```

### The Complete Stack

```
┌────────────────────────────────────────────────────────┐
│                  Agent Harness (TypeScript)             │
│                                                        │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐     │
│  │  MCP     │    │  Direct  │    │  AST Search  │     │
│  │  Server  │    │  spawn() │    │  (future)    │     │
│  │  (stdio) │    │          │    │              │     │
│  └────┬─────┘    └────┬─────┘    └──────┬───────┘     │
│       │               │                 │              │
│       ▼               ▼                 ▼              │
│  ┌──────────────────────────┐    ┌──────────────┐     │
│  │     ripgrep (rg)         │    │   ast-grep   │     │
│  │  - JSON output parsing   │    │   (sg)       │     │
│  │  - Context management    │    │              │     │
│  │  - Result truncation     │    │              │     │
│  │  - Heuristic suggestions │    │              │     │
│  └──────────────────────────┘    └──────────────┘     │
│                                                        │
│  npm dependencies:                                     │
│  - ripgrep (precompiled binary)                        │
│  - @modelcontextprotocol/sdk (if using MCP)            │
│  - fast-glob (for file discovery without grep)         │
└────────────────────────────────────────────────────────┘
```

### What We Can Learn from Claude Code's Approach

Claude Code's Grep tool is the gold standard because:

1. **Three output modes** — `content`, `files_with_matches`, `count` — let the LLM choose the right level of detail
2. **head_limit** — a hard cap on output prevents runaway context consumption
3. **No regex complexity** — the LLM writes simple pipe-separated patterns; ripgrep handles the optimization
4. **Dedicated tool** — deliberately NOT exposed through the Bash tool, forcing consistent formatting and sandboxing
5. **Glob filtering** — `*.ts` or `**/*.tsx` narrows the search before it starts, reducing both latency and noise

---

## Sources

- [Claude Code Tools Reference](https://code.claude.com/docs/en/tools-reference) — Official Grep tool schema
- [Claude Code Tools (Independent Analysis)](https://blog.thepete.net/claude-code-tools/) — Reverse-engineered tool definitions
- [Aider-AI/grep-ast](https://github.com/Aider-AI/grep-ast) — Tree-sitter-backed grep with AST context
- [ast-grep/ast-grep-mcp](https://github.com/ast-grep/ast-grep-mcp) — MCP server for structural code search
- [ripgrep npm package](https://www.npmjs.com/package/ripgrep) — Precompiled ripgrep binaries for Node.js
- [@modelcontextprotocol/sdk](https://www.npmjs.com/package/@modelcontextprotocol/sdk) — Official MCP TypeScript SDK
- `ideas/regex_log_search_design.md` — Our regex_log_search tool design (this doc's companion)
