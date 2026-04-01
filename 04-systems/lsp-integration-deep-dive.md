# Claude Code LSP Integration: Comprehensive Reverse Engineering Report

**Analysis Date:** 2026-04-02
**Files Analyzed:** 7 TypeScript files, ~2,460 LOC
**Purpose:** Complete system architecture and protocol engineering documentation

---

## Executive Summary

Anthropic engineered Claude Code's LSP support as a **plugin-driven, multi-server, asynchronous diagnostic pipeline**. The system spawns independent LSP server processes, communicates via JSON-RPC over stdio, tracks server health/state, and delivers diagnostics to Claude's conversation context without blocking user interaction. The architecture emphasizes **graceful degradation** (LSP is optional), **crash recovery** (automatic restart on failure), and **deduplication** (preventing duplicate diagnostics across conversation turns).

**Key Design Principle:** Factory function pattern with closures for state encapsulation (no classes), enabling lazy-loading of the expensive vscode-jsonrpc library (~129KB) only when LSP servers are instantiated.

---

## 1. PROTOCOL AND COMMUNICATION

### 1.1 Transport Protocol

**Mechanism:** JSON-RPC 2.0 over stdin/stdout with child process spawning

**Implementation in `LSPClient.ts:88-105`:**
```typescript
process = spawn(command, args, {
  stdio: ['pipe', 'pipe', 'pipe'],
  env: { ...subprocessEnv(), ...options?.env },
  cwd: options?.cwd,
  windowsHide: true,
})
```

- **stdio configuration:** Standard three-stream piping (stdin, stdout, stderr)
- **Windows consideration:** `windowsHide: true` prevents visible console window on Windows
- **Environment merging:** Combines subprocess environment (`subprocessEnv()`) with caller-supplied env vars

**Message Connection Setup (LSPClient.ts:180-183):**
```typescript
const reader = new StreamMessageReader(process.stdout)
const writer = new StreamMessageWriter(process.stdin)
connection = createMessageConnection(reader, writer)
```

- Uses vscode-jsonrpc library's `StreamMessageReader` and `StreamMessageWriter`
- Handles framing, JSON encoding/decoding, and message ordering
- Protocol tracing enabled via `connection.trace(Trace.Verbose, {...})`

### 1.2 LSP Protocol Lifecycle

**Startup Sequence (LSPServerInstance.ts:135-264):**

1. **Spawn process** (`client.start()`)
   - Execute LSP server binary with args
   - Wait for 'spawn' event before using streams
   - Handle spawn failures (ENOENT, permission denied, etc.)

2. **Initialize** (`client.initialize()`)
   - Send `initialize` request with InitializeParams
   - Receive InitializeResult containing server capabilities
   - Send `initialized` notification
   - Set `isInitialized = true`

3. **Ready for requests** - Server now accepts LSP method calls

**Shutdown Sequence (LSPClient.ts:373-445):**
```typescript
await connection.sendRequest('shutdown', {})
await connection.sendNotification('exit', {})
```
- Graceful shutdown: shutdown request, then exit notification
- Fallback cleanup: dispose connection, kill process, clear event listeners
- State reset: `isInitialized = false`, `capabilities = undefined`

### 1.3 Initialization Parameters

**Generated in LSPServerInstance.ts:167-237:**

```typescript
const initParams: InitializeParams = {
  processId: process.pid,
  initializationOptions: config.initializationOptions ?? {},
  workspaceFolders: [{
    uri: pathToFileURL(workspaceFolder).href,
    name: path.basename(workspaceFolder),
  }],
  rootPath: workspaceFolder,        // Deprecated but still needed
  rootUri: workspaceUri,             // Deprecated but still needed
  capabilities: {
    workspace: {
      configuration: false,          // Don't support workspace/configuration
      workspaceFolders: false,        // Don't support workspace/didChangeWorkspaceFolders
    },
    textDocument: {
      synchronization: { didSave: true, ... },
      publishDiagnostics: { relatedInformation: true, ... },
      hover: { contentFormat: ['markdown', 'plaintext'] },
      definition: { linkSupport: true },
      references: {},
      documentSymbol: { hierarchicalDocumentSymbolSupport: true },
      callHierarchy: {},
    },
    general: { positionEncodings: ['utf-16'] },
  }
}
```

**Key Characteristics:**
- **Client capabilities:** Declares supported LSP features to server
- **Deprecated fields:** Both `rootPath`/`rootUri` included for compatibility with older servers
- **Disable workspace/configuration:** Prevents servers from requesting config not implemented
- **TextDocument features:** Hover, definition, references, document symbols, call hierarchy support

**Workspace Folder Setup (LSPServerInstance.ts:164-165):**
- Uses `pathToFileURL()` to convert filesystem paths to file:// URIs
- Handles workspace resolution via `config.workspaceFolder || getCwd()`

---

## 2. SERVER ARCHITECTURE AND STATE MACHINES

### 2.1 LSP Server Instance Lifecycle

**State Machine (LSPServerInstance.ts:74-78):**
```
stopped → starting → running
running → stopping → stopped
any state → error → starting (on retry)
```

**State Enumeration (LSPServerInstance.ts:113, config schema):**
- `'stopped'` - Server not running
- `'starting'` - Startup in progress
- `'running'` - Ready for requests
- `'stopping'` - Shutdown in progress
- `'error'` - Crashed or failed

**Health Check (LSPServerInstance.ts:338-340):**
```typescript
function isHealthy(): boolean {
  return state === 'running' && client.isInitialized
}
```

### 2.2 Crash Recovery and Retry Logic

**Transient Error Handling (LSPServerInstance.ts:342-410):**

**Transient Error Constant (LSPServerInstance.ts:17):**
```typescript
const LSP_ERROR_CONTENT_MODIFIED = -32801
```

**Retry Logic:**
- **Max retries:** `MAX_RETRIES_FOR_TRANSIENT_ERRORS = 3`
- **Base delay:** `RETRY_BASE_DELAY_MS = 500` ms
- **Exponential backoff:** `delay = 500 * 2^attempt` → 500ms, 1000ms, 2000ms

**Implementation (LSPServerInstance.ts:355-410):**
```typescript
for (let attempt = 0; attempt <= MAX_RETRIES_FOR_TRANSIENT_ERRORS; attempt++) {
  try {
    return await client.sendRequest(method, params)
  } catch (error) {
    const errorCode = (error as { code?: number }).code
    const isContentModifiedError = errorCode === LSP_ERROR_CONTENT_MODIFIED

    if (isContentModifiedError && attempt < MAX_RETRIES_FOR_TRANSIENT_ERRORS) {
      const delay = RETRY_BASE_DELAY_MS * Math.pow(2, attempt)
      await sleep(delay)
      continue
    }
    break
  }
}
```

**Crash Recovery (LSPServerInstance.ts:121-125, 142-150):**
- Crash callback: `onCrash` callback triggered on non-zero exit codes
- Sets state to `'error'` and increments `crashRecoveryCount`
- Max crash recoveries: `config.maxRestarts ?? 3` (default 3)
- Prevents unbounded child process spawning on persistent crashes

### 2.3 Multi-Server Manager

**Manager Type (LSPServerManager.ts:59-420):**

**Core Data Structures (LSPServerManager.ts:61-64):**
```typescript
const servers: Map<string, LSPServerInstance> = new Map()
const extensionMap: Map<string, string[]> = new Map()
const openedFiles: Map<string, string> = new Map()  // URI → server name
```

**File Extension Mapping (LSPServerManager.ts:88-117):**
- Derives file extensions from each server's `extensionToLanguage` mapping
- Maps extensions (lowercase, e.g., ".ts", ".py") to server names
- Multiple servers can handle same extension (first registered wins)

**File Open Tracking (LSPServerManager.ts:270-298):**
```typescript
// Prevent duplicate didOpen notifications
if (openedFiles.get(fileUri) === server.name) {
  return  // Already opened on this server
}

await server.sendNotification('textDocument/didOpen', {
  textDocument: {
    uri: fileUri,
    languageId: server.config.extensionToLanguage[ext] || 'plaintext',
    version: 1,
    text: content,
  },
})
openedFiles.set(fileUri, server.name)
```

---

## 3. DOCUMENT SYNCHRONIZATION

### 3.1 Synchronization Notifications

**Three-Phase Lifecycle:**

1. **didOpen** (LSPServerManager.ts:289-296)
   - First notification for a file
   - Includes full document text
   - Sets initial version number (1)
   - Triggered on file open or first change

2. **didChange** (LSPServerManager.ts:327-333)
   - Subsequent file modifications
   - Only sent if file already open on server
   - Uses full-text replacement: `contentChanges: [{ text: content }]`
   - Incremental changes not implemented (always full-text sync)

3. **didSave** (LSPServerManager.ts:354-357)
   - Sent after file written to disk
   - Triggers server-side diagnostics
   - Lightweight notification (no content)

4. **didClose** (LSPServerManager.ts:377-390)
   - Removes file from server tracking
   - Cleans up `openedFiles` map
   - Allows file to be reopened later

**Safety Features (LSPServerManager.ts:312-324):**
```typescript
// If file hasn't been opened on this server yet, open it first
// LSP servers require didOpen before didChange
if (openedFiles.get(fileUri) !== server.name) {
  return openFile(filePath, content)
}
```

---

## 4. DIAGNOSTIC DELIVERY PIPELINE

### 4.1 Diagnostic Notification Handling

**Passive Feedback Handler Registration (passiveFeedback.ts:125-328):**

**Handler Setup (passiveFeedback.ts:161-278):**
```typescript
serverInstance.onNotification(
  'textDocument/publishDiagnostics',
  (params: unknown) => {
    // 1. Validate params structure
    if (!params || typeof params !== 'object' || !('uri' in params) || !('diagnostics' in params)) {
      return  // Skip invalid notifications
    }

    // 2. Convert LSP → Claude diagnostic format
    const diagnosticFiles = formatDiagnosticsForAttachment(params)

    // 3. Register for async delivery
    registerPendingLSPDiagnostic({ serverName, files: diagnosticFiles })
  },
)
```

**Severity Mapping (passiveFeedback.ts:18-35):**
```typescript
function mapLSPSeverity(lspSeverity: number | undefined): 'Error' | 'Warning' | 'Info' | 'Hint' {
  switch (lspSeverity) {
    case 1: return 'Error'     // DiagnosticSeverity.Error
    case 2: return 'Warning'   // DiagnosticSeverity.Warning
    case 3: return 'Info'      // DiagnosticSeverity.Information
    case 4: return 'Hint'      // DiagnosticSeverity.Hint
    default: return 'Error'
  }
}
```

### 4.2 Diagnostic Registry and Deduplication

**Registry State (LSPDiagnosticRegistry.ts:49-56):**
```typescript
const pendingDiagnostics = new Map<string, PendingLSPDiagnostic>()
const deliveredDiagnostics = new LRUCache<string, Set<string>>({
  max: MAX_DELIVERED_FILES,  // 500 files tracked
})
```

**Deduplication Strategy (LSPDiagnosticRegistry.ts:136-184):**

1. **Within-batch deduplication:** Same file/diagnostic combination in single delivery
2. **Cross-turn deduplication:** Track delivered diagnostics across conversation turns
3. **Deduplication key (LSPDiagnosticRegistry.ts:110-124):**
   ```typescript
   function createDiagnosticKey(diag: {...}): string {
     return jsonStringify({
       message: diag.message,
       severity: diag.severity,
       range: diag.range,
       source: diag.source || null,
       code: diag.code || null,
     })
   }
   ```

**Volume Limiting (LSPDiagnosticRegistry.ts:257-289):**
- Max 10 diagnostics per file
- Max 30 total diagnostics per delivery
- Sorted by severity (Errors first)
- Truncated with logging

**Delivery Checkpoint (LSPDiagnosticRegistry.ts:193-338):**
```typescript
export function checkForLSPDiagnostics(): Array<{
  serverName: string
  files: DiagnosticFile[]
}> {
  // Collect all pending diagnostics
  // Deduplicate across all files
  // Apply volume limiting
  // Track in deliveredDiagnostics LRU
  // Return deduplicated files ready for attachment
}
```

---

## 5. ERROR HANDLING AND RESILIENCE

### 5.1 Process-Level Error Handling

**Spawn Validation (LSPClient.ts:106-131):**
```typescript
// Wait for spawn event before using streams
// This catches ENOENT (command not found), permission errors, etc.
await new Promise<void>((resolve, reject) => {
  const onSpawn = (): void => { cleanup(); resolve() }
  const onError = (error: Error): void => { cleanup(); reject(error) }
  spawnedProcess.once('spawn', onSpawn)
  spawnedProcess.once('error', onError)
})
```

**Process Error Handlers (LSPClient.ts:143-179):**
```typescript
// Handle crashes during operation (after successful spawn)
process.on('error', error => {
  if (!isStopping) {
    startFailed = true
    startError = error
    logError(new Error(`LSP server ${serverName} failed to start: ${error.message}`))
  }
})

// Handle non-zero exit codes
process.on('exit', (code, _signal) => {
  if (code !== 0 && code !== null && !isStopping) {
    isInitialized = false
    const crashError = new Error(`LSP server ${serverName} crashed with exit code ${code}`)
    onCrash?.(crashError)
  }
})

// Handle stdin stream errors (prevent unhandled rejections)
process.stdin.on('error', (error: Error) => {
  if (!isStopping) {
    logForDebugging(`LSP server ${serverName} stdin error: ${error.message}`)
  }
})
```

### 5.2 Connection-Level Error Handling

**Error & Close Handlers (LSPClient.ts:187-207):**
```typescript
connection.onError(([error, _message, _code]) => {
  if (!isStopping) {
    startFailed = true
    startError = error
    logError(new Error(`LSP server ${serverName} connection error: ${error.message}`))
  }
})

connection.onClose(() => {
  if (!isStopping) {
    isInitialized = false
    logForDebugging(`LSP server ${serverName} connection closed`)
  }
})
```

### 5.3 Intentional Shutdown Flag

**Stopping Marker (LSPClient.ts:62, 144, 189, 202, 322, 377):**
```typescript
let isStopping = false  // Track intentional shutdown

// During stop()
isStopping = true
// ... perform cleanup ...
isStopping = false  // Reset for potential restart
```

**Purpose:** Distinguish intentional shutdown from crashes, preventing spurious error logging during normal teardown.

### 5.4 Request-Level Error Handling

**Health Check Before Requests (LSPServerInstance.ts:355-363):**
```typescript
async function sendRequest<T>(method: string, params: unknown): Promise<T> {
  if (!isHealthy()) {
    const error = new Error(
      `Cannot send request to LSP server '${name}': server is ${state}` +
      `${lastError ? `, last error: ${lastError.message}` : ''}`
    )
    throw error
  }
  // ... attempt request with retry logic ...
}
```

---

## 6. PLUGIN CONFIGURATION SYSTEM

### 6.1 Configuration Loading

**Source (config.ts:15-79):**
- LSP servers loaded only from plugins (not user/project settings)
- Loaded asynchronously during initialization
- Parallel plugin processing via Promise.all()
- Merged into single configuration object

**Configuration Schema (plugins/schemas.ts:708-788):**

```typescript
export const LspServerConfigSchema = z.strictObject({
  command: z.string().min(1),
  args: z.array(nonEmptyString()).optional(),
  extensionToLanguage: z.record(fileExtension(), nonEmptyString()),
  transport: z.enum(['stdio', 'socket']).default('stdio'),
  env: z.record(z.string(), z.string()).optional(),
  initializationOptions: z.unknown().optional(),
  settings: z.unknown().optional(),
  workspaceFolder: z.string().optional(),
  startupTimeout: z.number().int().positive().optional(),
  shutdownTimeout: z.number().int().positive().optional(),
  restartOnCrash: z.boolean().optional(),
  maxRestarts: z.number().int().nonnegative().optional(),
})
```

### 6.2 Type Definitions

**Core Configuration Type (LSPServerConfig):**
- `command` (required): Server executable (no spaces - use args for arguments)
- `args` (optional): Command-line arguments
- `extensionToLanguage` (required): Map from file extensions to LSP language IDs
- `transport` (default: 'stdio'): Currently only stdio implemented
- `env` (optional): Environment variables
- `initializationOptions` (optional): Server-specific init options
- `settings` (optional): Server settings (via workspace/didChangeConfiguration)
- `workspaceFolder` (optional): Workspace path (defaults to cwd)
- `startupTimeout` (optional): Max initialization wait (ms)
- `shutdownTimeout` (optional): Not yet implemented
- `restartOnCrash` (optional): Not yet implemented
- `maxRestarts` (optional, default: 3): Max crash recovery attempts

**Scoped Configuration (ScopedLspServerConfig):**
- Derived from LspServerConfig with plugin scope prefix
- Server names scoped to avoid collisions (e.g., "plugin-name:typescript-lsp")

---

## 7. LAZY INITIALIZATION AND SINGLETON MANAGEMENT

### 7.1 Global Singleton Pattern (manager.ts:20-35)

```typescript
let lspManagerInstance: LSPServerManager | undefined
let initializationState: InitializationState = 'not-started'
let initializationError: Error | undefined
let initializationGeneration = 0
let initializationPromise: Promise<void> | undefined
```

**State Transitions (manager.ts:14):**
```typescript
type InitializationState = 'not-started' | 'pending' | 'success' | 'failed'
```

### 7.2 Initialization (manager.ts:145-208)

**Lazy Initialization Pattern:**
1. Called during Claude Code startup
2. Creates manager synchronously
3. Starts async initialization without blocking
4. Returns immediately (non-blocking)

**Generation Counter (manager.ts:35, 173):**
- Prevents stale initializations from updating state
- Incremented on reinit/shutdown
- Current generation checked before state updates

**Initialization Promise Tracking (manager.ts:180-207):**
```typescript
initializationPromise = lspManagerInstance
  .initialize()
  .then(() => {
    if (currentGeneration === initializationGeneration) {
      initializationState = 'success'
      registerLSPNotificationHandlers(lspManagerInstance)
    }
  })
  .catch((error: unknown) => {
    if (currentGeneration === initializationGeneration) {
      initializationState = 'failed'
      initializationError = error as Error
      lspManagerInstance = undefined
    }
  })
```

### 7.3 Lazy vscode-jsonrpc Loading (LSPServerInstance.ts:106-112)

**Dynamic Require (avoiding ESM issues):**
```typescript
const { createLSPClient } = require('./LSPClient.js') as {
  createLSPClient: typeof createLSPClientType
}
```

**Benefit:** Defers loading of vscode-jsonrpc (~129KB) until first LSP server instantiation, not at module import time.

---

## 8. LSP CAPABILITIES AND FEATURES

### 8.1 Advertised Capabilities

**Client Capabilities in Initialize Request (LSPServerInstance.ts:188-236):**

**Workspace Support:**
- `configuration: false` - Don't support workspace/configuration requests
- `workspaceFolders: false` - Don't support workspace/didChangeWorkspaceFolders

**TextDocument Synchronization:**
- `didSave: true` - Trigger diagnostics on file save
- `willSave: false` - Don't support pre-save notifications
- `willSaveWaitUntil: false` - No pre-save wait support

**TextDocument Features:**
- `hover` - Hover information (markdown + plaintext)
- `definition` - Go-to-definition with link support
- `references` - Find references
- `documentSymbol` - Document outline with hierarchy support
- `callHierarchy` - Call hierarchy navigation
- `publishDiagnostics` - Diagnostic notifications with related information and tags

**General:**
- `positionEncodings: ['utf-16']` - UTF-16 position encoding

### 8.2 Unsupported Features (Explicitly Disabled)

- `workspace/configuration` - Would require config provider
- `workspace/didChangeWorkspaceFolders` - Would require workspace tracking
- Dynamic registration - All capabilities statically declared
- Incremental text document synchronization
- `textDocument/willSave` callbacks
- `textDocument/willSaveWaitUntil` support

---

## 9. CONFIGURATION VALIDATION AND INITIALIZATION

### 9.1 Server Initialization (LSPServerManager.ts:71-148)

**Validation Steps:**
```typescript
// 1. Validate config has required fields
if (!config.command) {
  throw new Error(`Server ${serverName} missing required 'command' field`)
}
if (!config.extensionToLanguage || Object.keys(config.extensionToLanguage).length === 0) {
  throw new Error(`Server ${serverName} missing required 'extensionToLanguage' field`)
}

// 2. Map file extensions to server
const fileExtensions = Object.keys(config.extensionToLanguage)
for (const ext of fileExtensions) {
  const normalized = ext.toLowerCase()
  if (!extensionMap.has(normalized)) extensionMap.set(normalized, [])
  extensionMap.get(normalized)!.push(serverName)
}

// 3. Create server instance
const instance = createLSPServerInstance(serverName, config)
servers.set(serverName, instance)

// 4. Register default handlers (workspace/configuration)
instance.onRequest('workspace/configuration', (params) => {
  return params.items.map(() => null)  // Return null config for each item
})
```

**Error Handling:** Continues with other servers if one fails (graceful degradation).

---

## 10. SECURITY BOUNDARIES AND ATTACK VECTORS

### 10.1 Process Isolation

**Security Model:**
- LSP servers run as separate child processes (independent privilege boundary)
- Stdio pipes restrict communication to JSON-RPC only
- No direct file system access (files passed via protocol)
- No access to Claude Code's memory or network

### 10.2 URI Validation and Path Traversal Prevention

**URI Handling (passiveFeedback.ts:47-61):**
```typescript
let uri: string
try {
  uri = params.uri.startsWith('file://')
    ? fileURLToPath(params.uri)
    : params.uri
} catch (error) {
  // Gracefully fallback to original URI
  uri = params.uri
}
```

**File Path Normalization (LSPServerManager.ts:274, 318, 356, 381):**
```typescript
const fileUri = pathToFileURL(path.resolve(filePath)).href
```
- Resolves relative paths to absolute paths
- Converts to file:// URIs
- Prevents directory traversal via relative path components

### 10.3 Configuration Validation

**Schema Validation (plugins/schemas.ts:708-788):**
- Zod strict object validation
- Command field validated (no spaces)
- extensionToLanguage required and non-empty
- Positive integer timeouts
- Type-safe configuration

**Path Traversal Prevention in Plugin (lspPluginIntegration.ts:28-45):**
```typescript
function validatePathWithinPlugin(pluginPath: string, relativePath: string): string | null {
  const resolvedPluginPath = resolve(pluginPath)
  const resolvedFilePath = resolve(pluginPath, relativePath)
  const rel = relative(resolvedPluginPath, resolvedFilePath)

  if (rel.startsWith('..') || resolve(rel) === rel) {
    return null  // Path escapes plugin directory
  }
  return resolvedFilePath
}
```

### 10.4 Handler Isolation and Error Containment

**Error Isolation (passiveFeedback.ts:249-276, LSPServerManager.ts:270-343):**
```typescript
// Errors in handler don't break notification loop
try {
  // Handler logic
} catch (error) {
  const err = toError(error)
  logError(err)
  // Don't re-throw - isolate errors to this server only
}
```

**Per-Server Failure Tracking (passiveFeedback.ts:136-137, 232-247):**
```typescript
const diagnosticFailures: Map<string, { count: number; lastError: string }> = new Map()

// Track consecutive failures per server
if (failures.count >= 3) {
  logForDebugging(`WARNING: LSP diagnostic handler for ${serverName} has failed...`)
}
```

---

## 11. LIFECYCLE AND TIMING

### 11.1 Startup Sequence

1. **Startup (Claude Code starts)**
   - `initializeLspServerManager()` called
   - Creates manager, marks state as 'pending'
   - Returns immediately (non-blocking)

2. **Async Initialization (background)**
   - `getAllLspServers()` loads plugin configs
   - Creates LSPServerInstance objects (no servers started yet)
   - Registers notification handlers for all servers
   - Sets state to 'success'

3. **First Use (file open/hover/etc.)**
   - `ensureServerStarted(filePath)` called
   - Gets server for file extension
   - Calls `server.start()` if not running
   - Server binary spawned, initialization handshake occurs
   - Server now ready for requests

4. **Active Use (subsequent requests)**
   - Requests routed to running servers
   - Diagnostics accumulated in registry
   - Delivered to conversation as attachments

### 11.2 Shutdown Sequence

1. **Shutdown Initiated**
   - `shutdownLspServerManager()` called
   - Filters servers in 'running' or 'error' state
   - Calls `stop()` on each server via Promise.allSettled()

2. **Per-Server Stop (LSPClient.ts:373-445)**
   - Send `shutdown` request
   - Send `exit` notification
   - Dispose connection
   - Kill process (graceful timeout not implemented)
   - Clear event listeners

3. **Final State Reset**
   - Clear servers, extensionMap, openedFiles
   - Clear pendingDiagnostics, deliveredDiagnostics
   - Increment generation counter

### 11.3 Timeout Handling

**Initialization Timeout (LSPServerInstance.ts:240-248):**
```typescript
if (config.startupTimeout !== undefined) {
  await withTimeout(
    initPromise,
    config.startupTimeout,
    `LSP server '${name}' timed out after ${config.startupTimeout}ms during initialization`,
  )
}
```

**Timeout Implementation (LSPServerInstance.ts:499-511):**
```typescript
function withTimeout<T>(promise: Promise<T>, ms: number, message: string): Promise<T> {
  let timer: ReturnType<typeof setTimeout>
  const timeoutPromise = new Promise<never>((_, reject) => {
    timer = setTimeout((rej, msg) => rej(new Error(msg)), ms, reject, message)
  })
  return Promise.race([promise, timeoutPromise]).finally(() => clearTimeout(timer!))
}
```

**Timeout Cleanup:**
- On timeout, client.stop() called to kill spawned process
- Prevents orphaned child processes
- Clears initialization promise to prevent unhandled rejections

---

## 12. HANDLER REGISTRATION AND NOTIFICATION FLOW

### 12.1 Lazy Handler Queuing

**Pre-Connection Registration (LSPClient.ts:337-350, 352-371):**
```typescript
const pendingHandlers: Array<{ method: string; handler: ... }> = []
const pendingRequestHandlers: Array<{ method: string; handler: ... }> = []

onNotification(method: string, handler: ...): void {
  if (!connection) {
    // Queue handler for application when connection is ready
    pendingHandlers.push({ method, handler })
    return
  }
  connection.onNotification(method, handler)
}
```

**Handler Application on Connection Ready (LSPClient.ts:228-244):**
```typescript
for (const { method, handler } of pendingHandlers) {
  connection.onNotification(method, handler)
}
pendingHandlers.length = 0  // Clear the queue
```

### 12.2 Reverse-Direction Requests

**Server-to-Client Requests (LSPServerManager.ts:125-135):**

Some LSP servers send requests TO the client (reverse direction):
```typescript
instance.onRequest('workspace/configuration', (params) => {
  logForDebugging(`LSP: Received workspace/configuration request from ${serverName}`)
  return params.items.map(() => null)
})
```

Example: TypeScript language server sends `workspace/configuration` requests even though client declares no support.

---

## 13. DIAGNOSTIC ATTACHMENT SYSTEM

### 13.1 Diagnostic File Format

**DiagnosticFile Structure (LSPDiagnosticRegistry.ts:16):**
```typescript
export type PendingLSPDiagnostic = {
  serverName: string
  files: DiagnosticFile[]
  timestamp: number
  attachmentSent: boolean
}
```

**Individual Diagnostic Structure (passiveFeedback.ts:63-92):**
```typescript
{
  message: string
  severity: 'Error' | 'Warning' | 'Info' | 'Hint'
  range: {
    start: { line: number; character: number }
    end: { line: number; character: number }
  }
  source?: string    // Server identifier
  code?: string      // Diagnostic code
}
```

### 13.2 Delivery Pipeline

**Flow (LSPDiagnosticRegistry.ts:33-38):**
1. LSP server sends `textDocument/publishDiagnostics` notification
2. `registerPendingLSPDiagnostic()` stores diagnostic
3. `checkForLSPDiagnostics()` retrieves pending diagnostics
4. Deduplication, volume limiting applied
5. `getLSPDiagnosticAttachments()` converts to Attachment[]
6. `getAttachments()` delivers to conversation automatically

**Async Pattern:**
- Diagnostics registered when received
- Checked on next user query
- Delivered as attachments without blocking
- Prevents interrupting user interaction

---

## 14. CONSTANTS AND MAGIC NUMBERS

### Critical Values

| Constant | Value | Location | Purpose |
|----------|-------|----------|---------|
| `LSP_ERROR_CONTENT_MODIFIED` | -32801 | LSPServerInstance.ts:17 | Transient error code for retry logic |
| `MAX_RETRIES_FOR_TRANSIENT_ERRORS` | 3 | LSPServerInstance.ts:22 | Max attempts on content-modified errors |
| `RETRY_BASE_DELAY_MS` | 500 | LSPServerInstance.ts:28 | Initial backoff delay in exponential sequence |
| `MAX_DIAGNOSTICS_PER_FILE` | 10 | LSPDiagnosticRegistry.ts:42 | Diagnostic volume limit per file |
| `MAX_TOTAL_DIAGNOSTICS` | 30 | LSPDiagnosticRegistry.ts:43 | Total diagnostic limit per delivery |
| `MAX_DELIVERED_FILES` | 500 | LSPDiagnosticRegistry.ts:46 | LRU cache size for cross-turn deduplication |

---

## 15. DEPENDENCY ANALYSIS

### Key External Dependencies

**vscode-jsonrpc/node.js:**
- `createMessageConnection()` - Creates JSON-RPC connection
- `StreamMessageReader` - Reads JSON-RPC messages from stream
- `StreamMessageWriter` - Writes JSON-RPC messages to stream
- `Trace` enum - For protocol tracing
- ~129KB library size (lazy-loaded)

**vscode-languageserver-protocol:**
- `InitializeParams` - Server initialization request type
- `InitializeResult` - Server initialization response type
- `ServerCapabilities` - Server feature declarations
- `PublishDiagnosticsParams` - Diagnostic notification type

**Built-in Node.js:**
- `child_process.spawn()` - Process spawning
- `fs` and `path` - File system operations
- `url.pathToFileURL()` - URI conversion

### Internal Dependencies

- `logForDebugging()`, `logError()` - Debugging and error logging
- `errorMessage()`, `toError()` - Error utilities
- `subprocessEnv()` - Environment setup
- `sleep()` - Delay utility
- `jsonStringify()` - JSON serialization
- `LRUCache` - Bounded memory caching

---

## 16. TESTING AND OBSERVABILITY

### 16.1 Debug Logging

**Pervasive Logging (throughout codebase):**
```typescript
logForDebugging(`LSP client started for ${serverName}`)
logForDebugging(`[LSP SERVER ${serverName}] ${output}`)
logForDebugging(`LSP Diagnostics: Deduplication removed ${count} duplicate(s)`)
```

**Error Logging:**
```typescript
logError(new Error(`LSP server ${serverName} crashed with exit code ${code}`))
```

### 16.2 Test-Only Functions

**Manager Reset (manager.ts:48-53):**
```typescript
export function _resetLspManagerForTesting(): void {
  initializationState = 'not-started'
  initializationError = undefined
  initializationPromise = undefined
  initializationGeneration++
}
```

**Diagnostic Reset (LSPDiagnosticRegistry.ts:357-363):**
```typescript
export function resetAllLSPDiagnosticState(): void {
  pendingDiagnostics.clear()
  deliveredDiagnostics.clear()
}
```

---

## 17. BARE MODE AND OPTIONAL LSP

### 17.1 Optional Feature Flag

**Bare Mode Check (manager.ts:147-149):**
```typescript
if (isBareMode()) {
  return  // --bare / SIMPLE: no LSP
}
```

**Purpose:** LSP is for editor integration (diagnostics, hover, go-to-def in REPL). Scripted `-p` calls have no use for it.

**LSP Disabled When:**
- `--bare` flag used
- `SIMPLE` environment variable set
- Headless subcommand path (no initialization)

---

## 18. REINIT AND PLUGIN REFRESH

### 18.1 Reinit on Plugin Reload

**Reinit Function (manager.ts:226-253):**
```typescript
export function reinitializeLspServerManager(): void {
  if (initializationState === 'not-started') return

  // Shutdown old servers (best-effort)
  if (lspManagerInstance) {
    void lspManagerInstance.shutdown().catch(...)
  }

  // Reset state and reinit
  lspManagerInstance = undefined
  initializationState = 'not-started'
  initializationGeneration++
  initializeLspServerManager()
}
```

**Issue Fix (manager.ts:216-220):**
Fixes https://github.com/anthropics/claude-code/issues/15521:
- `loadAllPlugins()` memoized and called early in startup
- Memoized result cached before marketplaces reconciled
- Results in 0 servers on initial LSP manager init
- Reinit now called on plugin refresh to pick up new servers

---

## 19. INTERFACE CONTRACTS

### 19.1 LSPClient Interface

**Minimum interface for JSON-RPC communication (LSPClient.ts:21-41):**
```typescript
type LSPClient = {
  readonly capabilities: ServerCapabilities | undefined
  readonly isInitialized: boolean
  start(command: string, args: string[], options?: {...}) => Promise<void>
  initialize(params: InitializeParams) => Promise<InitializeResult>
  sendRequest<TResult>(method: string, params: unknown) => Promise<TResult>
  sendNotification(method: string, params: unknown) => Promise<void>
  onNotification(method: string, handler: (params: unknown) => void) => void
  onRequest<TParams, TResult>(method: string, handler: (params: TParams) => TResult | Promise<TResult>) => void
  stop() => Promise<void>
}
```

### 19.2 LSPServerInstance Interface

**Per-server lifecycle management (LSPServerInstance.ts:33-65):**
```typescript
type LSPServerInstance = {
  readonly name: string
  readonly config: ScopedLspServerConfig
  readonly state: LspServerState
  readonly startTime: Date | undefined
  readonly lastError: Error | undefined
  readonly restartCount: number
  start() => Promise<void>
  stop() => Promise<void>
  restart() => Promise<void>
  isHealthy() => boolean
  sendRequest<T>(method: string, params: unknown) => Promise<T>
  sendNotification(method: string, params: unknown) => Promise<void>
  onNotification(method: string, handler: (params: unknown) => void) => void
  onRequest<TParams, TResult>(method: string, handler: (params: TParams) => TResult | Promise<TResult>) => void
}
```

### 19.3 LSPServerManager Interface

**Multi-server orchestration (LSPServerManager.ts:16-43):**
```typescript
type LSPServerManager = {
  initialize() => Promise<void>
  shutdown() => Promise<void>
  getServerForFile(filePath: string) => LSPServerInstance | undefined
  ensureServerStarted(filePath: string) => Promise<LSPServerInstance | undefined>
  sendRequest<T>(filePath: string, method: string, params: unknown) => Promise<T | undefined>
  getAllServers() => Map<string, LSPServerInstance>
  openFile(filePath: string, content: string) => Promise<void>
  changeFile(filePath: string, content: string) => Promise<void>
  saveFile(filePath: string) => Promise<void>
  closeFile(filePath: string) => Promise<void>
  isFileOpen(filePath: string) => boolean
}
```

---

## 20. KEY ARCHITECTURAL DECISIONS

### 20.1 Design Patterns

1. **Factory Functions with Closures (not classes)**
   - State encapsulation without class overhead
   - Clean separation of public API from private state
   - Enables lazy requires

2. **Lazy Initialization**
   - Non-blocking startup (LSP init in background)
   - Servers start on first use (not during init)
   - Generation counter prevents stale state updates

3. **Promise-Based Async**
   - All blocking operations (start, stop, initialize) return Promises
   - Proper error propagation
   - Timeout handling with cleanup

4. **Fail-Safe Error Handling**
   - Graceful degradation (LSP optional, continue without)
   - Per-server error isolation (one server failure doesn't break others)
   - Comprehensive logging for diagnostics

5. **Deduplication Pipeline**
   - Within-batch deduplication (same batch)
   - Cross-turn deduplication (LRU-tracked across turns)
   - Volume limiting (prevent spam)

### 20.2 Implementation Tradeoffs

| Decision | Tradeoff |
|----------|----------|
| Full-text sync only | Simpler implementation, but uses more bandwidth |
| Stdio-only transport | Eliminates socket implementation complexity, good for local use |
| No workspace/configuration | Simpler, but some servers expect to negotiate config |
| Process-per-server | Better isolation, but more resource usage |
| Flat extension map | First registered server wins (ambiguity), but simple logic |
| LRU diagnostic cache | Bounded memory, but recent diagnostics may be redelivered |
| Single initialization attempt | No retry loop, but consistent behavior |

---

## 21. IMPLEMENTATION GAPS AND UNIMPLEMENTED FEATURES

### 21.1 Explicitly Unimplemented (Validation Errors)

**Blocked in Config (LSPServerInstance.ts:95-104):**
```typescript
if (config.restartOnCrash !== undefined) {
  throw new Error(`LSP server '${name}': restartOnCrash is not yet implemented...`)
}
if (config.shutdownTimeout !== undefined) {
  throw new Error(`LSP server '${name}': shutdownTimeout is not yet implemented...`)
}
```

### 21.2 Partial Features

**Document Sync:**
- Only full-text synchronization (not incremental)
- Version numbering basic (always 1)

**Socket Transport:**
- Defined in schema but not implemented (only stdio works)

**File Close Integration:**
```typescript
// NOTE: Currently available but not yet integrated with compact flow.
// TODO: Integrate with compact - call closeFile() when compact removes files from context
```

### 21.3 Known Limitations

- No workspace folder change notifications (workspaceFolders: false)
- No configuration requests (workspace/configuration returns null)
- No dynamic capability registration
- No multi-workspace support (single workspace per server)
- No progress reporting ($/progress notifications ignored)
- No diagnostic versioning support

---

## 22. SECURITY CONSIDERATIONS

### 22.1 Threat Model

**Process Escape:** LSP server crashes don't affect Claude Code (separate process)

**Command Injection:** Command field validated (no spaces), args array prevents shell injection

**Path Traversal:** File paths converted to URIs, resolved to absolute paths

**Malicious Diagnostics:** Validated before delivery, serialized safely

**Configuration Injection:** Schema validation via Zod, strict object mode

### 22.2 Mitigation Strategies

1. **Process Isolation:** Separate child process, stdio-only communication
2. **Input Validation:** Schema validation, path normalization, URI conversion
3. **Error Isolation:** Handler errors don't propagate, per-server failure tracking
4. **Graceful Degradation:** LSP optional, one server failure doesn't break others
5. **Resource Limits:** Volume limiting (10/30 diagnostics), LRU cache (500 files), max restarts (3)

---

## 23. COMPLETE FILE ROLE SUMMARY

| File | LOC | Role |
|------|-----|------|
| LSPClient.ts | 447 | JSON-RPC transport layer, process lifecycle, connection management |
| LSPServerInstance.ts | 512 | Single server state machine, health checks, request retry logic |
| LSPServerManager.ts | 420 | Multi-server routing, file extension mapping, document sync orchestration |
| config.ts | 79 | Plugin LSP server configuration loading |
| manager.ts | 290 | Global singleton, async initialization, startup/shutdown orchestration |
| LSPDiagnosticRegistry.ts | 387 | Diagnostic storage, deduplication, volume limiting, cross-turn tracking |
| passiveFeedback.ts | 329 | Notification handler registration, LSP→Claude format conversion, error isolation |
| **TOTAL** | **2,460** | Complete LSP integration system |

---

## 24. CONCLUSION

Anthropic engineered Claude Code's LSP support as a **robust, plugin-driven, multi-server system** with:

- **Transport:** JSON-RPC 2.0 over stdio (process spawning + stream pipes)
- **State Management:** Finite state machines per server + singleton manager
- **Resilience:** Crash recovery (3 retries), transient error retry (exponential backoff), graceful degradation
- **Diagnostics:** Asynchronous delivery, within/cross-turn deduplication, volume limiting
- **Configuration:** Plugin-sourced via Zod schema validation
- **Isolation:** Per-server error containment, process separation
- **Initialization:** Non-blocking background setup, lazy server startup

The implementation prioritizes **reliability and observability** over feature completeness, with comprehensive logging and error isolation ensuring that a broken LSP server doesn't crash Claude Code.

---

**End of Report**
