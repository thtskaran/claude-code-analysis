# Claude Code — Implementation Deep-Dive Report

> **Generated**: 2026-03-31
> **Scope**: Line-by-line implementation analysis of critical subsystems
> **Prerequisite**: Phase 1 architectural report (`analysis-report.md`)

> **Self-contained reading note**: code identifiers, file names, and line ranges in this document are provenance. The subsystem explanations are intended to remain useful without access to the original source tree.
> **Reading guidance**: treat the named modules as labels for implementation areas. The behavioral explanations are the actual handoff artifact.

---

## Table of Contents

- [Deep-Dive 1: The Compaction Engine](#deep-dive-1-the-compaction-engine)
- [Deep-Dive 2: The Speculation System](#deep-dive-2-the-speculation-system)
- [Deep-Dive 3: StreamingToolExecutor](#deep-dive-3-streamingtoolexecutor)
- [Deep-Dive 4: The Bash Tool](#deep-dive-4-the-bash-tool)
- [Deep-Dive 5: The Edit Tool](#deep-dive-5-the-edit-tool)
- [Deep-Dive 6: The Agent Tool](#deep-dive-6-the-agent-tool)
- [Deep-Dive 7: The Hook System](#deep-dive-7-the-hook-system)
- [Deep-Dive 8: Session Memory & Context Collapse & Memdir](#deep-dive-8-session-memory--context-collapse--memdir)
- [Deep-Dive 9: Token Budget & Output Token Management](#deep-dive-9-token-budget--output-token-management)
- [Deep-Dive 10: Fast Mode](#deep-dive-10-fast-mode)
- [Deep-Dive 11: Prompt Cache Break Detection](#deep-dive-11-prompt-cache-break-detection)
- [Deep-Dive 12: Error Messages Sent to the Model](#deep-dive-12-error-messages-sent-to-the-model)
- [Deep-Dive 13: State Store & Bridge](#deep-dive-13-state-store--bridge)
- [Deep-Dive 14: Terminal Rendering Engine](#deep-dive-14-terminal-rendering-engine)
- [Deep-Dive 15: Analytics & Telemetry](#deep-dive-15-analytics--telemetry)
- [Implementation Patterns Cheat Sheet](#implementation-patterns-cheat-sheet)

---

## Deep-Dive 1: The Compaction Engine

### 1A: Microcompact (`src/services/compact/microCompact.ts`, 531 lines)

**What is "content-clearing"?** It replaces the `content` field of a `tool_result` block with the literal string `'[Old tool result content cleared]'`. The block structure is preserved — only content is overwritten:

```typescript
// microCompact.ts:475-483
const newContent = message.message.content.map(block => {
  if (
    block.type === 'tool_result' &&
    clearSet.has(block.tool_use_id) &&
    block.content !== TIME_BASED_MC_CLEARED_MESSAGE
  ) {
    tokensSaved += calculateToolResultTokens(block)
    touched = true
    return { ...block, content: TIME_BASED_MC_CLEARED_MESSAGE }
  }
  return block
})
```

The constant (`microCompact.ts:36`):
```typescript
export const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'
```

The guard `block.content !== TIME_BASED_MC_CLEARED_MESSAGE` prevents double-counting tokens on already-cleared blocks. The constant is duplicated (not imported) to break a circular dependency — a test asserts equality with the canonical source.

**Compactable tool whitelist** (`microCompact.ts:41-50`):

```typescript
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,     // 'Read'
  ...SHELL_TOOL_NAMES,     // 'Bash', 'PowerShell'
  GREP_TOOL_NAME,          // 'Grep'
  GLOB_TOOL_NAME,          // 'Glob'
  WEB_SEARCH_TOOL_NAME,    // 'WebSearch'
  WEB_FETCH_TOOL_NAME,     // 'WebFetch'
  FILE_EDIT_TOOL_NAME,     // 'Edit'
  FILE_WRITE_TOOL_NAME,    // 'Write'
])
```

The "last 5" comes from `keepRecent` in the GrowthBook config, default 5 (`timeBasedMCConfig.ts:33`). The algorithm: collect all compactable tool IDs in order, `slice(-keepRecent)` to keep the last N, everything else goes into `clearSet`.

**The `cache_edits` API path** — this is the cached microcompact mechanism. Instead of mutating message content client-side, it tells the API to delete specific tool results from the server-side cache:

```typescript
// Type (claude.ts:3052-3055)
type CachedMCEditsBlock = {
  type: 'cache_edits'
  edits: { type: 'delete'; cache_reference: string }[]
}
```

Each `tool_result` block in the cached prefix gets tagged with `cache_reference: tool_use_id`. A `cache_edits` block with `{ type: 'delete', cache_reference: '<id>' }` entries is spliced into a user message. The server removes the matching cached content without rewriting the entire prefix. The client-side message array is **not mutated**.

**60-minute gap detection** (`microCompact.ts:422-444`):

```typescript
const lastAssistant = messages.findLast(m => m.type === 'assistant')
const gapMinutes = (Date.now() - new Date(lastAssistant.timestamp).getTime()) / 60_000
if (!Number.isFinite(gapMinutes) || gapMinutes < config.gapThresholdMinutes) {
  return null
}
```

Compares `Date.now()` to the timestamp of the last assistant message. Default threshold: 60 minutes. Rationale: the server's 1-hour cache TTL has expired, so the prefix will be rewritten anyway.

**`tengu_slate_heron` interaction** — this is a GrowthBook feature flag (not compile-time). When `enabled: true`, the time-based path fires first and **short-circuits** cached microcompact (the cache is cold, editing assumes a warm cache). When `enabled: false` (default), time-based MC is skipped and cached MC handles context management.

### Surprises

- The constant is duplicated to break a circular dependency, with a test ensuring sync — a pragmatic but unusual pattern.
- Cached microcompact uses `cache_edits` which is an internal/beta Anthropic API feature, not publicly documented.

---

### 1B: Full Compaction (`src/services/compact/compact.ts`, ~60KB)

**The compaction prompt** is assembled in layers. The complete base prompt (`prompt.ts:61-143`) has 9 sections:

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
[... NO_TOOLS_PREAMBLE ...]

Before providing your final summary, wrap your analysis in <analysis> tags...
1. Chronologically analyze each message... identify:
   - The user's explicit requests and intents
   - Your approach to addressing the user's requests
   - Key decisions, technical concepts and code patterns
   - Specific details like: file names, full code snippets, function signatures, file edits
   - Errors that you ran into and how you fixed them
   - Pay special attention to specific user feedback

Your summary should include:
1. Primary Request and Intent - What was the user trying to accomplish?
2. Key Technical Concepts - What technologies, patterns, and approaches were discussed?
3. Files and Code Sections - What specific files were examined/modified?
4. Errors and Fixes - Document each error and its fix
5. Problem Solving - What was the overall approach?
6. All user messages that are not tool results
7. Pending Tasks - What is not yet complete?
8. Current Work - Where did the conversation leave off?
9. Optional Next Step - What would logically come next?

REMINDER: Do NOT call any tools. Respond with plain text only.
```

The system prompt for the summarizer: `'You are a helpful AI assistant tasked with summarizing conversations.'`

**Message selection**: Full compaction summarizes ALL messages. There is no "keep last N verbatim" in the full path. Partial compaction uses a `pivotIndex` to split: `direction === 'up_to'` summarizes before pivot and keeps after; `direction === 'from'` summarizes after pivot and keeps before.

**API-round grouping** (`grouping.ts:22-63`):

```typescript
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  const groups: Message[][] = []
  let current: Message[] = []
  let lastAssistantId: string | undefined

  for (const msg of messages) {
    if (msg.type === 'assistant' && msg.message.id !== lastAssistantId && current.length > 0) {
      groups.push(current)
      current = [msg]
    } else {
      current.push(msg)
    }
    if (msg.type === 'assistant') { lastAssistantId = msg.message.id }
  }
  if (current.length > 0) { groups.push(current) }
  return groups
}
```

A new group starts when an assistant message has a different `message.id` from the previous one. Streaming chunks from the same API response share an id and stay grouped.

**Retry-with-dropping** (`compact.ts:243-291`): On prompt-too-long, `truncateHeadForPTLRetry` drops oldest groups. If the token gap is parseable, it drops groups until accumulated tokens ≥ gap. Otherwise drops 20% of groups. Max 3 retries (`MAX_PTL_RETRIES`).

**Post-compaction file restoration** — files selected by **most recently accessed** (sorted by `readFileState` timestamp), not most frequently referenced:

```typescript
// compact.ts:1415-1463
const recentFiles = Object.entries(readFileState)
  .map(([filename, state]) => ({ filename, ...state }))
  .filter(file => !shouldExcludeFromPostCompactRestore(...) && !preservedReadPaths.has(...))
  .sort((a, b) => b.timestamp - a.timestamp)  // Most recent first
  .slice(0, maxFiles)  // POST_COMPACT_MAX_FILES_TO_RESTORE = 5
```

Token budget: 50K total, 5K per file (`POST_COMPACT_TOKEN_BUDGET`, `POST_COMPACT_MAX_TOKENS_PER_FILE`).

**Summary injection**: The summary becomes a **user message** with `isCompactSummary: true`, placed after a system compact boundary marker:

```
[SystemCompactBoundaryMessage]
[UserMessage: "This session is being continued from a previous conversation...
  Summary: <formatted summary>
  If you need specific details, read the full transcript at: <path>"]
[Preserved messages (partial only)]
[File attachments]
[Hook results]
```

**Image stripping**: `block.type === 'image'` → `{ type: 'text', text: '[image]' }`. Same for `block.type === 'document'` → `'[document]'`. Applied at both top-level content blocks and nested inside `tool_result` content arrays.

**Cache interaction**: After compaction, `notifyCompaction()` resets the cache-read baseline so the inevitable cache miss isn't flagged as anomalous.

### Surprises

- The `<analysis>` tags in the compaction prompt are **stripped** from the final summary — they exist purely to improve summarization quality via chain-of-thought, then are discarded.
- The "no tools" constraint is enforced both at the prompt level (instructions) AND via the API request having empty tools array.

---

### 1C: Session Memory Compaction (`src/services/compact/sessionMemoryCompact.ts`)

Session Memory is an incrementally-maintained markdown file with structured sections (title, current state, task specification, files, workflow, errors, learnings, etc.). Stored at `~/.claude/session-memory/`. The template (`DEFAULT_SESSION_MEMORY_TEMPLATE`) has sections capped at 2000 tokens each, 12000 total.

**How it differs from full compaction**: Session memory compaction uses the pre-built session memory AS the summary — it does NOT call the LLM at compaction time. The summary was already built incrementally via background forked agents. It then keeps recent unsummarized messages verbatim (min 10K tokens, min 5 messages, max 40K tokens).

**Interaction with CLAUDE.md**: The extraction prompt says `"Do not include information that's already in the CLAUDE.md files"`. After SM compaction, `processSessionStartHooks('compact')` restores CLAUDE.md and other context.

---

### 1D: Context Collapse (`marble_origami`)

Context collapse is a **read-time projection** — original messages stay in the REPL array; collapsed sections are replaced by summary placeholders only when constructing the API payload. Feature flag: `CONTEXT_COLLAPSE` (build-time) / `marble_origami` (codename).

**Key difference from compaction**: Compaction replaces history with a single summary (destructive). Collapse archives message ranges with summary placeholders (non-destructive, reversible).

**Thresholds**: ~90% of effective context = start committing staged collapses. ~95% = blocking spawn. Autocompact at ~93% is **suppressed** when collapse is active to prevent it from racing collapse.

**Persistence**: Collapse state is persisted as append-only `ContextCollapseCommitEntry` log entries in the JSONL transcript. On resume, `restoreFromEntries` replays the commit log.

---

## Deep-Dive 2: The Speculation System

**Location**: `src/services/PromptSuggestion/speculation.ts` (691 lines)

### The Overlay Filesystem

There is **no real OverlayFS**. It's a userspace file-level intercept via a `canUseTool` callback. The overlay is a plain temp directory:

```typescript
// speculation.ts:80-82
function getOverlayPath(id: string): string {
  return join(getClaudeTempDir(), 'speculation', String(process.pid), id)
}
```

**Copy-on-write mechanism**: The `canUseTool` callback (lines 461-632) intercepts every tool call. For write tools (Edit, Write, NotebookEdit):

```typescript
// speculation.ts:528-539
if (isWriteTool) {
  if (!writtenPathsRef.current.has(rel)) {
    const overlayFile = join(overlayPath, rel)
    await mkdir(dirname(overlayFile), { recursive: true })
    try { await copyFile(join(cwd, rel), overlayFile) } catch { /* new file */ }
    writtenPathsRef.current.add(rel)
  }
  input = { ...input, [pathKey]: join(overlayPath, rel) }
}
```

For reads: if the file was previously written in speculation, redirect to overlay; otherwise read from real filesystem.

### What Gets Speculated

The system predicts the **user's next message**, then pre-computes the model's response. Pipeline:

1. After each turn, `executePromptSuggestion` runs a forked agent with a suggestion prompt: *"Predict what THEY would type... THE TEST: Would they think 'I was just about to type that'?"*
2. If suggestion generated, `startSpeculation` runs a full forked agent loop as if the user typed that text
3. Up to `MAX_SPECULATION_TURNS = 20` turns and `MAX_SPECULATION_MESSAGES = 100` messages

### Boundary Detection

```typescript
type CompletionBoundary =
  | { type: 'complete'; completedAt: number; outputTokens: number }
  | { type: 'bash'; command: string; completedAt: number }
  | { type: 'edit'; toolName: string; filePath: string; completedAt: number }
  | { type: 'denied_tool'; toolName: string; detail: string; completedAt: number }
```

Speculation stops when:
1. **File edit requiring permission** (non-acceptEdits mode) → `'edit'` boundary
2. **Non-read-only bash command** → `'bash'` boundary
3. **Any other non-safe tool** (WebFetch, MCP, TaskCreate, etc.) → `'denied_tool'` boundary
4. **Message count limit** (100 messages) → abort
5. **Turn limit** (20 turns) → abort
6. **Natural completion** → `'complete'` boundary

### Commit/Rollback

**Acceptance** (`acceptSpeculation`): `copyOverlayToMain` copies overlay files back to real filesystem (atomic per-file via `copyFile`). Then `safeRemoveOverlay` deletes the overlay.

**Rejection** (`abortSpeculation`): `safeRemoveOverlay` deletes the overlay with `rm -rf`. Real filesystem is never touched.

### Cache Sharing

The entire system is designed for **prompt cache sharing** between speculation and the main loop. `CacheSafeParams` captures the exact system prompt, tools, model, and message prefix. Comment in `promptSuggestion.ts`:

```
// DO NOT override any API parameter that differs from the parent request.
// The fork piggybacks on the main thread's prompt cache by sending identical
// cache-key params. PR #18143 tried effort:'low' and caused a 45x spike in
// cache writes (92.7% → 61% hit rate).
```

For prompt suggestion: `skipCacheWrite: true` (reads cache but doesn't write, since no future request will extend that prefix).

### User Experience

- **Before acceptance**: Suggestion shown as dimmed placeholder text in the input field
- **Speculation runs invisibly** — no visual indicator
- **Accept**: Tab key or Enter with empty input
- **Reject**: Any keystroke aborts speculation and clears the suggestion
- Currently gated to `USER_TYPE === 'ant'` (Anthropic employees only)

### Surprises

- Speculation is essentially **database-style optimistic execution with lazy commit semantics** — a technique rarely seen in AI agents.
- `requireCanUseTool: true` flag is critical — it forces the canUseTool callback even when hooks would auto-approve, ensuring path rewriting always happens.

---

## Deep-Dive 3: StreamingToolExecutor

**Location**: `src/services/tools/StreamingToolExecutor.ts`

### Queue Data Structure

Simple array (`TrackedTool[]`), not priority queue. Tools ordered by insertion order (streaming order from API):

```typescript
private tools: TrackedTool[] = []

type TrackedTool = {
  id: string; block: ToolUseBlock; assistantMessage: AssistantMessage;
  status: ToolStatus; isConcurrencySafe: boolean;
  promise?: Promise<void>; results?: Message[];
  pendingProgress: Message[];
  contextModifiers?: Array<(context: ToolUseContext) => ToolUseContext>;
}

type ToolStatus = 'queued' | 'executing' | 'completed' | 'yielded'
```

### Concurrency-Safe Classification Table

| Tool | Concurrent-Safe? | Condition |
|---|---|---|
| **Read, Glob, Grep, WebFetch, WebSearch** | Always `true` | Read-only tools |
| **Agent, LSP, ToolSearch, Config, Brief** | Always `true` | Independent execution |
| **AskUserQuestion, EnterPlanMode, ExitPlanMode** | Always `true` | UI tools |
| **TaskCreate/Get/List/Update/Stop** | Always `true` | Task management |
| **RemoteTrigger, CronList, MCP Resources** | Always `true` | Remote/listing tools |
| **Bash** | Conditional | `true` only if `isReadOnly(input)` passes |
| **PowerShell** | Conditional | `true` only if `isReadOnlyCommand(input.command)` |
| **TaskOutput** | Always `true` | `isReadOnly` returns `true` |
| **Edit, Write, NotebookEdit** | Always `false` | File mutation tools |
| **Skill, SendMessage, TodoWrite** | Always `false` | Side-effect tools |
| **Worktree, Team, Cron mutation** | Always `false` | State-changing tools |
| **MCP tools** | Conditional | `true` only if server declares `readOnlyHint: true` |

### State Machine

```
addTool()        processQueue()         executeTool finishes     getCompletedResults()
(none) ──────> queued ──────────────> executing ──────────────> completed ──────────────> yielded
```

The concurrency gate (`canExecuteTool`):
```typescript
private canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executingTools = this.tools.filter(t => t.status === 'executing')
  return executingTools.length === 0 ||
    (isConcurrencySafe && executingTools.every(t => t.isConcurrencySafe))
}
```

### Streaming Overlap

Yes — tool execution **starts during streaming**. When a `tool_use` block completes in the stream, `addTool()` is called immediately, which triggers `processQueue()` (non-awaited). Concurrent-safe tools start executing while the model continues generating subsequent tool_use blocks.

### Abort Controller Tree

```
toolUseContext.abortController       (L0 — query/REPL)
    └── siblingAbortController       (L1 — per-executor)
        ├── toolAbortController[0]   (L2 — per-tool)
        ├── toolAbortController[1]
        └── toolAbortController[N]
```

Uses `WeakRef` for memory-safe parent-child linking. **Only Bash errors** trigger sibling abort (kills peer tools). Other tool errors do not cascade.

### Progress Fast Path

Progress messages bypass the ordering barrier — yielded immediately for ALL tools regardless of position:

```typescript
// Always yield pending progress messages immediately, regardless of tool status
while (tool.pendingProgress.length > 0) {
  const progressMessage = tool.pendingProgress.shift()!
  yield { message: progressMessage, newContext: this.toolUseContext }
}
```

### Surprises

- The `break` vs `continue` distinction in `processQueue` is subtle and critical: non-concurrent tools act as barriers; concurrent tools behind them can still be skipped.
- Results are emitted in insertion order, not completion order — this is essential for deterministic conversation history.

---

## Deep-Dive 4: The Bash Tool

**Location**: `src/tools/BashTool/BashTool.tsx`, `src/utils/Shell.ts`, `src/utils/ShellCommand.ts`

### Shell Spawning

Only **bash** or **zsh** supported. Detection via `findSuitableShell()`. **New process per command** via `spawn()`:

```typescript
const childProcess = spawn(spawnBinary, shellArgs, {
  env: { ...subprocessEnv(), SHELL: binShell, GIT_EDITOR: 'true', CLAUDECODE: '1', ... },
  cwd,
  stdio: ['pipe', outputHandle?.fd, outputHandle?.fd],
  detached: true,
  windowsHide: true,
})
```

Shell args: `bash -c [-l] <command>`. Login shell (`-l`) used on first run, skipped after shell snapshot captured.

### Environment

Key env vars set: `GIT_EDITOR: 'true'` (prevents editor popups), `CLAUDECODE: '1'` (marker), `TMUX` isolated to Claude's socket, `TMPDIR` set to sandbox temp.

Env vars scrubbed in GHA: `ANTHROPIC_API_KEY`, `AWS_SECRET_ACCESS_KEY`, `ACTIONS_RUNTIME_TOKEN`, and ~20 more secrets.

Shell profile sourced from a saved **snapshot** (captured at startup). Extended globs explicitly disabled for security: `shopt -u extglob` (bash) / `setopt NO_EXTENDED_GLOB` (zsh).

### Timeout

Default: **30 minutes**. On timeout, goes **straight to SIGKILL via tree-kill** — no SIGTERM-then-SIGKILL escalation:

```typescript
#doKill(code?: number): void {
  this.#status = 'killed'
  if (this.#childProcess.pid) { treeKill(this.#childProcess.pid, 'SIGKILL') }
  this.#resolveExitCode(code ?? SIGKILL)
}
```

### Output Capture

Both stdout and stderr go to the **same file fd** (O_APPEND for atomic interleaving). Progress extracted by polling file tail every 1 second (4096 bytes).

### Output Truncation

**NOT the "5K start + 5K end" pattern described in Phase 1.** The actual implementation:

- **Layer 1** (`EndTruncatingAccumulator`): Keeps the start, drops the end. Max 30K chars default (`BASH_MAX_OUTPUT_DEFAULT`), upper limit 150K (`BASH_MAX_OUTPUT_UPPER_LIMIT`).
- **Layer 2** (`formatOutput`): Truncates to `maxOutputLength` chars from the start, appends `"... [N lines truncated] ..."`.

The 5K+5K pattern exists only in `formatError()` for **error messages** (not tool output): `fullMessage.slice(0, 5000) + '...' + fullMessage.slice(-5000)`.

### Working Directory Persistence

The shell provider appends `pwd -P >| <cwdFilePath>` to every command. After completion, the cwd is read back from that temp file:

```typescript
let newCwd = readFileSync(nativeCwdFilePath, { encoding: 'utf8' }).trim()
if (newCwd.normalize('NFC') !== cwd) { setCwd(newCwd, cwd) }
```

### YOLO/Bash Classifier

The bash classifier is a **stub in the external build** — returns `{ matches: false }` always. The YOLO classifier is behind `feature('TRANSCRIPT_CLASSIFIER')` and performs a side-query to a Claude model with the current transcript to decide allow/deny.

### Surprises

- No SIGTERM grace period — immediate SIGKILL. Aggressive but prevents zombie processes.
- Extended glob patterns explicitly disabled as a security measure against injection.

---

## Deep-Dive 5: The Edit Tool

**Location**: `src/tools/FileEditTool/FileEditTool.ts`, `src/tools/FileEditTool/utils.ts`

### Matching Algorithm

**Exact string search** via `String.includes()` / `String.indexOf()`, with curly-quote normalization fallback:

```typescript
export function findActualString(fileContent: string, searchString: string): string | null {
  if (fileContent.includes(searchString)) { return searchString }
  const normalizedSearch = normalizeQuotes(searchString)
  const normalizedFile = normalizeQuotes(fileContent)
  const searchIndex = normalizedFile.indexOf(normalizedSearch)
  if (searchIndex !== -1) { return fileContent.substring(searchIndex, searchIndex + searchString.length) }
  return null
}
```

### Uniqueness Check

```typescript
const matches = file.split(actualOldString).length - 1
if (matches > 1 && !replace_all) {
  return { result: false, message: `Found ${matches} matches...` }
}
```

### replace_all Mode

Uses `String.replaceAll()` with `() => replace` callback to prevent `$1` interpretation:

```typescript
const f = replaceAll
  ? (content, search, replace) => content.replaceAll(search, () => replace)
  : (content, search, replace) => content.replace(search, () => replace)
```

### Concurrent Edit Safety

**No file locking.** Uses read-before-write staleness check:
1. File must have been read first (`readFileState` check)
2. At write time, mtime compared to read timestamp
3. If modified since read: error unless content is identical
4. Comment warns: "avoid async operations between here and writing to disk to preserve atomicity"

### Undo/Rollback

Integrates with **file history / checkpoint system** via `fileHistoryTrackEdit()` — creates a backup keyed by message UUID before the edit.

### Large File Handling

Hard limit: `MAX_EDIT_FILE_SIZE = 1 GiB`. Files above this are rejected with error message.

### Encoding Detection

BOM check: `0xFF 0xFE` → `utf16le`, otherwise `utf8`. Line endings detected from first 4096 bytes (`\r\n` vs `\n`). Original encoding and line endings preserved on write.

### De-sanitization

Before applying edits, known API sanitization patterns are reversed:

```typescript
const DESANITIZATIONS: Record<string, string> = {
  '<fnr>': '<function_results>',
  '<n>': '<name>',
  '\n\nH:': '\n\nHuman:',
  '\n\nA:': '\n\nAssistant:',
  // ... many more
}
```

### Surprises

- The curly-quote normalization fallback is clever — handles the common case where the model generates `"smart quotes"` but the file has `"straight quotes"`.
- De-sanitization table reveals API-level string transformations that the tool must undo.

---

## Deep-Dive 6: The Agent Tool

**Location**: `src/tools/AgentTool/AgentTool.tsx`, `src/tools/AgentTool/runAgent.ts`, `src/coordinator/`

### Subagent Lifecycle

Each invocation starts a **brand new `query()` loop** — a fresh multi-turn API conversation. Normal agents get a single user message with the prompt. Fork agents inherit the parent's full conversation context.

**System prompt**: Each agent type has its own `getSystemPrompt()`. Fork children inherit the parent's exact rendered system prompt for cache sharing.

**Tools**: Resolved via `resolveAgentTools()` — parent's tools filtered through agent's `tools`/`disallowedTools` fields. `ALL_AGENT_DISALLOWED_TOOLS` removes TaskOutput, ExitPlanMode, EnterPlanMode, AskUserQuestion, TaskStop, and (for non-ant users) **Agent itself**.

**Model**: Priority: explicit param > agent definition > default subagent model > parent's model.

**Thinking**: Regular subagents have thinking **disabled**. Fork children inherit parent's thinking config.

### Built-In Agent Types

| Type | Model | Tools | Key Trait |
|---|---|---|---|
| `general-purpose` | Default subagent | All (`['*']`) | Full capabilities |
| `Explore` | Haiku (ext) / inherit (ant) | Read-only (no Edit/Write/Agent) | Fast codebase exploration |
| `Plan` | Inherit | Read-only | Architecture/planning specialist |
| `verification` | Inherit | Read-only, background=true | Runs tests, verifies changes |
| `claude-code-guide` | Haiku | Read+Glob+Grep+Web | Documentation expert |
| `statusline-setup` | Sonnet | Read+Edit | Status line configuration |
| `fork` | Inherit | Parent's exact pool | Full parent context, `permissionMode: 'bubble'` |

### Coordinator Pattern

Not a thread pool or actor model — it's a **system prompt transformation + agent forcing** pattern:
1. Main agent's system prompt replaced with coordinator instructions
2. All spawns forced async (fire-and-forget)
3. Workers complete → `<task-notification>` XML injected as user message
4. Coordinator has only: Agent, SendMessage, TaskStop tools

### Inter-Agent Communication

**Running agents**: Messages queued via `queuePendingMessage()`, delivered at next tool round.
**Stopped agents**: Auto-resumed from disk transcript with new message.
**Teammates**: File-based mailbox system (message passing via filesystem).

### Nesting Limits

**External users**: Cannot nest — Agent tool is in the disallow list for subagents.
**Ant users**: Can nest (Agent tool not disallowed). Fork children have a specific recursive guard.
**No hard depth limit** — prevention via tool disallow lists, fork guards, maxTurns limits, and AbortController propagation.

### Worktree Isolation

When `isolation: "worktree"`, creates a git worktree via `git worktree add`. Agent runs with cwd overridden to worktree path. On completion: if no changes, worktree deleted automatically. If changes exist, worktree path/branch returned in result. Symlinks node_modules to avoid disk bloat.

### Surprises

- Fork agents inherit the parent's EXACT prompt cache — `CacheSafeParams` enforces this. Cache sharing across agent boundaries is a major optimization.
- The `permissionMode: 'bubble'` means fork child permission prompts surface in the parent's terminal — a clever UX for transparent delegation.

---

## Deep-Dive 7: The Hook System

**Location**: `src/utils/hooks.ts` (~2000 lines), `src/schemas/hooks.ts`

### Hook Types

1. **Command**: Shell subprocess. Exit code 0 = success, 2 = blocking error (stderr shown to model), other = non-blocking.
2. **Prompt**: LLM side-query with JSON schema output (`{ok: true}` or `{ok: false, reason: "..."}`)
3. **Agent**: Full multi-turn verifier agent (max 50 turns) with tools. System prompt instructs it to read the transcript and verify conditions.
4. **HTTP**: POST to URL with header env-var interpolation and `allowedEnvVars` security list.
5. **Function**: In-memory TypeScript callback (session-only, not persisted).

### Hook Output Injection

`<user-prompt-submit-hook>` output is wrapped in `<system-reminder>` and injected as a meta user message:

```typescript
createUserMessage({
  content: wrapInSystemReminder(
    `${attachment.hookName} hook additional context: ${attachment.content.join('\n')}`
  ),
  isMeta: true,
})
```

### Input Modification

Hooks CAN modify tool input via the `updatedInput` field in `hookSpecificOutput`. The PreToolUse hook receives the full tool name and input, and can return modified input that replaces the original.

### Security

All hooks require workspace trust. HTTP hooks require explicit `allowedEnvVars` list. The agent hook type's prompt explicitly says it cannot write files (read-only tools only).

---

## Deep-Dive 8: Session Memory & Context Collapse & Memdir

### Session Memory

Incrementally maintained markdown file with structured sections (title, current state, task spec, files, workflow, errors, learnings, key results, worklog). Updated by background forked agents when token/tool-call thresholds are met. Max 2000 tokens per section, 12000 total.

### Context Collapse (`marble_origami`)

A **read-time projection** system. Original messages stay in memory; collapsed sections become `<collapsed id="...">summary</collapsed>` placeholders in the API payload only. Uses staged queue with risk scores. Commits at ~90% context, blocks at ~95%. State persisted as append-only commit log in JSONL transcript.

### Memdir (Auto Memory)

Persistent file-based memory at `~/.claude/projects/<sanitized-git-root>/memory/`. Individual markdown files with YAML frontmatter (type: user/feedback/project/reference). `MEMORY.md` index always loaded into context (capped at 200 lines/25KB). Relevant memories selected via LLM side-query (Sonnet, up to 5 memories). Staleness warnings added for memories >1 day old.

### Surprises

- Session memory compaction replaces traditional compaction entirely — no LLM call at compaction time because the summary was pre-built incrementally.
- Context collapse is non-destructive (messages are archived, not deleted) while compaction is destructive.
- Memdir uses an LLM to SELECT which memories to recall — not keyword matching.

---

## Deep-Dive 9: Token Budget & Output Token Management

### max_tokens Resolution

```typescript
// claude.ts:1591-1594
const maxOutputTokens =
  retryContext?.maxTokensOverride ||      // Recovery/escalation override
  options.maxOutputTokensOverride ||       // Caller override
  getMaxOutputTokensForModel(options.model) // Default per model
```

Defaults: Opus 4.6 = 64K, Sonnet 4.6 = 32K. Capped default: 8K (`CAPPED_DEFAULT_MAX_TOKENS`). Upper limits: Opus/Sonnet 4.6 = 128K, older = 64K.

### 8K→64K Escalation

Triggered when `stop_reason === 'max_tokens'` AND the capped 8K default was used AND no env override:

```typescript
// query.ts:1185-1221
if (capEnabled && maxOutputTokensOverride === undefined && !process.env.CLAUDE_CODE_MAX_OUTPUT_TOKENS) {
  state = { ...state, maxOutputTokensOverride: ESCALATED_MAX_TOKENS, transition: { reason: 'max_output_tokens_escalate' } }
  continue
}
```

Fires once per turn (guarded by `maxOutputTokensOverride === undefined`).

### Recovery Message (verbatim)

```
Output token limit hit. Resume directly — no apology, no recap of what you were doing.
Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.
```

Up to 3 recovery attempts per turn (`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`).

### Thinking Token Budget

- **Adaptive** (Opus 4.6, Sonnet 4.6): `{ type: 'adaptive' }` — no explicit budget
- **Budget-based** (older models): `budget_tokens = min(maxOutputTokens - 1, modelUpperLimit - 1)`
- For Opus 4.6 non-adaptive: 127,999 tokens

### Token Budget Continuation

User-requested budgets (e.g., "+500k") continue until 90% of target. Diminishing returns detected when 3+ continuations have <500 token deltas. Nudge message: `"Stopped at N% of token target (X / Y). Keep working — do not summarize."`

---

## Deep-Dive 10: Fast Mode

Fast mode is the **same Opus 4.6 model with `speed: 'fast'` parameter**. It does NOT change the model. Only supported for first-party Anthropic API, Opus 4.6, paid subscriptions.

### Cooldown State Machine

```typescript
type FastModeRuntimeState =
  | { status: 'active' }
  | { status: 'cooldown'; resetAt: number; reason: CooldownReason }

type CooldownReason = 'rate_limit' | 'overloaded'
```

Triggered by rate limits (429) or server overload (529). Duration determined by server-provided `resetTimestamp` from rate-limit headers. Uses absolute timestamp (not duration) for robustness to clock skew.

### What Changes

Only one thing: `speed: 'fast'` added to API request body. Tools, prompts, thinking config, all other parameters remain identical. Beta header latched (sticky once set) for cache stability.

---

## Deep-Dive 11: Prompt Cache Break Detection

**Location**: `src/services/api/promptCacheBreakDetection.ts` (728 lines)

### Hash Function

Uses **Bun.hash (wyhash)** when available, falls back to **djb2** for Node.js:

```typescript
function computeHash(data: unknown): number {
  const str = jsonStringify(data)
  if (typeof Bun !== 'undefined') { return typeof Bun.hash(str) === 'bigint' ? Number(Bun.hash(str) & 0xffffffffn) : Bun.hash(str) }
  return djb2Hash(str)
}
```

### What's Hashed

| Field | Content | Purpose |
|---|---|---|
| `systemHash` | System prompt (cache_control stripped) | Detect text changes |
| `toolsHash` | All tool schemas (cache_control stripped) | Detect tool changes |
| `cacheControlHash` | `system.map(b => b.cache_control)` | Detect scope/TTL flips |
| `perToolHashes` | Each tool individually | Identify WHICH tool changed |
| `extraBodyHash` | Extra body params | Detect env config changes |

### Detection Threshold

```typescript
const tokenDrop = prevCacheRead - cacheReadTokens
if (cacheReadTokens >= prevCacheRead * 0.95 || tokenDrop < MIN_CACHE_MISS_TOKENS) {
  return  // Not a break
}
// MIN_CACHE_MISS_TOKENS = 2_000
```

Cache break detected when BOTH: >5% drop AND ≥2000 absolute token drop.

### TTL Detection

Compares `Date.now()` to last assistant message timestamp:
- >1h gap → `'possible 1h TTL expiry (prompt unchanged)'`
- >5min gap → `'possible 5min TTL expiry (prompt unchanged)'`
- <5min gap → `'likely server-side (prompt unchanged, <5min gap)'`

### Action Taken

**Telemetry only** — fires `tengu_prompt_cache_break` event + writes a diff file for debugging + logs a warning visible with `--debug`. No corrective action.

---

## Deep-Dive 12: Error Messages Sent to the Model

### Complete Catalog

| Scenario | Message | Source |
|---|---|---|
| **Tool not found** | `<tool_use_error>Error: No such tool available: {name}</tool_use_error>` | `toolExecution.ts:400` |
| **Zod validation** | `<tool_use_error>InputValidationError: {details}</tool_use_error>` | `toolExecution.ts:668` |
| **Tool-specific validation** | `<tool_use_error>{message}</tool_use_error>` | `toolExecution.ts:721` |
| **Permission denied** | `{permissionDecision.message}` (is_error: true) | `toolExecution.ts:1023` |
| **User cancel** | `"The user doesn't want to take this action right now. STOP what you are doing and wait for the user to tell you how to proceed."` | `messages.ts:210` |
| **User reject** | `"The user doesn't want to proceed with this tool use. The tool use was rejected... STOP what you are doing and wait for the user to tell you how to proceed."` | `messages.ts:212` |
| **Subagent reject** | `"Permission for this tool use was denied. Try a different approach or report the limitation to complete your task."` | `messages.ts:214` |
| **Auto reject** | `"Permission to use {tool} has been denied. {workaround guidance}"` | `messages.ts:219` |
| **User interrupt** | `"[Request interrupted by user for tool use]"` | `messages.ts:208` |
| **Bash timeout** | `"Command timed out after {duration}"` (prepended to stderr) | `ShellCommand.ts` |
| **Bash interrupted** | `"<error>Command was aborted before completion</error>"` | `BashTool.tsx:603` |
| **Bash no output** | `"Command failed with no output"` | `toolErrors.ts:14` |
| **File not found** | `"File does not exist. Note: your current working directory is {cwd}. Did you mean {suggestion}?"` | `FileReadTool.ts:641` |
| **Edit: same string** | `"No changes to make: old_string and new_string are exactly the same."` | `FileEditTool.ts:153` |
| **Edit: not found** | `"String to replace not found in file.\nString: {old_string}"` | `FileEditTool.ts:321` |
| **Edit: multiple matches** | `"Found {N} matches... set replace_all to true or provide more context."` | `FileEditTool.ts:336` |
| **Edit: not read** | `"File has not been read yet. Read it first before writing to it."` | `FileEditTool.ts:280` |
| **Edit: modified since read** | `"File has been modified since read, either by the user or by a linter. Read it again."` | `FileEditTool.ts:306` |
| **Edit: too large** | `"File is too large to edit ({size}). Maximum editable file size is {max}."` | `FileEditTool.ts:192` |
| **Hook blocking** | `"Execution stopped by PreToolUse hook: {reason}"` | `toolExecution.ts:1025` |
| **Max output tokens** | `"Claude's response exceeded the {N} output token maximum."` | `claude.ts:2269` |
| **Context exhausted** | `"The model has reached its context window limit."` | `claude.ts:2283` |
| **MCP tool error** | Raw MCP `result.error` string wrapped in `formatError()` | `mcp/client.ts:3141` |
| **Rate limit** | `"You've hit your {limit}{resetMessage}"` | `rateLimitMessages.ts:333` |

### Design Insight

Error messages are carefully crafted to guide model behavior:
- Cancel/reject messages say "STOP" and "wait for the user" — prevents the model from retrying
- File edit errors suggest specific fixes ("set replace_all", "provide more context")
- The `<tool_use_error>` XML wrapper helps the model distinguish system errors from content

---

## Deep-Dive 13: State Store & Bridge

### State Store Pattern

Minimal **Zustand-like external store** (35 lines total):

```typescript
export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => {
      const prev = state; const next = updater(prev)
      if (Object.is(next, prev)) return
      state = next; onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => { listeners.add(listener); return () => listeners.delete(listener) },
  }
}
```

React integration via `useSyncExternalStore`:
```typescript
export function useAppState(selector) {
  const store = useAppStore()
  return useSyncExternalStore(store.subscribe, () => selector(store.getState()), () => selector(store.getState()))
}
```

**Persisted to disk**: `mainLoopModel`, `expandedView`, `verbose`. Everything else is session-scoped.

### Bridge Protocol

Two transports: v1 (WebSocket + HTTP) and v2 (SSE + HTTP). Authentication via OAuth Bearer token with JWT session tokens. Capabilities: initialize, set_model, set_thinking_tokens, set_permission_mode, interrupt, send messages.

---

## Deep-Dive 14: Terminal Rendering Engine

### Patch Optimizer (`src/ink/optimizer.ts`)

Single-pass, 7 optimization rules:

1. **Remove empty stdout**: Skip zero-length strings
2. **Merge consecutive cursorMove**: Sum x/y deltas
3. **Remove no-op cursorMove**: (0,0) deltas dropped
4. **Collapse cursorTo**: Last one wins (absolute positioning)
5. **Concat adjacent styleStr**: Merge adjacent ANSI style transitions
6. **Dedupe hyperlinks**: Same URI → skip
7. **Cancel cursor hide/show pairs**: `result.pop()` removes preceding hide

### Frame Diffing

Cell-by-cell diff via `diffEach(prev.screen, next.screen, callback)`. For each changed cell: move virtual cursor, emit style transition, write character.

DECSTBM hardware scroll optimization when in alt-screen: uses `CSI n S/T` instead of rewriting entire viewport.

Wide character handling: Unicode 12.0+ emoji detection with VS16 compensation.

### Render Frequency

**~60fps** via lodash `throttle(fn, 16ms, { leading: true, trailing: true })`. Scroll drain frames at ~250fps (4ms interval). Slow render logging at >50ms.

### Surprises

- The custom Ink fork uses Yoga for layout (same engine as React Native) — yoga layout runs on every render at 1-3ms.
- Wide character compensation for emoji is remarkably detailed — tracks Unicode 12.0+ symbols and text-default emoji with variation selectors.

---

## Deep-Dive 15: Analytics & Telemetry

### Event Taxonomy

~200+ events across categories: lifecycle, API/streaming, tool use, OAuth, MCP, user actions, compaction/context, voice, memory/team, UI, files, settings, plugins, speculation, agents.

### Two Backends

1. **Datadog**: POSTs to `https://http-intake.logs.us5.datadoghq.com`. Flushes every 15s or 100 events. Client token embedded. Filtered to `DATADOG_ALLOWED_EVENTS` set.
2. **1P (First-Party)**: OpenTelemetry `BatchLogRecordProcessor` → `https://api.anthropic.com/api/event_logging/batch`. Failed events persisted to disk as JSONL for retry.

### PII Protections

- Type system marker: `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` forces developer attestation
- Metadata type excludes strings: `{ [key: string]: boolean | number | undefined }`
- MCP tool names sanitized to generic `'mcp_tool'`
- User IDs hashed (SHA256 mod 30 buckets) for Datadog
- `_PROTO_*` keys stripped before Datadog, kept only for 1P privileged columns

### GrowthBook

Remote evaluation (`remoteEval: true`). User attributes: device ID, session ID, platform, org UUID, subscription type, rate limit tier, app version. Disk cache fallback for offline/startup.

### Kill Switches

| Flag | What It Kills |
|---|---|
| `tengu_frond_boric` | Per-sink analytics (Datadog/1P) |
| `tengu_amber_quartz_disabled` | Voice mode |
| `tengu_amber_flint` | Agent swarms |
| `tengu_sandbox_disabled_commands` | Sandbox for specific commands |
| `tengu_auto_mode_config` (.enabled='disabled') | Auto permission mode |

---

## Implementation Patterns Cheat Sheet

### Top 30 Copy-Pasteable Patterns

1. **Minimal external store** (35 lines, no deps):
```typescript
function createStore<T>(init: T): Store<T> {
  let state = init; const listeners = new Set<() => void>()
  return {
    getState: () => state,
    setState: (fn) => { const p = state; state = fn(p); if (!Object.is(state, p)) for (const l of listeners) l() },
    subscribe: (l) => { listeners.add(l); return () => listeners.delete(l) },
  }
}
```

2. **Discriminated union state machine**: `{ status: 'cooldown'; resetAt: number } | { status: 'active' }` — invalid states unrepresentable.

3. **WeakRef abort controller tree**: Parent→child propagation with GC-safe cleanup via `WeakRef`.

4. **Copy-on-write overlay via path rewriting**: Intercept tool calls, rewrite file paths to temp dir, copy-back on commit.

5. **Content-clearing for context management**: Replace tool result content with `'[Old tool result content cleared]'` — preserves message structure while freeing tokens.

6. **API-round grouping by assistant message ID**: Group messages by shared `message.id` for atomic truncation.

7. **Shell cwd tracking via `pwd -P >| file`**: Append pwd capture to every command, read back after completion.

8. **Uniqueness check via split-count**: `file.split(searchString).length - 1` — O(n) but simple and correct.

9. **Curly-quote normalization fallback**: Try exact match first, then normalize quotes and retry.

10. **Prompt cache boundary marker**: `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` splits global-cacheable from per-session content.

11. **Streaming tool execution during model response**: `addTool()` called on each `tool_use` block as it completes in stream, triggers async `processQueue()`.

12. **Progress fast-path bypassing ordering barrier**: Progress messages yielded immediately regardless of tool ordering constraints.

13. **Circuit breaker on autocompact**: Skip after 3 consecutive failures.

14. **Stale-but-safe feature flag cache**: Return cached disk value immediately, refresh async in background.

15. **Per-tool schema hashing for cache break attribution**: Hash individual tools to identify exactly which one changed.

16. **NO_TOOLS_PREAMBLE + NO_TOOLS_TRAILER sandwich**: Repeat "do not call tools" at start and end of compaction prompt.

17. **`<analysis>` chain-of-thought then strip**: Prompt for analysis tags to improve quality, then remove them from output.

18. **Recovery message that says "no apology, no recap"**: Prevents model from wasting tokens on politeness after hitting output limit.

19. **Exit code 2 = blocking hook error**: Convention allows hooks to signal severity without string parsing.

20. **`replace(() => value)` to prevent $1 interpretation**: Callback form of String.replace avoids regex replacement patterns.

21. **De-sanitization table for API transformations**: Undo known API-level string mutations before applying edits.

22. **mtime-based read-write staleness detection**: Compare file modification time to read timestamp, reject if stale.

23. **Tool schema session memoization**: Cache serialized schemas per session to prevent GrowthBook flips from churning cached tokens.

24. **Single-pass patch optimizer**: 7 rules applied in one iteration — merge, dedupe, cancel, collapse.

25. **DECSTBM hardware scroll**: Use terminal scroll regions instead of rewriting viewport.

26. **Type marker for PII attestation**: `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` forces conscious review.

27. **Metadata type excludes strings**: `{ [key: string]: boolean | number | undefined }` — structural prevention of PII leakage.

28. **File-based mailbox for inter-agent communication**: Write JSON messages to disk, poll for delivery.

29. **Fork recursive guard via message content scanning**: Check for boilerplate tag in messages to detect fork children.

30. **Absolute timestamp for cooldown expiry**: `resetAt: number` instead of duration — robust to clock skew and system sleep.

---

---

## Deep-Dive 16: MCP Protocol Internals

### InProcessTransport — Full Implementation

**Location**: `src/services/mcp/InProcessTransport.ts` (63 lines)

```typescript
class InProcessTransport implements Transport {
  private peer: InProcessTransport | undefined
  private closed = false

  onclose?: () => void
  onerror?: (error: Error) => void
  onmessage?: (message: JSONRPCMessage) => void

  _setPeer(peer: InProcessTransport): void { this.peer = peer }
  async start(): Promise<void> {}

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.closed) throw new Error('Transport is closed')
    // Deliver asynchronously to avoid stack depth issues
    queueMicrotask(() => { this.peer?.onmessage?.(message) })
  }

  async close(): Promise<void> {
    if (this.closed) return
    this.closed = true
    this.onclose?.()
    if (this.peer && !this.peer.closed) {
      this.peer.closed = true
      this.peer.onclose?.()
    }
  }
}

export function createLinkedTransportPair(): [Transport, Transport] {
  const a = new InProcessTransport()
  const b = new InProcessTransport()
  a._setPeer(b)
  b._setPeer(a)
  return [a, b]
}
```

Key: `queueMicrotask()` prevents stack overflow from synchronous request/response ping-pong. Closing one side closes the peer (guarded by `closed` flag).

### Claude.ai Proxy Transport

Connection flow:
1. **Config fetch**: `GET ${baseUrl}/v1/mcp_servers?limit=1000` with OAuth Bearer token and `mcp-servers-2025-12-04` beta header
2. **Transport creation**: `StreamableHTTPClientTransport` pointed at `${MCP_PROXY_URL}/${MCP_PROXY_PATH}/{server_id}}`
3. **Auth wrapper**: `createClaudeAiProxyFetch` — adds OAuth token, handles 401 with automatic token refresh and one retry

```typescript
// client.ts:372-422
export function createClaudeAiProxyFetch(innerFetch: FetchLike): FetchLike {
  return async (url, init) => {
    const doRequest = async () => {
      await checkAndRefreshOAuthTokenIfNeeded()
      const currentTokens = getClaudeAIOAuthTokens()
      const headers = new Headers(init?.headers)
      headers.set('Authorization', `Bearer ${currentTokens.accessToken}`)
      return { response: await innerFetch(url, { ...init, headers }), sentToken: currentTokens.accessToken }
    }
    const { response, sentToken } = await doRequest()
    if (response.status !== 401) return response
    const tokenChanged = await handleOAuth401Error(sentToken).catch(() => false)
    if (!tokenChanged) return response
    return (await doRequest()).response
  }
}
```

### Tool Schema Normalization (MCP → Internal)

| MCP Field | Internal Field | Mapping |
|---|---|---|
| `tool.name` | `name` | `mcp__<normalized_server>__<normalized_tool>` |
| `tool.inputSchema` | `inputJSONSchema` | Direct passthrough |
| `tool.description` | `prompt()` | Truncated at 2048 chars |
| `tool.annotations.readOnlyHint` | `isConcurrencySafe()` | Boolean, default false |
| `tool.annotations.destructiveHint` | `isDestructive()` | Boolean, default false |
| `tool.annotations.openWorldHint` | `isOpenWorld()` | Boolean, default false |
| `tool.annotations.title` | `userFacingName()` | Falls back to `tool.name` |
| `tool._meta['anthropic/searchHint']` | `searchHint` | String, whitespace-normalized |
| `tool._meta['anthropic/alwaysLoad']` | `alwaysLoad` | Boolean |

### Elicitation Flow (Error -32042)

Full flow in `callMCPToolWithUrlElicitationRetry` (client.ts:2813-3027):

```
1. Call MCP tool → catch McpError with code -32042
2. Extract elicitations from error.data.elicitations
3. Validate: mode='url', url, elicitationId, message all present
4. Try elicitation hooks first (can resolve programmatically)
5. If no hook: display to user (REPL: queue in appState; SDK: handleElicitation callback)
6. Run elicitation result hooks on user's response
7. If accepted: loop back and retry the tool call
8. If declined/cancelled: return error message to model
9. Max 3 elicitation retries per tool call
```

### SdkControlTransport (IDE Bridge)

Two classes for CLI↔SDK bidirectional messaging:

- **SdkControlClientTransport** (CLI-side): `send()` calls `sendMcpMessage(serverName, message)` which goes through stdout control channel to SDK process, awaits response synchronously.
- **SdkControlServerTransport** (SDK-side): `send()` calls callback to pass MCP response back to Query layer.

Supports multiple SDK MCP servers simultaneously — `serverName` parameter routes messages.

### MCP Resource Tools

- **ListMcpResourcesTool**: Returns `[{ uri, name, mimeType?, description?, server }]`. Empty result: `"No resources found. MCP servers may still provide tools even if they have no resources."`
- **ReadMcpResourceTool**: Reads resource content. Binary blobs: base64-decoded, saved to disk, model gets filepath + human-readable message instead of raw base64.

### Error Boundary

- **fetchToolsForClient**: Unicode sanitization on all results. Outer try/catch returns `[]` — single server failure never poisons others.
- **callMCPTool**: Promise.race with timeout. isError field checked. 401 → McpAuthError. 404/-32001 → session expiry → cache clear + retry. AbortError → silent return. All MCP errors wrapped in `TelemetrySafeError` for telemetry safety.

### Surprises

- Binary resource blobs are persisted to disk rather than sent inline — prevents context bloat from base64 encoding.
- Session expiry detection checks for both HTTP 404 AND JSON-RPC code -32001 by parsing the error message string.

---

## Deep-Dive 17: The Permission Classifier (Auto Mode)

### What Model Powers It?

Priority: (1) `CLAUDE_CODE_AUTO_MODE_MODEL` env var (ant-only), (2) `tengu_auto_mode_config.model` GrowthBook config, (3) **same model as main agent loop** (defaults to Opus 4.6).

### Classification Prompt

The actual prompt text is **not present in this external build**. It's loaded from `.txt` files via build-time `feature('TRANSCRIPT_CLASSIFIER')` dead-code elimination:

```typescript
const BASE_PROMPT = feature('TRANSCRIPT_CLASSIFIER')
  ? txtRequire(require('./yolo-classifier-prompts/auto_mode_system_prompt.txt'))
  : ''

const EXTERNAL_PERMISSIONS_TEMPLATE = feature('TRANSCRIPT_CLASSIFIER')
  ? txtRequire(require('./yolo-classifier-prompts/permissions_external.txt'))
  : ''
```

Two template variants: `permissions_external.txt` (external users) and `permissions_anthropic.txt` (Anthropic-internal). Templates have XML-tagged substitution zones:
- `<user_allow_rules_to_replace>` — user's allow rules from settings
- `<user_deny_rules_to_replace>` — user's deny rules from settings
- `<user_environment_to_replace>` — environment context

CLAUDE.md content is also included as user intent context.

### Features Used

1. **System prompt**: Base prompt + permissions template + user rules (allow/deny/environment)
2. **CLAUDE.md content**: Wrapped in `<user_claude_md>` tags
3. **Conversation transcript**: Full history but **assistant text deliberately excluded** — only tool_use blocks from assistant messages are included to prevent model-authored text from influencing the classifier
4. **The action being classified**: Each tool's `toAutoClassifierInput()` method (e.g., Bash returns `input.command`)

### Two-Stage XML Classification

**Stage 1 ("fast")**: max_tokens=64, stop_sequences on `</block>`, suffix nudges immediate decision. Parses `<block>yes</block>` or `<block>no</block>`.

**Stage 2 ("thinking")**: max_tokens=4096, chain-of-thought with `<thinking>` tags. Only triggered when stage 1 blocks (escalation).

Three modes: `'both'` (default: fast→escalate if blocked), `'fast'` (stage 1 only), `'thinking'` (stage 2 only).

### Fast Paths (Before Classifier)

Before calling the API, safe tools are auto-allowed without classification:
- Read-only: FileRead, Grep, Glob, LSP, ToolSearch
- Task management: TaskCreate/Get/Update/List
- Plan mode: EnterPlanMode, ExitPlanMode
- Coordination: SendMessage, TeamCreate/Delete, Sleep

Additionally: if the action would be allowed under `acceptEdits` mode, skip the classifier.

### Latency

Classification runs **synchronously in the hot path**, blocking tool execution. Stage 1 (max_tokens=64) mitigates latency for allowed actions. Only blocks escalate to the slower stage 2.

### Fallback on Error

| Condition | Behavior |
|---|---|
| API error + `tengu_iron_gate_closed` = true (default) | **Fail closed** — deny |
| API error + `tengu_iron_gate_closed` = false | **Fail open** — fall through to interactive prompting |
| Transcript too long (interactive) | Fall back to interactive prompting |
| Transcript too long (headless) | Abort — permanent condition |
| Parse failure | Block for safety |
| 3 consecutive or 20 total denials | Fall back to interactive prompting |

### Surprises

- Assistant text is deliberately excluded from the classifier transcript — a defense against the model crafting text to influence its own permission decisions.
- The two-stage approach is a latency optimization: stage 1 at 64 tokens is fast for the common allow case; only blocks get the expensive chain-of-thought.
- Eval harnesses exist in a separate `sandbox/` directory (not shipped) — referenced in comments pointing to Python classifier eval scripts.

---

## Deep-Dive 18: Session Memory Extraction (Full Detail)

### Extraction Prompt (Verbatim)

The full user prompt (`SessionMemory/prompts.ts:43-81`):

```
IMPORTANT: This message and these instructions are NOT part of the actual user
conversation. Do NOT include any references to "note-taking", "session notes
extraction", or these update instructions in the notes content.

Based on the user conversation above (EXCLUDING this note-taking instruction
message as well as system prompt, claude.md entries, or any past session
summaries), update the session notes file.

The file {{notesPath}} has already been read for you. Here are its current contents:
<current_notes_content>
{{currentNotes}}
</current_notes_content>

Your ONLY task is to use the Edit tool to update the notes file, then stop.
You can make multiple edits (update every section as needed) - make all Edit
tool calls in parallel in a single message. Do not call any other tools.

CRITICAL RULES FOR EDITING:
- The file must maintain its exact structure with all sections, headers, and
  italic descriptions intact
- NEVER modify, delete, or add section headers
- NEVER modify or delete the italic _section description_ lines
- ONLY update the actual content that appears BELOW the italic _section
  descriptions_ within each existing section
- Do NOT add any new sections, summaries, or information outside the existing structure
- Write DETAILED, INFO-DENSE content for each section
- Do not include information that's already in the CLAUDE.md files
- Keep each section under ~2000 tokens
- IMPORTANT: Always update "Current State" to reflect the most recent work
```

### Background Agent Configuration

- **Model**: Inherits parent's model (required for prompt cache sharing)
- **Tools**: Inherits parent's full tool set, but `canUseTool` restricts to ONLY `Edit` tool targeting the exact memory file
- **Query source**: `'session_memory'` (prevents autocompact deadlock)
- **Concurrency**: Wrapped in `sequential()` — one extraction at a time
- **Registration**: Post-sampling hook — fires after every model sampling in main REPL

### Trigger Thresholds

| Threshold | Default | Description |
|---|---|---|
| `minimumMessageTokensToInit` | 10,000 | Total context before first extraction |
| `minimumTokensBetweenUpdate` | 5,000 | Growth since last extraction |
| `toolCallsBetweenUpdates` | 3 | Tool calls since last extraction |

Trigger condition: `(tokenThreshold AND toolCallThreshold) OR (tokenThreshold AND no tools in last turn)`.

### Compaction Integration

When autocompact triggers, `trySessionMemoryCompaction()` is tried FIRST:
1. Waits for in-progress extraction (polls 1s, 15s timeout)
2. Reads session memory from disk
3. If empty (matches template), falls back to legacy compaction
4. Keeps recent messages after `lastSummarizedMessageId` (min 10K tokens, min 5 messages, max 40K)
5. Uses session memory content AS the compaction summary — **no API call**

### Template Customization

- Custom template: `~/.claude/session-memory/config/template.md`
- Custom prompt: `~/.claude/session-memory/config/prompt.md`
- Both fall back to defaults on ENOENT
- Custom prompts support `{{variableName}}` substitution

---

## Deep-Dive 19: Testing Patterns

### No Tests in This Build

This is a **source-only distribution** without the test suite. Zero test files (`*.test.ts`, `*.spec.ts`, `__tests__/`) exist. No test framework configuration (vitest, jest, bun:test) is present.

### Evidence of External Test Infrastructure

1. **`_resetForTesting()` exports** in: `autoModeState.ts`, `analytics/index.ts`, `fullscreen.ts` — the naming convention for test-accessible state reset
2. **Eval harness references in comments**: `sandbox/alexg/evals/cc_report_bpc_eval.py`, `sandbox/alexg/evals/tool_denial_bpc_eval.py` — Python-based classifier evaluation scripts
3. **Runtime mocking**: `src/services/mockRateLimits.ts` and `src/commands/mock-limits/` — runtime mock for rate limits (accessible via `/mock-limits` command, ant-only)

### Surprises

- The test suite is completely separate from the shipped source — a clean separation between production code and test infrastructure.
- The `_resetForTesting()` convention is elegant — functions that clear module-scoped state, invisible in production but essential for test isolation.

---

## Deep-Dive 20: Voice Input System

### Single File

`src/voice/voiceModeEnabled.ts` (55 lines) — three exported functions:

1. **`isVoiceGrowthBookEnabled()`**: Build-time `feature('VOICE_MODE')` gate + runtime `tengu_amber_quartz_disabled` kill switch
2. **`hasVoiceAuth()`**: Requires Claude.ai OAuth token (not API keys, not Bedrock/Vertex)
3. **`isVoiceModeEnabled()`**: Both gates must pass

### How Voice Enters the Agent Loop

Voice input uses a **speech-to-text streaming service** (`src/services/voiceStreamSTT.ts`) connecting to Claude.ai's `voice_stream` endpoint. The `tengu_cobalt_frost` feature flag gates the Nova 3 STT model. Transcribed text is fed into the normal user input path — same as typed text — entering the agent loop as a standard user message through the REPL's `onQuery` handler.

### Feature Status

Gated behind `feature('VOICE_MODE')` (build-time DCE) and requires Claude.ai OAuth. The kill switch `tengu_amber_quartz_disabled` can remotely disable it.

---

## Final Implementation Patterns Cheat Sheet (Addendum)

### Patterns 31-40

31. **`queueMicrotask` for async delivery in linked transports**: Prevents stack overflow from synchronous request/response ping-pong in in-process MCP.

32. **Two-stage classifier with fast-path escalation**: Stage 1 at 64 max_tokens for fast allow; only blocks escalate to 4096-token chain-of-thought.

33. **Exclude assistant text from classifier transcript**: Defense against model crafting text to influence its own permission decisions.

34. **Session memory as compaction summary (zero API cost)**: Pre-built incrementally, used directly at compaction time without an LLM summarization call.

35. **`canUseTool` callback for tool restriction**: Rather than modifying tool lists, pass a callback that gates specific tools to specific paths — preserves cache-key compatibility.

36. **Sequential wrapper for background tasks**: `sequential()` ensures only one extraction runs at a time without locks.

37. **`_resetForTesting()` convention**: Module-scoped state reset functions, invisible in production, essential for test isolation.

38. **OAuth 401 retry with token staleness check**: On 401, refresh token, but verify the new token differs from the one that failed before retrying.

39. **Binary blob persistence for MCP resources**: Save base64-decoded blobs to disk, give model filepath instead of inline content.

40. **Build-time DCE for sensitive prompts**: Classifier prompts loaded via `feature('TRANSCRIPT_CLASSIFIER')` — completely eliminated from external builds.

---

*End of deep-dive report. All 20 deep-dives complete with verbatim code, line numbers, and surprises.*
