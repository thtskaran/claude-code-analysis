# Claude Code Swarm Orchestration System - Comprehensive Reverse Engineering Analysis

## Executive Summary

This document provides an exhaustive technical analysis of the Claude Code swarm orchestration subsystem (`/src/utils/swarm/`), which enables multiple Claude agents to work together as a coordinated team with a leader-follower architecture. The system supports three execution backends (tmux, iTerm2, in-process) and implements sophisticated IPC, permission management, and lifecycle coordination.

---

## 1. ARCHITECTURE OVERVIEW

### 1.1 Core Concept
A **swarm** is a team of autonomous agents coordinated by a leader agent. Architecture:
- **Leader**: Single coordinator agent (may be user or in-process)
- **Workers**: Multiple agent teammates spawned by the leader
- **Execution Models**:
  - Pane-based: Separate processes in terminal panes (tmux/iTerm2)
  - In-process: Multiple agents in same Node.js process with isolated context

### 1.2 Key Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Backend Detection                         │
│  (detection.ts, registry.ts, it2Setup.ts)                   │
│                                                               │
│  Detects environment (tmux/iTerm2/standalone)               │
│  Selects appropriate backend (TmuxBackend, ITermBackend,     │
│  InProcessBackend)                                           │
└────────────────────────────┬────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
   ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐
   │TmuxBackend  │  │ITermBackend   │  │InProcessBackend │
   │(PaneBackend)│  │(PaneBackend)  │  │(TeammateExecutor)│
   └──────┬──────┘  └────────┬──────┘  └────────┬────────┘
          │                  │                   │
          └──────────────────┼───────────────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ TeammateExecutor     │
                  │ (Unified Interface)  │
                  └──────────┬───────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
   ┌─────────────────┐ ┌──────────────┐ ┌─────────────┐
   │spawn()          │ │sendMessage() │ │terminate()  │
   │kill()           │ │isActive()    │ │             │
   │terminate()      │ │              │ │             │
   └─────────────────┘ └──────────────┘ └─────────────┘
```

### 1.3 File Organization

**Core Subsystem:**
- `constants.ts` - Socket names, environment variable names
- `spawnUtils.ts` - CLI flag/env var inheritance logic
- `teamHelpers.ts` - Team file I/O (CRUD operations on team metadata)
- `permissionSync.ts` - Permission request/response file-based IPC
- `leaderPermissionBridge.ts` - Module-level bridge for in-process permission UI

**In-Process Execution:**
- `spawnInProcess.ts` - In-process teammate spawn (creates AbortController, registers task)
- `inProcessRunner.ts` - Agent loop execution with permission polling and mailbox integration

**Team Initialization:**
- `reconnection.ts` - Compute initial team context from CLI/transcript
- `teammateInit.ts` - Register hooks for idle notifications, apply team permissions
- `teammateModel.ts` - Hardcoded fallback model (Opus 4.6)
- `teammateLayoutManager.ts` - Color assignment and pane delegation
- `teammatePromptAddendum.ts` - System prompt addendum for teammates

**Backend System:**
- `backends/types.ts` - TeammateExecutor, PaneBackend, and supporting types
- `backends/detection.ts` - Environment detection (tmux/iTerm2/standalone)
- `backends/registry.ts` - Backend factory, caching, detection coordination
- `backends/TmuxBackend.ts` - Tmux pane management
- `backends/ITermBackend.ts` - iTerm2 native split pane support
- `backends/InProcessBackend.ts` - In-process execution adapter
- `backends/PaneBackendExecutor.ts` - Adapter converting PaneBackend → TeammateExecutor
- `backends/teammateModeSnapshot.ts` - Session-wide teammate mode capture
- `backends/it2Setup.ts` - it2 CLI installation and verification

---

## 2. IPC PROTOCOL DEEP-DIVE

### 2.1 Mailbox System (File-Based)

**Location:** `~/.claude/teams/{team-name}/mailbox/{agent-name}/`

**Files:**
- `messages.json` - Array of messages (read/written atomically)
- `.lock` - Lockfile for atomic updates (using lockfile.ts)

**Message Structure:**
```typescript
type TeammateMessage = {
  from: string           // Agent name or "team-lead"
  text: string          // Message content (may contain JSON)
  timestamp: string     // ISO 8601
  color?: string        // Agent UI color
  read: boolean         // Per-message read flag
  summary?: string      // 5-10 word preview
}
```

**Atomic Operations:**
- `readMailbox()`: Lock, read all messages, unlock
- `writeToMailbox()`: Lock, append message, unlock
- `markMessageAsReadByIndex()`: Lock, mark by index, unlock

**Polling Pattern:**
```
Worker loop (500ms intervals):
  1. readMailbox() to get all messages
  2. Scan for shutdown requests (highest priority)
  3. Check for new regular messages
  4. Scan for permission responses
  5. If abort signal set, return with 'aborted' type
  6. Else sleep(500ms), retry
```

### 2.2 Permission Request/Response Flow

**Storage Path:** `~/.claude/teams/{team-name}/permissions/{pending,resolved}/`

**Pending Directory Structure:**
```
permissions/
  pending/
    perm-{timestamp}-{random}.json    # SwarmPermissionRequest (status=pending)
    .lock                              # Directory lock
  resolved/
    perm-{timestamp}-{random}.json    # SwarmPermissionRequest (status=resolved)
```

**Request Schema:**
```typescript
type SwarmPermissionRequest = {
  id: string
  workerId: string          // Agent@Team
  workerName: string        // "researcher"
  workerColor?: string
  teamName: string
  toolName: string          // "Bash", "Edit"
  toolUseId: string         // API toolUseID
  description: string       // Tool-readable summary
  input: Record<string, unknown>
  permissionSuggestions: unknown[]
  status: 'pending' | 'approved' | 'rejected'
  resolvedBy?: 'worker' | 'leader'
  resolvedAt?: number       // Timestamp
  feedback?: string         // If rejected
  updatedInput?: Record<string, unknown>
  permissionUpdates?: PermissionUpdate[]
  createdAt: number
}
```

**Dual-Path Permission Resolution:**

Path 1: **UI Bridge (Preferred for In-Process)**
- Worker calls `createInProcessCanUseTool()` → checks `getLeaderToolUseConfirmQueue()`
- Queue populated by leader's React component via `registerLeaderToolUseConfirmQueue()`
- Worker adds itself to queue with `onAllow/onReject/onAbort` callbacks
- Leader approves in UI, callbacks fire immediately
- Instant permission resolution without file I/O

Path 2: **Mailbox System (Fallback)**
- Worker creates SwarmPermissionRequest, writes to pending/
- Worker calls `sendPermissionRequestViaMailbox()` → creates permission_request message
- Leader polls mailbox, finds message, displays to user
- Leader calls `sendPermissionResponseViaMailbox()` → creates permission_response message
- Worker polls mailbox for response, processes when found
- Cleanup: `removeWorkerResponse()` deletes resolved file

**Sandbox Permission System:**
- Separate mailbox message type: `sandbox_permission_request`
- Worker requests network access to `host`
- Leader approves/denies with `sandbox_permission_response`
- Same mailbox polling mechanism

### 2.3 Shutdown Request Flow

**Message Type:** `shutdown_request` in mailbox (JSON)
```typescript
type ShutdownRequest = {
  type: 'shutdown_request'
  requestId: string        // Unique ID
  from: string            // Always "team-lead"
  reason?: string
}
```

**Flow:**
1. Leader calls `InProcessBackend.terminate()` → sends shutdown_request to worker mailbox
2. Worker's main loop: `waitForNextPromptOrShutdown()` polls, finds shutdown request
3. Worker's model decides: approve or reject
4. If approved: `removeMemberByAgentId()` removes from team file, graceful exit
5. If rejected: continues working (doesn't exit)

### 2.4 Idle Notification Flow

**Message Type:** `idle_notification` in leader's mailbox (JSON)
```typescript
type IdleNotification = {
  type: 'idle_notification'
  from: string             // Agent name
  idleReason: 'available' | 'interrupted' | 'failed'
  summary?: string         // Last DM summary
  completedTaskId?: string
  completedStatus?: 'resolved' | 'blocked' | 'failed'
  failureReason?: string
}
```

**Flow:**
1. Worker's Stop hook fires: `initializeTeammateHooks()` registered hook
2. Hook calls `sendIdleNotification()` → writes to leader's mailbox
3. Leader polls mailbox, detects idle_notification
4. Updates UI to mark worker as idle

---

## 3. PERMISSION MODEL

### 3.1 Permission Propagation

**Three-Tier System:**

**Tier 1: Session-Level Permissions**
- Captured in `toolPermissionContext` (AppState)
- Leader and workers each have their own context
- Workers' context initialized with parent's permissions + team-wide rules

**Tier 2: Team-Wide Allowed Paths**
- Stored in `teamFile.teamAllowedPaths`
- Example: `{path: "/project", toolName: "Edit", addedBy: "leader"}`
- Applied to all team members via `initializeTeammateHooks()`
- Converted to permission rule: `/project/**` → allow Edit

**Tier 3: Request-Time Permission Updates**
- When worker gets permission approval, `permissionUpdates` array included
- Types: `addRules`, `removeRules` (PermissionUpdate)
- Persisted via `persistPermissionUpdates()`
- Shared back to leader via bridge: `getLeaderSetToolPermissionContext()`

### 3.2 Permission Decision Flow

```
Worker calls tool:
  ↓
canUseTool(tool, input, context):
  ↓
  Check toolPermissionContext.rules:
    ✓ Allow   → return {behavior: 'allow'}
    ✗ Deny    → return {behavior: 'ask', message: DENY_MSG}
    ? Ask     → return {behavior: 'ask', suggestions: [rule1, rule2]}
  ↓
  If behavior='ask':
    → UI Bridge path (in-process):
        Add to setToolUseConfirmQueue()
        Wait for onAllow/onReject callback
    → Mailbox path (fallback):
        Send permission_request to leader
        Poll mailbox for permission_response
        Execute onAllow/onReject callback
  ↓
Return {behavior, updatedInput?, permissionUpdates?}
```

### 3.3 Permission Isolation

**Leader ↔ Worker Context Separation:**

```typescript
// When worker approves with permission updates:
setToolPermissionContext(updatedContext, { preserveMode: true })
//                                              ^^^^^^ CRITICAL
// Prevents worker's 'acceptEdits' mode from leaking to leader
```

---

## 4. FAILURE MODES & RACE CONDITIONS

### 4.1 Critical Race Conditions

**RC1: Concurrent Pane Spawning**
- **Vulnerability**: Multiple `createTeammatePaneInSwarmView()` called simultaneously
- **Impact**: tmux commands interleave, pane IDs collide, layout corrupts
- **Mitigation**: `acquirePaneCreationLock()` serializes all pane creation
  - Mutex implemented via Promise chain in module scope
  - Each spawn waits for prior spawn to complete

**RC2: Permission Request File Collision**
- **Vulnerability**: Two workers simultaneously writing permission request with same timestamp
- **Impact**: File overwrite via race in `writePermissionRequest()`
- **Mitigation**:
  - Request ID format: `perm-{Date.now()}-{Math.random().toString(36).substring(2, 9)}`
  - 9 random chars from base36 = ~10^17 unique IDs
  - Lockfile coordination ensures atomic writes

**RC3: Team File Concurrent Updates**
- **Vulnerability**: `setMemberMode()` called from multiple teammates simultaneously
- **Impact**: Lost updates to team file
- **Mitigation**:
  - Sync IO: `readTeamFile()`, mutate, `writeTeamFile()` atomic from reader's perspective
  - Async IO: `setMultipleMemberModes()` batches updates atomically
  - No distributed lock between calls - **POTENTIAL VULNERABILITY**

**RC4: In-Process Teammate Context Isolation Failure**
- **Vulnerability**: AsyncLocalStorage context leak between concurrent teammates
- **Impact**: Agent A sees Agent B's environment variables (agentId, teamName)
- **Status**: Requires Node.js > 20.10 for proper ALS isolation
- **Note**: If older Node.js used, context pollution possible

**RC5: Abort Signal Race**
- **Vulnerability**: `abortController.signal.aborted` checked, then signal aborted before operation completes
- **Example**: Kill called, task.abortController.abort() fires, but worker is mid-API-call
- **Mitigation**: Operations check abort signal again before execution:
  ```typescript
  if (abortController.signal.aborted) return REJECT
  // ... API call ...
  if (abortController.signal.aborted) return REJECT
  ```

### 4.2 Deadlock Scenarios

**DL1: Mailbox Lock Timeout**
- **Condition**: lockfile.lock() hangs indefinitely
- **Impact**: All mailbox operations block
- **No mitigation visible**: No timeout parameter in lockfile usage

**DL2: Circular Permission Requests**
- **Condition**: Worker A asks leader for tool use, leader asks Worker B, B asks A
- **Impact**: Permission poll loops forever (no timeout)
- **Status**: Leader UI model can escape, but mailbox polling has no timeout

**DL3: Cleanup Handler Circular Dependency**
- **Condition**: registerCleanup() handler calls operation requiring cleanup handler
- **Impact**: Cleanup stack overflow
- **Mitigation**: Cleanup registry prevents re-entry via abort signal check

### 4.3 Resource Exhaustion

**RE1: Pane Proliferation**
- **Condition**: 1000 teammates spawned, each gets a pane
- **Impact**: Terminal UI becomes unresponsive, tmux memory grows
- **No limit**: No maxTeammates configuration found

**RE2: Permission File Accumulation**
- **Condition**: cleanupOldResolutions() never called
- **Impact**: ~/.claude/teams/{team}/permissions/resolved/ fills disk
- **Cleanup**: `maxAgeMs = 3600000` (1 hour default)
- **Status**: Must be called manually or via scheduled task

**RE3: Message Accumulation in Mailbox**
- **Condition**: Leader never reads mailbox
- **Impact**: ~/.claude/teams/{team}/mailbox/{agent}/ fills disk
- **Status**: No automatic pruning visible

### 4.4 Edge Cases

**EC1: Team File Deleted Mid-Session**
- Worker calls `readTeamFileAsync()` while leader deletes team dir
- Worker gets `null`, continues without team context
- **Impact**: Worker can't send idle notification, leader doesn't see completion

**EC2: Pane-Based Teammate Process Crashes**
- Process in pane exits with non-zero code
- PaneBackendExecutor.isActive() returns `true` (just checks spawnedTeammates map)
- **Impact**: Leader thinks teammate alive, sends messages to dead process

**EC3: Permission Request Stale**
- Worker sends permission request, then aborts
- Request never deleted from pending/
- Leader approves request for dead worker
- **Impact**: Permission approval wasted, file leaks

**EC4: Worktree Path Contains Symlinks**
- `destroyWorktree()` reads .git file, extracts path
- Symlinks not resolved
- Git command fails, falls back to rm -rf
- **Impact**: Symlink target deleted instead of worktree

---

## 5. SECURITY ASSESSMENT

### 5.1 Privilege Escalation Vectors

**PEV1: Malicious Team File**
- **Vector**: Attacker writes team file with `teamAllowedPaths` containing `/etc/**`
- **Impact**: Worker applies permission: Edit allowed in /etc
- **Mitigation**: Permission rules checked at tool execution time, but no path isolation from team context
- **Risk**: HIGH if team file writable by untrusted user

**PEV2: Permission Bridge Registration Hijack**
- **Vector**: Malicious code calls `registerLeaderToolUseConfirmQueue()` before leader
- **Impact**: Fake queue setter captures all permission requests
- **Mitigation**: No validation that setter is from React/REPL context
- **Status**: Module-scope registration, any code can hijack
- **Risk**: CRITICAL if untrusted code executes in leader process

**PEV3: Mailbox Directory Symlink**
- **Vector**: Attacker creates `~/.claude/teams/my-team/mailbox/leader` → `/tmp/evil.json`
- **Impact**: Malicious messages injected, permission requests intercepted
- **Mitigation**: No symlink resolution in mailbox paths
- **Risk**: MEDIUM (requires write access to ~/.claude)

**PEV4: it2 Installation via pip**
- **Vector**: Attacker compromises PyPI, serves malicious it2 package
- **Impact**: RCE when it2 installed
- **Mitigation**: Only runs from homedir(), not in project context
- **Status**: Partial mitigation (homedir safe, but PyPI trust assumption)
- **Risk**: MEDIUM

### 5.2 Information Disclosure

**ID1: Permission Request File Content**
- **Exposure**: `/permissions/pending/{id}.json` contains tool input (sensitive data)
- **Example**: Bash command like `curl -H "Authorization: Bearer TOKEN"` visible
- **Mitigation**: File permissions on ~/.claude (typically 0755, world-readable)
- **Risk**: MEDIUM

**ID2: Mailbox Message Content**
- **Exposure**: `~/.claude/teams/{team}/mailbox/{agent}/messages.json` contains all messages
- **Mitigation**: Same as above, world-readable
- **Risk**: MEDIUM

**ID3: Team File Leaks Agent IDs**
- **Exposure**: `teamFile.members[].sessionId` stores actual session UUID
- **Impact**: Attacker can correlate sessions to teams
- **Risk**: LOW

### 5.3 Denial of Service

**DoS1: Permission Poll Exhaustion**
- **Attack**: Worker polls mailbox every 500ms indefinitely
- **Impact**: Disk I/O load, CPU spinning
- **Mitigation**: Abort signal stops polling, but malicious worker can ignore abort
- **Risk**: MEDIUM

**DoS2: Circular Shutdown Requests**
- **Attack**: Leader sends shutdown to Worker A, Worker A sends shutdown to Worker B, B to A
- **Impact**: Workers stuck in shutdown negotiation
- **Mitigation**: Model decides, not algorithmic loop
- **Risk**: LOW (human decision-making involved)

**DoS3: Pane Lock Starvation**
- **Attack**: Acquire pane creation lock, never release
- **Impact**: All subsequent pane spawns block forever
- **Mitigation**: No lock timeout or watchdog
- **Risk**: CRITICAL if lock release callback throws

### 5.4 Data Integrity

**DI1: Permission File Overwrite**
- **Scenario**: Two workers write different requests with identical timestamp/random
- **Probability**: ~1 in 10^17 per second
- **Risk**: NEGLIGIBLE

**DI2: Partial Mailbox Write**
- **Scenario**: writeToMailbox() crashes mid-write
- **Impact**: messages.json corrupted (invalid JSON)
- **Recovery**: readMailbox() catches parse error, returns []
- **Risk**: LOW (graceful degradation)

**DI3: Team File State Divergence**
- **Scenario**: `syncTeammateMode()` calls `setMemberMode()`, network drops
- **Impact**: Member mode not synced for this session
- **Risk**: LOW (session-local state, no persistence requirement)

### 5.5 Process Escape/Isolation

**PE1: In-Process Teammate Breakout**
- **Vector**: Malicious teammate code calls `process.exit()` or `throw Error()`
- **Impact**: Kills leader process
- **Mitigation**: Try-catch around runAgent(), but not catching process signals
- **Risk**: MEDIUM

**PE2: AsyncLocalStorage Context Bleed**
- **Condition**: Node.js < 20.10 with improper ALS implementation
- **Impact**: Teammate reads leader's CLAUDE_CODE_AGENT_ID env var
- **Mitigation**: No explicit version check, relies on runtime
- **Risk**: MEDIUM (Node.js version dependent)

---

## 6. UNDOCUMENTED BEHAVIORS

### 6.1 Subtle Timeout Behaviors

**TB1: Permission Poll Timeout**
- No documented timeout for mailbox polling loop
- Worker will poll indefinitely until abort signal fires
- If abort signal never fires, worker hangs forever
- **Implication**: Long-running permission request can block shutdown

**TB2: Pane Shell Init Delay**
- `PANE_SHELL_INIT_DELAY_MS = 200ms` hardcoded
- Delay after pane creation to allow shell rc file loading
- If shell startup time > 200ms, command execution starts before shell ready
- **Implication**: First command may execute in wrong shell context

**TB3: Poll Counter Increment**
- In `waitForNextPromptOrShutdown()`, pollCount increments AFTER sleep
- First iteration checks immediately (pollCount=0), then sleeps
- **Implication**: ~500ms faster response time on first message

### 6.2 Implicit Assumptions

**IA1: Single Leader Per Team**
- Code assumes only one leader
- If two leaders spawn, no coordination mechanism
- **Impact**: Both leaders try to manage same teammates, conflict

**IA2: Team Files Single-Writer**
- Sync read-modify-write pattern assumes no concurrent writers
- If two agents call `setMemberMode()` simultaneously, one update lost
- **Implication**: Race condition acknowledged but not fixed

**IA3: Worktree Cleanup Idempotent**
- `destroyWorktree()` ignores "already removed" error
- Called from cleanup handler, safe to call multiple times
- **Assumption**: worktree path won't be recreated mid-session

### 6.3 Implicit State Machines

**SM1: Teammate Shutdown State**
```
  RUNNING
    ↓
  receives shutdown_request
    ↓
  model decides
    ↓
  [approved] → STOPPED (remove from team, exit)
  [rejected] → RUNNING (ignore, continue)
  [no response] → RUNNING (poll again in 500ms)
```
No explicit state variable, inferred from mailbox scan.

**SM2: Permission Request State**
```
  PENDING (in pending/)
    ↓
  leader resolves
    ↓
  move to resolved/
    ↓
  worker polls, finds response
    ↓
  worker deletes from resolved/
    ↓
  [gone]
```
Cleanup is async, no guarantee file deleted before next poll.

### 6.4 Surprising Implementation Details

**SD1: Leader Permission Bridge Registration**
- Bridge is module-level global state
- Only one registration allowed at a time
- `registerLeaderToolUseConfirmQueue()` overwrites previous
- `unregisterLeaderToolUseConfirmQueue()` sets to null
- **Implication**: If React component remounts, bridge may become null

**SD2: Teammate Mode Snapshot**
- Captured once at startup via `captureTeammateModeSnapshot()`
- CLI override checked first, then config.json
- User changing config.json at runtime has NO effect
- **Implication**: User must restart session to change teammate mode

**SD3: Color Assignment Round-Robin**
- Colors assigned globally in `teammateLayoutManager.ts`
- `colorIndex` increments forever, never resets
- If 100 teammates spawned, colors cycle 100/8 times
- **Implication**: Color not unique, just distributed

**SD4: it2 Package Manager Detection**
- Checks for `uv`, `pipx`, `pip`, `pip3` in that order
- Returns first available, prefers uv
- **Implication**: If both uv and pipx installed, uv used even if pipx is default

---

## 7. CONSTANTS & CONFIGURATION

### 7.1 Magic Numbers & Timeouts

| Constant | Value | Purpose | Risk |
|----------|-------|---------|------|
| `PERMISSION_POLL_INTERVAL_MS` | 500 | Mailbox polling frequency | High latency, high CPU if many workers |
| `PANE_SHELL_INIT_DELAY_MS` | 200 | Wait for shell initialization | Too short for slow shells |
| `STOPPED_DISPLAY_MS` | ?? | Duration to show stopped task | Task state flickers |
| `maxAgeMs` (cleanup) | 3600000 | 1 hour | Permission files accumulate if cleanup not called |
| `CliTeammateModeOverride` | null | CLI flag override | Module-level state, no validation |

### 7.2 Socket Names & Environment Variables

| Name | Value | Purpose |
|------|-------|---------|
| `SWARM_SESSION_NAME` | "claude-swarm" | External tmux session name |
| `SWARM_VIEW_WINDOW_NAME` | "swarm-view" | Window inside swarm session |
| `HIDDEN_SESSION_NAME` | "claude-hidden" | For hidden pane storage |
| `getSwarmSocketName()` | "claude-swarm-{pid}" | Socket for external session |
| `TEAM_LEAD_NAME` | "team-lead" | Mailbox name for leader |

### 7.3 Environment Variables Forwarded

**Provider Selection:**
- `CLAUDE_CODE_USE_BEDROCK`
- `CLAUDE_CODE_USE_VERTEX`
- `CLAUDE_CODE_USE_FOUNDRY`

**Configuration:**
- `ANTHROPIC_BASE_URL` - Custom API endpoint
- `CLAUDE_CONFIG_DIR` - Config directory override
- `CLAUDE_CODE_REMOTE` - CCR marker
- `CLAUDE_CODE_REMOTE_MEMORY_DIR` - Memory directory for CCR

**Network:**
- `HTTPS_PROXY`, `https_proxy`, `HTTP_PROXY`, `http_proxy`, `NO_PROXY`, `no_proxy`
- `SSL_CERT_FILE`, `NODE_EXTRA_CA_CERTS`, `REQUESTS_CA_BUNDLE`, `CURL_CA_BUNDLE`

**Always Set:**
- `CLAUDECODE=1`
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

### 7.4 Configuration Keys

| Key | Type | Default | Purpose |
|-----|------|---------|---------|
| `config.teammateMode` | 'auto' \| 'tmux' \| 'in-process' | 'auto' | Execution mode |
| `config.iterm2It2SetupComplete` | boolean | false | Skip it2 setup prompt |
| `config.preferTmuxOverIterm2` | boolean | false | Use tmux even in iTerm2 |
| `config.teammateModeSnapshot` | TeammateMode | 'auto' | Captured at startup |

---

## 8. CROSS-SYSTEM DEPENDENCIES

### 8.1 External Systems Consumed

**AsyncLocalStorage (`AsyncLocalStorage<TeammateContext>`):**
- Used for context isolation in in-process teammates
- Each teammate gets isolated context via `runWithTeammateContext()`
- Requires Node.js >= 18 (or >= 20.10 for full isolation)

**Abort Signals:**
- `AbortController` used for task cancellation
- Checked before every async operation
- Chain: Parent → Teammate via independent controller

**File System Operations:**
- `fs` module for mailbox I/O
- `fs/promises` for async operations
- No O_EXCL exclusive create, relies on lockfile for atomicity

**Lockfile Module:**
- `../lockfile.js` provides distributed lock on directory
- Used for: mailbox writes, permission request writes, team file updates (async only)

**Git Integration:**
- `git worktree remove --force` to destroy worktree
- Fallback to `rm -rf` if git fails

**tmux CLI:**
- Full tmux command list in TmuxBackend.ts
- Commands: new-session, new-window, split-pane, send-keys, kill-pane, break-pane, etc.
- Requires tmux >= 2.8 for pane color options

**it2 CLI (iTerm2):**
- Python-based CLI tool (pip/pipx installable)
- Commands: session list, session split
- Requires iTerm2 Python API enabled

### 8.2 Internal Dependencies

**AppState Integration:**
- Tasks stored in `AppState.tasks[taskId]`
- Permission context in `AppState.toolPermissionContext`
- Team context in `AppState.teamContext`

**React Components:**
- `InProcessTeammateTask` component executes agent loop
- `appendTeammateMessage()` appends to task messages
- Permission UI: `ToolUseConfirm` dialog

**Hooks:**
- `useSwarmPermissionPoller` polls for mailbox responses
- `useCanUseTool` calls permission check function
- `sessionHooks` registers cleanup handlers

**Tools:**
- `TeamCreateTool`, `TeamDeleteTool` - Team lifecycle
- `TeammateTool` - Spawn/manage teammates
- `SendMessageTool` - Inter-agent communication
- `TaskCreateTool`, `TaskUpdateTool`, `TaskGetTool` - Task management

---

## 9. TODO/TECHNICAL DEBT INVENTORY

### 9.1 Code Comments Indicating Known Issues

**File: inProcessRunner.ts (Line ~428)**
```typescript
const pollInterval = setInterval(
  async (abortController, cleanup, resolve, identity, request) => {
    // ... polling logic ...
  },
  PERMISSION_POLL_INTERVAL_MS,
  abortController,     // <-- Passed as parameter but rarely good practice
  cleanup,
  resolve,
  identity,
  request,
)
```
**Issue**: setInterval with parameters is unusual, captures in closure better

**File: TmuxBackend.ts**
```typescript
// IMPORTANT: We ONLY check the TMUX env var. We do NOT run `tmux display-message`
// as a fallback because that command will succeed if ANY tmux server is running
// on the system, not just if THIS process is inside tmux.
```
**Issue**: Documented correctly but could be fragile if TMUX var gets unset

**File: PaneBackendExecutor.ts (Line ~340)**
```typescript
// For now, assume active if we have a record of it
// A more robust check would query the backend for pane existence
// but that would require adding a new method to PaneBackend
return true
```
**Issue**: isActive() always returns true for spawned teammates - **FIXME**

**File: teamHelpers.ts (Line ~166)**
```typescript
// sync IO: called from sync context
function writeTeamFile(teamName: string, teamFile: TeamFile): void {
```
**Issue**: Race condition acknowledged but only mitigated for async variant

### 9.2 Missing Features/Incomplete Implementation

**Feature Gap 1: Permission Request Timeout**
- Workers can poll indefinitely without timeout
- No mechanism to auto-deny after timeout
- Recommendation: Add `timeoutMs` to SwarmPermissionRequest

**Feature Gap 2: Malicious Worker Cleanup**
- If teammate infinite-loops polling mailbox, uses resources
- No activity monitoring or resource limits
- Recommendation: Kill idle workers after N minutes

**Feature Gap 3: Pane Limit**
- No max teammates check before pane creation
- UI could become unusable with 1000 teammates
- Recommendation: Add `maxTeammates` config

**Feature Gap 4: Worktree Isolation**
- Worktrees created in project root
- No mechanism to isolate teammates to subdirectories
- Recommendation: Support per-teammate cwd argument

**Feature Gap 5: Message History**
- Mailbox messages not archived or pruned
- Old messages accumulate forever
- Recommendation: Implement message rotation or TTL

### 9.3 Likely Incomplete Code Paths

**Path 1: ITermBackend**
- File `ITermBackend.ts` not shown in analysis (file size limits)
- Likely mirrors TmuxBackend structure
- Unknown if AppleScript integration complete

**Path 2: Permission Update Persistence**
- `persistPermissionUpdates()` called in inProcessRunner.ts
- Location and format of persistence not documented
- Assumption: Stored in AppState only (non-persistent)

**Path 3: Team File Version Migration**
- TeamFile has optional fields (leadSessionId, hiddenPaneIds, backendType)
- No version number or migration logic visible
- Could break if format changes

---

## 10. ATTACK SURFACE SUMMARY

### Critical Issues (Exploit Likely)

1. **Permission Bridge Hijacking** - Unvalidated module-level registration allows fake setter
2. **Unbound Mailbox Polling** - Workers poll indefinitely, can exhaust resources
3. **Concurrent Team File Writes** - Race condition in sync path (setMemberMode)
4. **Missing isActive() Validation** - Pane-based teammates assumed alive

### High-Risk Issues (Probable Exploitation Path)

5. **Permission File Content Disclosure** - Tool input (with secrets) world-readable
6. **Malicious Team File Injection** - Untrusted team-wide permission rules
7. **Symlink Attack on Mailbox** - Symlink in ~/.claude/teams can redirect messages
8. **it2 Supply Chain** - pip install of it2 from untrusted PyPI

### Medium-Risk Issues (Possible Exploitation)

9. **Shell Initialization Timing** - 200ms may be insufficient for all shells
10. **Async Context Pollution** - Old Node.js versions leak ALS context
11. **Pane Lock Deadlock** - No timeout on pane creation lock acquisition
12. **Incomplete Cleanup** - Orphaned panes/worktrees if cleanup crashes

### Recommended Mitigations

1. **Validate Permission Bridge Setter**: Check if called from React context only
2. **Add Polling Timeout**: Max 5 minutes of waiting for permission/shutdown response
3. **Make Team File Atomic**: Use lockfile even for sync path (or all async)
4. **Implement Pane Liveness Check**: Query backend for pane existence in isActive()
5. **Isolate Permission Files**: Store in mode 0600, gzip if secrets included
6. **Validate Team File**: Reject paths with .. or /etc in teamAllowedPaths
7. **Symlink Resolution**: Use fs.realpathSync in mailbox path construction
8. **Sandboxed it2 Install**: Pin it2 version, use hash verification

---

## CONCLUSION

The Claude Code swarm system is a sophisticated multi-layer IPC architecture supporting both process-isolated and in-process execution. The design successfully abstracts three backend types through a unified TeammateExecutor interface, enabling flexible deployment. However, the system exhibits several concerning security gaps:

- **Privilege escalation vectors** via unvalidated permission bridge registration
- **Resource exhaustion** via unbound polling loops
- **Data integrity issues** from race conditions in team file updates
- **Information disclosure** through world-readable permission and mailbox files

The implementation prioritizes functionality over security isolation. For production use with untrusted code, additional sandboxing and validation layers would be required.

**Risk Level**: MEDIUM (acceptable for internal/trusted development, risky for multi-tenant)

