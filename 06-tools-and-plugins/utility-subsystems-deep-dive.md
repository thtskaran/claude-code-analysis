# Utility Subsystems Deep Dive

## Overview

This document provides an exhaustive analysis of Claude Code's four core utility subsystems:

1. **Ultraplan** — Multi-step task planning in remote CCR (Claude Code Remote) sessions
2. **DeepLink** — OS-level URI scheme handling (`claude-cli://`) for launching sessions
3. **DXT** — Plugin/extension packaging format and manifest validation
4. **FilePersistence** — File upload and persistence from local BYOC environments

These systems represent critical integration points in Claude Code's architecture, handling everything from session initialization to plugin distribution to file synchronization. Analysis is based on complete source inspection across 12 files.

---

## 1. ULTRAPLAN: Remote Planning System

### 1.1 What Ultraplan Is

Ultraplan is a multi-step task execution system that extends Claude Code's planning capabilities to remote CCR (Claude Code Remote) containers. When a user types the keyword "ultraplan" in a prompt, Claude Code:

1. Launches a dedicated remote session in a browser-based UI
2. Instructs the remote Claude to plan multi-step tasks
3. Shows the user an approval/rejection interface in a PlanModal
4. Returns the approved plan to the local session for execution

The system is implemented across two files:
- `ultraplan/ccrSession.ts` — Event polling, plan extraction, phase tracking
- `ultraplan/keyword.ts` — Trigger detection, keyword filtering

### 1.2 Core Data Structures

#### ExitPlanModeScanner (ccrSession.ts, lines 80-181)

A pure stateful classifier that ingests SDK message streams and tracks ExitPlanMode verdicts:

```typescript
class ExitPlanModeScanner {
  private exitPlanCalls: string[] = []           // Tool use IDs for ExitPlanMode calls
  private results = new Map<string, ToolResultBlockParam>()  // tool_use_id → tool_result
  private rejectedIds = new Set<string>()        // Rejected plans (user said "no")
  private terminated: { subtype: string } | null // Error subtypes (error_max_turns, etc)
  private rescanAfterRejection = false
  everSeenPending = false                        // Lifecycle tracking for diagnostics
}
```

**Invariant:** The scanner maintains a bidirectional mapping of tool use IDs to their results, tracking rejections in a separate set so the state machine can distinguish between "pending approval," "rejected and trying again," and "approved."

#### ScanResult Union (lines 50-56)

Represents the outcome of processing a batch of new events:

```typescript
type ScanResult =
  | { kind: 'approved'; plan: string }         // User approved in CCR
  | { kind: 'teleport'; plan: string }         // User approved locally (plan in rejection)
  | { kind: 'rejected'; id: string }           // User explicitly rejected
  | { kind: 'pending' }                        // Waiting for user decision
  | { kind: 'terminated'; subtype: string }    // CCR session ended
  | { kind: 'unchanged' }                      // No new status info
```

#### UltraplanPhase Enum (lines 66)

Represents the pill/detail-view state for the UI:

```typescript
type UltraplanPhase = 'running' | 'needs_input' | 'plan_ready'
```

**Transitions:**
- `running` → `needs_input` (turn ends, no ExitPlanMode call yet)
- `needs_input` → `running` (user replies in browser)
- `running` → `plan_ready` (ExitPlanMode emitted, no result yet)
- `plan_ready` → `running` (rejected)
- `plan_ready` → (session ends, return approved plan)

### 1.3 Event Processing: The ingest() Method

The core logic for ingesting a batch of SDKMessages and computing the next state (lines 101-180).

**Algorithm:**
1. Scan all `assistant` messages for `EXIT_PLAN_MODE_V2_TOOL_NAME` tool_use blocks → append to `exitPlanCalls`
2. Scan all `user` messages for tool_result blocks → store in results Map
3. Detect `result` events (turn-end signals) with non-success subtypes → mark `terminated`
4. **Decision precedence (lines 135-161):**
   - If new events or recent rejection: rescan from the latest non-rejected ExitPlanMode call
   - Approved > Teleport > Rejected > Pending > Unchanged
   - A single batch can contain both an approved result AND a subsequent terminated event; the approved plan is preserved

**Key insight:** pollRemoteSessionEvents paginates up to 50 pages per call, spanning seconds of CCR activity. A batch may contain multiple state transitions. The scanner handles this by scanning backward through `exitPlanCalls`, finding the last non-rejected ID, and checking if a result exists for it.

### 1.4 Plan Extraction

Two separate extraction functions handle the two approval paths:

#### extractApprovedPlan (lines 333-349)

Scrapes plan text from approved tool_result content using markers:

```typescript
// Markers (in preference order):
// "## Approved Plan (edited by user):\n<text>"
// "## Approved Plan:\n<text>"
```

When the ExitPlanModeV2Tool is invoked with default action, the remote writes the plan to a file inside CCR and calls ExitPlanMode({allowedPrompts}). The tool result echoes the plan back with a marker so local can extract it.

**Error handling:** Throws if neither marker is found — indicates the remote hit the empty-plan or isAgent branch, suggesting misconfiguration.

#### extractTeleportPlan (lines 321-329)

For the "teleport" path: when the user clicks "execute here" in the browser, the remote sends a rejection (is_error === true) with a sentinel marker:

```typescript
const ULTRAPLAN_TELEPORT_SENTINEL = '__ULTRAPLAN_TELEPORT_LOCAL__'
```

The plan text follows on the next line. Returns null if sentinel absent, so callers treat it as a normal rejection.

### 1.5 Main Poll Loop: pollForApprovedExitPlanMode

Entry point (lines 198-306) for listening to a CCR session until user approval or timeout.

**Arguments:**
- `sessionId` — Remote session ID
- `timeoutMs` — Poll deadline
- `onPhaseChange` — Callback for UI updates
- `shouldStop` — Caller's abort signal

**Algorithm:**
1. Create scanner and cursor (null)
2. Loop until deadline or approval:
   - Call `pollRemoteSessionEvents(sessionId, cursor)` with 3s backoff on transient failures
   - Max 5 consecutive failures → throws `UltraplanPollError('network_or_unknown')`
   - Ingest new events, update cursor
   - Check phase (lines 283-295):
     - `scanner.hasPendingPlan` → phase = 'plan_ready'
     - else `(sessionStatus === 'idle' || 'requires_action') && newEvents.length === 0` → phase = 'needs_input'
     - else phase = 'running'
   - Emit phase change callback if different
   - Return on approved/teleport
   - Throw on terminated
3. If timeout: throw based on everSeenPending flag to distinguish "never reached ExitPlanMode" (session startup failure) from "user rejected repeatedly"

**Key implementation detail:** The "quiet idle" check (line 283-285) is crucial. CCR briefly flips to 'idle' between tool turns. The scanner only trusts idle when `newEvents.length === 0`, so the phase snaps back to 'running' on the first poll that sees the user's reply event, even if `session_status` lags.

### 1.6 Keyword Detection

The keyword module (`ultraplan/keyword.ts`) provides filtered trigger detection to prevent false positives.

#### Filtered Trigger Detection (lines 46-95)

Function `findKeywordTriggerPositions(text, keyword)` detects "ultraplan" and "ultrareview" while skipping:

**Exclusion rules:**
1. **Quoted contexts:** Inside paired delimiters (backticks, quotes, angle brackets, braces, brackets, parens)
   - Single quotes treated specially: only delimit if not apostrophe (non-word char before/after)
2. **Path/identifier context:** Preceded/followed by `/`, `\`, `-`, or followed by `.` + word char
   - Prevents `src/ultraplan/foo.ts`, `--ultraplan-mode`, `ultraplan.tsx` from triggering
3. **Questions:** Followed by `?` (don't invoke on "what is ultraplan?")
4. **Slash commands:** Text starting with `/` is routed to processSlashCommand, not keyword detection
   - `/rename ultraplan foo` doesn't trigger

**Output:** Returns array of `{ word, start, end }` positions, matching the shape of thinking trigger positions.

#### API

- `findUltraplanTriggerPositions(text)` — Array of triggers
- `findUltrareviewTriggerPositions(text)` — Array of triggers
- `hasUltraplanKeyword(text)` — Boolean check
- `hasUltrareviewKeyword(text)` — Boolean check
- `replaceUltraplanKeyword(text)` — Replace first trigger "ultraplan" → "plan", preserving casing

### 1.7 Error Handling

#### UltraplanPollError (lines 34-44)

Custom error with structured failure reason:

```typescript
type PollFailReason =
  | 'terminated'              // CCR session ended
  | 'timeout_pending'         // Timeout after seeing ExitPlanMode
  | 'timeout_no_plan'         // Timeout before ExitPlanMode
  | 'extract_marker_missing'  // Extraction failed (malformed result)
  | 'network_or_unknown'      // Network issues or unknown errors
  | 'stopped'                 // Caller invoked shouldStop()
```

Includes `rejectCount` so callers know how many rejections user made before giving up.

### 1.8 Integration with Main Loop

Ultraplan is initiated when:
1. User types "ultraplan" in prompt → keyword detection fires
2. `replaceUltraplanKeyword()` modifies the prompt to "please plan..." (singular)
3. Main session calls `teleportToRemote()` with set_permission_mode control_request to enter plan mode
4. Remote Claude shows ExitPlanModeV2Tool in the toolbox
5. Local calls `pollForApprovedExitPlanMode()` in a background thread
6. UI shows PlanModal with approval/rejection buttons
7. Poll returns `{ plan, rejectCount, executionTarget }`
8. Plan is either executed locally or archived (if teleport)

---

## 2. DEEPLINK: OS-Level URI Scheme Handling

### 2.1 What DeepLink Is

DeepLink enables external applications (browsers, documentation sites, CI/CD systems) to launch Claude Code sessions with pre-filled prompts and working directories via the `claude-cli://` URI scheme.

**Example URIs:**
```
claude-cli://open
claude-cli://open?q=hello+world
claude-cli://open?q=fix+tests&repo=owner/repo
claude-cli://open?cwd=/path/to/project
```

The system is implemented across six files:
- `deepLink/parseDeepLink.ts` — URI parsing and validation
- `deepLink/protocolHandler.ts` — OS callback routing
- `deepLink/registerProtocol.ts` — OS registration (macOS/Linux/Windows)
- `deepLink/terminalLauncher.ts` — Terminal detection and subprocess spawning
- `deepLink/terminalPreference.ts` — Preference persistence
- `deepLink/banner.ts` — Security warning banner

### 2.2 URI Schema and Parsing

#### DeepLinkAction Type (parseDeepLink.ts, lines 25-29)

```typescript
type DeepLinkAction = {
  query?: string        // Pre-filled prompt (not auto-submitted)
  cwd?: string          // Working directory (absolute path)
  repo?: string         // GitHub owner/repo slug, resolved against MRU
}
```

#### parseDeepLink() Function (lines 84-153)

**Input validation pipeline:**
1. **Protocol check:** Must be `claude-cli://` or `claude-cli:` (with fallback normalization)
2. **Hostname validation:** Must be `open`
3. **CWD validation:**
   - Must be absolute (starts with `/` or `X:\` on Windows)
   - No control characters (0x00-0x1F, 0x7F) — prevents command injection
   - Max 4096 bytes (PATH_MAX)
4. **Repo slug validation:**
   - Pattern: `^[\w.-]+/[\w.-]+$` (exactly one slash)
   - Prevents path traversal
5. **Query validation:**
   - Sanitized with `partiallySanitizeUnicode()` to strip hidden characters (zero-width spaces, RTL markers, etc.)
   - No control characters
   - Max 5000 bytes (Windows cmd.exe 8191 limit after wrapping + expansion)

**Error handling:** All validation errors throw descriptive messages so the OS UI can show the user why the link failed.

#### buildDeepLink() Function (lines 158-170)

Reverse of parsing — constructs a `claude-cli://open` URL from an action object.

### 2.3 Protocol Handler Flow

#### handleDeepLinkUri (protocolHandler.ts, lines 36-75)

Entry point when OS invokes `claude --handle-uri <url>`. Runs in headless context (no TTY).

**Algorithm:**
1. Parse URI via parseDeepLink()
2. Resolve cwd via resolveCwd() (precedence: explicit > repo lookup > home)
3. If repo was resolved, read last git fetch time (FETCH_HEAD mtime)
4. Launch terminal via launchInTerminal(), passing:
   - claudePath (process.execPath, the running binary)
   - query, cwd, repo, lastFetchMs
5. Return exit code (0 = success, 1 = error)

#### resolveCwd (lines 117-136)

Implements three-tier precedence:
1. **Explicit cwd:** Use as-is
2. **Repo slug:** Look up in `githubRepoPaths` config (MRU list), return first existing path
3. **Fallback:** Use home directory

Returns tuple `{ cwd, resolvedRepo? }` so launched instance knows which clone was selected (for banner freshness warnings).

#### handleUrlSchemeLaunch (lines 84-105)

macOS-specific: when the OS launches Claude as the app bundle's executable via URL handler, uses NAPI module to read the Apple Event:

```typescript
if (process.env.__CFBundleIdentifier === MACOS_BUNDLE_ID) {
  const { waitForUrlEvent } = await import('url-handler-napi')
  const url = waitForUrlEvent(5000)
  return await handleDeepLinkUri(url)
}
```

Returns null if not a URL launch (normal CLI invocation).

### 2.4 OS Protocol Registration

#### macOS (registerProtocol.ts, lines 75-138)

Creates a minimal .app bundle in `~/Applications/Claude Code URL Handler.app`:

**Components:**
1. `Info.plist` — Declares CFBundleURLTypes with scheme `claude-cli`
2. `Contents/MacOS/claude` — Symlink to the signed claude binary (avoids shipping separate executable)
3. LaunchServices registration via `/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister`

**Why symlink?** Avoids shipping a separate binary that would need signing and allowlisting by endpoint security tools (Santa, etc).

**Commit marker:** The symlink is written LAST among throwing fs calls. If Info.plist write fails, no symlink exists → isProtocolHandlerCurrent returns false → next session retries.

#### Linux (lines 144-180)

Creates `.desktop` file in `$XDG_DATA_HOME/applications/` (default `~/.local/share/applications/`):

```ini
[Desktop Entry]
Name=Claude Code URL Handler
Comment=Handle claude-cli:// deep links for Claude Code
Exec="/path/to/claude" --handle-uri %u
Type=Application
MimeType=x-scheme-handler/claude-cli;
```

Registers via `xdg-mime default`:
```bash
xdg-mime default claude-code-url-handler.desktop x-scheme-handler/claude-cli
```

**Headless handling:** On WSL/Docker/CI where xdg-utils isn't installed, the .desktop file is enough (some apps read MimeType directly). No failure — artifact check short-circuits next session.

#### Windows (lines 185-209)

Three registry keys under `HKEY_CURRENT_USER\Software\Classes\claude-cli`:
1. Default value: `URL:Claude Code URL Handler`
2. `URL Protocol` key: empty string
3. `shell\open\command` default value: `"C:\path\to\claude.exe" --handle-uri "%1"`

All written via `reg add` with `/f` (force) flag.

#### isProtocolHandlerCurrent() (lines 263-290)

Verifies registration is up-to-date by reading back the artifact:
- **macOS:** Readlink symlink target, compare with expected binary path
- **Linux:** Read .desktop Exec line, check for exact match
- **Windows:** Query registry, check command value

**Why read back?** Install paths can change (installer update, different method) or artifacts can be deleted. Reading detects stale registrations so they self-heal.

#### ensureDeepLinkProtocolRegistered (lines 298-348)

Fire-and-forget background task that:
1. Checks feature flag `tengu_lodestone_enabled`
2. Checks if registration is current
3. If stale/missing, attempts registration
4. On EACCES/ENOSPC (deterministic errors), writes failure marker with 24h backoff to prevent spam

### 2.5 Terminal Detection and Launching

#### Terminal Detection (terminalLauncher.ts, lines 183-194)

Platform-specific, ordered by preference:

**macOS (lines 64-121):**
1. Check stored preference from previous interactive session (TERM_PROGRAM env var)
2. Check `$TERM_PROGRAM` env var (iTerm, Ghostty, etc)
3. Use mdfind (Spotlight) to check installed bundles by bundle ID
4. Fallback: check /Applications directly
5. Last resort: Terminal.app (always available)

**Linux (lines 127-152):**
1. Check `$TERMINAL` env var
2. Check `x-terminal-emulator` alternative
3. Walk priority list: ghostty, kitty, alacritty, wezterm, gnome-terminal, konsole, xfce4-terminal, mate-terminal, tilix, xterm

**Windows (lines 157-178):**
1. Windows Terminal (wt.exe) — modern, preferred
2. PowerShell 7+ (pwsh.exe) — separate install
3. Windows PowerShell 5.1 (powershell.exe) — built-in
4. cmd.exe — always available

#### Launch Paths: Pure argv vs Shell String

Two fundamentally different approaches based on terminal capabilities:

**Pure argv (no shell, no quoting needed):**
- macOS: Ghostty, Alacritty, Kitty, WezTerm (via `open -na --args`)
- Linux: all ten terminals
- Windows: Windows Terminal

**Shell string (user input shell-quoted):**
- macOS: iTerm, Terminal.app (AppleScript has no argv interface)
- Windows: PowerShell, cmd.exe (no argv exec mode)

For shell paths, `shellQuote()` (POSIX), `psQuote()` (PowerShell), and `cmdQuote()` (cmd.exe) are load-bearing — correctness is non-negotiable.

#### macOS Terminal Launch Paths (launchMacosTerminal, lines 255-357)

**iTerm (lines 266-287):**
```applescript
tell application "iTerm"
  if running then
    create window with default profile
  else
    activate
  end if
  tell current session of current window
    write text '<shell-quoted-command>'
  end tell
end tell
```

Checks if running first — if already running (even with zero windows), creates new window; if not, activate lets startup open the first.

**Terminal (lines 290-300):**
```applescript
tell application "Terminal"
  do script '<shell-quoted-command>'
  activate
end tell
```

**Ghostty/Alacritty/Kitty/WezTerm (lines 306-345):**
All use pure argv via `open -na <App> --args ...`:
```bash
open -na Ghostty --args --window-save-state=never --working-directory=<cwd> -e <claude> <args...>
```

Each has terminal-specific flags for cwd:
- Ghostty: `--working-directory=<cwd>`
- Alacritty: `--working-directory <cwd>`
- Kitty: `--directory <cwd>`
- WezTerm: `start --cwd <cwd> --`

#### Linux Terminal Launch Paths (launchLinuxTerminal, lines 359-417)

All pure argv. Each terminal has native cwd flags (or spawn({cwd}) for those without):
- gnome-terminal: `--working-directory=<cwd>`
- konsole: `--workdir <cwd>`
- kitty: `--directory <cwd>`
- wezterm: `start --cwd <cwd> --`
- alacritty: `--working-directory <cwd>`
- ghostty: `--working-directory=<cwd>`
- xfce4/mate: `--working-directory=<cwd>`
- tilix: `--working-directory=<cwd>`
- xterm/x-terminal-emulator/generic: spawn({cwd}) — inherit terminal's cwd

#### Windows Terminal Launch Paths (launchWindowsTerminal, lines 419-469)

**Windows Terminal (pure argv):**
```powershell
wt.exe -d <cwd> -- <claude.exe> <args...>
```

**PowerShell (shell string):**
```powershell
pwsh.exe -NoExit -Command "Set-Location '<cwd>'; & '<claude>' <arg1> <arg2>..."
```

Uses `psQuote()` — single-quoted strings with no escape sequences (only `''` for literal quote).

**cmd.exe (shell string):**
```cmd
cmd.exe /k "cd /d <cwd> && <claude.exe> <args...>"
```

Uses `cmdQuote()` — strips `"` characters (can't be safely escaped), doubles `%` (env var expansion), doubles trailing `\` (child process sees backslash before our closing quote).

**Bypass for cmd.exe:** libuv's default quoting assumes MSVCRT rules. For cmd.exe, set `windowsVerbatimArguments: true` to skip automatic escaping since our `cmdQuote()` is already correct.

#### spawnDetached() (lines 476-499)

Spawns terminal detached so handler process exits immediately:

```typescript
const child = spawn(command, args, {
  detached: true,           // Allows terminal to outlive handler
  stdio: 'ignore',          // Don't inherit stdio
  cwd: opts.cwd,
  windowsVerbatimArguments: opts.windowsVerbatimArguments,
})
child.once('spawn', () => {
  child.unref()             // Detach completely
  resolve(true)
})
```

Resolves false on spawn failure (ENOENT, EACCES) rather than throwing.

### 2.6 Security Banner

#### buildDeepLinkBanner (banner.ts, lines 54-75)

Constructs a multi-line warning for deep-link-originated sessions. Always shows the working directory (so user sees which CLAUDE.md will load):

```
This session was opened by an external deep link in /path/to/project
```

Optional lines based on context:
- **Repo freshness (if `?repo=` was used):**
  ```
  Resolved owner/repo from local clones · last fetched 2 days ago
  ```
  or
  ```
  Resolved owner/repo from local clones · last fetched never — CLAUDE.md may be stale
  ```
- **Prompt length (if `?q=` was used):**
  ```
  The prompt below was supplied by the link — review carefully before pressing Enter.
  ```
  or (if >1000 chars)
  ```
  The prompt below (4521 chars) was supplied by the link — scroll to review the entire prompt before pressing Enter.
  ```

#### readLastFetchTime (lines 88-102)

Reads `.git/FETCH_HEAD` mtime to show git freshness:

```typescript
const gitDir = await getGitDir(cwd)
const commonDir = await getCommonDir(gitDir)  // For git worktrees
const [local, common] = await Promise.all([...])
return local > common ? local : common  // Whichever is newer
```

For git worktrees, checks both the worktree's FETCH_HEAD and the main repo's (shared) FETCH_HEAD, returning the newer.

### 2.7 Terminal Preference Persistence

#### updateDeepLinkTerminalPreference (terminalPreference.ts, lines 38-54)

Called fire-and-forget from interactive startup to capture the terminal the user is actually running:

```typescript
const termProgram = process.env.TERM_PROGRAM  // e.g., "iTerm.app", "Apple_Terminal"
const app = TERM_PROGRAM_TO_APP[termProgram.toLowerCase()]  // Normalize to app name
saveGlobalConfig(current => ({ ...current, deepLinkTerminal: app }))
```

Stored in global config (per-machine, not synced). The handler's detectMacosTerminal reads this so headless LaunchServices context doesn't lose the user's preference.

---

## 3. DXT: Plugin Packaging Format

### 3.1 What DXT Is

DXT (Distribution eXTension) is Claude Code's format for distributing plugins as single-file packages. A DXT file is a ZIP archive containing:
- `manifest.json` — Plugin metadata (author, name, version, MCPs, etc)
- Plugin source code, assets, and supporting files

DXT is built on the MCPB (Model Context Protocol Bundle) standard from Anthropic. The system is implemented across two files:
- `dxt/helpers.ts` — Manifest parsing, validation, extension ID generation
- `dxt/zip.ts` — ZIP extraction, validation, security scanning

### 3.2 Manifest Validation

#### validateManifest (helpers.ts, lines 13-34)

Validates a manifest JSON object against the MCPB schema:

```typescript
export async function validateManifest(
  manifestJson: unknown,
): Promise<McpbManifest>
```

**Process:**
1. Lazy-import `@anthropic-ai/mcpb` (deferred to avoid loading ~700KB of zod bound closures at startup)
2. Use `McpbManifestSchema.safeParse(manifestJson)` for validation
3. On failure, flatten and join all errors into a single message
4. Return parsed manifest on success

**Why lazy-import?** The mcpb package uses zod v3, which eagerly creates 24 `.bind(this)` closures per schema instance (~300 instances in schemas.js and schemas-loose.js). Deferring keeps ~700KB of bound closures out of the startup heap for sessions that never touch .dxt files.

#### parseAndValidateManifestFromText (lines 39-51)

1. Parse JSON from string
2. Call validateManifest()
3. Throw with context if JSON parsing fails

#### parseAndValidateManifestFromBytes (lines 56-61)

1. Decode bytes as UTF-8
2. Call parseAndValidateManifestFromText()

### 3.3 Extension ID Generation

#### generateExtensionId (helpers.ts, lines 67-88)

Creates a unique, canonical ID from author and extension name:

```typescript
function generateExtensionId(
  manifest: McpbManifest,
  prefix?: 'local.unpacked' | 'local.dxt',
): string
```

**Process:**
1. Extract `manifest.author.name` and `manifest.name`
2. Sanitize each:
   - Lowercase
   - Replace whitespace with `-`
   - Strip non-alphanumeric/dash/underscore/dot
   - Collapse multiple dashes
   - Strip leading/trailing dashes
3. Combine: `{sanitizedAuthor}.{sanitizedName}` or `{prefix}.{sanitizedAuthor}.{sanitizedName}`

**Example:**
- Author: "Acme Corp", Name: "Code Linter" → `acme-corp.code-linter`
- With prefix: `local.dxt.acme-corp.code-linter`

**Consistency:** Same algorithm as directory backend for parity across sources (marketplace, local unpacked, DXT archives).

### 3.4 ZIP Security and Validation

#### Security Model

DXT files are ZIP archives, which are notorious for zip bombs (excessive compression, path traversal, symlink attacks). The zip.ts module implements comprehensive validation.

#### File Size Limits (lines 7-12)

```typescript
const LIMITS = {
  MAX_FILE_SIZE: 512 * 1024 * 1024,        // 512MB per file
  MAX_TOTAL_SIZE: 1024 * 1024 * 1024,      // 1024MB total
  MAX_FILE_COUNT: 100000,                   // File count
  MAX_COMPRESSION_RATIO: 50,                // Zip bomb ratio
  MIN_COMPRESSION_RATIO: 0.5,               // Already-compressed content
}
```

#### validateZipFile (lines 63-102)

Per-file validation called as the filter during unzip:

```typescript
function validateZipFile(
  file: ZipFileMetadata,
  state: ZipValidationState,
): FileValidationResult
```

**Checks:**
1. **File count:** Increment counter, fail if > MAX_FILE_COUNT
2. **Path safety:** Call `isPathSafe(file.name)` (prevents traversal/absolute paths)
3. **Individual file size:** Fail if > 512MB
4. **Total uncompressed size:** Fail if cumulative > 1024MB
5. **Compression ratio:** Compute `totalUncompressed / compressedSize`, fail if > 50:1

Returns `{ isValid: true }` or `{ isValid: false, error: string }`.

#### isPathSafe (lines 44-58)

Prevents path traversal and absolute paths:

```typescript
function isPathSafe(filePath: string): boolean {
  if (containsPathTraversal(filePath)) return false    // Check for ".." segments
  const normalized = normalize(filePath)               // Resolve "." segments
  if (isAbsolute(normalized)) return false             // No absolute paths
  return true
}
```

#### unzipFile (lines 113-141)

Main unzip function with integrated validation:

```typescript
export async function unzipFile(
  zipData: Buffer,
): Promise<Record<string, Uint8Array>>
```

**Process:**
1. Lazy-import fflate (deferred to avoid ~196KB of top-level lookup tables at startup)
2. Create validation state (fileCount = 0, totalUncompressedSize = 0, compressedSize = zipData.length)
3. Call `unzipSync()` with filter function
4. Filter throws on any validation failure, stopping extraction immediately
5. Return record of path → Uint8Array

**Why sync?** Avoids fflate worker termination crashes in bun (async workers can hang).

#### parseZipModes (lines 160-203)

Extracts Unix file permissions from ZIP central directory. fflate's `unzipSync()` returns only data, losing executable bits (everything becomes 0644). This helper reconstructs them for parity with git-clone paths.

**Algorithm:**
1. Find End of Central Directory (EOCD) record (signature 0x06054b50) — scan backwards in trailing ~22 + 65535 bytes
2. Read entry count and central directory offset from EOCD
3. Walk central directory entries (sig 0x02014b50)
4. For each entry:
   - Read `versionMadeBy` (host OS in high byte)
   - If high byte === 3 (Unix), extract mode from `externalAttr` high 16 bits
   - Store in result map
5. Return map of path → mode

**Limitation:** Doesn't handle ZIP64 (archives >4GB or >65535 entries), but marketplace DXTs (~3.5MB) and MCPB bundles are tiny.

#### readAndUnzipFile (lines 209-226)

High-level wrapper that reads file from disk and unzips:

```typescript
export async function readAndUnzipFile(
  filePath: string,
): Promise<Record<string, Uint8Array>>
```

1. Read file asynchronously
2. Call unzipFile()
3. Wrap errors with context (distinguish ENOENT from other errors)

---

## 4. FILEPERSISTENCE: Local to Remote File Sync

### 4.1 What FilePersistence Is

FilePersistence is a system for syncing files created during a Claude Code session back to the Files API (Anthropic's file storage). It's active only in BYOC (Bring Your Own Compute) environments where files are created locally on the user's machine and need to be uploaded to Anthropic's infrastructure.

**Lifecycle:**
1. User starts Claude Code session in BYOC environment
2. Session creates files in `{cwd}/{sessionId}/outputs/`
3. At end of each turn, scanOutputsDirectory finds modified files
4. Files are uploaded to Files API via uploadSessionFiles
5. File IDs are collected and returned to the turn loop

The system is implemented across two files:
- `filePersistence/filePersistence.ts` — Main orchestration, BYOC vs Cloud branching
- `filePersistence/outputsScanner.ts` — Directory scanning, mtime-based filtering

### 4.2 Architecture: Environment Branching

#### getEnvironmentKind (outputsScanner.ts, lines 25-31)

Determines deployment mode from `CLAUDE_CODE_ENVIRONMENT_KIND`:

```typescript
type EnvironmentKind = 'byoc' | 'anthropic_cloud'
```

- **BYOC:** User's own infrastructure, needs file upload
- **anthropic_cloud** (1P): Anthropic's cloud, rclone handles sync (xattr-based file IDs)

Only BYOC mode currently uploads. Cloud mode is a placeholder (lines 247-250) for future xattr-based file ID reading.

### 4.3 Main Orchestration

#### runFilePersistence (filePersistence.ts, lines 51-144)

Entry point called at end of each turn:

```typescript
export async function runFilePersistence(
  turnStartTime: TurnStartTime,
  signal?: AbortSignal,
): Promise<FilesPersistedEventData | null>
```

**Arguments:**
- `turnStartTime` — Timestamp from turn start (for mtime filtering)
- `signal` — AbortSignal for cancellation

**Process:**
1. Check environment: return null if not BYOC
2. Get session access token: return null if missing
3. Get CLAUDE_CODE_REMOTE_SESSION_ID: log error and return null if missing
4. Assemble FilesApiConfig (oauth token, session ID)
5. Compute outputsDir: `{cwd}/{sessionId}/outputs/`
6. Check abort signal: return null if already aborted
7. Call executeBYOCPersistence() or executeCloudPersistence()
8. Check result: return null if no files and no failures
9. Log completion event with success/failure counts and duration
10. Return result or throw

**Error handling:** All errors are caught, logged, and returned as FilesPersistedEventData with empty successes and a single failure entry pointing to outputsDir.

#### executeBYOCPersistence (lines 150-240)

BYOC-specific workflow:

```typescript
async function executeBYOCPersistence(
  turnStartTime: TurnStartTime,
  config: FilesApiConfig,
  outputsDir: string,
  signal?: AbortSignal,
): Promise<FilesPersistedEventData>
```

**Algorithm:**
1. Find modified files via `findModifiedFiles(turnStartTime, outputsDir)`
2. Return empty result if no files
3. Check abort signal: return empty if aborted
4. Check file count limit: return failure if > FILE_COUNT_LIMIT (1000 files)
5. Build file list with relative paths:
   ```typescript
   filesToProcess = modifiedFiles.map(filePath => ({
     path: filePath,
     relativePath: relative(outputsDir, filePath),
   }))
   ```
6. Filter security: skip files where relativePath starts with `..` (outside outputs dir)
7. Call `uploadSessionFiles(filesToProcess, config, DEFAULT_UPLOAD_CONCURRENCY)` (concurrency: 10)
8. Separate results into persistedFiles and failedFiles
9. Return summary

**Concurrency:** 10 concurrent uploads (balanced between throughput and API rate limits).

#### executeFilePersistence (lines 256-269)

Wrapper that calls runFilePersistence and emits result via callback:

```typescript
export async function executeFilePersistence(
  turnStartTime: TurnStartTime,
  signal: AbortSignal,
  onResult: (result: FilesPersistedEventData) => void,
): Promise<void>
```

Catches and logs errors, invokes onResult only if result is non-null.

### 4.4 File Discovery

#### findModifiedFiles (outputsScanner.ts, lines 62-126)

Scans outputs directory recursively for files modified since turn start:

```typescript
export async function findModifiedFiles(
  turnStartTime: TurnStartTime,
  outputsDir: string,
): Promise<string[]>
```

**Algorithm:**
1. Recursively list outputsDir with `fs.readdir(..., { withFileTypes: true, recursive: true })`
2. Return empty array if directory doesn't exist
3. Filter entries:
   - Skip symlinks (security)
   - Keep regular files only
   - Build full paths using parentPath (Node 20+) or fallback
4. Parallelize stat calls: `Promise.all(filePaths.map(lstat))`
5. For each stat result:
   - Skip if became symlink between readdir and stat (race condition)
   - Keep if `mtimeMs >= turnStartTime`
6. Return filtered list

**Race condition handling:** Between readdir and stat, a file might become a symlink. The code detects this and skips.

**Concurrency:** All stat calls parallelized for efficiency.

### 4.5 Data Types

#### TurnStartTime (inferred from filePersistence.ts lines 47)

```typescript
type TurnStartTime = number  // milliseconds since epoch
```

Captured at turn start so only files modified during the turn are persisted.

#### FilesPersistedEventData (inferred)

```typescript
type FilesPersistedEventData = {
  files: PersistedFile[]      // Successfully uploaded files
  failed: FailedPersistence[] // Upload failures
}

type PersistedFile = {
  filename: string            // File path
  file_id: string             // Files API ID
}

type FailedPersistence = {
  filename: string
  error: string
}
```

#### FilesApiConfig (from filesApi.ts integration)

```typescript
type FilesApiConfig = {
  oauthToken: string          // Session access token
  sessionId: string            // CLAUDE_CODE_REMOTE_SESSION_ID
}
```

### 4.6 Feature Flag and Gating

#### isFilePersistenceEnabled (lines 278-287)

Checks if file persistence should run:

```typescript
export function isFilePersistenceEnabled(): boolean {
  return feature('FILE_PERSISTENCE') &&
         getEnvironmentKind() === 'byoc' &&
         !!getSessionIngressAuthToken() &&
         !!process.env.CLAUDE_CODE_REMOTE_SESSION_ID
}
```

**Gating conditions:**
1. Feature flag ON (bun:bundle feature)
2. Environment is BYOC
3. Session access token available
4. CLAUDE_CODE_REMOTE_SESSION_ID set

Only public-api/sessions users trigger file persistence, not CLI users.

#### Constants (inferred from lines 34)

```typescript
const OUTPUTS_SUBDIR = 'outputs'
const FILE_COUNT_LIMIT = 1000
const DEFAULT_UPLOAD_CONCURRENCY = 10
```

### 4.7 Integration with Turn Loop

File persistence runs at the end of each turn:
1. Capture `turnStartTime = Date.now()` at turn start
2. Model executes tools, creates files in `outputs/` subdirectory
3. After model finishes, call `executeFilePersistence(turnStartTime, signal, onResult)`
4. Upload runs in background/parallel with other end-of-turn tasks
5. Result is collected and reported in turn metadata

---

## 5. Cross-System Integration

### 5.1 Main Loop Entry Points

**Ultraplan:**
- Triggered by keyword detection during input processing
- Launches remote session via teleportToRemote()
- Polls remote for approval via pollForApprovedExitPlanMode()

**DeepLink:**
- Initial session launch (OS calls `--handle-uri`)
- Detected via CLI flag `--deep-link-origin`
- Banner shown to user before any interaction

**DXT:**
- Plugin loading at session init
- Manifest validated during plugin discovery
- ZIP extracted and files cached

**FilePersistence:**
- Runs at end of each turn (fire-and-forget)
- Parallel with model inference on next turn
- Cancellable via AbortSignal

### 5.2 Shared Utilities

**Logging:**
- `logForDebugging()` — All modules use for debug output
- `logError()` — For errors in main session
- `logEvent()` — Analytics events (deepLink registration, ultraplan status, persistence stats)

**Path handling:**
- `getCwd()` — Get current working directory
- `getDisplayPath()` — Shorten paths for UI
- Git utilities — For repo detection, last fetch time

**Error handling:**
- `errorMessage()` — Normalize error objects to strings
- `isENOENT()` — Distinguish file-not-found from other errors
- Error types with context throughout

**Configuration:**
- `getGlobalConfig()` / `saveGlobalConfig()` — Per-machine config (deeplink terminal preference)
- `getInitialSettings()` — User settings (disableDeepLinkRegistration flag)

### 5.3 Security Boundaries

**DeepLink → Shell:**
- Shell quoting (shellQuote, psQuote, cmdQuote) is load-bearing
- Pure argv paths preferred (no shell interpretation)
- Control character filtering on input

**DXT → ZIP extraction:**
- Path traversal prevention (no "..", no absolute paths)
- File size limits (per-file, total, compression ratio)
- Symlink skipping (don't follow)

**FilePersistence → Output directory:**
- Only scan outputs/ subdirectory
- Skip symlinks
- Check relativePath doesn't escape via ".."

**Ultraplan → Plan extraction:**
- Sentinel marker check (prevents executing wrong text)
- Marker presence validation (throws if missing)

---

## 6. Feature Flags and Configuration

### 6.1 Feature Flags

| Flag | Module | Purpose |
|------|--------|---------|
| `tengu_lodestone_enabled` | DeepLink | Enable deep link protocol registration |
| `FILE_PERSISTENCE` | FilePersistence | Enable file upload system |

### 6.2 Environment Variables

| Variable | Module | Purpose |
|----------|--------|---------|
| `CLAUDE_CODE_ENVIRONMENT_KIND` | FilePersistence | BYOC vs Cloud mode |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | FilePersistence, others | Remote session ID |
| `TERM_PROGRAM` | DeepLink | Terminal detection hint |
| `__CFBundleIdentifier` | DeepLink | macOS URL launch detection |

### 6.3 Configuration Files

| Path | Module | Purpose |
|------|--------|---------|
| `~/.claude/config.json` | DeepLink | Global config (per-machine, terminal preference) |
| `~/.claude.json` | Settings | User settings (synced, disableDeepLinkRegistration) |

---

## 7. Performance Considerations

### 7.1 Lazy Imports

**MCPB manifest validation (dxt/helpers.ts):**
- Deferred import of `@anthropic-ai/mcpb`
- Saves ~700KB startup cost for non-plugin sessions

**fflate zip extraction (dxt/zip.ts):**
- Deferred import of `fflate`
- Saves ~196KB startup cost for non-DXT sessions

### 7.2 Concurrency

**File persistence uploads (filePersistence.ts):**
- 10 concurrent uploads (DEFAULT_UPLOAD_CONCURRENCY)
- Parallelized stat calls during directory scan

**Terminal detection (terminalLauncher.ts):**
- macOS mdfind and /Applications checks run in parallel
- Linux terminal check uses which() sequentially (short list)

### 7.3 Polling Strategy

**Ultraplan polling (ultraplan/ccrSession.ts):**
- 3-second interval (POLL_INTERVAL_MS)
- Paginated events (up to 50 pages per call)
- Backoff on transient failures (max 5 consecutive)

---

## 8. Error Handling Summary

### 8.1 By Module

| Module | Error Types | Handling |
|--------|-------------|----------|
| Ultraplan | UltraplanPollError | Structured failures, rejectCount tracking |
| DeepLink | URI parsing errors | Throw with context, exit code 1 |
| DXT | Manifest validation | Throw with error details |
| FilePersistence | Upload failures | Collect as FailedPersistence, continue |

### 8.2 Transient vs Permanent

- **Transient (retry):** Network errors, temporary ENOENT
- **Permanent (give up, log, backoff):** EACCES, ENOSPC (permissions), invalid input, validation failures

---

## 9. Security Analysis

### 9.1 Threat Model

| Threat | Mitigation |
|--------|-----------|
| Prompt injection via deep link | Control char filtering, long-prompt warning |
| Path traversal in DXT | isPathSafe check, ".." rejection, normalization |
| Zip bomb | Compression ratio limit (50:1), file size limits |
| Command injection via shell | Shell-specific quoting (shellQuote, psQuote, cmdQuote) |
| Symlink attacks | Symlink skipping in ZIP extraction and file scanning |
| Stale CLAUDE.md | Git freshness check (FETCH_HEAD mtime) |

### 9.2 Load-Bearing Code

- **Shell quoting functions:** cmdQuote, psQuote, shellQuote (correctness is non-negotiable)
- **Path safety check:** isPathSafe (prevents directory traversal)
- **Zip validation filter:** validateZipFile (runs during extraction, can stop immediately)
- **Marker extraction:** extractApprovedPlan (throws if marker missing, prevents wrong plan execution)

---

## 10. Testing Opportunities

### 10.1 Suggested Unit Tests

**Ultraplan:**
- Scanner state machine (pending → approved, pending → rejected → pending → approved)
- Marker extraction (with and without edits, missing marker error)
- Phase transitions (running → needs_input → running)

**DeepLink:**
- URI parsing (valid/invalid URIs, control char rejection, max length enforcement)
- Shell quoting (spaces, quotes, metacharacters in paths/queries)
- Terminal detection (mock TERM_PROGRAM, check priority order)

**DXT:**
- Manifest validation (valid/invalid schemas, error flattening)
- Extension ID generation (sanitization, prefix handling)
- ZIP security (zip bomb detection, path traversal rejection, compression ratio check)

**FilePersistence:**
- File discovery (mtime filtering, symlink skipping, race condition handling)
- Upload concurrency (parallelization correctness)
- Security (relativePath escape detection)

### 10.2 Integration Tests

- Full deep link flow (parse → register → detect terminal → launch)
- Ultraplan poll loop (event batches, phase changes, timeout scenarios)
- DXT extraction (large archives, edge cases like nested symlinks)
- File persistence (scan → upload → collection in BYOC mode)

---

## 11. Conclusion

These four utility subsystems represent sophisticated engineering addressing non-trivial problems:

- **Ultraplan** solves multi-step planning in a distributed system with polling, rejection handling, and lifecycle tracking
- **DeepLink** bridges OS-level URI schemes to terminal launch across three platforms with security warnings and terminal preference persistence
- **DXT** implements secure plugin distribution with comprehensive zip bomb detection and path traversal prevention
- **FilePersistence** synchronizes files from BYOC to cloud infrastructure with concurrent uploads and abort signal support

All four prioritize security (input validation, shell escaping, path safety) and performance (lazy imports, concurrency, efficient scanning). They integrate seamlessly with the main session loop via feature flags, environment variables, and callbacks.

