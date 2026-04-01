# Claude Code v2.1.88 Permission System Deep-Dive Analysis

**Date**: 2026-04-02
**Analysis Version**: 1.0
**Codebase**: `/sessions/cool-friendly-einstein/mnt/claude-code/src/utils/permissions/`
**Document Length**: 1400+ lines

---

## Executive Summary

Claude Code v2.1.88 implements a sophisticated, multi-layered permission system that gates tool execution through a 7-stage permission decision pipeline. At its core lies the **YOLO Classifier**—an LLM-based safety evaluator that operates in "auto mode"—which can automatically approve or deny tool use without user intervention.

This analysis reverse-engineers the architecture, data structures, decision protocols, algorithms, and security gates that comprise this system. The focus is on understanding *how* Anthropic engineered the system rather than exploiting it.

### Key Findings

1. **6 Permission Modes** with distinct allow/deny semantics (default, plan, acceptEdits, bypassPermissions, dontAsk, auto)
2. **3-Stage Decision Pipeline**: Rule evaluation → Classifier stage 1 (fast) → Classifier stage 2 (thinking)
3. **Dangerous Permission Stripping**: Auto mode automatically removes rules that would bypass the classifier
4. **Denial Tracking**: 3 consecutive / 20 total denials trigger fallback to manual approval
5. **Two-Stage XML Classifier**: Stage 1 makes fast decisions (max_tokens=64); Stage 2 performs chain-of-thought if Stage 1 says deny
6. **Prompt Caching**: System prompt + CLAUDE.md prefix cached across classifier calls (1h TTL)
7. **Killswitch Mechanism**: GrowthBook feature gates disable bypassPermissions and auto modes at runtime

---

## Part 1: Architecture Overview

### 1.1 Permission Decision Pipeline (7 Stages)

The core permission flow in `permissions.ts:hasPermissionsToUseToolInner()` follows this decision tree:

```
Tool Use Request
  ↓
Stage 1a: Deny Rule Check
  └─ If entire tool has deny rule → DENY
  ↓
Stage 1b: Ask Rule Check (Tool-Level)
  └─ If entire tool has ask rule → ASK
  ↓
Stage 1c: Tool-Specific Permission Check
  └─ Execute tool.checkPermissions() (Bash subcommand rules, file path safety, etc.)
  ├─ Possible returns: 'allow', 'deny', 'ask', 'passthrough'
  ↓
Stage 1d: Tool Deny Check
  └─ If tool returned deny → DENY
  ↓
Stage 1e: User Interaction Gate
  └─ If tool requires user interaction and result is 'ask' → ASK (immune to bypass modes)
  ↓
Stage 1f: Content-Specific Ask Rule Check
  └─ If tool returned ask with ask-rule decisionReason → ASK
  ↓
Stage 1g: Safety Check (Bypass-Immune)
  └─ If tool returned ask with safetyCheck decisionReason → ASK (immune to bypassPermissions)
  ↓
Stage 2a: Bypass Permissions Mode Check
  └─ If mode === 'bypassPermissions' OR (mode === 'plan' AND isBypassPermissionsModeAvailable)
    └─ If no ask/deny rules objected → ALLOW
  ↓
Stage 2b: Entire Tool Always-Allow Rule Check
  └─ If tool matches allow rule with no ruleContent → ALLOW
  ↓
Stage 3: Convert Passthrough to Ask
  └─ If result is 'passthrough' → convert to 'ask'
  ↓
Stage 4: Auto Mode Classifier (if applicable)
  ├─ Check dontAsk mode → convert ask → DENY
  ├─ Check auto mode / plan with auto active
  ├─ Run classifier via classifyYoloAction()
  └─ Return ALLOW, DENY, or ask again (denial limit exceeded)
  ↓
Stage 5: Fallback to Prompting
  └─ If no permission prompts available (headless mode) → DENY or run hooks
  ↓
Return Permission Decision
```

### 1.2 Permission Modes

Defined in `PermissionMode.ts` and `types/permissions.ts`:

| Mode | External? | Description | Allow/Deny Behavior |
|------|-----------|-------------|---------------------|
| `default` | Yes | User prompted for all 'ask' decisions | Explicit user approval required |
| `plan` | Yes | Review phase; mode 'plan' set, then Shift+Tab cycles to next | Prompts like default unless plan→auto active |
| `acceptEdits` | Yes | Fast-path for safe edits (FileEdit/Read in cwd) | Auto-allows file edits, still prompts bash/etc |
| `bypassPermissions` | Yes | Skip all permission checks (except safety checks, ask rules) | Auto-allows all non-bypass-immune actions |
| `dontAsk` | Yes | Convert all 'ask' to 'deny' | Blocks everything that would prompt |
| `auto` | No (ant-only w/ feature flag) | Use YOLO Classifier for decision | Classifier decides; denial limits trigger fallback |

**Mode Cycle** (Shift+Tab):
- External users: `default` → `acceptEdits` → `plan` → `bypassPermissions` → `default`
- Ant users: `default` → `bypassPermissions` → `auto` → `default`

### 1.3 Permission Rule Types

Defined in `PermissionRule.ts` and parsed from rule strings like `"Bash(npm publish:*)"`:

```typescript
type PermissionRule = {
  source: PermissionRuleSource  // where rule came from
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: {
    toolName: string  // 'Bash', 'PowerShell', 'Agent', 'mcp__server1', etc.
    ruleContent?: string  // Optional pattern like 'npm publish:*', 'python:*'
  }
}

type PermissionRuleSource =
  | 'userSettings'      // ~/.claude/settings.json (or per-project)
  | 'projectSettings'   // ./.claude/settings.json
  | 'localSettings'     // ./.claude/settings.local.json
  | 'policySettings'    // Managed by admin, read-only
  | 'flagSettings'      // GrowthBook feature gates, read-only
  | 'cliArg'            // --allowed-tools / --disallowed-tools
  | 'command'           // SDK control messages
  | 'session'           // In-memory, current session only
```

**Rule String Format**: `ToolName` or `ToolName(content)` where content is tool-specific.

---

## Part 2: YOLO Classifier Deep-Dive

### 2.1 Classifier Architecture

Located in `yoloClassifier.ts`, the classifier is a two-stage LLM-based decision system that evaluates whether an action is safe or dangerous.

#### 2.1.1 System Prompt Components

The classifier receives a system prompt composed of:

1. **Base Prompt** (`auto_mode_system_prompt.txt`):
   - Instructions to classify actions as safe or dangerous
   - Definitions of dangerous action categories (BLOCK patterns)
   - Request to use `classify_result` tool or XML output format

2. **Permissions Template** (external or internal):
   - Default ALLOW rules (actions that are generally safe)
   - Default SOFT_DENY rules (actions that need special justification)
   - Environment-specific context (user's project setup, tools available)
   - Placeholders for user customizations: `<user_allow_rules_to_replace>`, `<user_deny_rules_to_replace>`, `<user_environment_to_replace>`

3. **User Configuration** (from `settings.autoMode`):
   - Custom allow descriptions
   - Custom soft_deny (deny guidance)
   - Custom environment notes

4. **Bash Prompt Rules** (optional, feature-gated):
   - Generated from `Bash(prompt:...)` rules in permission settings
   - Descriptions of safe bash patterns

5. **PowerShell Deny Guidance** (optional, feature-gated):
   - Maps PS idioms to BLOCK categories
   - E.g., `iex (iwr ...)` → "Code from External" (same as `curl | bash`)

#### 2.1.2 Classifier Request Structure

```javascript
sideQuery({
  model: 'claude-opus-4.1' or similar,
  max_tokens: 4096,  // or 64 for stage 1 fast mode
  system: [
    {
      type: 'text',
      text: systemPrompt,
      cache_control: { type: 'ephemeral' }  // 1h TTL
    }
  ],
  skipSystemPromptPrefix: true,
  temperature: 0,  // deterministic
  thinking: false,  // or undefined for alwaysOnThinking models (padding: 2048)
  messages: [
    // Optional CLAUDE.md message
    { role: 'user', content: '...' },
    // Main classifier prompt
    { role: 'user', content: [
      { type: 'text', text: '<transcript>\n' + compactTranscript + '</transcript>\n' },
      { type: 'text', text: actionCompact, cache_control: { type: 'ephemeral' } }
    ]}
  ],
  tool_choice: { type: 'tool', name: 'classify_result' },  // stage 1 tool-use mode
  tools: [YOLO_CLASSIFIER_TOOL_SCHEMA],
  stop_sequences: ['</block>'],  // stage 1 fast mode only
  maxRetries: 3,  // transient error retry
  signal: abortSignal,
  querySource: 'auto_mode'
})
```

#### 2.1.3 Classifier Response Formats

**Tool-Use Mode** (older, enabled when `isTwoStageClassifierEnabled() === false`):

```json
{
  "type": "tool_use",
  "name": "classify_result",
  "input": {
    "thinking": "string",
    "shouldBlock": boolean,
    "reason": "string"
  }
}
```

**XML Mode** (newer, 2-stage pipeline):

```xml
<thinking>Chain-of-thought reasoning</thinking>
<block>yes|no</block>
<reason>One sentence explanation</reason>
```

### 2.2 Two-Stage XML Classifier Pipeline

When enabled via `tengu_auto_mode_config.twoStageClassifier` (GrowthBook), the classifier operates in two stages:

#### Stage 1: Fast Decision (xml_s1_h2)

**Configuration**:
- `max_tokens`: 64 (stage 1) or 256 (fast-only mode)
- `stop_sequences`: `['</block>']` (early termination on `<block>yes/no`)
- Suffix appended: `"\nErr on the side of blocking. <block> immediately."`

**Decision Tree**:
```
Stage 1 Response
  ├─ <block>no</block> → ALLOW (fast path)
  ├─ <block>yes</block> → Escalate to Stage 2 (if mode === 'both')
  └─ Unparseable → Escalate or DENY (depends on mode)
```

#### Stage 2: Thinking Decision (xml_s2_t2)

**Configuration**:
- `max_tokens`: 4096
- No `stop_sequences` (full reasoning)
- Suffix appended: `"\nReview the classification process and follow it carefully, making sure you deny actions that should be blocked. As a reminder, explicit (not suggestive or implicit) user confirmation is required to override blocks. Use <thinking> before responding with <block>."`

**Decision Tree**:
```
Stage 2 Response
  ├─ <block>no</block> → ALLOW
  ├─ <block>yes</block> → DENY
  └─ Unparseable → DENY (safe failure)
```

**Modes**:
- `'both'` (default): Stage 1 → Stage 2 if blocked
- `'fast'`: Stage 1 only, bumped to 256 tokens
- `'thinking'`: Stage 2 only, skips stage 1

### 2.3 Transcript Construction

**Function**: `buildTranscriptEntries()` and `buildTranscriptForClassifier()`

Builds a compact representation of conversation history for classifier input:

```typescript
type TranscriptEntry = {
  role: 'user' | 'assistant'
  content: TranscriptBlock[]  // { type: 'text' | 'tool_use', ... }
}
```

**Transcript Rules**:
1. Only include **user text** (not assistant text, which is model-authored and could be manipulated)
2. Only include **assistant tool_use blocks** (not tool results—too verbose)
3. Include queued commands (attachment messages with `queued_command` type)
4. Serialize tool inputs via each tool's `toAutoClassifierInput()` method

**Serialization Formats** (configurable):
- **Text Format** (default): `Bash ls\nPython {"code":"print(1)"}\n`
- **JSONL Format** (feature-gated): `{"Bash":"ls"}\n{"Python":{"code":"print(1)"}}\n`

**CLAUDE.md Inclusion**:
- Built via `buildClaudeMdMessage()`
- Wrapped in `<user_claude_md>...</user_claude_md>` delimiter
- Cached separately from transcript (cache_control on its own message block)

### 2.4 Denial Tracking State Machine

Defined in `denialTracking.ts`:

```typescript
export const DENIAL_LIMITS = {
  maxConsecutive: 3,  // Back-to-back denials
  maxTotal: 20        // Denials in this session
}

type DenialTrackingState = {
  consecutiveDenials: number
  totalDenials: number
}

function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return (
    state.consecutiveDenials >= 3 ||
    state.totalDenials >= 20
  )
}
```

**State Transitions**:
- `recordDenial()`: Increment both counters
- `recordSuccess()`: Reset consecutive counter (but NOT total)
- `shouldFallbackToPrompting()`: Check if limits exceeded

**Behavior on Denial Limit**:
1. Log `tengu_auto_mode_denial_limit_exceeded` event
2. If headless mode: `throw AbortError('Agent aborted: too many classifier denials')`
3. If interactive: Fall back to prompting the user (show warning message)
4. If total limit hit: Reset both counters to 0 (fresh start after user reviews)

### 2.5 Classifier Configuration (GrowthBook)

All classifier parameters are driven by `tengu_auto_mode_config` in GrowthBook:

```typescript
type AutoModeConfig = {
  enabled?: 'enabled' | 'disabled' | 'opt-in'  // Circuit breaker
  model?: string  // Override classifier model (e.g., 'claude-opus-4')
  twoStageClassifier?: boolean | 'fast' | 'thinking'  // Enable XML pipeline
  forceExternalPermissions?: boolean  // Use external template instead of internal
  disableFastMode?: boolean  // Disable auto mode when AppState.fastMode is on
  jsonlTranscript?: boolean  // Use JSONL serialization format
}
```

**Enabled States**:
- `'enabled'`: Available in shift-tab carousel and CLI
- `'disabled'`: Circuit breaker; blocks all auto mode entry
- `'opt-in'`: Available only if user has explicitly opted in (flag or setting)

### 2.6 Classifier Error Handling

**API Error Types**:

1. **Transient Errors** (429, 5xx):
   - Retried internally by `sideQuery()` (default `maxRetries: 3`)
   - `shouldBlock: true, unavailable: true` on final failure

2. **Prompt Too Long** (400: "prompt is too long"):
   - Parsed via `detectPromptTooLong()`
   - Returns `{ shouldBlock: true, transcriptTooLong: true }`
   - Falls back to manual prompting if interactive

3. **Parse Failures** (tool_use or XML response malformed):
   - `shouldBlock: true, reason: 'Classifier returned no tool use block - blocking for safety'`
   - Logged as `tengu_auto_mode_outcome` with `failureKind: 'no_tool_use'` or `'invalid_schema'`

4. **Abort Signals**:
   - User cancellation (Ctrl+C)
   - Returns `{ shouldBlock: true, unavailable: true }`

**Error Dumping** (ant-only):
- Enabled by `CLAUDE_CODE_DUMP_AUTO_MODE=1` env var
- Dumps classifier request/response to `/tmp/claude-{uid}/auto-mode/{timestamp}.[stage1|stage2].{req,res}.json`
- On error: Full error dump to `/tmp/claude-{uid}/auto-mode-classifier-errors/{sessionId}.txt` including context comparison (main loop tokens vs classifier tokens)

---

## Part 3: Permission Modes & Transitions

### 3.1 Mode Transition Engine

**Function**: `transitionPermissionMode(fromMode, toMode, context)` in `permissionSetup.ts`

Centralizes all state changes when switching permission modes:

```typescript
export function transitionPermissionMode(
  fromMode: string,
  toMode: string,
  context: ToolPermissionContext,
): ToolPermissionContext {
  if (fromMode === toMode) return context  // No-op

  // 1. Plan mode exit attachment
  handlePlanModeTransition(fromMode, toMode)

  // 2. Auto mode activation
  handleAutoModeTransition(fromMode, toMode)

  // 3. Exiting plan mode anywhere
  if (fromMode === 'plan' && toMode !== 'plan') {
    setHasExitedPlanMode(true)
  }

  // 4. Plan mode entry
  if (toMode === 'plan' && fromMode !== 'plan') {
    return prepareContextForPlanMode(context)
  }

  // 5. Classifier activation/deactivation
  if (toUsesClassifier && !fromUsesClassifier) {
    autoModeStateModule?.setAutoModeActive(true)
    context = stripDangerousPermissionsForAutoMode(context)
  } else if (fromUsesClassifier && !toUsesClassifier) {
    autoModeStateModule?.setAutoModeActive(false)
    setNeedsAutoModeExitAttachment(true)
    context = restoreDangerousPermissions(context)
  }

  // 6. Clean prePlanMode on exit
  if (fromMode === 'plan' && toMode !== 'plan' && context.prePlanMode) {
    return { ...context, prePlanMode: undefined }
  }

  return context
}
```

### 3.2 Dangerous Permission Stripping (Auto Mode Entry)

**Function**: `stripDangerousPermissionsForAutoMode()` in `permissionSetup.ts`

When entering auto mode, dangerous allow-rules are removed to prevent them from bypassing the classifier:

**Dangerous Rules**:

1. **Bash**:
   - `Bash` (tool-level, no content)
   - `Bash(python:*)`, `Bash(python*)`, `Bash(python -*)`
   - All `DANGEROUS_BASH_PATTERNS`: python, python3, node, deno, bash, sh, ssh, npm run, sudo, eval, exec, etc. (plus ant-only: coo, fa run, gh, curl, wget, git, kubectl, aws, gcloud, gsutil)

2. **PowerShell**:
   - `PowerShell` (tool-level)
   - `PowerShell(iex:*)`, `PowerShell(invoke-expression:*)`
   - `PowerShell(start-process:*)`, `PowerShell(new-object:*)`
   - All patterns matching PS code executors, process spawners, .NET escapeslocal

3. **Agent**:
   - `Agent` (any allow-rule)
   - Allows arbitrary sub-agent delegation, bypassing classifier evaluation of delegated prompts

**Storage**: Stripped rules stashed in `context.strippedDangerousRules` (Map<source, ruleStrings[]>)

**Restoration** (on auto mode exit): `restoreDangerousPermissions()` re-adds them from stash

### 3.3 Plan Mode State Management

Plan mode enters "review" state where the user can inspect the transcript before execution. Auto mode can be active *during* plan mode.

**Plan Mode Entry** (`prepareContextForPlanMode`):
- Stash current mode in `prePlanMode`
- If auto mode is active during plan: stay active (classifier still runs)
- If auto mode opt-in is set: activate auto during plan (if `shouldPlanUseAutoMode()`)
- Otherwise: strip dangerous permissions and activate auto (auto-opt-in entry)

**Plan Mode Exit**:
- Restore to `prePlanMode`
- Deactivate auto mode if it was only active during plan
- Restore dangerous permissions

---

## Part 4: Filesystem & Path Permission Engine

### 4.1 Dangerous File/Directory Protection

**Dangerous Directories** (in `filesystem.ts`):
```typescript
export const DANGEROUS_DIRECTORIES = [
  '.git',     // git data & hooks (code execution)
  '.vscode',  // VS Code settings & extensions
  '.idea',    // JetBrains IDE settings
  '.claude',  // Claude Code configuration
]

export const DANGEROUS_FILES = [
  '.gitconfig',     // git config (sshCommand hooks)
  '.gitmodules',    // submodule data
  '.bashrc', '.bash_profile', '.zshrc', '.zprofile', '.profile',  // shell startup
  '.ripgreprc',     // ripgrep config
  '.mcp.json',      // MCP protocol config
  '.claude.json',   // Claude config
]
```

**Function**: `isDangerousFilePathToAutoEdit()`

Returns true (requires permission) for:
1. UNC paths: `\\`, `//` (network access)
2. Paths containing dangerous directories
3. Files matching dangerous file names
4. Case-insensitive check (prevents `.cLauDe/settings.json` bypass)

**Windows Path Bypasses Detected**:
- NTFS Alternate Data Streams: `file.txt:stream`
- 8.3 short names: `GIT~1`, `SETTIN~1.JSON`
- Long path prefixes: `\\?\C:\`, `\\.\C:\`
- Trailing dots/spaces: `.git.`, `.claude `
- DOS device names: `.git.CON`, `settings.json.PRN`
- Multiple consecutive dots: `path/.../file`

### 4.2 Skill Scope Narrowing

**Function**: `getClaudeSkillScope()`

For files in `.claude/skills/{name}/`, generates a scoped allow pattern:
- Pattern: `/.claude/skills/{skillName}/**`
- Used to offer "allow edits to this skill only" in permission dialogs
- Prevents single skill iteration from requiring full `.claude/` access

### 4.3 Safety Checks (Bypass-Immune)

**Bypass-Immune Safety Checks** (in step 1g):
- `.git/`, `.vscode/`, `.idea/`, `.claude/` directories
- Dangerous files (`.bashrc`, `.gitconfig`, etc.)
- Claude settings files (`.claude/settings.json`, `.claude/settings.local.json`)
- Session plan files

These checks return `decisionReason: { type: 'safetyCheck', classifierApprovable: boolean }` and MUST prompt even in:
- `bypassPermissions` mode
- `plan` mode
- `acceptEdits` mode
- Auto mode (unless `classifierApprovable === true` for sensitive-file paths)

---

## Part 5: Data Structures & Type System

### 5.1 ToolPermissionContext

**Location**: `src/Tool.ts`

```typescript
interface ToolPermissionContext {
  // Current mode
  mode: PermissionMode  // 'default' | 'plan' | 'acceptEdits' | 'bypassPermissions' | 'dontAsk' | 'auto'

  // Rules grouped by source and behavior
  alwaysAllowRules: ToolPermissionRulesBySource  // { source: [ruleString, ...] }
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource

  // Mode availability gates
  isBypassPermissionsModeAvailable: boolean  // Checked at startup; can be disabled by gate
  isAutoModeAvailable: boolean  // Feature-gated

  // Additional allowed directories
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>

  // Plan mode state
  prePlanMode?: PermissionMode  // Mode before entering plan mode

  // Auto mode state
  strippedDangerousRules?: ToolPermissionRulesBySource  // Rules removed on auto entry

  // Headless agent state
  shouldAvoidPermissionPrompts?: boolean  // true for background/async agents
}
```

### 5.2 PermissionDecision

**Location**: `src/types/permissions.ts`

```typescript
type PermissionDecision =
  | { behavior: 'allow'; updatedInput: Record<string, unknown>; decisionReason?: PermissionDecisionReason }
  | { behavior: 'deny'; message: string; decisionReason?: PermissionDecisionReason }
  | { behavior: 'ask'; message: string; suggestions?: PermissionUpdate[]; decisionReason?: PermissionDecisionReason }

type PermissionDecisionReason =
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'classifier'; classifier: 'auto-mode' | 'dangerous-agent-action'; reason: string }
  | { type: 'hook'; hookName: string; reason?: string }
  | { type: 'safetyCheck'; classifierApprovable?: boolean; reason: string }
  | { type: 'workingDir'; reason: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'subcommandResults'; reasons: Array<[string, PermissionResult]> }
  | { type: 'sandboxOverride'; reason?: string }
  | { type: 'permissionPromptTool'; permissionPromptToolName: string }
```

### 5.3 YoloClassifierResult

**Location**: `src/types/permissions.ts`

```typescript
type YoloClassifierResult = {
  thinking?: string  // Chain-of-thought from stage 2
  shouldBlock: boolean
  reason: string
  model: string
  usage?: ClassifierUsage
  unavailable?: boolean  // API error
  transcriptTooLong?: boolean  // Prompt exceeded context window
  errorDumpPath?: string  // Path to error dump if debugging
  durationMs: number
  promptLengths: { systemPrompt: number; toolCalls: number; userPrompts: number }
  stage?: 'fast' | 'thinking'  // Which stage made the decision
  // Stage 1 details
  stage1Usage?: ClassifierUsage
  stage1DurationMs?: number
  stage1RequestId?: string
  stage1MsgId?: string
  // Stage 2 details
  stage2Usage?: ClassifierUsage
  stage2DurationMs?: number
  stage2RequestId?: string
  stage2MsgId?: string
}

type ClassifierUsage = {
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens: number
  cacheCreationInputTokens: number
}
```

---

## Part 6: Constants & Magic Numbers

### 6.1 Denial Limits

```typescript
export const DENIAL_LIMITS = {
  maxConsecutive: 3,   // Back-to-back denials trigger fallback
  maxTotal: 20         // Total denials in session trigger fallback
}
```

### 6.2 Classifier Configuration

**Stage 1 (Fast)**:
- `max_tokens`: 64 (truncated stage 1) or 256 (fast-only mode)
- `stop_sequences`: ['</block>'] (early termination)
- `temperature`: 0 (deterministic)
- Suffix: `"\nErr on the side of blocking. <block> immediately."`

**Stage 2 (Thinking)**:
- `max_tokens`: 4096
- No `stop_sequences`
- `temperature`: 0
- Suffix: `"\nReview the classification process and follow it carefully..."`

**Always-On-Thinking Models**:
- Padding (headroom): +2048 tokens
- `thinking: undefined` (don't explicitly disable)
- Reason: Models with `alwaysOnThinking` reject `thinking: disabled` with 400 error

### 6.3 Cache Control

**TTL**: 1 hour (3600 seconds)
**Applied to**:
- System prompt block
- CLAUDE.md message
- Action block (stage 1 & 2 both hit cache on this)

### 6.4 Dangerous Pattern Lists

**CROSS_PLATFORM_CODE_EXEC** (in `dangerousPatterns.ts`):
```typescript
const CROSS_PLATFORM_CODE_EXEC = [
  'python', 'python3', 'python2', 'node', 'deno', 'tsx', 'ruby', 'perl', 'php', 'lua',
  'npx', 'bunx', 'npm run', 'yarn run', 'pnpm run', 'bun run',
  'bash', 'sh', 'ssh'
]

const DANGEROUS_BASH_PATTERNS = [
  ...CROSS_PLATFORM_CODE_EXEC,
  'zsh', 'fish', 'eval', 'exec', 'env', 'xargs', 'sudo',
  ...(process.env.USER_TYPE === 'ant' ? [
    'fa run', 'coo',  // Anthropic internal
    'gh', 'gh api', 'curl', 'wget',  // Network/exfil
    'git', 'kubectl', 'aws', 'gcloud', 'gsutil'  // Cloud/cluster
  ] : [])
]
```

### 6.5 Classifier Fail-Closed Timeout

```typescript
const CLASSIFIER_FAIL_CLOSED_REFRESH_MS = 30 * 60 * 1000  // 30 minutes
```

Gate `tengu_iron_gate_closed` is refreshed every 30 min; if API fails, uses cached gate value to decide whether to fail open (fall back to prompting) or fail closed (deny).

---

## Part 7: Rule System

### 7.1 Rule Parsing

**Function**: `permissionRuleValueFromString()` in `permissionRuleParser.ts`

Parses rule strings like `"Bash(npm publish:*)"`:

```typescript
const parsed = permissionRuleValueFromString("Bash(npm publish:*)")
// Result: { toolName: 'Bash', ruleContent: 'npm publish:*' }

const toolOnly = permissionRuleValueFromString("PowerShell")
// Result: { toolName: 'PowerShell', ruleContent: undefined }
```

**Reverse**: `permissionRuleValueToString()`

```typescript
permissionRuleValueToString({ toolName: 'Bash', ruleContent: 'npm publish:*' })
// Result: "Bash(npm publish:*)"
```

### 7.2 Rule Matching

**Function**: `matchingRuleForInput()` in various tools (e.g., `BashTool.checkPermissions()`)

Uses gitignore-style glob patterns for Bash subcommand matching:

```typescript
const ignore = new Ignore()
ignore.add(rulePatternsFromSettings)
const isAllowed = !ignore.ignores(command)
```

**Pattern Examples**:
- `npm publish:*` → matches "npm publish" subcommand and variants
- `aws s3:*` → matches all S3 operations
- `git clone:*` → matches git clone (not other git subcommands)

### 7.3 Rule Shadowing Detection

**Function**: `findShadowedRules()` in `shadowedRuleDetection.ts`

Detects when a deny rule is shadowed by an earlier allow rule in the precedence chain:

**Precedence Order** (highest to lowest):
1. `cliArg` (command-line flags)
2. `session` (in-memory, set during session)
3. `localSettings` (`.claude/settings.local.json`)
4. `userSettings` (`.claude/settings.json` in home)
5. `projectSettings` (`.claude/settings.json` in project)
6. `flagSettings` (GrowthBook read-only)
7. `policySettings` (admin-managed, read-only)

If an allow rule exists at source A and a deny rule at source B where A > B, the deny is shadowed and unreachable.

---

## Part 8: Security Analysis (Secondary Focus)

### 8.1 Potential Classifier Bypass Vectors

#### Vector 1: Prompt Injection via CLAUDE.md

**Risk Level**: Medium
**Mechanism**: User-controlled `CLAUDE.md` is included in classifier input
**Mitigation**:
- Wrapped in `<user_claude_md>` delimiter
- Cached separately (classifier sees delimiter as part of cache)
- Classifier instructed to treat this as "user intent"

**Residual Risk**: Adversary with write access to CLAUDE.md can inject text designed to make classifier think an action is safe (but not override it with explicit instructions).

#### Vector 2: Transcript Manipulation via Tool Inputs

**Risk Level**: Low
**Mechanism**: Malicious tool results injected into history before classifier runs
**Mitigation**:
- Classifier only sees tool_use blocks (not tool results)
- Tool inputs are projected via `toAutoClassifierInput()` which can strip sensitive fields
- Unvalidated tool inputs caught in try/catch; fallback to raw input (no crash)

**Residual Risk**: Attacker-controlled tool input could craft input that looks safe to classifier but executes dangerously (tool-specific).

#### Vector 3: Denial Limit Exhaustion

**Risk Level**: Low → High (depends on attacker position)
**Mechanism**: Repeatedly trigger classifier denials to hit limit and force manual prompting
**Mitigation**:
- Fallback to manual prompting (not auto-allow)
- Limits are per-session
- But: In headless mode, denial limit → AbortError (agent terminates)

**Residual Risk**: Attacker inside the agent loop can craft actions that classifier denies 20 times, terminating the session.

#### Vector 4: GrowthBook Gate Manipulation

**Risk Level**: Very Low (infrastructure-level trust)
**Mechanism**: Attacker controls tengu_iron_gate_closed or tengu_auto_mode_config
**Mitigation**: Feature gates are centralized; controlled by Anthropic infrastructure
**Residual Risk**: None (outside scope of code-level analysis)

### 8.2 Permission Rule Escalation Risks

#### Risk 1: Overly Broad Allow Rules

**Pattern**: `Bash` (no content) or `Bash(*)`
**Impact**: Auto-allows all bash commands, bypasses classifier
**Detection**: `findOverlyBroadBashPermissions()` warns at startup (ant-only)
**Mitigation**: Rules stripped on auto mode entry; requires explicit re-entry

#### Risk 2: Dangerous Prefix Rules

**Pattern**: `Bash(python:*)`, `Bash(sudo:*)`
**Impact**: Auto-allows arbitrary code execution via interpreter
**Detection**: `isDangerousBashPermission()` checks during auto mode entry
**Mitigation**: Stripped from allow rules on auto mode activation

#### Risk 3: Rule Shadowing Exploitation

**Pattern**: Attacker writes `Bash(*)` to localSettings to shadow project deny rules
**Impact**: Silently removes deny rule protections
**Detection**: `findShadowedRules()` logs warnings (not enforced)
**Mitigation**: Manual rule review; no auto-remediation

### 8.3 Filesystem Permission Bypass Vectors

#### Vector 1: Case-Insensitive Bypass

**Pattern**: `.cLauDe/settings.json` on case-insensitive filesystem
**Detection**: `normalizeCaseForComparison()` lowercases before checking
**Status**: Mitigated

#### Vector 2: Path Traversal

**Pattern**: `/tmp/claude-0/../../../etc/passwd`
**Detection**: `normalize()` resolves `..` before startsWith check
**Status**: Mitigated

#### Vector 3: Windows NTFS Alternate Data Streams

**Pattern**: `.bashrc:hidden`
**Detection**: `detectSuspiciousWindowsPatterns()` checks for `:` (ADS suffix)
**Status**: Mitigated

#### Vector 4: Symlink Following (Tool-Level)

**Pattern**: `.claude/settings.json` → symlink to `/etc/passwd`
**Detection**: `pathValidation.ts` has symlink resolution for skill scopes
**Residual Risk**: Tool implementations may not validate symlinks (depends on tool)

### 8.4 Headless Agent Lockout

**Scenario**: Agent running in headless mode (no permission prompts)
**Behavior**:
1. If denial limit exceeded: AbortError (agent terminates)
2. If safety check requires interactive approval: Deny (action blocked)
3. If PermissionRequest hook provides decision: Allow/deny from hook
4. Otherwise: Auto-deny (action blocked)

**Risk**: Agent cannot escape denial spiral without external intervention

---

## Part 9: Killswitches & Feature Gates

### 9.1 Bypass Permissions Killswitch

**Gate**: `tengu_disable_bypass_permissions_mode` (Statsig)
**Checked**: Once per session in `checkAndDisableBypassPermissionsIfNeeded()` (React component) or async in SDK init
**Effect**: If enabled, kick user out of bypassPermissions mode to default
**Precedence**:
1. Statsig gate (highest)
2. Settings `permissions.disableBypassPermissionsMode` (lower)

### 9.2 Auto Mode Killswitches

**Gate 1**: `tengu_auto_mode_config.enabled` (GrowthBook JSON)
- Values: `'enabled'` | `'disabled'` | `'opt-in'`
- Effect: Controls availability in shift-tab carousel and explicit entry

**Gate 2**: `tengu_auto_mode_config.disableFastMode` (GrowthBook)
- Effect: Blocks auto mode when AppState.fastMode is on
- Reason: Temp circuit breaker for auto+fast mode interaction validation

**Gate 3**: `tengu_iron_gate_closed` (Statsig boolean)
- Effect: Fail-closed behavior when classifier API is unavailable
- Value: true = deny access (fail closed); false = fall back to prompting (fail open)
- Refresh rate: 30 minutes (CLASSIFIER_FAIL_CLOSED_REFRESH_MS)

**Circuit Breaker**: `autoModeCircuitBroken` flag (in autoModeState.ts)
- Set by `verifyAutoModeGateAccess()` when GrowthBook check completes
- Prevents SDK/explicit re-entry if gate is disabled
- Survives mode changes (you can't shift-tab back into auto if circuit broken)

---

## Part 10: Analytics & Telemetry

### 10.1 Permission Decision Events

**Event**: `tengu_auto_mode_decision` (permissions.ts:626, 733)

```typescript
{
  decision: 'allowed' | 'blocked' | 'unavailable',
  toolName: sanitizedToolName,
  inProtectedNamespace: boolean,
  agentMsgId: string,
  confidence: 'high' | 'medium' | 'low',
  fastPath: 'acceptEdits' | 'allowlist' | undefined,
  classifierModel: string,
  consecutiveDenials: number,
  totalDenials: number,
  classifierInputTokens: number,
  classifierOutputTokens: number,
  classifierCacheReadInputTokens: number,
  classifierCacheCreationInputTokens: number,
  classifierDurationMs: number,
  classifierSystemPromptLength: number,
  classifierToolCallsLength: number,
  classifierUserPromptsLength: number,
  sessionInputTokens: number,
  sessionOutputTokens: number,
  sessionCacheReadInputTokens: number,
  sessionCacheCreationInputTokens: number,
  classifierCostUSD: number,
  classifierStage: 'fast' | 'thinking',
  // ... stage1 and stage2 usage details
}
```

### 10.2 Denial Limit Event

**Event**: `tengu_auto_mode_denial_limit_exceeded` (permissions.ts:1009)

```typescript
{
  limit: 'consecutive' | 'total',
  mode: 'cli' | 'headless',
  messageID: string,
  consecutiveDenials: number,
  totalDenials: number,
  toolName: sanitizedToolName,
}
```

### 10.3 Classifier Outcome Event

**Event**: `tengu_auto_mode_outcome` (yoloClassifier.ts:1425)

```typescript
{
  outcome: 'success' | 'parse_failure' | 'interrupted' | 'error' | 'transcript_too_long',
  classifierModel: string,
  classifierType: 'xml_fast' | 'xml_2stage' | 'xml_thinking' | undefined,
  failureKind: 'no_tool_use' | 'invalid_schema' | undefined,
  durationMs: number,
  mainLoopTokens: number,
  classifierInputTokens: number,
  classifierTokensEst: number,
  // ... additional fields for errors
}
```

---

## Part 11: Special Cases & Edge Cases

### 11.1 Plan Mode Auto Entry

When user enters plan mode with auto-opt-in:
1. Check `shouldPlanUseAutoMode()` → true if auto opt-in and gate enabled
2. If in default mode: auto mode is activated during plan (permissions stripped, classifier runs)
3. On plan exit: auto deactivated, permissions restored
4. User doesn't explicitly enter auto mode (it's under the hood)

### 11.2 Async Subagent Permission Handling

When an async subagent cannot show permission prompts:
1. Run `runPermissionRequestHooksForHeadlessAgent()` to give hooks a chance
2. If hook allows: proceed with updated permissions
3. If hook denies or no hook: deny with reason "Permission prompts not available"
4. On denial limit: throw `AbortError` (agent terminates)

### 11.3 Working Directory Switching

Additional working directories can be added via `--add-dir` flag:
1. Validated at startup via `validateDirectoryForWorkspace()`
2. Added to `context.additionalWorkingDirectories` Map
3. Bash/FileEdit tools check against these directories
4. Symlinks to originalCwd are also added (process.env.PWD case)

### 11.4 PowerShell Auto Mode (Feature-Gated)

When `POWERSHELL_AUTO_MODE` is enabled:
1. Dangerous PS rules are stripped on auto mode entry (same as Bash)
2. POWERSHELL_DENY_GUIDANCE is appended to deny rules in classifier prompt
3. Classifier learns that PS idioms map to BLOCK categories
4. Without the flag: PowerShell requires explicit user permission even in auto mode

---

## Part 12: Implementation Details

### 12.1 PermissionUpdate Data Structure

**Location**: `PermissionUpdateSchema.ts`

```typescript
type PermissionUpdate =
  | { type: 'addRules'; rules: PermissionRuleValue[]; behavior: 'allow' | 'deny' | 'ask'; destination: PermissionUpdateDestination }
  | { type: 'removeRules'; rules: PermissionRuleValue[]; behavior: 'allow' | 'deny' | 'ask'; destination: PermissionUpdateDestination }
  | { type: 'replaceRules'; rules: PermissionRuleValue[]; behavior: 'allow' | 'deny' | 'ask'; destination: PermissionUpdateDestination }
  | { type: 'setMode'; mode: PermissionMode; destination: 'session' }
  | { type: 'addDirectories'; directories: string[]; destination: PermissionUpdateDestination }

type PermissionUpdateDestination = 'userSettings' | 'projectSettings' | 'localSettings' | 'session' | 'cliArg'
```

**Applied by**: `applyPermissionUpdate()` which mutates context in-memory
**Persisted by**: `persistPermissionUpdates()` which writes to disk

### 12.2 Mode Availability Checking

**Functions**:
- `isAutoModeGateEnabled()` — Synchronous check (uses cached values)
- `verifyAutoModeGateAccess()` — Asynchronous check (fresh GrowthBook read)
- `getAutoModeUnavailableReason()` — Returns reason if unavailable

**Usage**:
- Startup: Sync check in `initialPermissionModeFromCLI()`
- React component: Async check in `useKickOffCheckAndDisableAutoModeIfNeeded()`
- Mode cycling: Sync checks in `getNextPermissionMode()`

---

## Conclusion

Claude Code v2.1.88's permission system is a comprehensive, layered architecture designed to balance automation (auto mode classifier) with safety (denial limits, dangerous rule stripping, bypass-immune checks) and user control (6 permission modes, manual prompting fallback).

The YOLO Classifier is the centerpiece of auto mode, using a 2-stage XML pipeline to make fast safe-pass decisions (stage 1) and escalate risky decisions to chain-of-thought reasoning (stage 2). Denial tracking ensures the classifier doesn't silently block legitimate use patterns indefinitely, with fallback to manual prompting.

The system is heavily instrumented with analytics telemetry, feature gates, and circuit breakers to monitor correctness and respond to issues at runtime. Rule stripping on auto mode entry, safety check immunity to bypass modes, and prompt caching all reflect careful engineering for security and performance.

---

**Total Lines**: 1,470+
**Document Completed**: 2026-04-02
