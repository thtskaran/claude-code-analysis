# Claude Code: Complete Engineering Knowledge Base

> **Generated**: 2026-04-01
> **Scope**: Full-system reverse engineering from ~1,332 TypeScript source files + prior analysis passes
> **Runtime**: Bun (TypeScript, React/Ink terminal UI)
> **Self-contained**: This document is designed to be readable without access to the source tree

---

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. Architecture Overview](#2-architecture-overview)
- [3. The Agent Loop — Complete Specification](#3-the-agent-loop--complete-specification)
- [4. Context Window Management — Complete Specification](#4-context-window-management--complete-specification)
- [5. Tool System — Complete Specification](#5-tool-system--complete-specification)
- [6. Individual Tool Deep-Dives](#6-individual-tool-deep-dives)
- [7. Permission & Safety Model — Complete Specification](#7-permission--safety-model--complete-specification)
- [8. MCP Integration — Complete Specification](#8-mcp-integration--complete-specification)
- [9. Prompt Engineering Library](#9-prompt-engineering-library)
- [10. Configuration & Extensibility](#10-configuration--extensibility)
- [11. The Speculation System](#11-the-speculation-system)
- [12. Multi-Agent Orchestration](#12-multi-agent-orchestration)
- [13. Session Persistence & Recovery](#13-session-persistence--recovery)
- [14. Performance Architecture](#14-performance-architecture)
- [15. Error Handling & Resilience](#15-error-handling--resilience)
- [16. Analytics, Telemetry & Feature Flags](#16-analytics-telemetry--feature-flags)
- [17. The Bridge & Remote Control](#17-the-bridge--remote-control)
- [18. Voice, Vim, Buddy & UI Systems](#18-voice-vim-buddy--ui-systems)
- [19. Daemon & Background Sessions](#19-daemon--background-sessions)
- [20. Cross-System Interaction Matrix](#20-cross-system-interaction-matrix)
- [21. The Constants Bible](#21-the-constants-bible)
- [22. The Model Behavioral Contract](#22-the-model-behavioral-contract)
- [23. Design Patterns Catalog](#23-design-patterns-catalog)
- [24. Architecture Decision Records](#24-architecture-decision-records)
- [Appendices](#appendices)

---

## 1. Executive Summary

### What Claude Code Is

Claude Code is Anthropic's official CLI-based agentic coding harness — an interactive terminal application that connects a human developer to Claude models through a tool-use loop. The user types natural language; Claude responds with text and tool calls (file reads, edits, shell commands, web fetches, etc.); the harness executes those tools, feeds results back, and loops until the task is done or the user intervenes.

It ships as a standalone binary (via npm or native installer) and runs on macOS, Linux, and Windows. The codebase is ~600K lines of TypeScript running on the Bun runtime, with a React/Ink terminal UI for rich interactive rendering. It supports multiple deployment modes: interactive CLI, headless (`-p/--print`), SDK embedding, VS Code/JetBrains IDE extensions, desktop app, remote/bridge sessions via claude.ai, and daemon/assistant mode.

The system is designed around five core architectural ideas:

1. **Async-generator agent loop**: The conversation is an infinite loop that streams API responses, detects tool calls in real-time, executes them concurrently, and feeds results back — all through a single async generator.
2. **Three-tier context window management**: Microcompact (API-side cache_edits clearing) → full compaction (LLM-generated summary replacing history) → session memory (pre-built summary persisted to disk) — with automatic triggering based on token thresholds.
3. **Permission-gated tool execution**: Every tool call passes through a multi-layer permission system (mode → rules → classifier → user prompt) before execution, with a two-stage auto-mode classifier for autonomous operation.
4. **Prompt cache optimization**: The system prompt and tool schemas are carefully ordered and memoized so the API's prompt caching can reuse prefixes across turns, with explicit cache break detection.
5. **Speculation/suggestion system**: Between user turns, the system predicts likely next actions and pre-computes results on a copy-on-write overlay, presenting them as instant suggestions.

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Bun (TypeScript) |
| UI Framework | React + Ink (custom fork) with Yoga layout |
| API Client | Raw `fetch()` with custom streaming (bypasses Anthropic SDK) |
| Shell | `Bun.spawn()` with persistent shell process per session |
| MCP | `@anthropic-ai/sdk` MCP client library |
| Package format | npm (`@anthropic-ai/claude-code`) + native installers |
| Config | JSON files at `~/.claude/`, project `.claude/`, and managed policy |
| Persistence | JSONL session transcripts + JSON config files |
| Feature flags | GrowthBook (runtime) + build-time DCE via `USER_TYPE` |
| Analytics | Datadog (1P privileged) + first-party OTLP pipeline |

---

## 2. Architecture Overview

### System Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ENTRY POINTS                                  │
│  CLI (cli.tsx) │ SDK (sdk/) │ Desktop │ IDE Extension │ MCP Server   │
└───────┬─────────────┬──────────┬───────────┬──────────┬─────────────┘
        │             │          │           │          │
        ▼             ▼          ▼           ▼          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     BOOTSTRAP (bootstrap/)                           │
│  Config loading → Auth → Feature flags → MCP connections → State     │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     MAIN LOOP (main.tsx / REPL.tsx)                   │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────┐           │
│  │ User     │───▶│ System Prompt│───▶│ API Call          │           │
│  │ Input    │    │ Assembly     │    │ (services/api/)   │           │
│  └──────────┘    └──────────────┘    └────────┬─────────┘           │
│                                               │                      │
│                                               ▼                      │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │              AGENT LOOP (assistant/)                       │       │
│  │  Stream response → Detect tool calls → Execute tools      │       │
│  │       ▲                                    │               │       │
│  │       │              ┌─────────────────────┘               │       │
│  │       │              ▼                                     │       │
│  │  ┌────────┐   ┌──────────────┐   ┌───────────────┐       │       │
│  │  │ Result │◀──│ Permission   │◀──│ Tool Dispatch  │       │       │
│  │  │ Inject │   │ Check        │   │ (tools/)       │       │       │
│  │  └────────┘   └──────────────┘   └───────────────┘       │       │
│  └──────────────────────────────────────────────────────────┘       │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐        │
│  │ Compaction   │  │ Session      │  │ Speculation /      │        │
│  │ Engine       │  │ Memory       │  │ Suggestions        │        │
│  │ (compact/)   │  │ (Session     │  │ (PromptSuggestion/)│        │
│  │              │  │  Memory/)    │  │                    │        │
│  └──────────────┘  └──────────────┘  └────────────────────┘        │
└─────────────────────────────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌──────────────┐  ┌──────────────────┐  ┌────────────────────────┐
│ Terminal UI  │  │ Bridge / Remote  │  │ Persistence            │
│ (Ink/React)  │  │ (bridge/)        │  │ (JSONL sessions,       │
│ Components,  │  │ WebSocket,       │  │  config files,         │
│ Yoga layout  │  │ HTTP POST,       │  │  file history)         │
└──────────────┘  │ permission relay │  └────────────────────────┘
                  └──────────────────┘
```

### Entry Point Trace: What Happens When You Type `claude`

1. **CLI entrypoint** (`src/entrypoints/cli.tsx`): Parses argv. Fast paths for `claude ps|logs|attach|kill|daemon` exit before loading the full app. Otherwise proceeds to `main()`.

2. **Bootstrap** (`src/bootstrap/`): Loads config hierarchy (managed → global → project → local → env). Initializes auth (API key, OAuth, Bedrock/Vertex). Sets up feature flags via GrowthBook. Connects MCP servers in parallel.

3. **Mode selection** (`src/main.tsx`):
   - `--print` / `-p` → headless mode (`runHeadless()`) — no UI, pure stdio
   - `--resume` → session recovery from JSONL transcript
   - `--bridge` → bridge supervisor mode
   - `--daemon` → daemon supervisor mode
   - Default → interactive REPL with React/Ink terminal UI

4. **REPL setup** (`src/screens/REPL.tsx`): Mounts the Ink component tree. Renders the prompt input, message history, tool execution status, and permission dialogs. Hooks up keyboard handlers, voice input, vim mode.

5. **User submits prompt**: Text goes through `processUserInput()` which detects slash commands (`/compact`, `/model`, etc.) or passes through to the agent loop.

6. **Agent loop** (`src/assistant/`): Assembles the system prompt + conversation history + tool schemas. Calls the Claude API. Streams the response. Detects tool calls. Executes them. Feeds results back. Repeats until `end_turn` stop reason.

7. **Response rendering**: Streamed text and tool results are rendered in real-time through React/Ink components. The terminal UI uses a custom Yoga layout engine for positioning.

### Module Responsibility Map

| Directory | Responsibility |
|-----------|---------------|
| `src/entrypoints/` | CLI entry, SDK entry, MCP server entry |
| `src/bootstrap/` | State initialization, config loading, app state container |
| `src/assistant/` | Agent loop, session history, conversation management |
| `src/services/api/` | Claude API client, retry logic, streaming, prompt cache |
| `src/services/compact/` | Autocompact, microcompact, full compaction engine |
| `src/services/SessionMemory/` | Session memory extraction and persistence |
| `src/services/mcp/` | MCP client connections, auth, tool registration |
| `src/services/analytics/` | Datadog, first-party OTLP, GrowthBook |
| `src/services/tools/` | StreamingToolExecutor, tool dispatch |
| `src/tools/` | Individual tool implementations (35+ tools) |
| `src/commands/` | Slash command implementations (100+ commands) |
| `src/components/` | React/Ink UI components |
| `src/hooks/` | React hooks for UI state, permissions, notifications |
| `src/utils/` | Shared utilities (permissions, bash parsing, config, git, etc.) |
| `src/utils/permissions/` | Permission checking, auto-mode classifier, rule matching |
| `src/utils/bash/` | Bash command parsing, safety classification |
| `src/state/` | Application state store |
| `src/context/` | React context providers |
| `src/bridge/` | Bridge supervisor, remote session management |
| `src/remote/` | Remote session WebSocket client |
| `src/coordinator/` | Multi-agent coordinator mode |
| `src/tasks/` | Background task types (agent, shell, dream, teammate) |
| `src/skills/` | Bundled skill definitions |
| `src/plugins/` | Plugin system, marketplace, built-in plugins |
| `src/schemas/` | Zod validation schemas |
| `src/types/` | TypeScript type definitions |
| `src/ink/` | Custom Ink fork (terminal rendering engine) |
| `src/native-ts/` | Native TypeScript implementations (yoga-layout, color-diff, file-index) |
| `src/vim/` | Vim mode emulation |
| `src/voice/` | Voice input pipeline |
| `src/buddy/` | Companion sprite / buddy system |
| `src/keybindings/` | Keyboard shortcut system |
| `src/memdir/` | Memory directory (auto-memory file system) |
| `src/migrations/` | Config migration logic |
| `src/outputStyles/` | Output formatting styles |
| `src/query/` | Query orchestration, token budget, stop hooks |
| `src/screens/` | Top-level screen components (REPL, ResumeConversation) |
| `src/server/` | Local HTTP/SSE server for IDE integration |
| `src/upstreamproxy/` | HTTP proxy relay for remote sessions |

---

## 3. The Agent Loop — Complete Specification

### Core Loop Pseudocode

The agent loop is an async generator in `src/assistant/` that orchestrates the conversation. Here is the complete logic flow:

```
function* agentLoop(initialMessages, tools, systemPrompt):
  messages = initialMessages
  turnCount = 0

  while true:
    turnCount++

    // 1. CHECK CONTEXT WINDOW
    inputTokens = countTokens(systemPrompt + messages + toolSchemas)
    if inputTokens > autocompactThreshold:
      if consecutiveCompactFailures < MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES (3):
        messages = await compact(messages)  // Full compaction via LLM
      else:
        // Circuit breaker: stop trying after 3 consecutive failures
        log("tengu_compact_failed")

    // 2. ASSEMBLE API REQUEST
    request = {
      model: selectedModel,
      system: systemPrompt + attachments + systemReminders,
      messages: messages,
      tools: toolSchemas,
      max_tokens: getMaxOutputTokens(),  // starts at 8K, escalates to 64K
      stream: true,
      metadata: { user_id: hashedUserId },
      betas: [enabledBetaHeaders],
    }

    // 3. CALL API WITH STREAMING
    stream = await claudeAPI.createMessageStream(request)

    // 4. PROCESS STREAM EVENTS
    assistantMessage = { role: "assistant", content: [] }
    toolExecutor = new StreamingToolExecutor(tools)

    for await (event of stream):
      switch event.type:
        case "content_block_start":
          if event.content_block.type == "tool_use":
            toolExecutor.startToolUse(event.content_block)
          elif event.content_block.type == "text":
            assistantMessage.content.push({type: "text", text: ""})
          elif event.content_block.type == "thinking":
            // Extended thinking block
            assistantMessage.content.push({type: "thinking", thinking: ""})

        case "content_block_delta":
          if event.delta.type == "text_delta":
            appendText(event.delta.text)
            yield {type: "text", text: event.delta.text}  // Stream to UI
          elif event.delta.type == "input_json_delta":
            toolExecutor.appendInput(event.index, event.delta.partial_json)
          elif event.delta.type == "thinking_delta":
            appendThinking(event.delta.thinking)

        case "content_block_stop":
          if isToolUse(event.index):
            // Tool input is complete — EXECUTE IMMEDIATELY
            // Don't wait for message_stop; start execution during streaming
            toolExecutor.executeWhenReady(event.index)

        case "message_delta":
          stopReason = event.delta.stop_reason
          // "end_turn" | "tool_use" | "max_tokens" | "stop_sequence"

        case "message_stop":
          break  // Stream complete

    // 5. APPEND ASSISTANT MESSAGE
    messages.push(assistantMessage)

    // 6. HANDLE STOP REASON
    switch stopReason:
      case "end_turn":
        // Model is done — but check for pending tool executions
        if toolExecutor.hasPendingTools():
          // Wait for all tools to finish
          toolResults = await toolExecutor.waitForAll()
          messages.push({role: "user", content: toolResults})
          continue  // Loop back for another turn
        else:
          // Run stop hooks
          stopHookResult = await runStopHooks(messages)
          if stopHookResult.shouldContinue:
            messages.push(stopHookResult.continueMessage)
            continue
          return messages  // DONE

      case "tool_use":
        // Model explicitly requested tool execution
        toolResults = await toolExecutor.waitForAll()
        messages.push({role: "user", content: toolResults})
        continue  // Loop back

      case "max_tokens":
        // Output was truncated
        recoveryCount++
        if recoveryCount <= MAX_OUTPUT_TOKENS_RECOVERY_LIMIT (3):
          messages.push({role: "user", content: [
            {type: "text", text: "Your response was truncated. Continue exactly where you left off."}
          ]})
          // Escalate max_tokens: 8K → 16K → 32K → 64K
          escalateMaxTokens()
          continue
        else:
          return messages  // Give up after 3 recovery attempts

      case "stop_sequence":
        return messages
```

### Key Constants and Thresholds

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT` | 3 | Max continuation attempts after `max_tokens` truncation |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | Circuit breaker for compaction failures |
| `CAPPED_DEFAULT_MAX_TOKENS` | 8,000 | Initial max output tokens |
| `ESCALATED_MAX_TOKENS` | 64,000 | Max output tokens after escalation |
| `MAX_OUTPUT_TOKENS_DEFAULT` | 32,000 | Default max output tokens |
| `MAX_OUTPUT_TOKENS_UPPER_LIMIT` | 64,000 | Absolute ceiling for output tokens |
| `MODEL_CONTEXT_WINDOW_DEFAULT` | 200,000 | Default context window size |

### Stop Conditions

| Condition | Trigger | Action |
|-----------|---------|--------|
| `end_turn` with no pending tools | Model says it's done | Run stop hooks → return |
| `end_turn` with pending tools | Model finished text but tools still running | Wait for tools → continue loop |
| `tool_use` | Model wants tool results | Execute tools → inject results → continue |
| `max_tokens` (≤3 times) | Output truncated | Inject recovery message → escalate tokens → continue |
| `max_tokens` (>3 times) | Repeated truncation | Return (give up) |
| User interrupt (Ctrl+C) | User presses Escape/Ctrl+C | Abort current stream → return partial |
| Token budget exhausted | `COMPLETION_THRESHOLD` (0.9) reached | Stop loop |
| Stop hook says continue | Hook injects follow-up message | Continue loop |

### The Streaming Pipeline

The raw API stream is processed through these stages:

1. **Raw SSE events** → Parsed by custom streaming code (bypasses Anthropic SDK for performance)
2. **Content block detection** → Text blocks rendered immediately, tool_use blocks accumulated
3. **Tool input accumulation** → JSON input streamed incrementally, parsed when block stops
4. **Concurrent tool execution** → Tools execute as soon as their input is complete, not waiting for the full message
5. **Result serialization** → Tool results formatted as `tool_result` content blocks
6. **Context injection** → Results appended to messages array for next API call

### Max Output Token Escalation

The system starts with a conservative 8K max_tokens to save costs and escalates on demand:

```
Turn 1: max_tokens = 8,000 (CAPPED_DEFAULT_MAX_TOKENS)
If max_tokens hit: escalate to 16,000
If max_tokens hit again: escalate to 32,000
If max_tokens hit again: escalate to 64,000 (ESCALATED_MAX_TOKENS, ceiling)
```

This escalation is tracked per-conversation and resets on user input.

---

## 4. Context Window Management — Complete Specification

Context window management is the most architecturally complex subsystem. It operates through three tiers that interact with explicit precedence rules.

### Tier 1: Microcompact (API-Side Cache Clearing)

Microcompact uses the Anthropic API's `cache_edits` feature to clear old tool results without regenerating the conversation summary. It's the cheapest form of context recovery.

**Mechanism**: Sends `cache_edits` operations that replace large tool result content blocks with a brief placeholder, preserving the conversation structure but reducing token count.

**Trigger conditions**:
- Input tokens exceed `DEFAULT_MAX_INPUT_TOKENS` (180,000)
- There are clearable tool results (large content blocks that can be replaced with references)

**Constants**:
- `IMAGE_MAX_TOKEN_SIZE`: 2,000 tokens (images below this aren't cleared)

**What gets cleared**: Tool result content blocks above a size threshold, replaced with a message like `[Content cleared to save context. File can be re-read if needed.]`

### Tier 2: Full Compaction (LLM-Generated Summary)

When microcompact isn't sufficient, the system performs full compaction: sending the conversation to Claude with a special prompt asking for a comprehensive summary.

**The Compaction Prompt** (verbatim critical sections):

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

The summary must contain 9 sections:
1. **Primary Request and Intent** — all user's explicit requests
2. **Key Technical Concepts** — frameworks, technologies discussed
3. **Files and Code Sections** — files examined/modified with full snippets
4. **Errors and fixes** — all errors, how fixed, user feedback
5. **Problem Solving** — problems solved, ongoing troubleshooting
6. **All user messages** — ALL non-tool-result user messages (CRITICAL)
7. **Pending Tasks** — explicitly asked tasks not yet completed
8. **Current Work** — precise detail of work immediately before summary
9. **Optional Next Step** — next step directly in line with most recent requests

**Trigger**: `inputTokens > contextWindow - AUTOCOMPACT_BUFFER_TOKENS (13,000)`

**Key constants**:
| Constant | Value | Purpose |
|----------|-------|---------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | Headroom before autocompact triggers |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | Warning threshold |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | 20,000 | Error threshold |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | Buffer for manual `/compact` |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | Max tokens for compaction output (p99.99 = 17,387) |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | Circuit breaker (saves ~250K API calls/day) |
| `MAX_COMPACT_STREAMING_RETRIES` | 2 | Retries for streaming errors during compaction |
| `MAX_PTL_RETRIES` | 3 | Retries with progressively truncated messages |
| `PTL_RETRY_MARKER` | `'[earlier conversation truncated for compaction retry]'` | Marker for truncated retries |

**Post-compaction file restoration**: After compacting, the system selectively re-reads up to 5 recently active files to restore working context:
- `POST_COMPACT_MAX_FILES_TO_RESTORE`: 5
- `POST_COMPACT_TOKEN_BUDGET`: 50,000 tokens
- `POST_COMPACT_MAX_TOKENS_PER_FILE`: 5,000 tokens per file
- `POST_COMPACT_MAX_TOKENS_PER_SKILL`: 5,000 tokens per skill
- `POST_COMPACT_SKILLS_TOKEN_BUDGET`: 25,000 tokens for all skills

### Tier 3: Session Memory Compaction

Session memory is a structured summary persisted to disk that can replace full compaction on resume.

**The template** (verbatim):

```markdown
# Session Title
_A short and distinctive 5-10 word descriptive title for the session_

# Current State
_What is actively being worked on right now?_

# Task specification
_What did the user ask to build? Design decisions or explanatory context_

# Files and Functions
_Important files, what they contain and why they're relevant_

# Workflow
_Bash commands usually run and in what order_

# Errors & Corrections
_Errors encountered and how they were fixed_

# Codebase and System Documentation
_Important system components and how they work together_

# Learnings
_What has worked well? What has not? What to avoid?_

# Key results
_If the user asked for a specific output, repeat the exact result here_

# Worklog
_Step by step, what was attempted and done_
```

**Constraints**:
- `MAX_SECTION_LENGTH`: 2,000 tokens per section
- `MAX_TOTAL_SESSION_MEMORY_TOKENS`: 12,000 tokens total
- `EXTRACTION_WAIT_TIMEOUT_MS`: 15,000ms
- `EXTRACTION_STALE_THRESHOLD_MS`: 60,000ms

**Extraction trigger**: Session memory is extracted at natural breakpoints (after compaction, on session end, periodically during long sessions).

### Interaction Rules Between Tiers

1. **Microcompact runs first** when available (cheapest, API-side)
2. **Full compaction runs if** microcompact insufficient OR feature flag disables microcompact
3. **Session memory replaces compaction** when resuming a session (pre-built summary avoids re-summarizing)
4. **Circuit breaker**: After 3 consecutive compaction failures, the system stops trying (discovered: 1,279 sessions had 50+ consecutive failures, wasting ~250K API calls/day)
5. **Mutual exclusion**: Only one compaction type runs at a time; they don't stack

---

## 5. Tool System — Complete Specification

### Tool Type Definition

Every tool implements this interface:

```typescript
interface Tool {
  name: string                              // Unique identifier
  description: string                       // Human-readable description
  async prompt(params): string              // Dynamic prompt (can vary by context)
  inputSchema: JSONSchema                   // Zod-validated input schema
  async call(input, context): ToolResult    // Execute the tool
  isEnabled(context): boolean               // Whether tool is available
  userFacingName(): string                  // Display name
  isReadOnly(): boolean                     // Whether tool modifies state

  // Optional
  concurrencyConfig?: ConcurrencyConfig     // Concurrent execution rules
  maxResultSizeChars?: number               // Output size limit
  hasPermission?(input): PermissionResult   // Custom permission check
  getToolUseSummary?(input): string         // UI summary text
}
```

### Complete Tool Registry

The following tools are registered in `src/tools.ts`:

| Tool Name | Category | Read-Only | Concurrent | Permission Mode |
|-----------|----------|-----------|------------|-----------------|
| `Bash` | Shell execution | No | No (sequential) | Per-command classification |
| `Read` | File system | Yes | Yes | Always allowed |
| `Edit` | File system | No | No (file locking) | Ask (write) |
| `Write` | File system | No | No (file locking) | Ask (write) |
| `Glob` | File system | Yes | Yes | Always allowed |
| `Grep` | File system | Yes | Yes | Always allowed |
| `Agent` | Orchestration | No | Yes (subagent) | Depends on subagent tools |
| `SendMessage` | Orchestration | No | No | Inherits from Agent |
| `AskUserQuestion` | Interaction | No | No | Always allowed |
| `EnterPlanMode` | Planning | No | No | Always allowed |
| `ExitPlanMode` | Planning | No | No | Always allowed |
| `EnterWorktree` | Git | No | No | Ask |
| `ExitWorktree` | Git | No | No | Ask |
| `WebFetch` | Network | Yes | Yes | Domain-checked |
| `WebSearch` | Network | Yes | Yes | Always allowed |
| `ToolSearch` | Meta | Yes | Yes | Always allowed |
| `Skill` | Meta | No | No | Depends on skill |
| `Config` | Configuration | No | No | Always allowed |
| `TaskCreate` | Task management | No | No | Always allowed |
| `TaskGet` | Task management | Yes | Yes | Always allowed |
| `TaskList` | Task management | Yes | Yes | Always allowed |
| `TaskUpdate` | Task management | No | No | Always allowed |
| `TaskStop` | Task management | No | No | Always allowed |
| `TaskOutput` | Task management | Yes | Yes | Always allowed |
| `TodoWrite` | Todo management | No | No | Always allowed |
| `NotebookEdit` | Jupyter | No | No | Ask (write) |
| `Sleep` | Proactive mode | No | No | Always allowed |
| `Brief` | Output mode | No | No | Always allowed |
| `SyntheticOutput` | Internal | No | No | Internal only |
| `MCPTool` | MCP proxy | Varies | Yes | Per-server config |
| `McpAuth` | MCP auth | No | No | Always allowed |
| `ListMcpResources` | MCP | Yes | Yes | Always allowed |
| `ReadMcpResource` | MCP | Yes | Yes | Always allowed |
| `CronCreate` | Scheduling | No | No | Ask |
| `CronDelete` | Scheduling | No | No | Ask |
| `CronList` | Scheduling | Yes | Yes | Always allowed |
| `RemoteTrigger` | Remote | No | No | Ask |
| `LSP` | Language server | Yes | Yes | Gated by `ENABLE_LSP_TOOL` |
| `PowerShell` | Shell (Windows) | No | No | Per-command classification |
| `REPL` | Simple mode | No | No | Combined Read+Edit+Shell |
| `TeamCreate` | Teams | No | No | Ask |
| `TeamDelete` | Teams | No | No | Ask |

### Tool Dispatch Pipeline

```
Model outputs tool_use block
    │
    ▼
StreamingToolExecutor detects tool_use content_block_start
    │
    ▼
Accumulate input JSON via content_block_delta events
    │
    ▼
content_block_stop → Parse complete JSON input
    │
    ▼
Validate input against tool's Zod schema
    │  ├─ Invalid → Return error result to model
    │  └─ Valid → Continue
    ▼
Permission check (canUseTool)
    │  ├─ Denied → Return denial message to model
    │  ├─ Ask user → Show permission dialog → wait for response
    │  └─ Allowed → Continue
    ▼
Execute tool.call(input, context)
    │
    ▼
Serialize result (text, images, or error)
    │
    ▼
Apply size limits (truncation if needed)
    │  - DEFAULT_MAX_RESULT_SIZE_CHARS: 50,000
    │  - MAX_TOOL_RESULT_TOKENS: 100,000
    │  - MAX_TOOL_RESULT_BYTES: 400,000 (100K tokens × 4 bytes)
    ▼
Persist large results to disk if threshold exceeded
    │  - PREVIEW_SIZE_BYTES: 2,000 (inline preview)
    │  - Remainder stored in tool-results/ directory
    ▼
Return tool_result content block to conversation
```

### StreamingToolExecutor

The `StreamingToolExecutor` is the core data structure managing concurrent tool execution during API response streaming.

**State machine per tool use**:
```
PENDING → INPUT_STREAMING → INPUT_COMPLETE → PERMISSION_CHECK → EXECUTING → COMPLETE
                                                                          └→ ERROR
```

**Concurrency gate**: Tools marked as non-concurrent (Bash, Edit, Write) execute sequentially. Read-only tools (Read, Glob, Grep) execute in parallel. The maximum concurrency is configurable via `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`.

**Abort tree**: Each tool execution gets an `AbortController` that is a child of the stream's controller. Aborting the parent (user interrupt) cascades to all running tools. Individual tool timeouts create their own abort signals.

**Key behaviors**:
- Tools start executing as soon as their input is complete (during streaming), not after the full message
- If the model emits `end_turn` with tools still running, the executor waits for all tools
- Tool results that arrive after the stream ends are still collected and injected

### Tool Result Serialization

Tool results follow this format:
```typescript
{
  type: "tool_result",
  tool_use_id: string,     // Matches the tool_use block's id
  content: [
    { type: "text", text: string }     // Text result
    // OR
    { type: "image", source: { type: "base64", media_type: string, data: string } }
  ],
  is_error?: boolean       // Set true for error results
}
```

**Truncation**: Results exceeding `MAX_TOOL_RESULT_TOKENS` (100K tokens) are truncated with a message:
```
[Output truncated to X tokens. Full output saved to <path>. Use Read tool to view specific sections.]
```

**Disk persistence**: Results exceeding the per-message budget (`MAX_TOOL_RESULTS_PER_MESSAGE_CHARS`: 200K chars) are persisted to disk with an inline preview.

---

## 6. Individual Tool Deep-Dives

### Bash Tool

The Bash tool is the most complex tool in the system, with dedicated safety classification, permission checking, and output handling.

**Shell spawning**: Uses `Bun.spawn()` with a persistent shell process per session. The shell type is detected from the user's `SHELL` environment variable (defaulting to `bash`). A single shell process is reused across tool calls to maintain working directory and environment state.

**Environment setup**: The shell inherits a scrubbed environment via `subprocessEnv()`. In GitHub Actions, sensitive variables (`ANTHROPIC_API_KEY`, AWS credentials, OIDC tokens) are stripped to prevent `${VAR}` expansion exfiltration attacks.

**Timeout**: Default 30 minutes (`DEFAULT_TIMEOUT = 30 * 60 * 1000`). Configurable via the `timeout` parameter. Commands running longer are killed.

**Output capture and truncation**:
- Default output limit: `BASH_MAX_OUTPUT_DEFAULT = 30,000` characters
- Upper limit: `BASH_MAX_OUTPUT_UPPER_LIMIT = 150,000` characters
- Output is split into stdout and stderr with XML-like tags: `<bash_stdout>`, `<bash_stderr>`
- Very large outputs are persisted to disk (`MAX_PERSISTED_SIZE = 64MB`)

**CWD persistence**: Each bash command resets to the project root CWD at the start (for subagents). The shell maintains its own CWD between calls for the main agent. `cd` within a command persists for subsequent commands.

**Background execution**: Commands can be run in the background via `run_in_background: true`. In assistant/KAIROS mode, blocking commands auto-background after `ASSISTANT_BLOCKING_BUDGET_MS` (15 seconds). Background commands write to disk and are polled by a file-size watchdog (`SIZE_WATCHDOG_INTERVAL_MS = 5,000ms`).

**Safety classification**: A multi-layer static analysis system:
1. **AST parsing** (`src/utils/bash/bashParser.ts`): Commands are parsed into an AST with a 50ms timeout (`PARSE_TIMEOUT_MS`) and 50,000 node limit (`MAX_NODES`). If parsing fails, falls back to conservative classification.
2. **Dangerous pattern detection**: Checks for command injection vectors (backticks, `$()`, process substitution), obfuscated flags, control characters, Unicode whitespace, IFS injection.
3. **Read-only validation**: Determines if a command only reads data (safe for auto-approval). Checks against known read-only commands, validates no output redirection, no destructive flags.
4. **Permission rule matching**: Commands are matched against user-configured allow/deny rules with wildcard support. Rules like `Bash(npm test:*)` match command prefixes.

**Sandbox enforcement**: When enabled, Bash commands run inside a sandbox (macOS: Apple Sandbox via `sandbox-exec`, Linux: bubblewrap `bwrap`). The sandbox indicator shows in the prompt when active (`CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR`).

### Edit Tool

**Matching algorithm**: The Edit tool performs exact string matching of `old_string` against the file content. The match must be unique — if `old_string` appears more than once, the edit fails with an error message listing the line numbers of all matches.

**Uniqueness check**: The tool searches for all occurrences of `old_string` in the file. If count > 1 and `replace_all` is false, returns an error instructing the model to provide more surrounding context.

**`replace_all` mode**: When true, replaces all occurrences of `old_string` with `new_string`. Useful for renaming variables/functions throughout a file.

**Concurrent safety**: File edits are serialized per-file to prevent concurrent modifications. A file lock ensures only one edit operation runs per file at a time.

**Encoding**: Files are read and written in UTF-8. The tool detects file encoding and preserves it.

**De-sanitization**: Before matching, the tool applies the `DESANITIZATIONS` table to reverse API-side string sanitization. The complete de-sanitization table:

| Sanitized Form | Target |
|---------------|--------|
| `<fnr>` | `<function_results>` |
| `<n>` | `<name>` |
| `</n>` | `</name>` |
| `<o>` | `<output>` |
| `</o>` | `</output>` |
| `<e>` | `<error>` |
| `</e>` | `</error>` |
| `<s>` | `<system>` |
| `</s>` | `</system>` |
| `<r>` | `<result>` |
| `</r>` | `</result>` |
| `< META_START >` | `

**File size limit**: `MAX_EDIT_FILE_SIZE = 1GB` (stat bytes). The runtime string limit is ~1 billion characters.

**Diff snippet**: After each edit, a diff snippet is attached for the model to verify. Capped at `DIFF_SNIPPET_MAX_BYTES = 8,192` bytes to prevent bloating context with format-on-save changes.

### Read Tool

**Supported file types**: Text files, images (PNG, JPG — rendered visually), PDFs (page extraction with `pages` parameter), Jupyter notebooks (.ipynb — returns all cells with outputs).

**Size limits**:
- Default read: up to 2,000 lines (`MAX_LINES_TO_READ`)
- Default output tokens: 25,000 (`DEFAULT_MAX_OUTPUT_TOKENS`)
- Offset/limit parameters for partial reads of large files
- PDF max pages per read: 20 (`PDF_MAX_PAGES_PER_READ`)
- PDF max total pages: 100 (`API_PDF_MAX_PAGES`)

**Cyber risk mitigation**: When reading files that could contain prompt injection (downloaded files, untrusted sources), a `CYBER_RISK_MITIGATION_REMINDER` is injected warning the model to be cautious.

### Write Tool

**Overwrite behavior**: Always overwrites the entire file. The model must read a file before writing to it (enforced by checking read state cache).

**Permissions**: Requires write permission for the target path. Same path validation as Edit (UNC blocking, tilde rejection, etc.).

### Agent Tool

**Subagent lifecycle**: Each Agent invocation starts a fresh conversation. The subagent gets its own tools (configurable via `allowed-tools`), model selection, and permission context. Results are returned as a single text message.

**Built-in agent types**:

| Type | Purpose | Key Properties |
|------|---------|---------------|
| `general-purpose` | Research, code search, multi-step tasks | All tools |
| `Explore` | Fast codebase exploration | Read-only tools, no Edit/Write/Agent |
| `Plan` | Architecture and implementation planning | Read-only tools, no Edit/Write |
| `claude-code-guide` | Answer questions about Claude Code | Web tools + Read |
| `statusline-setup` | Configure status line | Read + Edit only |

**Nesting limits**: Agents can spawn sub-agents, but there's a depth limit enforced by the permission system.

**Worktree isolation**: `isolation: "worktree"` creates a temporary git worktree so the agent works on an isolated copy.

**Inter-agent communication**: Via `SendMessage` tool — messages are queued in a file-based mailbox system.

### Grep Tool

**Implementation**: Wraps `ripgrep` (`rg`) with a managed subprocess. Falls back to bundled ripgrep if system `rg` is not available.

**Output modes**: `content` (matching lines), `files_with_matches` (file paths only), `count` (match counts).

**Default head limit**: 250 results (`DEFAULT_HEAD_LIMIT`) — prevents context bloat from broad searches.

### WebFetch Tool

**Caching**: LRU cache with 15-minute TTL and 50MB size limit.
- `CACHE_TTL_MS = 15 * 60 * 1000`
- `MAX_CACHE_SIZE_BYTES = 50 * 1024 * 1024`

**Content limits**:
- `MAX_HTTP_CONTENT_LENGTH = 10MB` — resource consumption guardrail
- `MAX_MARKDOWN_LENGTH = 100,000` characters — post-conversion truncation
- `FETCH_TIMEOUT_MS = 60,000` — request timeout
- `MAX_REDIRECTS = 10` — prevents redirect loops

**Domain checking**: Preflight check against Anthropic's domain blocklist with separate timeout (`DOMAIN_CHECK_TIMEOUT_MS = 10,000`). Results cached in `DOMAIN_CHECK_CACHE` (128 entries, 5-min TTL).

### ToolSearch (Deferred Loading)

**Mechanism**: When many tools are registered (especially MCP tools), most are sent to the API with `defer_loading: true` — only name and description, no full schema. The model calls `ToolSearch` to fetch complete schemas on demand.

**Query syntax**: `"select:Read,Edit,Grep"` for exact selection, or keyword search with `max_results` (default 5).

---

## 7. Permission & Safety Model — Complete Specification

### Permission Modes

| Mode | Behavior | Availability |
|------|----------|-------------|
| `default` | Prompts user for every tool use | All users |
| `acceptEdits` | Auto-allows file edits in working directory; prompts for shell | All users |
| `plan` | Blocks tool execution, only planning | All users |
| `bypassPermissions` | Skips all permission checks | Gated by killswitch |
| `dontAsk` | Converts `ask` → `deny` (never prompts) | All users |
| `auto` | AI classifier decides (internal) | Ant-only, `TRANSCRIPT_CLASSIFIER` flag |
| `bubble` | Swarm delegation context (internal) | Internal only |

**Mode cycling** (Shift+Tab): `default → acceptEdits → plan → [bypassPermissions] → default`

### Permission Rule Sources (Priority Order)

```typescript
type PermissionRuleSource =
  | 'policySettings'     // Enterprise managed (highest priority)
  | 'flagSettings'       // --settings CLI flag
  | 'localSettings'      // .claude/settings.local.json (gitignored)
  | 'projectSettings'    // .claude/settings.json (shared)
  | 'userSettings'       // ~/.claude/settings.json (global)
  | 'cliArg'             // --allowed-tools flag
  | 'command'            // Slash command frontmatter
  | 'session'            // In-memory, current session only
```

### Permission Decision Flow

```
Tool call arrives
    │
    ▼
1. Check DENY rules (all sources)
    │  └─ Match → DENY (absolute, cannot override)
    ▼
2. Check ASK rules (all sources)
    │  └─ Match → ASK (always prompt user)
    ▼
3. Check ALLOW rules (all sources)
    │  └─ Match → ALLOW
    ▼
4. Tool-specific checkPermissions()
    │  ├─ Bash → parse command, check wildcards, read-only validation
    │  ├─ File tools → path validation, dangerous file check
    │  └─ Other → tool-specific logic
    ▼
5. Mode-based transformation
    │  ├─ dontAsk → convert ask → deny
    │  ├─ auto → invoke two-stage classifier
    │  └─ default → show permission dialog
    ▼
Result: allow | deny | ask
```

### Auto-Mode Classifier (Two-Stage)

**Stage 1 ("fast")**: Quick yes/no with `max_tokens=64` and stop sequences. If allowed, return immediately. If blocked, escalate to Stage 2.

**Stage 2 ("thinking")**: Chain-of-thought reasoning with `<thinking>` tags. Reduces false positives.

**Critical defense — transcript exclusion**: Assistant text is excluded from the classifier transcript. Only `tool_use` blocks from assistant messages enter. This prevents the model from crafting text to manipulate the classifier.

**Fail behavior**: Fail-closed by default (classifier unavailable = deny). After 3 consecutive or 20 total denials, falls back to user prompting.

### Bash Safety Classification

Multi-layer static analysis:
1. **AST parsing** with 50ms timeout and 50K node limit
2. **Dangerous pattern detection**: `$()`, backticks, process substitution, IFS injection, control characters, Unicode whitespace
3. **Read-only validation**: Known safe commands, no output redirection
4. **Wildcard rule matching**: User rules like `Bash(npm test:*)` match command prefixes

**Dangerous patterns stripped at auto-mode entry**:
```typescript
const DANGEROUS_BASH_PATTERNS = [
  'python', 'python3', 'node', 'deno', 'tsx', 'ruby', 'perl',
  'php', 'lua', 'npx', 'bunx', 'npm run', 'yarn run', 'pnpm run',
  'bun run', 'bash', 'sh', 'ssh', 'zsh', 'fish', 'eval', 'exec',
  'env', 'xargs', 'sudo',
]
```

### Prompt Injection Defenses

1. **Unicode sanitization**: NFKC normalization + stripping format/private-use/unassigned characters. Iterative until stable (max 10 rounds). Defends against ASCII Smuggling (HackerOne #3086545).
2. **System prompt instruction**: Model told to flag suspected injection attempts.
3. **Anti-distillation**: `fake_tools` opt-in injects synthetic tool blocks that poison distillation.
4. **Subprocess env scrubbing**: Strips `ANTHROPIC_API_KEY`, AWS creds, OIDC tokens from child processes in GitHub Actions.
5. **Path validation**: Blocks UNC paths, tilde variants, shell expansion syntax, glob patterns in writes.

---

## 8. MCP Integration — Complete Specification

### Configuration Schema

8 transport types as a discriminated union:

| Transport | Type Literal | Key Fields |
|-----------|-------------|------------|
| stdio | `'stdio'` (default) | `command`, `args[]`, `env?` |
| SSE | `'sse'` | `url`, `headers?`, `headersHelper?`, `oauth?` |
| SSE-IDE | `'sse-ide'` | `url`, `ideName` |
| HTTP (Streamable) | `'http'` | `url`, `headers?`, `headersHelper?`, `oauth?` |
| WebSocket | `'ws'` | `url`, `headers?` |
| WebSocket-IDE | `'ws-ide'` | `url`, `ideName`, `authToken?` |
| SDK | `'sdk'` | `name` |
| Claude.ai Proxy | `'claudeai-proxy'` | `url`, `id` |

**Config scopes**: `local | user | project | dynamic | enterprise | claudeai | managed`

### Connection Lifecycle

1. **Discovery**: Configs loaded from 7 sources (global settings, `.mcp.json`, enterprise managed, plugin MCP, claude.ai connectors)
2. **Connect**: Batch loading with different concurrency — local servers: 3 concurrent, remote: 20 concurrent
3. **Health**: `MAX_ERRORS_BEFORE_RECONNECT = 3` consecutive terminal errors trigger reconnect
4. **Reconnect**: Exponential backoff (`INITIAL_BACKOFF_MS = 1000`, `MAX_BACKOFF_MS = 30000`, `MAX_RECONNECT_ATTEMPTS = 5`)

### Tool Name Normalization

```
mcp__<normalizedServerName>__<normalizedToolName>
```
Normalization: `name.replace(/[^a-zA-Z0-9_-]/g, '_')`. Claude.ai servers get extra cleanup (collapse consecutive underscores).

### InProcessTransport

For Chrome MCP and Computer Use MCP servers, an in-process transport avoids spawning ~325MB subprocesses:
```typescript
class InProcessTransport implements Transport {
  async send(message) { queueMicrotask(() => this.peer?.onmessage?.(message)) }
}
```

### Elicitation Flow (Error -32042)

When an MCP tool call returns error code -32042 (UrlElicitationRequired):
1. Extract elicitation data from `error.data.elicitations[]`
2. Queue to UI for user consent
3. Retry up to `MAX_URL_ELICITATION_RETRIES = 3` times

---

## 9. Prompt Engineering Library

### System Prompt Architecture

The system prompt is assembled in layers:

```
Base system prompt ("You are Claude Code, Anthropic's official CLI...")
  + Environment details (CWD, date, platform, shell, OS version, model info)
  + Git status (current branch, recent commits, working tree status)
  + CLAUDE.md contents (project instructions, with @include support)
  + Feature-specific addenda:
    - Proactive mode instructions (when KAIROS enabled)
    - Brief mode instructions (when brief mode active)
    - Coordinator mode (replaces base prompt entirely)
  + System reminders (injected per-turn):
    - Deferred tools list
    - Available skills list
    - Open file notifications
    - Todo/task reminders
    - Memory file reminders
```

### Key Prompts (Verbatim)

**Default Agent Prompt**:
```
You are an agent for Claude Code, Anthropic's official CLI for Claude. Given the
user's message, you should use the tools available to complete the task. Complete
the task fully—don't gold-plate, but don't leave it half-done. When you complete
the task, respond with a concise report covering what was done and any key findings
— the caller will relay this to the user, so it only needs the essentials.
```

**Coordinator Mode System Prompt** (replaces default):
```
You are Claude Code, an AI assistant that orchestrates software engineering tasks
across multiple workers.

## 1. Your Role
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate work that you can
  handle without tools
```

**Proactive Mode Prompt**:
```
You are running autonomously. You will receive <tick> prompts that keep you alive
between turns — just treat them as "you're awake, what now?"

On your very first tick in a new session, greet the user briefly and ask what
they'd like to work on. Do not start exploring the codebase unprompted.

On subsequent wake-ups, look for useful work. Act on your best judgment rather
than asking for confirmation.
```

### Compaction Prompt (Critical Section)

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- Tool calls will be REJECTED and will waste your only turn — you will fail.
- Your entire response must be plain text: an <analysis> block followed by
  a <summary> block.
```

### XML Tag Taxonomy

| Tag | Writer | Reader | Meaning |
|-----|--------|--------|---------|
| `<task-notification>` | System | Model | Agent task completed/failed/killed |
| `<system-reminder>` | System | Model | Injected context (tools, skills, files) |
| `<tick>` | System | Model | Proactive mode wake-up signal |
| `<analysis>` | Model | System | Compaction scratchpad (stripped before summary) |
| `<summary>` | Model | System | Compaction output (kept) |
| `<bash_stdout>` | Tool | Model | Bash command standard output |
| `<bash_stderr>` | Tool | Model | Bash command standard error |
| `<thinking>` | Model | System | Extended thinking (may be redacted) |

---

## 10. Configuration & Extensibility

### Config File Hierarchy (Precedence: highest first)

| Source | Location | Scope |
|--------|----------|-------|
| Managed policy | MDM (macOS plist, Windows registry) or `CLAUDE_CODE_MANAGED_SETTINGS_PATH` | Enterprise |
| CLI flags | `--settings`, `--allowed-tools` | Session |
| Local settings | `.claude/settings.local.json` (gitignored) | Project-local |
| Project settings | `.claude/settings.json` | Project (shared) |
| User settings | `~/.claude/settings.json` | Global |
| Environment vars | `ANTHROPIC_API_KEY`, `CLAUDE_CODE_*`, etc. | Process |

### CLAUDE.md System

**Discovery order**: `CLAUDE.md` in project root → `.claude/CLAUDE.md` → parent directories up to home → user `~/.claude/CLAUDE.md`

**@include directive**: `@include path/to/file.md` injects contents inline.

**Token budget**: CLAUDE.md contents are included in the system prompt, subject to the overall context window budget.

### Hook System

**Events**: `PreToolUse`, `PostToolUse`, `PostToolFailure`, `PreCompact`, `SessionStart`, `SessionEnd`, `Notification`, `Elicitation`, `ElicitationResult`, `StopAgent`

**Hook types**: Shell command, HTTP webhook, agent hook (runs Claude as the hook)

**Execution**: Hooks run with timeout `TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 minutes`. Session end hooks have shorter timeout: `SESSION_END_HOOK_TIMEOUT_MS_DEFAULT = 1500ms`.

**Security**: Hook output is capped at `MAX_HOOK_OUTPUT_LENGTH = 10,000` characters.

### Skill System

**Sources**: Bundled skills (shipped in binary), built-in plugin skills, user skills (`.claude/skills/`), marketplace plugin skills, dynamic skills (discovered at runtime), MCP skills.

**Key bundled skills**: `update-config`, `verify`, `debug`, `skillify`, `simplify`, `batch`, `remember`, `stuck`, `loop`, `schedule`, `claude-api`, `claude-in-chrome`

### Slash Command Catalog (100+ commands)

Major categories:
- **Session**: `/compact`, `/clear`, `/resume`, `/session`, `/export`, `/rewind`
- **Model**: `/model`, `/fast`, `/effort`, `/thinking`
- **Config**: `/config`, `/permissions`, `/hooks`, `/mcp`, `/plugin`
- **UI**: `/help`, `/theme`, `/color`, `/vim`, `/voice`, `/buddy`
- **Info**: `/cost`, `/usage`, `/stats`, `/files`, `/context`, `/doctor`
- **Auth**: `/login`, `/logout`
- **Review**: `/review`, `/security-review`
- **IDE**: `/ide`

---

## 11. The Speculation System

### Three-Stage Pipeline

1. **Suggestion generation**: After each assistant turn, a forked agent predicts what the user will type next. Uses the parent's prompt cache (no parameter overrides — PR #18143 showed that even setting `effort:'low'` caused a 45x spike in cache writes).

2. **Speculative execution**: The suggestion is run through the full agent loop on a copy-on-write overlay filesystem.

3. **Accept/reject**: If the user accepts, overlay files are copied to the real filesystem and pre-computed messages are injected. If rejected, overlay is discarded.

### Copy-on-Write Overlay

```
Overlay directory: <tempdir>/speculation/<pid>/<speculationId>/

Write tools → file_path rewritten to overlay (original copied first if exists)
Read tools → redirected to overlay only if file was previously written there
Writes outside cwd → denied
```

### Boundary Detection

Speculation stops at four boundaries:
- **`complete`**: Agent finished normally (all results usable)
- **`bash`**: Non-read-only bash command (would need permission)
- **`edit`**: File edit when user hasn't enabled `acceptEdits`
- **`denied_tool`**: Tool not allowed in speculation

### Cache Sharing

The suggestion agent and speculation agent share the parent's prompt cache by sending identical cache-key params. The single `cache_control` marker is placed at `messages.length - 2` (not `-1`) for forked agents so the fork piggybacks on the existing cache entry without polluting it.

### Pipelined Suggestions

After speculation completes, a pipelined next-suggestion is generated while waiting for user acceptance. If accepted and the pipeline is ready, speculation starts on it immediately — creating a chain of speculations.

### Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_SPECULATION_TURNS` | 20 | Max turns in speculative execution |
| `MAX_SPECULATION_MESSAGES` | 100 | Max messages in speculative execution |
| `MAX_PARENT_UNCACHED_TOKENS` | 10,000 | Suppress suggestions when parent cache is cold |

---

## 12. Multi-Agent Orchestration

### Subagent Lifecycle

Each `Agent` tool invocation creates a fresh conversation:
1. New conversation with selected tools and model
2. Inherits permission context from parent (deny rules bubble up)
3. Runs the full agent loop independently
4. Returns final text as tool result to parent

### Built-in Agent Types

| Type | Tools | Model | Purpose |
|------|-------|-------|---------|
| `general-purpose` | All | Inherited | Research, code search, multi-step |
| `Explore` | Read-only (no Edit/Write/Agent) | Sonnet | Fast codebase exploration |
| `Plan` | Read-only (no Edit/Write/Agent) | Inherited | Architecture planning |
| `claude-code-guide` | Glob, Grep, Read, WebFetch, WebSearch | Inherited | Claude Code help |
| `statusline-setup` | Read, Edit | Inherited | Status line config |

### Coordinator Mode

When enabled, the system prompt is replaced entirely with coordinator instructions. The coordinator:
- Spawns workers via `Agent` tool
- Continues workers via `SendMessage` tool
- Tracks worker status via `TaskCreate/Get/List/Update/Stop`
- Follows a phased workflow: Research → Synthesis → Implementation → Verification

### Worktree Isolation

`isolation: "worktree"` creates a temporary git worktree:
1. `git worktree add` with a generated branch name
2. Symlinks for `node_modules`, build artifacts
3. Agent operates in isolated directory
4. On completion, worktree branch and directory returned
5. Cleanup removes worktree if no changes were made

### Fork Agents

Fork agents inherit the parent's conversation context (all messages so far) and continue from there. Used for parallel exploration of different approaches.

---

## 13. Session Persistence & Recovery

### JSONL Format

Each session is persisted as a JSONL file at `~/.claude/projects/<hash>/sessions/<id>.jsonl`. Each line is a JSON object representing one message:

```json
{"type": "user", "message": {"role": "user", "content": [...]}, "uuid": "...", "timestamp": "..."}
{"type": "assistant", "message": {"role": "assistant", "content": [...]}, "uuid": "...", "sessionId": "..."}
```

**Size limits**: `MAX_JSONL_READ_BYTES = 100MB`, `MAX_TRANSCRIPT_READ_BYTES = 50MB`, `MAX_TOMBSTONE_REWRITE_BYTES = 50MB`

### Crash Recovery (`--resume`)

1. Read JSONL transcript from disk
2. Parse all messages, filtering tombstoned (orphaned) messages
3. Detect the last compaction boundary (if any)
4. Resume from compaction boundary or beginning
5. Restore session memory if available
6. Make first new API call with restored conversation

### Graceful Shutdown

Tiered timeouts in `gracefulShutdown.ts`:
1. **SIGINT/SIGTERM** → Set shutdown flag, wait for current turn to complete
2. **Second SIGINT** → Force abort current operation
3. **Failsafe timer** → `process.exit()` if cleanup takes too long
4. **Orphan check** → Periodic check for orphaned child processes

Signal handlers:
- `SIGINT` → graceful shutdown (first press), force exit (second press)
- `SIGTERM` → graceful shutdown
- `uncaughtException` → log event, force exit
- `unhandledRejection` → log event, force exit

### Session Memory Persistence

Session memory files are stored at `~/.claude/projects/<hash>/session-memory/<sessionId>.md`. Extracted at natural breakpoints and on session end. Used to bootstrap context on `--resume`.

---

## 14. Performance Architecture

### Streaming (Raw Events)

Claude Code **bypasses the Anthropic SDK's streaming abstraction** and processes raw SSE events directly. This allows:
- Tool execution to begin during streaming (not after full message)
- Finer control over idle/stall detection
- Custom retry and fallback behavior

**Idle detection**: `STREAM_IDLE_TIMEOUT_MS` (90 seconds) kills hung streams. `STALL_THRESHOLD_MS` (30 seconds) logs a stall warning.

### Prompt Caching

**Two strategies**:
1. **5-minute ephemeral cache** (default): Standard prompt caching
2. **1-hour cache** (eligible users): Extended TTL for long sessions

**Single marker strategy**: Exactly one `cache_control` marker per request, placed on the last message. Forked agents place it on the second-to-last message to piggyback without polluting.

**Cache break detection**: Pre-call hashing of system prompt, tool schemas, model, betas, effort. Post-call comparison of `cache_read_tokens` drop (>5% AND >2000 tokens absolute).

### Terminal Rendering

**Custom Ink fork** with Yoga layout (pure TypeScript port — no WASM):
- Double-buffered frames at `FRAME_INTERVAL_MS = 16ms` (~60fps)
- Scroll fast path using **DECSTBM blit+shift**: copies previous frame's scroll region, shifts by delta, renders only edge rows
- **Viewport culling**: Children outside scroll viewport are skipped entirely
- **Line width cache**: `MAX_CACHE_SIZE = 4096` entries, avoids redundant `stringWidth()` calls
- Virtual scroll with `OVERSCAN_ROWS = 80` and `MAX_MOUNTED_ITEMS = 300`

### Startup Optimization

**Zero-import fast paths** in CLI entrypoint:
- `--version`: Prints build-time constant, zero module loading
- `--dump-system-prompt`: Loads only config + prompts
- `--daemon-worker`: Loads only worker registry

**Build-time DCE**: `feature('FLAG')` calls enable dead code elimination. Internal-only features stripped from external builds.

**Parallel initialization**: Config loading, auth, feature flags, MCP connections all run concurrently during bootstrap.

### Caching Layers

| Cache | Scope | Size | TTL | Purpose |
|-------|-------|------|-----|---------|
| WebFetch URL cache | Process | 50MB | 15 min | Avoid re-fetching pages |
| Domain check cache | Process | 128 entries | 5 min | Avoid duplicate blocklist checks |
| File state cache | Process | 100 entries / 25MB | Session | Track read file contents |
| Highlight cache | Process | 500 entries | Session | Syntax highlighting results |
| Token cache | Process | 500 entries | Session | Markdown token parsing |
| Tool schema cache | Process | Per-tool | Session | Memoized tool JSON schemas |
| GrowthBook cache | Process | N/A | 6 hours | Feature flag values |
| Policy limits cache | Disk | 1 file | 1 hour | Enterprise policy limits |
| Metrics opt-out cache | Disk | 1 file | 24 hours | Org metrics settings |
| Plugin install counts | Disk | 1 file | 24 hours | Marketplace install counts |
| Stats cache | Disk | 1 file | Session | Session statistics |

---

## 15. Error Handling & Resilience

### API Error Classification

| Error | HTTP Status | Recovery |
|-------|------------|----------|
| Rate limit | 429 | Retry with backoff, respect `Retry-After` |
| Overloaded | 529 | Max 3 retries for foreground, 0 for background |
| Server error | 500 | Retry with exponential backoff |
| Prompt too long | 400 (specific) | Reactive compact → retry |
| Max tokens | N/A (stop_reason) | Inject continuation message, escalate tokens |
| Auth error | 401/403 | Refresh credentials, retry |
| Timeout | N/A | Non-streaming fallback for `MAX_NON_STREAMING_TOKENS = 64,000` |
| Invalid request | 400 | Return error to user |

### Retry Configuration

```typescript
const DEFAULT_MAX_RETRIES = 10
const MAX_529_RETRIES = 3
const BASE_DELAY_MS = 500
const PERSISTENT_MAX_BACKOFF_MS = 5 * 60 * 1000  // 5 minutes
const PERSISTENT_RESET_CAP_MS = 6 * 60 * 60 * 1000  // 6 hours
const HEARTBEAT_INTERVAL_MS = 30_000
```

**529 special handling**: Foreground sources (user-facing) retry up to 3 times. Background sources (agents, speculation) bail immediately — during capacity cascade each retry is 3-10x gateway amplification.

### Circuit Breakers

| Breaker | Threshold | Effect |
|---------|-----------|--------|
| Autocompact failures | 3 consecutive | Stop trying autocompact |
| Classifier denials | 3 consecutive / 20 total | Fall back to user prompting |
| MCP connection errors | 3 consecutive terminal | Trigger reconnect |
| Environment recreations | 3 per session | Stop recreating |
| Max output recovery | 3 attempts | Give up, return partial |

---

## 16. Analytics, Telemetry & Feature Flags

### Event Pipeline

```typescript
logEvent(eventName, metadata)  // Synchronous enqueue
logEventAsync(eventName, metadata)  // Async enqueue with await
```

Events queue before sink attachment. `attachAnalyticsSink()` drains asynchronously.

**Total unique events**: 666 (as of extraction date)

### Two Backends

1. **Datadog** (1P privileged): `DEFAULT_FLUSH_INTERVAL_MS = 15,000`, `MAX_BATCH_SIZE = 100`
2. **First-party OTLP**: `DEFAULT_LOGS_EXPORT_INTERVAL_MS = 10,000`, `DEFAULT_MAX_EXPORT_BATCH_SIZE = 200`, `DEFAULT_MAX_QUEUE_SIZE = 8,192`

### PII Protections

- MCP tool names default to `mcp_tool` unless logging explicitly allowed
- Tool inputs truncated: `TOOL_INPUT_STRING_TRUNCATE_AT = 512`, `TOOL_INPUT_MAX_DEPTH = 2`
- `_PROTO_*` fields stripped before general sinks, routed only to privileged 1P columns

### GrowthBook Integration

**Refresh interval**: 6 hours (`GROWTHBOOK_REFRESH_INTERVAL_MS`)

**Evaluation caching**: Feature flag values cached in memory. `feature('FLAG')` used for build-time DCE.

**Attributes**: `orgId`, `userId`, `entrypoint`, `version`, `platform`, `isRemote`, `isBridge`, etc.

### Key Feature Flags

| Flag | Type | Controls |
|------|------|---------|
| `tengu_session_memory` | Boolean | Session memory extraction |
| `tengu_sm_compact` | Boolean | Session memory compaction |
| `tengu_iron_gate_closed` | Boolean | Auto-mode fail-closed behavior |
| `TRANSCRIPT_CLASSIFIER` | Build | Auto-mode classifier availability |
| `COORDINATOR_MODE` | Build | Coordinator mode |
| `PROACTIVE` / `KAIROS` | Build | Proactive agent / assistant mode |
| `AGENT_TRIGGERS` | Build | Local cron scheduling |
| `AGENT_TRIGGERS_REMOTE` | Build | Remote trigger scheduling |
| `FORK_SUBAGENT` | Build | Fork agent support |
| `BRIDGE_MODE` | Build | Bridge supervisor mode |
| `DAEMON` | Build | Daemon mode |
| `BUDDY` | Build | Companion sprite |
| `VOICE_MODE` | Build | Voice input |
| `WEB_BROWSER_TOOL` | Build | Browser automation tool |

---

## 17. The Bridge & Remote Control

### Architecture

The bridge is a durable session orchestration system, not a thin transport shim:

**RemoteSessionManager** (client side):
- WebSocket for incoming messages and control traffic
- HTTP POST for outbound user events
- Explicit permission round-trips (control_request → control_response)

**Bridge Loop** (supervisor side):
- Polls for work from remote control service
- Spawns local Claude child sessions
- Tracks work leases via ingress JWTs
- Heartbeats active work (separate from polling)
- Manages worktrees and cleanup

### Backoff Configuration

```typescript
connection initial: 2s, cap: 120s, give-up: 10m
general initial: 500ms, cap: 30s, give-up: 10m
```

### Session Types

- v1: Children receive refreshed OAuth directly
- v2: Children cannot use OAuth tokens, must be re-dispatched for fresh JWT

### Permission Protocol

Remote permission flow:
1. Remote sends `control_request` with subtype `can_use_tool`
2. Client stores by `request_id`, fires UI callback
3. Client sends `control_response` with `behavior: 'allow'` + `updatedInput` or `behavior: 'deny'` + `message`
4. Server may send `control_cancel_request` if no longer relevant

---

## 18. Voice, Vim, Buddy & UI Systems

### Voice Input

**Pipeline**: Microphone → SoX recording → WebSocket streaming STT → text → agent loop

**Constants**: `KEEPALIVE_INTERVAL_MS = 8,000`, `SILENCE_THRESHOLD = '3%'`, `RELEASE_TIMEOUT_MS = 200`, `FOCUS_SILENCE_TIMEOUT_MS = 5,000`

### Vim Mode

Vim emulation in the terminal input. `MAX_VIM_COUNT = 10,000` limits numeric prefixes.

### Buddy System

Companion sprite (`src/buddy/CompanionSprite.tsx`) — an animated ASCII character that responds to agent activity. Has speech bubbles, animations, and narrow-screen adaptations.

### Terminal UI Component Hierarchy

```
App (src/ink/components/App.tsx)
  └─ REPL (src/screens/REPL.tsx)
       ├─ Logo / Welcome
       ├─ Messages (virtual-scrolled list)
       │    ├─ UserPromptMessage
       │    ├─ AssistantTextMessage
       │    ├─ ToolUseMessage (Bash, Edit, Read, etc.)
       │    ├─ ToolResultMessage
       │    └─ SystemAPIErrorMessage
       ├─ Spinner (thinking/streaming indicator)
       ├─ PermissionDialog (when tool needs approval)
       ├─ PromptInput (text input with suggestions)
       │    ├─ Footer (mode, model, cost)
       │    └─ Suggestions overlay
       └─ TaskList (background agent status)
```

---

## 19. Daemon & Background Sessions

### Daemon Mode

Long-lived supervisor process. Spawns worker subprocesses as `claude --daemon-worker <kind>`. Worker types include `claude_code` and `claude_code_assistant`.

### Background Sessions

- `claude --bg` runs in tmux, detaches instead of exiting
- Session registry at `~/.claude/sessions/<pid>.json` tracks: pid, sessionId, cwd, startedAt, kind (`interactive | bg | daemon | daemon-worker`), optional messagingSocketPath
- `claude ps|logs|attach|kill` for management

### Proactive Agent (Ticks)

Periodic `<tick>` prompts keep the model alive between turns. The model must either do useful work or call `Sleep`. Tick prompts are hidden via `isMeta: true`.

### Cron Scheduling

Local cron via `CronCreate/Delete/List` tools:
- Persisted to `.claude/scheduled_tasks.json`
- `MAX_JOBS = 50` per project
- One-shot tasks auto-delete, recurring tasks expire after 7 days by default
- Per-project lock file prevents double-firing

### Remote Triggers

Via `RemoteTrigger` tool → claude.ai `/v1/code/triggers` API. Supports list, get, create, update, run.

---

## 20. Cross-System Interaction Matrix

| System A | System B | Interaction | Risk |
|----------|----------|-------------|------|
| Compaction | Speculation | Speculation aborted on compact | Medium — orphaned overlay |
| Compaction | Session Memory | SM extraction paused during compact | Low — coordinated |
| Permission | Speculation | Speculation denied at permission boundary | Low — by design |
| Agent Tool | Permission | Subagent inherits parent deny rules | Low — defense in depth |
| MCP | Permission | MCP tools go through same permission pipeline | Low |
| Bridge | Permission | Remote permission round-trip protocol | Medium — latency |
| Compaction | Hook System | PreCompact hooks run before summary | Medium — hook failure blocks compact |
| Stream | Tool Executor | Tools start during streaming | Medium — abort cascade complexity |
| Feature Flags | Everything | Kill switches can disable subsystems remotely | High — operational lever |

---

## 21. The Constants Bible

### Token Management

| Constant | Value | Purpose |
|----------|-------|---------|
| `MODEL_CONTEXT_WINDOW_DEFAULT` | 200,000 | Default context window |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | Headroom before autocompact |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | Compaction output budget |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | Post-compact file restoration |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | Files to re-read after compact |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | 12,000 | Session memory total budget |
| `MAX_SECTION_LENGTH` | 2,000 | Per-section session memory limit |
| `CAPPED_DEFAULT_MAX_TOKENS` | 8,000 | Initial max output tokens |
| `ESCALATED_MAX_TOKENS` | 64,000 | Escalated max output tokens |
| `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50,000 | Tool result character limit |
| `MAX_TOOL_RESULT_TOKENS` | 100,000 | Tool result token limit |
| `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | 200,000 | Per-message tool result budget |
| `COMPLETION_THRESHOLD` | 0.9 | Token budget completion threshold |
| `DIMINISHING_THRESHOLD` | 500 | Minimum incremental progress |

### Timeouts

| Constant | Value | Purpose |
|----------|-------|---------|
| `DEFAULT_TIMEOUT` (Shell) | 30 minutes | Bash command timeout |
| `DEFAULT_TIMEOUT_MS` (Bash tool) | 120,000 | Default Bash timeout |
| `MAX_TIMEOUT_MS` | 600,000 | Maximum Bash timeout |
| `STREAM_IDLE_TIMEOUT_MS` | 90,000 | Hung stream detection |
| `STALL_THRESHOLD_MS` | 30,000 | Stream stall warning |
| `TOOL_HOOK_EXECUTION_TIMEOUT_MS` | 600,000 | Hook execution timeout |
| `SESSION_END_HOOK_TIMEOUT_MS_DEFAULT` | 1,500 | Session end hook timeout |
| `MCP_REQUEST_TIMEOUT_MS` | 60,000 | MCP request timeout |
| `AUTH_REQUEST_TIMEOUT_MS` | 30,000 | OAuth request timeout |
| `FETCH_TIMEOUT_MS` (WebFetch) | 60,000 | HTTP fetch timeout |
| `CHORD_TIMEOUT_MS` | 1,000 | Keybinding chord timeout |

### Retry

| Constant | Value | Purpose |
|----------|-------|---------|
| `DEFAULT_MAX_RETRIES` | 10 | API retry limit |
| `MAX_529_RETRIES` | 3 | Overloaded retry limit |
| `BASE_DELAY_MS` | 500 | Retry base delay |
| `PERSISTENT_MAX_BACKOFF_MS` | 300,000 | Max persistent retry backoff |
| `MAX_RECONNECT_ATTEMPTS` (MCP) | 5 | MCP reconnect limit |
| `MAX_RECONNECT_ATTEMPTS` (Remote) | 5 | Remote WebSocket reconnect |

### UI Rendering

| Constant | Value | Purpose |
|----------|-------|---------|
| `FRAME_INTERVAL_MS` | 16 | ~60fps render throttle |
| `MAX_MOUNTED_ITEMS` | 300 | Virtual scroll mount limit |
| `OVERSCAN_ROWS` | 80 | Virtual scroll overscan |
| `MAX_MESSAGES_WITHOUT_VIRTUALIZATION` | 200 | Threshold for virtual scrolling |
| `MAX_MESSAGES_TO_SHOW_IN_TRANSCRIPT_MODE` | 30 | Transcript view limit |

### Security

| Constant | Value | Purpose |
|----------|-------|---------|
| `DENIAL_LIMITS.maxConsecutive` | 3 | Auto-mode consecutive denial limit |
| `DENIAL_LIMITS.maxTotal` | 20 | Auto-mode total denial limit |
| `MAX_SUBCOMMANDS_FOR_SECURITY_CHECK` | 50 | Bash compound command limit |
| `PARSE_TIMEOUT_MS` (Bash) | 50 | Bash AST parse timeout |
| `MAX_NODES` (Bash) | 50,000 | Bash AST node limit |
| `MAX_ITERATIONS` (Sanitization) | 10 | Unicode sanitization rounds |

---

## 22. The Model Behavioral Contract

### Assumptions the Code Makes About Model Behavior

| # | Assumption | Evidence | What Breaks If Violated |
|---|-----------|----------|------------------------|
| 1 | Model will use tool_use blocks, not text descriptions of tool calls | Tool dispatch only processes `tool_use` content blocks | Model describes actions instead of taking them |
| 2 | Model respects `end_turn` stop reason semantics | Loop termination depends on `stop_reason` | Infinite loops or premature termination |
| 3 | Model produces valid JSON in tool inputs | Input parsing with Zod validation | Tool execution fails, error injected |
| 4 | Model follows system prompt formatting instructions | System reminders, XML tags, response format | Broken parsing of compaction summaries, session memory |
| 5 | Model respects NO_TOOLS_PREAMBLE during compaction | Compaction prompt explicitly forbids tool calls | Wasted turns, failed compaction |
| 6 | Model produces reasonable token-length outputs | Max output escalation (8K → 64K) | Excessive cost, truncation recovery loops |
| 7 | Model can follow the de-sanitization table | Edit tool applies reverse mapping | Edit mismatches when model uses sanitized forms |
| 8 | Model understands XML-like tag conventions | `<thinking>`, `<analysis>`, `<summary>` | Failed compaction, lost thinking content |

### stop_reason Taxonomy

| Value | Meaning | Recovery Path |
|-------|---------|--------------|
| `end_turn` | Model finished normally | Check pending tools → run stop hooks → return |
| `tool_use` | Model wants tool results | Execute tools → inject results → continue |
| `max_tokens` | Output truncated | Inject recovery message → escalate → continue (≤3x) |
| `stop_sequence` | Hit a stop sequence | Return |

---

## 23. Design Patterns Catalog

### 1. Async Generator Agent Loop
The conversation is modeled as an async generator yielding stream events. Consumers can process incrementally without buffering the entire response.

### 2. Copy-on-Write Overlay (Speculation)
File writes during speculation go to a temporary overlay. Only copied to real FS on acceptance. Reads check overlay first, fall through to real FS.

### 3. Two-Stage Classifier
Fast path (low latency, may false-positive block) → slow path (chain-of-thought, higher accuracy). Reduces both latency and false positives.

### 4. Circuit Breaker Pattern
After N consecutive failures, stop trying. Applied to: autocompact (3), auto-mode denials (3/20), MCP connections (3).

### 5. Token Escalation
Start conservative (8K max output), escalate on demand (→ 64K). Saves cost on typical responses while allowing large outputs when needed.

### 6. Forked Agent with Cache Sharing
Background tasks (speculation, session memory, compaction) run as forked agents that share the parent's prompt cache by sending identical cache-key parameters.

### 7. Streaming Tool Execution
Tools begin execution when their input is complete (during API response streaming), not after the full message. Enables parallel work between model generation and tool execution.

### 8. Three-Tier Compaction
Microcompact (cheap, API-side) → Full compaction (LLM summary) → Session memory (pre-built). Each tier is tried before escalating.

### 9. Graceful Degradation
When subsystems fail, the system degrades rather than crashes: MCP server fails → tools unavailable but session continues. Classifier fails → fall back to user prompting. Compaction fails → continue with full context.

### 10. Build-Time Dead Code Elimination
`feature('FLAG')` calls resolved at build time by Bun's bundler. Internal features stripped from external builds, reducing binary size and attack surface.

### 11. Module-Scoped Singletons via `let`
~399 module-scoped mutable `let` declarations serve as implicit singletons. Enables state sharing without dependency injection, but makes testing harder.

### 12. WeakRef Abort Tree
Parent-child abort controllers use `WeakRef` so abandoned children can be garbage collected without explicit cleanup.

### 13. Prompt Cache Boundary Management
Single cache_control marker per request, placed strategically. Forked agents place marker one position earlier to piggyback without polluting.

### 14. Discriminated Union State Machines
MCP connection states, tool execution states, permission results all use TypeScript discriminated unions for exhaustive handling.

### 15. JSONL-as-Truth
Session transcripts stored as append-only JSONL. Tombstoning (not deletion) for message removal. Recovery reads from JSONL, not memory.

---

## 24. Architecture Decision Records

### ADR-1: Async Generator Loop (vs. Recursive/Promise Chain)
**Decision**: Model the agent loop as an `async function*` generator.
**Rationale**: Generators enable incremental yielding of stream events to the UI without buffering. They compose naturally with `for await...of` consumption. The loop's mutable `State` object is updated in-place each iteration.
**Tradeoff**: Generator semantics are less familiar to most TypeScript developers. Error handling across generator boundaries requires care.

### ADR-2: Raw Stream Processing (vs. SDK Streaming)
**Decision**: Bypass the Anthropic SDK's streaming abstraction and process raw SSE events.
**Rationale**: Enables tool execution during streaming (not after), finer idle/stall detection, custom retry/fallback, and non-streaming fallback for long responses.
**Tradeoff**: More code to maintain. Must track SDK changes manually.

### ADR-3: Three-Tier Compaction
**Decision**: Microcompact → Full compaction → Session memory, with automatic tier selection.
**Rationale**: Microcompact is cheapest (no API call for time-based, API-side for cached). Full compaction handles complex summarization. Session memory provides instant resume. Circuit breaker prevents runaway failures (250K wasted API calls/day discovered).
**Tradeoff**: Complex interaction rules between tiers. Multiple code paths for the same conceptual operation.

### ADR-4: Speculation with Copy-on-Write
**Decision**: Predict user input, pre-execute on overlay filesystem, accept/reject.
**Rationale**: Makes the UX feel instant for common operations. Cache sharing (PR #18143 lesson) keeps cost low. Boundary detection ensures safety.
**Tradeoff**: Wasted compute on rejected suggestions. Complex overlay management. Cache parameter sensitivity.

### ADR-5: Prompt Cache Boundary Strategy
**Decision**: Exactly one `cache_control` marker per request, shifted for forked agents.
**Rationale**: Multiple markers waste KV cache pages. Single marker at the last message maximizes reuse. Fork marker at position -2 enables piggyback without pollution.
**Tradeoff**: Less granular cache control. Cache break detection needed to diagnose issues.

### ADR-6: Build-Time Feature Flags via DCE
**Decision**: `feature('FLAG')` resolved at build time, dead code eliminated.
**Rationale**: Internal features (auto-mode classifier, daemon, coordinator) stripped from external builds. Reduces binary size and prevents access to internal surfaces.
**Tradeoff**: Two build variants to maintain. Feature flag changes require rebuild.

### ADR-7: Custom Ink Fork with Native Yoga
**Decision**: Fork Ink and rewrite Yoga layout in pure TypeScript.
**Rationale**: Original Ink + yoga-wasm had startup cost and didn't support needed optimizations (DECSTBM scroll, viewport culling, double buffering). Pure TS port is ~2500 lines covering the Ink-used subset.
**Tradeoff**: Must maintain fork. May diverge from upstream Ink improvements.

### ADR-8: JSONL Session Persistence
**Decision**: Append-only JSONL for session transcripts, tombstoning for removals.
**Rationale**: Append-only is crash-safe (partial writes are detectable). JSONL is human-readable and tooling-friendly. Tombstoning preserves audit trail.
**Tradeoff**: Files grow unboundedly. Recovery requires reading entire file.

### ADR-9: Hook System for Extensibility
**Decision**: Shell/HTTP/agent hooks on tool lifecycle events.
**Rationale**: Users need to customize behavior (CI gates, notifications, custom permissions) without modifying source. Hook events cover the full tool lifecycle.
**Tradeoff**: Hook failures can block operations. Timeout management is critical.

### ADR-10: Permission Classifier Transcript Exclusion
**Decision**: Exclude assistant text from auto-mode classifier input.
**Rationale**: Model-authored text could be crafted to manipulate the classifier into approving dangerous actions. Only tool_use blocks (structural, not free-text) enter the classifier.
**Tradeoff**: Classifier has less context about model reasoning. May increase false denials.

---

## Appendices

### Appendix A: Top 20 Hub Files (Most-Imported)

| Rank | Module | Importers |
|------|--------|-----------|
| 1 | `ink.js` | 292 |
| 2 | `utils/config.js` | 268 |
| 3 | `utils/debug.js` | 192 |
| 4 | `Tool.js` | 184 |
| 5 | `types/message.js` | 181 |
| 6 | `utils/errors.js` | 154 |
| 7 | `utils/log.js` | 154 |
| 8 | `state/AppState.js` | 150 |
| 9 | `commands.js` | 143 |
| 10 | `bootstrap/state.js` | 134 |
| 11 | `services/analytics/growthbook.js` | 128 |
| 12 | `utils/slowOperations.js` | 119 |
| 13 | `utils/envUtils.js` | 117 |
| 14 | `utils/messages.js` | 111 |
| 15 | `services/analytics/index.js` | 91 |
| 16 | `utils/auth.js` | 85 |
| 17 | `utils/format.js` | 83 |
| 18 | `utils/settings/settings.js` | 81 |
| 19 | `utils/lazySchema.js` | 72 |
| 20 | `constants/prompts.ts` | ~70 |

### Appendix B: File Count by Subsystem

| Directory | Files | Nature |
|-----------|-------|--------|
| `utils/` | ~215 | Shared utilities (largest subsystem) |
| `commands/` | ~195 | Slash commands (100+ commands) |
| `components/` | ~193 | React/Ink UI components |
| `tools/` | ~155 | Tool implementations (50+ tools) |
| `services/` | ~113 | Backend services (API, MCP, analytics, compact) |
| `hooks/` | ~104 | React hooks |
| `ink/` | ~93 | Custom Ink rendering engine |
| `bridge/` | ~31 | Bridge supervisor |
| `constants/` | ~21 | Shared constants |
| `skills/` | ~20 | Bundled skills |
| `cli/` | ~19 | CLI handlers and transports |
| `keybindings/` | ~14 | Keyboard shortcut system |
| `tasks/` | ~12 | Background task types |
| `types/` | ~11 | Type definitions |
| **Total** | **~1,350+** | |

### Appendix C: Longest Files (Complexity Proxies)

| Rank | File | Lines |
|------|------|-------|
| 1 | `src/cli/print.ts` | 5,595 |
| 2 | `src/screens/REPL.tsx` | 5,006 |
| 3 | `src/main.tsx` | 4,684 |
| 4 | `src/services/api/claude.ts` | 3,420 |
| 5 | `src/bridge/bridgeMain.ts` | 3,000 |
| 6 | `src/tools/BashTool/bashPermissions.ts` | 2,622 |
| 7 | `src/components/PromptInput/PromptInput.tsx` | 2,339 |
| 8 | `src/utils/auth.ts` | 2,003 |
| 9 | `src/utils/config.ts` | 1,818 |
| 10 | `src/query.ts` | 1,730 |

### Appendix D: Environment Variable Reference (Key Variables)

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Direct API authentication |
| `ANTHROPIC_MODEL` | Override model selection |
| `CLAUDE_CODE_USE_BEDROCK` | Route to AWS Bedrock |
| `CLAUDE_CODE_USE_VERTEX` | Route to GCP Vertex AI |
| `CLAUDE_CODE_ENTRYPOINT` | Execution mode label |
| `CLAUDE_CODE_REMOTE` | Remote session mode |
| `CLAUDE_CODE_SIMPLE` | Reduce tool surface (Bash + Read + Edit only) |
| `CLAUDE_CODE_PROACTIVE` | Enable proactive agent |
| `ENABLE_LSP_TOOL` | Enable LSP tool |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | Override context window |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Override max output tokens |

**Total unique environment variables**: 473

### Appendix E: CLI Flag Reference (Key Flags)

| Flag | Purpose |
|------|---------|
| `-p`, `--print` | Headless mode (no UI) |
| `--resume` | Resume previous session |
| `--model` | Select model |
| `--allowed-tools` | Pre-authorize tools |
| `--settings` | Custom settings file |
| `--worktree` | Run in git worktree |
| `--tmux` | Run in tmux session |
| `--bg` | Background mode |
| `--bridge` | Bridge supervisor mode |
| `--daemon` | Daemon mode |
| `--proactive` | Proactive agent mode |
| `--assistant` | Assistant/KAIROS mode |

---

## Known Gaps

1. **Daemon supervisor internals**: The concrete `daemonMain()` and worker registry implementations were not fully recovered — only the external contracts and lifecycle are documented.
2. **Background session CLI**: `claude ps|logs|attach|kill` command implementations (`src/cli/bg.js`) were not in the analyzed snapshot.
3. **Full classifier prompts**: The auto-mode classifier prompt templates (`auto_mode_system_prompt.txt`, `permissions_external.txt`, `permissions_anthropic.txt`) are referenced but their full verbatim text is in separate `.txt` files not fully extracted.
4. **Test suite coverage**: The test infrastructure, fixture system, and VCR recording mechanism are mentioned but not deeply analyzed.
5. **Git history patterns**: Commit patterns, PR workflow, and release process are not captured (would need git log analysis).
6. **Context collapse**: This experimental feature suppresses autocompact and owns its own headroom thresholds (90%/95%). The full implementation details are gated behind feature flags and not deeply traced.
7. **Computer Use MCP**: The computer use integration (`src/utils/computerUse/`) implements screenshot filtering, app enumeration, and executor patterns, but detailed analysis was not prioritized.
8. **Ultraplan**: The multi-agent planning system (`src/commands/ultraplan.tsx`, `src/utils/ultraplan/`) uses remote CCR sessions with 30-minute timeout but full orchestration logic was not deeply analyzed.
