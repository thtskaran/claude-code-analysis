# Claude Code v2.1.88 Context Compaction Subsystem: Deep-Dive Analysis

## Executive Summary

Claude Code's context compaction system is a multi-layered, state-machine-driven architecture designed to manage context overflow in long-running conversations. The system employs six distinct compaction strategies, each optimized for specific scenarios:

1. **Time-based Microcompaction** — Content-clears old tool results when server cache expires (60min threshold)
2. **Cached Microcompaction** — Uses native API `cache_edits` to surgically remove specific tool results without invalidating cached prefix
3. **Reactive Compaction** — Triggered by API `prompt_too_long` errors, truncates messages from head
4. **Session Memory Compaction** — (Experimental) Uses pre-extracted session summaries instead of full message summarization
5. **Full Summarization** — Traditional compaction: forked agent generates 9-section narrative summary
6. **Partial/Selective Compaction** — User-initiated compaction of specific message ranges

The architecture prioritizes **cache safety** throughout, ensuring compaction respects Anthropic's prompt caching boundaries and circuit breaker logic prevents cascade failures when context is irrecoverably over-limit.

---

## Part 1: Compaction Pipeline Overview

### 6-Stage Flow from Token Monitoring to Warning Reset

The complete compaction lifecycle follows this deterministic flow:

```
Stage 1: Token Monitoring & Decision
    ├─ shouldAutoCompact() checks thresholds
    ├─ calculateTokenWarningState() computes warning/error states
    └─ Returns: boolean indicating if compact needed

Stage 2: Microcompaction (Lightweight)
    ├─ evaluateTimeBasedTrigger() → maybeTimeBasedMicrocompact()
    │   └─ Clears old tool results if gap > 60min
    ├─ cachedMicrocompactPath() → getAPIContextManagement()
    │   └─ Queues cache_edits for API (no local changes)
    └─ Returns: messages (possibly with cleared tool_results or pending edits)

Stage 3: Session Memory Compaction (Optional Experimental)
    ├─ trySessionMemoryCompaction()
    │   ├─ Checks shouldUseSessionMemoryCompaction() gates
    │   ├─ Loads pre-extracted session memory
    │   ├─ calculateMessagesToKeepIndex() determines preserved segment
    │   └─ Returns: CompactionResult with preserved messages + summary
    └─ If succeeds: skip full summarization, proceed to Stage 5

Stage 4: Full Summarization (Traditional)
    ├─ compactConversation() or partialCompactConversation()
    ├─ streamCompactSummary() calls forked agent
    │   ├─ Tries cache-sharing via runForkedAgent()
    │   └─ Falls back to streaming path (queryModelWithStreaming)
    ├─ truncateHeadForPTLRetry() loop handles prompt_too_long on compact request itself
    ├─ stripImagesFromMessages() + stripReinjectedAttachments() data prep
    └─ Returns: 9-section summary text

Stage 5: Post-Compaction State Restoration
    ├─ createPostCompactFileAttachments() restores recently-read files
    ├─ createSkillAttachmentIfNeeded() re-injects invoked skill content
    ├─ createPlanAttachmentIfNeeded() + createPlanModeAttachmentIfNeeded()
    ├─ getDeferredToolsDeltaAttachment() + getAgentListingDeltaAttachment()
    └─ processSessionStartHooks() restores CLAUDE.md + memory context

Stage 6: Cache Safety & Cleanup
    ├─ notifyCompaction() resets cache baseline (prompt_cache_break_detection)
    ├─ markPostCompaction() sets global post-compact flag
    ├─ reAppendSessionMetadata() re-appends title/tags to end (16KB tail window)
    ├─ runPostCompactCleanup() clears module-level tracking caches
    │   └─ resetMicrocompactState() + clearSystemPromptSections() + clearClassifierApprovals()
    ├─ clearCompactWarningSuppression() → suppressCompactWarning() toggle
    └─ Event logging: tengu_compact with full telemetry
```

### Threshold Calculations: The Token Budget Model

All thresholds are derived from a single **effective context window**:

```typescript
// Base calculation (autoCompact.ts)
effectiveContextWindow = contextWindowForModel - MAX_OUTPUT_TOKENS_FOR_SUMMARY
                       = contextWindowForModel - min(modelMaxTokens, 20_000)

// Threshold layering
autoCompactThreshold    = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS (13K)
warningThreshold        = effectiveContextWindow - WARNING_THRESHOLD_BUFFER_TOKENS (20K)
errorThreshold          = effectiveContextWindow - ERROR_THRESHOLD_BUFFER_TOKENS (20K)
blockingLimit           = effectiveContextWindow - MANUAL_COMPACT_BUFFER_TOKENS (3K)
```

**Example for 200K window model:**
- Output reserved: 20K
- Effective window: 180K
- Auto-compact at: 167K (13K buffer)
- Warning at: 160K
- Error at: 160K
- Manual compact blocks at: 177K

### Circuit Breaker Logic

**File:** `autoCompact.ts:70`

```typescript
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

When autocompact fails:
1. First failure: `consecutiveFailures = 1`, retry next turn
2. Second failure: `consecutiveFailures = 2`, retry next turn
3. Third failure: `consecutiveFailures = 3`, **skip all future compaction attempts this session**

This prevents pathological loops where context is irrecoverably over-limit (e.g., single message > effective window) and compaction will always fail.

---

## Part 2: Full Summarization Algorithm

### The Forked Agent Architecture

**File:** `compact.ts:1136–1396` (streamCompactSummary function)

Full summarization uses a **forked agent** pattern to reuse the main conversation's prompt cache:

#### Path 1: Cache-Sharing Forked Agent (Primary)
```typescript
// Feature: tengu_compact_cache_prefix (default: true for 3P, false-able for ants)
// When enabled:
runForkedAgent({
  promptMessages: [summaryRequest],
  cacheSafeParams,                    // Inherits main thread's system/tools/model
  canUseTool: createCompactCanUseTool(),
  maxTurns: 1,                         // Single response only
  skipCacheWrite: true,                // Don't persist fork's cache
  overrides: { abortController }
})
```

**Key insight:** The fork sends the EXACT SAME system/tools/model as the main thread, so the API cache key matches. The prefix (system prompt + shared messages) cache-hits, reducing cost by ~76% per compact call.

**Fallback logic:** If fork fails (no text response, error), attempts streaming path.

#### Path 2: Streaming Path (Fallback)
```typescript
queryModelWithStreaming({
  messages: normalizeMessagesForAPI(stripImagesFromMessages(...)),
  systemPrompt: asSystemPrompt(['You are a helpful AI assistant...']),
  thinkingConfig: { type: 'disabled' },
  tools: useToolSearch ? [FileReadTool, ToolSearchTool, ...mcpTools] : [FileReadTool],
  maxOutputTokensOverride: min(COMPACT_MAX_OUTPUT_TOKENS, modelMax)
  // COMPACT_MAX_OUTPUT_TOKENS = 20_000
})
```

**Feature:** `tengu_compact_streaming_retry` (default: false)
- When enabled: retry streaming up to 2 times with exponential backoff
- Preserves cache-sharing attempt — only falls back if it fails

### Message Normalization & Stripping

**Key preprocessing steps before summarization:**

1. **Image Stripping** (`compact.ts:145–200`)
   ```typescript
   stripImagesFromMessages(messages) // Replaces image/document blocks with [image]/[document]
   ```
   Rationale: Images waste tokens in compaction summary (user already described intent) and can themselves hit prompt-too-long on the compact request.

2. **Re-injected Attachment Stripping** (`compact.ts:211–223`)
   ```typescript
   stripReinjectedAttachments(messages)
   // Removes: skill_discovery, skill_listing (re-injected post-compact anyway)
   ```

3. **Only Keep Post-Boundary Messages** (`compact.ts:1296`)
   ```typescript
   getMessagesAfterCompactBoundary(messages)
   // Discards prior summary + boundary (would be redundant to summarize a summary)
   ```

### Prompt-Too-Long Retry Loop (CC-1180)

**File:** `compact.ts:450–491`

When the compact request ITSELF hits `prompt_too_long`:

```typescript
for (;;) {
  summaryResponse = await streamCompactSummary(...)
  summary = getAssistantMessageText(summaryResponse)

  if (!summary?.startsWith(PROMPT_TOO_LONG_ERROR_MESSAGE)) break

  ptlAttempts++
  if (ptlAttempts > MAX_PTL_RETRIES) {  // 3 retries
    throw new Error(ERROR_MESSAGE_PROMPT_TOO_LONG)
  }

  // Truncate oldest API-round groups until tokenGap is covered
  truncated = truncateHeadForPTLRetry(messagesToSummarize, summaryResponse)
  if (!truncated) throw

  messagesToSummarize = truncated
  retryCacheSafeParams.forkContextMessages = truncated
}
```

**truncateHeadForPTLRetry logic:**
- Groups messages by `groupMessagesByApiRound()` (API response boundaries)
- Calculates token gap from error response
- Drops oldest groups until gap covered, keeping at least 1 group
- Preserves API-safe split points (tool_use/tool_result pairs stay together)
- Inserts synthetic `[earlier conversation truncated for compaction retry]` marker when dropping group 0

### The Complete Summarization Prompt (9-Section Template)

**File:** `prompt.ts:61–143`

Structure with examples:

```
[NO_TOOLS_PREAMBLE]
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

[BASE_COMPACT_PROMPT]
Your task is to create a detailed summary of the conversation so far,
paying close attention to the user's explicit requests and your previous actions.

[DETAILED_ANALYSIS_INSTRUCTION_BASE]
Before providing your final summary, wrap your analysis in <analysis> tags...

Your summary should include the following sections:

1. Primary Request and Intent
   Capture all of the user's explicit requests and intents in detail

2. Key Technical Concepts
   List all important technical concepts, technologies, and frameworks discussed

3. Files and Code Sections
   Enumerate specific files and code sections examined, modified, or created.
   Pay special attention to full code snippets and why edits are important.

4. Errors and fixes
   List all errors that you ran into, and how you fixed them.
   Pay special attention to specific user feedback.

5. Problem Solving
   Document problems solved and any ongoing troubleshooting efforts.

6. All user messages
   List ALL user messages that are not tool results.
   These are critical for understanding users' feedback and changing intent.

7. Pending Tasks
   Outline any pending tasks that you have explicitly been asked to work on.

8. Current Work
   Describe in detail precisely what was being worked on immediately before
   this summary request, with file names and code snippets.

9. Optional Next Step
   List the next step related to the most recent work.
   IMPORTANT: ensure that this step is DIRECTLY in line with the user's
   most recent explicit requests.
   Include direct quotes from the most recent conversation.

[Example format with <analysis> and <summary> wrappers...]

[Additional Instructions if provided...]

[NO_TOOLS_TRAILER]
REMINDER: Do NOT call any tools. Respond with plain text only —
an <analysis> block followed by a <summary> block.
```

**Analysis section removal:** `formatCompactSummary(summary)` strips `<analysis>...</analysis>` tags post-generation. The analysis block is a drafting scratchpad that improves quality but adds no informational value once the summary is complete.

### Post-Compaction Skill Re-injection (5-File Budget)

**File:** `compact.ts:1494–1534`

Invoked skills are re-injected to preserve their content across compaction:

```typescript
// Constants
export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5          // Max 5 files
export const POST_COMPACT_TOKEN_BUDGET = 50_000             // 50K total
export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000       // 5K each
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000      // 5K per skill (truncated)
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000      // 25K for skills
```

**Algorithm:**
1. Sort invoked skills by `invokedAt` descending (most-recent first)
2. Truncate each skill to 5K tokens max (keeps head where setup instructions live)
3. Filter by token budget (25K total for all skills, 50K for all files combined)
4. Attach as `invoked_skills` message type

**Rationale:** Skills can be large (e.g., verify=18.7KB). Per-skill truncation beats dropping entire skills, and keeping the top of the file preserves critical instructions.

### SystemCompactBoundaryMessage Creation

**File:** `compact.ts:598–611`

Creates the marker that separates pre-compact (summarized) from post-compact context:

```typescript
const boundaryMarker = createCompactBoundaryMessage(
  isAutoCompact ? 'auto' : 'manual',
  preCompactTokenCount ?? 0,
  messages.at(-1)?.uuid                           // logicalParentUuid
)

// Optionally add discovered tools (tools loaded in pre-compact context)
if (preCompactDiscovered.size > 0) {
  boundaryMarker.compactMetadata.preCompactDiscoveredTools =
    [...preCompactDiscovered].sort()
}

// For partial compact with preserved segment (messagesToKeep):
boundary = annotateBoundaryWithPreservedSegment(
  boundary,
  anchorUuid,           // Last summary message or boundary itself
  messagesToKeep        // Messages kept between summary and any later content
)
```

---

## Part 3: Microcompaction Strategies

### 3A: Time-Based Microcompaction (Expiration Detection)

**File:** `microCompact.ts:412–530`

Triggers when the gap since the last main-loop assistant message **exceeds the server cache TTL** (default 60 minutes):

```typescript
export function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource?: QuerySource
): { gapMinutes: number; config: TimeBasedMCConfig } | null {
  const config = getTimeBasedMCConfig()        // GrowthBook: tengu_slate_heron

  if (!config.enabled || !querySource || !isMainThreadSource(querySource)) {
    return null
  }

  const lastAssistant = messages.findLast(m => m.type === 'assistant')
  if (!lastAssistant) return null

  const gapMinutes = (Date.now() - new Date(lastAssistant.timestamp).getTime()) / 60_000

  if (!Number.isFinite(gapMinutes) || gapMinutes < config.gapThresholdMinutes) {
    return null
  }

  return { gapMinutes, config }
}
```

**Configuration (GrowthBook):**
```typescript
type TimeBasedMCConfig = {
  enabled: boolean              // Master switch
  gapThresholdMinutes: number   // Default: 60 (safe choice: 1h cache TTL guaranteed expired)
  keepRecent: number            // Default: 5 (keep most-recent 5 tool results)
}
```

**Content Clearing Action:**
```typescript
// Replaces tool result content with marker
const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'

// Applies to compactable tools only:
COMPACTABLE_TOOLS = {
  File{Read,Edit,Write}, Bash/Shell, Grep, Glob, WebSearch, WebFetch
}

// Logic:
keepSet = new Set(compactableIds.slice(-keepRecent))    // Last N tools
clearSet = compactableIds.filter(id => !keepSet.has(id))

// Mutates messages directly (cache is cold anyway):
for each tool_result in clearSet:
  tool_result.content = TIME_BASED_MC_CLEARED_MESSAGE
```

**Token Savings Calculation:**
```typescript
tokensSaved = sum(calculateToolResultTokens(block) for block in clearSet)
```

**Side Effects:**
- Resets `cachedMCState` (API cache is invalidated, so prior cache_edits plans are moot)
- Calls `notifyCacheDeletion()` for cache break detection
- Logs `tengu_time_based_microcompact` event

---

### 3B: Cached Microcompaction (Native API cache_edits)

**File:** `microCompact.ts:305–399`

Uses Anthropic's native `cache_edits` parameter to surgically remove tool results **without invalidating the cached prefix**:

```typescript
async function cachedMicrocompactPath(
  messages: Message[],
  querySource: QuerySource | undefined
): Promise<MicrocompactResult> {
  const mod = await getCachedMCModule()
  const config = mod.getCachedMCConfig()  // GrowthBook: count-based thresholds

  // Register all tool results from current messages
  const compactableToolIds = new Set(collectCompactableToolIds(messages))
  for (const message of messages) {
    if (message.type === 'user' && Array.isArray(message.message.content)) {
      const groupIds: string[] = []
      for (const block of message.message.content) {
        if (block.type === 'tool_result' && compactableToolIds.has(block.tool_use_id)) {
          mod.registerToolResult(state, block.tool_use_id)
          groupIds.push(block.tool_use_id)
        }
      }
      mod.registerToolMessage(state, groupIds)
    }
  }

  // Determine which tools to delete based on count thresholds
  const toolsToDelete = mod.getToolResultsToDelete(state)

  if (toolsToDelete.length > 0) {
    // Create cache_edits block for API layer
    const cacheEdits = mod.createCacheEditsBlock(state, toolsToDelete)
    pendingCacheEdits = cacheEdits

    // Notify cache break detection
    notifyCacheDeletion(querySource ?? 'repl_main_thread')

    // Return messages UNCHANGED — cache_reference and cache_edits added at API layer
    return {
      messages,
      compactionInfo: {
        pendingCacheEdits: {
          trigger: 'auto',
          deletedToolIds: toolsToDelete,
          baselineCacheDeletedTokens: (lastAsst?.message.usage?.cache_deleted_input_tokens ?? 0)
        }
      }
    }
  }

  return { messages }
}
```

**Key Architectural Point:** Local messages are **NOT modified**. Instead, `pendingCacheEdits` is queued and consumed at the API call site (apiMicrocompact.ts integration), where cache_edits are added to the API request as a sibling to cache_control.

**GrowthBook Integration:**
```typescript
// Config from cachedMicrocompact.js (not visible here, assumed to include):
{
  triggerThreshold: number   // e.g., 30 tools accumulated
  keepRecent: number         // e.g., keep last 10
}
```

---

### 3C: Token Estimation Per Message Type

**File:** `microCompact.ts:164–205`

Used for threshold decisions and budget calculations:

```typescript
export function estimateMessageTokens(messages: Message[]): number {
  let totalTokens = 0

  for (const message of messages) {
    if (message.type !== 'user' && message.type !== 'assistant') continue
    if (!Array.isArray(message.message.content)) continue

    for (const block of message.message.content) {
      if (block.type === 'text') {
        totalTokens += roughTokenCountEstimation(block.text)
      } else if (block.type === 'tool_result') {
        // Detailed calculation:
        if (typeof block.content === 'string') {
          totalTokens += roughTokenCountEstimation(block.content)
        } else if (Array.isArray(block.content)) {
          // Array of TextBlockParam | ImageBlockParam | DocumentBlockParam
          for (const item of block.content) {
            if (item.type === 'text') {
              totalTokens += roughTokenCountEstimation(item.text)
            } else if (item.type === 'image' || item.type === 'document') {
              // Conservative estimate: 2000 tokens regardless of format
              totalTokens += IMAGE_MAX_TOKEN_SIZE
            }
          }
        }
      } else if (block.type === 'image' || block.type === 'document') {
        totalTokens += IMAGE_MAX_TOKEN_SIZE
      } else if (block.type === 'thinking') {
        // Only count the thinking text, not JSON wrapper
        totalTokens += roughTokenCountEstimation(block.thinking)
      } else if (block.type === 'redacted_thinking') {
        totalTokens += roughTokenCountEstimation(block.data)
      } else if (block.type === 'tool_use') {
        // Count name + input, not JSON wrapper or id
        totalTokens += roughTokenCountEstimation(
          block.name + jsonStringify(block.input ?? {})
        )
      } else {
        // server_tool_use, web_search_tool_result, etc.
        totalTokens += roughTokenCountEstimation(jsonStringify(block))
      }
    }
  }

  // Conservative padding: multiply by 4/3 since estimation is approximate
  return Math.ceil(totalTokens * (4 / 3))
}
```

**roughTokenCountEstimation baseline:** ~4 characters per token (conservative)

---

## Part 4: Auto-Compaction Triggers & Thresholds

**File:** `autoCompact.ts` (complete threshold system)

### Effective Context Window Calculation

```typescript
export function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY  // 20_000
  )

  let contextWindow = getContextWindowForModel(model, getSdkBetas())

  // Allow override for testing (env var: CLAUDE_CODE_AUTO_COMPACT_WINDOW)
  const autoCompactWindow = process.env.CLAUDE_CODE_AUTO_COMPACT_WINDOW
  if (autoCompactWindow) {
    const parsed = parseInt(autoCompactWindow, 10)
    if (!isNaN(parsed) && parsed > 0) {
      contextWindow = Math.min(contextWindow, parsed)
    }
  }

  return contextWindow - reservedTokensForSummary
}
```

### Threshold Layering

```typescript
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000           // Proactive threshold
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000     // User warning
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000       // Error state
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000         // Blocking limit

export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)

  let autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS

  // Allow percentage override for testing (env var: CLAUDE_AUTOCOMPACT_PCT_OVERRIDE)
  const envPercent = process.env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE
  if (envPercent) {
    const parsed = parseFloat(envPercent)
    if (!isNaN(parsed) && parsed > 0 && parsed <= 100) {
      const percentageThreshold = Math.floor(
        effectiveContextWindow * (parsed / 100)
      )
      return Math.min(percentageThreshold, autocompactThreshold)
    }
  }

  return autocompactThreshold
}
```

### Warning State Calculation

```typescript
export function calculateTokenWarningState(
  tokenUsage: number,
  model: string
): {
  percentLeft: number
  isAboveWarningThreshold: boolean
  isAboveErrorThreshold: boolean
  isAboveAutoCompactThreshold: boolean
  isAtBlockingLimit: boolean
} {
  const autoCompactThreshold = getAutoCompactThreshold(model)
  const threshold = isAutoCompactEnabled()
    ? autoCompactThreshold
    : getEffectiveContextWindowSize(model)

  const percentLeft = Math.max(
    0,
    Math.round(((threshold - tokenUsage) / threshold) * 100)
  )

  const warningThreshold = threshold - WARNING_THRESHOLD_BUFFER_TOKENS
  const errorThreshold = threshold - ERROR_THRESHOLD_BUFFER_TOKENS

  const isAboveWarningThreshold = tokenUsage >= warningThreshold
  const isAboveErrorThreshold = tokenUsage >= errorThreshold
  const isAboveAutoCompactThreshold =
    isAutoCompactEnabled() && tokenUsage >= autoCompactThreshold

  const actualContextWindow = getEffectiveContextWindowSize(model)
  const defaultBlockingLimit = actualContextWindow - MANUAL_COMPACT_BUFFER_TOKENS

  // Allow override (env var: CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE)
  const blockingLimitOverride = process.env.CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE
  const blockingLimit = blockingLimitOverride
    ? parseInt(blockingLimitOverride, 10)
    : defaultBlockingLimit

  const isAtBlockingLimit = tokenUsage >= blockingLimit

  return {
    percentLeft,
    isAboveWarningThreshold,
    isAboveErrorThreshold,
    isAboveAutoCompactThreshold,
    isAtBlockingLimit
  }
}
```

### Auto-Compact Decision Logic

```typescript
export async function shouldAutoCompact(
  messages: Message[],
  model: string,
  querySource?: QuerySource,
  snipTokensFreed = 0
): Promise<boolean> {
  // Recursion guards (forked agents skip)
  if (querySource === 'session_memory' || querySource === 'compact') {
    return false
  }

  // Context-collapse agent protection
  if (feature('CONTEXT_COLLAPSE') && querySource === 'marble_origami') {
    return false
  }

  // Global disable checks
  if (!isAutoCompactEnabled()) {
    return false
  }

  // Reactive-only mode suppression
  if (feature('REACTIVE_COMPACT')) {
    if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_cobalt_raccoon', false)) {
      return false
    }
  }

  // Context-collapse mode suppression
  if (feature('CONTEXT_COLLAPSE')) {
    const { isContextCollapseEnabled } =
      require('../contextCollapse/index.js')
    if (isContextCollapseEnabled()) {
      return false
    }
  }

  // Calculate token usage accounting for snip savings
  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const threshold = getAutoCompactThreshold(model)
  const effectiveWindow = getEffectiveContextWindowSize(model)

  logForDebugging(
    `autocompact: tokens=${tokenCount} threshold=${threshold} effectiveWindow=${effectiveWindow}${snipTokensFreed > 0 ? ` snipFreed=${snipTokensFreed}` : ''}`
  )

  const { isAboveAutoCompactThreshold } = calculateTokenWarningState(tokenCount, model)
  return isAboveAutoCompactThreshold
}
```

### Full autoCompactIfNeeded Flow

```typescript
export async function autoCompactIfNeeded(
  messages: Message[],
  toolUseContext: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  querySource?: QuerySource,
  tracking?: AutoCompactTrackingState,
  snipTokensFreed?: number
): Promise<{
  wasCompacted: boolean
  compactionResult?: CompactionResult
  consecutiveFailures?: number
}> {
  // Circuit breaker check
  if (
    tracking?.consecutiveFailures !== undefined &&
    tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES
  ) {
    return { wasCompacted: false }
  }

  const model = toolUseContext.options.mainLoopModel
  const shouldCompact = await shouldAutoCompact(
    messages,
    model,
    querySource,
    snipTokensFreed
  )

  if (!shouldCompact) {
    return { wasCompacted: false }
  }

  // Build recompaction info for telemetry
  const recompactionInfo: RecompactionInfo = {
    isRecompactionInChain: tracking?.compacted === true,
    turnsSincePreviousCompact: tracking?.turnCounter ?? -1,
    previousCompactTurnId: tracking?.turnId,
    autoCompactThreshold: getAutoCompactThreshold(model),
    querySource
  }

  // Try session memory compaction first (experimental path)
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages,
    toolUseContext.agentId,
    recompactionInfo.autoCompactThreshold
  )
  if (sessionMemoryResult) {
    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)
    if (feature('PROMPT_CACHE_BREAK_DETECTION')) {
      notifyCompaction(querySource ?? 'compact', toolUseContext.agentId)
    }
    markPostCompaction()
    return {
      wasCompacted: true,
      compactionResult: sessionMemoryResult
    }
  }

  // Fall back to full summarization
  try {
    const compactionResult = await compactConversation(
      messages,
      toolUseContext,
      cacheSafeParams,
      true,  // suppressFollowUpQuestions for auto-compact
      undefined,  // No custom instructions
      true,  // isAutoCompact
      recompactionInfo
    )

    setLastSummarizedMessageId(undefined)
    runPostCompactCleanup(querySource)

    return {
      wasCompacted: true,
      compactionResult,
      consecutiveFailures: 0  // Reset on success
    }
  } catch (error) {
    if (!hasExactErrorMessage(error, ERROR_MESSAGE_USER_ABORT)) {
      logError(error)
    }

    const prevFailures = tracking?.consecutiveFailures ?? 0
    const nextFailures = prevFailures + 1
    if (nextFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
      logForDebugging(
        `autocompact: circuit breaker tripped after ${nextFailures} consecutive failures`,
        { level: 'warn' }
      )
    }

    return { wasCompacted: false, consecutiveFailures: nextFailures }
  }
}
```

---

## Part 5: API-Level Context Management (Native cache_edits)

**File:** `apiMicrocompact.ts`

Exposes the native Anthropic API `cache_edits` parameter for server-side context management:

```typescript
export type ContextEditStrategy =
  | {
      type: 'clear_tool_uses_20250919'
      trigger?: { type: 'input_tokens'; value: number }
      keep?: { type: 'tool_uses'; value: number }
      clear_tool_inputs?: boolean | string[]
      exclude_tools?: string[]
      clear_at_least?: { type: 'input_tokens'; value: number }
    }
  | {
      type: 'clear_thinking_20251015'
      keep: { type: 'thinking_turns'; value: number } | 'all'
    }

export type ContextManagementConfig = {
  edits: ContextEditStrategy[]
}
```

### Trigger Thresholds

```typescript
const DEFAULT_MAX_INPUT_TOKENS = 180_000          // Trigger at 180K tokens
const DEFAULT_TARGET_INPUT_TOKENS = 40_000        // Keep 40K tokens after clear
```

### Strategy Construction

```typescript
export function getAPIContextManagement(options?: {
  hasThinking?: boolean
  isRedactThinkingActive?: boolean
  clearAllThinking?: boolean
}): ContextManagementConfig | undefined {
  const strategies: ContextEditStrategy[] = []

  // Thinking preservation (non-redacted only)
  if (options?.hasThinking && !options?.isRedactThinkingActive) {
    strategies.push({
      type: 'clear_thinking_20251015',
      keep: options?.clearAllThinking
        ? { type: 'thinking_turns', value: 1 }
        : 'all'
    })
  }

  // Tool clearing (ant-only)
  if (process.env.USER_TYPE !== 'ant') {
    return strategies.length > 0 ? { edits: strategies } : undefined
  }

  const useClearToolResults = isEnvTruthy(process.env.USE_API_CLEAR_TOOL_RESULTS)
  const useClearToolUses = isEnvTruthy(process.env.USE_API_CLEAR_TOOL_USES)

  if (!useClearToolResults && !useClearToolUses) {
    return strategies.length > 0 ? { edits: strategies } : undefined
  }

  // Tool results clearing (ast results of compactable tools)
  if (useClearToolResults) {
    const triggerThreshold = parseInt(process.env.API_MAX_INPUT_TOKENS ?? '180000')
    const keepTarget = parseInt(process.env.API_TARGET_INPUT_TOKENS ?? '40000')

    const TOOLS_CLEARABLE_RESULTS = [
      ...SHELL_TOOL_NAMES,
      GLOB_TOOL_NAME,
      GREP_TOOL_NAME,
      FILE_READ_TOOL_NAME,
      WEB_FETCH_TOOL_NAME,
      WEB_SEARCH_TOOL_NAME
    ]

    strategies.push({
      type: 'clear_tool_uses_20250919',
      trigger: { type: 'input_tokens', value: triggerThreshold },
      clear_at_least: { type: 'input_tokens', value: triggerThreshold - keepTarget },
      clear_tool_inputs: TOOLS_CLEARABLE_RESULTS
    })
  }

  // Tool uses clearing (drop file edits/writes)
  if (useClearToolUses) {
    const triggerThreshold = parseInt(process.env.API_MAX_INPUT_TOKENS ?? '180000')
    const keepTarget = parseInt(process.env.API_TARGET_INPUT_TOKENS ?? '40000')

    const TOOLS_CLEARABLE_USES = [
      FILE_EDIT_TOOL_NAME,
      FILE_WRITE_TOOL_NAME,
      NOTEBOOK_EDIT_TOOL_NAME
    ]

    strategies.push({
      type: 'clear_tool_uses_20250919',
      trigger: { type: 'input_tokens', value: triggerThreshold },
      clear_at_least: { type: 'input_tokens', value: triggerThreshold - keepTarget },
      exclude_tools: TOOLS_CLEARABLE_USES  // Keep these, clear others
    })
  }

  return strategies.length > 0 ? { edits: strategies } : undefined
}
```

---

## Part 6: Session Memory Compaction (Experimental)

**File:** `sessionMemoryCompact.ts` (630 lines)

Alternative to full message summarization: uses pre-extracted session memory file instead.

### Configuration

```typescript
type SessionMemoryCompactConfig = {
  minTokens: number                    // Preserve at least this many tokens
  minTextBlockMessages: number         // Preserve at least N messages with text
  maxTokens: number                    // Hard cap on preserved content
}

const DEFAULT_SM_COMPACT_CONFIG = {
  minTokens: 10_000,
  minTextBlockMessages: 5,
  maxTokens: 40_000
}
```

Configuration is loaded from GrowthBook (`tengu_sm_compact_config`) once per session.

### Message Preservation Calculation

**File:** `sessionMemoryCompact.ts:324–397`

```typescript
export function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number
): number {
  const config = getSessionMemoryCompactConfig()

  // Start from message after lastSummarizedIndex
  let startIndex = lastSummarizedIndex >= 0 ? lastSummarizedIndex + 1 : messages.length

  // Calculate current tokens and text-block count
  let totalTokens = 0
  let textBlockMessageCount = 0
  for (let i = startIndex; i < messages.length; i++) {
    totalTokens += estimateMessageTokens([messages[i]!])
    if (hasTextBlocks(messages[i]!)) {
      textBlockMessageCount++
    }
  }

  // Already at max cap
  if (totalTokens >= config.maxTokens) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // Already meet both minimums
  if (totalTokens >= config.minTokens &&
      textBlockMessageCount >= config.minTextBlockMessages) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // Expand backwards until meeting both minimums or hitting max cap
  const idx = messages.findLastIndex(m => isCompactBoundaryMessage(m))
  const floor = idx === -1 ? 0 : idx + 1  // Don't go before compact boundary

  for (let i = startIndex - 1; i >= floor; i--) {
    const msg = messages[i]!
    const msgTokens = estimateMessageTokens([msg])
    totalTokens += msgTokens
    if (hasTextBlocks(msg)) {
      textBlockMessageCount++
    }
    startIndex = i

    if (totalTokens >= config.maxTokens) {
      break
    }

    if (totalTokens >= config.minTokens &&
        textBlockMessageCount >= config.minTextBlockMessages) {
      break
    }
  }

  return adjustIndexToPreserveAPIInvariants(messages, startIndex)
}
```

### API Invariant Preservation

**File:** `sessionMemoryCompact.ts:232–314`

Adjusts start index to prevent splitting tool_use/tool_result pairs and thinking blocks:

```typescript
export function adjustIndexToPreserveAPIInvariants(
  messages: Message[],
  startIndex: number
): number {
  if (startIndex <= 0 || startIndex >= messages.length) {
    return startIndex
  }

  let adjustedIndex = startIndex

  // Step 1: Handle tool_use/tool_result pairs
  // Collect ALL tool_result IDs from kept range
  const allToolResultIds: string[] = []
  for (let i = startIndex; i < messages.length; i++) {
    allToolResultIds.push(...getToolResultIds(messages[i]!))
  }

  if (allToolResultIds.length > 0) {
    // Find tool_use IDs already in kept range
    const toolUseIdsInKeptRange = new Set<string>()
    for (let i = adjustedIndex; i < messages.length; i++) {
      const msg = messages[i]!
      if (msg.type === 'assistant' && Array.isArray(msg.message.content)) {
        for (const block of msg.message.content) {
          if (block.type === 'tool_use') {
            toolUseIdsInKeptRange.add(block.id)
          }
        }
      }
    }

    // Look for tool_uses NOT already in kept range
    const neededToolUseIds = new Set(
      allToolResultIds.filter(id => !toolUseIdsInKeptRange.has(id))
    )

    // Find preceding assistant messages with matching tool_uses
    for (let i = adjustedIndex - 1; i >= 0 && neededToolUseIds.size > 0; i--) {
      if (hasToolUseWithIds(messages[i]!, neededToolUseIds)) {
        adjustedIndex = i
        // Remove found IDs
        const msg = messages[i]!
        if (msg.type === 'assistant' && Array.isArray(msg.message.content)) {
          for (const block of msg.message.content) {
            if (block.type === 'tool_use' && neededToolUseIds.has(block.id)) {
              neededToolUseIds.delete(block.id)
            }
          }
        }
      }
    }
  }

  // Step 2: Handle thinking blocks sharing message.id with kept assistant messages
  const messageIdsInKeptRange = new Set<string>()
  for (let i = adjustedIndex; i < messages.length; i++) {
    const msg = messages[i]!
    if (msg.type === 'assistant' && msg.message.id) {
      messageIdsInKeptRange.add(msg.message.id)
    }
  }

  // Look backwards for messages with same message.id (may contain thinking)
  for (let i = adjustedIndex - 1; i >= 0; i--) {
    const msg = messages[i]!
    if (msg.type === 'assistant' &&
        msg.message.id &&
        messageIdsInKeptRange.has(msg.message.id)) {
      adjustedIndex = i
    }
  }

  return adjustedIndex
}
```

---

## Part 7: Message Grouping for API Safety

**File:** `grouping.ts` (63 lines)

Groups messages by API-round boundaries for PTL retry:

```typescript
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined  // message.id of last seen assistant

  for (const msg of messages) {
    // Boundary: new assistant response (different message.id)
    if (
      msg.type === 'assistant' &&
      msg.message.id !== lastAssistantId &&
      current.length > 0
    ) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }

    if (msg.type === 'assistant') {
      lastAssistantId = msg.message.id
    }
  }

  if (current.length > 0) {
    groups.push(current)
  }

  return groups
}
```

**Key insight:** API contract guarantees all tool_uses are resolved before next assistant turn, so message.id is the sole boundary gate. No need to track unresolved tool_use IDs.

---

## Part 8: Post-Compaction State Reset

**File:** `postCompactCleanup.ts` (77 lines)

Clears caches and tracking state invalidated by compaction:

```typescript
export function runPostCompactCleanup(querySource?: QuerySource): void {
  // Determine if this is main-thread compact (skip for subagents)
  const isMainThreadCompact =
    querySource === undefined ||
    querySource.startsWith('repl_main_thread') ||
    querySource === 'sdk'

  // Always reset
  resetMicrocompactState()
  clearSystemPromptSections()
  clearClassifierApprovals()
  clearSpeculativeChecks()
  clearBetaTracingState()

  // Main-thread only
  if (isMainThreadCompact) {
    // Context-collapse state
    if (feature('CONTEXT_COLLAPSE')) {
      require('../contextCollapse/index.js').resetContextCollapse()
    }

    // Memory file caches
    getUserContext.cache.clear?.()
    resetGetMemoryFilesCache('compact')
  }

  // Attribution hooks (ant-only, async)
  if (feature('COMMIT_ATTRIBUTION')) {
    void import('../../utils/attributionHooks.js').then(m =>
      m.sweepFileContentCache()
    )
  }

  // Session storage cache
  clearSessionMessagesCache()

  // NOTE: Intentionally NOT clearing sentSkillNames — re-injecting skill_listing
  // (~4K tokens) is pure cache_creation. Model retains SkillTool in schema and
  // invoked_skills preserves used skills. See compactConversation() for rationale.
}
```

**Critical Safety Note:** Subagents (querySource starts with `agent:*`) run in the same process and share module-level state with the main thread. Only reset main-thread state for main-thread compacts.

---

## Part 9: Warning & Suppression State

**File:** `compactWarningState.ts` (19 lines) + `compactWarningHook.ts` (17 lines)

Simple state machine for suppressing duplicate warnings:

```typescript
// compactWarningState.ts
export const compactWarningStore = createStore<boolean>(false)

export function suppressCompactWarning(): void {
  compactWarningStore.setState(() => true)
}

export function clearCompactWarningSuppression(): void {
  compactWarningStore.setState(() => false)
}

// compactWarningHook.ts
export function useCompactWarningSuppression(): boolean {
  return useSyncExternalStore(
    compactWarningStore.subscribe,
    compactWarningStore.getState
  )
}
```

**Usage pattern:**
1. After successful compaction: call `suppressCompactWarning()`
2. Start of new microcompact attempt: call `clearCompactWarningSuppression()`
3. UI checks `useCompactWarningSuppression()` to hide warning after successful compact (avoid noise while waiting for next API response with accurate token count)

---

## Part 10: All Constants, Thresholds & Feature Gates

### Core Thresholds (autoCompact.ts)
```typescript
MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000          // Reserve for summary generation
AUTOCOMPACT_BUFFER_TOKENS = 13_000              // Proactive threshold buffer
WARNING_THRESHOLD_BUFFER_TOKENS = 20_000        // User warning buffer
ERROR_THRESHOLD_BUFFER_TOKENS = 20_000          // Error display buffer
MANUAL_COMPACT_BUFFER_TOKENS = 3_000            // Manual compact blocking buffer
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3        // Circuit breaker threshold
```

### File & Skill Restoration (compact.ts)
```typescript
POST_COMPACT_MAX_FILES_TO_RESTORE = 5           // Max recent files to re-read
POST_COMPACT_TOKEN_BUDGET = 50_000              // Total token budget for files
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000        // Per-file token limit
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000       // Per-skill truncation limit
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000       // Total budget for all skills
```

### Microcompaction (microCompact.ts)
```typescript
TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'
IMAGE_MAX_TOKEN_SIZE = 2000                     // Conservative estimate for images

COMPACTABLE_TOOLS = {
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME
}
```

### API Cache Management (apiMicrocompact.ts)
```typescript
DEFAULT_MAX_INPUT_TOKENS = 180_000              // Trigger threshold
DEFAULT_TARGET_INPUT_TOKENS = 40_000            // Target after clearing
```

### Session Memory Compact (sessionMemoryCompact.ts)
```typescript
DEFAULT_SM_COMPACT_CONFIG = {
  minTokens: 10_000,
  minTextBlockMessages: 5,
  maxTokens: 40_000
}
```

### Compaction Streaming (compact.ts)
```typescript
MAX_COMPACT_STREAMING_RETRIES = 2               // Retry attempts for streaming
MAX_PTL_RETRIES = 3                             // Retries for prompt-too-long on compact
PTL_RETRY_MARKER = '[earlier conversation truncated for compaction retry]'
COMPACT_MAX_OUTPUT_TOKENS = 20_000              // Max output for compact response
```

### Feature Gates (GrowthBook)
```typescript
tengu_compact_cache_prefix                      // Default: true (3P), allow false
tengu_compact_streaming_retry                   // Default: false
tengu_cached_microcompact                       // Cached MC enabled?
tengu_slate_heron                               // Time-based MC config
tengu_sm_compact_config                         // SM-compact thresholds
tengu_sm_compact                                // SM-compact enabled?
tengu_session_memory                            // Session memory extraction enabled?
tengu_cobalt_raccoon                            // Suppress proactive autocompact (reactive-only mode)
CONTEXT_COLLAPSE (feature gate)                 // Suppress autocompact in collapse mode
PROMPT_CACHE_BREAK_DETECTION (feature gate)     // Notify cache baseline resets
KAIROS (feature gate)                           // Session transcript persistence
PROACTIVE (feature gate)                        // Autonomous mode instructions
CACHED_MICROCOMPACT (feature gate)              // Native cache_edits support
```

### Environment Overrides
```typescript
CLAUDE_CODE_AUTO_COMPACT_WINDOW                 // Override context window size
CLAUDE_AUTOCOMPACT_PCT_OVERRIDE                 // Override as % of window
CLAUDE_CODE_BLOCKING_LIMIT_OVERRIDE             // Override blocking limit
DISABLE_COMPACT                                 // Disable all compaction
DISABLE_AUTO_COMPACT                            // Disable only auto-compact
ENABLE_CLAUDE_CODE_SM_COMPACT                   // Force SM-compact on
DISABLE_CLAUDE_CODE_SM_COMPACT                  // Force SM-compact off
USE_API_CLEAR_TOOL_RESULTS                      // Enable API tool result clearing
USE_API_CLEAR_TOOL_USES                         // Enable API tool use clearing
API_MAX_INPUT_TOKENS                            // API strategy trigger threshold
API_TARGET_INPUT_TOKENS                         // API strategy target tokens
USER_TYPE                                        // "ant" for internal builds
```

---

## Part 11: Cache-Safe Design

### Prompt Cache Prefix Sharing

**Strategy:** The forked agent reuses the main thread's cached prefix by sending **identical** system prompt, tools, and model. This reduces compaction API costs by ~76% and enables feature `tengu_compact_cache_prefix`.

```typescript
// Fork inherits from cacheSafeParams:
{
  systemPrompt,
  tools,
  model,
  messages (prefix for context)  // Matches main thread
}

// When fork response completes, cache hit recorded:
logEvent('tengu_compact_cache_sharing_success', {
  cacheReadInputTokens,
  cacheCreationInputTokens,
  cacheHitRate: cacheRead / (cacheRead + cacheCreate + input)
})
```

### cache_edits Integration

**Strategy:** Cached microcompaction uses native API `cache_edits` parameter (added at API call site, not locally). This preserves the cached prefix while surgically removing tool results.

**Two types of edits:**
1. `clear_tool_uses_20250919` — Remove specific tool results by ID
2. `clear_thinking_20251015` — Control thinking block retention

### Cache Break Detection Avoidance

After compaction, system calls:
```typescript
if (feature('PROMPT_CACHE_BREAK_DETECTION')) {
  notifyCompaction(querySource, agentId)
}
```

This registers a cache baseline reset, so the post-compact drop in `cache_read_input_tokens` is not flagged as a false-positive "break" (system prompt change, network issue, etc.).

### Boundary Annotations for Relink

**File:** `compact.ts:341–367`

Partial compaction annotates boundaries with preserved segment metadata for loader relink:

```typescript
export function annotateBoundaryWithPreservedSegment(
  boundary: SystemCompactBoundaryMessage,
  anchorUuid: UUID,
  messagesToKeep: readonly Message[] | undefined
): SystemCompactBoundaryMessage {
  const keep = messagesToKeep ?? []
  if (keep.length === 0) return boundary

  return {
    ...boundary,
    compactMetadata: {
      ...boundary.compactMetadata,
      preservedSegment: {
        headUuid: keep[0]!.uuid,       // First preserved message
        anchorUuid,                    // Summary or boundary (where chain resumes)
        tailUuid: keep.at(-1)!.uuid    // Last preserved message
      }
    }
  }
}
```

The loader uses this to repair message chain pointers after dedup-skip (messages kept aren't re-written, just re-pointed to).

---

## Part 12: Security & Attack Surface Analysis

### Summary Manipulation Risks

**Risk:** User injects prompt-injection payloads in earlier messages, model summarizes them in the 9-section format, later turns execute the "summary" as instructions.

**Mitigations:**
1. `NO_TOOLS_PREAMBLE` + `NO_TOOLS_TRAILER` explicitly forbid tool use in compaction agent
2. Analysis section stripped post-generation — drafting scratchpad not passed to next turn
3. Summary format is structured (numbered sections) — model doesn't execute arbitrary text as instructions
4. Separate forked agent — isolation reduces scope of execution

**Residual risk:** LOW. The summarized text is framed as historical context ("the summary below covers the earlier portion"), not as executable instructions.

### Skill Re-injection Post-Compaction

**Risk:** Attacker modifies skill content during compact, re-injected skill now contains malicious code.

**Mitigations:**
1. Skills are stored in sandbox-controlled filesystem — attacker would need write access to `/Users/.../.claude/skills/` or equivalent
2. Skill content is captured at invoke time (`getInvokedSkillsForAgent()`) — modifications after invoke don't affect re-injection
3. Per-skill truncation (5K tokens max) limits metadata that could encode large payloads

**Residual risk:** VERY LOW. Skill injection is a local sandboxing problem, not a protocol vulnerability.

### Token Budget Gaming

**Risk:** User crafts messages with high-entropy content to force compaction at lower actual token count, causing repeated compactions and DoS.

**Mitigations:**
1. Compaction is rate-limited by circuit breaker (`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`)
2. `tokenCountWithEstimation()` uses consistent 4-chars/token baseline — consistent under user control
3. Token estimates are validated against API `usage.input_tokens` post-call

**Residual risk:** MEDIUM. Pathological conversations could trigger many compactions, but circuit breaker prevents runaway.

### File Attachment Restoration Exhaustion

**Risk:** Post-compact file restoration reads 5 recently-accessed files from disk. Attacker repeatedly accesses huge files pre-compact to exhaust post-compact budget.

**Mitigations:**
1. File size per token cap: `POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000`
2. Total budget: `POST_COMPACT_TOKEN_BUDGET = 50_000`
3. File list filtered for excluded paths (plan files, CLAUDE.md, memory files)
4. Re-read respects original `FileReadTool` permissions and size limits

**Residual risk:** LOW. Even worst-case (5 * 5K = 25K tokens) is within budget and doesn't degrade user experience.

---

## Part 13: Integration Points & State Machine

### Query Loop Integration

```
query() [main loop]
├─ microcompactMessages()
│  ├─ evaluateTimeBasedTrigger() → maybeTimeBasedMicrocompact()
│  └─ cachedMicrocompactPath() [queues cache_edits]
├─ callModel()
│  ├─ getAPIContextManagement() [inserts cache_edits block]
│  └─ [API call with potentially modified messages + cache_edits]
├─ [Handle response]
└─ [Check token usage]
   ├─ shouldAutoCompact()
   └─ autoCompactIfNeeded()
      ├─ trySessionMemoryCompaction() [optional experimental]
      └─ compactConversation() [traditional full summarization]
         ├─ executePreCompactHooks()
         ├─ streamCompactSummary()
         │  ├─ runForkedAgent() [cache-sharing attempt]
         │  └─ queryModelWithStreaming() [fallback]
         ├─ createPostCompactFileAttachments()
         ├─ createSkillAttachmentIfNeeded()
         ├─ executePostCompactHooks()
         └─ logEvent('tengu_compact')
      └─ runPostCompactCleanup()
         ├─ resetMicrocompactState()
         ├─ clearSystemPromptSections()
         ├─ resetGetMemoryFilesCache()
         └─ clearSessionMessagesCache()
```

### State Transitions

```
[NORMAL] (tokens < threshold)
└─ isAboveWarningThreshold? → [WARNING]
   └─ isAboveAutoCompactThreshold? → [AUTOCOMPACT_PENDING]
      └─ autoCompactIfNeeded() succeeds → [COMPACTED]
         └─ runPostCompactCleanup() → [NORMAL] (reset baseline)
      └─ autoCompactIfNeeded() fails → [CIRCUIT_BREAKER] (after 3 failures)
         └─ Skip all future compaction attempts

[WARNING] (tokens >= warningThreshold)
└─ User sees warning: "N% context left until autocompact"
   └─ suppressCompactWarning() [after successful compact]
   └─ clearCompactWarningSuppression() [start new compact attempt]

[ERROR] (tokens >= errorThreshold)
└─ User sees error: "Context full, compacting required"

[BLOCKING] (tokens >= blockingLimit)
└─ Cannot send message, must manually /compact
```

---

## Conclusion

Claude Code's compaction system is a sophisticated multi-tier architecture balancing three competing goals:

1. **Efficiency:** Cache-sharing forks reduce compaction costs by 76%; cached microcompaction eliminates API calls entirely
2. **Safety:** Circuit breaker prevents cascade failures; strict tool denial in compaction agent prevents prompt injection
3. **Usability:** 9-section summary format preserves all critical details for context continuation; rich re-injection (files, skills, plans) restores model's working context post-compact

The token budget model is fully deterministic and testable: all thresholds derive from effective context window, and all decisions are logged for analysis. The 13K proactive buffer ensures API prompt-too-long errors are prevented during normal operation, while the reactive-compact fallback handles edge cases where context grows unexpectedly.

**Total codebase:** 3,960 lines across 11 files, with clear separation of concerns: core orchestration (compact.ts), trigger logic (autoCompact.ts, microCompact.ts), data prep (prompt.ts, grouping.ts), state cleanup (postCompactCleanup.ts), and peripheral UI (compactWarningState.ts, compactWarningHook.ts).

