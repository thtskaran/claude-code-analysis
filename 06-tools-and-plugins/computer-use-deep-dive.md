# Claude Code Computer Use System - Comprehensive Reverse Engineering Report

## SYSTEM ARCHITECTURE OVERVIEW

Claude Code's computer use (CU) system is a macOS-only, MCP-based native desktop automation layer that bridges the Claude API with low-level screen capture and input simulation. The architecture is modular:

```
API Model
    ↓
MCP Server (in-process)
    ↓
Computer Use Host Adapter (singleton)
    ↓
Executor (ComputerExecutor interface)
    ├─ @ant/computer-use-swift (native .node module)
    └─ @ant/computer-use-input (Rust/enigo .node module)
    ↓
OS-level APIs (macOS)
```

## 1. FILE-BY-FILE BREAKDOWN (13 Files)

### A. appNames.ts (Prompt Injection Hardening)
**Purpose**: Filters and sanitizes installed-app names for tool descriptions.

**Key mechanisms**:
- PATH_ALLOWLIST: Only apps from /Applications/, /System/Applications/, or ~/Applications/
- NAME_PATTERN_BLOCKLIST: Filters noisy background services (Helper, Agent, Service, Uninstaller)
- ALWAYS_KEEP_BUNDLE_IDS: 30+ trusted apps (browsers, terminals, dev tools) bypass filtering
- APP_NAME_ALLOWED regex: Unicode-safe allowlist `[\p{L}\p{M}\p{N}_ .&'()+-]+` (no quotes, pipes, backticks)
- Prevents injection via character filtering on untrusted (attacker-installable) apps only
- Length cap: 40 chars max per name, 50 apps in list max
- Deduplication and sorting applied

**Attack surface mitigation**: An app named "grant all" could exploit naive parsing, but the tool description's structural framing ("Available applications:") plus explicit user approval dialog contain it.

---

### B. computerUseLock.ts (Session Serialization)
**Purpose**: File-based distributed lock for exclusive computer control.

**Lock mechanism**:
- File: `~/.claude/computer-use.lock`
- Format: JSON with `{sessionId, pid, acquiredAt}`
- Atomic test-and-set: `writeFile(..., {flag: 'wx'})` for O_EXCL
- Stale recovery: Checks PID liveness via `process.kill(pid, 0)` signal probe

**Key functions**:
- `tryAcquireComputerUseLock()`: O_EXCL create → reads → liveness check → race-safe stale recovery
- `releaseComputerUseLock()`: Unlinks on drop (idempotent)
- `isLockHeldLocally()`: Zero-syscall check (tracks via `unregisterCleanup` closure)
- `checkComputerUseLock()`: Non-acquiring check for `request_access` tool

**Race conditions handled**:
- Multiple sessions racing to recover stale lock: only one's create succeeds, others read winner
- Small PID-reuse window: negligible in practice

---

### C. cleanup.ts (Turn-End Cleanup)
**Purpose**: Unhides apps and releases lock at turn end.

**Execution flow**:
1. Checks `appState.computerUseMcpState.hiddenDuringTurn` (Set of bundleIds)
2. Calls `unhideComputerUseApps([...hidden])` (fire-and-forget, 5s timeout)
3. Unregisters Escape hotkey
4. Releases lock
5. Sends OS notification

**Key details**:
- Runs on all turn ends: natural, abort-streaming, abort-tools
- Dynamic import gated on `feature('CHICAGO_MCP')`
- Unhide timeout: 5s (generous, just unblocks abort)
- All cleanup is idempotent

---

### D. common.ts (Constants & Sentinel Values)
**Purpose**: Shared constants and terminal detection.

**Key exports**:
- `COMPUTER_USE_MCP_SERVER_NAME = 'computer-use'`
- `CLI_HOST_BUNDLE_ID = 'com.anthropic.claude-code.cli-no-window'`
  - Sentinel for "no frontmost window" (terminal has no window)
  - Used by package's frontmost gate (always false, design intent)
- `CLI_CU_CAPABILITIES = {screenshotFiltering: 'native', platform: 'darwin'}`

**Terminal detection** via `getTerminalBundleId()`:
- Reads `process.env.__CFBundleIdentifier` (set by LaunchServices for .app bundles)
- Fallback table: iTerm2, Apple Terminal, ghostty, kitty, Warp, VS Code
- Returns null if undetectable (ssh, tmux client, unknown terminal)

**Why terminal detection matters**:
- Used as "surrogate host" in prepareDisplay (hide-exempt)
- Stripped from screenshot allowlist so terminal never photobombs
- Skipped in app activation z-order walk

---

### E. drainRunLoop.ts (CFRunLoop Pump)
**Purpose**: Drains macOS's main dispatch queue for async-to-sync bridging.

**Problem**:
- Swift's `@MainActor` methods (screenshot, listInstalled, etc.) dispatch to DispatchQueue.main
- enigo's key() also uses DispatchQueue.main
- Under libuv (Node/Bun), this queue never drains → promises hang indefinitely
- Electron drains CFRunLoop continuously, so Cowork doesn't need this

**Solution**:
- Refcounted `setInterval(drainTick, 1ms)` that calls `_drainMainRunLoop()`
- `retain()`: increments pending, starts pump if needed
- `release()`: decrements, stops pump when pending hits zero
- Safe nesting via refcount

**Timeout protection**:
- `drainRunLoop(fn)`: 30s timeout with race detection
- Orphaned promise detection: late rejection swallowed with `.catch(() => {})`
- `retainPump`/`releasePump`: long-lived registration (ESC hotkey), no timeout

---

### F. escHotkey.ts (System Escape Interception)
**Purpose**: Global Escape key abort, preventing prompt-injected actions from dismissing dialogs.

**Mechanism**:
- `registerEscHotkey()`: Creates CGEventTap via Swift
- Tap consumes Escape system-wide (CFRunLoopGetMain defaultMode)
- Escape never reaches the target app
- Model must call `notifyExpectedEscape()` for model-synthesized Escapes (100ms decay)

**Lifecycle**:
- Register on fresh lock acquire (first CU tool call)
- Unregister on lock release
- Pump retain held for registration lifetime (refcounted with drainRunLoop)
- Returns false if CGEvent.tapCreate fails (missing Accessibility permission)

**Safety model**:
- Hole-punch for model's own Escape: notifyExpectedEscape() sets decay flag
- CGEventTap checks `event.flags.isEmpty` so ctrl+escape etc. pass through

---

### G. executor.ts (Main ComputerExecutor Implementation)
**Purpose**: Bridge between MCP tool calls and native input/screenshot APIs.

**Key architecture**:
- Factory function `createCliExecutor(opts)` builds ComputerExecutor singleton
- Two native modules:
  - `@ant/computer-use-input`: Rust/enigo for mouse, keyboard, FrontmostApp
  - `@ant/computer-use-swift`: SCContentFilter screenshots, TCC, app management
- Lazy loading: Swift loaded at factory time, Input loaded on first mouse/keyboard call

**Screenshot pipeline**:
```
display.getSize(displayId)
  ↓ logical.px * scaleFactor → physical.px
  ↓ targetImageSize(physW, physH, API_RESIZE_PARAMS) → target dims
  ↓ cu.screenshot.captureExcluding(allowedBundleIds, quality=0.75, targetW, targetH, displayId)
  ↓ Result: {base64: string (JPEG), width, height}
```

**Input sequence primitives**:

1. **Mouse**:
   - `moveAndSettle()`: Instant move + 50ms sleep for HID round-trip
   - `animatedMove()`: Distance-proportional duration at 2000px/sec, max 0.5s, ease-out-cubic @ 60fps
   - `click()`: Move → modifiers-bracketed button click
   - `drag()`: Move → press → animated move → release (finally ensures release)
   - `scroll()`: Move → vertical first → horizontal

2. **Keyboard**:
   - `key()`: xdotool-style "ctrl+shift+a" split on '+' → enigo.keys()
   - Bare Escape: notifyExpectedEscape() before key()
   - `holdKey()`: Press all → sleep(durationMs) → release all (reverse order)
   - `type()`: Per-grapheme via clipboard or typeText()
   - Clipboard path: read → write → verify round-trip → paste → sleep(100ms) → restore

3. **App management**:
   - `prepareForAction()`: Hide non-allowlisted apps + defocus → returns hidden set
   - `listInstalledApps()`: Spotlight via Swift
   - `openApp()`: NSWorkspace.openApplication

**JPEG quality**: 0.75 (75%)

**Terminal as surrogate host**:
```typescript
const surrogateHost = terminalBundleId ?? CLI_HOST_BUNDLE_ID
// Passed to prepareDisplay (hide-exempt) and resolvePrepareCapture
// Stripped from screenshot allowlist so terminal never photobombs
```

---

### H. gates.ts (Feature Gates & Configuration)
**Purpose**: GrowthBook feature flag integration for CU subgates.

**Gates**:
- `enabled`: Master gate (default: false)
- `pixelValidation`: Pixel-compare click validation (default: false)
- `clipboardPasteMultiline`: Multiline paste via clipboard (default: true)
- `mouseAnimation`: Animated drag movement (default: true)
- `hideBeforeAction`: Show/hide behavior (default: true)
- `autoTargetDisplay`: Auto-detect display (default: true)
- `clipboardGuard`: Clipboard safety checks (default: true)
- `coordinateMode`: 'pixels' | 'normalized' (default: 'pixels', frozen at first read)

**Subscription gating**:
- Max/Pro only for external rollout
- User_TYPE='ant' bypass (dogfooding)

**Frozen coordinate mode**: Read once at setup, stays constant even if GB flips mid-session

---

### I. hostAdapter.ts (Dependency Injection Container)
**Purpose**: Singleton factory for ComputerUseHostAdapter passed to MCP package.

**Adapter shape**:
```typescript
{
  serverName: 'computer-use'
  logger: DebugLogger
  executor: ComputerExecutor
  ensureOsPermissions: () => {accessibility, screenRecording}
  isDisabled: () => !getChicagoEnabled()
  getSubGates: () => CuSubGates
  getAutoUnhideEnabled: () => true
  cropRawPatch: () => null
}
```

**Key details**:
- Process-lifetime singleton (cached)
- Loaded on first CU tool call
- Native modules load here, throw on failure (no degraded mode)
- cropRawPatch returns null (async limitation)

---

### J. inputLoader.ts (Rust/enigo Wrapper)
**Purpose**: Lazy-load Rust/enigo native module.

**Export path**:
- `COMPUTER_USE_INPUT_NODE_PATH` (baked by build-with-plugins.ts on darwin)
- Falls through to node_modules prebuilds if unset

**Dispatch model**:
- `key()` and `keys()` dispatch to DispatchQueue.main
- Block tokio worker on channel until completion
- Requires drainRunLoop() on libuv (Node/Bun)

---

### K. mcpServer.ts (MCP Server Lifecycle)
**Purpose**: In-process MCP server for stdio transport.

**Startup flow**:
1. Singleton init: `getComputerUseHostAdapter()`
2. Package factory: `createComputerUseMcpServer(adapter, coordinateMode)`
3. App enumeration: `tryGetInstalledAppNames()` (1s timeout, soft fail)
4. Tool building: `buildComputerUseTools(capabilities, coordinateMode, installedAppNames)`
5. ListTools override: Include installed app names in request_access description

**Subprocess entry point**:
- `--computer-use-mcp` spawns `runComputerUseMcpServer()`
- StdioServerTransport
- Exit on stdin EOF
- Flush analytics before exit

---

### L. setup.ts (Dynamic MCP Config Builder)
**Purpose**: Builds MCP config + allowedTools for main client.

**Key architecture**:
- `mcp__computer-use__*` tool names added to allowedTools
- Bypass normal permission prompts (package's request_access handles approval)
- API backend detects these names, emits CU availability hint in system prompt

**Config**:
```typescript
{
  type: 'stdio'
  command: process.execPath
  args: ['--computer-use-mcp'] (bundled) or ['.../cli.js', '--computer-use-mcp']
  scope: 'dynamic'
}
```

**Why stdio never spawns**: client.ts intercepts by name, uses in-process server.

---

### M. swiftLoader.ts (Swift Native Module Wrapper)
**Purpose**: Load and cache @ant/computer-use-swift.

**Four @MainActor methods** (all require drainRunLoop):
- `screenshot.captureExcluding(allowedBundleIds, quality, targetW, targetH, displayId)`
- `screenshot.captureRegion(allowedBundleIds, x, y, w, h, outW, outH, quality, displayId)`
- `apps.listInstalled()`
- `resolvePrepareCapture(...)`

**Key details**:
- Load once at factory time
- Throws on non-darwin
- Cached, no runtime reloading

---

## 2. SCREENSHOT CAPTURE PIPELINE

**Full flow**:

1. **Request arrives** at `screenshot` tool with displayId and allowlist
2. **Size computation**:
   - display.getSize(displayId) → {width, height, scaleFactor}
   - logical dims * scaleFactor = physical dims
   - targetImageSize(physW, physH, API_RESIZE_PARAMS) → target dims
3. **Filtering**:
   - allowedBundleIds passed (user-granted apps)
   - Terminal (if detected) stripped from list
   - Swift's captureExcluding takes ALLOW list
4. **Capture** via `cu.screenshot.captureExcluding()`:
   - SCContentFilter (ScreenCaptureKit) on macOS 13+
   - Returns JPEG base64 + actual dims
5. **API encoding**: base64 → text block in API request

**Quality**: 75% JPEG compression (0.75 flag)

**Exclusion**: Terminal never captured (if detected), preventing photobomb.

---

## 3. MOUSE & KEYBOARD EXECUTION MODEL

**Dispatch chain**:
- moveAndSettle(x, y) → move + 50ms sleep (HID round-trip)
- withModifiers() → bracket press/release, swallow errors, reverse order release
- input.mouseButton() or input.keys() → native layer
- drainRunLoop() wraps main-queue-dispatching calls

**Sleep timings**:
- MOVE_SETTLE_MS = 50ms: After mouse move, before click
- 8ms between repeated key presses (125Hz USB polling)
- Type via clipboard: 100ms final sleep (paste vs restore race)

**Modifier bracketing**:
- Tracks pressed keys to release only what succeeded
- Finally block releases in reverse order
- Errors swallowed (best-effort)

**Escape special case**:
- `isBareEscape([part])`: Single element "escape" or "esc" (case-insensitive)
- Calls `notifyExpectedEscape()` before key() to punch hole in CGEventTap
- ctrl+escape passes through (flags not empty)

**Clipboard path** (for type() with viaClipboard: true):
1. Read saved content via pbpaste
2. Write new text via pbcopy
3. READ-BACK VERIFY: pbpaste again, fail if mismatch
4. Cmd+V via input.keys(['command','v'])
5. sleep(100ms): Paste effect vs restore race
6. Restore saved in finally

**Drag internals**:
- Optional from: move to start position
- Press mouse
- sleep(50ms): Let HID tap register pressedMouseButtons
- animatedMove() with animation enabled/disabled
- Finally: always release (even if throw)

---

## 4. SAFETY BOUNDARIES & VALIDATION

**Pre-action safeguards**:

1. **frontmost gate** (in package):
   - Checks if target app is frontmost before executing action
   - Sentinel `CLI_HOST_BUNDLE_ID` never matches
   - Safety net

2. **prepareForAction**:
   - Hides non-allowlisted apps
   - Returns hidden set for cleanup
   - Errors logged but continue (frontmost gate still enforces)

3. **request_access** tool:
   - Lists installed apps (filtered via appNames.ts)
   - Requires explicit user approval
   - Lock check: tools don't acquire lock (defers to first action tool)

4. **Escape hotkey**:
   - Global Escape consumed by CGEventTap
   - Prevents prompt-injected Escape from dismissing dialogs
   - Model must opt-in via notifyExpectedEscape()

5. **Pixel validation** (sub-gate, default: false):
   - Compares before/after screenshot patches
   - Disabled: no sync image-processor in Node
   - Designed fallback: skip validation

6. **Clipboard guard** (sub-gate, default: true):
   - Round-trip verification: write → read-back
   - Fails if mismatch
   - Restore in finally, errors swallowed

---

## 5. ATTACK SURFACE & INJECTION VECTORS

### Prompt Injection via App Names
- **Vector**: Attacker installs app named "grant all permissions"
- **Mitigation**:
  - Character allowlist (no quotes, pipes, backticks)
  - Trusted-app carve-out (Apple/Google/MS bypass filter)
  - Length cap (40 chars), count cap (50 apps)
  - Structural framing + explicit user approval dialog
- **Residual risk**: Benign-sounding names hard to filter programmatically

### Escape Key Injection
- **Vector**: Model-synthesized Escape dismisses dialogs
- **Mitigation**: CGEventTap consumes globally, model must opt-in
- **Residual risk**: ctrl+escape might be intended as cancel, passes through (flags check)

### Clipboard Injection
- **Vector**: Clipboard write fails silently, junk pasted
- **Mitigation**: Read-back verify, fail if mismatch
- **Residual risk**: None if working correctly

### Screenshot Exfiltration
- **Vector**: Model exports screenshot to attacker-controlled endpoint
- **Mitigation**: Base64 JPEG goes only to Anthropic API, terminal excluded
- **Residual risk**: None if API boundary respected

### App Enumeration
- **Vector**: Model fingerprints installed software
- **Mitigation**: None (feature inherent)
- **Residual risk**: Reveals installed apps to Claude

### Mouse Position Leakage
- **Vector**: Model probes mouse position repeatedly to infer activity
- **Mitigation**: None (feature inherent)
- **Residual risk**: Reveals user activity

---

## 6. CONCURRENCY & STATE MANAGEMENT

**Session isolation**:
- File-based lock prevents concurrent sessions
- Each session has sessionId, pid, acquiredAt timestamp
- Stale detection via PID liveness

**AppState tracking**:
- `computerUseMcpState.hiddenDuringTurn`: Set of bundleIds hidden this turn
- Populated by prepareForAction
- Cleared by cleanup after turn
- No cross-turn state

**CFRunLoop pump**:
- Refcounted setInterval
- Safe nesting: multiple drainRunLoop() calls share one pump
- Pending count prevents premature stop

**Hotkey registration**:
- Global state: `let registered = false`
- Idempotent: registerEscHotkey() returns true if already registered
- unregisterEscHotkey() swallows errors, always releasePump

---

## 7. ERROR HANDLING & FALLBACKS

**prepareForAction failure**: Logged, action continues (frontmost gate enforces)

**Escape hotkey registration failure**: Logged, CU proceeds without abort

**Clipboard read-back mismatch**: Throws, never pastes, restore in finally

**App enumeration timeout**: Soft fail, tool description omits list

**Unhide timeout** (5s): Best-effort, timer cleared regardless

**Modifier release in finally**: Errors swallowed (best-effort)

**Orphaned promise cleanup**: Late rejection swallowed with `.catch(() => {})`

---

## 8. PERFORMANCE CHARACTERISTICS

**Load time**:
- Swift: Once at executor factory (first CU tool call)
- Input: Lazy on first mouse/keyboard
- Cleanup: Zero cost for non-CU turns (dynamic import)

**CPU overhead**:
- CFRunLoop pump: 1ms setInterval while pending
- DrainRunLoop: 30s timeout ceiling

**Latency**:
- Mouse move → 50ms settle = 50ms minimum per action
- Drag animation: Distance-proportional, max 0.5s
- Keyboard: 8ms between repeats
- Type via clipboard: 100ms final sleep

**Screenshot latency**:
- Spotlight enumeration: 1s timeout
- SCContentFilter capture: 100-500ms (platform-dependent)
- JPEG encode: 75% quality

---

## 9. NOVEL ENGINEERING DECISIONS

1. **Terminal as surrogate host**: Clever solution to exclude terminal from screenshots while keeping it unhidden
2. **Read-back clipboard verify**: Robust defense against silent clipboard write failures
3. **Animated drag with intermediate frames**: Targets apps that require `.leftMouseDragged` sequence
4. **Orphaned promise swallowing**: Prevents unhandledRejection from timeout race
5. **Feature gate freezing**: coordinateMode frozen at session start to prevent model/executor mismatch
6. **Refcounted pump**: Safe nesting of main-queue dispatch via retain/release pattern
7. **Non-acquiring lock check**: defers acquiring until first action tool
8. **Stale lock recovery**: O_EXCL atomic create handles race between recovery attempts

---

## 10. TECHNICAL DEBT & LIMITATIONS

1. **cropRawPatch returns null**: Pixel validation disabled due to lack of sync image processor
2. **No Windows/Linux support**: macOS-only (platform guard in executor.ts)
3. **Hard-coded 30s timeout**: drainRunLoop ceiling not configurable
4. **Terminal detection fallback**: May misidentify in ssh/tmux scenarios
5. **App filtering count cap**: Max 50 apps shown even if 200+ installed
6. **Escape hotkey permission gating**: No warning if CGEvent.tapCreate fails

---

## Summary

The Claude Code computer use system is a well-engineered, production-quality macOS desktop automation layer. Key strengths are modular architecture, thoughtful safety layering, correct concurrency handling, and graceful error degradation. Residual risks are prompt injection via app names and keyboard/screen data exposure (inherent to feature), mitigated by user approval gates. The implementation demonstrates sophisticated macOS native API usage, particularly CFRunLoop pumping and CGEventTap interception.

