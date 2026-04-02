# Knowledge Tree

> This file is the single source of truth for the skill. Keep it updated when docs are added/removed/moved.
> Format: `path | topics | description`

## Architecture
01-architecture/codebase-structure.md | architecture, modules, dependencies, monolith, service-map | Full module inventory of 38 services with file counts and dependency graph
01-architecture/bootstrap-entry-point.md | bootstrap, startup, initialization, entry-points, prefetch | 10-step boot sequence, 4 entry points (CLI/SDK/MCP/Sandbox), 10 startup modes
01-architecture/component-architecture-deep-dive.md | react, components, ui, props, rendering, component-tree | 389 React components mapped across 81,546 LOC with full prop signatures
01-architecture/hooks-system-deep-dive.md | react, hooks, state, side-effects, lifecycle | All 104 custom React hooks with signatures, dependencies, and side effects
01-architecture/ink-rendering-engine-deep-dive.md | rendering, terminal, ui, performance, buffer, diffing, ansi | 6-stage render pipeline with Int32Array cell packing and frame diffing
01-architecture/query-engine-deep-dive.md | agent-loop, streaming, async-generator, tool-injection, core-loop | The core async generator conversation loop with mid-stream tool injection
01-architecture/build-system-deep-dive.md | build, bundling, bun, feature-flags, dead-code, compilation | Bun bundler config, 88 build-time feature flags, dead code elimination
01-architecture/data-flow-and-coupling.md | data-flow, coupling, modules, dependencies | Module coupling analysis and data flow paths
01-architecture/state-migrations-output-deep-dive.md | state, migrations, persistence, schema-evolution | State management patterns and migration system
01-architecture/native-ts-reimplementations-deep-dive.md | performance, dependencies, custom-implementations | Custom TypeScript reimplementations replacing npm packages for performance
01-architecture/terminal-ui-deep-dive.md | terminal, ui, tui, interaction | Terminal UI system internals
01-architecture/ast-analysis.md | ast, static-analysis, code-structure | AST-level analysis of the codebase
01-architecture/graph-and-complexity.md | complexity, graph, metrics | Codebase complexity metrics and dependency graphs

## Extracted Knowledge
02-master-extraction/KNOWLEDGE_BASE.md | index, reference, all-systems, lookup, master-index | Master cross-reference of every subsystem — start here if unsure where to look
02-master-extraction/deep-extraction.md | findings, patterns, protocols, behaviors, exhaustive | 2,251 discrete findings across types, patterns, protocols, behaviors
02-master-extraction/claude-code-rnd-extraction.md | synthesis, appendices, comprehensive | Master synthesis with 19 appendices (A through S)
02-master-extraction/code-archaeology.md | history, dead-code, evolution, legacy | Historical patterns, dead code analysis, evolution traces
02-master-extraction/deep-dive-report.md | report, analysis, deep-dive | Deep dive analysis report
02-master-extraction/analysis-report.md | report, summary | Analysis summary report

## Prompts & Model Behavior
03-prompts/prompt-bible.md | prompts, system-prompt, instructions, prompt-engineering, cache-boundaries | Full 32K-line system prompt with 20 prompt engineering techniques, 7 static + 13 dynamic sections
03-prompts/model-behavioral-contract.md | model-behavior, invariants, tool-contracts, expectations | 24+ behavioral invariants the code expects from model output — what breaks when violated

## Core Runtime Systems
04-systems/api-client-deep-dive.md | api, streaming, providers, retry, backoff, cost-tracking | 5-provider streaming pipeline (Anthropic/Bedrock/Vertex/Foundry/Azure) with retry and cost tracking
04-systems/compaction-algorithms-deep-dive.md | compaction, context-window, memory-pressure, summarization, context-management | 3-tier compaction: microcompact, full summarization, session memory. 93% trigger threshold. Circuit breakers.
04-systems/model-selection-deep-dive.md | models, fallback, routing, selection, capability-matrix | 11 models, 5-tier fallback chain, capability matrix, model routing logic
04-systems/memdir-semantic-memory-deep-dive.md | memory, persistence, semantic-memory, knowledge-base | 7 interconnected memory subsystems architecture
04-systems/memory-extraction-pipeline-deep-dive.md | memory, extraction, pipeline, persistence | How memories are extracted from conversations and stored
04-systems/settings-policy-infrastructure-deep-dive.md | settings, policy, configuration, scope-merging, enterprise | 5-scope priority merging (managed > user > project > local > flag)
04-systems/shell-entrypoints-proxy-deep-dive.md | shell, process, proxy, entrypoints | Shell management, process proxy, entrypoint routing
04-systems/lsp-integration-deep-dive.md | lsp, language-server, ide, diagnostics | Language Server Protocol integration
04-systems/team-memory-sync-deep-dive.md | team, sync, multi-agent, shared-memory | Cross-agent memory synchronization
04-systems/prompt-suggestion-deep-dive.md | suggestions, autocomplete, ux | Suggestion engine internals
04-systems/suggestions-and-secure-storage-deep-dive.md | credentials, secure-storage, keychain, suggestions | Credential storage via OS keychain, suggestion system
04-systems/telemetry-event-catalog.md | telemetry, events, analytics, sampling, observability | 200+ event types with sampling rates and PII classification
04-systems/telemetry-event-inventory.md | telemetry, events, inventory | Full telemetry event inventory
04-systems/feature-flag-encyclopedia.md | feature-flags, build-time, runtime, gating, rollout | 88 build-time + 600+ runtime flags for safe feature rollout
04-systems/growthbook-gates-inventory.md | ab-testing, growthbook, gates, experiments | A/B testing gates and experiment conditions
04-systems/constants-encyclopedia.md | constants, thresholds, magic-numbers, configuration | All hardcoded constants, thresholds, and magic numbers
04-systems/environment-variable-contract.md | env-vars, configuration, contracts, defaults | Every environment variable with its contract and default
04-systems/environment-variable-inventory.md | env-vars, inventory | Full environment variable listing

## Security & Permissions
05-security/permissions-yolo-deep-dive.md | permissions, classifier, safety, auto-mode, yolo, security | 2-stage YOLO classifier (64-token fast pass + 4096-token reasoning at T=0). 6 permission modes. 7-stage pipeline.
05-security/bash-parser-deep-dive.md | bash, parser, command-injection, ast, security, shell-safety | Hand-rolled recursive descent parser. 15 dangerous AST types. 35+ blocked builtins. Path constraint checking.
05-security/deep-security-audit.md | security, vulnerabilities, audit, cross-system | Comprehensive cross-subsystem security analysis
05-security/oauth-lifecycle-deep-dive.md | oauth, pkce, tokens, authentication, refresh | OAuth 2.0 with PKCE, token lifecycle, refresh logic
05-security/terminal-ui-security.md | ui-security, terminal, injection | UI-layer security measures
05-security/team-memory-security.md | multi-agent, security, isolation, boundaries | Multi-agent security boundaries and isolation
05-security/desanitization-map.md | sanitization, encoding, escaping, injection | Where sanitization is applied and removed across the system
05-security/security-audit.md | security, audit, findings | Additional security findings

## Tools, Commands & Plugins
06-tools-and-plugins/command-system-deep-dive.md | commands, dispatch, execution, tools | 189 commands, 3 dispatch types (prompt/local/local-jsx), execution pipeline
06-tools-and-plugins/plugin-system-deep-dive.md | plugins, lifecycle, dependencies, marketplace, mcp | 6-phase plugin lifecycle with DFS dependency resolution and homograph detection
06-tools-and-plugins/tool-contract-inventory.md | tools, schemas, contracts, input-output | Every tool's input/output contract and validation rules
06-tools-and-plugins/computer-use-deep-dive.md | computer-use, screen-capture, input, automation | Screen capture, input handling, action sequences
06-tools-and-plugins/claude-in-chrome-deep-dive.md | browser, chrome, extension, dom | Chrome extension integration with tab/DOM/JS context management
06-tools-and-plugins/bundled-skills-catalog.md | skills, built-in, catalog | Built-in skill definitions and trigger conditions
06-tools-and-plugins/buddy-system-deep-dive.md | pair-programming, collaboration, buddy | Pair programming system
06-tools-and-plugins/keybindings-system.md | keybindings, shortcuts, input | Keyboard shortcut system
06-tools-and-plugins/command-tool-skill-surface.md | surface-area, commands, tools, skills | Full command/tool/skill surface area mapping
06-tools-and-plugins/utility-subsystems-deep-dive.md | utilities, helpers, services | Supporting utility modules
06-tools-and-plugins/small-services-deep-dive.md | services, small, supporting | Smaller service modules
06-tools-and-plugins/unmapped-systems.md | unmapped, undocumented | Systems not yet fully documented
06-tools-and-plugins/plugin-marketplace-contract.md | marketplace, distribution, contracts | Plugin marketplace integration contracts
06-tools-and-plugins/computer-use-protocols.md | computer-use, protocols | Computer use protocol specifications

## Agent Orchestration
07-agents/swarm-orchestration-deep-dive.md | swarm, multi-agent, coordination, ipc, leader-follower | Leader-follower architecture, file-based mailbox IPC, 3 execution backends (tmux/iTerm2/in-process)
07-agents/session-teleport-deep-dive.md | session, teleport, state-transfer, git-bundle | Cross-environment state preservation via git bundles
07-agents/coordinator-voice-moreright-deep-dive.md | coordinator, voice, team, policy | Team coordination, voice pipeline, policy enforcement
07-agents/swarm-bridge-sdk.md | sdk, bridge, swarm, communication | SDK for swarm communication
07-agents/remote-bridge-protocol.md | remote, bridge, protocol, session | Remote session management protocol
07-agents/daemon-proactive-agent.md | daemon, background, proactive, agent | Background/proactive agent capabilities

## History & Evolution
08-history/migration-product-history.md | migrations, history, evolution, versioning | 11 shipped migrations, permission system evolution, model aliasing changes
08-history/community-osint.md | community, research, external, osint | External research and community findings
08-history/Wriction-matrix.md | attribution, contribution | Contribution tracking

## Diagrams (SVG)
infographics/architecture-overview.svg | diagram, architecture, overview | Full system architecture visual
infographics/bootstrap-sequence.svg | diagram, bootstrap, startup | 10-step startup sequence visual
infographics/ink-rendering-pipeline.svg | diagram, rendering, pipeline | 6-stage rendering pipeline visual
infographics/api-client-pipeline.svg | diagram, api, streaming | API request flow visual
infographics/memory-systems.svg | diagram, memory | 7 memory subsystems visual
infographics/compaction-state-machine.svg | diagram, compaction, state-machine | Compaction state machine visual
infographics/compaction-tiers.svg | diagram, compaction, tiers | Compaction tier decision tree visual
infographics/permission-engine.svg | diagram, permissions, security | 7-stage permission pipeline visual
infographics/yolo-classifier-pipeline.svg | diagram, yolo, classifier | 2-stage YOLO classifier visual
infographics/command-dispatch.svg | diagram, commands, dispatch | Command dispatch flow visual
infographics/plugin-lifecycle.svg | diagram, plugins, lifecycle | Plugin 6-phase lifecycle visual
infographics/streaming-tool-executor.svg | diagram, tools, streaming | Concurrent tool execution visual
infographics/swarm-orchestration.svg | diagram, swarm, agents | Multi-agent swarm visual
infographics/agent-loop-flow.svg | diagram, agent-loop | Core agent loop visual
infographics/system-prompt-ordering.svg | diagram, prompts, ordering | Prompt section ordering visual
