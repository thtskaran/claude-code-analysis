# Prompt Suggestion & Speculative Execution: Deep Reverse Engineering

## PART 1 - PROMPT SUGGESTION ENGINE

### 1. Architecture: Post-Sampling Hook with Forked Agent

The prompt suggestion system is a **post-sampling hook** that executes after Claude generates a response. It runs via `executePromptSuggestion()` triggered from `REPLHookContext` with `querySource: 'repl_main_thread'`.

**Key Design Principle:** The suggestion engine runs a **forked Claude agent** (independent, parallel) that reuses the parent's prompt cache to generate suggestions cheaply. This is NOT a post-processing heuristic or template-based system — it's a lightweight language model call.

**Entry Point:**
```
executePromptSuggestion(context: REPLHookContext) →
  tryGenerateSuggestion(...) →
  generateSuggestion(...) →
  runForkedAgent(...)
```

The hook is designed to be **non-blocking**. After suggestion generation, if speculation is enabled, `startSpeculation()` is invoked in the background.

---

### 2. Generation Pipeline: `runForkedAgent` with Cache Reuse (~60-70% Hit Rate)

The suggestion pipeline reuses the parent conversation's prompt cache to achieve roughly **60-70% cache hit rates** on the fork call.

**Cache Architecture:**
- **Parent request**: Processes full conversation, builds prompt cache (cached_input_tokens, cache_creation_input_tokens)
- **Fork request**: Sends IDENTICAL cache-key parameters (system, tools/permissions, model, messages through last assistant response, thinking settings) to hit the parent's cache
- **Cache billing**: Only new tokens (the suggestion prompt + model's suggestion output) are billed at standard rates; cached tokens are ~90% cheaper

**Critical Cache Preservation Rules:**
The code contains extensive comments documenting what busts the cache:
- DO NOT override `effortValue` or `maxOutputTokens` on the fork, even via `output_config` or `getAppState`. PR #18143 tested `effort: 'low'` and caused a **45x spike in cache writes** (92.7% → 61% hit rate)
- DO NOT use `tools: []` to deny tools. Instead, use the `canUseTool` callback (client-side permission check). Setting empty tools array changes the cache key to 0% hits
- Safe overrides are:
  - `abortController` (not sent to API, client-side only)
  - `skipTranscript` (client-side only)
  - `skipCacheWrite` (controls cache_control markers, not the key)
  - `canUseTool` (client-side permission check)

**Code Snippet:**
```typescript
const result = await runForkedAgent({
  promptMessages: [createUserMessage({ content: prompt })],
  cacheSafeParams, // Don't override tools/thinking settings - busts cache
  canUseTool, // Deny via callback, NOT tools:[]
  querySource: 'prompt_suggestion',
  forkLabel: 'prompt_suggestion',
  overrides: {
    abortController,
  },
  skipTranscript: true,
  skipCacheWrite: true,
})
```

The fork's payload: parent's output tokens + suggestion prompt tokens + new generation.

---

### 3. The SUGGESTION_PROMPT: Intent Prediction with User Style Matching

The suggestion prompt is a carefully crafted system prompt that predicts what the user would naturally type next. It's framed as **predicting user intent**, not providing advice.

**Core Test Question:**
```
"THE TEST: Would they think 'I was just about to type that'?"
```

**Prompt Structure:**
```
[SUGGESTION MODE: Suggest what the user might naturally type next into Claude Code.]

FIRST: Look at the user's recent messages and original request.

Your job is to predict what THEY would type - not what you think they should do.

THE TEST: Would they think "I was just about to type that"?

EXAMPLES:
- User asked "fix the bug and run tests", bug is fixed → "run the tests"
- After code written → "try it out"
- Claude offers options → suggest the one the user would likely pick, based on conversation
- Claude asks to continue → "yes" or "go ahead"
- Task complete, obvious follow-up → "commit this" or "push it"
- After error or misunderstanding → silence (let them assess/correct)

Be specific: "run the tests" beats "continue".

NEVER SUGGEST:
- Evaluative ("looks good", "thanks")
- Questions ("what about...?")
- Claude-voice ("Let me...", "I'll...", "Here's...")
- New ideas they didn't ask about
- Multiple sentences

Stay silent if the next step isn't obvious from what the user said.

Format: 2-12 words, match the user's style. Or nothing.

Reply with ONLY the suggestion, no quotes or explanation.
```

**Prompt Variants:**
- Currently only one variant: `'user_intent'`
- Infrastructure allows for `'stated_intent'` variant but not yet implemented
- Both currently map to identical `SUGGESTION_PROMPT`

**Extracted Output:**
The fork searches all returned messages for the first text block:
```typescript
for (const msg of result.messages) {
  if (msg.type !== 'assistant') continue
  const textBlock = msg.message.content.find(b => b.type === 'text')
  if (textBlock?.type === 'text') {
    const suggestion = textBlock.text.trim()
    if (suggestion) {
      return { suggestion, generationRequestId }
    }
  }
}
```

The system extracts `generationRequestId` from the first assistant message for RL dataset joins.

---

### 4. Guard Checks: Conversation Maturity, Errors, Cache Freshness, AppState

**Guard Checks Applied in `tryGenerateSuggestion()`:**

1. **Abort Check**: If `abortController.signal.aborted`, return null immediately

2. **Conversation Maturity** (≥2 assistant turns):
   ```typescript
   const assistantTurnCount = count(messages, m => m.type === 'assistant')
   if (assistantTurnCount < 2) {
     logSuggestionSuppressed('early_conversation', ...)
     return null
   }
   ```
   Suggestions don't appear until the assistant has responded at least twice (ensuring context for good predictions)

3. **API Error Check**: Last assistant message must not be an API error
   ```typescript
   const lastAssistantMessage = getLastAssistantMessage(messages)
   if (lastAssistantMessage?.isApiErrorMessage) {
     logSuggestionSuppressed('last_response_error', ...)
     return null
   }
   ```

4. **Cache Freshness Check** (≤10,000 uncached tokens):
   ```typescript
   const MAX_PARENT_UNCACHED_TOKENS = 10_000

   export function getParentCacheSuppressReason(lastAssistantMessage) {
     const usage = lastAssistantMessage.message.usage
     const inputTokens = usage.input_tokens ?? 0
     const cacheWriteTokens = usage.cache_creation_input_tokens ?? 0
     const outputTokens = usage.output_tokens ?? 0

     return inputTokens + cacheWriteTokens + outputTokens > 10_000
       ? 'cache_cold'
       : null
   }
   ```
   If parent request has >10k uncached tokens, fork suggestion is too expensive and is suppressed

5. **AppState Suppression Checks** via `getSuggestionSuppressReason()`:
   - `promptSuggestionEnabled === false`: user disabled feature
   - `pendingWorkerRequest || pendingSandboxRequest`: waiting for permission
   - `elicitation.queue.length > 0`: active permission dialog
   - `toolPermissionContext.mode === 'plan'`: in planning mode
   - Rate limit (external users only): `USER_TYPE === 'external' && currentLimits.status !== 'allowed'`

**Suppression Reasons Logged:**
- `'disabled'`, `'pending_permission'`, `'elicitation_active'`, `'plan_mode'`, `'rate_limit'`
- `'aborted'`, `'early_conversation'`, `'last_response_error'`, `'cache_cold'`, `'empty'` (from filtering)

---

### 5. 14-Point Quality Filter: Multi-Stage Validation

**`shouldFilterSuggestion(suggestion, promptId, source)` applies 14 distinct filters:**

```typescript
const filters: Array<[string, () => boolean]> = [
  ['done', () => lower === 'done'],

  ['meta_text', () =>
    lower === 'nothing found' || lower === 'nothing found.' ||
    lower.startsWith('nothing to suggest') || lower.startsWith('no suggestion') ||
    /\bsilence is\b|\bstay(s|ing)? silent\b/.test(lower) ||
    /^\W*silence\W*$/.test(lower)
  ],

  ['meta_wrapped', () =>
    /^\(.*\)$|^\[.*\]$/.test(suggestion)
  ],

  ['error_message', () =>
    lower.startsWith('api error:') || lower.startsWith('prompt is too long') ||
    lower.startsWith('request timed out') || lower.startsWith('invalid api key') ||
    lower.startsWith('image was too large')
  ],

  ['prefixed_label', () => /^\w+:\s/.test(suggestion)],

  ['too_few_words', () => {
    if (wordCount >= 2) return false
    if (suggestion.startsWith('/')) return false // Allow slash commands

    const ALLOWED_SINGLE_WORDS = new Set([
      'yes', 'yeah', 'yep', 'yea', 'yup', 'sure', 'ok', 'okay', // Affirmatives
      'push', 'commit', 'deploy', 'stop', 'continue', 'check', 'exit', 'quit', // Actions
      'no', // Negation
    ])
    return !ALLOWED_SINGLE_WORDS.has(lower)
  }],

  ['too_many_words', () => wordCount > 12],

  ['too_long', () => suggestion.length >= 100],

  ['multiple_sentences', () => /[.!?]\s+[A-Z]/.test(suggestion)],

  ['has_formatting', () => /[\n*]|\*\*/.test(suggestion)],

  ['evaluative', () =>
    /thanks|thank you|looks good|sounds good|that works|that worked|that's all|
     nice|great|perfect|makes sense|awesome|excellent/.test(lower)
  ],

  ['claude_voice', () =>
    /^(let me|i'll|i've|i'm|i can|i would|i think|i notice|
       here's|here is|here are|that's|this is|this will|
       you can|you should|you could|sure,|of course|certainly)/i.test(suggestion)
  ],
]
```

**Filter Behaviors:**
- **Single-word whitelist**: 23 pre-approved single-word commands bypass the 2-word minimum (yes, push, commit, etc.)
- **Slash command whitelist**: `/help` and other slash commands bypass the 2-word minimum
- **Meta-text detection**: Catches the model "thinking out loud" (e.g., "silence is golden", wrapped in parens)
- **Evaluative filter**: Suppresses praise/acknowledgments ("looks good", "thanks")
- **Claude-voice filter**: Suppresses assistant-like phrasing at the start
- **Formatting filter**: Rejects suggestions with newlines, bold markdown, etc.

Each rejection is logged with the filter reason for telemetry.

---

### 6. Configuration Hierarchy: Env Var → GrowthBook → User Setting

**`shouldEnablePromptSuggestion()` uses a 4-tier hierarchy:**

1. **Environment Variable Override** (highest priority):
   - `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=false`: force disabled
   - `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=true`: force enabled
   - Logs: `enabled=true/false, source: 'env'`

2. **GrowthBook Feature Flag** (if no env override):
   - `getFeatureValue_CACHED_MAY_BE_STALE('tengu_chomp_inflection', false)`
   - Default: `false` (feature flag disabled by default)
   - If flag is false, logs `source: 'growthbook'` and returns false

3. **Non-Interactive Session Check**:
   - If `getIsNonInteractiveSession()` is true, disable suggestions
   - Logs: `source: 'non_interactive'` (print mode, piped input, SDK)

4. **Swarm Teammate Check**:
   - If `isAgentSwarmsEnabled() && isTeammate()`, disable suggestions
   - Only the swarm leader should show suggestions
   - Logs: `source: 'swarm_teammate'`

5. **User Settings** (lowest priority):
   - `getInitialSettings()?.promptSuggestionEnabled !== false`
   - User can disable via settings UI
   - Default: `true` if not explicitly set to false
   - Logs: `source: 'setting'`

**Initialization Event:**
Every time the suggestion system initializes, it logs a `'tengu_prompt_suggestion_init'` event with `enabled` boolean and `source` field.

---

### 7. Telemetry: Comprehensive Multi-Dimensional Tracking

#### Initialization Telemetry:
```typescript
logEvent('tengu_prompt_suggestion_init', {
  enabled: boolean,
  source: 'env' | 'growthbook' | 'non_interactive' | 'swarm_teammate' | 'setting',
})
```

#### Suppression Telemetry:
```typescript
logEvent('tengu_prompt_suggestion', {
  source?: 'cli' | 'sdk',
  outcome: 'suppressed',
  reason: 'aborted' | 'early_conversation' | 'last_response_error' | 'cache_cold' |
          'disabled' | 'pending_permission' | 'elicitation_active' | 'plan_mode' |
          'rate_limit' | 'empty' | 'done' | 'meta_text' | 'meta_wrapped' | 'error_message' |
          'prefixed_label' | 'too_few_words' | 'too_many_words' | 'too_long' |
          'multiple_sentences' | 'has_formatting' | 'evaluative' | 'claude_voice',
  prompt_id: 'user_intent' | 'stated_intent',
  suggestion?: string (USER_TYPE==='ant' only),
})
```

#### Acceptance/Rejection Telemetry:
```typescript
logEvent('tengu_prompt_suggestion', {
  source: 'sdk' (from logSuggestionOutcome),
  outcome: 'accepted' | 'ignored',
  prompt_id: 'user_intent' | 'stated_intent',
  generationRequestId?: string,
  timeToAcceptMs?: number (if accepted),
  timeToIgnoreMs?: number (if ignored),
  similarity: number (ratio of userInput length to suggestion length),
  suggestion?: string (USER_TYPE==='ant' only),
  userInput?: string (USER_TYPE==='ant' only),
})
```

#### Similarity Calculation:
```typescript
const similarity = Math.round((userInput.length / (suggestion.length || 1)) * 100) / 100
const wasAccepted = userInput === suggestion
```
- 1:1 match = accepted
- Similarity ratio used to measure how close user's actual input was to suggestion
- Anthropic-only: logs actual `suggestion` and `userInput` text for RL dataset construction

#### Internal vs External Data:
- Anthropic users (`USER_TYPE === 'ant'`): logs full suggestion text, user input, `generationRequestId` for RL training data
- External users: logs only outcome, reason, similarity metrics, no text content

---

## PART 2 - SPECULATIVE EXECUTION

### 8. How Speculation Works: Pre-Computing Response to Suggested Input

Speculative execution is the "second act" that runs after a suggestion is generated. It **pre-computes Claude's response to the suggested input in an isolated environment** so that when the user accepts the suggestion, the speculated work is already done and immediately injected into the conversation.

**Trigger Chain:**
1. Suggestion is generated via `generateSuggestion()`
2. If `isSpeculationEnabled()` returns true, `startSpeculation()` is invoked in the background
3. Speculated computation runs in parallel (non-blocking)
4. If user accepts suggestion before speculation completes, partial speculated work is used
5. If user rejects suggestion or types something else, speculation is aborted and discarded

**Speculative Execution Goals:**
- **Speed**: User gets instant work (up to 20 turns pre-computed) when they accept the suggestion
- **Preview**: User can see what Claude will do BEFORE committing to the suggestion
- **Fallback**: If speculation doesn't complete, it gracefully falls back to a normal query

**Time Savings Calculation:**
```typescript
const timeSavedMs = Math.min(acceptedAt, boundary?.completedAt ?? Infinity) - startTime
```
The system measures the time from suggestion generation start to the earlier of: acceptance time or speculation completion time.

---

### 9. Overlay Filesystem: Copy-on-Write Isolation in `~/.claude/tmp/speculation`

Speculative execution operates on an **isolated copy-on-write filesystem overlay** to prevent side effects from polluting the main working directory.

**Overlay Structure:**
```
~/.claude/tmp/speculation/{process_id}/{speculation_id}/
```

Example: `~/.claude/tmp/speculation/12345/a1b2c3d4/`

**Copy-on-Write Logic:**

**Write Operations (Edit, Write, NotebookEdit):**
1. Check if file is already in overlay at `{overlayPath}/{rel}`
2. If NOT yet copied, copy the original from main cwd to overlay:
   ```typescript
   if (!writtenPathsRef.current.has(rel)) {
     const overlayFile = join(overlayPath, rel)
     await mkdir(dirname(overlayFile), { recursive: true })
     try {
       await copyFile(join(cwd, rel), overlayFile)
     } catch {
       // Original may not exist (new file creation) - that's fine
     }
     writtenPathsRef.current.add(rel)
   }
   ```
3. Rewrite the file path to point to overlay:
   ```typescript
   input = { ...input, [pathKey]: join(overlayPath, rel) }
   ```
4. Tool executes on overlay copy

**Read Operations (Read, Glob, Grep, etc.):**
1. If file was previously written in this speculation, read from overlay:
   ```typescript
   if (writtenPathsRef.current.has(rel)) {
     input = { ...input, [pathKey]: join(overlayPath, rel) }
   }
   ```
2. Otherwise, read from main cwd (no rewrite)
   ```typescript
   // Otherwise read from main (no rewrite)
   ```

**Overlay Cleanup:**
```typescript
safeRemoveOverlay(overlayPath): void {
  rm(
    overlayPath,
    { recursive: true, force: true, maxRetries: 3, retryDelay: 100 },
    () => {},
  )
}
```
- Recursive delete with force flag
- Retries up to 3 times with 100ms delay (for Windows file locking)
- Non-blocking callback (errors are swallowed)

**File Acceptance:**
When speculation is accepted, `copyOverlayToMain()` copies all written files back to main cwd:
```typescript
async function copyOverlayToMain(overlayPath: string, writtenPaths: Set<string>, cwd: string) {
  let allCopied = true
  for (const rel of writtenPaths) {
    const src = join(overlayPath, rel)
    const dest = join(cwd, rel)
    try {
      await mkdir(dirname(dest), { recursive: true })
      await copyFile(src, dest)
    } catch {
      allCopied = false
      logForDebugging(`[Speculation] Failed to copy ${rel} to main`)
    }
  }
  return allCopied
}
```

---

### 10. Tool Permission Model: SAFE_READ_ONLY_TOOLS with Write Blocking

Speculation uses a **allowlist-based tool permission model** with four categories:

#### Safe Read-Only Tools (Always Allowed):
```typescript
const SAFE_READ_ONLY_TOOLS = new Set([
  'Read',
  'Glob',
  'Grep',
  'ToolSearch',
  'LSP',
  'TaskGet',
  'TaskList',
])
```

These tools can execute without speculation boundaries because they don't modify state.

#### Write Tools (Require Permission Mode):
```typescript
const WRITE_TOOLS = new Set(['Edit', 'Write', 'NotebookEdit'])
```

Write tools trigger an **edit boundary** unless the user has pre-approved auto-editing:
```typescript
const appState = context.toolUseContext.getAppState()
const { mode, isBypassPermissionsModeAvailable } = appState.toolPermissionContext

const canAutoAcceptEdits =
  mode === 'acceptEdits' ||
  mode === 'bypassPermissions' ||
  (mode === 'plan' && isBypassPermissionsModeAvailable)

if (!canAutoAcceptEdits) {
  // Stop at file edit boundary
  return denySpeculation(
    'Speculation paused: file edit requires permission',
    'speculation_edit_boundary',
  )
}
```

Permission modes:
- `'acceptEdits'`: user has pre-approved all edits
- `'bypassPermissions'`: user has bypassed permissions entirely
- `'plan'` + `isBypassPermissionsModeAvailable`: user is in plan mode and can auto-promote to bypassing
- All others: deny edits, set boundary, abort speculation

#### Bash Tool (Read-Only Command Allowlist):
```typescript
if (tool.name === 'Bash') {
  const command = input.command as string ?? ''
  if (!command || checkReadOnlyConstraints({ command }, commandHasAnyCd(command)).behavior !== 'allow') {
    // Stop at bash boundary
    updateActiveSpeculationState(..., boundary: { type: 'bash', command, ... })
    return denySpeculation(..., 'speculation_bash_boundary')
  }
  // Read-only bash is allowed
  return { behavior: 'allow', ... }
}
```

Bash commands are validated against `checkReadOnlyConstraints()` which allows:
- `ls`, `find`, `grep`, `cat`, `head`, `tail`, `wc`, `file` type commands
- Denies: `cd`, `rm`, `mv`, `curl`, `git push`, etc.

#### All Other Tools (Denied):
```typescript
// Deny all other tools by default
logForDebugging(`[Speculation] Stopping at denied tool: ${tool.name}`)
updateActiveSpeculationState(setAppState, () => ({
  boundary: {
    type: 'denied_tool',
    toolName: tool.name,
    detail: extractedDetail,
    completedAt: Date.now(),
  },
}))
return denySpeculation(
  `Tool ${tool.name} not allowed during speculation`,
  'speculation_unknown_tool',
)
```

Any tool not in the allowlist or write tools sets triggers a `denied_tool` boundary.

---

### 11. Speculation Boundaries: Edit, Bash, Denied_Tool, Complete, Turn/Message Limits

Boundaries are the "stop signals" that halt speculation and prevent it from going too far.

#### Four Boundary Types:

**1. Edit Boundary** (file edit requires user permission):
```typescript
boundary: { type: 'edit', toolName: 'Edit'|'Write'|'NotebookEdit', filePath: string, completedAt: number }
```
Set when a write tool is requested but user hasn't pre-approved edits.

**2. Bash Boundary** (non-read-only command):
```typescript
boundary: { type: 'bash', command: string, completedAt: number }
```
Set when Bash command fails read-only validation (e.g., `rm`, `cd`).

**3. Denied_Tool Boundary** (disallowed tool):
```typescript
boundary: { type: 'denied_tool', toolName: string, detail: string, completedAt: number }
```
Set when any tool outside the allowlist is attempted (e.g., WebFetch, WebSearch).

**4. Complete Boundary** (full completion):
```typescript
boundary: { type: 'complete', completedAt: number, outputTokens: number }
```
Set when speculation finishes naturally without hitting any boundary.

#### Turn/Message Limits:

```typescript
const MAX_SPECULATION_TURNS = 20
const MAX_SPECULATION_MESSAGES = 100
```

**Turn limit**: A "turn" is an assistant message. Speculation aborts after 20 assistant turns.

**Message limit**: A "message" is any user or assistant message. Speculation aborts after 100 total messages.

Both limits are enforced in the `onMessage` callback:
```typescript
onMessage: msg => {
  if (msg.type === 'assistant' || msg.type === 'user') {
    messagesRef.current.push(msg)
    if (messagesRef.current.length >= MAX_SPECULATION_MESSAGES) {
      abortController.abort()
    }
  }
}
```

---

### 12. Message Injection on Acceptance: Stripping & Merging State

When speculation is accepted, speculated messages are injected into the main conversation via `handleSpeculationAccept()`.

#### Message Preparation (`prepareMessagesForInjection()`):

This function **cleans** speculated messages before injection by:

1. **Strip Thinking Blocks**: Remove all `type: 'thinking'` and `type: 'redacted_thinking'` blocks
   ```typescript
   b.type !== 'thinking' && b.type !== 'redacted_thinking'
   ```

2. **Strip Pending Tool Uses**: Remove `tool_use` blocks without corresponding successful results
   ```typescript
   !(b.type === 'tool_use' && !toolIdsWithSuccessfulResults.has(b.id!))
   ```

3. **Strip Interrupted Tools**: Remove `tool_result` blocks that failed or have interrupt messages
   ```typescript
   !(
     b.type === 'tool_result' &&
     !toolIdsWithSuccessfulResults.has(b.tool_use_id!)
   )

   // Successful results: NOT error AND no interrupt message
   const isSuccessful = (b: ToolResult) =>
     !b.is_error &&
     !(typeof b.content === 'string' && b.content.includes(INTERRUPT_MESSAGE_FOR_TOOL_USE))
   ```

4. **Strip Interrupt Messages**: Remove standalone user interrupt messages (created when speculation was aborted)
   ```typescript
   !(
     b.type === 'text' &&
     (b.text === INTERRUPT_MESSAGE || b.text === INTERRUPT_MESSAGE_FOR_TOOL_USE)
   )
   ```

5. **Drop Empty Messages**: If a message has no non-whitespace content after cleaning, drop it
   ```typescript
   const hasNonWhitespaceContent = content.some(
     (b: { type: string; text?: string }) =>
       b.type !== 'text' || (b.text !== undefined && b.text.trim() !== ''),
   )
   if (!hasNonWhitespaceContent) return null
   ```

#### Injection Sequence:

```typescript
// 1. Inject user message first for instant visual feedback
const userMessage = createUserMessage({ content: input })
setMessages(prev => [...prev, userMessage])

// 2. Accept speculation (copies overlay files, calculates time saved)
const result = await acceptSpeculation(...)

// 3. Clean speculated messages
let cleanMessages = prepareMessagesForInjection(speculationMessages)

// 4. If speculation didn't complete, drop trailing assistant messages
// (models without prefill reject conversations ending in assistant turns)
if (!isComplete) {
  const lastNonAssistant = cleanMessages.findLastIndex(m => m.type !== 'assistant')
  cleanMessages = cleanMessages.slice(0, lastNonAssistant + 1)
}

// 5. Inject speculated messages
setMessages(prev => [...prev, ...cleanMessages])

// 6. Merge file state cache (Read files extracted from tool results)
const extracted = extractReadFilesFromMessages(cleanMessages, cwd, READ_FILE_STATE_CACHE_SIZE)
readFileState.current = mergeFileStateCaches(readFileState.current, extracted)

// 7. Optionally inject ANT-only feedback message (time saved, tokens, turns)
if (feedbackMessage) {
  setMessages(prev => [...prev, feedbackMessage])
}
```

#### Feedback Message (ANT-only):
```typescript
function createSpeculationFeedbackMessage(messages, boundary, timeSavedMs, sessionTotalMs) {
  if (process.env.USER_TYPE !== 'ant') return null

  const toolUses = countToolsInMessages(messages)
  const tokens = boundary?.type === 'complete' ? boundary.outputTokens : null

  // Example: "[ANT-ONLY] Speculated 2 tool uses · 1,234 tokens · +2s 450ms saved (5s 200ms this session)"
}
```

Shows:
- Tool use count or turn count
- Output tokens (if complete)
- Time saved for this speculation
- Session total time saved

---

### 13. Pipelined Suggestions: NEXT Suggestion Generation During Speculation

Pipelined suggestions enable **suggestion chains** — generating the NEXT suggestion while the FIRST speculation runs.

**Trigger:** In `startSpeculation()`, after `runForkedAgent()` completes:
```typescript
// Pipeline: generate the next suggestion while we wait for the user to accept
void generatePipelinedSuggestion(
  contextRef.current,
  suggestionText,
  messagesRef.current,
  setAppState,
  abortController,
)
```

**Implementation:**

The `generatePipelinedSuggestion()` function:
1. Checks if a new suggestion should be generated (guards: suppression reasons)
2. Augments the context with the first suggestion + speculated messages:
   ```typescript
   const augmentedContext: REPLHookContext = {
     ...context,
     messages: [
       ...context.messages,
       createUserMessage({ content: suggestionText }),
       ...speculatedMessages,
     ],
   }
   ```
3. Generates a new suggestion from this augmented context
4. Stores it in `speculationState.pipelinedSuggestion`

**Promotion Logic:**

If speculation completes fully (reaches `'complete'` boundary), the pipelined suggestion is **promoted** into the main suggestion state:
```typescript
if (isComplete && speculationState.pipelinedSuggestion) {
  const { text, promptId, generationRequestId } = speculationState.pipelinedSuggestion
  logForDebugging(`[Speculation] Promoting pipelined suggestion: "${text.slice(0, 50)}..."`)

  setAppState(prev => ({
    ...prev,
    promptSuggestion: {
      text,
      promptId,
      shownAt: Date.now(),
      acceptedAt: 0,
      generationRequestId,
    },
  }))

  // Start speculation on the pipelined suggestion
  const augmentedContext: REPLHookContext = {
    ...speculationState.contextRef.current,
    messages: [
      ...speculationState.contextRef.current.messages,
      createUserMessage({ content: input }),
      ...cleanMessages,
    ],
  }
  void startSpeculation(text, augmentedContext, setAppState, true)
}
```

This creates a **chain of suggestions and speculations** that keep going as long as the user accepts them.

**Chain Example:**
1. Suggestion A generated → Speculation A starts
2. Pipelined Suggestion B generated (during Speculation A)
3. User accepts Suggestion A → Speculation A completes
4. Suggestion B promoted to main
5. Speculation B starts (augmented with Suggestion A's results)
6. Pipelined Suggestion C generated (during Speculation B)
7. Loop continues...

---

### 14. Currently Anthropic-Only (`USER_TYPE === 'ant'`)

Speculation is currently **restricted to Anthropic internal users only**.

**Gating:**
```typescript
export function isSpeculationEnabled(): boolean {
  const enabled =
    process.env.USER_TYPE === 'ant' &&
    (getGlobalConfig().speculationEnabled ?? true)
  logForDebugging(`[Speculation] enabled=${enabled}`)
  return enabled
}
```

**Reasoning:**
- Feature is still in development/validation phase
- Anthropic-only telemetry includes sensitive details (suggestion text, user input, message content)
- Speculation has non-trivial infrastructure (overlay filesystem, etc.)
- Performance characteristics need internal validation before external rollout

**Configuration:**
```typescript
getGlobalConfig().speculationEnabled ?? true
```
Even for Anthropic, speculation can be disabled via config: `{ speculationEnabled: false }`

---

### 15. Error Handling: Fail-Open Strategy

Speculation uses a **fail-open strategy**: errors never break the conversation, and users never see speculative errors.

#### Error Categories:

**1. During `startSpeculation()`** (main speculation run):
```typescript
try {
  const result = await runForkedAgent({ ... })
  // ... use result
} catch (error) {
  abortController.abort()
  safeRemoveOverlay(overlayPath)

  if (error instanceof Error && error.name === 'AbortError') {
    // User aborted (user typed before speculation finished)
    resetSpeculationState(setAppState)
    return
  }

  // Log error telemetry, but don't crash
  logError(error instanceof Error ? error : new Error('Speculation failed'))
  logSpeculation(id, 'error', startTime, suggestionText.length, messagesRef.current, null, {
    error_type: error instanceof Error ? error.name : 'Unknown',
    error_message: errorMessage(error).slice(0, 200),
    error_phase: 'start',
    is_pipelined: isPipelined,
  })

  resetSpeculationState(setAppState)
}
```

**2. During `handleSpeculationAccept()`** (message injection):
```typescript
try {
  // ... prepare and inject messages
  return { queryRequired: !isComplete }
} catch (error) {
  // Fail open: log error and fall back to normal query flow
  logError(error instanceof Error ? error : new Error('handleSpeculationAccept failed'))

  logSpeculation(speculationState.id, 'error', startTime, suggestionLength, messagesRef.current, boundary, {
    error_type: error instanceof Error ? error.name : 'Unknown',
    error_message: errorMessage(error).slice(0, 200),
    error_phase: 'accept',
    is_pipelined: speculationState.isPipelined,
  })

  safeRemoveOverlay(getOverlayPath(speculationState.id))
  resetSpeculationState(setAppState)

  // Query required so user's message is processed normally (without speculated work)
  return { queryRequired: true }
}
```

**3. During `generatePipelinedSuggestion()`**:
```typescript
try {
  // ... generate suggestion
} catch (error) {
  if (error instanceof Error && error.name === 'AbortError') return
  logForDebugging(`[Speculation] Pipelined suggestion failed: ${errorMessage(error)}`)
  // Silently fails - pipelined suggestion is optional
}
```

#### Key Behaviors:
- **No crashes**: All errors are caught and logged
- **Overlay cleanup**: Failed speculation deletes the overlay directory
- **State reset**: Speculation state is reset to IDLE
- **Query fallback**: If error occurs during acceptance, `queryRequired: true` ensures user's input is processed normally (no speculated work is used)
- **No user visibility**: Users never see error messages from speculation
- **Abort distinction**: AbortError (user typed, deliberate abort) is handled separately from true errors

---

### 16. Telemetry: Comprehensive Speculation Tracking

#### Speculation Events:

```typescript
logEvent('tengu_speculation', {
  speculation_id: string,
  outcome: 'accepted' | 'aborted' | 'error',
  duration_ms: number,
  suggestion_length: number,
  tools_executed: number,
  completed: boolean,
  boundary_type?: 'edit' | 'bash' | 'denied_tool' | 'complete',
  boundary_tool?: string,
  boundary_detail?: string,
  ...extras,
})
```

#### Outcome-Specific Metrics:

**Accepted:**
```typescript
extras: {
  message_count: number,
  time_saved_ms: number,
  is_pipelined: boolean,
}
```

**Aborted:**
```typescript
extras: {
  abort_reason: 'user_typed',
  is_pipelined: boolean,
}
```

**Error:**
```typescript
extras: {
  error_type: string,
  error_message: string (first 200 chars),
  error_phase: 'start' | 'accept',
  is_pipelined: boolean,
}
```

#### Boundary Tool & Detail Extraction:

```typescript
function getBoundaryTool(boundary: CompletionBoundary | null): string | undefined {
  switch (boundary?.type) {
    case 'bash': return 'Bash'
    case 'edit': case 'denied_tool': return boundary.toolName
    case 'complete': return undefined
  }
}

function getBoundaryDetail(boundary: CompletionBoundary | null): string | undefined {
  switch (boundary?.type) {
    case 'bash': return boundary.command.slice(0, 200)
    case 'edit': return boundary.filePath
    case 'denied_tool': return boundary.detail
    case 'complete': return undefined
  }
}
```

#### Session Total Tracking:

Every accepted speculation increments `speculationSessionTimeSavedMs`:
```typescript
speculationSessionTimeSavedMs: prev.speculationSessionTimeSavedMs + timeSavedMs
```

Feedback message shows this running total to Anthropic users.

#### Speculation-Accept Transcript Entry:

When speculation is accepted with `timeSavedMs > 0`, an entry is written to the transcript:
```typescript
const entry: SpeculationAcceptMessage = {
  type: 'speculation-accept',
  timestamp: new Date().toISOString(),
  timeSavedMs,
}
void appendFile(getTranscriptPath(), jsonStringify(entry) + '\n', { mode: 0o600 })
```

This allows historical analysis of speculation value across sessions.

---

## Integration: Suggestion → Speculation → Pipelined Chain

The complete flow:

1. **Post-Sampling Hook** → `executePromptSuggestion()` is called after Claude responds
2. **Guard Checks** → Verify conversation maturity, no errors, cache freshness, user settings
3. **Fork Generation** → `runForkedAgent()` with cache reuse generates a suggestion
4. **Quality Filter** → 14-point filter rejects bad suggestions
5. **Speculation Start** → If enabled, `startSpeculation(suggestion)` runs in background
   - Overlay filesystem created
   - Tool permission checks applied
   - Claude runs in isolated environment
   - Pre-computes response up to a boundary (edit, bash, denied tool, complete, or limit)
6. **Pipelined Generation** → While speculation runs, next suggestion is generated using (original messages + suggestion + speculated messages)
7. **User Action** → User accepts or rejects suggestion, or types something else
8. **Acceptance** → If accepted before speculation completes:
   - Overlay files copied back to main
   - Speculated messages injected (with thinking/pending tools stripped)
   - File state cache merged
   - Pipelined suggestion promoted (if speculation completed fully)
   - Chain continues with new speculation
9. **Rejection/Timeout** → Overlay deleted, speculation state reset, normal query flow

**Result:** Suggestions that feel instant because speculated work is pre-computed and ready to merge when the user accepts.

---

## Architecture Diagram

```
Post-Sampling Hook
        ↓
[Guard Checks: maturity, errors, cache, AppState]
        ↓
    [Abort?] → return null
        ↓
generateSuggestion(runForkedAgent)
        ↓
[14-Point Quality Filter]
        ↓
    [Reject?] → log suppression, return null
        ↓
Store in AppState.promptSuggestion
        ↓
[isSpeculationEnabled?]
        ↓
    startSpeculation(suggestion)
    ┌─────────────────────────┐
    │ Overlay FS Created      │
    │ ~/.claude/tmp/...       │
    │ Copy-on-write isolation │
    │                         │
    │ runForkedAgent (fork)   │
    │ ├─ Read tools allowed   │
    │ ├─ Write tools guarded  │
    │ ├─ Bash read-only only  │
    │ └─ All others denied    │
    │                         │
    │ Boundaries:            │
    │ - Edit (permission)     │
    │ - Bash (state change)   │
    │ - Denied tool           │
    │ - Complete              │
    │ - Turn/message limits   │
    │                         │
    │ Parallel: pipelined     │
    │ suggestion generation   │
    └─────────────────────────┘
         │             │
    [Aborted by user] [Complete]
         │             │
         ↓             ↓
    [Overlay        [Promote
     deleted]       pipelined]
         │             │
         └─────┬───────┘
               ↓
        [User accepts?]
               ↓
        acceptSpeculation()
        ├─ Copy overlay → main
        ├─ Remove overlay
        ├─ Inject clean messages
        ├─ Merge file state
        └─ Log telemetry
               ↓
        [Return speculated result]
               ↓
        [Start next speculation]
        [on pipelined suggestion?]
               ↓
           [Chain continues]
```

---

## Key Implementation Insights

### Cache Reuse Strategy
The ~60-70% cache hit rate is achieved by:
- Reusing parent's full conversation (system, messages, thinking) as cache key
- Only adding suggestion prompt (~200 tokens) + generation (~5-10 tokens)
- Avoiding parameters that change the cache key
- Using `canUseTool` callback instead of `tools: []` array

### Overlay Filesystem Design
The copy-on-write overlay:
- Prevents side effects from breaking main conversation
- Allows files to be committed atomically (all-or-nothing)
- Enables quick rollback (just delete overlay)
- Handles new file creation naturally (copy fails gracefully)

### Tool Permission Integration
Speculation respects the user's permission mode:
- Edit boundary halts if user hasn't approved auto-editing
- But doesn't reject edits — just pauses for user to confirm
- Maintains consistency with main CLI permission model

### Fail-Open Philosophy
- No speculation error ever breaks the user's conversation
- Errors are logged for telemetry but never surfaced
- If acceptance fails, normal query flow resumes
- User always gets their work done, just without speculation

### Telemetry Instrumentation
- Init telemetry: tracks feature enablement hierarchy
- Suppression telemetry: 23+ distinct suppression reasons
- Outcome telemetry: acceptance/rejection, similarity ratios, internal text (ANT only)
- Speculation telemetry: outcome, boundaries, tools executed, time saved, error details

