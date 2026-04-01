# Deep Dive: Coordinator, Voice, and MoreRight Modules

## Overview

This analysis covers three specialized subsystems in Claude Code:
- **Coordinator Mode** (`coordinatorMode.ts`, 369 lines) — orchestration framework for multi-agent task execution
- **Voice Mode** (`voiceModeEnabled.ts`, 54 lines) — feature-gated voice input with auth requirements
- **MoreRight** (`useMoreRight.tsx`, 25 lines) — React hook stub for external builds (internal implementation excluded)

All three are feature-gated, have external build stubs, and integrate with the core agent execution and message handling systems.

---

## 1. Coordinator Mode (`src/coordinator/coordinatorMode.ts`)

### 1.1 What is Coordinator Mode?

Coordinator mode transforms Claude Code from a single-agent assistant into a **multi-agent orchestration system**. The coordinator itself (Claude) acts as a supervisor that:

1. **Receives user requests** directly
2. **Spawns workers** asynchronously via the Agent Tool
3. **Synthesizes findings** across multiple workers
4. **Directs implementation** with synthesized specifications
5. **Verifies results** before reporting to the user

This is distinctly different from the "swarm system" in that the coordinator is **explicitly in control**, makes synthesis decisions, and never delegates understanding to workers.

### 1.2 Core Functions

#### `isCoordinatorMode(): boolean`
- **Line 36-41**
- Returns `true` if:
  1. Feature flag `COORDINATOR_MODE` is enabled in `bun:bundle`
  2. Environment variable `CLAUDE_CODE_COORDINATOR_MODE` is truthy
- Uses `isEnvTruthy()` for flexible truthiness checking
- Two-layer gate: compile-time feature flag + runtime env var

#### `matchSessionMode(sessionMode: 'coordinator' | 'normal' | undefined): string | undefined`
- **Line 49-78**
- **Purpose**: Reconcile coordinator mode between resumed sessions and current runtime
- **Logic**:
  - Reads `sessionMode` from stored session data
  - Compares with `isCoordinatorMode()` result
  - If mismatch, flips `CLAUDE_CODE_COORDINATOR_MODE` env var to sync
  - Logs telemetry event `tengu_coordinator_mode_switched`
- **Returns**: Warning message if switched, undefined if already synchronized
- **Design insight**: Sessions persist mode state. Resuming a coordinator session in normal mode (or vice versa) requires re-sync before work continues.

#### `getCoordinatorUserContext(mcpClients: ReadonlyArray<{name: string}>, scratchpadDir?: string): {[k: string]: string}`
- **Line 80-109**
- **Purpose**: Generate context string injected into the user's initial message when coordinator mode is active
- **Behavior when coordinator is disabled**: Returns empty object `{}`
- **When enabled, builds context with**:
  - **Worker tools**: List of tools available to spawned workers
    - Simple mode: `BASH_TOOL_NAME`, `FILE_READ_TOOL_NAME`, `FILE_EDIT_TOOL_NAME`
    - Full mode: All tools from `ASYNC_AGENT_ALLOWED_TOOLS` excluding internal worker tools
  - **MCP servers**: Names of connected MCP servers workers can access
  - **Scratchpad directory**: If gated by `tengu_scratch` feature flag
- **Integration**: Context returned as single key `workerToolsContext` in user message
- **Dependency injection**: `scratchpadDir` passed from `QueryEngine.ts`

#### `getCoordinatorSystemPrompt(): string`
- **Line 111-369**
- **Purpose**: System prompt governing the entire coordinator behavior (369 lines of detailed guidance)
- **Content structure**:
  1. **Role definition** (Line 116-126): Coordinator orchestrates workers, synthesizes results, answers directly when possible
  2. **Tool inventory** (Line 128-150):
     - `Agent` tool: Spawn new worker
     - `SendMessage` tool: Continue existing worker
     - `TaskStop` tool: Halt a running worker
     - Optional PR subscription tools for GitHub integration
  3. **Worker result format** (Line 142-160): XML `<task-notification>` with status, summary, result, usage metrics
  4. **Task workflow phases** (Line 198-234):
     - Research (parallel workers)
     - Synthesis (coordinator synthesizes findings)
     - Implementation (targeted changes per spec)
     - Verification (test and prove it works)
  5. **Concurrency management** (Line 213-217):
     - Read-only tasks run in parallel freely
     - Write-heavy tasks serialized per file set
     - Verification can overlap with implementation on different areas
  6. **Worker prompt engineering** (Line 251-335):
     - Workers cannot see coordinator's conversation — every prompt must be self-contained
     - Synthesis is the coordinator's most critical job (Line 255)
     - Never delegate understanding with phrases like "based on your findings"
     - Include file paths, line numbers, exact changes needed
     - Specify purpose statements for calibration
     - Decision table for continue vs. spawn:
       - Continue if worker's context overlaps with next task
       - Spawn fresh if implementing narrow scope after broad research, or verifying different code
  7. **Failure handling** (Line 229-249): Continue same worker with context of error, or change approach
  8. **Example session** (Line 337-368): Walkthrough of parallel research, synthesis, implementation, and continuation

### 1.3 Feature Gating and Dependencies

**Feature gates**:
- `COORDINATOR_MODE` (compile-time, `bun:bundle`)
- `tengu_scratch` (scratchpad enablement, via GrowthBook)

**Service dependencies**:
- `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` — Feature flag service (GrowthBook integration)
- `logEvent()` — Analytics telemetry
- `isEnvTruthy()` — Flexible environment variable parsing

**Tool constant imports** (9 imports):
- `AGENT_TOOL_NAME`, `BASH_TOOL_NAME`, `FILE_EDIT_TOOL_NAME`, `FILE_READ_TOOL_NAME`
- `SEND_MESSAGE_TOOL_NAME`, `SYNTHETIC_OUTPUT_TOOL_NAME`, `TASK_STOP_TOOL_NAME`
- `TEAM_CREATE_TOOL_NAME`, `TEAM_DELETE_TOOL_NAME`

**Internal worker tools** (Line 29-34):
```typescript
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```
These are hidden from the public tool list exposed to workers — they're coordinator-only.

### 1.4 Integration Points

1. **QueryEngine.ts** (dependency injection):
   - Passes `scratchpadDir` to `getCoordinatorUserContext()`
   - Calls `matchSessionMode()` during session resume

2. **Anthropic CLI entry points**:
   - Message initialization: Injects `workerToolsContext` if coordinator is active
   - System prompt: Uses `getCoordinatorSystemPrompt()` to govern all behavior

3. **Session persistence**:
   - Stores mode state ('coordinator' | 'normal') in session metadata
   - Resume reconstructs mode via `matchSessionMode()`

4. **Agent Tool & SendMessage Tool**:
   - Worker spawning via `Agent` tool (spec not shown here)
   - Worker continuation via `SendMessage` tool
   - Results arrive as XML `<task-notification>` user-role messages

### 1.5 How Coordinator Differs from Swarm

| Aspect | Coordinator | Swarm (inferred) |
|--------|------------|------------------|
| **Control model** | Centralized: Coordinator makes all decisions | Decentralized: Agents self-organize or peer-coordinate |
| **Synthesis** | Explicit: Coordinator reads findings, synthesizes specs | Implicit: Each agent builds on previous results |
| **Context flow** | Coordinator → worker (one-way) | Peer-to-peer or broadcast |
| **Understanding** | Coordinator never delegates understanding to workers | Agents delegate or distribute reasoning |
| **Verification** | Coordinator directs verification worker | Verification integrated into each agent |
| **Tool access** | Workers get filtered tool set (hide internal tools) | Possibly full tool access |
| **Scaling** | High coordination overhead but clearer intent | Lower overhead, emergent behavior |

---

## 2. Voice Mode (`src/voice/voiceModeEnabled.ts`)

### 2.1 What is Voice Mode?

Voice mode enables speech-to-text input for Claude Code. Users can speak queries instead of typing. The implementation is feature-gated and auth-restricted.

### 2.2 Core Functions

#### `isVoiceGrowthBookEnabled(): boolean`
- **Line 16-23**
- **Purpose**: Kill-switch check for voice UI visibility
- **Logic**:
  1. Check compile-time feature flag `VOICE_MODE` (from `bun:bundle`)
  2. If enabled, check GrowthBook feature value `tengu_amber_quartz_disabled` (default `false`)
  3. Return `true` unless the GrowthBook flag is explicitly `true` (kill-switch)
- **Design**: "Positive ternary pattern" — negates the kill-switch to avoid inlining string literals in external builds
- **Caching**: Uses `_CACHED_MAY_BE_STALE` suffix — GrowthBook values may be stale on disk but fresh install defaults to enabled
- **Use case**: Determines if voice command should be registered in CLI or shown in UI

#### `hasVoiceAuth(): boolean`
- **Line 32-44**
- **Purpose**: Check if user has Anthropic OAuth credentials for voice access
- **Auth requirement**: Voice uses `voice_stream` endpoint on `claude.ai`, which:
  - Requires Anthropic OAuth (not API keys)
  - Not available via Bedrock, Vertex, or Foundry
- **Checks**:
  1. `isAnthropicAuthEnabled()` — Verify auth provider is Anthropic OAuth
  2. `getClaudeAIOAuthTokens()` — Retrieve memoized token (spawns `security` on macOS, ~20-50ms first call, cached thereafter)
  3. `Boolean(tokens?.accessToken)` — Verify access token exists
- **Memoization**: Token fetch is memoized, cleared on refresh (~1/hour)
- **Comment (Line 39-41)**: Without access token check, voice UI renders but fails silently on `connectVoiceStream` call

#### `isVoiceModeEnabled(): boolean`
- **Line 52-54**
- **Purpose**: Combined runtime check for voice functionality
- **Logic**: `hasVoiceAuth() && isVoiceGrowthBookEnabled()`
- **Use case**: Called from command-time paths (`/voice` command, ConfigTool, VoiceModeNotice)

### 2.3 Feature Gating and Dependencies

**Feature flags**:
- `VOICE_MODE` (compile-time, from `bun:bundle`)
- `tengu_amber_quartz_disabled` (GrowthBook kill-switch, default `false`)

**Service dependencies**:
- `getFeatureValue_CACHED_MAY_BE_STALE()` — GrowthBook integration with stale-friendly caching
- `getClaudeAIOAuthTokens()` — Memoized keychain/token store access
- `isAnthropicAuthEnabled()` — Auth provider detection

**Token management**:
- Spawns `security` process on macOS for keychain access (first call only)
- Memoization clears on token refresh cycle (~hourly)
- Acceptable for runtime checks since call cost is amortized

### 2.4 Integration Points

1. **Voice command** (`/voice` command, `voice.ts`, `voice/index.ts`):
   - Calls `isVoiceModeEnabled()` to check if voice is available
   - Proceeds to `connectVoiceStream()` if enabled

2. **ConfigTool**:
   - Checks `isVoiceGrowthBookEnabled()` for UI visibility
   - Allows user to see/enable voice if conditions met

3. **VoiceModeNotice** (component):
   - Conditional rendering based on `isVoiceModeEnabled()`
   - Alerts user if voice unavailable

4. **React render paths**:
   - For component rendering, use `useVoiceEnabled()` (memoized version)
   - For command-time checks, use `isVoiceModeEnabled()` directly

### 2.5 What We Know About Voice Implementation

Based on the code:
- **API endpoint**: `voice_stream` on `claude.ai`
- **Transport**: OAuth token-authenticated streaming
- **Auth**: Anthropic OAuth only (not API keys)
- **Audio capture**: Likely Web Audio API (browser) or native audio (CLI)
- **Speech-to-text**: Server-side (handled by `voice_stream` endpoint)
- **Mode of operation**: Not revealed in this stub, but references to "push-to-talk" or "streaming" would appear in actual voice implementation
- **Missing details**: Actual audio capture mechanism, streaming transport details, codec/format hidden in `connectVoiceStream()` (not shown)

---

## 3. MoreRight Module (`src/moreright/useMoreRight.tsx`)

### 3.1 What is MoreRight?

**MoreRight is a React hook stub for external builds.** The file explicitly states:

```
// Stub for external builds — the real hook is internal only.
```

This means:
- **Real implementation**: Internal to Anthropic, not in external/open builds
- **This file**: Placeholder for external consumers (type-safe stub)
- **Purpose of stub**: Type compatibility and safe no-op behavior

### 3.2 Function Signature

```typescript
export function useMoreRight(_args: {
  enabled: boolean
  setMessages: (action: M[] | ((prev: M[]) => M[])) => void
  inputValue: string
  setInputValue: (s: string) => void
  setToolJSX: (args: M) => void
}): {
  onBeforeQuery: (input: string, all: M[], n: number) => Promise<boolean>
  onTurnComplete: (all: M[], aborted: boolean) => Promise<void>
  render: () => null
}
```

### 3.3 Hook Parameters

| Parameter | Type | Purpose |
|-----------|------|---------|
| `enabled` | `boolean` | Feature flag to activate MoreRight functionality |
| `setMessages` | `(action: M[] \| ((prev: M[]) => M[])) => void` | Update message list (imperative or functional) |
| `inputValue` | `string` | Current user input text |
| `setInputValue` | `(s: string) => void` | Callback to update input text |
| `setToolJSX` | `(args: M) => void` | Callback to update tool rendering output |

### 3.4 Hook Return Object

| Method | Signature | Purpose |
|--------|-----------|---------|
| `onBeforeQuery` | `(input: string, all: M[], n: number) => Promise<boolean>` | Pre-query hook: validate/transform input before sending. Returns `true` to proceed, `false` to block. |
| `onTurnComplete` | `(all: M[], aborted: boolean) => Promise<void>` | Post-query hook: invoked after turn completes or aborts |
| `render` | `() => null` | React render method (stub returns `null`, real implementation may render UI) |

### 3.5 Stub Behavior

The external stub is a **safe no-op**:

```typescript
return {
  onBeforeQuery: async () => true,      // Always proceed
  onTurnComplete: async () => {},        // No-op
  render: () => null                     // No UI
}
```

### 3.6 Self-Contained Design

**Key insight from comments (Line 3-5)**:

```typescript
// Self-contained: no relative imports. Typecheck sees this file at
// scripts/external-stubs/src/moreright/ before overlay, where ../types/
// would resolve to scripts/external-stubs/src/types/ (doesn't exist).
```

This reveals:
- **Build process**: Stub directory (`scripts/external-stubs/`) is overlaid on source during external builds
- **Prevents broken imports**: No relative imports to avoid resolution failures
- **Type safety**: Generic `M = any` type used instead of importing from `../types/`
- **Isolation**: Stub is standalone, can be typechecked independently

### 3.7 What MoreRight Likely Does (Inferred)

Given the hook signature and placement:
- **Pre-query transformation**: `onBeforeQuery()` intercepts user input, possibly augmenting or validating it
- **Message mutation**: Access to `setMessages()` suggests MoreRight can inject messages or modify the conversation
- **Input manipulation**: `setInputValue()` callback allows rewriting user input before submission
- **Tool rendering**: `setToolJSX()` suggests MoreRight can customize how tools are rendered
- **Turn lifecycle**: `onTurnComplete()` runs after query completes, possibly for cleanup or state updates
- **Conditional activation**: `enabled` flag gates the feature

**Name "MoreRight"**: Unknown without internal docs. Possibilities:
- "More Right Pane" (UI area on the right)?
- "More Sophisticated Right-Hand Side" (enhanced context/sidebar)?
- Internal code name for a feature being tested?

---

## 4. Feature Gating Architecture

All three modules use **layered feature gating**:

### 4.1 Coordinator Mode

| Layer | Gate Type | Source | Default |
|-------|-----------|--------|---------|
| Compile-time | `COORDINATOR_MODE` | `bun:bundle` | Feature disabled unless bundle flag set |
| Runtime | `CLAUDE_CODE_COORDINATOR_MODE` | Environment variable | `isEnvTruthy()` check |

**Entry point**: `isCoordinatorMode()` checks both layers.

### 4.2 Voice Mode

| Layer | Gate Type | Source | Default |
|-------|-----------|--------|---------|
| Compile-time | `VOICE_MODE` | `bun:bundle` | Feature disabled unless bundle flag set |
| Kill-switch | `tengu_amber_quartz_disabled` | GrowthBook | `false` (enabled by default, unless kill-switch flipped) |
| Auth | OAuth token | Keychain (macOS) / Token store | Required for runtime activation |

**Entry points**:
- `isVoiceGrowthBookEnabled()` — UI visibility (compile-time + kill-switch)
- `hasVoiceAuth()` — Auth check (OAuth + token presence)
- `isVoiceModeEnabled()` — Full check (both gates + auth)

### 4.3 MoreRight

**No feature flag** in the stub. Enabled/disabled via the `enabled` parameter passed to the hook.

---

## 5. Integration Summary

### 5.1 Coordinator → Rest of System

**Inbound**:
- Feature flag check: `bun:bundle`
- Environment variable: `CLAUDE_CODE_COORDINATOR_MODE`
- Session state: Loaded from session metadata
- MCP clients: Passed from CLI initialization

**Outbound**:
- Injects `workerToolsContext` into user message
- Sets system prompt to coordinator-specific variant
- Initializes `Agent`, `SendMessage`, `TaskStop` tool handlers
- Logs telemetry events

**Data flow**:
```
User request
  → QueryEngine.ts
    → matchSessionMode() [reconcile mode]
    → getCoordinatorUserContext() [inject worker context]
    → getCoordinatorSystemPrompt() [set coordinator behavior]
    → Message with <task-notification> from workers
    → Coordinator processes and synthesizes
    → Response to user
```

### 5.2 Voice Mode → Rest of System

**Inbound**:
- Feature flag check: `bun:bundle`
- GrowthBook feature value: `tengu_amber_quartz_disabled`
- Anthropic auth: Token from keychain/secure store
- Feature value cache: `_CACHED_MAY_BE_STALE` pattern

**Outbound**:
- Registers `/voice` command only if enabled
- Configures voice UI visibility in settings
- Triggers `connectVoiceStream()` on command invocation
- Requires Anthropic OAuth (not API key)

**Data flow**:
```
/voice command
  → isVoiceModeEnabled() check
    → isVoiceGrowthBookEnabled() [compile-time + kill-switch]
    → hasVoiceAuth() [OAuth + token]
  → connectVoiceStream() [real implementation, not shown]
  → Audio capture and STT
  → Input processed as user text
```

### 5.3 MoreRight → Rest of System

**Integration points** (inferred):
- Called from message input/output handling
- Hooks into query lifecycle (`onBeforeQuery`, `onTurnComplete`)
- Can mutate conversation state and input
- Render output injected into UI

**Data flow**:
```
User input
  → useMoreRight.onBeforeQuery() [pre-query hook]
    → [possibly augment/validate input]
    → Proceed if returns true
  → Query execution
  → useMoreRight.onTurnComplete() [post-query hook]
  → useMoreRight.render() [inject UI if applicable]
```

---

## 6. Architectural Insights

### 6.1 Dependency Injection

**Coordinator** uses dependency injection for `scratchpadDir`:
```typescript
getCoordinatorUserContext(mcpClients, scratchpadDir)
```

This is passed from `QueryEngine.ts` (higher in dependency graph) to avoid circular dependencies.

**Comment (Line 19-24)**:
```typescript
// Checks the same gate as isScratchpadEnabled() in
// utils/permissions/filesystem.ts. Duplicated here because importing
// filesystem.ts creates a circular dependency (filesystem -> permissions
// -> ... -> coordinatorMode).
```

This reveals **circular dependency prevention** through:
- Duplication of feature gate logic
- Explicit dependency injection
- Abstraction layers

### 6.2 Stale Cache Patterns

Both **Coordinator** (via GrowthBook) and **Voice** use `_CACHED_MAY_BE_STALE` suffix:

```typescript
checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_scratch')
getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
```

**Design philosophy**:
- Cache is acceptable if stale (no real-time consistency needed)
- Fresh installs default to enabled (don't wait for feature service)
- Cache clears on significant events (token refresh, session change)

### 6.3 Stub Architecture

**MoreRight** exemplifies Anthropic's stub strategy:
- Real implementation internal only
- External builds get type-safe no-op stub
- Stub is self-contained (no imports from internal types)
- Safe to call but does nothing in external builds
- Enables type checking without exposing implementation

---

## 7. Known Limitations and Unknowns

### 7.1 Coordinator Mode

**Known**:
- Multi-worker orchestration via Agent tool
- Synthesis-driven architecture
- Parallel research, serialized implementation
- Session persistence of mode state

**Unknown**:
- Max worker count limits
- Timeout/resource constraints for workers
- How workers inherit parent context (full history?)
- Cross-worker communication mechanism
- Cost/token optimization strategies

### 7.2 Voice Mode

**Known**:
- Requires Anthropic OAuth (not API keys)
- Uses `voice_stream` endpoint on `claude.ai`
- Memoized keychain access on macOS
- GrowthBook kill-switch for emergency disable

**Unknown**:
- Audio codec/format
- Streaming vs. batch STT
- Latency targets
- Supported languages
- Echo cancellation / noise suppression
- Push-to-talk vs. always-listening
- Hardware requirements
- Concurrent voice session limits

### 7.3 MoreRight

**Known**:
- React hook for message/input mutation
- Pre and post-query hooks
- Can render UI

**Unknown**:
- Purpose ("MoreRight" name meaning)
- Implementation details (all hidden)
- Feature flag name (not exposed)
- Use cases in Anthropic
- User-facing behavior

---

## 8. Code Quality Observations

### 8.1 Coordinator

**Strengths**:
- Exceptionally detailed system prompt (369 lines)
- Clear example sessions
- Explicit guidance on synthesis
- Anti-patterns documented
- Comprehensive tool inventory

**Comments**: Lines 19-24 show thoughtful circular dependency prevention.

### 8.2 Voice

**Strengths**:
- Clear feature gate logic
- Well-documented auth requirements
- Explicit mentions of auth provider limitations
- Memoization strategy well-explained

**Design patterns**: "Positive ternary pattern" comment shows intentional external build optimization.

### 8.3 MoreRight

**Strengths**:
- Self-contained, no relative imports
- Type-safe generic `M = any` approach
- Safe no-op for external builds

**Limitations**: Stub reveals nothing about real implementation.

---

## 9. Comparison Table

| Feature | Coordinator | Voice | MoreRight |
|---------|-------------|-------|-----------|
| **Type** | Module (orchestration) | Module (feature flag) | React hook |
| **Scope** | 369 lines system + logic | 54 lines feature gating | 25 lines stub |
| **Feature gates** | 2 (compile + env) | 3 (compile + GrowthBook + auth) | 1 (param) |
| **External build** | Full implementation | Full implementation | Stub only |
| **Auth required** | None | Anthropic OAuth | N/A |
| **Complexity** | High (synthesis guidance) | Low (checks only) | Unknown |
| **Dependencies** | 9 tool names, analytics, env | 2 auth services, GrowthBook | None (stub) |
| **Stability** | Active (feature in development) | Active (production feature) | Unknown |

---

## 10. Summary

These three modules represent different layers of Claude Code's system:

1. **Coordinator**: Core orchestration paradigm for multi-agent workflows — explicit synthesis-driven control
2. **Voice**: User interaction enhancement — OAuth-gated, kill-switch-protected feature for accessibility
3. **MoreRight**: Internal experimentation framework — stub exposes hook interface but hides implementation

All three use **layered feature gating**, **dependency injection**, and **stale-cache-friendly design patterns**. Coordinator is the most complex, requiring detailed guidance. Voice is straightforward feature detection. MoreRight is intentionally opaque to external builds.
