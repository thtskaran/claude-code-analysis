# Claude Code v2.1.88 API Client Subsystem - Deep-Dive Analysis

## Executive Summary

Claude Code's API client subsystem is a sophisticated, multi-layered system engineered to:

1. **Orchestrate streaming and non-streaming API interactions** with the Anthropic SDK
2. **Support multi-provider authentication** (1P API Key, OAuth, AWS Bedrock, Azure Foundry, Google Vertex AI)
3. **Implement intelligent retry logic** with exponential backoff, 529 overload handling, and fallback model strategies
4. **Manage prompt caching** with cache-break detection, TTL scoping, and session stability
5. **Persist session state** with idempotent transcript ingestion and conflict resolution
6. **Track usage, quotas, and rate limits** with sophisticated overage management
7. **Instrument telemetry** via analytics events, request tracing, and performance profiling

This document provides an exhaustive technical breakdown of the architecture, data structures, protocols, and algorithms that power this system.

---

## Part 1: Core API Request Orchestration (claude.ts)

### 1.1 Architecture Overview

**File**: `/sessions/cool-friendly-einstein/mnt/claude-code/src/services/api/claude.ts` (3,419 lines)

Claude.ts is the central orchestrator for API requests. It implements two request pathways:

```
User Request
    ↓
queryModel (async generator)
    ├── [Pre-call Phase] System Prompt Construction, Tool Schema Building
    ├── [Streaming Path] withRetry() → SDK streaming → Event demultiplexing
    ├── [Fallback Path] executeNonStreamingRequest() → Non-streaming API
    ├── [Post-call Phase] Usage tracking, cost calculation, cache-break detection
    └── [Error Path] Error classification, user message formatting

queryModelWithStreaming (public API)
    ↓
Yields: StreamEvent | AssistantMessage | SystemAPIErrorMessage

queryModelWithoutStreaming (public API)
    ↓
Returns: AssistantMessage
```

### 1.2 Request Pipeline: Pre-Call Phase

#### 1.2.1 System Prompt Construction

**Function**: `queryModel()`, lines 1017-1503

The system prompt is constructed via:

```typescript
systemPrompt = asSystemPrompt([
  getAttributionHeader(fingerprint),           // Source attribution
  getCLISyspromptPrefix(options),              // CLI-specific header
  ...systemPrompt,                              // User-provided system
  ...(advisorModel ? [ADVISOR_TOOL_INSTRUCTIONS] : []),  // Advisor instructions
  ...(injectChromeHere ? [CHROME_TOOL_SEARCH_INSTRUCTIONS] : []),  // Chrome MCP instructions
].filter(Boolean))

// Wrapped as system_prompt blocks with cache_control:
system = buildSystemPromptBlocks(systemPrompt, enablePromptCaching, {
  skipGlobalCacheForSystemPrompt: needsToolBasedCacheMarker,
  querySource: options.querySource,
})
```

**Key Design Decisions**:
- **Attribution fingerprinting** via `computeFingerprintFromMessages()` — embeds user fingerprint in system prompt for attribution
- **Cache control injection** — appends `cache_control: { type: 'ephemeral', ttl?: '1h', scope?: 'global' }` to last system block
- **1h TTL gating** — enabled only for:
  - Anthropic internal users (ant)
  - Claude AI subscribers within quota
  - Query source on GrowthBook allowlist (pattern matching: prefix with `*` suffix)
- **Global cache scope** — enables only when no MCP tools are present (MCP tools are dynamic, per-user, can't be globally cached)

#### 1.2.2 Tool Schema Construction

**Lines**: 1064-1246

Tool processing:

```typescript
// 1. Determine if tool search is enabled
let useToolSearch = await isToolSearchEnabled(model, tools, ...)

// 2. Identify deferred tools (MCP tools with dynamic loading)
const deferredToolNames = new Set<string>()
if (useToolSearch) {
  for (const t of tools) {
    if (isDeferredTool(t)) deferredToolNames.add(t.name)
  }
}

// 3. Filter tools based on discovery state and search mode
let filteredTools: Tools
if (useToolSearch) {
  const discoveredToolNames = extractDiscoveredToolNames(messages)
  filteredTools = tools.filter(tool => {
    if (!deferredToolNames.has(tool.name)) return true      // Keep non-deferred
    if (toolMatchesName(tool, TOOL_SEARCH_TOOL_NAME)) return true  // Keep ToolSearch
    return discoveredToolNames.has(tool.name)  // Only deferred tools already discovered
  })
} else {
  filteredTools = tools.filter(t => !toolMatchesName(t, TOOL_SEARCH_TOOL_NAME))
}

// 4. Build tool schemas with defer_loading flag
const toolSchemas = await Promise.all(
  filteredTools.map(tool =>
    toolToAPISchema(tool, {
      getToolPermissionContext: options.getToolPermissionContext,
      tools,
      agents: options.agents,
      allowedAgentTypes: options.allowedAgentTypes,
      model: options.model,
      deferLoading: willDefer(tool),  // Set defer_loading: true for MCP tools in tool-search mode
    }),
  ),
)
```

**Defer Loading Mechanism**:
- **Purpose**: MCP tools are discovered dynamically via tool_reference blocks, not predeclared
- **Effect**: API omits deferred tools from the model's tool list until tool_reference discovery
- **Benefit**: Eliminates tool quantity limits and allows hot-adding tools mid-conversation

#### 1.2.3 Message Normalization

**Lines**: 1259-1345

Messages flow through a normalization pipeline:

```typescript
// 1. Normalize for API (handles backward compatibility)
let messagesForAPI = normalizeMessagesForAPI(messages, filteredTools)

// 2. Model-specific post-processing (remove tool-search fields if unsupported)
if (!useToolSearch) {
  messagesForAPI = messagesForAPI.map(msg => {
    switch (msg.type) {
      case 'user':
        return stripToolReferenceBlocksFromUserMessage(msg)
      case 'assistant':
        return stripCallerFieldFromAssistantMessage(msg)
      default:
        return msg
    }
  })
}

// 3. Repair tool_use/tool_result pairing (handles remote session resumption)
messagesForAPI = ensureToolResultPairing(messagesForAPI)

// 4. Strip advisor blocks if not using advisor beta
if (!betas.includes(ADVISOR_BETA_HEADER)) {
  messagesForAPI = stripAdvisorBlocks(messagesForAPI)
}

// 5. Cap media items (>100 media items causes API rejection)
messagesForAPI = stripExcessMediaItems(messagesForAPI, API_MAX_MEDIA_PER_REQUEST)
```

**Key Invariants**:
- Tool result pairing: Every `tool_use` must have matching `tool_result` in user message
- Media cap: Maximum `API_MAX_MEDIA_PER_REQUEST` (100) media items across all messages
- Block coherence: Only send blocks the API supports (advisor, tool_reference only with beta headers)

#### 1.2.4 Beta Headers and Feature Flags

**Lines**: 1071-1256

Beta headers control API features:

```typescript
const betas = getMergedBetas(options.model, { isAgenticQuery })

// Conditionally append beta headers:
if (isAdvisorEnabled()) {
  betas.push(ADVISOR_BETA_HEADER)  // 'advisor-2025-03-12'
}

if (useToolSearch) {
  const toolSearchHeader = getToolSearchBetaHeader()
  if (getAPIProvider() !== 'bedrock') {
    if (!betas.includes(toolSearchHeader)) {
      betas.push(toolSearchHeader)
    }
  } else {
    // Bedrock: headers go in extraBodyParams, not betas array
  }
}

// Latched headers (sticky-on per session):
// These never change mid-session to avoid cache key flips
if (!afkHeaderLatched && shouldIncludeFirstPartyOnlyBetas() && isAutoModeActive()) {
  afkHeaderLatched = true
  setAfkModeHeaderLatched(true)
}

if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true)
}
```

**Critical Design**: Latched headers prevent mid-session cache-key flips:
- User toggles fast mode: header is latched ➜ speed parameter stays dynamic ➜ cache stable
- Auto mode activates: AFK_MODE_BETA_HEADER latched ➜ classifier queries also get header ➜ consistent cache

### 1.3 Request Execution: Streaming Path

**Lines**: 1776-2114

The streaming request is wrapped in `withRetry()` which implements automatic retry logic:

```typescript
const generator = withRetry(
  () => getAnthropicClient({ maxRetries: 0, model, fetchOverride, source }),
  async (anthropic, attempt, context) => {
    const params = paramsFromContext(context)
    captureAPIRequest(params, options.querySource)

    // Generate client request ID for correlation (1P only)
    clientRequestId = getAPIProvider() === 'firstParty' && isFirstPartyAnthropicBaseUrl()
      ? randomUUID()
      : undefined

    // Use raw stream, not BetaMessageStream (avoids O(n²) partial JSON parsing)
    const result = await anthropic.beta.messages
      .create({ ...params, stream: true }, {
        signal,
        ...(clientRequestId && { headers: { [CLIENT_REQUEST_ID_HEADER]: clientRequestId } }),
      })
      .withResponse()

    streamRequestId = result.request_id
    streamResponse = result.response
    return result.data  // Stream<BetaRawMessageStreamEvent>
  },
  {
    model: options.model,
    fallbackModel: options.fallbackModel,
    thinkingConfig,
    ...(isFastModeEnabled() ? { fastMode: isFastMode } : false),
    signal,
    querySource: options.querySource,
  },
)
```

#### 1.3.1 Streaming Idle Watchdog

**Lines**: 1868-1929

Network timeouts don't trigger on streaming bodies (only on initial fetch). Claude Code implements a client-side idle watchdog:

```typescript
const STREAM_IDLE_TIMEOUT_MS = parseInt(process.env.CLAUDE_STREAM_IDLE_TIMEOUT_MS || '', 10) || 90_000
const STREAM_IDLE_WARNING_MS = STREAM_IDLE_TIMEOUT_MS / 2  // 45s default

function resetStreamIdleTimer(): void {
  clearStreamIdleTimers()
  if (!streamWatchdogEnabled) return

  streamIdleWarningTimer = setTimeout(() => {
    logForDebugging(`Streaming idle warning: no chunks for ${STREAM_IDLE_WARNING_MS / 1000}s`)
    logEvent('tengu_streaming_idle_warning', { timeout_ms: STREAM_IDLE_TIMEOUT_MS })
  }, STREAM_IDLE_WARNING_MS, STREAM_IDLE_WARNING_MS)

  streamIdleTimer = setTimeout(() => {
    streamIdleAborted = true
    streamWatchdogFiredAt = performance.now()
    logEvent('tengu_streaming_idle_timeout', {
      model: options.model,
      request_id: streamRequestId ?? 'unknown',
      timeout_ms: STREAM_IDLE_TIMEOUT_MS,
    })
    releaseStreamResources()
  }, STREAM_IDLE_TIMEOUT_MS)
}
```

**Two-tier protection**:
1. **Warning at 50%** (45s default): Logged for diagnostic visibility
2. **Abort at 100%** (90s default): Calls `signal.abort()`, triggers non-streaming fallback

#### 1.3.2 Stream Event Demultiplexing

**Lines**: 1940-2114

Raw message stream events flow through a state machine:

```typescript
for await (const part of stream) {
  resetStreamIdleTimer()  // Feed the watchdog

  // Detect stalls (>30 second gaps between chunks)
  if (lastEventTime !== null) {
    const timeSinceLastEvent = now - lastEventTime
    if (timeSinceLastEvent > STALL_THRESHOLD_MS) {
      stallCount++
      totalStallTime += timeSinceLastEvent
      logEvent('tengu_streaming_stall', {
        stall_duration_ms: timeSinceLastEvent,
        stall_count: stallCount,
        total_stall_time_ms: totalStallTime,
        event_type: part.type,
      })
    }
  }
  lastEventTime = now

  switch (part.type) {
    case 'message_start':
      partialMessage = part.message
      ttftMs = Date.now() - start  // Time to first token
      usage = updateUsage(usage, part.message?.usage)

      // Capture research field (ant-only)
      if (process.env.USER_TYPE === 'ant' && 'research' in part.message) {
        research = (part.message as any).research
      }
      break

    case 'content_block_start':
      switch (part.content_block.type) {
        case 'tool_use':
          contentBlocks[part.index] = { ...part.content_block, input: '' }
          break
        case 'server_tool_use':
          contentBlocks[part.index] = { ...part.content_block, input: {} }
          if (part.content_block.name === 'advisor') {
            isAdvisorInProgress = true
            logEvent('tengu_advisor_tool_call', { model: options.model, advisor_model: advisorModel })
          }
          break
        case 'text':
          contentBlocks[part.index] = { ...part.content_block, text: '' }
          break
        case 'thinking':
          contentBlocks[part.index] = { ...part.content_block, thinking: '', signature: '' }
          break
        default:
          contentBlocks[part.index] = { ...part.content_block }
          if ((part.content_block.type as string) === 'advisor_tool_result') {
            isAdvisorInProgress = false
          }
          break
      }
      break

    case 'content_block_delta':
      const contentBlock = contentBlocks[part.index]
      if (!contentBlock) throw new RangeError('Content block not found')

      if (feature('CONNECTOR_TEXT') && delta.type === 'connector_text_delta') {
        if (contentBlock.type !== 'connector_text') throw new Error('Type mismatch')
        contentBlock.connector_text += delta.connector_text
      } else {
        switch (delta.type) {
          case 'input_json_delta':
            if (contentBlock.type !== 'tool_use' && contentBlock.type !== 'server_tool_use') {
              throw new Error('Type mismatch')
            }
            contentBlock.input += delta.input_json
            break
          case 'text_delta':
            if (contentBlock.type !== 'text') throw new Error('Type mismatch')
            contentBlock.text += delta.text
            break
          case 'thinking_delta':
            if (contentBlock.type !== 'thinking') throw new Error('Type mismatch')
            contentBlock.thinking += delta.thinking
            break
          // ... other delta types
        }
      }
      break

    case 'message_delta':
      usage = updateUsage(usage, part.delta?.usage)
      stopReason = part.delta?.stop_reason
      break

    case 'message_stop':
      // Stream complete
      break
  }
}
```

**Key Invariants**:
- Tool inputs accumulate via JSON delta streaming (no re-parsing)
- Thinking blocks accumulate token-by-token
- Connector text accumulated (experimental text block type)
- Usage is merged on each update (input tokens stable, output accumulates)

### 1.4 Fallback to Non-Streaming

**Function**: `executeNonStreamingRequest()`, lines 818-917

When streaming fails with 529 or timeout, Claude Code falls back to non-streaming:

```typescript
const fallbackTimeoutMs = getNonstreamingFallbackTimeoutMs()
// Remote: 120s (under CCR idle kill ~5min)
// Local: 300s (long enough for slow backends, under 10min API boundary)

const generator = withRetry(
  () => getAnthropicClient({ maxRetries: 0, model, fetchOverride, source }),
  async (anthropic, attempt, context) => {
    const start = Date.now()
    const retryParams = paramsFromContext(context)
    captureRequest(retryParams)
    onAttempt(attempt, start, retryParams.max_tokens)

    const adjustedParams = adjustParamsForNonStreaming(retryParams, MAX_NON_STREAMING_TOKENS)

    try {
      return await anthropic.beta.messages.create(
        { ...adjustedParams, model: normalizeModelStringForAPI(adjustedParams.model) },
        { signal, timeout: fallbackTimeoutMs },
      )
    } catch (err) {
      if (err instanceof APIUserAbortError) throw err

      // Instrumentation: distinguish timeout from other fallback errors
      logEvent('tengu_nonstreaming_fallback_error', {
        model: clientOptions.model,
        error: err instanceof Error ? err.name : 'unknown',
        attempt,
        timeout_ms: fallbackTimeoutMs,
        request_id: originatingRequestId ?? 'unknown',
      })
      throw err
    }
  },
  {
    model: retryOptions.model,
    fallbackModel: retryOptions.fallbackModel,
    thinkingConfig: retryOptions.thinkingConfig,
    ...(isFastModeEnabled() && { fastMode: retryOptions.fastMode }),
    signal: retryOptions.signal,
    initialConsecutive529Errors: retryOptions.initialConsecutive529Errors,
    querySource: retryOptions.querySource,
  },
)
```

**Timeout Configuration**:
- **Remote (CCR)**: `120_000ms` to complete before container idle kill
- **Local**: `300_000ms` (close to 10min API boundary, avoids pathological slow backends)
- **Override**: `API_TIMEOUT_MS` env var
- **Post-fallback**: Consumes remaining `signal` abort window; if killed mid-fallback, logs analytics event

### 1.5 Output Configuration

**Function**: `configureEffortParams()` and `configureTaskBudgetParams()`, lines 440-501

#### 1.5.1 Effort Levels

```typescript
function configureEffortParams(
  effortValue: EffortValue | undefined,
  outputConfig: BetaOutputConfig,
  extraBodyParams: Record<string, unknown>,
  betas: string[],
  model: string,
): void {
  if (!modelSupportsEffort(model) || 'effort' in outputConfig) return

  if (effortValue === undefined) {
    // No effort specified: include beta header to enable server-side auto-selection
    betas.push(EFFORT_BETA_HEADER)
  } else if (typeof effortValue === 'string') {
    // String effort level (e.g. 'high', 'medium', 'low')
    outputConfig.effort = effortValue
    betas.push(EFFORT_BETA_HEADER)
  } else if (process.env.USER_TYPE === 'ant') {
    // Numeric effort override (ant-only, uses anthropic_internal)
    const existingInternal = (extraBodyParams.anthropic_internal as Record<string, unknown>) || {}
    extraBodyParams.anthropic_internal = {
      ...existingInternal,
      effort_override: effortValue,  // e.g. 500, 1000 (thinking tokens)
    }
  }
}
```

**Effort Strategy**:
- **String levels**: User specifies 'high'/'medium'/'low' ➜ sent in output_config.effort
- **Numeric override** (ant-only): Direct control over thinking token budget via anthropic_internal

#### 1.5.2 Task Budget (EAP)

```typescript
type TaskBudgetParam = {
  type: 'tokens'
  total: number
  remaining?: number
}

export function configureTaskBudgetParams(
  taskBudget: Options['taskBudget'],
  outputConfig: BetaOutputConfig & { task_budget?: TaskBudgetParam },
  betas: string[],
): void {
  if (!taskBudget || 'task_budget' in outputConfig || !shouldIncludeFirstPartyOnlyBetas()) {
    return
  }
  outputConfig.task_budget = {
    type: 'tokens',
    total: taskBudget.total,
    ...(taskBudget.remaining !== undefined && { remaining: taskBudget.remaining }),
  }
  if (!betas.includes(TASK_BUDGETS_BETA_HEADER)) {
    betas.push(TASK_BUDGETS_BETA_HEADER)
  }
}
```

**Task Budget Purpose** (EAP):
- Communicates total token budget for an agentic task
- Model paces output within the budget
- Caller decrements `remaining` across retries/tool calls
- Beta: `task-budgets-2026-03-13`

### 1.6 Cache Management

**Function**: `getCacheControl()` and `should1hCacheTTL()`, lines 358-434

#### 1.6.1 Cache Control Headers

```typescript
export function getCacheControl({
  scope,
  querySource,
}: {
  scope?: CacheScope
  querySource?: QuerySource
} = {}): {
  type: 'ephemeral'
  ttl?: '1h'
  scope?: CacheScope
} {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    ...(scope === 'global' && { scope }),
  }
}
```

**Ephemeral Cache** (5min default TTL):
- Type: `'ephemeral'` (not `'static'` — temporary cache for session)
- TTL: Defaults to 5 minutes
- Scope: `'global'` (shared across users) or omitted (user-scoped)

#### 1.6.2 1-Hour TTL Logic

```typescript
function should1hCacheTTL(querySource?: QuerySource): boolean {
  // 3P Bedrock users opt-in via env var
  if (getAPIProvider() === 'bedrock' && isEnvTruthy(process.env.ENABLE_PROMPT_CACHING_1H_BEDROCK)) {
    return true
  }

  // Latch eligibility in bootstrap state (session stability)
  let userEligible = getPromptCache1hEligible()
  if (userEligible === null) {
    userEligible = process.env.USER_TYPE === 'ant' || (isClaudeAISubscriber() && !currentLimits.isUsingOverage)
    setPromptCache1hEligible(userEligible)
  }
  if (!userEligible) return false

  // Cache allowlist in bootstrap state (prevents mid-session GrowthBook updates from breaking cache)
  let allowlist = getPromptCache1hAllowlist()
  if (allowlist === null) {
    const config = getFeatureValue_CACHED_MAY_BE_STALE<{ allowlist?: string[] }>('tengu_prompt_cache_1h_config', {})
    allowlist = config.allowlist ?? []
    setPromptCache1hAllowlist(allowlist)
  }

  // Pattern matching: 'repl_main_thread*' matches 'repl_main_thread' and 'repl_main_thread:outputStyle:Explanatory'
  return querySource !== undefined && allowlist.some(pattern =>
    pattern.endsWith('*')
      ? querySource.startsWith(pattern.slice(0, -1))
      : querySource === pattern,
  )
}
```

**1h TTL Eligibility**:
- **Ant users**: Always eligible
- **Claude AI subscribers**: Eligible unless using overage
- **Overage users**: Ineligible (session-latched to prevent mid-session flips)
- **GrowthBook allowlist**: Pattern-based (`repl_main_thread*`, `sdk`, `agent:*`, etc.)
- **Latching**: Both eligibility and allowlist are cached in bootstrap state ➜ stable even if GrowthBook updates mid-session

#### 1.6.3 Cache Break Detection

**See section 2.6 (promptCacheBreakDetection.ts)**

---

## Part 2: Multi-Provider Client Factory (client.ts)

### 2.1 Architecture

**File**: `client.ts` (389 lines)

The client factory supports four providers:

```
┌─ getAnthropicClient()
│
├─ Bedrock (AWS) ──────────→ AnthropicBedrock
│  ├─ AWS credentials (IAM, STS, service account)
│  ├─ Configurable region per model
│  └─ Bearer token auth (AWS_BEARER_TOKEN_BEDROCK)
│
├─ Foundry (Azure) ────────→ AnthropicFoundry
│  ├─ API key auth (ANTHROPIC_FOUNDRY_API_KEY)
│  └─ Azure AD (DefaultAzureCredential)
│
├─ Vertex (Google) ────────→ AnthropicVertex
│  ├─ GoogleAuth (service account, ADC, gcloud CLI)
│  ├─ Project ID fallback to avoid metadata server timeout
│  └─ Model-specific region config
│
└─ 1P API ─────────────────→ Anthropic
   ├─ API key auth (ANTHROPIC_API_KEY)
   └─ OAuth token auth (for Claude AI subscribers)
```

### 2.2 Client Configuration

**Lines**: 88-315

Common client args across all providers:

```typescript
const ARGS = {
  defaultHeaders: {
    'x-app': 'cli',
    'User-Agent': getUserAgent(),
    'X-Claude-Code-Session-Id': getSessionId(),
    ...customHeaders,  // ANTHROPIC_CUSTOM_HEADERS
    ...(containerId ? { 'x-claude-remote-container-id': containerId } : {}),
    ...(remoteSessionId ? { 'x-claude-remote-session-id': remoteSessionId } : {}),
    ...(clientApp ? { 'x-client-app': clientApp } : {}),
  },
  maxRetries: 0,  // Disabled (handled in withRetry)
  timeout: parseInt(process.env.API_TIMEOUT_MS || '600000', 10),  // 600s default
  dangerouslyAllowBrowser: true,  // Required for browser-compatible SDK
  fetchOptions: getProxyFetchOptions({ forAnthropicAPI: true }),
  ...(resolvedFetch && { fetch: resolvedFetch }),
}
```

**Headers**:
- `x-app: 'cli'` — Identifies requests from CLI
- `X-Claude-Code-Session-Id` — Session tracking
- `x-claude-remote-container-id` — For remote container identification
- `x-claude-remote-session-id` — For remote sessions
- `x-client-app` — SDK consumer identification
- `x-anthropic-additional-protection` — Optional security header

### 2.3 Bedrock Client (AWS)

**Lines**: 153-189

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
  const { AnthropicBedrock } = await import('@anthropic-ai/bedrock-sdk')

  // Region resolution for small fast model (Haiku)
  const awsRegion = model === getSmallFastModel() && process.env.ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION
    ? process.env.ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION
    : getAWSRegion()

  const bedrockArgs: ConstructorParameters<typeof AnthropicBedrock>[0] = {
    ...ARGS,
    awsRegion,
    ...(isEnvTruthy(process.env.CLAUDE_CODE_SKIP_BEDROCK_AUTH) && { skipAuth: true }),
    ...(isDebugToStdErr() && { logger: createStderrLogger() }),
  }

  // Bearer token auth (e.g., for API key-based access)
  if (process.env.AWS_BEARER_TOKEN_BEDROCK) {
    bedrockArgs.skipAuth = true
    bedrockArgs.defaultHeaders = {
      ...bedrockArgs.defaultHeaders,
      Authorization: `Bearer ${process.env.AWS_BEARER_TOKEN_BEDROCK}`,
    }
  } else if (!isEnvTruthy(process.env.CLAUDE_CODE_SKIP_BEDROCK_AUTH)) {
    // Refresh AWS credentials and extract
    const cachedCredentials = await refreshAndGetAwsCredentials()
    if (cachedCredentials) {
      bedrockArgs.awsAccessKey = cachedCredentials.accessKeyId
      bedrockArgs.awsSecretKey = cachedCredentials.secretAccessKey
      bedrockArgs.awsSessionToken = cachedCredentials.sessionToken
    }
  }

  return new AnthropicBedrock(bedrockArgs) as unknown as Anthropic
}
```

**Auth Flow**:
1. **Bearer token** (highest priority): `AWS_BEARER_TOKEN_BEDROCK` ➜ skip AWS SDK auth
2. **AWS credentials** (via SDK): IAM, STS, environment variables, credential files
3. **Credential refresh**: STS assume-role if configured, with session token support

**Region Config**:
- **Global**: `AWS_REGION` or `AWS_DEFAULT_REGION`
- **Per-model override**: `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` (Haiku)

### 2.4 Foundry Client (Azure)

**Lines**: 191-219

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)) {
  const { AnthropicFoundry } = await import('@anthropic-ai/foundry-sdk')

  let azureADTokenProvider: (() => Promise<string>) | undefined

  if (!process.env.ANTHROPIC_FOUNDRY_API_KEY) {
    if (isEnvTruthy(process.env.CLAUDE_CODE_SKIP_FOUNDRY_AUTH)) {
      // Mock provider (testing/proxy)
      azureADTokenProvider = () => Promise.resolve('')
    } else {
      // Real Azure AD auth
      const { DefaultAzureCredential, getBearerTokenProvider } = await import('@azure/identity')
      azureADTokenProvider = getBearerTokenProvider(
        new AzureCredential(),
        'https://cognitiveservices.azure.com/.default',
      )
    }
  }

  const foundryArgs: ConstructorParameters<typeof AnthropicFoundry>[0] = {
    ...ARGS,
    ...(azureADTokenProvider && { azureADTokenProvider }),
    ...(isDebugToStdErr() && { logger: createStderrLogger() }),
  }

  return new AnthropicFoundry(foundryArgs) as unknown as Anthropic
}
```

**Auth Options**:
1. **API key** (highest priority): `ANTHROPIC_FOUNDRY_API_KEY`
2. **Azure AD** (DefaultAzureCredential):
   - Environment variables (service principal)
   - Managed identity (Azure VMs, App Service)
   - Azure CLI token
   - Interactive browser login

**Resource Config**:
- `ANTHROPIC_FOUNDRY_RESOURCE`: Resource name (e.g., `my-resource`)
- `ANTHROPIC_FOUNDRY_BASE_URL`: Alternative full URL

### 2.5 Vertex Client (Google)

**Lines**: 221-297

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
  // Refresh GCP credentials if needed
  if (!isEnvTruthy(process.env.CLAUDE_CODE_SKIP_VERTEX_AUTH)) {
    await refreshGcpCredentialsIfNeeded()
  }

  const [{ AnthropicVertex }, { GoogleAuth }] = await Promise.all([
    import('@anthropic-ai/vertex-sdk'),
    import('google-auth-library'),
  ])

  // Prevent metadata server timeout by providing fallback projectId
  const hasProjectEnvVar = process.env['GCLOUD_PROJECT'] || process.env['GOOGLE_CLOUD_PROJECT']
  const hasKeyFile = process.env['GOOGLE_APPLICATION_CREDENTIALS']

  const googleAuth = isEnvTruthy(process.env.CLAUDE_CODE_SKIP_VERTEX_AUTH)
    ? ({
        getClient: () => ({ getRequestHeaders: () => ({}) }),
      } as unknown as GoogleAuth)
    : new GoogleAuth({
        scopes: ['https://www.googleapis.com/auth/cloud-platform'],
        // Only use projectId fallback if no env var or keyfile
        ...(hasProjectEnvVar || hasKeyFile
          ? {}
          : { projectId: process.env.ANTHROPIC_VERTEX_PROJECT_ID }),
      })

  const vertexArgs: ConstructorParameters<typeof AnthropicVertex>[0] = {
    ...ARGS,
    region: getVertexRegionForModel(model),
    googleAuth,
    ...(isDebugToStdErr() && { logger: createStderrLogger() }),
  }

  return new AnthropicVertex(vertexArgs) as unknown as Anthropic
}
```

**Region Resolution** (via `getVertexRegionForModel()`):
1. **Model-specific env var** (highest priority):
   - `VERTEX_REGION_CLAUDE_3_5_HAIKU`
   - `VERTEX_REGION_CLAUDE_HAIKU_4_5`
   - `VERTEX_REGION_CLAUDE_3_5_SONNET`
   - `VERTEX_REGION_CLAUDE_3_7_SONNET`
2. **Global region**: `CLOUD_ML_REGION`
3. **Default**: `us-east5`

**Auth Chain**:
1. **Service account** (JSON via `GOOGLE_APPLICATION_CREDENTIALS`)
2. **ADC** (Application Default Credentials) — implicit credential discovery
3. **Metadata server** (GCE, GKE, etc.)
4. **Gcloud CLI** token

**Metadata Server Timeout Mitigation**:
- Problem: Metadata server checks timeout at 12s if not on GCP
- Solution: Provide `projectId` fallback if user hasn't configured other discovery methods
- Effect: Skips metadata server lookup, uses provided projectId directly

### 2.6 First-Party API Client

**Lines**: 300-315

```typescript
const clientConfig: ConstructorParameters<typeof Anthropic>[0] = {
  apiKey: isClaudeAISubscriber() ? null : apiKey || getAnthropicApiKey(),
  authToken: isClaudeAISubscriber()
    ? getClaudeAIOAuthTokens()?.accessToken
    : undefined,
  // Staging OAuth base URL (ant-only)
  ...(process.env.USER_TYPE === 'ant' && isEnvTruthy(process.env.USE_STAGING_OAUTH)
    ? { baseURL: getOauthConfig().BASE_API_URL }
    : {}),
  ...ARGS,
  ...(isDebugToStdErr() && { logger: createStderrLogger() }),
}

return new Anthropic(clientConfig)
```

**Auth Modes**:
- **API key**: `ANTHROPIC_API_KEY` or via API key helper
- **OAuth token**: `getClaudeAIOAuthTokens()?.accessToken` (Claude AI subscribers)
- **Priority**: OAuth > API key

### 2.7 Custom Headers

**Function**: `getCustomHeaders()`, lines 330-354

Parses `ANTHROPIC_CUSTOM_HEADERS` env var:

```typescript
function getCustomHeaders(): Record<string, string> {
  const customHeaders: Record<string, string> = {}
  const customHeadersEnv = process.env.ANTHROPIC_CUSTOM_HEADERS

  if (!customHeadersEnv) return customHeaders

  // Parse "Header: Value" format, split by newlines
  const headerStrings = customHeadersEnv.split(/\n|\r\n/)

  for (const headerString of headerStrings) {
    if (!headerString.trim()) continue

    const colonIdx = headerString.indexOf(':')
    if (colonIdx === -1) continue

    const name = headerString.slice(0, colonIdx).trim()
    const value = headerString.slice(colonIdx + 1).trim()
    if (name) {
      customHeaders[name] = value
    }
  }

  return customHeaders
}
```

**Usage**: For proxy authentication, custom routing, analytics headers, etc.

### 2.8 Request ID Injection

**Function**: `buildFetch()`, lines 358-389

Generates client-side request IDs for timeout correlation:

```typescript
function buildFetch(fetchOverride: ClientOptions['fetch'], source: string | undefined): ClientOptions['fetch'] {
  const inner = fetchOverride ?? globalThis.fetch
  const injectClientRequestId = getAPIProvider() === 'firstParty' && isFirstPartyAnthropicBaseUrl()

  return (input, init) => {
    const headers = new Headers(init?.headers)

    // Generate client-side request ID (only 1P, not Bedrock/Vertex/Foundry)
    if (injectClientRequestId && !headers.has(CLIENT_REQUEST_ID_HEADER)) {
      headers.set(CLIENT_REQUEST_ID_HEADER, randomUUID())
    }

    try {
      const url = input instanceof Request ? input.url : String(input)
      const id = headers.get(CLIENT_REQUEST_ID_HEADER)
      logForDebugging(
        `[API REQUEST] ${new URL(url).pathname}${id ? ` ${CLIENT_REQUEST_ID_HEADER}=${id}` : ''} source=${source ?? 'unknown'}`,
      )
    } catch {
      // Never let logging crash the fetch
    }

    return inner(input, { ...init, headers })
  }
}

export const CLIENT_REQUEST_ID_HEADER = 'x-client-request-id'
```

**Purpose**: When streaming times out (no server request ID returned), the client-side ID correlates with server logs via correlation ID.

---

## Part 3: Retry Logic & Error Recovery (withRetry.ts)

### 3.1 Architecture

**File**: `withRetry.ts` (822 lines)

Core retry loop with selective backoff and model fallback:

```
withRetry()
  ├─ [Attempt 1..N] Operation
  ├─ [On Error] Classify + decide retry strategy
  │  ├─ Fast mode overload rejection → fallback to standard speed
  │  ├─ Fast mode rate limit (429) → short wait + retry or cooldown
  │  ├─ 401 (auth) → refresh credentials + retry
  │  ├─ 529 (overloaded) → exponential backoff or fallback model
  │  ├─ 429 (rate limit) → exponential backoff
  │  ├─ 500+ server errors → exponential backoff
  │  └─ Context overflow → reduce max_tokens + retry
  └─ [Return] Final result or CannotRetryError
```

### 3.2 Core Loop

**Lines**: 170-517

```typescript
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client: Anthropic, attempt: number, context: RetryContext) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T> {
  const maxRetries = getMaxRetries(options)
  const retryContext: RetryContext = {
    model: options.model,
    thinkingConfig: options.thinkingConfig,
    ...(isFastModeEnabled() && { fastMode: options.fastMode }),
  }

  let client: Anthropic | null = null
  let consecutive529Errors = options.initialConsecutive529Errors ?? 0
  let lastError: unknown
  let persistentAttempt = 0

  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    if (options.signal?.aborted) {
      throw new APIUserAbortError()
    }

    // Fast mode fallback: 429/529 handling
    const wasFastModeActive = isFastModeEnabled() && retryContext.fastMode && !isFastModeCooldown()

    try {
      // Mock rate limits (ant employees testing)
      if (process.env.USER_TYPE === 'ant') {
        const mockError = checkMockRateLimitError(retryContext.model, wasFastModeActive)
        if (mockError) throw mockError
      }

      // Refresh client on auth errors or stale connections
      const isStaleConnection = isStaleConnectionError(lastError)
      if (isStaleConnection && getFeatureValue_CACHED_MAY_BE_STALE('tengu_disable_keepalive_on_econnreset', false)) {
        disableKeepAlive()
      }

      if (
        client === null ||
        (lastError instanceof APIError && lastError.status === 401) ||
        isOAuthTokenRevokedError(lastError) ||
        isBedrockAuthError(lastError) ||
        isVertexAuthError(lastError) ||
        isStaleConnection
      ) {
        // Force OAuth token refresh on 401
        if ((lastError instanceof APIError && lastError.status === 401) || isOAuthTokenRevokedError(lastError)) {
          const failedAccessToken = getClaudeAIOAuthTokens()?.accessToken
          if (failedAccessToken) {
            await handleOAuth401Error(failedAccessToken)
          }
        }
        client = await getClient()
      }

      return await operation(client, attempt, retryContext)
    } catch (error) {
      lastError = error
      logForDebugging(
        `API error (attempt ${attempt}/${maxRetries + 1}): ${error instanceof APIError ? `${error.status} ${error.message}` : errorMessage(error)}`,
        { level: 'error' },
      )

      // [Error classification and retry decision logic — see 3.3 below]
      // ...
    }
  }

  throw new CannotRetryError(lastError, retryContext)
}
```

### 3.3 Error Classification & Retry Decisions

**Lines**: 254-514

#### 3.3.1 Fast Mode Fallback (429/529)

```typescript
if (wasFastModeActive && !isPersistentRetryEnabled() && error instanceof APIError && (error.status === 429 || is529Error(error))) {
  // Check if 429 is due to overage being disabled
  const overageReason = error.headers?.get('anthropic-ratelimit-unified-overage-disabled-reason')
  if (overageReason !== null && overageReason !== undefined) {
    handleFastModeOverageRejection(overageReason)
    retryContext.fastMode = false
    continue
  }

  const retryAfterMs = getRetryAfterMs(error)
  if (retryAfterMs !== null && retryAfterMs < SHORT_RETRY_THRESHOLD_MS) {
    // Short retry-after: wait and retry with fast mode still active
    await sleep(retryAfterMs, options.signal, { abortError })
    continue
  }

  // Long or unknown retry-after: cooldown (switch to standard speed)
  const cooldownMs = Math.max(
    retryAfterMs ?? DEFAULT_FAST_MODE_FALLBACK_HOLD_MS,
    MIN_COOLDOWN_MS,
  )
  const cooldownReason: CooldownReason = is529Error(error) ? 'overloaded' : 'rate_limit'
  triggerFastModeCooldown(Date.now() + cooldownMs, cooldownReason)
  if (isFastModeEnabled()) {
    retryContext.fastMode = false
  }
  continue
}
```

**Fast Mode Fallback Strategy**:
- **Short retry-after** (`<5s`): Wait and retry preserving cache (same model name)
- **Long retry-after** (`≥5s`): Cooldown ➜ switches model to standard speed ➜ cache key stable

#### 3.3.2 Fast Mode Rejection by API

```typescript
if (wasFastModeActive && isFastModeNotEnabledError(error)) {
  handleFastModeRejectedByAPI()
  retryContext.fastMode = false
  continue
}
```

If API returns error indicating fast mode is not enabled, permanently disable for session.

#### 3.3.3 Background Query Sources (No 529 Retry)

```typescript
// Non-foreground sources bail immediately on 529
if (is529Error(error) && !shouldRetry529(options.querySource)) {
  logEvent('tengu_api_529_background_dropped', { query_source: options.querySource })
  throw new CannotRetryError(error, retryContext)
}
```

**Foreground vs Background**:
- **Foreground** (user blocking): `repl_main_thread`, `sdk`, `agent:*`, `side_question`, `auto_mode`, `bash_classifier`
  - Retry 529: User waits for result
  - Each retry amplifies gateway traffic 3-10×
- **Background** (user doesn't wait): `summaries`, `titles`, `suggestions`, `compaction`, etc.
  - No retry on 529: User never sees these fail
  - Prevents cascading amplification during capacity event

#### 3.3.4 Consecutive 529 Errors & Model Fallback

```typescript
if (is529Error(error) && (process.env.FALLBACK_FOR_ALL_PRIMARY_MODELS || (!isClaudeAISubscriber() && isNonCustomOpusModel(options.model)))) {
  consecutive529Errors++
  if (consecutive529Errors >= MAX_529_RETRIES) {  // MAX_529_RETRIES = 3
    if (options.fallbackModel) {
      logEvent('tengu_api_opus_fallback_triggered', {
        original_model: options.model,
        fallback_model: options.fallbackModel,
        provider: getAPIProviderForStatsig(),
      })
      throw new FallbackTriggeredError(options.model, options.fallbackModel)
    }

    // External users (non-ant, non-subscriber) with no fallback
    if (process.env.USER_TYPE === 'external' && !process.env.IS_SANDBOX && !isPersistentRetryEnabled()) {
      logEvent('tengu_api_custom_529_overloaded_error', {})
      throw new CannotRetryError(new Error(REPEATED_529_ERROR_MESSAGE), retryContext)
    }
  }
}
```

**529 Strategy**:
- **Count consecutive 529s**: Max 3 before fallback
- **Fallback model**: Switch to smaller model (e.g., Sonnet) if configured
- **External users**: Cap at 3 consecutive 529s, fail with user-facing message
- **Ant/internal**: Persist indefinitely (with chunked backoff)

#### 3.3.5 Context Window Overflow

```typescript
const overflowData = parseMaxTokensContextOverflowError(error)
if (overflowData) {
  const { inputTokens, contextLimit } = overflowData
  const safetyBuffer = 1000
  const availableContext = Math.max(0, contextLimit - inputTokens - safetyBuffer)

  if (availableContext < FLOOR_OUTPUT_TOKENS) {  // FLOOR_OUTPUT_TOKENS = 3000
    throw error
  }

  const minRequired = (retryContext.thinkingConfig.type === 'enabled' ? retryContext.thinkingConfig.budgetTokens : 0) + 1
  const adjustedMaxTokens = Math.max(FLOOR_OUTPUT_TOKENS, availableContext, minRequired)
  retryContext.maxTokensOverride = adjustedMaxTokens

  logEvent('tengu_max_tokens_context_overflow_adjustment', {
    inputTokens,
    contextLimit,
    adjustedMaxTokens,
    attempt,
  })

  continue
}
```

**Overflow Handling**:
- Parse API error: `"input length and \`max_tokens\` exceed context limit: 188059 + 20000 > 200000"`
- Extract: inputTokens, maxTokens, contextLimit
- Calculate: `available = contextLimit - inputTokens - safetyBuffer(1000)`
- Set: `maxTokensOverride = max(3000, available, minThinking+1)`
- Retry: With reduced max_tokens

### 3.4 Exponential Backoff

**Function**: `getRetryDelay()`, lines 530-548

```typescript
export function getRetryDelay(
  attempt: number,
  retryAfterHeader?: string | null,
  maxDelayMs = 32000,
): number {
  if (retryAfterHeader) {
    const seconds = parseInt(retryAfterHeader, 10)
    if (!isNaN(seconds)) {
      return seconds * 1000
    }
  }

  const baseDelay = Math.min(
    BASE_DELAY_MS * Math.pow(2, attempt - 1),  // BASE_DELAY_MS = 500
    maxDelayMs,
  )
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

**Backoff Formula**:
- **Attempt 1**: 500ms + jitter(0-125ms) = 500-625ms
- **Attempt 2**: 1000ms + jitter(0-250ms) = 1000-1250ms
- **Attempt 3**: 2000ms + jitter(0-500ms) = 2000-2500ms
- **Attempt 4**: 4000ms + jitter(0-1000ms) = 4000-5000ms
- **Attempt 5**: 8000ms + jitter(0-2000ms) = 8000-10000ms
- **Attempt 6**: 16000ms + jitter(0-4000ms) = 16000-20000ms
- **Attempt 7+**: 32000ms (capped) + jitter = 32000-36000ms

**Jitter**: 25% of base delay, prevents thundering herd

### 3.5 Persistent Retry (Unattended Mode)

**Lines**: 100-104, 433-460

For ant-only unattended sessions (`CLAUDE_CODE_UNATTENDED_RETRY=true`):

```typescript
function isPersistentRetryEnabled(): boolean {
  return feature('UNATTENDED_RETRY') && isEnvTruthy(process.env.CLAUDE_CODE_UNATTENDED_RETRY)
}

// [In error catch:]
const persistent = isPersistentRetryEnabled() && isTransientCapacityError(error)
if (attempt > maxRetries && !persistent) {
  throw new CannotRetryError(error, retryContext)
}

// [Persistent loop:]
let remaining = delayMs
while (remaining > 0) {
  if (options.signal?.aborted) throw new APIUserAbortError()
  if (error instanceof APIError) {
    yield createSystemAPIErrorMessage(error, remaining, reportedAttempt, maxRetries)
  }
  const chunk = Math.min(remaining, HEARTBEAT_INTERVAL_MS)  // HEARTBEAT_INTERVAL_MS = 30_000
  await sleep(chunk, options.signal, { abortError })
  remaining -= chunk
}
// Clamp so loop never terminates (attempt stays capped, persistentAttempt grows)
if (attempt >= maxRetries) attempt = maxRetries
```

**Persistent Retry Logic**:
- **Max backoff**: 5 minutes (PERSISTENT_MAX_BACKOFF_MS)
- **Reset cap**: 6 hours (PERSISTENT_RESET_CAP_MS) — stops waiting even if retry-after is huge
- **Chunked waits**: 30s chunks, each `yield`ed as `SystemAPIErrorMessage` for keep-alive
- **Infinite loop**: for-loop attempt clamped at maxRetries, persistentAttempt grows unbounded

### 3.6 Error Classes

**Lines**: 144-168

```typescript
export class CannotRetryError extends Error {
  constructor(
    public readonly originalError: unknown,
    public readonly retryContext: RetryContext,
  ) {
    // Preserve original stack trace
  }
}

export class FallbackTriggeredError extends Error {
  constructor(
    public readonly originalModel: string,
    public readonly fallbackModel: string,
  ) {
    // Signals: switch to fallbackModel and retry
  }
}
```

### 3.7 Helpers

#### 3.7.1 OAuth Token Refresh

```typescript
const isOAuthTokenRevokedError = (error: unknown): boolean =>
  error instanceof APIError &&
  error.status === 403 &&
  error.message?.includes('OAuth token has been revoked')
```

#### 3.7.2 Stale Connection Detection

```typescript
function isStaleConnectionError(error: unknown): boolean {
  if (!(error instanceof APIConnectionError)) return false
  const details = extractConnectionErrorDetails(error)
  return details?.code === 'ECONNRESET' || details?.code === 'EPIPE'
}
```

#### 3.7.3 Fast Mode Not Enabled

```typescript
function isFastModeNotEnabledError(error: unknown): boolean {
  return error instanceof APIError && error.message?.includes('speed parameter is not supported')
}
```

---

## Part 4: Error Classification & Handling (errors.ts)

### 4.1 Error Classification

**File**: `errors.ts` (1,207 lines)

Central repository for error messages and classifications.

**Key Error Messages** (constants):

```typescript
export const API_ERROR_MESSAGE_PREFIX = 'API Error'
export const PROMPT_TOO_LONG_ERROR_MESSAGE = 'Prompt is too long'
export const CREDIT_BALANCE_TOO_LOW_ERROR_MESSAGE = 'Credit balance is too low'
export const INVALID_API_KEY_ERROR_MESSAGE = 'Not logged in · Please run /login'
export const INVALID_API_KEY_ERROR_MESSAGE_EXTERNAL = 'Invalid API key · Fix external API key'
export const ORG_DISABLED_ERROR_MESSAGE_ENV_KEY_WITH_OAUTH = 'Your ANTHROPIC_API_KEY belongs to a disabled organization · Unset the environment variable to use your subscription instead'
export const ORG_DISABLED_ERROR_MESSAGE_ENV_KEY = 'Your ANTHROPIC_API_KEY belongs to a disabled organization · Update or unset the environment variable'
export const TOKEN_REVOKED_ERROR_MESSAGE = 'OAuth token revoked · Please run /login'
export const CCR_AUTH_ERROR_MESSAGE = 'Authentication error · This may be a temporary network issue, please try again'
export const REPEATED_529_ERROR_MESSAGE = 'Repeated 529 Overloaded errors'
export const CUSTOM_OFF_SWITCH_MESSAGE = 'Opus is experiencing high load, please use /model to switch to Sonnet'
export const API_TIMEOUT_ERROR_MESSAGE = 'Request timed out'
```

### 4.2 Media Size Error Detection

**Function**: `isMediaSizeError()`, lines 133-139

```typescript
export function isMediaSizeError(raw: string): boolean {
  return (
    (raw.includes('image exceeds') && raw.includes('maximum')) ||
    (raw.includes('image dimensions exceed') && raw.includes('many-image')) ||
    /maximum of \d+ PDF pages/.test(raw)
  )
}
```

Used by reactive compact to decide whether to strip media and retry.

### 4.3 Prompt Too Long Parsing

**Function**: `parsePromptTooLongTokenCounts()`, lines 85-96

```typescript
export function parsePromptTooLongTokenCounts(rawMessage: string): {
  actualTokens: number | undefined
  limitTokens: number | undefined
} {
  const match = rawMessage.match(/prompt is too long[^0-9]*(\d+)\s*tokens?\s*>\s*(\d+)/i)
  return {
    actualTokens: match ? parseInt(match[1]!, 10) : undefined,
    limitTokens: match ? parseInt(match[2]!, 10) : undefined,
  }
}
```

Extracts token counts from API error string like "prompt is too long: 137500 tokens > 135000 maximum".

---

## Part 5: Session Persistence (sessionIngress.ts)

### 5.1 Architecture

**File**: `sessionIngress.ts` (514 lines)

Handles transcript persistence via session ingress API with:
- **Idempotent append** via UUID chain
- **409 conflict handling** with server state adoption
- **Per-session sequential execution** (no concurrent writes)

### 5.2 Core Append Logic

**Function**: `appendSessionLog()`, lines 193-212

```typescript
export async function appendSessionLog(
  sessionId: string,
  entry: TranscriptMessage,
  url: string,
): Promise<boolean> {
  const sessionToken = getSessionIngressAuthToken()
  if (!sessionToken) {
    logForDiagnosticsNoPII('error', 'session_persist_fail_jwt_no_token')
    return false
  }

  const headers: Record<string, string> = {
    Authorization: `Bearer ${sessionToken}`,
    'Content-Type': 'application/json',
  }

  const sequentialAppend = getOrCreateSequentialAppend(sessionId)
  return sequentialAppend(entry, url, headers)  // Sequential wrapper
}
```

**Flow**:
1. Get session JWT token (from session-ingress auth module)
2. Get or create sequential wrapper for the session (prevents concurrent writes)
3. Call sequential wrapper with entry, URL, headers

### 5.3 Retry with Idempotency

**Function**: `appendSessionLogImpl()`, lines 63-186

```typescript
async function appendSessionLogImpl(
  sessionId: string,
  entry: TranscriptMessage,
  url: string,
  headers: Record<string, string>,
): Promise<boolean> {
  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {  // MAX_RETRIES = 10
    try {
      const lastUuid = lastUuidMap.get(sessionId)
      const requestHeaders = { ...headers }
      if (lastUuid) {
        requestHeaders['Last-Uuid'] = lastUuid  // Optimistic concurrency control
      }

      const response = await axios.put(url, entry, {
        headers: requestHeaders,
        validateStatus: status => status < 500,
      })

      if (response.status === 200 || response.status === 201) {
        lastUuidMap.set(sessionId, entry.uuid)  // Record last successful UUID
        return true
      }

      if (response.status === 409) {
        // Concurrent modification detected
        const serverLastUuid = response.headers['x-last-uuid']
        if (serverLastUuid === entry.uuid) {
          // Our entry IS on server already (recovered from stale state)
          lastUuidMap.set(sessionId, entry.uuid)
          logForDiagnosticsNoPII('info', 'session_persist_recovered_from_409')
          return true
        }

        // Server has a different last UUID — adopt it and retry
        if (serverLastUuid) {
          lastUuidMap.set(sessionId, serverLastUuid as UUID)
          logForDebugging(`Session 409: adopting serverLastUuid=${serverLastUuid}, retrying entry ${entry.uuid}`)
        } else {
          // Re-fetch session to discover current head
          const logs = await fetchSessionLogsFromUrl(sessionId, url, headers)
          const adoptedUuid = findLastUuid(logs)
          if (adoptedUuid) {
            lastUuidMap.set(sessionId, adoptedUuid)
          } else {
            // Can't determine server state — give up
            logError(new Error(`Session persistence conflict for ${sessionId}, entry ${entry.uuid}`))
            logForDiagnosticsNoPII('error', 'session_persist_fail_concurrent_modification')
            return false
          }
        }
        logForDiagnosticsNoPII('info', 'session_persist_409_adopt_server_uuid')
        continue  // Retry with updated lastUuid
      }

      if (response.status === 401) {
        logForDiagnosticsNoPII('error', 'session_persist_fail_bad_token')
        return false  // Non-retryable
      }

      // Other 4xx/5xx — retryable
      logForDiagnosticsNoPII('error', 'session_persist_fail_status', { status: response.status, attempt })
    } catch (error) {
      // Network errors — retryable
      logForDiagnosticsNoPII('error', 'session_persist_fail_status', { status: (error as AxiosError).status, attempt })
    }

    if (attempt === MAX_RETRIES) {
      logForDiagnosticsNoPII('error', 'session_persist_error_retries_exhausted', { attempt })
      return false
    }

    const delayMs = Math.min(BASE_DELAY_MS * Math.pow(2, attempt - 1), 8000)
    await sleep(delayMs)
  }

  return false
}
```

**Idempotency Protocol**:
- **Request header**: `Last-Uuid: {lastUuid}` — tell server "I know up to this UUID"
- **Response header**: `x-last-uuid: {currentUuid}` — server returns its current head
- **On 409**: Adopt server's UUID, retry with updated header
- **Semantic**: "Last-Uuid mismatch" means another writer advanced the chain

### 5.4 Session Fetching

**Function**: `getTeleportEvents()`, lines 291-402

For CCR v2, fetches paginated worker events:

```typescript
export async function getTeleportEvents(
  sessionId: string,
  accessToken: string,
  orgUUID: string,
): Promise<Entry[] | null> {
  const baseUrl = `${getOauthConfig().BASE_API_URL}/v1/code/sessions/${sessionId}/teleport-events`
  const headers = {
    ...getOAuthHeaders(accessToken),
    'x-organization-uuid': orgUUID,
  }

  const all: Entry[] = []
  let cursor: string | undefined
  let pages = 0
  const maxPages = 100

  while (pages < maxPages) {
    const params: Record<string, string | number> = { limit: 1000 }
    if (cursor !== undefined) {
      params.cursor = cursor
    }

    let response
    try {
      response = await axios.get<TeleportEventsResponse>(baseUrl, {
        headers,
        params,
        timeout: 20000,
        validateStatus: status => status < 500,
      })
    } catch (e) {
      logForDiagnosticsNoPII('error', 'teleport_events_fetch_fail')
      return null
    }

    if (response.status === 404) {
      // 404 on page 0: Session not found
      // 404 mid-pagination: Session deleted between pages — return what we have
      return pages === 0 ? null : all
    }

    if (response.status === 401) {
      throw new Error('Your session has expired. Please run /login to sign in again.')
    }

    if (response.status !== 200) {
      return null
    }

    const { data, next_cursor } = response.data
    for (const ev of data) {
      if (ev.payload !== null) {
        all.push(ev.payload)
      }
    }

    pages++
    if (next_cursor == null) break  // == covers both null and undefined
    cursor = next_cursor
  }

  return all
}
```

**Pagination**:
- **Per-page**: 1000 items (server max)
- **Cursor-based**: Opaque cursor echoed until unset
- **Max pages**: 100 (guard against infinite loops)
- **Payload extraction**: `data[i].payload` is the Entry

---

## Part 6: Prompt Cache Break Detection (promptCacheBreakDetection.ts)

### 6.1 Architecture

**File**: `promptCacheBreakDetection.ts` (727 lines)

Two-phase cache break detection:

```
Phase 1: recordPromptState()
  ├─ Compute hashes of system, tools, betas, effort, etc.
  ├─ Diff against last recorded state
  └─ Store pending changes for phase 2

Phase 2: [After API response received]
  ├─ Check if cache_read_input_tokens dropped unexpectedly
  ├─ Correlate drop with pending changes
  └─ Log tengu_cache_break_detected event with diagnostics
```

### 6.2 State Tracking

**Type**: `PreviousState`, lines 28-69

```typescript
type PreviousState = {
  systemHash: number
  toolsHash: number
  cacheControlHash: number  // Catches scope/TTL flips
  toolNames: string[]
  perToolHashes: Record<string, number>  // Per-tool schema hash
  systemCharCount: number
  model: string
  fastMode: boolean
  globalCacheStrategy: string  // 'tool_based' | 'system_prompt' | 'none'
  betas: string[]
  autoModeActive: boolean  // AFK_MODE_BETA_HEADER presence
  isUsingOverage: boolean
  cachedMCEnabled: boolean
  effortValue: string
  extraBodyHash: number  // CLAUDE_CODE_EXTRA_BODY changes
  callCount: number
  pendingChanges: PendingChanges | null
  prevCacheReadTokens: number | null
  cacheDeletionsPending: boolean
  buildDiffableContent: () => string
}
```

### 6.3 Phase 1: Record State

**Function**: `recordPromptState()`, lines 247-343

```typescript
export function recordPromptState(snapshot: PromptStateSnapshot): void {
  try {
    const key = getTrackingKey(querySource, agentId)
    if (!key) return  // Untracked source

    // Compute hashes
    const strippedSystem = stripCacheControl(system)
    const strippedTools = stripCacheControl(toolSchemas)
    const systemHash = computeHash(strippedSystem)
    const toolsHash = computeHash(strippedTools)
    const cacheControlHash = computeHash(system.map(b => ('cache_control' in b ? b.cache_control : null)))

    const toolNames = toolSchemas.map(t => ('name' in t ? t.name : 'unknown'))
    const systemCharCount = getSystemCharCount(system)

    const prev = previousStateBySource.get(key)

    if (!prev) {
      // First call: just record state
      const perToolHashes = computePerToolHashes(strippedTools, toolNames)
      const sortedBetas = [...betas].sort()

      previousStateBySource.set(key, {
        systemHash,
        toolsHash,
        cacheControlHash,
        toolNames,
        perToolHashes,
        systemCharCount,
        model,
        fastMode: fastMode ?? false,
        globalCacheStrategy,
        betas: sortedBetas,
        autoModeActive,
        isUsingOverage,
        cachedMCEnabled,
        effortValue: effortStr,
        extraBodyHash,
        callCount: 1,
        pendingChanges: null,
        prevCacheReadTokens: null,
        cacheDeletionsPending: false,
        buildDiffableContent: lazyDiffableContent,
      })
      return
    }

    // Subsequent calls: detect changes
    const changes: PendingChanges = {
      systemPromptChanged: systemHash !== prev.systemHash,
      toolSchemasChanged: toolsHash !== prev.toolsHash,
      modelChanged: model !== prev.model,
      fastModeChanged: fastMode !== prev.fastMode,
      cacheControlChanged: cacheControlHash !== prev.cacheControlHash,
      globalCacheStrategyChanged: globalCacheStrategy !== prev.globalCacheStrategy,
      betasChanged: !isEqual(sortedBetas, prev.betas),
      autoModeChanged: autoModeActive !== prev.autoModeActive,
      overageChanged: isUsingOverage !== prev.isUsingOverage,
      cachedMCChanged: cachedMCEnabled !== prev.cachedMCEnabled,
      effortChanged: effortStr !== prev.effortValue,
      extraBodyChanged: extraBodyHash !== prev.extraBodyHash,
      addedToolCount: 0,
      removedToolCount: 0,
      systemCharDelta: systemCharCount - prev.systemCharCount,
      addedTools: [],
      removedTools: [],
      changedToolSchemas: [],
      previousModel: prev.model,
      newModel: model,
      prevGlobalCacheStrategy: prev.globalCacheStrategy,
      newGlobalCacheStrategy: globalCacheStrategy,
      addedBetas: sortedBetas.filter(b => !prev.betas.includes(b)),
      removedBetas: prev.betas.filter(b => !sortedBetas.includes(b)),
      prevEffortValue: prev.effortValue,
      newEffortValue: effortStr,
      buildPrevDiffableContent: prev.buildDiffableContent,
    }

    // Detect tool additions/removals
    const prevToolSet = new Set(prev.toolNames)
    const currToolSet = new Set(toolNames)

    for (const tool of currToolSet) {
      if (!prevToolSet.has(tool)) {
        changes.addedTools.push(tool)
        changes.addedToolCount++
      }
    }

    for (const tool of prevToolSet) {
      if (!currToolSet.has(tool)) {
        changes.removedTools.push(tool)
        changes.removedToolCount++
      }
    }

    // Detect tool schema changes (per-tool hashing)
    if (changes.toolSchemasChanged && changes.addedToolCount === 0 && changes.removedToolCount === 0) {
      // Tool list unchanged but schemas changed — find which ones
      const currPerToolHashes = computePerToolHashes(strippedTools, toolNames)
      for (const toolName of toolNames) {
        if (currPerToolHashes[toolName] !== prev.perToolHashes[toolName]) {
          changes.changedToolSchemas.push(toolName)
        }
      }
    }

    prev.pendingChanges = changes
    prev.callCount++
    prev.buildDiffableContent = lazyDiffableContent
  } catch (error) {
    logError(error)
  }
}
```

### 6.4 Phase 2: Detect Breaks

**Function**: `checkResponseForCacheBreak()`, lines 370-528

Called after API response with usage data:

```typescript
export function checkResponseForCacheBreak({
  usage,
  model,
  querySource,
  agentId,
}: {
  usage: BetaUsage | undefined
  model: string
  querySource: QuerySource
  agentId?: AgentId
}): void {
  if (isExcludedModel(model)) return  // Haiku has different caching behavior

  const key = getTrackingKey(querySource, agentId)
  if (!key) return

  const prev = previousStateBySource.get(key)
  if (!prev || !prev.pendingChanges) return

  const changes = prev.pendingChanges
  const currCacheReadTokens = usage?.cache_read_input_tokens ?? 0
  const cacheReadDelta = currCacheReadTokens - (prev.prevCacheReadTokens ?? 0)

  // Cache break detection: token drop without system changes
  const isExpectedBreak = changes.systemPromptChanged || changes.toolSchemasChanged ||
                         changes.betasChanged || changes.cacheControlChanged ||
                         changes.modelChanged || changes.fastModeChanged ||
                         changes.globalCacheStrategyChanged || changes.extraBodyChanged

  const minTokenDropToFlag = MIN_CACHE_MISS_TOKENS  // 2000
  const isTokenDropSuspicious = cacheReadDelta < -minTokenDropToFlag && !isExpectedBreak

  // TTL-based break: likely cache expiration, not a client issue
  const lastCompletion = getLastApiCompletionTimestamp()
  const timeSinceLastCompletion = lastCompletion ? Date.now() - lastCompletion : 0
  const isLikelyTTLExpiration = timeSinceLastCompletion > CACHE_TTL_1HOUR_MS

  if (isTokenDropSuspicious && !isLikelyTTLExpiration) {
    // Log diagnostics
    try {
      const breakDiffPath = getCacheBreakDiffPath()
      const prevDiffableContent = changes.buildPrevDiffableContent()
      const currDiffableContent = prev.buildDiffableContent()
      const diff = createPatch('prev', prevDiffableContent, currDiffableContent, 'previous', 'current')

      await mkdir(dirname(breakDiffPath), { recursive: true })
      await writeFile(breakDiffPath, diff, 'utf-8')

      logEvent('tengu_cache_break_detected', {
        model,
        querySource,
        cacheReadDelta,
        prevCacheReadTokens: prev.prevCacheReadTokens ?? 0,
        currCacheReadTokens,
        callCount: prev.callCount,
        systemPromptChanged: changes.systemPromptChanged,
        toolSchemasChanged: changes.toolSchemasChanged,
        modelChanged: changes.modelChanged,
        betasChanged: changes.betasChanged,
        breakDiffPath: breakDiffPath as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS,
      })
    } catch (error) {
      logError(error)
    }
  }

  // Update for next call
  prev.prevCacheReadTokens = currCacheReadTokens
  prev.cacheDeletionsPending = false
  prev.pendingChanges = null
}
```

**Decision Tree**:
1. **Is token drop > 2000 tokens?** ➜ Potentially suspicious
2. **Did client-side state change?** (system, tools, betas, etc.) ➜ Expected, not a break
3. **Is it TTL expiration?** (>1 hour since last completion) ➜ Expected, not a break
4. **No explanation?** ➜ Log cache break event with diff patch

---

## Part 7: Logging & Telemetry (logging.ts)

### 7.1 Query Logging

**Function**: `logAPIQuery()`, lines 171-233

```typescript
export function logAPIQuery({
  model,
  messagesLength,
  temperature,
  betas,
  permissionMode,
  querySource,
  queryTracking,
  thinkingType,
  effortValue,
  fastMode,
  previousRequestId,
}: {
  model: string
  messagesLength: number
  temperature: number
  betas?: string[]
  permissionMode?: PermissionMode
  querySource: string
  queryTracking?: QueryChainTracking
  thinkingType?: 'adaptive' | 'enabled' | 'disabled'
  effortValue?: EffortLevel | null
  fastMode?: boolean
  previousRequestId?: string | null
}): void {
  logEvent('tengu_api_query', {
    model,
    messagesLength,
    temperature,
    provider: getAPIProviderForStatsig(),
    buildAgeMins: getBuildAgeMinutes(),
    ...(betas?.length && { betas: betas.join(',') }),
    permissionMode,
    querySource,
    ...(queryTracking && { queryChainId: queryTracking.chainId, queryDepth: queryTracking.depth }),
    thinkingType,
    effortValue,
    fastMode,
    ...(previousRequestId && { previousRequestId }),
    ...getAnthropicEnvMetadata(),
  })
}
```

### 7.2 Error Logging

**Function**: `logAPIError()`, lines 235-396

```typescript
export function logAPIError({
  error,
  model,
  messageCount,
  messageTokens,
  durationMs,
  durationMsIncludingRetries,
  attempt,
  requestId,
  clientRequestId,
  didFallBackToNonStreaming,
  promptCategory,
  headers,
  queryTracking,
  querySource,
  llmSpan,
  fastMode,
  previousRequestId,
}: {...}): void {
  const gateway = detectGateway({ headers, baseUrl: process.env.ANTHROPIC_BASE_URL })
  const errStr = getErrorMessage(error)
  const status = error instanceof APIError ? String(error.status) : undefined
  const errorType = classifyAPIError(error)
  const connectionDetails = extractConnectionErrorDetails(error)

  logEvent('tengu_api_error', {
    model,
    error: errStr,
    status,
    errorType,
    messageCount,
    messageTokens,
    durationMs,
    durationMsIncludingRetries,
    attempt,
    provider: getAPIProviderForStatsig(),
    requestId,
    clientRequestId,
    didFallBackToNonStreaming,
    ...(gateway && { gateway }),
    ...(queryTracking && { queryChainId: queryTracking.chainId, queryDepth: queryTracking.depth }),
    ...(querySource && { querySource }),
    fastMode,
    ...(previousRequestId && { previousRequestId }),
    ...getAnthropicEnvMetadata(),
  })

  // Log to OpenTelemetry
  void logOTelEvent('api_error', {
    model,
    error: errStr,
    status_code: String(status),
    duration_ms: String(durationMs),
    attempt: String(attempt),
    speed: fastMode ? 'fast' : 'normal',
  })

  // End tracing span
  endLLMRequestSpan(llmSpan, {
    success: false,
    statusCode: status ? parseInt(status) : undefined,
    error: errStr,
    attempt,
  })
}
```

### 7.3 Success Logging

**Function**: `logAPISuccess()`, lines 398-579

Logs cache hit rate, token counts, duration, cost, etc.

---

## Part 8: Additional Services

### 8.1 Grove (grove.ts)

Settings/notifications service for privacy preferences:

```typescript
export type GroveConfig = {
  grove_enabled: boolean
  domain_excluded: boolean
  notice_is_grace_period: boolean
  notice_reminder_frequency: number | null
}

export async function getGroveSettings(): Promise<ApiResult<AccountSettings>>
export async function updateGroveSettings(groveEnabled: boolean): Promise<void>
export async function isQualifiedForGrove(): Promise<boolean>
```

**Cache**: 24 hours (GROVE_CACHE_EXPIRATION_MS)

### 8.2 Bootstrap (bootstrap.ts)

Fetches client config on startup:

```typescript
async function fetchBootstrapAPI(): Promise<BootstrapResponse | null>
export async function fetchBootstrapData(): Promise<void>
```

Returns: `client_data`, `additional_model_options` (cached to disk)

### 8.3 Files API (filesApi.ts)

Downloads/uploads files with retry and progress tracking:

```typescript
export async function downloadFile(fileId: string, config: FilesApiConfig): Promise<Buffer>
export async function uploadFile(filePath: string, relativePath: string, config: FilesApiConfig): Promise<UploadResult>
export async function downloadSessionFiles(files: File[], config: FilesApiConfig, concurrency?: number): Promise<DownloadResult[]>
```

**Limits**:
- Max file size: 500MB
- Concurrency: 5 parallel downloads
- Retry: 3 attempts with exponential backoff
- Timeout: 60s per download

### 8.4 Error Utilities (errorUtils.ts)

SSL/TLS error extraction:

```typescript
export function extractConnectionErrorDetails(error: unknown): ConnectionErrorDetails | null
export function getSSLErrorHint(error: unknown): string | null
export function formatAPIError(error: APIError): string
```

Walks error cause chain to find SSL error codes, maps to user-friendly messages.

---

## Part 9: Constants & Thresholds

### 9.1 Timing

| Constant | Value | Purpose |
|----------|-------|---------|
| `BASE_DELAY_MS` (withRetry) | 500ms | Initial retry backoff |
| `MAX_NON_STREAMING_TOKENS` | 4000 | Max output tokens for non-streaming fallback |
| `STREAM_IDLE_TIMEOUT_MS` | 90s | Idle watchdog timeout (configurable) |
| `STALL_THRESHOLD_MS` | 30s | Stall detection gap |
| `CACHE_TTL_5MIN_MS` | 5min | Prompt cache default TTL |
| `CACHE_TTL_1HOUR_MS` | 1h | Prompt cache 1h TTL |
| `SHORT_RETRY_THRESHOLD_MS` | 5s | Fast mode: short vs long retry-after |
| `MIN_COOLDOWN_MS` | 30s | Fast mode: minimum cooldown |
| `DEFAULT_FAST_MODE_FALLBACK_HOLD_MS` | 60s | Fast mode fallback hold duration |
| `PERSISTENT_MAX_BACKOFF_MS` | 5min | Persistent retry max backoff |
| `PERSISTENT_RESET_CAP_MS` | 6h | Persistent retry absolute cap |
| `HEARTBEAT_INTERVAL_MS` | 30s | Keep-alive yield interval |

### 9.2 Retry Counts

| Constant | Value | Purpose |
|----------|-------|---------|
| `DEFAULT_MAX_RETRIES` | 10 | Max standard retry attempts |
| `MAX_RETRIES` (files) | 3 | File upload/download retry limit |
| `MAX_529_RETRIES` | 3 | Consecutive 529 before fallback |

### 9.3 Media & Content Limits

| Constant | Value | Purpose |
|----------|-------|---------|
| `API_MAX_MEDIA_PER_REQUEST` | 100 | Max images/documents per request |
| `MAX_FILE_SIZE_BYTES` (files) | 500MB | Max file upload size |
| `API_PDF_MAX_PAGES` | 50 | Max PDF pages |
| `PDF_TARGET_RAW_SIZE` | 1MB | Target decompressed PDF size |

### 9.4 Cache & Session

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_TRACKED_SOURCES` | 10 | Cache break detection sources to track |
| `MIN_CACHE_MISS_TOKENS` | 2000 | Min token drop to flag as cache break |
| `GROVE_CACHE_EXPIRATION_MS` | 24h | Grove settings cache lifetime |
| `CAPPED_DEFAULT_MAX_TOKENS` | — | Model-specific output token cap |

---

## Part 10: Request/Response Flow Diagram

```
User Request
    ↓
queryModel() [async generator]
    ├─ [Pre-call]
    │  ├─ computeFingerprintFromMessages()
    │  ├─ getMergedBetas()
    │  ├─ normalizeMessagesForAPI()
    │  ├─ buildSystemPromptBlocks()
    │  ├─ toolToAPISchema() × toolCount
    │  └─ recordPromptState() [cache break detection phase 1]
    │
    ├─ [Streaming Attempt]
    │  ├─ withRetry()
    │  │  ├─ getAnthropicClient() [multi-provider factory]
    │  │  ├─ anthropic.beta.messages.create({ stream: true })
    │  │  └─ [Retry logic on 401/429/529/timeouts]
    │  │
    │  └─ Stream Event Loop [1940-2114]
    │     ├─ message_start → partialMessage, TTFB
    │     ├─ content_block_start → initialize block
    │     ├─ content_block_delta → accumulate (JSON delta, text, thinking)
    │     ├─ message_delta → update usage
    │     └─ message_stop → finalize
    │
    ├─ [On Stream Timeout/Error]
    │  └─ executeNonStreamingRequest()
    │     ├─ withRetry() with fallback model
    │     └─ anthropic.beta.messages.create() [non-streaming]
    │
    └─ [Post-call]
       ├─ checkResponseForCacheBreak() [phase 2]
       ├─ calculateUSDCost()
       ├─ addToTotalSessionCost()
       ├─ logAPIQuery()
       ├─ logAPISuccess() [or logAPIError()]
       └─ setLastApiCompletionTimestamp()

Yields: StreamEvent | AssistantMessage | SystemAPIErrorMessage
```

---

## Part 11: Security Considerations

### 11.1 Token Handling

- **OAuth tokens**: Refreshed on 401 via `handleOAuth401Error()`
- **API keys**: Never logged, passed only to SDK
- **Bearer token (Bedrock)**: Passed in Authorization header via AWS_BEARER_TOKEN_BEDROCK env var
- **Client request ID**: First-party only, not sent to 3P providers

### 11.2 Credential Refresh

- **Bedrock**: AWS STS credential refresh with session token support
- **Vertex**: GCP credential refresh via `refreshGcpCredentialsIfNeeded()`
- **Foundry**: Azure AD token provider handles refresh
- **OAuth**: Session JWT token passed in Authorization header

### 11.3 Request Signing

- **1P API**: SDK handles signing via API key or OAuth token
- **Bedrock**: SDK signs requests with AWS SigV4
- **Vertex**: SDK signs requests with Google OAuth bearer token
- **Foundry**: SDK signs requests with Azure AD token

### 11.4 Error Message Sanitization

- **HTML stripping**: CloudFlare error pages sanitized (extracting title only)
- **Error cause chain**: Limited to 5 levels to prevent infinite loops
- **SSL error extraction**: Walks cause chain to find root error code

### 11.5 Header Injection Prevention

- Custom headers parsed from `ANTHROPIC_CUSTOM_HEADERS` env var (no risk of injection)
- Client request ID generated via `randomUUID()` (collision-resistant)
- Session headers (session ID, container ID) set by system, not user

---

## Conclusion

Claude Code's API client subsystem is a production-grade system combining:

1. **Robust request orchestration** via streaming with fallback to non-streaming
2. **Multi-provider support** with per-provider credential management
3. **Intelligent retry logic** with exponential backoff, circuit breakers, and model fallback
4. **Session-stable caching** via latched beta headers and eligibility latching
5. **Diagnostic instrumentation** via cache-break detection, connection error extraction, and detailed telemetry
6. **Idempotent persistence** via UUID-based transcript ingress with 409 conflict resolution

This architecture demonstrates careful engineering for resilience, observability, and performance at scale.

