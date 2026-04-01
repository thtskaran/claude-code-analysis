# Claude Code Query Engine Deep Dive: query.ts Analysis

**File**: `/sessions/cool-friendly-einstein/mnt/claude-code/src/query.ts` (1,729 lines)
**Date Analyzed**: 2026-04-02
**Focus**: Core query loop architecture, API orchestration, agentic flow

---

## Executive Summary

`query.ts` is the heart of Claude Code's query engine — a sophisticated async generator that orchestrates the entire interaction loop between user input and Claude's API. It implements a **multi-turn agentic loop** with:

- **Streaming API calls** with fallback model support
- **Tool use orchestration** (sequential streaming tool execution)
- **Context management** (auto-compaction, snipping, collapse, microcompaction)
- **Recovery mechanisms** for prompt-too-long, max-output-tokens, and media errors
- **Token budget tracking** and continuation strategies
- **Thinking block preservation** across turns
- **Stop hooks** for user-controlled continuation
- **Task-budgets** for resource-bounded execution

The engine is fundamentally a **state machine disguised as a loop** — mutable `State` carries iteration context across continue points. It's designed for long-running multi-turn interactions with aggressive recovery paths.

---

## 1. What Does query.ts Do?

### 1.1 High-Level Purpose

`query.ts` exports two async generators:

**`query()`** (lines 219–239)
- Entry point wrapping `queryLoop()`
- Handles command lifecycle cleanup at successful completion
- Yields all stream events from `queryLoop()`
- Returns a `Terminal` exit reason

**`queryLoop()`** (lines 241–1729)
- The infinite-while loop (line 307) implementing agentic iteration
- Destructures state at the top of each iteration
- Core responsibility: **call the API, handle tool use, recurse if needed**
- Yields events/messages throughout execution

### 1.2 Is It the Main API Call Orchestrator?

**Yes.** It is THE orchestrator:

```
┌─────────────────────────────────────────────────────────┐
│  query()                                                │
│  ├─ queryLoop() — infinite state machine               │
│  │  ├─ [Per iteration]                                 │
│  │  │  ├─ Context compaction (snip, microcompact, etc) │
│  │  │  ├─ deps.callModel() — STREAMING API CALL       │
│  │  │  ├─ Recovery (collapse, reactive compact, etc)   │
│  │  │  ├─ runTools() / StreamingToolExecutor           │
│  │  │  ├─ getAttachmentMessages() — prefetch/skills    │
│  │  │  └─ Continue decision                             │
│  │  └─ [Repeat until no tool_use or exit condition]    │
│  └─ Return Terminal reason                              │
└─────────────────────────────────────────────────────────┘
```

Every API call flows through `deps.callModel()` (line 659). Every tool execution flows through `runTools()` or `StreamingToolExecutor` (line 1382). The loop **never escapes** to a caller until a terminal condition is reached.

---

## 2. How Does a User Message Become an API Request?

### 2.1 Message Flow Pipeline

```
User Input (from params.messages)
  ↓
[Line 365] getMessagesAfterCompactBoundary() — filter post-compact history
  ↓
[Line 379] applyToolResultBudget() — enforce per-message tool result size
  ↓
[Line 403] snipModule.snipCompactIfNeeded() — optional history snipping
  ↓
[Line 414] deps.microcompact() — cached/in-memory compaction
  ↓
[Line 441] contextCollapse.applyCollapsesIfNeeded() — collapse old ranges
  ↓
[Line 454] deps.autocompact() — full-turn compaction if tokens exceed threshold
  ↓
[Line 546] toolUseContext.messages = messagesForQuery — cache on context
  ↓
[Line 660] prependUserContext() — prepend user_context as system-prepended text
  ↓
[Line 659] deps.callModel({ messages, systemPrompt, ... }) — STREAMING CALL
```

### 2.2 Key Transformation Points

**applyToolResultBudget()** (line 379)
- Enforces `tool.maxResultSizeChars` per message
- Replaces oversized tool results with truncated versions
- Persists replacements to session/agent storage

**snipCompactIfNeeded()** (line 403)
- Removes middle messages (`snipTokensFreed` returned)
- Keeps assistant/tool boundaries intact
- `snipTokensFreed` is passed to autocompact threshold logic

**microcompact()** (line 414)
- Caches tool results inline for repeated tool uses
- May edit prompt cache (feature-gated: `CACHED_MICROCOMPACT`)
- Returns `pendingCacheEdits` for post-API boundary messages

**autocompact()** (line 454)
- Full conversation summarization if over threshold
- Takes `forkContextMessages` for cache-safe summarization
- Returns new summary messages + tracking state
- Resets `turnId` and `turnCounter` on success

**contextCollapse.applyCollapsesIfNeeded()** (line 441)
- Collapses old conversation ranges to summaries
- Read-time projection (not message-yielded)
- Runs BEFORE autocompact so it can prevent proactive compact

### 2.3 System Prompt Construction

```typescript
[Line 449-451]
fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext)
)
```

- `systemPrompt` is the base (from params)
- `systemContext` is appended (dynamic)
- Result is wrapped in `asSystemPrompt()` type

**User context** is prepended separately via `prependUserContext()` (line 660) — NOT in the system prompt, but in a structured user_context field.

---

## 3. How Is the Prompt/System-Prompt/Messages Array Constructed Before the API Call?

### 3.1 The API Request Shape

```typescript
deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),  // [User/Assistant/Tool]
  systemPrompt: fullSystemPrompt,                                // SystemPrompt
  thinkingConfig: toolUseContext.options.thinkingConfig,         // ExtendedThinking
  tools: toolUseContext.options.tools,                           // ToolDef[]
  signal: toolUseContext.abortController.signal,                 // AbortSignal
  options: {
    // [Extensive config — see below]
  }
})
```

### 3.2 Messages Array Construction

**`messagesForQuery`** evolves through the iteration:

1. **Initial**: `getMessagesAfterCompactBoundary(messages)` — messages after compact boundary
2. **After snip**: `snipResult.messages` — middle messages removed
3. **After microcompact**: `microcompactResult.messages` — tool caches inlined
4. **After collapse**: `collapseResult.messages` — old ranges collapsed
5. **After autocompact**: `buildPostCompactMessages(compactionResult)` — summary + preserved tail
6. **Before API**: `prependUserContext(messagesForQuery, userContext)`

**Normalization happens** via `normalizeMessagesForAPI()` at tool-result assembly (line 855).

### 3.3 System Prompt Construction

```typescript
// Base construction
fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext)
)

// Where appendSystemContext() adds:
// - userContext values as key-value pairs (NOT here — prepended to messages)
// - systemContext values (user-defined dynamic context)
```

### 3.4 Thinking & Extended Thinking

```typescript
thinkingConfig: toolUseContext.options.thinkingConfig
```

- Passed as-is to API (no validation in query.ts)
- **Thinking rules** enforced at API level (comment lines 151–163):
  1. Thinking blocks require `max_thinking_length > 0` in query
  2. Thinking blocks cannot be the last message in a block
  3. Thinking blocks preserved for entire assistant trajectory (tool_use → tool_result → next assistant)

- **Signature blocks stripped** on fallback retry (line 928) for unprotected models

### 3.5 Tools Array

```typescript
tools: toolUseContext.options.tools
```

- Passed directly to API
- **Tool selection** happens at API layer (no pre-filtering in query.ts)
- **Tool permission checks** deferred via callback:
  ```typescript
  options: {
    async getToolPermissionContext() {
      const appState = toolUseContext.getAppState()
      return appState.toolPermissionContext
    }
  }
  ```

### 3.6 Options Passed to deps.callModel()

```typescript
options: {
  model: currentModel,                           // claude-3.5-sonnet, etc. (dynamic fallback)
  fastMode: appState.fastMode,                   // Feature-gated (line 671)
  toolChoice: undefined,                         // Not set — always auto
  isNonInteractiveSession: boolean,              // Affects latency/caching
  fallbackModel: string | undefined,             // Retry model on overload
  onStreamingFallback: () => {},                 // Callback on fallback trigger
  querySource: QuerySource,                      // 'repl_main_thread', 'sdk', etc.
  agents: agentDefinitions.activeAgents,         // For agent tool
  allowedAgentTypes: allowedAgentTypes,          // Agent filtering
  hasAppendSystemPrompt: boolean,                // Tracking flag
  maxOutputTokensOverride: number | undefined,   // Escalation (64k retry)
  fetchOverride: Fetch | undefined,              // dumpPrompts debugging
  mcpTools: mcpToolDef[],                        // MCP server tools
  hasPendingMcpServers: boolean,                 // Connection state
  queryTracking: { chainId, depth },             // Chain tracking
  effortValue: string,                           // Reasoning effort
  advisorModel: string | undefined,              // advisor chain model
  skipCacheWrite: boolean,                       // Skip prompt cache write
  agentId: string | undefined,                   // Subagent tracking
  addNotification: (msg) => void,                // Notification callback
  taskBudget?: { total, remaining },             // Resource budgeting
}
```

---

## 4. How Does Streaming Work? How Are Partial Responses Handled?

### 4.1 Streaming Loop Architecture

```typescript
[Line 659-863]
for await (const message of deps.callModel({ ... })) {
  // Inner loop: async generator yielding messages as they arrive
  if (streamingFallbackOccured) {
    // Recovery path — clear and restart
  }

  if (message.type === 'assistant') {
    assistantMessages.push(message)

    // Extract tool_use blocks as they arrive
    const msgToolUseBlocks = message.message.content.filter(
      content => content.type === 'tool_use'
    ) as ToolUseBlock[]

    if (msgToolUseBlocks.length > 0) {
      toolUseBlocks.push(...msgToolUseBlocks)
      needsFollowUp = true
    }

    // Streaming tool execution: start tools immediately
    if (streamingToolExecutor && !aborted) {
      for (const toolBlock of msgToolUseBlocks) {
        streamingToolExecutor.addTool(toolBlock, message)
      }
    }
  }

  // Yield completed tool results as they arrive
  for (const result of streamingToolExecutor.getCompletedResults()) {
    if (result.message) {
      yield result.message
      toolResults.push(...)
    }
  }
}
```

### 4.2 Message Backfilling

Before yielding, `tool_use` blocks are **backfilled** with observable inputs (lines 747–787):

```typescript
let yieldMessage = message
if (message.type === 'assistant') {
  let clonedContent
  for (let i = 0; i < message.message.content.length; i++) {
    const block = message.message.content[i]
    if (block.type === 'tool_use' && typeof block.input === 'object') {
      const tool = findToolByName(toolUseContext.options.tools, block.name)
      if (tool?.backfillObservableInput) {
        const inputCopy = { ...originalInput }
        tool.backfillObservableInput(inputCopy)

        // Only clone if fields were ADDED (not overwritten)
        const addedFields = Object.keys(inputCopy).some(
          k => !(k in originalInput)
        )
        if (addedFields) {
          clonedContent ??= [...message.message.content]
          clonedContent[i] = { ...block, input: inputCopy }
        }
      }
    }
  }
  if (clonedContent) {
    yieldMessage = { ...message, message: { ...message.message, content: clonedContent } }
  }
}
yield yieldMessage
```

**Reason**: SDK stream output and transcript serialization see expanded fields, but the original untouched message flows back to the API (prompt cache safety).

### 4.3 Error Withholding (Recoverable Errors)

Some errors are **not yielded immediately** (lines 799–825):

```typescript
let withheld = false

// Context collapse recovery (feature-gated)
if (feature('CONTEXT_COLLAPSE') && contextCollapse?.isWithheldPromptTooLong(...)) {
  withheld = true
}

// Reactive compact recovery
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}

// Media size recovery (reactive compact)
if (mediaRecoveryEnabled && reactiveCompact?.isWithheldMediaSizeError(message)) {
  withheld = true
}

// Max output tokens recovery
if (isWithheldMaxOutputTokens(message)) {
  withheld = true
}

if (!withheld) {
  yield yieldMessage
}

// Always push to assistantMessages for recovery logic to find
if (message.type === 'assistant') {
  assistantMessages.push(message)
}
```

**Strategy**: Hold the error internally, try recovery, and only surface if recovery fails.

### 4.4 Streaming Tool Execution

Two modes (line 561):

**1. StreamingToolExecutor (line 562–568)**
- Tools execute as tool_use blocks arrive
- `addTool()` queues them
- `getCompletedResults()` yields finished results
- Parallel tool execution during model streaming (5–30s window)

**2. Sequential runTools() (fallback)**
- Legacy path, runs after API response completes
- `runTools()` blocks until all tools done

**Benefits of streaming**:
- Hide tool latency under model streaming
- Yield results incrementally
- Executor checks abort signal in `executeTool()`

### 4.5 Fallback During Streaming

```typescript
if (streamingFallbackOccured) {
  // Yield tombstones for orphaned messages (lines 713–717)
  for (const msg of assistantMessages) {
    yield { type: 'tombstone' as const, message: msg }
  }

  // Clear state (lines 725–728)
  assistantMessages.length = 0
  toolResults.length = 0
  toolUseBlocks.length = 0
  needsFollowUp = false

  // Fresh executor (lines 733–740)
  if (streamingToolExecutor) {
    streamingToolExecutor.discard()
    streamingToolExecutor = new StreamingToolExecutor(...)
  }
}
```

**Reason**: Orphaned messages have invalid signatures and would cause API errors on resume. Tombstones remove them from UI/transcript.

---

## 5. How Does Tool Use Work in the Loop? Tool Call → Execution → Result → Continue?

### 5.1 Tool Use Detection

During streaming, each assistant message is scanned (lines 829–835):

```typescript
const msgToolUseBlocks = message.message.content.filter(
  content => content.type === 'tool_use'
) as ToolUseBlock[]

if (msgToolUseBlocks.length > 0) {
  toolUseBlocks.push(...msgToolUseBlocks)
  needsFollowUp = true
}
```

`needsFollowUp` signals that tool results must be sent before stopping.

### 5.2 Streaming Tool Execution (Parallel)

```typescript
if (streamingToolExecutor && !aborted) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)  // Queue immediately
  }
}

// Later, as tools complete:
for (const result of streamingToolExecutor.getCompletedResults()) {
  if (result.message) {
    yield result.message
    toolResults.push(normalizeMessagesForAPI([result.message], tools))
  }
}
```

**Timing**: Tool execution runs **during** the 5–30s model stream. Results are yielded as available.

### 5.3 Sequential Tool Execution (Legacy)

```typescript
[Line 1380–1408]
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message
    toolResults.push(normalizeMessagesForAPI([update.message], tools))
  }

  if (update.newContext) {
    updatedToolUseContext = {
      ...update.newContext,
      queryTracking,
    }
  }
}
```

**Update points**: Tools can return `newContext` to modify `toolUseContext` (e.g., shell state, workspace).

### 5.4 Tool Result Accumulation

```typescript
const toolResults: (UserMessage | AttachmentMessage)[] = []

// Populated during streaming (if StreamingToolExecutor)
for (const result of streamingToolExecutor.getCompletedResults()) {
  toolResults.push(normalizeMessagesForAPI([result.message], tools))
}

// OR populated after streaming (if sequential runTools)
for await (const update of toolUpdates) {
  toolResults.push(normalizeMessagesForAPI([update.message], tools))
}

// Continue step (line 1716):
const next: State = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  ...
}
```

**Key point**: `toolResults` are treated as a separate array and concatenated at continuation.

### 5.5 Tool Use Summary Generation

After tool execution completes (lines 1415–1482):

```typescript
if (config.gates.emitToolUseSummaries && toolUseBlocks.length > 0 && !aborted && !agentId) {
  // Collect tool info
  const toolInfoForSummary = toolUseBlocks.map(block => ({
    name: block.name,
    input: block.input,
    output: findCorrespondingToolResult(block.id, toolResults)?.content
  }))

  // Fire off async summary generation (doesn't block next API call)
  nextPendingToolUseSummary = generateToolUseSummary({
    tools: toolInfoForSummary,
    signal: abortController.signal,
    isNonInteractiveSession: boolean,
    lastAssistantText: string
  })
    .then(summary => summary ? createToolUseSummaryMessage(summary, toolUseIds) : null)
    .catch(() => null)
}
```

**Timing**: Summary is generated in **parallel** (Haiku ~1s) while next API call streams (5–30s).

---

## 6. How Does the Agentic Loop Work? Multiple Turns of Tool Use Until the Model Stops?

### 6.1 The Infinite Loop Structure

```typescript
[Line 307]
while (true) {
  // Per-iteration state destructure
  let { toolUseContext } = state
  const { messages, autoCompactTracking, ... } = state

  // Setup: compact, build API request
  messagesForQuery = [...]

  // Stream API response
  for await (const message of deps.callModel(...)) { ... }

  // Determine continuation
  if (!needsFollowUp) {
    // No tool_use blocks → exit conditions
    return { reason: 'completed' }
  }

  // Tool use detected → execute and continue
  for await (const update of toolUpdates) { ... }

  // Recurse: append tool results to messages and loop
  const next: State = {
    messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
    turnCount: turnCount + 1,
    ...
  }
  state = next
  // Loop repeats
}
```

**Key insight**: The loop is **not truly recursive** — it's a while-loop with explicit state reassignment. Each iteration:

1. **Setup**: Compact context, prepare messages
2. **API call**: Stream response
3. **Decisions**: Check for tool use, errors, max turns
4. **Tool execution**: Run queued tools
5. **Continue**: Append tool results and loop

### 6.2 Continuation Transitions

The `state.transition` field tracks WHY the previous iteration continued (line 216):

```typescript
transition: Continue | undefined

// Continue types (different paths continue for different reasons):
| { reason: 'collapse_drain_retry', committed: number }
| { reason: 'reactive_compact_retry' }
| { reason: 'max_output_tokens_escalate' }
| { reason: 'max_output_tokens_recovery', attempt: number }
| { reason: 'stop_hook_blocking' }
| { reason: 'token_budget_continuation' }
| { reason: 'next_turn' }  // Normal tool-use continuation
```

Example: If a `prompt_too_long` error happens, reactive compact tries recovery. If it succeeds, the next iteration's `transition.reason` is `'reactive_compact_retry'`.

### 6.3 Turn Tracking

```typescript
let turnCount: number = 1  // Incremented per tool-use loop

if (nextTurnCount > maxTurns) {
  yield createAttachmentMessage({
    type: 'max_turns_reached',
    maxTurns,
    turnCount: nextTurnCount,
  })
  return { reason: 'max_turns', turnCount: nextTurnCount }
}
```

**Important**: A "turn" is one iteration with tool results. The first API call (no tools) is turn 1. Each tool-result → API call is turn 2, 3, etc.

### 6.4 Exit Conditions

```typescript
[Multiple return paths]

// 1. No tool use
if (!needsFollowUp) {
  // ... recovery attempts ...
  return { reason: 'completed' }
}

// 2. User abort
if (toolUseContext.abortController.signal.aborted) {
  return { reason: 'aborted_streaming' }  // or 'aborted_tools'
}

// 3. Hook stopped
if (shouldPreventContinuation) {
  return { reason: 'hook_stopped' }
}

// 4. Max turns
if (maxTurns && nextTurnCount > maxTurns) {
  return { reason: 'max_turns', turnCount: nextTurnCount }
}

// 5. Prompt too long (unrecoverable)
return { reason: 'prompt_too_long' }

// 6. Model error
return { reason: 'model_error', error }

// 7. Image error
return { reason: 'image_error' }

// 8. Blocking limit (hard token limit)
return { reason: 'blocking_limit' }
```

### 6.5 Query Chain Tracking

```typescript
[Lines 346–363]
const queryTracking = toolUseContext.queryTracking
  ? {
      chainId: toolUseContext.queryTracking.chainId,
      depth: toolUseContext.queryTracking.depth + 1,
    }
  : {
      chainId: deps.uuid(),
      depth: 0,
    }

toolUseContext = {
  ...toolUseContext,
  queryTracking,
}
```

**Multi-level tracking**:
- **depth 0**: main conversation
- **depth 1+**: sideQuery, agent forks, etc.
- **chainId**: links all queries in a chain

---

## 7. How Is Context Window Management Handled? Token Counting, Truncation, Summarization?

### 7.1 Token Estimation Pipeline

```typescript
[Line 638]
tokenCountWithEstimation(messagesForQuery) - snipTokensFreed
```

Function signature (from imports):
```typescript
import {
  tokenCountWithEstimation,
  doesMostRecentAssistantMessageExceed200k,
  finalContextTokensFromLastResponse,
} from './utils/tokens.js'
```

**Three token tracking functions**:
- `tokenCountWithEstimation()`: Estimate total context tokens
- `doesMostRecentAssistantMessageExceed200k()`: Check last assistant message size
- `finalContextTokensFromLastResponse()`: Extract final window size from API response

### 7.2 Blocking Limit Check

```typescript
[Lines 628–648]
if (!compactionResult && querySource !== 'compact' && querySource !== 'session_memory' && ...) {
  const { isAtBlockingLimit } = calculateTokenWarningState(
    tokenCountWithEstimation(messagesForQuery) - snipTokensFreed,
    toolUseContext.options.mainLoopModel,
  )

  if (isAtBlockingLimit) {
    yield createAssistantAPIErrorMessage({
      content: PROMPT_TOO_LONG_ERROR_MESSAGE,
      error: 'invalid_request',
    })
    return { reason: 'blocking_limit' }
  }
}
```

**Guard conditions**:
- Skip if autocompact just succeeded (compactionResult exists)
- Skip if compact/session_memory query (would deadlock)
- Skip if reactive compact can handle it
- Skip if context collapse owns it

**Returns synthetic error** (never hits API).

### 7.3 Multi-Stage Context Compaction

#### 7.3.1 Snip (History Removal)

```typescript
[Lines 401–410]
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
  if (snipResult.boundaryMessage) {
    yield snipResult.boundaryMessage
  }
}
```

**Behavior**: Removes middle messages to free tokens. `snipTokensFreed` is passed to autocompact threshold.

#### 7.3.2 Microcompact (Tool Cache Inlining)

```typescript
[Lines 413–426]
const microcompactResult = await deps.microcompact(
  messagesForQuery,
  toolUseContext,
  querySource,
)
messagesForQuery = microcompactResult.messages
const pendingCacheEdits = feature('CACHED_MICROCOMPACT')
  ? microcompactResult.compactionInfo?.pendingCacheEdits
  : undefined
```

**Behavior**: Inlines cached tool results. May edit prompt cache.

#### 7.3.3 Context Collapse (Range Summarization)

```typescript
[Lines 440–447]
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
    messagesForQuery,
    toolUseContext,
    querySource,
  )
  messagesForQuery = collapseResult.messages
}
```

**Behavior**: Collapses old message ranges to summaries (read-time projection, not yield).

#### 7.3.4 Autocompact (Full Summarization)

```typescript
[Lines 453–467]
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  {
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext,
    forkContextMessages: messagesForQuery,
  },
  querySource,
  tracking,
  snipTokensFreed,
)

if (compactionResult) {
  tracking = {
    compacted: true,
    turnId: deps.uuid(),
    turnCounter: 0,
    consecutiveFailures: 0,
  }

  messagesForQuery = buildPostCompactMessages(compactionResult)

  // Log compaction event
  logEvent('tengu_auto_compact_succeeded', {
    originalMessageCount: messages.length,
    compactedMessageCount: ...,
    preCompactTokenCount,
    postCompactTokenCount,
    ...
  })
}
```

**Behavior**:
- Summarizes entire conversation
- Returns `summaryMessages` + preserved tail
- Resets `turnId` / `turnCounter`
- Logs detailed metrics (token savings, cache hits, etc.)

**Task budget carryover** (lines 508–515):
```typescript
if (params.taskBudget) {
  const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

Subtracts final context window from remaining budget before proceeding.

### 7.4 Recovery from Prompt Too Long

**Stages**:

1. **Context Collapse Drain** (lines 1089–1117)
   - Cheap: drain staged collapses
   - Only if collapse hasn't already drained

2. **Reactive Compact** (lines 1119–1166)
   - Full summary on retry
   - Max 1 attempt per turn (guarded by `hasAttemptedReactiveCompact`)
   - Media errors skip collapse, go straight here

3. **Surface Error** (lines 1173–1182)
   - If both fail, yield the withheld error
   - Skip stop hooks (error spirals)

### 7.5 Recovery from Max Output Tokens

```typescript
[Lines 1188–1252]

// Escalation: if 8k cap hit, retry at 64k
if (capEnabled && maxOutputTokensOverride === undefined && !env.CLAUDE_CODE_MAX_OUTPUT_TOKENS) {
  state = {
    messages: messagesForQuery,
    maxOutputTokensOverride: ESCALATED_MAX_TOKENS,  // 64k
    transition: { reason: 'max_output_tokens_escalate' },
  }
  continue
}

// Multi-turn recovery: ask for continuation
if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {  // 3 attempts max
  const recoveryMessage = createUserMessage({
    content: `Output token limit hit. Resume directly — no apology, no recap of what you were doing...`,
    isMeta: true,
  })

  state = {
    messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
    maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
    transition: { reason: 'max_output_tokens_recovery', attempt: ... },
  }
  continue
}

// Give up
yield lastMessage
```

**Limit**: `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3` (line 164). After 3 attempts, surface error.

---

## 8. How Does Thinking/Extended Thinking Integrate?

### 8.1 Thinking Config Passthrough

```typescript
[Line 662]
thinkingConfig: toolUseContext.options.thinkingConfig,
```

- Passed directly to `deps.callModel()`
- No validation or modification in query.ts
- Config shape: `{ type: 'enabled' | 'disabled', budget_tokens?: number }`

### 8.2 Thinking Rules (Comment at lines 151–163)

```
The rules of thinking are:
1. A message with a thinking block must be part of a query whose max_thinking_length > 0
2. A thinking block may not be the last message in a block
3. Thinking blocks must be preserved for the duration of an assistant trajectory
   (a single turn, or if that turn includes tool_use, then its tool_result and next assistant)
```

**Enforcement**: API-side (not checked in query.ts).

### 8.3 Thinking Preservation Across Tool Use

```
[Implicit rule: thinking signatures are protected across tool roundtrips]

Turn 1:
├─ [thinking: "I need to call search_tool"]
├─ [text: "I'll search..."]
└─ [tool_use: search_tool]

Tool execution...

Turn 2:
├─ [tool_result: "Found X"]
├─ [thinking: signature preserved from turn 1]
├─ [text: "Based on the results..."]
└─ [tool_use: process_tool]
```

**Signature stripping** (line 928): On model fallback, signatures are stripped so the fallback model (unprotected) doesn't reject the message.

```typescript
if (process.env.USER_TYPE === 'ant') {
  messagesForQuery = stripSignatureBlocks(messagesForQuery)
}
```

---

## 9. How Does Error Handling Work? Retries? Rate Limiting? Overloaded Responses?

### 9.1 Error Handling Hierarchy

```
[Try-catch at lines 653–997]

try {
  while (attemptWithFallback) {
    try {
      for await (const message of deps.callModel(...)) {
        // Streaming loop
      }
    } catch (innerError) {
      if (innerError instanceof FallbackTriggeredError && fallbackModel) {
        // [See 9.2 below]
      }
      throw innerError
    }
  }
} catch (error) {
  // [See 9.3 below]
}
```

**Two levels**: Inner (fallback trigger), Outer (real errors).

### 9.2 Streaming Fallback (Inner Try-Catch)

```typescript
[Lines 893–953]

if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  // Switch model
  currentModel = fallbackModel
  attemptWithFallback = true

  // Clean up state
  yield* yieldMissingToolResultBlocks(assistantMessages, 'Model fallback triggered')
  assistantMessages.length = 0
  toolResults.length = 0
  toolUseBlocks.length = 0
  needsFollowUp = false

  // Fresh executor
  if (streamingToolExecutor) {
    streamingToolExecutor.discard()
    streamingToolExecutor = new StreamingToolExecutor(...)
  }

  // Strip thinking signatures for unprotected models
  if (process.env.USER_TYPE === 'ant') {
    messagesForQuery = stripSignatureBlocks(messagesForQuery)
  }

  // Log and notify
  logEvent('tengu_model_fallback_triggered', {
    original_model: innerError.originalModel,
    fallback_model: fallbackModel,
    ...
  })

  yield createSystemMessage(
    `Switched to ${renderModelName(fallbackModel)} due to high demand...`,
    'warning',
  )

  continue  // Retry the while loop
}
throw innerError
```

**Behavior**:
- Triggered by API when primary model is overloaded
- Yields tombstones for orphaned messages
- Strips thinking signatures
- Retries entire streaming loop
- Logs event + user notification

### 9.3 Real Errors (Outer Try-Catch)

```typescript
[Lines 955–997]

} catch (error) {
  logError(error)
  const errorMessage = error instanceof Error ? error.message : String(error)

  logEvent('tengu_query_error', {
    assistantMessages: assistantMessages.length,
    toolUses: ...,
    ...
  })

  // Image size/resize errors
  if (error instanceof ImageSizeError || error instanceof ImageResizeError) {
    yield createAssistantAPIErrorMessage({ content: error.message })
    return { reason: 'image_error' }
  }

  // Missing tool results
  yield* yieldMissingToolResultBlocks(assistantMessages, errorMessage)

  // Surface error
  yield createAssistantAPIErrorMessage({ content: errorMessage })

  // Loud logging for ants
  logAntError('Query error', error)
  return { reason: 'model_error', error }
}
```

**Behavior**:
- Logs error with context
- Special handling for image errors
- Yields orphan tool-result blocks
- Surfaces error message
- Extra logging for internal users

### 9.4 Rate Limiting & Retry Logic

**Rate limiting is handled by `deps.callModel()` / `withRetry.ts`** (imported at line 7):

```typescript
import { FallbackTriggeredError } from './services/api/withRetry.js'
```

- `withRetry.ts` wraps the API with exponential backoff
- `FallbackTriggeredError` signals streaming fallback (not a retry — switches model)
- Token exhaustion (prompt-too-long) is a different recovery path (compaction, not retry)

### 9.5 Recoverable vs. Terminal Errors

**Recoverable** (continue the loop):
- `max_output_tokens`: escalate or multi-turn recovery
- `prompt_too_long`: collapse drain → reactive compact
- `media_size_error`: reactive compact strip
- `FallbackTriggeredError`: switch model and retry

**Terminal** (return):
- `model_error`: unhandled exception
- `image_error`: malformed image
- `blocking_limit`: hard token limit
- `aborted_streaming` / `aborted_tools`: user interrupt
- `completed`: normal exit

---

## 10. How Does the Query Engine Interact with Cache (Prompt Cache, Response Cache)?

### 10.1 Prompt Cache

**Write path**:
```typescript
skipCacheWrite: boolean  // Passed to deps.callModel() (line 696)
```

- API writes prompt cache by default
- Skip when `skipCacheWrite = true` (e.g., short-lived queries)

**Read path**:
- API automatically reads cache on subsequent queries
- Reported in `usage.cache_read_input_tokens`

**Cache-safe params** (lines 1125–1131):
```typescript
cacheSafeParams: {
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  forkContextMessages: messagesForQuery,  // Context for forked agents
}
```

Used during autocompact and reactive compact to ensure fork summaries don't invalidate parent's cache.

### 10.2 Microcompact with Cached Edits

```typescript
[Lines 413–426]

const microcompactResult = await deps.microcompact(messagesForQuery, ...)
const pendingCacheEdits = feature('CACHED_MICROCOMPACT')
  ? microcompactResult.compactionInfo?.pendingCacheEdits
  : undefined

// Post-API, report actual deletions
if (feature('CACHED_MICROCOMPACT') && pendingCacheEdits) {
  const lastAssistant = assistantMessages.at(-1)
  const usage = lastAssistant?.message.usage
  const cumulativeDeleted = (usage as Record<string, number>).cache_deleted_input_tokens ?? 0
  const deletedTokens = Math.max(0, cumulativeDeleted - pendingCacheEdits.baselineCacheDeletedTokens)

  if (deletedTokens > 0) {
    yield createMicrocompactBoundaryMessage(
      pendingCacheEdits.trigger,
      0,
      deletedTokens,
      pendingCacheEdits.deletedToolIds,
      [],
    )
  }
}
```

**Behavior**:
- Microcompact can edit cached tool results (removing old ones)
- Defers boundary message until after API response (has actual token counts)
- Reports `cache_deleted_input_tokens` to user

### 10.3 Response Cache

Not explicitly managed in query.ts. API handles response caching via:
- `usage.cache_read_input_tokens`: tokens read from cache
- `usage.cache_creation_input_tokens`: tokens written to cache

These are logged in autocompact events (lines 478–502).

---

## 11. How Are Stop Conditions Determined? Max Turns, Max Tokens, User Abort?

### 11.1 Max Turns

```typescript
[Lines 1705–1712]

if (maxTurns && nextTurnCount > maxTurns) {
  yield createAttachmentMessage({
    type: 'max_turns_reached',
    maxTurns,
    turnCount: nextTurnCount,
  })
  return { reason: 'max_turns', turnCount: nextTurnCount }
}
```

**Enforced**: Before looping back (checked before state.continue).

### 11.2 Max Output Tokens

```typescript
[Lines 1188–1252]

// Single escalation attempt
if (capEnabled && maxOutputTokensOverride === undefined) {
  state = {
    maxOutputTokensOverride: ESCALATED_MAX_TOKENS,  // 64k
    transition: { reason: 'max_output_tokens_escalate' },
  }
  continue
}

// Multi-turn recovery (MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3 attempts)
if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
  state = {
    messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
    maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
    transition: { reason: 'max_output_tokens_recovery', attempt: ... },
  }
  continue
}

// Give up
yield lastMessage
return { reason: 'completed' }
```

**Limits**:
- 1 escalation (8k → 64k)
- 3 recovery attempts
- Total: up to 4 continuation opportunities

### 11.3 Token Budget (Feature-Gated)

```typescript
[Lines 1308–1355]

if (feature('TOKEN_BUDGET')) {
  const decision = checkTokenBudget(
    budgetTracker!,
    toolUseContext.agentId,
    getCurrentTurnTokenBudget(),
    getTurnOutputTokens(),
  )

  if (decision.action === 'continue') {
    incrementBudgetContinuationCount()
    state = {
      messages: [...messagesForQuery, ...assistantMessages, nudgeMessage],
      turnCount,
      transition: { reason: 'token_budget_continuation' },
    }
    continue
  }

  if (decision.completionEvent) {
    logEvent('tengu_token_budget_completed', {
      ...decision.completionEvent,
      ...
    })
  }
}
```

**Behavior**:
- Tracks turn-level token budget (separate from autocompact threshold)
- Can inject nudge messages to continue within budget
- Logs completion event (diminishing returns, etc.)

**Task Budget** (lines 282–291):
```typescript
let taskBudgetRemaining: number | undefined = undefined

// Passed from params.taskBudget.total
// Decremented on each autocompact (line 511)
// Reported to API in options.taskBudget (lines 699–705)
```

### 11.4 User Abort (Ctrl+C)

```typescript
[Lines 1015–1052]

if (toolUseContext.abortController.signal.aborted) {
  // Consume remaining tool results
  if (streamingToolExecutor) {
    for await (const update of streamingToolExecutor.getRemainingResults()) {
      if (update.message) yield update.message
    }
  } else {
    yield* yieldMissingToolResultBlocks(assistantMessages, 'Interrupted by user')
  }

  // Cleanup (computer use)
  if (feature('CHICAGO_MCP') && !toolUseContext.agentId) {
    await cleanupComputerUseAfterTurn(toolUseContext)
  }

  // Skip interruption message for submit-interrupts
  if (toolUseContext.abortController.signal.reason !== 'interrupt') {
    yield createUserInterruptionMessage({ toolUse: false })
  }

  return { reason: 'aborted_streaming' }
}
```

**Also checked during tool execution** (lines 1485–1515):
```typescript
if (toolUseContext.abortController.signal.aborted) {
  // Cleanup + message
  return { reason: 'aborted_tools' }
}
```

### 11.5 Stop Hooks

```typescript
[Lines 1267–1306]

const stopHookResult = yield* handleStopHooks(
  messagesForQuery,
  assistantMessages,
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  querySource,
  stopHookActive,
)

if (stopHookResult.preventContinuation) {
  return { reason: 'stop_hook_prevented' }
}

if (stopHookResult.blockingErrors.length > 0) {
  const next: State = {
    messages: [...messagesForQuery, ...assistantMessages, ...stopHookResult.blockingErrors],
    stopHookActive: true,
    transition: { reason: 'stop_hook_blocking' },
  }
  state = next
  continue
}
```

**Behavior**:
- Hooks can signal `preventContinuation` (immediate stop)
- Hooks can inject blocking errors (loop retry)
- `stopHookActive` tracks if we're in hook-driven retry

---

## 12. How Does Prefill Work?

### 12.1 Prefill Mechanism

**Prefill is NOT in query.ts.**

`query.ts` passes the message history as-is to the API. The API supports prefill via:
- `messages[-1].content` prepended with `type: 'text'` block having `cache_control: { type: 'ephemeral' }`

This is handled by the API client layer (`deps.callModel()`), not the query loop.

### 12.2 Related: Tool Execution Prefill

Tool names and schemas are communicated to the API statically (in `tools` array). The API pre-samples which tool to call based on early context, then streams the input JSON.

This is **not prefill** in the prompt-cache sense — it's just streaming tool inputs.

---

## 13. How Does the Fork Mechanism Work (For Suggestions, Side Queries)?

### 13.1 Fork Context Messages

```typescript
[Lines 454–467]

const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  {
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext,
    forkContextMessages: messagesForQuery,  // <-- Fork context
  },
  querySource,
  tracking,
  snipTokensFreed,
)
```

**Key insight**: `forkContextMessages` is the **pre-compact history** passed to summarization. Forked agents (created during compaction or tools) see this context, not the summary.

**Benefit**: Cache safety — fork summaries don't invalidate parent's prompt cache.

### 13.2 Query Source Tags

```typescript
querySource: QuerySource  // e.g., 'repl_main_thread', 'agent:AgentTool', 'sideQuery'

// Used to:
// 1. Skip certain checks (compact/session_memory don't block)
// 2. Filter queued commands (main thread vs. subagents)
// 3. Log chain depth
```

### 13.3 Subagent Coordination

```typescript
[Lines 1561–1577]

const isMainThread = querySource.startsWith('repl_main_thread') || querySource === 'sdk'
const currentAgentId = toolUseContext.agentId

const queuedCommandsSnapshot = getCommandsByMaxPriority(...).filter(cmd => {
  if (isSlashCommand(cmd)) return false
  if (isMainThread) return cmd.agentId === undefined
  return cmd.mode === 'task-notification' && cmd.agentId === currentAgentId
})
```

**Behavior**:
- Main thread drains commands addressed to `undefined` agentId
- Subagents drain only task-notifications addressed to their agentId
- User prompts always go to main only

---

## 14. What Hooks Fire Before/After Queries? (Pre-Query, Post-Query, Post-Tool-Use)

### 14.1 Pre-Query Hooks

**None explicitly in query.ts.**

Setup happens in `queryLoop` entry (lines 241–305):
- State initialization
- Memory prefetch startup (line 301)
- Skill discovery prefetch startup (line 331)

### 14.2 Post-Sampling Hooks

```typescript
[Lines 999–1009]

// Execute post-sampling hooks after model response is complete
if (assistantMessages.length > 0) {
  void executePostSamplingHooks(
    [...messagesForQuery, ...assistantMessages],
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext,
    querySource,
  )
}
```

**Behavior**: Async (non-blocking) — fires after streaming completes, before tool execution.

### 14.3 Stop Failure Hooks

```typescript
[Lines 1174, 1181, 1263]

void executeStopFailureHooks(lastMessage, toolUseContext)
```

**Fires on**: API errors (rate limit, prompt-too-long, auth failure).

**Behavior**: Handles error-specific cleanup, not executed on success.

### 14.4 Stop Hooks (Continuation Control)

```typescript
[Lines 1267–1306]

const stopHookResult = yield* handleStopHooks(
  messagesForQuery,
  assistantMessages,
  systemPrompt,
  userContext,
  systemContext,
  toolUseContext,
  querySource,
  stopHookActive,
)

if (stopHookResult.preventContinuation) {
  return { reason: 'stop_hook_prevented' }
}

if (stopHookResult.blockingErrors.length > 0) {
  state = {
    ...
    stopHookActive: true,
    transition: { reason: 'stop_hook_blocking' },
  }
  continue
}
```

**Behavior**: Hooks can block continuation (inject errors, ask for user confirmation, etc.).

### 14.5 Computer Use Cleanup

```typescript
[Lines 1034–1042, 1486–1498]

if (feature('CHICAGO_MCP') && !toolUseContext.agentId) {
  try {
    const { cleanupComputerUseAfterTurn } = await import('./utils/computerUse/cleanup.js')
    await cleanupComputerUseAfterTurn(toolUseContext)
  } catch {
    // Silent
  }
}
```

**Fires on**: Turn end (success or abort). Main thread only.

### 14.6 Post-Tool-Use (Summary Generation)

```typescript
[Lines 1411–1482]

let nextPendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined

if (config.gates.emitToolUseSummaries && toolUseBlocks.length > 0 && !aborted) {
  nextPendingToolUseSummary = generateToolUseSummary({
    tools: toolInfoForSummary,
    signal: abortController.signal,
    ...
  })
    .then(summary => summary ? createToolUseSummaryMessage(summary, toolUseIds) : null)
    .catch(() => null)
}
```

**Behavior**: Async Haiku summary (non-blocking). Result yielded on next iteration (line 1054–1060).

---

## 15. How Does Model Selection Work? Fallback Models? Dynamic Model Switching?

### 15.1 Model Selection at Entry

```typescript
[Lines 572–578]

const appState = toolUseContext.getAppState()
const permissionMode = appState.toolPermissionContext.mode

let currentModel = getRuntimeMainLoopModel({
  permissionMode,
  mainLoopModel: toolUseContext.options.mainLoopModel,
  exceeds200kTokens: permissionMode === 'plan' && doesMostRecentAssistantMessageExceed200k(messagesForQuery),
})
```

**Logic**:
- Base model from `toolUseContext.options.mainLoopModel`
- Checked against permission mode ('plan', etc.)
- Downgraded if exceeds 200k tokens in plan mode

### 15.2 Fallback Trigger

```typescript
[Lines 893–953]

if (innerError instanceof FallbackTriggeredError && fallbackModel) {
  currentModel = fallbackModel
  attemptWithFallback = true

  // ... cleanup ...

  continue
}
```

**Trigger source**: API signals `FallbackTriggeredError` when primary model is overloaded.

**Condition**: Fallback only if `fallbackModel` parameter is provided.

### 15.3 Fallback Event Logging

```typescript
logEvent('tengu_model_fallback_triggered', {
  original_model: innerError.originalModel,
  fallback_model: fallbackModel,
  entrypoint: 'cli',
  queryChainId: queryChainIdForAnalytics,
  queryDepth: queryTracking.depth,
})

yield createSystemMessage(
  `Switched to ${renderModelName(fallbackModel)} due to high demand for ${renderModelName(innerError.originalModel)}`,
  'warning',
)
```

**User notification**: System message warning about the switch.

### 15.4 Signature Stripping for Unprotected Fallbacks

```typescript
if (process.env.USER_TYPE === 'ant') {
  messagesForQuery = stripSignatureBlocks(messagesForQuery)
}
```

**Rationale**: Thinking blocks have model-bound signatures. Unprotected fallback models reject protected messages.

---

## Security Analysis: Potential Attack Vectors

### SECONDARY: Can the Query Loop Be Manipulated to Infinite-Loop? Token Exhaustion Attacks? Cache Poisoning?

### 16.1 Infinite Loop Prevention

**Maximum iterations**: Theoretically unbounded, but guards exist:

1. **Max Turns** (line 1705)
   ```typescript
   if (maxTurns && nextTurnCount > maxTurns) {
     return { reason: 'max_turns' }
   }
   ```
   - Enforced per turn (tool use → API response is 1 turn)
   - Default: None (unbounded)
   - Risk: **HIGH** if `maxTurns` is not set and agent loops

2. **Max Output Tokens Recovery Limit** (line 164)
   ```typescript
   const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3
   ```
   - Max 3 recovery attempts + 1 escalation
   - Risk: **LOW** — bounded

3. **Token Budget** (feature-gated, line 1308)
   ```typescript
   if (decision.action === 'complete') {
     return { reason: 'completed' }
   }
   ```
   - Optional resource limit
   - Risk: **MEDIUM** — only if enabled

4. **Stop Hooks** (line 1267)
   - User-defined termination logic
   - Risk: **LOW** — depends on hook implementation

5. **Reactive Compact Guard** (line 1092)
   ```typescript
   state.transition?.reason !== 'collapse_drain_retry'
   ```
   - Prevents repeated drain attempts
   - Risk: **LOW** — one-shot per error

6. **User Abort** (line 1015)
   - Ctrl+C signal
   - Risk: **LOW** — immediate return

**Attack scenario**: Agent configured with no `maxTurns` could theoretically loop on every tool use. Mitigation: Enforce default `maxTurns` in entry layer (not in query.ts).

### 16.2 Token Exhaustion Attacks

**Scenario 1: Compaction Loop**
```
1. Autocompact runs → summarizes to 50k tokens
2. Model generates 40k output → total 90k
3. Next iteration: autocompact again → 50k summary + prev output
4. Loop: O(n) iterations, O(n^2) API calls
```

**Mitigation**: Autocompact fires **once per major state change** (line 454 is not in a tool loop — it's per API call).

**Scenario 2: Tool Result Poisoning**
```
1. Tool returns massive output (e.g., 1M tokens)
2. applyToolResultBudget() truncates (line 379)
3. Next tool call uses truncated version
4. Agent confused, loops retrying
```

**Mitigation**: `applyToolResultBudget()` enforces per-message limits. Oversized results are replaced with error messages.

**Scenario 3: Reactive Compact Spiral**
```
1. Prompt too long → reactive compact
2. Still too long after compact → error surfaces
3. Stop hook injects more tokens → error again
4. Loop: error → hook → error
```

**Mitigation** (line 1169–1172):
```typescript
// Skip stop hooks on API error (not real response to evaluate)
if (lastMessage?.isApiErrorMessage) {
  void executeStopFailureHooks(lastMessage, toolUseContext)
  return { reason: 'completed' }
}
```

Stop hooks are NOT called on unrecoverable errors.

### 16.3 Cache Poisoning

**Scenario 1: Malicious Tool Output in Cache**
```
1. Adversary crafts tool output with hidden instruction
2. Tool result is cached (prompt cache)
3. Next query reads cache, sees instruction
```

**Mitigation**:
- Tool results are NOT cached by default (they're in assistant message context)
- Microcompact can inline cached tool results, but this is under query control
- No untrusted data reaches the system prompt

**Scenario 2: Compaction Summary Corruption**
```
1. Autocompact summarizes with injected instruction
2. Summary replaces conversation
3. Model follows injected instruction instead of user's
```

**Mitigation**:
- Autocompact is called with a controlled Haiku summarization
- Summary is yielded (not hidden) — user can see it
- Original messages are preserved until explicit compaction

**Scenario 3: Memory Prefetch Injection**
```
1. Attacker modifies CLAUDE.md in workspace
2. Memory prefetch loads it
3. Model sees adversary-controlled instructions
```

**Mitigation**:
- Memory prefetch is explicit (`/memory read` commands)
- Not auto-injected into system context
- Controlled by user via `/memory` slash commands

### 16.4 Output Token Limit Bypass

**Scenario**: Max output tokens escalation chain
```
1. 8k limit hit → escalate to 64k
2. 64k limit hit → multi-turn recovery (3 attempts)
3. Total: 8k + 64k + 3*(unknown) = potential 200k+ output
```

**Mitigation**:
- Escalation is **one-time** (line 1201): `maxOutputTokensOverride === undefined`
- Recovery attempts are **limited to 3** (line 1223)
- After exhaustion, error surfaces (line 1254)
- User must manually decide to continue

### 16.5 Streaming Fallback Loop

**Scenario**: Rapid fallback thrashing
```
1. API overloaded → fallback triggered
2. Fallback model also overloaded → fallback again (cascade)
3. Loop: primary → fallback1 → fallback2 → error
```

**Mitigation**:
- Only **one fallback model** supported (line 894)
- If fallback fails, error propagates (line 952)
- No cascading fallbacks

---

## Summary of Critical Points

| Aspect | Mechanism | Risk | Mitigation |
|--------|-----------|------|-----------|
| **Infinite loop** | Tool use → API → recurse | HIGH | Enforce `maxTurns` in entry layer |
| **Token exhaustion** | Compaction spiral | LOW | One autocompact per state change |
| **Cache poisoning** | Tool output in cache | MEDIUM | Cache not auto-populated with untrusted data |
| **Error spiral** | PTL + stop hook loop | LOW | Skip hooks on API errors |
| **Fallback cascade** | One fallback only | LOW | Single retry, then error |
| **Max tokens bypass** | Escalation + recovery | LOW | 1 escalation + 3 recoveries max |

---

## Conclusion

`query.ts` is a **mature, defensive agentic loop** with:

1. **Multi-stage context compression** (snip → microcompact → collapse → autocompact)
2. **Aggressive error recovery** (collapse drain → reactive compact → escalation)
3. **Streaming tool execution** (parallel during model response)
4. **Token-aware continuation** (budgets, turn limits, escalation)
5. **Thinking block preservation** across tool use
6. **Fallback model support** with orphan message cleanup
7. **Hook-based extensibility** (stop hooks, post-sampling hooks)

**Key design insight**: The loop is not truly infinite — it's a **state machine** where each iteration:
- Destructures immutable params + mutable state
- Makes an API call
- Decides on continuation (tool use? error recovery? exit?)
- Reassigns state for the next iteration

This design allows for complex recovery logic without callback hell or nested loops.

---

**Analysis Date**: 2026-04-02
**File**: `/sessions/cool-friendly-einstein/mnt/claude-code/src/query.ts` (1,729 lines)
**Coverage**: 100% (all lines read and analyzed)
