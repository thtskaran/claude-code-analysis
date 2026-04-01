# Claude Code v2.1.88 — Complete Architecture Analysis

**The definitive reverse-engineering documentation of Anthropic's AI-powered CLI IDE.**

This comprehensive analysis covers 82 documents totaling 112,000+ lines across 15 architectural diagrams. Every system, hook, component, tool, and security mechanism is documented in encyclopedic detail.

---

## Architecture Map

![Architecture Overview](infographics/architecture-overview.svg)

---

## Section Directory

### 01 — Architecture (14 documents)

The foundational layer covering structural patterns, component design, rendering pipeline, and system bootstrap.

**Core Documents:**

- [codebase-structure.md](01-architecture/codebase-structure.md) — Module inventory and layer decomposition. Maps the physical filesystem structure to logical system boundaries.
- [hooks-system-deep-dive.md](01-architecture/hooks-system-deep-dive.md) — All 104 React hooks across 8 domain categories (2,306 lines). The definitive guide to input, voice, permissions, notifications, and team coordination.
- [component-architecture-deep-dive.md](01-architecture/component-architecture-deep-dive.md) — Complete inventory of 389 React components across 5 subsystems (1,695 lines).
- [ink-rendering-engine-deep-dive.md](01-architecture/ink-rendering-engine-deep-dive.md) — The 6-stage terminal rendering pipeline: State → Virtual DOM → Reconciliation → Layout → Yoga → Terminal Output (2,351 lines).
- [build-system-deep-dive.md](01-architecture/build-system-deep-dive.md) — Bun bundler configuration, 88 feature flags, and the source map leak (1,265 lines).
- [bootstrap-entry-point.md](01-architecture/bootstrap-entry-point.md) — The 10-step startup sequence from CLI invocation through initialization to REPL readiness.
- [query-engine-deep-dive.md](01-architecture/query-engine-deep-dive.md) — The core agent loop executing LLM queries, tool calls, streaming responses, and state updates.
- [terminal-ui-deep-dive.md](01-architecture/terminal-ui-deep-dive.md) — Terminal UI rendering, viewport management, and interactive component lifecycle.
- [native-ts-reimplementations-deep-dive.md](01-architecture/native-ts-reimplementations-deep-dive.md) — TypeScript reimplementations of Ink.js native terminal functions.
- [state-migrations-output-deep-dive.md](01-architecture/state-migrations-output-deep-dive.md) — State persistence, migrations between versions, and output serialization.
- [ast-analysis.md](01-architecture/ast-analysis.md) — Abstract syntax tree structure analysis for code introspection systems.
- [data-flow-and-coupling.md](01-architecture/data-flow-and-coupling.md) — Inter-component data dependencies and coupling analysis.
- [graph-and-complexity.md](01-architecture/graph-and-complexity.md) — Dependency graph visualization and cyclomatic complexity metrics.
- [DOCUMENT-INDEX.md](01-architecture/DOCUMENT-INDEX.md) — Detailed index of all documents in this section with metadata.

---

### 02 — Master Extraction (6 documents)

High-level synthesis and knowledge consolidation across the entire codebase.

- [claude-code-rnd-extraction.md](02-master-extraction/claude-code-rnd-extraction.md) — Master extraction document with 19 appendices (A-S) covering all major systems, design patterns, and architectural decisions. The starting point for newcomers.
- [KNOWLEDGE_BASE.md](02-master-extraction/KNOWLEDGE_BASE.md) — Consolidated knowledge base distilling key insights from all analysis documents.
- [deep-dive-report.md](02-master-extraction/deep-dive-report.md) — Deep-dive findings and technical highlights across systems.
- [deep-extraction.md](02-master-extraction/deep-extraction.md) — Semantic-level extraction of design patterns, algorithms, and architectural principles.
- [analysis-report.md](02-master-extraction/analysis-report.md) — Comprehensive analysis report with findings and recommendations.
- [code-archaeology.md](02-master-extraction/code-archaeology.md) — Historical code patterns, legacy systems, and evolution analysis.

---

### 03 — Prompts (2 documents)

System prompts and model behavioral contracts that shape agent behavior.

- [prompt-bible.md](03-prompts/prompt-bible.md) — Complete system prompt collection spanning 32K+ lines. Every prompt section, conditional modifier, and behavioral instruction is documented with context and justification.
- [model-behavioral-contract.md](03-prompts/model-behavioral-contract.md) — Model behavior specifications, instruction precedence, output constraints, and safety boundaries.

---

### 04 — Systems (19 documents)

Mission-critical subsystems implementing core functionality.

- [api-client-deep-dive.md](04-systems/api-client-deep-dive.md) — Streaming request pipeline with retry logic, exponential backoff, and multi-provider factory pattern (2,165 lines). Covers Anthropic API, OpenAI, Bedrock, and Google provider implementations.
- [compaction-algorithms-deep-dive.md](04-systems/compaction-algorithms-deep-dive.md) — Six compaction strategies for token limit management: deletion, summarization, context windowing, agent delegation, archival, and smart recompression (1,497 lines).
- [model-selection-deep-dive.md](04-systems/model-selection-deep-dive.md) — Model selection logic across 11 models and 4 providers with cost, latency, and capability tradeoffs (1,092 lines).
- [shell-entrypoints-proxy-deep-dive.md](04-systems/shell-entrypoints-proxy-deep-dive.md) — Shell detection, proxy tunnel construction, environment variable bridging, and subprocess management (1,663 lines).
- [settings-policy-infrastructure-deep-dive.md](04-systems/settings-policy-infrastructure-deep-dive.md) — Settings and policy management system for user preferences and organizational controls.
- [lsp-integration-deep-dive.md](04-systems/lsp-integration-deep-dive.md) — Language Server Protocol integration for IDE features and code intelligence.
- [memdir-semantic-memory-deep-dive.md](04-systems/memdir-semantic-memory-deep-dive.md) — Persistent semantic memory system with automatic vector embeddings and retrieval.
- [memory-extraction-pipeline-deep-dive.md](04-systems/memory-extraction-pipeline-deep-dive.md) — Automatic extraction of relevant context from chat history into long-term memory.
- [team-memory-sync-deep-dive.md](04-systems/team-memory-sync-deep-dive.md) — Synchronization of memory across team members and shared contexts.
- [feature-flag-encyclopedia.md](04-systems/feature-flag-encyclopedia.md) — Complete inventory of 88 feature flags with conditions, rollout percentages, and dependencies.
- [constants-encyclopedia.md](04-systems/constants-encyclopedia.md) — All constants used throughout the system: timeouts, thresholds, limits, and configuration values.
- [environment-variable-contract.md](04-systems/environment-variable-contract.md) — Environment variable specifications and requirements.
- [environment-variable-inventory.md](04-systems/environment-variable-inventory.md) — Complete enumeration of all environment variables.
- [telemetry-event-catalog.md](04-systems/telemetry-event-catalog.md) — Schema and taxonomy of all telemetry events.
- [telemetry-event-inventory.md](04-systems/telemetry-event-inventory.md) — Complete inventory of telemetry events by category.
- [growthbook-gates-inventory.md](04-systems/growthbook-gates-inventory.md) — GrowthBook experimentation gates and rollout configurations.
- [settings-policy-schema.md](04-systems/settings-policy-schema.md) — Complete schema definition for settings and policy objects.
- [prompt-suggestion-deep-dive.md](04-systems/prompt-suggestion-deep-dive.md) — Prompt suggestion engine for guided interaction patterns.
- [suggestions-and-secure-storage-deep-dive.md](04-systems/suggestions-and-secure-storage-deep-dive.md) — Suggestion system and secure storage mechanisms for sensitive data.

---

### 05 — Security (8 documents)

Comprehensive security architecture and vulnerability analysis.

- [permissions-yolo-deep-dive.md](05-security/permissions-yolo-deep-dive.md) — Two-stage YOLO permission classifier with 6 permission modes and hierarchical decision flow (1,041 lines). Covers auto-approval heuristics, user prompts, and swarm leader delegation.
- [bash-parser-deep-dive.md](05-security/bash-parser-deep-dive.md) — Hand-rolled bash parser for command validation with fail-closed design and comprehensive test coverage (1,399 lines).
- [deep-security-audit.md](05-security/deep-security-audit.md) — Comprehensive security audit covering all attack vectors, mitigations, and threat modeling.
- [security-audit.md](05-security/security-audit.md) — Security audit findings and recommendations.
- [oauth-lifecycle-deep-dive.md](05-security/oauth-lifecycle-deep-dive.md) — OAuth 2.0 authentication flow, token management, and credential storage.
- [desanitization-map.md](05-security/desanitization-map.md) — Patterns for reversible data desanitization in specific contexts.
- [team-memory-security.md](05-security/team-memory-security.md) — Security considerations for shared team memory contexts.
- [terminal-ui-security.md](05-security/terminal-ui-security.md) — Terminal UI security including input validation and output sanitization.

---

### 06 — Tools & Plugins (14 documents)

Tools, commands, plugins, and extensibility mechanisms.

- [command-system-deep-dive.md](06-tools-and-plugins/command-system-deep-dive.md) — Comprehensive command system covering approximately 150 slash commands across 3 execution types (1,850 lines).
- [plugin-system-deep-dive.md](06-tools-and-plugins/plugin-system-deep-dive.md) — Complete plugin lifecycle from discovery through execution to delisting (1,279 lines).
- [tool-contract-inventory.md](06-tools-and-plugins/tool-contract-inventory.md) — Tool contracts and execution signatures for all integrated tools.
- [command-tool-skill-surface.md](06-tools-and-plugins/command-tool-skill-surface.md) — Surface area mapping of commands, tools, and skills.
- [computer-use-deep-dive.md](06-tools-and-plugins/computer-use-deep-dive.md) — Computer use capabilities and protocols for desktop automation.
- [computer-use-protocols.md](06-tools-and-plugins/computer-use-protocols.md) — Protocol specifications for computer use interactions.
- [claude-in-chrome-deep-dive.md](06-tools-and-plugins/claude-in-chrome-deep-dive.md) — Chrome extension architecture and browser automation capabilities.
- [buddy-system-deep-dive.md](06-tools-and-plugins/buddy-system-deep-dive.md) — Companion sprite system for persistent contextual assistance.
- [bundled-skills-catalog.md](06-tools-and-plugins/bundled-skills-catalog.md) — Inventory and documentation of built-in skills.
- [keybindings-system.md](06-tools-and-plugins/keybindings-system.md) — Keyboard binding system and customization mechanisms.
- [plugin-marketplace-contract.md](06-tools-and-plugins/plugin-marketplace-contract.md) — Marketplace API contract for plugin distribution.
- [small-services-deep-dive.md](06-tools-and-plugins/small-services-deep-dive.md) — Architectural analysis of small utility services.
- [utility-subsystems-deep-dive.md](06-tools-and-plugins/utility-subsystems-deep-dive.md) — Utility subsystems supporting tools and plugins.
- [unmapped-systems.md](06-tools-and-plugins/unmapped-systems.md) — Inventory of remaining unmapped areas requiring further analysis.

---

### 07 — Agents (6 documents)

Agent orchestration, coordination, and distributed execution.

- [swarm-orchestration-deep-dive.md](07-agents/swarm-orchestration-deep-dive.md) — Leader-follower orchestration pattern for distributed agent coordination via file-based IPC.
- [coordinator-voice-moreright-deep-dive.md](07-agents/coordinator-voice-moreright-deep-dive.md) — Coordinator system and voice interaction mechanisms.
- [daemon-proactive-agent.md](07-agents/daemon-proactive-agent.md) — Background daemon agents performing proactive work without user prompting.
- [remote-bridge-protocol.md](07-agents/remote-bridge-protocol.md) — Protocol for bridging to remote Claude.ai session via persistent WebSocket.
- [session-teleport-deep-dive.md](07-agents/session-teleport-deep-dive.md) — Session teleportation enabling migration between environments.
- [swarm-bridge-sdk.md](07-agents/swarm-bridge-sdk.md) — SDK for building swarm bridge integrations.

---

### 08 — History (3 documents)

Product evolution, attribution, and community intelligence.

- [migration-product-history.md](08-history/migration-product-history.md) — Product evolution timeline documenting feature development, architectural decisions, and version migrations.
- [Wriction-matrix.md](08-history/Wriction-matrix.md) — Attribution and contribution tracking across subsystems.
- [community-osint.md](08-history/community-osint.md) — Community intelligence including public discussions, user feedback, and ecosystem analysis.

---

## Infographics

15 architectural diagrams documenting key system flows, pipelines, and decision trees.

| Diagram | Description | File |
|---------|-------------|------|
| Architecture Overview | Full 8-layer system architecture | [View](infographics/architecture-overview.svg) |
| Agent Loop Flow | Core agent execution cycle with state management | [View](infographics/agent-loop-flow.svg) |
| API Client Pipeline | Request lifecycle with retry, exponential backoff, and streaming | [View](infographics/api-client-pipeline.svg) |
| Bootstrap Sequence | 10-step startup timeline from CLI invocation to REPL ready | [View](infographics/bootstrap-sequence.svg) |
| Command Dispatch | Slash command parsing to execution with context binding | [View](infographics/command-dispatch.svg) |
| Compaction State Machine | 6-stage token compaction pipeline | [View](infographics/compaction-state-machine.svg) |
| Compaction Tiers | Token threshold tiers for automatic compaction triggering | [View](infographics/compaction-tiers.svg) |
| Ink Rendering Pipeline | 6-stage terminal rendering with reconciliation and layout | [View](infographics/ink-rendering-pipeline.svg) |
| Memory Systems | All 7 memory subsystems and their interactions | [View](infographics/memory-systems.svg) |
| Permission Engine | Two-stage YOLO decision tree with escalation paths | [View](infographics/permission-engine.svg) |
| Plugin Lifecycle | Discovery, installation, execution, and delisting | [View](infographics/plugin-lifecycle.svg) |
| Streaming Tool Executor | Concurrent tool execution with streaming output | [View](infographics/streaming-tool-executor.svg) |
| Swarm Orchestration | Leader-follower IPC coordination pattern | [View](infographics/swarm-orchestration.svg) |
| System Prompt Ordering | Prompt section hierarchy and conditional application | [View](infographics/system-prompt-ordering.svg) |
| YOLO Classifier Pipeline | Two-stage automatic permission classification | [View](infographics/yolo-classifier-pipeline.svg) |

---

## Methodology

This analysis employs systematic reverse-engineering across multiple dimensions:

See [METHODOLOGY.md](METHODOLOGY.md) for complete documentation of research approaches, verification strategies, and analysis validation.

**Key Approaches:**

- Comprehensive file enumeration and structural mapping
- Deep reading of critical implementations
- Dependency graph tracing and coupling analysis
- Constant extraction and threshold cataloging
- Security analysis with threat modeling
- Integration pattern recognition
- Code archaeology tracking historical evolution

---

## Statistics

| Metric | Value |
|--------|-------|
| **Total Documents** | 82 |
| **Total Lines** | 112,000+ |
| **Architecture Documents** | 14 |
| **System Documents** | 19 |
| **Security Documents** | 8 |
| **Tools & Plugins Documents** | 14 |
| **Agent Documents** | 6 |
| **Extraction Documents** | 6 |
| **Prompt Documents** | 2 |
| **History Documents** | 3 |
| **Architectural Diagrams** | 15 |
| **React Hooks Documented** | 104 |
| **React Components Documented** | 389 |
| **Commands Documented** | 150+ |
| **Feature Flags Documented** | 88 |
| **Telemetry Events Documented** | 50+ |
| **Environment Variables Documented** | 30+ |
| **Code Examples** | 500+ |
| **Tables & Matrices** | 100+ |

---

## Quick Navigation

**First-Time Users:** Start with [Master Extraction](02-master-extraction/claude-code-rnd-extraction.md)

**Understanding the Agent Loop:** Read [Query Engine Deep Dive](01-architecture/query-engine-deep-dive.md) → [API Client Pipeline](04-systems/api-client-deep-dive.md) → [Swarm Orchestration](07-agents/swarm-orchestration-deep-dive.md)

**Building Tools:** [Command System](06-tools-and-plugins/command-system-deep-dive.md) → [Plugin System](06-tools-and-plugins/plugin-system-deep-dive.md) → [Tool Contracts](06-tools-and-plugins/tool-contract-inventory.md)

**Security Deep-Dive:** [Permissions YOLO](05-security/permissions-yolo-deep-dive.md) → [Bash Parser](05-security/bash-parser-deep-dive.md) → [Security Audit](05-security/deep-security-audit.md)

**Terminal Rendering:** [Bootstrap Entry Point](01-architecture/bootstrap-entry-point.md) → [Ink Rendering Engine](01-architecture/ink-rendering-engine-deep-dive.md) → [Terminal UI](01-architecture/terminal-ui-deep-dive.md)

**Memory Systems:** [Memory Extraction Pipeline](04-systems/memory-extraction-pipeline-deep-dive.md) → [Team Memory Sync](04-systems/team-memory-sync-deep-dive.md) → [Memdir](04-systems/memdir-semantic-memory-deep-dive.md)

**React Architecture:** [Hooks System](01-architecture/hooks-system-deep-dive.md) → [Components](01-architecture/component-architecture-deep-dive.md) → [State Patterns](01-architecture/state-migrations-output-deep-dive.md)

---

## About Claude Code v2.1.88

Claude Code is Anthropic's AI-powered command-line IDE combining:

- **Terminal UI** built on Ink.js for rich interactive experiences
- **React State Management** with 104 custom hooks coordinating all subsystems
- **LLM Agent Loop** executing Claude queries, tool calls, and streaming responses
- **Distributed Agents** via file-based mailbox IPC for team coordination
- **Voice Input** with hold-to-talk STT via Anthropic voice_stream API
- **File Indexing** via Rust native modules for O(n) fuzzy matching
- **Permission System** with two-stage YOLO auto-classifier and hierarchical escalation
- **Plugin Ecosystem** for extensibility with 150+ built-in commands
- **Memory Systems** with semantic embeddings and team synchronization
- **Settings & Policies** for organizational control and compliance

The architecture elegantly layers these systems with clean separation of concerns while maintaining tight coordination for responsive user interaction.

---

**Generated:** 2026-04-02
**Scope:** Claude Code v2.1.88 Reverse-Engineering Analysis
**Coverage:** Complete architectural documentation with code examples and security analysis
**Status:** Comprehensive analysis of public binary and reverse-engineered behavior
