# Small Services Deep Dive: autoDream, tips, AgentSummary, toolUseSummary, MagicDocs

## Overview

This analysis covers five specialized service modules that provide background automation, user guidance, and documentation features in Claude Code. These services run alongside the main REPL loop and handle memory consolidation, contextual hints, subagent monitoring, tool summarization, and auto-maintaining documentation.

---

## 1. autoDream: Background Memory Consolidation

### Purpose
autoDream is a **background memory consolidation system** that periodically triggers a "dream" process—a reflective pass over memory files to synthesize and organize learnings. It runs as an autonomous subagent in the background, reading session transcripts and updating memory directories without interrupting the user.

### Key Files
- `autoDream.ts`: Main orchestrator for the background process
- `config.ts`: Feature flag and user setting management
- `consolidationLock.ts`: Lock mechanism to prevent concurrent dreams
- `consolidationPrompt.ts`: System prompt builder for the dream agent

### Architecture

#### Trigger Gates (Ordered, Early Exit)

The system uses a **cascading gate pattern** with defensive early exits:

1. **Feature Gate** (`isGateOpen()`)
   - Checks KAIROS mode (disabled in KAIROS—disk-skill dream used instead)
   - Checks remote mode (disabled in SSH/remote)
   - Checks auto-memory enabled (must be enabled)
   - Checks GrowthBook feature flag `tengu_onyx_plover`
   - User setting `autoDreamEnabled` in settings.json overrides GrowthBook

2. **Time Gate** (`readLastConsolidatedAt()`)
   - Reads lock file mtime = last consolidation timestamp
   - Default: 24 hours minimum (configurable via GB)
   - Cost: one stat call per turn

3. **Scan Throttle** (SESSION_SCAN_INTERVAL_MS = 10 minutes)
   - After time-gate passes but session-gate fails, prevent repeated lock contention
   - Scans are expensive; throttle prevents high-frequency probes
   - Lock mtime doesn't advance if no sessions since last consolidation, so time-gate would fire every turn

4. **Session Gate** (`listSessionsTouchedSince()`)
   - Scans `~/.claude/sessions` for session files modified since last consolidation
   - Default: minimum 5 sessions (configurable via GB)
   - Excludes current session (its mtime is always fresh)
   - Cost: parallel stat of all session files in cwd

5. **Lock Gate** (`tryAcquireConsolidationLock()`)
   - Race-safe PID-based lock at `~/.claude-memory/.consolidate-lock`
   - Lock mtime = lastConsolidatedAt
   - Returns priorMtime for rollback on failure
   - Stale lock reclamation: if dead PID or mtime > 1 hour, reclaim

#### Lock Mechanism

Lock file path: `${getAutoMemPath()}/.consolidate-lock`

- **Acquire**: Write current PID, touch mtime to now, return prior mtime
- **Race handling**: Two processes both writing → last-writer wins, loser detects mismatch on re-read
- **Rollback**: On fork failure, rewind mtime to priorMtime (0 → unlink file) to release time-gate
- **Stale detection**: If holder PID dead or mtime > 60 minutes, reclaim

#### Forked Agent Execution

When all gates pass:

```javascript
const result = await runForkedAgent({
  promptMessages: [createUserMessage({ content: prompt })],
  cacheSafeParams: createCacheSafeParams(context),  // Share prompt cache
  canUseTool: createAutoMemCanUseTool(memoryRoot),  // Memory-only tools
  querySource: 'auto_dream',
  forkLabel: 'auto_dream',
  skipTranscript: true,  // Don't add to main transcript
  overrides: { abortController },
  onMessage: makeDreamProgressWatcher(taskId, setAppState),
})
```

**Tool constraints** (in prompt extra):
- Bash: read-only commands only (`ls`, `find`, `grep`, `cat`, `stat`, `wc`, `head`, `tail`)
- No file writes, redirects, or state modifications
- Memory API (from `extractMemories`) available

**Dream Task Tracking**:
- `registerDreamTask()`: Initialize task with session count + abort controller
- `addDreamTurn()`: Record progress (text summary, tool use count, touched file paths)
- `completeDreamTask()`: Mark success
- `failDreamTask()`: Mark failure
- Inline completion message appended to main transcript if files touched

#### Consolidation Prompt (buildConsolidationPrompt)

The dream prompt guides a 4-phase process:

1. **Phase 1—Orient**
   - `ls` memory directory, read entrypoint (`CLAUDE.md` analog)
   - Skim existing topic files to avoid duplicates
   - Review logs/ or sessions/ subdirs if present

2. **Phase 2—Gather Recent Signal**
   - Priority 1: Daily logs (`logs/YYYY/MM/YYYY-MM-DD.md`)
   - Priority 2: Existing memories that drifted (check against codebase)
   - Priority 3: Narrow transcript grep for specific context
   - Avoid exhaustive transcript reads

3. **Phase 3—Consolidate**
   - Merge new signal into existing topic files
   - Convert relative dates to absolute (interpretable after time passes)
   - Delete contradicted facts

4. **Phase 4—Prune and Index**
   - Update index (ENTRYPOINT_NAME, ~25KB max, ~150 chars per line)
   - Remove stale/wrong pointers
   - Resolve contradictions
   - Add new memory links

### Configuration

**Feature flag**: `tengu_onyx_plover` (GrowthBook)
- `enabled: boolean` — master kill switch
- `minHours: number` — time gate threshold (default 24)
- `minSessions: number` — session gate threshold (default 5)

**User setting**: `autoDreamEnabled` in settings.json (overrides GB if present)

### Integration Points

**Lifecycle**:
- `initAutoDream()` called at startup from `backgroundHousekeeping`
- `executeAutoDream(context)` called from postSamplingHooks (stopHooks phase)
- Per-turn cost when enabled: one GB cache read + one stat (lock check)

**State Management**:
- State closure-scoped inside `initAutoDream()` (testable—tests call in beforeEach)
- `lastSessionScanAt` persists across turns to throttle expensive session scans

**Error Handling**:
- `readLastConsolidatedAt()` → 0 on missing file
- `listSessionsTouchedSince()` → logs error, returns early
- `tryAcquireConsolidationLock()` → null on lock held or acquire failure
- Fork failure → rollback lock, log error, emit analytics
- User kill from bg-tasks dialog → respects abort signal, skips double-rollback

**Messaging**:
- Dream progress integrated into app state for UI display
- Completion message appended inline (same surface as extractMemories)
- Analytics: `tengu_auto_dream_fired`, `tengu_auto_dream_completed`, `tengu_auto_dream_failed`

---

## 2. Tips: Contextual User Guidance

### Purpose
The tip system provides **rotating contextual hints** shown on the spinner to guide users toward features, settings, and best practices. Tips are selected based on session history, user behavior, and feature flags, with cooldown periods to prevent fatigue.

### Key Files
- `tipRegistry.ts`: Comprehensive tip catalog (65+ tips, ~950 lines)
- `tipHistory.ts`: Persistence layer for shown tips
- `tipScheduler.ts`: Selection and filtering logic

### Tip Catalog Structure

#### Tip Interface
```typescript
type Tip = {
  id: string
  content: async () => string                    // Rendered content (can use theme)
  cooldownSessions: number                       // Min sessions between shows
  isRelevant: async (context?: TipContext) => boolean  // Dynamic relevance
}
```

#### Relevance Context (TipContext)
```typescript
type TipContext = {
  bashTools?: Set<string>         // CLI tools detected in session
  readFileState?: FileStateCache  // Files read in session
  theme?: Theme                   // Terminal theme
}
```

### Tip Categories

#### 1. **Onboarding Tips** (new-user-warmup)
- Show to users with < 10 startups
- Cooldown: 3 sessions

#### 2. **Feature Discovery** (10+ tips)
- Plan Mode (`plan-mode-for-complex-tasks`, 7+ days since last use)
- Permission modes (`default-permission-mode-config`)
- Git worktrees (`git-worktrees`, for users with > 50 startups)
- Terminal setup (`terminal-setup`, `shift-enter`)
- Memory system (`memory-command`, first use)
- Theme (`theme-command`)
- Prompt queue (`prompt-queue`, first 3 uses)
- Real-time steering (`enter-to-steer-in-realtime`)
- Todo lists (`todo-list`)
- Continue/resume (`continue`, `/resume --list`)
- Rename conversations (`rename-conversation`)
- Custom commands (`custom-commands`, skills in `.claude/skills/`)
- Custom agents (`custom-agents`, `/agents`)
- Color/rename for multi-clauding (`color-when-multi-clauding`)

#### 3. **IDE Integration** (5+ tips)
- IDE upsells (`desktop-app`, `desktop-shortcut`, `web-app`, `mobile-app`)
- VS Code/Cursor/Windsurf command installation (`vscode-command-install`)
- IDE connection (`ide-upsell-external-terminal`, if running IDEs detected)
- GitHub app (`install-github-app`)
- Slack app (`install-slack-app`)

#### 4. **Environment-Specific** (6+ tips)
- Color terminal (`colorterm-truecolor`, if COLORTERM unset)
- Image paste (`paste-images-mac` for macOS, `drag-and-drop-images` otherwise)
- Double-esc rewind (`double-esc` or `double-esc-code-restore` with file history)
- PowerShell tool (`powershell-tool-env`, Windows-only)
- Status line (`status-line`)
- Keyboard shortcuts (`shift-tab` for mode cycling, `image-paste`)

#### 5. **Plugin Upsells** (2+ dynamic tips)
- Frontend design (`frontend-design-plugin`, if editing `.html`/`.css` files)
- Vercel (`vercel-plugin`, if `vercel.json` or `vercel` CLI detected)

#### 6. **Pro Features** (3+ gated tips, 1P API customers only)
- Effort high (`effort-high-nudge`, 3-4 session cooldown)
- Subagent fanout (`subagent-fanout-nudge`, 3-4 session cooldown)
- Loop command (`loop-command-nudge`, requires KAIROS cron enabled)

#### 7. **Monetization** (2+ tips)
- Guest passes (`guest-passes`, 3 session cooldown)
- Overage credit (`overage-credit`, 3 session cooldown, gated by feature flag)

#### 8. **Admin Only** (ANT-ONLY, 2 tips)
- IMPORTANT prefix in CLAUDE.md
- `/skillify` to convert workflows to skills

#### 9. **Custom Tips** (user-defined)
- `settings.json.spinnerTipsOverride.tips: string[]`
- Can exclude built-in tips with `excludeDefault: true`

### Selection Logic

**getTipToShowOnSpinner(context?)**:
1. Check if tips disabled in settings (`spinnerTipsEnabled === false`) → skip
2. Call `getRelevantTips(context)` to filter by relevance + cooldown
3. Select tip with **longest time since last shown** (oldest)

**getRelevantTips(context?)**:
1. Load custom tips from settings
2. If `excludeDefault && customTips.length > 0` → return only custom
3. Otherwise: filter built-in tips by:
   - `isRelevant(context)` returns true
   - `getSessionsSinceLastShown(tipId) >= tip.cooldownSessions`
4. Combine with custom tips, return all

**selectTipWithLongestTimeSinceShown(availableTips)**:
- Map tips to (tip, sessionsSinceLast) tuples
- Sort by sessions descending
- Return first (oldest)

### Persistence (tipHistory)

```javascript
tipHistory[tipId] = numStartups  // When last shown
getSessionsSinceLastShown(tipId) = currentNumStartups - lastShown
```

**recordTipShown(tipId)**:
- Save to global config at `~/.claude/config.json`
- Key: `tipsHistory[tipId]`
- Value: current `numStartups` count

**Config location**: `~/.claude/config.json` (via `getGlobalConfig()`)

### Common Relevance Patterns

**Based on user stats**:
```typescript
const config = getGlobalConfig()
config.numStartups          // Total session count
config.memoryUsageCount     // Times /memory used
config.promptQueueUseCount  // Times prompt queue used
config.lastPlanModeUse      // Last timestamp plan mode used
config.lastPlanModeUse ? (Date.now() - lastUse) / (1000*60*60*24) : Infinity
```

**Based on environment**:
```typescript
env.terminal          // 'vscode', 'cursor', 'Apple_Terminal', etc.
getPlatform()         // 'macos', 'windows', 'linux'
env.isSSH()           // Remote session?
```

**Based on tooling detected**:
```typescript
await detectRunningIDEsCached()    // Currently running IDEs
isSupportedTerminal()              // Integrated terminal?
await isVSCodeInstalled()          // IDE installed?
```

**Based on feature flags**:
```typescript
getFeatureValue_CACHED_MAY_BE_STALE('tengu_tide_elm', 'off')  // Effort nudge variants
is1PApiCustomer()                  // Pro customer?
modelSupportsEffort(getMainLoopModel())  // Model supports effort?
```

**Based on file system**:
```typescript
context.readFileState               // Files read (via cacheKeys())
cacheKeys(readFileState).some(fp => /\.html$/.test(fp))
```

### Integration Points

**Entry point**: Shown on spinner during Claude's thinking phase
- Called from UI before each response
- Non-blocking (doesn't slow rendering)
- Recorded via `recordShownTip()` after display

**Analytics**: `logEvent('tengu_tip_shown', { tipIdLength, cooldownSessions })`

---

## 3. AgentSummary: Subagent Progress Monitoring

### Purpose
AgentSummary generates **real-time progress summaries** for coordinator-mode subagents. Every ~30 seconds, it forks the subagent's conversation to generate a 1-2 sentence description of the agent's current action, displayed in the UI for visibility into parallel work.

### Key File
- `agentSummary.ts`: Main summarization loop (~180 lines)

### Architecture

#### Timer-Based Execution

```typescript
startAgentSummarization(
  taskId: string,
  agentId: AgentId,
  cacheSafeParams: CacheSafeParams,
  setAppState: TaskContext['setAppState'],
): { stop: () => void }
```

**Lifecycle**:
1. Called when subagent task starts (from AgentTool)
2. Schedules first summary at SUMMARY_INTERVAL_MS (30 seconds)
3. Each summary completes, triggers next
4. Returns `{ stop }` function to cancel (called on agent exit)

**Timing**: Resets on **completion**, not initiation → prevents overlapping summaries

#### Summarization Prompt

```
Describe your most recent action in 3-5 words using present tense (-ing).
Name the file or function, not the branch.
Do not use tools.

Previous: "{last_summary}" — say something NEW.

Good: "Reading runAgent.ts"
Good: "Fixing null check in validate.ts"
Good: "Running auth module tests"
Good: "Adding retry logic to fetchUser"

Bad (past tense): "Analyzed the branch diff"
Bad (too vague): "Investigating the issue"
Bad (too long): "Reviewing full branch diff and AgentTool.tsx integration"
Bad (branch name): "Analyzed adam/background-summary branch diff"
```

**Format constraints**:
- 3-5 words, present tense (-ing verbs)
- Specific file/function names (not vague)
- Not past tense or branch names
- Previous summary passed to avoid repeats

#### Execution Process

**1. Minimal Transcript Check**
- Read current agent transcript via `getAgentTranscript(agentId)`
- Skip if < 3 messages (not enough context)
- Avoid expensive forks for agents still initializing

**2. Message Filtering**
- Call `filterIncompleteToolCalls()` to clean partial tool calls
- Prevents malformed context in fork

**3. Cache Sharing**
- Drop `forkContextMessages` from closure to prevent pinning old messages
- Rebuild from `getAgentTranscript()` each tick
- Share prompt cache via `cacheSafeParams` (same system, tools, model)

**4. Fork Execution**
```javascript
const result = await runForkedAgent({
  promptMessages: [createUserMessage({ content: buildSummaryPrompt(previousSummary) })],
  cacheSafeParams: forkParams,          // Share main thread cache
  canUseTool: async () => ({
    behavior: 'deny',                   // No tools allowed
    message: 'No tools needed for summary',
    decisionReason: { type: 'other', reason: 'summary only' }
  }),
  querySource: 'agent_summary',
  forkLabel: 'agent_summary',
  overrides: { abortController: summaryAbortController },
  skipTranscript: true,                 // Don't pollute transcript
})
```

**Cache note**: Does NOT set `maxOutputTokens` (would bust cache by mismatching thinking config)

**5. Result Extraction**
- Loop through messages, find first assistant non-error turn
- Extract text blocks, join and trim
- Update UI via `updateAgentSummary(taskId, summaryText, setAppState)`

#### Error Handling

- **Transcript unavailable**: Logged, skipped, next timer scheduled
- **Fork fails**: Error logged, next timer scheduled
- **Stopped**: Flag-based exit (prevents race on abort during in-flight fork)
- **Aborted by user**: Abort controller fired, cleanup on next cycle

### Integration Points

**Called from**: Agent task creation (AgentTool)

**UI display**: Shown in coordinator mode tasks panel (refresh every update)

**Abort**: On agent exit, `stop()` called to cancel pending timer + active fork

---

## 4. toolUseSummary: Tool Batch Summarization

### Purpose
toolUseSummary generates **human-readable summaries** of tool batches for SDK clients and mobile apps. When multiple tools complete, this module calls Haiku to produce a git-commit-subject-style label (~30 chars) describing what was accomplished.

### Key File
- `toolUseSummaryGenerator.ts`: ~110 lines

### Architecture

#### Summarization Function

```typescript
export async function generateToolUseSummary({
  tools: ToolInfo[],           // Array of { name, input, output }
  signal: AbortSignal,         // Cancellation
  isNonInteractiveSession: boolean,  // Context flag
  lastAssistantText?: string,  // Optional user intent
}): Promise<string | null>
```

#### System Prompt

```
Write a short summary label describing what these tool calls accomplished.
It appears as a single-line row in a mobile app and truncates around 30 characters,
so think git-commit-subject, not sentence.

Keep the verb in past tense and the most distinctive noun.
Drop articles, connectors, and long location context first.

Examples:
- Searched in auth/
- Fixed NPE in UserService
- Created signup endpoint
- Read config.json
- Ran failing tests
```

**Style requirements**:
- Past tense verb (primary)
- Distinctive noun (path, function, error type)
- Drop articles, prepositions, verbose context
- ~30 character target (truncates in mobile UI)

#### Input Preparation

1. **Truncate tool representations**:
   - Serialize each tool's input/output to JSON
   - Limit each to 300 chars (avoid huge outputs)
   - Format: `Tool: {name}\nInput: {input}\nOutput: {output}`

2. **Optional context**:
   - If `lastAssistantText` provided, prepend first 200 chars
   - Hint at user intent to Haiku ("User wanted to fix auth bug")

3. **Build user prompt**:
```
User's intent (from assistant's last message): [truncated intent]

Tools completed:

Tool: FileReadTool
Input: {"file_path": "/src/auth.ts"}
Output: {"type": "text", "content": "[... truncated ...]"}

Tool: BashTool
Input: {"command": "npm test"}
Output: "[output truncated at 300 chars]"

Label:
```

#### API Call

```javascript
const response = await queryHaiku({
  systemPrompt: asSystemPrompt([SYSTEM_PROMPT]),
  userPrompt: userPrompt,
  signal: signal,                    // Abort support
  options: {
    querySource: 'tool_use_summary_generation',
    enablePromptCaching: true,       // Cache the system prompt
    agents: [],
    isNonInteractiveSession: isNonInteractiveSession,
    hasAppendSystemPrompt: false,
    mcpTools: [],
  },
})
```

**Model**: Haiku (cheap, fast)

**Cache**: System prompt cached (reused per summary)

#### Result Processing

- Extract text blocks from response
- Join and trim
- Return string or null on empty

#### Error Handling

- **Parse errors**: Return null (non-critical)
- **API errors**: Log (via `logError()`), return null
- **Serialization**: Fallback to `[unable to serialize]` for bad values
- **No impact**: Summary generation never blocks or fails the user flow

### Integration Points

**Called from**: SDK client API when tool batch completes

**Display**: Mobile app single-line row (truncates at ~30 chars)

**Non-blocking**: Failures silently return null (no user-facing errors)

---

## 5. MagicDocs: Auto-Maintained Documentation

### Purpose
MagicDocs is a system for **automatically maintaining markdown documentation** tied to specific code or projects. When a file with a special header (`# MAGIC DOC: [title]`) is detected, a background process periodically updates it with new learnings from the conversation, keeping docs fresh without manual maintenance.

### Key Files
- `magicDocs.ts`: Main orchestrator (~255 lines)
- `prompts.ts`: Update prompt builder (~128 lines)

### Architecture

#### Magic Doc Detection

**Header pattern**: `# MAGIC DOC: [title]` (case-insensitive, start of file)

```typescript
const MAGIC_DOC_HEADER_PATTERN = /^#\s*MAGIC\s+DOC:\s*(.+)$/im
```

**Return value**:
```typescript
type MagicDocInfo = {
  title: string
  instructions?: string  // Optional italicized line after header
}
```

**Instructions line** (optional):
```markdown
# MAGIC DOC: My System Architecture
*Keep this terse and focus on integration points*

## Section A
...
```

Italics on the line immediately after header (allowing one blank line) are treated as custom instructions for the update agent.

#### Registration and Tracking

**Trigger**: File is read (via FileReadTool)

**Flow**:
1. FileReadListener detects read
2. Call `detectMagicDocHeader(content)`
3. If match, call `registerMagicDoc(filePath)`
4. Track in `trackedMagicDocs: Map<string, MagicDocInfo>`

**One-time registration**: `registerMagicDoc()` checks if already tracked, doesn't re-register

#### Background Update Process

**Trigger**: Post-sampling hook fires (after assistant turn completes)

**Conditions**:
- `querySource === 'repl_main_thread'` (skip agent turns, dreams, etc.)
- `!hasToolCallsInLastAssistantTurn(messages)` (update only when idle)
- `trackedMagicDocs.size > 0` (at least one doc tracked)

**Execution** (`updateMagicDoc(docInfo, context)`):

1. **Isolate file state**:
   - Clone `FileStateCache` to prevent dedup interference
   - Delete doc's entry so FileReadTool fetches fresh content
   - Create cloned `ToolUseContext` with new cache

2. **Read current doc**:
   - Call `FileReadTool` with cloned context
   - If ENOENT or unreadable → untrack and return
   - Extract text content

3. **Re-detect header**:
   - Call `detectMagicDocHeader(currentContent)`
   - If no header → untrack and return (removed by user)
   - Extract latest title + instructions

4. **Build update prompt**:
   - Call `buildMagicDocsUpdatePrompt()`
   - Pass current content, path, title, instructions

5. **Constrain tools**:
   - Create `canUseTool` callback
   - Allow ONLY `FILE_EDIT_TOOL_NAME` on this doc's path
   - Deny all other tools

6. **Fork update agent**:
   - `runAgent()` with forked context
   - Agent definition: model = 'sonnet', tools = [FILE_EDIT_TOOL_NAME]
   - Use cloned tool context
   - Override system prompt, user/system context

**Sequential execution**: Uses `sequential()` wrapper to ensure updates don't overlap

#### Update Prompt

**Location**: Load from `~/.claude/magic-docs/prompt.md` (fallback to default)

**Default template** (from prompts.ts):

```
IMPORTANT: This message and these instructions are NOT part of the actual user conversation.
Do NOT include references to "documentation updates", "magic docs", or these instructions in the document.

Based on the user conversation above (EXCLUDING this documentation update instruction message),
update the Magic Doc file to incorporate any NEW learnings, insights, or information that would be valuable to preserve.

File: {{docPath}}
Current contents:
<current_doc_content>
{{docContents}}
</current_doc_content>

Document title: {{docTitle}}
{{customInstructions}}

Your ONLY task: use the Edit tool to update if substantial new information exists, then stop.
You can make multiple edits in parallel (single message). If nothing substantial, respond without tools.

CRITICAL RULES:
- Preserve Magic Doc header exactly: # MAGIC DOC: {{docTitle}}
- Preserve italicized line after header (if present)
- Keep document CURRENT with latest codebase state (NOT a changelog)
- Update IN-PLACE to reflect current state (NO historical notes)
- Remove/replace outdated information rather than appending
- Delete sections no longer relevant
- Fix typos, grammar, broken formatting, wrong information
- Keep well-organized: clear headings, logical order, consistent formatting

DOCUMENTATION PHILOSOPHY:
- BE TERSE. High signal only.
- Focus on: WHY things exist, HOW components connect, WHERE to start reading, WHAT patterns used
- Skip: detailed implementation steps, exhaustive API docs, play-by-play narratives
- Document: high-level architecture, non-obvious patterns, key entry points, design decisions, critical dependencies
- Don't document: obvious code, exhaustive lists, step-by-step details, low-level mechanics

REMEMBER: Only update if substantial new information. Header must remain unchanged.
```

**Variable substitution** ({{varName}} syntax):
- `{{docContents}}`: Current file content
- `{{docPath}}`: File path
- `{{docTitle}}`: Magic Doc title
- `{{customInstructions}}`: Formatted instructions block (if provided)

**Custom instructions block** (if instructions exist):
```
DOCUMENT-SPECIFIC UPDATE INSTRUCTIONS:
The document author has provided specific instructions for how this file should be updated.
Pay extra attention and follow carefully:

"{instructions}"

These instructions take priority over general rules below.
```

#### Prompt Loading

**Path**: `${getClaudeConfigHomeDir()}/magic-docs/prompt.md`

Example: `~/.claude/magic-docs/prompt.md`

**Fallback**: If file doesn't exist or read fails, use default template

**Single-pass substitution**: Regex replace `/\{\{(\w+)\}\}/g` with variable lookup
- Avoids $ backreference corruption
- Prevents double-substitution if user content contains `{{varName}}`

### ANT-ONLY Feature

MagicDocs only initializes in ANT-ONLY mode (test environment):

```typescript
export async function initMagicDocs(): Promise<void> {
  if (process.env.USER_TYPE === 'ant') {
    registerFileReadListener(...)
    registerPostSamplingHook(...)
  }
}
```

**Why ANT-only**: Feature still in preview/testing; not released to production

### Integration Points

**Initialization**: Called from startup (likely `backgroundHousekeeping`)

**File detection**: Hooks into `FileReadTool` to detect reads

**Update trigger**: Post-sampling hook fires after each assistant turn (idle condition)

**Tool usage**: Uses forked agent with restricted tool access (FILE_EDIT_TOOL_NAME only)

**Cleanup**: `clearTrackedMagicDocs()` callable to reset state (for tests)

---

## Cross-Cutting Concerns

### Prompt Caching

**autoDream**: Shares cache via `createCacheSafeParams(context)`
- Reuses system prompt, tools, model across consolidated dream runs
- Cost savings on multi-session consolidations

**AgentSummary**: Shares cache via same `cacheSafeParams`
- Reuses main thread's prompt cache for subagent summaries
- Lightweight summaries piggyback on existing session cache

**toolUseSummary**: Caches system prompt at Haiku level
- `enablePromptCaching: true` in options
- Reused across multiple tool batch summaries

**MagicDocs**: No explicit cache sharing (runs via runAgent which handles it)

### Abort/Cancellation

**autoDream**:
- User kill from bg-tasks dialog → `abortController.signal.aborted`
- Check prevents double-rollback

**AgentSummary**:
- Maintains `summaryAbortController` for in-flight fork
- `stop()` function aborts + clears timer
- Flag-based stopped check prevents race conditions

**MagicDocs**:
- No explicit abort; runs to completion (fast, lightweight)

**toolUseSummary**:
- Accepts `signal: AbortSignal` parameter
- Passed to `queryHaiku()` for cancellation support

### Error Resilience

**All services log errors but don't fail**:
- autoDream: Logs to `logForDebugging()`, emits analytics
- Tips: Returns empty array on relevance check failure
- AgentSummary: Logs, skips summary, schedules next
- toolUseSummary: Returns null, doesn't block user flow
- MagicDocs: Logs errors, continues with other docs

### Analytics

**autoDream**:
- `tengu_auto_dream_fired`: minHours, sessions_since
- `tengu_auto_dream_completed`: cache stats, sessions_reviewed
- `tengu_auto_dream_failed`: {}

**Tips**:
- `tengu_tip_shown`: tipIdLength, cooldownSessions

**No analytics**: AgentSummary, toolUseSummary, MagicDocs

### Feature Flags (GrowthBook)

**autoDream**:
- `tengu_onyx_plover`: Feature flag + minHours + minSessions config
- Cached via `getFeatureValue_CACHED_MAY_BE_STALE()` (may be stale)

**Tips**:
- `tengu_tide_elm`: Effort high nudge variants ('off', 'copy_a', 'copy_b')
- `tengu_tern_alloy`: Subagent fanout nudge variants
- `tengu_timber_lark`: Loop command nudge variants

**User settings overrides**:
- autoDream: `settings.json.autoDreamEnabled`
- Tips: `settings.json.spinnerTipsOverride` (custom tips, excludeDefault flag)
- Effort level: `settings.json.effortLevel` (persisted user choice)

### State Management

**Global config** (`~/.claude/config.json`):
- Tip history: `tipsHistory: { [tipId]: numStartups }`
- Session count: `numStartups`
- Feature usage: `memoryUsageCount`, `promptQueueUseCount`, `lastPlanModeUse`, etc.
- IDE/setup tracking: `githubActionSetupCount`, `shiftEnterKeyBindingInstalled`, etc.
- Passes/credits: `hasVisitedPasses`

**Memory directory** (`~/.claude-memory/`):
- autoDream lock: `.consolidate-lock` (mtime = lastConsolidatedAt, body = PID)

**File system** (`~/.claude/magic-docs/`):
- Custom MagicDocs prompt: `prompt.md` (optional)

---

## Performance Characteristics

### Per-Turn Costs

**autoDream** (when enabled):
- Feature gate + config fetch: 1 GB cache read
- Time gate: 1 stat call (lock file)
- Total: ~negligible (early exits, cached GB)

**Tips** (when shown):
- Relevance checks: ~5-10 async checks (configurable, cached)
- History lookup: hash table access
- Total: ~milliseconds

**AgentSummary** (per 30 seconds):
- Transcript read: 1 file stat
- Fork execution: Full model call (but short, 3-5 words)
- Total: Model latency (not per-turn, background)

**toolUseSummary** (on tool batch complete):
- Haiku call: Brief model invocation (~100ms)
- System prompt cached
- Total: Model latency (async, non-blocking)

**MagicDocs** (post-sampling hook):
- Tracked docs check: hash table lookup (usually empty)
- On idle: Full forked agent run
- Total: Model latency if triggered (async, non-blocking)

### Concurrency & Locking

**autoDream**:
- PID-based lock prevents concurrent consolidations
- Time/session gates + scan throttle prevent high-frequency probes

**AgentSummary**:
- Timer-based (no concurrency, sequential summaries)
- Abort controller prevents overlapping forks

**MagicDocs**:
- `sequential()` wrapper ensures updates don't overlap
- File state cache cloning prevents interference

---

## Extensibility

### Custom Tips

Users can add tips via `settings.json`:

```json
{
  "spinnerTipsOverride": {
    "tips": [
      "My custom tip 1",
      "My custom tip 2"
    ],
    "excludeDefault": false
  }
}
```

If `excludeDefault: true` and custom tips present, skip all built-in tips.

### Custom MagicDocs Prompt

Users can override MagicDocs behavior via `~/.claude/magic-docs/prompt.md`:

```markdown
[Custom template with {{docContents}}, {{docPath}}, {{docTitle}}, {{customInstructions}} substitution]
```

### Feature Flag Variants

Three-way feature flag pattern for A/B testing (used by effort nudge, subagent fanout, loop command):

```typescript
getFeatureValue_CACHED_MAY_BE_STALE<'off' | 'copy_a' | 'copy_b'>('flag_name', 'off')
```

- `'off'`: Disabled
- `'copy_a'`: Variant A copy
- `'copy_b'`: Variant B copy

---

## Summary Table

| Service | Purpose | Trigger | Frequency | Blocking? | Rollback/Abort? |
|---------|---------|---------|-----------|-----------|-----------------|
| autoDream | Memory consolidation | Post-sampling hook | ~24h + 5 sessions | No | Yes (lock rollback) |
| Tips | User guidance | Before spinner display | Per-startup | No | N/A |
| AgentSummary | Subagent progress | Timer (30s intervals) | Every 30s per agent | No | Yes (abort signal) |
| toolUseSummary | Tool batch labels | On tool batch complete | Per batch | No | No (silent fail) |
| MagicDocs | Doc auto-update | Post-sampling hook (idle) | Variable | No | No (silent fail) |

---

## Testing Notes

**autoDream**: State is closure-scoped in `initAutoDream()` so tests can call per-test for fresh state

**Tips**: `recordTipShown()` and `getSessionsSinceLastShown()` use global config; tests should mock

**AgentSummary**: `startAgentSummarization()` returns stop function; tests can call to halt timer

**toolUseSummary**: Pure async function; testable with mock Haiku calls

**MagicDocs**: `clearTrackedMagicDocs()` callable; register/unregister flow tested separately
