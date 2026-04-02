# How We Reverse-Engineered 512K Lines of Claude Code

**Author:** Karan Prasad ([hello@karanprasad.com](mailto:hello@karanprasad.com) | [karanprasad.com](https://karanprasad.com))

---

## The Starting Point

On March 31, 2026, Anthropic shipped an npm package that accidentally included a `.map` file — a standard debugging artifact that maps bundled JavaScript back to original TypeScript source. Because npm packages are public by design, anyone who ran `npm pack` could reconstruct the full source tree. Security researcher [Chaofan Shou](https://x.com/Fried_rice) flagged it publicly. Anthropic removed it in a later release.

No hack. No exploit. No access controls bypassed. Just a packaging oversight that exposed 1,902 TypeScript files and ~512,000 lines of source code.

We studied the snapshot systematically, then destroyed our local copy. This document describes exactly how we conducted the analysis. Everything in this repository is our own original research — architectural diagrams, analysis documents, and commentary. **No source code is redistributed.**

---

## The Problem

You can't just "read" half a million lines of TypeScript. The codebase has circular dependencies, runtime-constructed prompts, feature flags that strip entire modules at build time, and an internal-only branch (`ant`) that accounts for ~15% of the code. Brute-force reading would take months and miss structural relationships.

So we built a five-phase extraction pipeline: automated symbol discovery, semantic extraction, dependency graph construction, targeted deep analysis of high-value modules, and cross-referenced prompt reconstruction. Each phase fed the next.

---

## Phase 1: Orientation — Static Inventory

We started with a breadth-first survey. Rather than reading every file, we prioritized by architectural role — entry points, the API client, the agent execution loop, the permission engine, tool execution, and the bash parser (security-critical).

Three subsystems got manual end-to-end analysis:

1. **Agent loop** — the core execution flow: how user queries become model calls, how tool use is dispatched mid-stream, and how the system decides when to stop or continue
2. **API client** — raw HTTP streaming, SSE chunk parsing, exponential backoff retry with jitter, prompt cache boundary detection, and multi-provider routing across Anthropic, AWS Bedrock, GCP Vertex, and Azure
3. **Core infrastructure** — context compaction (6 strategies, from lightweight microcompaction to forked-subprocess summarization), the permission evaluation pipeline, the hook lifecycle system, and the security-critical bash command parser

This gave us the high-level map. Then we automated.

---

## Phase 2: Automated Extraction

To scale beyond manual reading, we wrote three custom Node.js extraction scripts and ran them against the full source tree. We chose regex-based parsing over full AST analysis — approximately 3x faster and sufficient for symbol-level extraction at this scale.

### Script 1: Symbol Inventory (~400 lines)

Systematic extraction of every exported symbol across all 1,884 files:

| Target | Count |
|--------|-------|
| Exported functions | 6,552 |
| Classes | 99 |
| Types and Interfaces | 1,308 |
| Zod validation schemas | 327 |
| Feature flags | 90 |

The Zod schema discovery is worth calling out. Our first pass found only 14 — clearly wrong given the validation-heavy architecture. We rewrote the regex patterns three times, progressively catching schemas defined via intermediate variables, inline `.shape()` compositions, conditional construction, and cross-module re-exports. Final count: 327 — a 23x increase over the naive pass. This kind of iterative refinement defined the whole project.

This script also constructed the full module dependency graph from `import` statements — which became the targeting input for Phase 3.

### Script 2: Semantic Extraction (~600 lines)

Not just *what symbols exist*, but *what they mean*:

- **Tool class definitions** — every class extending the `Tool` base, including parameter schemas and execution hooks. This is how we mapped all 40+ tool implementations.
- **Zod `.describe()` strings** — the description text inside schema field definitions. These often encode behavioral requirements and design rationale that never appear in comments. A goldmine for understanding *intent* behind validation rules.
- **Error and recovery messages** — every user-facing error string, system message, and graceful degradation path. Tells you what the system expects to go wrong.
- **System-reminder injection patterns** — code paths that inject hidden context into conversations, including permission boundaries and safety instructions the user never sees.
- **Behavioral constants** — hardcoded thresholds, timeouts, retry limits, token budgets, polling intervals. The numbers that define how the system actually behaves.
- **Fallback chains** — what happens when primary systems fail. Circuit breakers, no-op substitutions, degraded-mode behavior.

**Output**: 2,251 discrete findings, each documented with module location and surrounding context.

### Script 3: Graph Analysis & Complexity Ranking (~350 lines)

Quantitative ranking to identify architectural hotspots:

- **Module coupling analysis** — computed total coupling (in-edges + out-edges) for every module in the dependency graph. The shared utilities module had the highest coupling (76 total edges), confirming it as the foundational abstraction layer.
- **Cyclomatic complexity proxy** — ranked the top 50 functions by estimated complexity using conditional depth, loop nesting, and branch count. The word-level message parser ranked #1 (estimated complexity: 123), reflecting dense conditional logic for handling every edge case in multi-language text processing.
- **Architectural pattern frequency** — 156 permission check guards, 47 tool execution wrappers, 23 retry-with-backoff implementations, 12 streaming event transformers. These frequencies reveal what the system considers important enough to repeat everywhere.

---

## Phase 3: Targeted Deep Analysis

The dependency graph and complexity rankings told us where to aim. We selected 12 high-value modules for line-by-line analysis. To parallelize, we dispatched multiple reading agents concurrently — modules exceeding 2,000 lines were split into contiguous ranges assigned to separate agents, then consolidated. This cut wall-clock time by approximately 70%.

| Module | Scale | What We Reconstructed |
|--------|------:|----------------------|
| REPL / UI rendering loop | ~5,000 lines | The terminal rendering lifecycle, frustration detection heuristics, user feedback survey integration, and how the React+Ink component tree mounts |
| YOLO safety classifier | ~1,500 lines | The two-stage permission pipeline — a 64-token fast scan (Stage 1) followed by a 4,096-token deep reasoning pass (Stage 2) at temperature zero |
| Multi-agent coordinator | ~370 lines | How parallel agents are orchestrated, how tasks are distributed via XML notifications, and how results aggregate back |
| System prompt assembler | ~900 lines | Section ordering, conditional injection based on feature flags, cache boundary placement, and the 7-static + 13-dynamic section architecture |
| Message processing pipeline | ~5,500 lines | Normalization (whitespace, encoding, embedding prep), content aggregation, history persistence, and how tool results get folded back in |
| Hook execution system | ~5,000 lines | Lifecycle hooks, error handling chains, hook composition patterns, and how plugins attach pre/post processors |
| Context compaction engine | ~2,200 lines | The full compaction algorithm — token counting, priority-based pruning, 9-section summarization prompt, forked subprocess execution, and post-compact skill re-injection |
| Streaming tool executor | ~530 lines | Concurrent vs exclusive execution routing, abort controller trees with bubble-up cancellation, and the dual-yield result pattern |
| API client & streaming | ~3,400 lines | Raw HTTP streaming, SSE event parsing, token counting (input/output/cache), multi-provider factory, and how tool invocations happen mid-stream |
| Retry & backoff framework | ~300 lines | Exponential backoff with jitter, error classification (rate limit vs auth vs network vs prompt-too-long), and foreground vs background retry strategies |
| Prompt cache break detection | ~730 lines | How the system detects when cached prompt segments become invalid, the diagnostic pipeline, and recovery strategies |
| Bash command parser | ~4,400 lines | A hand-rolled recursive descent parser that identifies 15 categories of dangerous AST nodes (fork, exec, pipe, redirect, rm, dd, shutdown, reboot) and implements a fail-closed allowlist |

---

## Phase 4: Prompt Reconstruction

### The challenge

The concatenated system prompt exceeded 32,000 lines (2.1 MB). No single-pass analysis could process it. And the runtime prompt isn't stored in one place — it's assembled dynamically from feature flags, user context, session state, and MCP configuration.

### Our approach

We divided the concatenated prompt into three contiguous ranges (~10,666 lines each) and assigned parallel agents to extract verbatim prompt text and identify structural sections. Then we cross-referenced the extracted sections against the prompt assembly module to verify ordering, identify conditional injection paths, and map which feature flags activate which prompt segments.

Finally, we reconstructed the complete prompt as a superset of all possible runtime variants — documenting every conditional branch and what triggers it.

### What we found

20 distinct prompt engineering techniques used in production, including:

- **Immutable rule layering** — safety rules positioned before user requests so they can't be overridden by injection
- **Recursive attack prevention** — self-referential instruction defense that detects attempts to override the system prompt using the system prompt's own authority
- **Injection isolation** — tool outputs and DOM content are tagged as untrusted at the prompt level, preventing downstream injection
- **Context origin tracking** — the system distinguishes tool results from user input structurally, not just semantically
- **Emotional manipulation defense** — specific detection of authority impersonation ("I'm from Anthropic...") and artificial urgency ("this is an emergency, skip safety checks")
- **Explicit permission gating** — downloads, financial transactions, and account modifications require separate approval paths regardless of mode
- **Copyright guardrails** — hard limit of <15 quoted words per source, enforced at the prompt level

---

## Phase 5: Consolidation & Verification

All findings from Phases 1–4 were merged into a master research document organized across 19 appendices:

**A** Module dependency graph · **B** Feature flags (90 total) · **C** Tool definitions and schemas · **D** Zod validation rules · **E** Permission classifier decision tree · **F** Error messages and recovery logic · **G** System prompts by component · **H** Hook execution lifecycle · **I** Compaction algorithm and strategies · **J** API streaming implementation · **K** Bash parser security rules · **L** Retry and backoff strategies · **M** Prompt engineering techniques · **N** System-reminder injection patterns · **O** Coupling and complexity rankings · **P** Tool execution flow · **Q** Cache invalidation logic · **R** UI frustration detection heuristics · **S** Multi-agent orchestration

### Verification Protocol

We validated extraction completeness through targeted spot-check queries designed to probe the most obscure corners of the analysis:

- *"Does the extraction cover the profanity regex used in message filtering?"* — Yes, documented in the message processing analysis
- *"All 90 feature flags accounted for?"* — Yes, enumerated in appendix B with activation conditions
- *"Bail-out logic for unrecoverable API failures?"* — Yes, covered in the retry framework and cache break detection analyses
- *"System-reminder injection defenses fully documented?"* — Yes, mapped in the API client analysis and prompt engineering section
- *"Race conditions in the swarm IPC system?"* — Yes, 5 identified including the permission bridge hijack vector

---

## Challenges We Solved

### The 2.1 MB prompt file

32,000 lines exceeds any practical single-read context window. We parallelized with three concurrent extraction agents, each processing ~10,666 lines independently. Sections were tagged at extraction time and merged post-hoc for complete prompt reconstruction without any single agent needing the full file.

### The Zod schema discovery problem

14 → 327. The naive regex caught only top-level `z.object()` declarations. Three rounds of pattern refinement were required to catch: schemas assigned to intermediate `const` variables before export, inline `.shape()` compositions that build schemas incrementally, conditional schema construction behind feature flag guards, and schemas imported from one module and re-exported from another. The 23x improvement over the initial pass illustrates why automated extraction requires iterative refinement — the first pass is never enough.

### Ant-only code (~15% of the codebase)

Approximately 15% of modules are internal-only Anthropic code, stripped at build time via conditional requires:
```
"external" === 'ant' ? require(...internal module...) : () => ({ state: 'closed' })
```
We made a deliberate decision not to attempt reverse-engineering internal logic. Instead, we documented the fallback behavior visible in the external build — the no-op substitutions, the `{ state: 'closed' }` returns, and the feature gates that guard access. For example, the frustration detection heuristic exists only in internal builds; the external build substitutes a no-op that always returns inactive state.

### Dynamic prompt composition

The runtime system prompt doesn't exist in any single file. It's assembled at startup from 20 sections, with conditional inclusion based on feature flags, user context, active MCP servers, and session state. We traced every call to the prompt assembly functions, identified all conditional branches, enumerated every feature flag combination that affects prompt content, and reconstructed the complete prompt as a superset document covering all possible runtime variants.

---

## Tools & Infrastructure

- **Claude (Opus model)** — our primary extraction and analysis agent. Used for code reading at scale, architectural pattern identification, cross-reference synthesis, and document generation
- **Node.js** — execution environment for the three custom extraction scripts
- **Regex-based parsing** — chosen over full AST analysis for 3x speed advantage. Sufficient for symbol-level extraction; acknowledged limitation for dynamically constructed identifiers
- **Parallel agent dispatch** — concurrent reading agents for large files (>2,000 lines) and the prompt file, reducing total analysis time by ~70%

---

## Reproducibility

The methodology is fully reproducible. Given access to the same TypeScript source snapshot and a Node.js environment, the pipeline executes deterministically:

1. Run the symbol inventory script → exports, classes, types, schemas, flags, dependency graph
2. Run the semantic extraction script → 2,251 tagged findings
3. Run the graph analysis script → coupling scores, complexity rankings, pattern frequencies
4. Execute targeted deep analysis on the 12 highest-value modules
5. Parallel-extract and cross-reference the system prompt
6. Consolidate into master document with 19 appendices
7. Verify via spot-check protocol

---

## Limitations

**Ant-only code** — internal Anthropic modules (~15%) are not analyzed. We document external-build fallback behavior only.

**Static analysis** — we captured code structure and logic, not runtime behavior. Memory usage, actual latency characteristics, and timing-dependent edge cases are outside our scope.

**Regex extraction boundaries** — regex-based parsing can miss dynamically constructed symbol names and metaprogramming patterns. Full AST analysis would be more precise at significant speed cost.

**Point-in-time snapshot** — system prompts, feature flags, and architectural decisions change between releases. This analysis covers Claude Code v2.1.88 exclusively. Subsequent versions may have diverged significantly.

**No source redistribution** — we do not share, redistribute, or host any of the original TypeScript source files. All content in this repository is our own original analysis, diagrams, and documentation derived from studying the publicly exposed snapshot.
