# Claude Code Interaction Matrix

This document is a subsystem-coupling map. It is meant to be readable without the original codebase.

Legend:
- `✓` direct, intentional coupling with a visible guard
- `⚠` direct coupling with stateful or high-blast-radius failure modes
- `✗` no material direct interaction found in the analyzed product surfaces
- `—` same subsystem

Subsystem key:
- `A` Agent Loop
- `B` Context Compaction
- `C` Prompt Cache (`cache boundary`, `break detection`)
- `D` Speculation
- `E` Tool Executor
- `F` Permission System (`permissions`, `classifier`)
- `G` Hook System
- `H` MCP Integration
- `I` Session Memory
- `J` Context Collapse (`marble_origami`)
- `K` Subagent / Coordinator
- `L` Fast Mode
- `M` Token Budget Management
- `N` Bridge / Remote Control
- `O` JSONL Persistence

Scope note:
- `J` is partly inferred from surrounding integration contracts because the dedicated context-collapse implementation is not fully present in the extracted code snapshot used for this analysis.
- path references later in this document are provenance only; the interaction descriptions are the transferable artifact

## Matrix

| Row\Col | A | B | C | D | E | F | G | H | I | J | K | L | M | N | O |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A | — | ⚠ | ⚠ | ✓ | ⚠ | ✓ | ✓ | ✓ | ✓ | ⚠ | ⚠ | ✓ | ⚠ | ✓ | ✓ |
| B | ⚠ | — | ✓ | ✗ | ✗ | ✗ | ✓ | ✗ | ⚠ | ⚠ | ✓ | ✗ | ✓ | ⚠ | ⚠ |
| C | ⚠ | ✓ | — | ⚠ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✗ | ✗ | ✗ |
| D | ✓ | ✗ | ⚠ | — | ✗ | ✓ | ✓ | ✗ | ✗ | ✗ | ✓ | ✗ | ✗ | ✗ | ✓ |
| E | ⚠ | ✗ | ✗ | ✗ | — | ⚠ | ⚠ | ⚠ | ✗ | ✗ | ✓ | ✗ | ✗ | ✓ | ✓ |
| F | ✓ | ✗ | ✗ | ✓ | ⚠ | — | ✓ | ⚠ | ✗ | ✗ | ⚠ | ✗ | ✗ | ⚠ | ✗ |
| G | ✓ | ✓ | ✗ | ✓ | ⚠ | ✓ | — | ⚠ | ✓ | ✗ | ✓ | ✗ | ✗ | ✗ | ✓ |
| H | ✓ | ✗ | ✗ | ✗ | ⚠ | ⚠ | ⚠ | — | ✗ | ✗ | ⚠ | ✗ | ✗ | ⚠ | ✓ |
| I | ✓ | ⚠ | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ | — | ✗ | ✓ | ✗ | ✗ | ✗ | ✓ |
| J | ⚠ | ⚠ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | — | ⚠ | ✗ | ✓ | ✗ | ⚠ |
| K | ⚠ | ✓ | ✓ | ✓ | ✓ | ⚠ | ✓ | ⚠ | ✓ | ⚠ | — | ✗ | ✓ | ✗ | ⚠ |
| L | ✓ | ✗ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | — | ✗ | ✗ | ✗ |
| M | ⚠ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ | ✗ | — | ✗ | ✗ |
| N | ✓ | ⚠ | ✗ | ✗ | ✓ | ⚠ | ✗ | ⚠ | ✗ | ✗ | ✗ | ✗ | ✗ | — | ⚠ |
| O | ✓ | ⚠ | ✗ | ✓ | ✓ | ✗ | ✓ | ✓ | ✓ | ⚠ | ⚠ | ✗ | ✗ | ⚠ | — |

## Detailed Notes

### A Pairs

- `A×B (⚠)`: `query.ts` drives snip, microcompact, autocompact, and reactive compaction, and then continues with the rewritten message list. Guards: autocompact skips `session_memory`, `compact`, and `marble_origami`, and suppresses itself while context collapse owns recovery; post-compact cleanup is main-thread aware. Failure: duplicated or cross-thread compaction can drop required `tool_use` or `tool_result` context, compact the wrong prefix, or corrupt shared state. Refs: `src/query.ts`, `src/services/compact/autoCompact.ts`, `src/services/compact/postCompactCleanup.ts`.
- `A×C (⚠)`: the loop relies on stable cache-safe parameters, cache-editing microcompact, and break detection around retries and compaction. Guards: prompt-cache tracking ignores short-lived fork sources, compaction resets the cache-read baseline, and cache-edit deletions are explicitly announced. Failure: false cache-break alarms or cache-key churn explode token writes and degrade follow-on latency. Refs: `src/query.ts`, `src/services/api/promptCacheBreakDetection.ts`, `src/services/compact/microCompact.ts`.
- `A×D (✓)`: the loop triggers prompt suggestion in stop hooks and later consumes accepted speculative state through the shared app/session flow. Guards: bare mode disables the background work, and suggestion generation runs in a fork with `skipTranscript` and `skipCacheWrite`. Failure: wasted background turns or stale suggestions competing with the active turn. Refs: `src/query/stopHooks.ts`, `src/services/PromptSuggestion/promptSuggestion.ts`, `src/services/PromptSuggestion/speculation.ts`.
- `A×E (⚠)`: the loop instantiates tool execution, streams results, and resumes the turn from executor output. Guards: executor ordering, sibling-cancel rules, and abort propagation are explicit. Failure: partial tool completion, wrong continuation state, or user interrupts that do not land cleanly. Refs: `src/query.ts`, `src/services/tools/StreamingToolExecutor.ts`.
- `A×F (✓)`: every tool-using turn passes through `canUseTool`, including classifier, coordinator, and interactive permission paths. Guards: explicit allow, deny, ask, abort, and coordinator/swarm forwarding branches. Failure: the loop either stalls on an unresolved permission or runs a tool the turn should not have been allowed to run. Refs: `src/query.ts`, `src/hooks/useCanUseTool.tsx`.
- `A×G (✓)`: the loop executes post-sampling hooks, stop hooks, and stop-hook retry handling. Guards: hooks are skipped for API-error paths that would otherwise loop forever, and hook output is schema-validated. Failure: infinite retry loops or hooks mutating turn state after a bad response. Refs: `src/query.ts`, `src/utils/hooks.ts`, `src/query/stopHooks.ts`.
- `A×H (✓)`: the loop threads MCP pending state into the model call and handles MCP session-expiry and elicitation-related errors as part of turn control. Guards: typed MCP errors and connection-state tracking. Failure: the loop keeps offering dead tools or mishandles auth/session expiration. Refs: `src/query.ts`, `src/services/mcp/client.ts`.
- `A×I (✓)`: stop hooks schedule session-memory extraction, and compaction may later consume that memory as a summary source. Guards: extraction is main-thread only and gated lazily. Failure: background extraction contends with the active loop or pollutes non-main-thread sessions. Refs: `src/query/stopHooks.ts`, `src/services/SessionMemory/sessionMemory.ts`.
- `A×J (⚠)`: the loop delegates withheld-overflow handling to context collapse before reactive compaction, and it applies collapses inline before the main model call. Guards: `isContextCollapseEnabled`, `isWithheldPromptTooLong`, and `recoverFromOverflow`; compaction also yields to collapse ownership. Failure: repeated overflow, wrong recovery ordering, or stale collapsed view state. Refs: `src/query.ts`, `src/services/compact/autoCompact.ts`.
- `A×K (⚠)`: subagents and coordinator workers are launched from the main turn and feed results, notifications, and sidechains back into it. Guards: recursion checks, agent deny rules, and team lifecycle constraints. Failure: agent storms, self-recursive forks, or hidden side effects from a worker mutating shared state. Refs: `src/query.ts`, `src/tools/AgentTool/AgentTool.tsx`, `src/tools/AgentTool/runAgent.ts`.
- `A×L (✓)`: the loop passes `fastMode` into the API call path. Guards: fast mode is availability-gated, model-gated, and cooldown-gated. Failure: requests bounce between fast and normal paths unexpectedly or keep requesting an unavailable speed tier. Refs: `src/query.ts`, `src/utils/fastMode.ts`, `src/services/api/claude.ts`.
- `A×M (⚠)`: token budget management can inject a continuation nudge and force another loop iteration after an otherwise complete answer. Guards: no budget for agents, stop at diminishing returns, and pre-compact context is deducted before the next iteration. Failure: runaway self-continuation or a budget that is double-counted after compaction. Refs: `src/query/tokenBudget.ts`, `src/query.ts`, `src/services/api/claude.ts`.
- `A×N (✓)`: remote sessions and bridge control messages feed user input, interrupts, and permission responses back into the active loop. Guards: typed control-request handling and explicit cancel/error responses. Failure: stuck remote permissions or dropped interrupts. Refs: `src/remote/RemoteSessionManager.ts`, `src/bridge/replBridge.ts`.
- `A×O (✓)`: the loop records content replacements, transcript messages, side effects, and queue state that power resume and analytics. Guards: dedup, parent-chain tracking, and sidechain/local-vs-remote handling. Failure: broken resume chains, duplicated messages, or lost transcript state after compaction or forking. Refs: `src/query.ts`, `src/utils/sessionStorage.ts`.

### B Pairs

- `B×C (✓)`: compaction and cached microcompact explicitly notify prompt-cache break detection so expected cache-read drops are not flagged as breaks. Guards: `notifyCompaction()` and `notifyCacheDeletion()`. Failure: every healthy compact looks like a cache regression and can trigger bad diagnostics or mitigation. Refs: `src/services/compact/compact.ts`, `src/services/compact/autoCompact.ts`, `src/services/api/promptCacheBreakDetection.ts`.
- `B×G (✓)`: compaction runs session-start hooks to rebuild context and post-compact hooks after summary creation. Guards: hook execution is separated by phase and validated. Failure: rebuilt context is missing or hook output mutates the post-compact prompt incorrectly. Refs: `src/services/compact/sessionMemoryCompact.ts`, `src/services/compact/compact.ts`, `src/utils/hooks.ts`.
- `B×I (⚠)`: session-memory compaction is the first compaction strategy tried in auto-compact and it depends on extracted session notes. Guards: extraction thresholds, main-thread-only extraction, and fallback when memory is missing or empty. Failure: compaction keeps stale summaries, compacts too aggressively, or falls back after already assuming memory coverage. Refs: `src/services/compact/autoCompact.ts`, `src/services/SessionMemory/sessionMemory.ts`, `src/services/compact/sessionMemoryCompact.ts`.
- `B×J (⚠)`: compaction and context collapse coordinate ownership of overflow recovery and both have reset logic in post-compact cleanup. Guards: autocompact suppression when context collapse is enabled and `marble_origami` query-source suppression. Failure: the two systems race, causing duplicate prefix rewriting or stale collapse-store state. Refs: `src/services/compact/autoCompact.ts`, `src/services/compact/postCompactCleanup.ts`.
- `B×K (✓)`: subagent compaction shares process-global state with the main thread, so cleanup has to be query-source aware. Guards: post-compact cleanup only resets main-thread-only caches for main-thread compacts. Failure: a worker compact corrupts the main thread’s context-collapse or memory-file caches. Refs: `src/services/compact/postCompactCleanup.ts`.
- `B×M (✓)`: compaction affects token-budget accounting because pre-compact context is charged before the smaller post-compact window replaces it. Guards: explicit `taskBudgetRemaining` deduction before message replacement. Failure: the model gets extra hidden budget or is prematurely exhausted. Refs: `src/query.ts`.
- `B×N (⚠)`: remote/bridge sessions surface compaction as status and `compact_boundary` messages, and remote WebSocket handling has compaction-specific close/retry behavior. Guards: typed compact-boundary adaptation and retry handling. Failure: remote viewers diverge from local state and can resume from the wrong visible chain. Refs: `src/remote/sdkMessageAdapter.ts`, `src/remote/SessionsWebSocket.ts`.
- `B×O (⚠)`: compaction writes `compact_boundary` and summary state into JSONL and the loader prunes stale collapse entries across that boundary. Guards: compact-boundary-aware restore and dedup. Failure: `/resume` reconstructs a chain that still points into pre-compact history or double-prunes the live chain. Refs: `src/utils/sessionStorage.ts`.

### C Pairs

- `C×D (⚠)`: prompt suggestion deliberately piggybacks on the parent prompt cache and warns not to override cache-key parameters; it also suppresses itself on a cold parent cache. Guards: `cacheSafeParams`, `skipCacheWrite`, and parent-cache suppress reasons. Failure: a suggestion fork busts the main cache and multiplies cache writes. Refs: `src/services/PromptSuggestion/promptSuggestion.ts`.
- `C×K (✓)`: prompt-cache break detection isolates tracked subagents by `agentId` and cleans up their tracking state on teardown. Guards: agent-specific tracking keys and `cleanupAgentTracking()`. Failure: one agent pollutes another agent’s cache-break history or leaves stale tracking behind. Refs: `src/services/api/promptCacheBreakDetection.ts`, `src/tools/AgentTool/runAgent.ts`.
- `C×L (✓)`: fast mode uses a sticky beta header specifically so mid-session toggles do not change the server-side prompt-cache key. Guards: session-latched fast-mode header while `speed='fast'` stays dynamic. Failure: every fast-mode toggle turns into a cold-cache rewrite. Refs: `src/services/api/claude.ts`, `src/utils/fastMode.ts`.

### D Pairs

- `D×F (✓)`: speculation and prompt suggestion are permission-aware and tool-constrained; suggestions deny all tools, and speculative execution enforces read-only constraints. Guards: `canUseTool` deny callback, permission suppression reasons, and read-only command checks. Failure: speculative work escapes into mutating tools or runs while a real permission dialog is pending. Refs: `src/services/PromptSuggestion/promptSuggestion.ts`, `src/services/PromptSuggestion/speculation.ts`.
- `D×G (✓)`: prompt suggestion is started from stop hooks rather than the hot path. Guards: disabled in bare mode and behind feature/env checks. Failure: hook-triggered speculation consumes resources after turns where the process is supposed to stay quiet. Refs: `src/query/stopHooks.ts`, `src/services/PromptSuggestion/promptSuggestion.ts`.
- `D×K (✓)`: both suggestion generation and speculation use forked-agent infrastructure and isolated overlays rather than mutating the live turn in place. Guards: `runForkedAgent`, overlay copy-back only on accept, and stripped interrupt or partial-tool state before injection. Failure: speculative state leaks into the main workspace before acceptance. Refs: `src/services/PromptSuggestion/promptSuggestion.ts`, `src/services/PromptSuggestion/speculation.ts`.
- `D×O (✓)`: accepted speculation writes a `speculation-accept` entry to the transcript. Guards: best-effort append and typed entry schema. Failure: lost acceptance telemetry or resume/debug traces that under-report speculative behavior. Refs: `src/services/PromptSuggestion/speculation.ts`, `src/types/logs.ts`.

### E Pairs

- `E×F (⚠)`: the executor is where permission decisions turn into actual tool execution or denial. Guards: `runToolUse()` routes through permission logic and abort handling, and only some tool failures cancel siblings. Failure: a tool executes without approval or approved sibling work is cancelled incorrectly. Refs: `src/services/tools/StreamingToolExecutor.ts`, `src/services/tools/toolExecution.ts`, `src/hooks/useCanUseTool.tsx`.
- `E×G (⚠)`: post-tool and failure hooks can block continuation, add context, or rewrite MCP output. Guards: hook results are typed and duplicate blocking errors are suppressed. Failure: a hook mutates a tool result twice or prevents continuation after the executor has already advanced state. Refs: `src/services/tools/toolHooks.ts`, `src/utils/hooks.ts`.
- `E×H (⚠)`: MCP tools execute through the same executor, and MCP-specific output can be rewritten by hooks. Guards: MCP tool metadata utilities, typed MCP errors, and `updatedMCPToolOutput` only for MCP tools. Failure: non-MCP tools accidentally receive MCP-shaped rewrites or MCP failures propagate as generic tool noise. Refs: `src/services/tools/toolExecution.ts`, `src/services/tools/toolHooks.ts`, `src/services/mcp/client.ts`.
- `E×K (✓)`: `AgentTool` is just another executable tool from the executor’s point of view. Guards: executor concurrency and agent-tool-specific restrictions. Failure: agent launch is ordered or cancelled incorrectly relative to sibling tools. Refs: `src/services/tools/StreamingToolExecutor.ts`, `src/tools/AgentTool/AgentTool.tsx`.
- `E×N (✓)`: remote sessions render executor progress through SDK tool-progress messages. Guards: typed SDK message adaptation. Failure: remote users see hanging tools with no progress or duplicate tool activity. Refs: `src/remote/sdkMessageAdapter.ts`, `src/remote/RemoteSessionManager.ts`.
- `E×O (✓)`: tool-generated messages are part of what gets recorded and resumed. Guards: only recordable message types are written into sidechains, and transcript dedup preserves chain integrity. Failure: `/resume` loses tool context or replays it twice. Refs: `src/tools/AgentTool/runAgent.ts`, `src/utils/sessionStorage.ts`.

### F Pairs

- `F×G (✓)`: permissions and hooks share a boundary because hook output can include permission changes, additional context, or tool-input rewrites consumed by the permission path. Guards: strict hook-response parsing and event-name validation. Failure: malformed hook output alters approval behavior or produces a dialog for the wrong tool payload. Refs: `src/utils/hooks.ts`, `src/services/tools/toolExecution.ts`.
- `F×H (⚠)`: permissions are MCP-aware, including server-wide wildcard rules, and MCP channels can relay permission responses. Guards: fully qualified MCP tool matching plus allowlisted, capability-gated channel relays. Failure: an MCP tool slips past the wrong rule or a compromised relay server fabricates approvals faster than the terminal path would. Refs: `src/utils/permissions/permissions.ts`, `src/services/mcp/channelPermissions.ts`.
- `F×K (⚠)`: subagents are filtered by permission rules, and coordinator/swarm workers can resolve permissions before the user dialog path. Guards: `Agent(agentType)` deny rules, teammate restrictions, and coordinator/swarm-specific handlers. Failure: a denied worker type still launches or a worker gets stuck waiting on a permission path that was supposed to be delegated. Refs: `src/tools/AgentTool/AgentTool.tsx`, `src/hooks/useCanUseTool.tsx`, `src/utils/permissions/permissions.ts`.
- `F×N (⚠)`: the permission dialog can race local UI, bridge callbacks, and channel callbacks. Guards: first-resolver-wins semantics, structured control messages, and cancellation handling. Failure: split-brain permission resolution where remote and local surfaces disagree on whether a tool was allowed. Refs: `src/hooks/useCanUseTool.tsx`, `src/remote/RemoteSessionManager.ts`, `src/services/mcp/channelPermissions.ts`.

### G Pairs

- `G×H (⚠)`: hooks and MCP cross at connection-start, elicitation, and post-tool mutation points. Guards: only MCP tools may receive `updatedMCPToolOutput`, and connection handlers are registered centrally. Failure: hooks rewrite the wrong tool payload or miss a live MCP elicitation path entirely. Refs: `src/services/tools/toolHooks.ts`, `src/services/mcp/useManageMCPConnections.ts`, `src/services/mcp/client.ts`.
- `G×I (✓)`: session memory is implemented as a post-sampling hook and session-memory compaction runs session-start hooks when rebuilding prompt context. Guards: registration only when auto-compact is enabled and not in remote mode. Failure: memory extraction runs in contexts where the rest of the hook machinery is not prepared for it. Refs: `src/services/SessionMemory/sessionMemory.ts`, `src/services/compact/sessionMemoryCompact.ts`.
- `G×K (✓)`: subagents run start hooks and clear per-agent hooks on teardown. Guards: `clearSessionHooks()` is keyed to the worker’s `agentId`. Failure: one agent inherits another agent’s hook state. Refs: `src/tools/AgentTool/runAgent.ts`, `src/utils/hooks.ts`.
- `G×O (✓)`: hooks consume transcript/session metadata and post-compact hooks run after transcript-affecting summary creation. Guards: base hook input includes transcript/session paths and output schema validation is strict. Failure: hooks act on stale persisted state or write side effects keyed to the wrong transcript. Refs: `src/utils/hooks.ts`, `src/services/compact/compact.ts`.

### H Pairs

- `H×K (⚠)`: agents can add MCP servers to the inherited MCP client set, but plugin-only policy can suppress user-controlled MCP definitions. Guards: admin-trusted-source check, additive merge, and cleanup only for inline-created clients. Failure: worker-specific MCP servers leak into other workers or are trusted when policy says they should not be. Refs: `src/tools/AgentTool/runAgent.ts`, `src/tools/AgentTool/AgentTool.tsx`.
- `H×N (⚠)`: bridge/remote flows surface MCP-backed channel permissions and remote session control over the same live tool set. Guards: allowlisted relay clients, explicit capabilities, and typed control requests. Failure: a remote or channel surface becomes an untrusted permission side door. Refs: `src/services/mcp/channelPermissions.ts`, `src/services/mcp/useManageMCPConnections.ts`, `src/remote/RemoteSessionManager.ts`.
- `H×O (✓)`: MCP-related tool activity and remote-ingress state ultimately land in the same transcript/log pipeline used for restore. Guards: ordered write queue and remote hydration before enabling persistence. Failure: restored sessions know about the wrong MCP-visible state or miss prior MCP side effects. Refs: `src/utils/sessionStorage.ts`, `src/utils/sessionRestore.ts`.

### I Pairs

- `I×K (✓)`: session memory uses `createSubagentContext()` and `runForkedAgent()` to isolate its extraction pass from the parent turn. Guards: only the exact memory file is editable in the extraction fork. Failure: the extraction worker mutates normal workspace files or pollutes the parent read cache. Refs: `src/services/SessionMemory/sessionMemory.ts`.
- `I×O (✓)`: session memory itself lives outside the transcript, but its summaries and compact boundaries become transcript state and influence restore. Guards: safe update of `lastSummarizedMessageId` and compact-boundary-aware message retention. Failure: restored sessions think memory covered messages that were never safely summarized. Refs: `src/services/SessionMemory/sessionMemory.ts`, `src/services/compact/sessionMemoryCompact.ts`, `src/utils/sessionStorage.ts`.

### J Pairs

- `J×K (⚠)`: transcript types describe context-collapse snapshot state as “staged queue + spawn trigger state” written after each `ctx-agent` spawn resolves, so collapse logic is coupled to agent spawning. Guards: snapshot restore is last-wins and compaction clears stale collapse state at compact boundaries. Failure: resumed collapse state points at the wrong staged spans or re-arms worker spawning incorrectly. Refs: `src/types/logs.ts`, `src/utils/sessionStorage.ts`. This pair is inferred from persistence contracts because the implementation files are absent.
- `J×M (✓)`: context collapse is the first overflow-recovery path before reactive compact, which directly changes how much of the budgeted turn can continue. Guards: overflow is checked with explicit collapse helpers before falling back. Failure: budgeted turns burn tokens retrying oversized prompts instead of collapsing first. Refs: `src/query.ts`.
- `J×O (⚠)`: context collapse persists ordered commit entries plus a last-wins snapshot and restores them before the first resumed query. Guards: restore resets state first, and compact boundaries clear stale collapse entries on load. Failure: `/resume` reconstructs a collapse store that refers to messages no longer reachable in the live chain. Refs: `src/utils/sessionStorage.ts`, `src/utils/sessionRestore.ts`, `src/types/logs.ts`.

### K Pairs

- `K×M (✓)`: token-budget continuation logic explicitly stops applying inside agents. Guard: `checkTokenBudget()` exits early when `agentId` is set. Failure: workers self-nudge forever or stop prematurely as though they were the top-level task. Refs: `src/query/tokenBudget.ts`.
- `K×O (⚠)`: subagents persist sidechain transcripts, metadata, and content-replacement state separately from the main session and must not poison the main UUID chain. Guards: sidechain dedup bypass is local-only, remote persistence keeps a single main-session UUID chain, and sidechain transcript paths are agent-specific. Failure: dangling parent UUIDs, 409 conflicts in remote persistence, or incomplete worker resumes. Refs: `src/utils/sessionStorage.ts`, `src/tools/AgentTool/runAgent.ts`.

### N Pairs

- `N×O (⚠)`: bridge and remote mode reuse the same JSONL model for local persistence, remote ingress writes, and remote hydration back to disk. Guards: ordered write queues, remote hydration before enabling persistence, and UUID-chain dedup rules. Failure: durable cross-device session corruption where local and remote transcripts disagree on parentage or replay order. Refs: `src/utils/sessionStorage.ts`, `src/remote/RemoteSessionManager.ts`, `src/bridge/replBridge.ts`.

## Dangerous Pairs

- `A×B`: the main loop hands compaction ownership around several times in one turn. A guard miss here rewrites the active prompt, not just telemetry.
- `A×J`: overflow recovery has two owners, context collapse and reactive compact. The wrong order leaves the turn repeatedly too large.
- `C×D`: speculation and prompt suggestion deliberately share the parent prompt cache. This is the easiest place to create a silent cache-cost regression.
- `E×F`: this is the last boundary before a tool actually runs. A bad interaction becomes a real side effect, not just a bad summary.
- `E×G`: tool hooks can block continuation or rewrite outputs after execution has already started. This is powerful and easy to get subtly wrong.
- `F×H`: MCP permissions combine wildcard matching with channel-based approval relay. The trust boundary is wider than the terminal.
- `H×K`: worker-specific MCP servers expand capability surface area dynamically. Policy mistakes here are hard to notice from the parent turn alone.
- `J×O`: context collapse is resume-sensitive and persists custom commit/snapshot state. If the log is inconsistent, resumed context can be structurally wrong.
- `K×O`: sidechain persistence is one of the highest-risk resume paths because local and remote dedup rules differ.
- `N×O`: remote hydration and remote persistence share the same transcript model. Ordering or dedup bugs here create durable cross-device session corruption.
