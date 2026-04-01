# Claude Code: Suggestions and Secure Storage Systems Deep Dive

**Document**: Comprehensive reverse-engineering analysis of two critical Anthropic-engineered systems in Claude Code
**Coverage**:
- `/src/utils/suggestions/` (5 files, ~1,213 lines)
- `/src/utils/secureStorage/` (6 files, ~629 lines)

---

## Part 1: Command Suggestions System

### Overview

The suggestions system provides intelligent, context-aware command/skill auto-completion and recommendation. It uses a multi-layered approach combining fuzzy matching (Fuse.js library), usage-based scoring, and fallback mechanisms to surface the most relevant commands to users.

### Architecture: Data Flow

```
User Input (e.g., "/com", "/")
    ↓
[Command Suggestions Parser]
    ├─ Detect input type (slash command vs path vs shell history vs Slack channel)
    ├─ Parse partial command/token
    └─ Route to appropriate suggestion engine
    ↓
[Ranking & Filtering Engine]
    ├─ Fuzzy match (Fuse.js with weighted fields)
    ├─ Usage scoring (exponential decay over 7-day half-life)
    ├─ Prefix matching (exact > alias > fuzzy)
    └─ Categorization (built-in, user, project, policy)
    ↓
[SuggestionItem Array] → UI Display
```

### 1. Command Suggestions (`commandSuggestions.ts`)

#### Core Functions

##### `generateCommandSuggestions(input: string, commands: Command[]): SuggestionItem[]`

Main entry point for command suggestion generation. Handles two distinct modes:

**Mode 1: Empty Query (user typed only "/")**
- Finds top 5 recently-used skills by `getSkillUsageScore()`
- Categorizes remaining commands into 5 buckets:
  1. Built-in (`local`, `local-jsx` type)
  2. User settings (`userSettings`, `localSettings` source)
  3. Project settings (`projectSettings` source)
  4. Policy settings (`policySettings` source)
  5. Other commands (plugins, etc.)
- Each category sorted alphabetically by command name
- Returns: recently-used skills first, then built-in, then user, project, policy, other

**Mode 2: Typed Query (e.g., "/com")**
- Strips leading "/" and converts to lowercase
- Runs Fuse.js fuzzy search against command database
- Implements sophisticated ranking:
  1. Exact name match (highest priority)
  2. Exact alias match
  3. Prefix name match (prefers shorter names)
  4. Prefix alias match (prefers shorter aliases)
  5. Fuzzy match with usage tiebreaker
- Returns sorted results as SuggestionItem array

**Key Heuristic: Hidden Command Handling**
- Commands marked `isHidden=true` are excluded from Fuse index build
- If user types exact name matching a hidden command AND no visible command shares the name, that hidden command is prepended to results
- Prevents duplicates when `isHidden` flips mid-session (OAuth expiry, feature flags)

##### `getCommandFuse(commands: Command[]): Fuse<CommandSearchItem>`

Builds and caches Fuse.js search index:

```typescript
const fuse = new Fuse(commandData, {
  includeScore: true,
  threshold: 0.3,         // Relatively strict (30% required similarity)
  location: 0,            // Prefer matches at string beginning
  distance: 100,          // Allow matching throughout descriptions
  keys: [
    { name: 'commandName', weight: 3 },     // Command names: highest weight
    { name: 'partKey', weight: 2 },         // Command parts (split by -_:): high
    { name: 'aliasKey', weight: 2 },        // Aliases: high
    { name: 'descriptionKey', weight: 0.5 } // Descriptions: low
  ]
})
```

**Cache Strategy**:
- Keyed by commands array identity (object reference)
- Commands array is memoized in REPL.tsx, so cache stays valid across renders
- Single cache entry across entire session
- Only rebuilds when commands array reference changes

**Data Transformation Before Indexing**:
- Command names split by separators (`/[:_-]/g`)
- Descriptions split into words, cleaned (lowercase, remove non-alphanumeric)
- Aliases stored as-is
- Results in `CommandSearchItem` with cleaned keys for fuzzy matching

##### `findMidInputSlashCommand(input: string, cursorOffset: number): MidInputSlashCommand | null`

Detects slash commands typed mid-input (not at position 0):

```typescript
// Pattern: whitespace + "/" + alphanumeric/dash/underscore/colon
const match = beforeCursor.match(/\s\/([a-zA-Z0-9_:-]*)$/)
```

Returns:
```typescript
{
  token: string,        // e.g., "/com"
  startPos: number,     // Position of "/" in input
  partialCommand: string // e.g., "com"
}
```

**Optimization Note**: Avoids lookbehind regex (`(?<=\s)`) because it defeats JSC JIT compilation. Instead captures whitespace and offsets `match.index` by 1.

##### `getBestCommandMatch(partialCommand: string, commands: Command[]): { suffix, fullCommand } | null`

Finds best matching command for inline ghost-text completion:

1. Generates suggestions for the partial command
2. Returns first suggestion that is a **prefix match** (e.g., "com" → "commit")
3. Returns suffix to display as ghost text ("mit" for "com" → "commit")
4. Returns null if no suitable completion exists

#### Data Structures

**SuggestionItem** (exported interface):
```typescript
{
  id: string                    // Unique identifier (deduped by command name + source)
  displayText: string           // e.g., "/commit" or "/commit (c)" if alias matched
  tag?: string                  // e.g., "workflow" for workflow-type commands
  description: string           // Full description + source + arguments info
  metadata: Command             // Original command object
}
```

**CommandSearchItem** (internal to Fuse.js):
```typescript
{
  descriptionKey: string[]      // Words from description (cleaned)
  partKey: string[] | undefined // Command parts if hyphenated (e.g., "voice" + "memo")
  commandName: string
  command: Command
  aliasKey: string[] | undefined // Command aliases (unchanged from Command.aliases)
}
```

#### Ranking Algorithm Details

When query typed (e.g., "/voice-m"):

1. **Run Fuse fuzzy search** → array of { r: FuseResult, name, aliases, usage }
2. **Sort by priority chain**:
   - Exact name match > Exact alias > Prefix name > Prefix alias > Fuzzy + usage
3. **Tiebreaker**: Among prefix name matches, prefer shorter names (closer to exact)
4. **Secondary tiebreaker**: Fuse score difference > usage score (7-day exponential decay)

**Usage Scoring** (from `skillUsageTracking.ts`):
```
score = usageCount * Math.max(recencyFactor, 0.1)
where recencyFactor = 0.5 ^ (daysSinceUse / 7)
```

Half-life of 7 days means:
- Skills used today: full weight
- Skills used 7 days ago: 50% weight
- Skills used 14 days ago: 25% weight
- Minimum floor of 0.1 prevents very old heavily-used skills from dropping to zero

#### Alias Handling

Commands can have `aliases` array. When displaying suggestions:
- If user typed a prefix matching an alias, show it in parentheses: `/commit (c)`
- Only show alias if user actually typed it (don't pollute suggestions list)
- Aliases get same weight as command names in fuzzy search (weight: 2)

#### Mid-Input Slash Commands

Via `findMidInputSlashCommand()`:
- Allows typing commands in middle of input: "Please run /gre grep"
- Ghost-text shows completion via `getBestCommandMatch()`
- Position tracking enables cursor placement after completion

#### Visibility & Permission Handling

- `isHidden=true` commands excluded from normal suggestions
- Hidden exact matches shown if no visible command conflicts
- Useful for feature flags, OAuth-gated commands, etc.

#### Argument Detection

`hasCommandArgs(input: string): boolean`
- Returns `false` if input has no args: "/" or "/commit "
- Returns `false` if trailing space: "/commit "
- Returns `true` if actual args: "/commit myfile"
- When args present, no suggestions shown (assumes user knows the command)

---

### 2. Shell History Completion (`shellHistoryCompletion.ts`)

#### Purpose

Provides ghost-text completions for shell commands by matching history.

#### Core Function

**`getShellHistoryCompletion(input: string): Promise<ShellHistoryMatch | null>`**

```typescript
export type ShellHistoryMatch = {
  fullCommand: string  // e.g., "ls -lah"
  suffix: string       // e.g., " -lah" (ghost text)
}
```

Algorithm:
1. Require `input.length >= 2` (avoid single-char matches)
2. Trim and validate input has content
3. Fetch commands from history (max 50 recent)
4. Find first command starting with exact input (including spaces)
5. Ensure it's not identical to input (avoid self-match)
6. Return { fullCommand, suffix }

**Key Detail**: Exact prefix match with space preservation
- "ls " matches "ls -lah" (space is significant)
- "ls  " (2 spaces) does NOT match "ls -lah" (exact space count required)

#### History Fetching & Caching

**`getShellHistoryCommands(): Promise<string[]>`**

1. Reads from `getHistory()` async iterator (from `history.js`)
2. Filters for entries with `display` starting with "!"
3. Removes "!" prefix to get actual command
4. Deduplicates via Set
5. Limits to 50 most recent unique commands
6. Caches for 60 seconds (TTL 60000ms)

**Cache invalidation**:
- `clearShellHistoryCache()`: explicit clear
- `prependToShellHistoryCache(command)`: adds new command to front (e.g., after user submits)
  - Dedupes: moves existing command to front if already cached
  - No-op if cache not yet populated

---

### 3. Directory Completion (`directoryCompletion.ts`)

#### Purpose

Provides path completion suggestions for directories and files.

#### Core Data Structures

```typescript
export type DirectoryEntry = {
  name: string
  path: string
  type: 'directory'
}

export type PathEntry = {
  name: string
  path: string
  type: 'directory' | 'file'
}
```

#### Main APIs

**`getDirectoryCompletions(partialPath: string, options?): Promise<SuggestionItem[]>`**
- Input: "src/u"
- Output: [{ id: "src/utils", displayText: "utils/", ... }, ...]
- Max 10 results by default
- Case-insensitive prefix matching

**`getPathCompletions(partialPath: string, options?): Promise<SuggestionItem[]>`**
- Like above but includes both files and directories
- Can filter by `includeFiles`, `includeHidden`
- Sorts directories first, then alphabetically
- Strips leading "./" from relative paths

**`parsePartialPath(partialPath: string, basePath?): ParsedPath`**
- Splits "src/comp" into { directory: "/abs/path/src", prefix: "comp" }
- Handles "~" expansion via `expandPath()`
- Treats trailing "/" as directory marker

#### Caching Strategy

Uses **LRU caches** (from `lru-cache` library):

```typescript
const directoryCache = new LRUCache<string, DirectoryEntry[]>({
  max: 500,           // Max 500 cached directories
  ttl: 5 * 60 * 1000  // 5-minute TTL
})

const pathCache = new LRUCache<string, PathEntry[]>({
  max: 500,
  ttl: 5 * 60 * 1000
})
```

**Cache key for path cache**: `"${dirPath}:${includeHidden}"`

#### Filesystem Integration

- Calls `getFsImplementation().readdir()` (async and sync variants)
- Filters on `entry.isDirectory()` and `entry.name.startsWith('.')`
- Limits results to 100 entries per directory
- Handles errors gracefully (returns [])

---

### 4. Skill Usage Tracking (`skillUsageTracking.ts`)

#### Purpose

Records and scores skill/command usage for ranking suggestions.

#### Core Functions

**`recordSkillUsage(skillName: string): void`**
- Called when user invokes a skill/command
- Updates global config with:
  - `usageCount++`
  - `lastUsedAt: Date.now()`
- **Debounced**: only writes to config if last write > 60 seconds ago
  - Process-lifetime debounce cache via `lastWriteBySkill: Map`
  - Rationale: 7-day half-life ranking algorithm doesn't need sub-minute granularity

**`getSkillUsageScore(skillName: string): number`**
- Reads from config `skillUsage[skillName]`
- Calculates exponential decay score:
  ```
  recencyFactor = 0.5 ^ (daysSinceUse / 7)
  score = usageCount * Math.max(recencyFactor, 0.1)
  ```
- Returns 0 if no usage data

#### Integration Points

- Called from `generateCommandSuggestions()` to rank skills in empty-query mode
- Called from ranking comparator to break ties in fuzzy search results
- Config storage: `getGlobalConfig()` and `saveGlobalConfig()`

#### Data Structure

```typescript
// In global config:
skillUsage: {
  [skillName: string]: {
    usageCount: number
    lastUsedAt: number  // timestamp ms
  }
}
```

---

### 5. Slack Channel Suggestions (`slackChannelSuggestions.ts`)

#### Purpose

Provides autocomplete suggestions for Slack channel names via MCP (Model Context Protocol) server.

#### High-Level Flow

```
User types "#cl"
    ↓
searchToken = "cl"
    ↓
[MCP Query Optimization]
    ├─ Strip trailing partial segment: "cl" → "cl" (no hyphens)
    └─ Check cache for prefix match
    ↓
[Slack MCP Server]
    └─ slack_search_channels(query="cl", limit=20, channel_types="public,private")
    ↓
[Response Processing]
    ├─ Unwrap JSON envelope: { results: "# Search Results\nName: #channel\n..." }
    ├─ Parse markdown for "Name: #channel-name" lines
    ├─ Deduplicate and cache
    └─ Filter locally for exact prefix match
    ↓
[SuggestionItem[]] → Highlighted in UI
```

#### Slack MCP Integration

**`findSlackClient(clients: MCPServerConnection[]): MCPServerConnection | undefined`**
- Searches for connected MCP server with name containing "slack"
- Prerequisite check: `hasSlackMcpServer(clients: MCPServerConnection[])`

**`fetchChannels(clients, query): Promise<string[]>`**
- Calls `slackClient.client.callTool()`
- Tool: `'slack_search_channels'`
- Arguments:
  ```typescript
  {
    query: string          // e.g., "claude-code-team"
    limit: 20              // Max results
    channel_types: string  // "public_channel,private_channel"
  }
  ```
- Timeout: 5 seconds
- Returns array of channel names (strings)

#### Response Parsing

**`unwrapResults(text: string): string`**
- Slack MCP wraps JSON: `{ "results": "# Search Results\n..." }`
- Detects and unwraps envelope, falls back to raw text if not JSON

**`parseChannels(text: string): string[]`**
- Regex: `/^Name:\s*#?([a-z0-9][a-z0-9_-]{0,79})\s*$/` per line
- Extracts channel names, deduplicates via Set
- Channel name rules: start with alphanumeric, 0-79 chars, lowercase + digits/hyphens/underscores

#### Caching Strategy

**Search-result cache** (Plain Map, not LRU):
```typescript
cache: Map<string, string[]>  // key = mcpQuery, value = [channel names]
```

Rationale: Needs to iterate all entries for prefix-match reuse (LRUCache doesn't expose iteration).

**Known-channels tracking** (Set):
```typescript
knownChannels: Set<string>  // All channels ever returned by MCP
knownChannelsVersion: number  // Incremented on new channels
```

Rationale: UI only highlights `#channel` in text if channel is in `knownChannels` (confirmed by MCP).

**Query Optimization**:

Via `mcpQueryFor(searchToken: string)`:
- Strips trailing partial segment: "claude-code-team-en" → "claude-code-team"
- Reason: Slack tokenizes on hyphens; partial words kill search
- Locally filters results to match full searchToken

Via `findReusableCacheEntry(mcpQuery, searchToken)`:
- Finds longest cached entry whose key is a prefix of mcpQuery
- Reuses cached results if they still match searchToken
- Avoids new MCP call when typing "c" → "cl" → "cla" (reuses "c" results if possible)

#### In-Flight Request Deduplication

```typescript
let inflightQuery: string | null = null
let inflightPromise: Promise<string[]> | null = null
```

When TTL expires and new query issued:
- If same query already in-flight, await existing promise (don't spawn duplicate)
- Once resolved, update cache and reset in-flight state
- Limits concurrent MCP requests to 1

#### Known Channels Highlighting

**`findSlackChannelPositions(text: string): Array<{ start, end }>`**
- Regex: `/(^|\s)#([a-z0-9][a-z0-9_-]{0,79})(?=\s|$)/g`
- Only returns positions for channels in `knownChannels` Set
- Used by UI to apply syntax highlighting (e.g., blue color for #channels)

**`subscribeKnownChannels`**
- Signal/event for UI to re-check highlighting when new channels discovered

#### Error Handling & Limits

- MCP timeout: 5 seconds
- Search results: cap at 20 from MCP, filter to 10 for UI display
- Cache size: max 50 entries, LRU-evict oldest on overflow
- Graceful fallback: returns [] if no Slack MCP or connection fails

---

## Part 2: Secure Storage System

### Overview

The secure storage system abstracts credential management across platforms (macOS Keychain, Windows Credential Manager, Linux libsecret fallback). It provides symmetric read/write operations with platform-specific backends, fallback mechanisms, and prefetching optimizations.

### Architecture: Platform Abstraction

```
getSecureStorage()
    ↓
[Platform Detection]
    ├─ macOS (darwin) → Keychain + fallback to plaintext
    ├─ Linux → plaintext (libsecret TODO)
    └─ Windows → plaintext (Credential Manager TODO)
    ↓
[SecureStorage Interface]
    ├─ read(): SecureStorageData | null
    ├─ readAsync(): Promise<SecureStorageData | null>
    ├─ update(data): { success, warning? }
    └─ delete(): boolean
    ↓
[Backend Implementation]
    └─ Actual I/O (spawn security, read .credentials.json, etc.)
```

### 1. Index & Platform Selection (`index.ts`)

**`getSecureStorage(): SecureStorage`**

```typescript
export function getSecureStorage(): SecureStorage {
  if (process.platform === 'darwin') {
    return createFallbackStorage(macOsKeychainStorage, plainTextStorage)
  }
  // TODO: add libsecret support for Linux
  return plainTextStorage
}
```

**Current Implementation**:
- **macOS**: Keychain (primary) + plaintext fallback
- **Linux**: plaintext only (libsecret support planned)
- **Windows**: plaintext only (Credential Manager support planned)

### 2. macOS Keychain Storage (`macOsKeychainStorage.ts`)

#### Core Implementation

The keychain storage uses native `security` CLI tool to interact with macOS Keychain.

**Service Name Format**:
```
"Claude Code${OAUTH_FILE_SUFFIX}${serviceSuffix}${dirHash}"
```

Examples:
- OAuth credentials: `"Claude Code-credentials"` (default dir)
- Legacy API key: `"Claude Code"` (no suffix)
- Non-default config dir: `"Claude Code-xyz12345"` (dir hash suffix)

**Username**: `process.env.USER` or `os.userInfo().username`, fallback: `"claude-code-user"`

#### Read Operations

**Synchronous: `read(): SecureStorageData | null`**

```typescript
1. Check cache: if within 30s TTL, return cached data
2. Spawn: security find-generic-password -a "username" -w -s "serviceName"
3. Parse: JSON parse the output
4. Cache: store with Date.now() timestamp
5. Return: data or null if not found
```

**Cache Strategy**:
- TTL: 30 seconds (cross-process staleness acceptable for OAuth tokens)
- Reason for long TTL: Sync spawn is ~500ms; with 50+ MCP connectors at startup, short TTL causes 5.5s event-loop stalls
- Stale-while-error: if refresh fails but cached data exists, serve stale rather than returning null (prevents "Not logged in" cascade)
- Cache invalidation: `clearKeychainCache()` sets `cachedAt=0` and increments `generation`

**Asynchronous: `readAsync(): Promise<SecureStorageData | null>`**

```typescript
1. Check cache (same as sync)
2. Check if read already in-flight
   - If yes, return existing promise (dedup concurrent requests)
   - If no, spawn async security command
3. On completion, capture generation number
   - If generation changed (another write happened), discard result
   - Otherwise, update cache
4. Return promise
```

**Deduplication**: If second caller awaits while first's promise in-flight, both await same promise. Avoids multiple concurrent spawns.

#### Write Operations

**`update(data: SecureStorageData): { success, warning? }`**

```typescript
1. Invalidate cache (clearKeychainCache)
2. Serialize: JSON stringify the data object
3. Hex-encode: Convert UTF-8 bytes to hex string
   - Reason: Avoids shell escaping issues, defeats plaintext-grep rules for monitors
4. Build command: add-generic-password -U -a "user" -s "service" -X "hexValue"
5. Check stdin line limit (4096 - 64 bytes)
   - If fits: spawn with stdin input (process monitors see only "security -i", not payload)
   - If exceeds: spawn with argv (fallback, less secure but prevents silent corruption)
6. Parse exit code; update cache on success
7. Return { success, warning? }
```

**Stdin vs Argv Decision**:
- Prefer stdin via `security -i` (command line args visible to process monitors like CrowdStrike)
- Fallback to argv if payload exceeds 4032-byte line buffer (4096 - 64 headroom)
- Rationale: Silent credential corruption from truncation is worse than argv exposure

**Cache Update**:
- Success: cache is invalidated before write, then updated on success
- Failure: cache remains stale (next read will retry)

#### Delete Operations

**`delete(): boolean`**

```typescript
1. Invalidate cache
2. Spawn: security delete-generic-password -a "user" -s "service"
3. Return: success if exit code 0
```

#### Keychain Lock Detection

**`isMacOsKeychainLocked(): boolean`**

```typescript
1. Check cache (process lifetime)
2. If cached, return it
3. Spawn: security show-keychain-info
4. Check exit code: 36 = locked, otherwise = unlocked
5. Cache result (immutable during session)
6. Return boolean
```

Used to display "keychain locked" messages in SSH sessions where keychain isn't auto-unlocked.

#### Error Handling

- `execSyncWithDefaults_DEPRECATED()` for sync reads
- `execFileNoThrow()` for async reads (captures exit code, no throw)
- `execaSync()` with `reject: false` for write/delete
- Errors logged via `logForDebugging()`
- Stale-while-error for reads (serve stale if refresh fails)

### 3. macOS Keychain Helpers (`macOsKeychainHelpers.ts`)

Shared utilities for keychain operations, designed to minimize module-init cost for `keychainPrefetch.ts`.

#### Service Name Construction

**`getMacOsKeychainStorageServiceName(serviceSuffix?): string`**

```typescript
const configDir = getClaudeConfigHomeDir()
const isDefaultDir = !process.env.CLAUDE_CONFIG_DIR
const dirHash = isDefaultDir ? ''
  : `-${createHash('sha256').update(configDir).digest('hex').substring(0, 8)}`
return `Claude Code${getOauthConfig().OAUTH_FILE_SUFFIX}${serviceSuffix}${dirHash}`
```

Logic:
- Default config dir: `~/.claude` → no suffix
- Custom config dir: `/path/to/config` → append 8-char SHA256 hash
- Supports multiple concurrent Claude Code instances with different config dirs

#### Credentials Service Suffix

```typescript
export const CREDENTIALS_SERVICE_SUFFIX = '-credentials'
```

- OAuth credentials stored under `"Claude Code-credentials"`
- Legacy API key under `"Claude Code"` (no suffix)
- **Critical**: Never change this constant—keychain lookup key would orphan existing entries

#### Username Helpers

**`getUsername(): string`**
- Returns `process.env.USER` if set
- Falls back to `os.userInfo().username`
- Final fallback: `"claude-code-user"`

#### Cache State Management

**`keychainCacheState` object**:
```typescript
{
  cache: { data: SecureStorageData | null; cachedAt: number }
  generation: number  // Incremented on invalidation
  readInFlight: Promise<...> | null
}
```

**TTL**: `KEYCHAIN_CACHE_TTL_MS = 30_000` (30 seconds)

**Cache Invalidation**: `clearKeychainCache()`
- Sets `cachedAt = 0` (marks invalid)
- Increments `generation` (causes in-flight async reads to discard results)
- Clears `readInFlight`

**Prefetch Priming**: `primeKeychainCacheFromPrefetch(stdout)`
- Called by `keychainPrefetch.ts` after spawn completes
- Only writes if cache not yet touched (`cachedAt === 0`)
- Allows prefetch to prime cache without overwriting auth-driven updates

#### Module-Init Cost Optimization

**Why this file is separate from macOsKeychainStorage.ts**:
- keychainPrefetch.ts imports only this file (not the main storage file)
- Avoids pulling in execa → human-signals → cross-spawn (~58ms sync init)
- Imports here (crypto, os, envUtils, oauth constants) already evaluated by startupProfiler.ts
- Net result: keychainPrefetch can fire early with minimal overhead

### 4. Keychain Prefetch (`keychainPrefetch.ts`)

#### Purpose

Parallelize macOS keychain reads with main.tsx module evaluation to avoid startup blocking.

#### Sequential vs Parallel Cost

**Sequential (old)**:
1. Read "Claude Code-credentials" (OAuth) → ~32ms
2. Read "Claude Code" (legacy API key) → ~33ms
3. Continue startup
- Total blocking: ~65ms

**Parallel (new)**:
1. Fire both spawns at startup (via `startKeychainPrefetch()`)
2. Subprocesses run in parallel with ~65ms of main.tsx imports
3. Await both via `ensureKeychainPrefetchCompleted()` in preAction
- Total blocking: ~0ms (mostly free)

#### API

**`startKeychainPrefetch(): void`**
- Called at main.tsx line 1 (top-level)
- Non-darwin: no-op
- Bare mode: no-op
- Spawns two `security find-generic-password` processes (non-blocking)

**`ensureKeychainPrefetchCompleted(): Promise<void>`**
- Called in main.tsx preAction alongside `ensureMdmSettingsLoaded()`
- Awaits prefetch promise (already running, mostly finished by then)
- Returns immediately on non-darwin

**`getLegacyApiKeyPrefetchResult(): { stdout: string | null } | null`**
- Consumed by `getApiKeyFromConfigOrMacOSKeychain()` in auth.ts
- Returns cached prefetch result or null if not yet completed
- Avoids sync spawn if prefetch already finished

#### Implementation Details

**`spawnSecurity(serviceName): Promise<SpawnResult>`**

```typescript
execFile('security',
  ['find-generic-password', '-a', username, '-w', '-s', serviceName],
  { encoding: 'utf-8', timeout: 10000 },
  (err, stdout) => {
    // Exit 44 (not found) = valid "no key" → prime as null
    // Timeout (err.killed) = maybe has key → don't prime, retry in sync path
    resolve({ stdout: err ? null : stdout?.trim(), timedOut: err?.killed })
  }
)
```

**Timeout**: 10 seconds
- If prefetch exceeds this, marks `timedOut = true`
- Sync read later will retry (doesn't prime cache with stale null)

**Timeout vs Error**:
- Exit code 44 (key not found): `err` is set but not killed → prime as null
- Actual timeout: `err.killed = true` → don't prime, retry in sync

#### Data Flow

```
main.tsx top-level
  ↓
startKeychainPrefetch()
  ├─ Spawn oauth read (to keychain)
  ├─ Spawn legacy read (to keychain)
  └─ Store promises (don't await)
  ↓
main.tsx imports run (~65ms)
  ↓
preAction awaits ensureKeychainPrefetchCompleted()
  └─ Likely finished by now
  ↓
macOsKeychainStorage.read() later
  └─ Hits prefetch-primed cache instead of spawning
```

---

### 5. Plain Text Fallback Storage (`plainTextStorage.ts`)

#### Purpose

Fallback credential storage when Keychain unavailable (Linux, Windows, Keychain locked). Also used as secondary in macOS fallback chain.

#### File Location & Naming

```typescript
storageDir = getClaudeConfigHomeDir()  // ~/.claude or CLAUDE_CONFIG_DIR
storageFileName = '.credentials.json'
storagePath = ~/.claude/.credentials.json
```

#### Read Operations

**Synchronous: `read(): SecureStorageData | null`**

```typescript
try {
  const data = getFsImplementation().readFileSync(storagePath, 'utf8')
  return jsonParse(data)
} catch {
  return null  // File not found or invalid JSON
}
```

**Asynchronous: `readAsync(): Promise<SecureStorageData | null>`**

```typescript
try {
  const data = await getFsImplementation().readFile(storagePath, 'utf8')
  return jsonParse(data)
} catch {
  return null
}
```

#### Write Operations

**`update(data: SecureStorageData): { success, warning }`**

```typescript
1. Ensure directory exists: mkdirSync(storageDir)
   - Catch EEXIST (expected if already exists)
2. Write file: writeFileSync_DEPRECATED(storagePath, jsonStringify(data))
3. Set permissions: chmodSync(storagePath, 0o600) [read/write for owner only]
4. Return { success: true, warning: "Warning: Storing credentials in plaintext." }
```

**Permissions**: `0o600` (rw-------)
- Owner read/write only
- No group or other access
- Mitigates exposure from accidental directory listing

#### Delete Operations

**`delete(): boolean`**

```typescript
try {
  getFsImplementation().unlinkSync(storagePath)
  return true
} catch (e) {
  if (getErrnoCode(e) === 'ENOENT') return true  // Already deleted
  return false
}
```

**Idempotent**: Treats "not found" as success.

#### Warning Message

Every write returns:
```typescript
{
  success: true,
  warning: "Warning: Storing credentials in plaintext."
}
```

UI displays this to user so they understand credentials are unencrypted on disk.

#### No Caching

Unlike Keychain storage, plaintext has no caching layer:
- Reads hit disk every time
- Slower but simpler (no cache invalidation complexity)
- Used as fallback only; not performance-critical

---

### 6. Fallback Storage Wrapper (`fallbackStorage.ts`)

#### Purpose

Provides two-tier failover: try primary storage, fall back to secondary on failure.

#### Implementation

**`createFallbackStorage(primary, secondary): SecureStorage`**

Returns a SecureStorage that wraps both backends.

**`read()`**:
```typescript
const result = primary.read()
if (result !== null && result !== undefined) return result
return secondary.read() || {}  // Return {} if both null
```

**`readAsync()`**:
```typescript
const result = await primary.readAsync()
if (result !== null && result !== undefined) return result
return (await secondary.readAsync()) || {}
```

**`update(data)`**:
```typescript
// First, capture current primary state
const primaryDataBefore = primary.read()

// Try primary update
const result = primary.update(data)
if (result.success) {
  // Migrate: if primary was previously empty, delete secondary
  // (avoid stale data shadowing new primary write)
  if (primaryDataBefore === null) {
    secondary.delete()  // Best-effort; ignore failure
  }
  return result
}

// Primary failed; try secondary
const fallbackResult = secondary.update(data)
if (fallbackResult.success) {
  // Secondary succeeded but primary failed
  // Delete stale primary entry if it exists
  // (otherwise read() would return stale data, shadowing fresh secondary write)
  if (primaryDataBefore !== null) {
    primary.delete()  // Best-effort
  }
  return {
    success: true,
    warning: fallbackResult.warning
  }
}

// Both failed
return { success: false }
```

**Key Logic**:
1. Prefer primary when both available
2. On primary write success + secondary has data: delete secondary (migrate to primary)
3. On primary write failure + secondary succeeds + primary has stale data: delete primary (avoid shadowing)
4. Prevents stale data from masking fresh writes

**`delete()`**:
```typescript
const primarySuccess = primary.delete()
const secondarySuccess = secondary.delete()
return primarySuccess || secondarySuccess  // Success if at least one succeeds
```

#### Real-World Scenario: macOS Keychain + Plaintext

**Scenario 1: User is logged in, OAuth token in Keychain**
- `read()` hits Keychain, returns OAuth token
- `read()` is fast (~sync spawn)
- Never touches plaintext file

**Scenario 2: Keychain locked (SSH session)**
- Keychain spawn times out or returns error
- Fallback reads plaintext `.credentials.json`
- User can continue working, plaintext creds serve as emergency backup

**Scenario 3: User logs in, writes OAuth token**
- `update()` tries Keychain (succeeds)
- Deletes old plaintext file (if existed)
- Future `read()` uses Keychain only

**Scenario 4: Keychain write fails, plaintext succeeds**
- `update()` Keychain fails (e.g., permission denied)
- Fallback to plaintext (succeeds)
- Delete Keychain's old entry to avoid stale shadowing
- Returns warning: "Storing credentials in plaintext."

---

## Part 3: Integration & Cross-Cutting Concerns

### Unified Suggestion System Integration

All suggestion engines feed the same UI component:

```typescript
// PromptInputFooterSuggestions.tsx consumes SuggestionItem[]
interface SuggestionItem {
  id: string            // Unique across all suggestion types
  displayText: string   // What to show user
  description?: string  // Help text
  metadata?: any        // Type-specific data (Command, PathEntry, etc.)
  tag?: string          // Visual indicator (e.g., "workflow")
}
```

Generators unified through this interface:
- Commands: `generateCommandSuggestions()` → SuggestionItem[]
- Shell history: `getShellHistoryCompletion()` → single match
- Directories: `getDirectoryCompletions()` → SuggestionItem[]
- Slack: `getSlackChannelSuggestions()` → SuggestionItem[]

Selection handled via `applyCommandSuggestion()`, path completion, etc.

### Secure Storage Integration Points

1. **OAuth Token Storage**
   - `/login` command stores OAuth tokens via `update()`
   - Session refresh reads tokens via `read()` / `readAsync()`
   - Logout deletes via `delete()`

2. **API Key Storage (Legacy)**
   - Fallback for users without OAuth
   - Stored under "Claude Code" (no "-credentials" suffix)
   - Prefetched at startup via `keychainPrefetch.ts`

3. **MCP Server Credentials**
   - Some connectors store credentials in secure storage
   - E.g., Slack MCP server OAuth token

### Performance Optimizations

#### Suggestions
- Fuse index cached by array identity (single cache entry)
- Command database pre-sorted in categories (avoiding sort per query)
- Usage scoring computed once per result in comparator
- Shell history cached 60s (avoids repeated file I/O)
- Directory entries cached 5 minutes (avoids repeated scandir)
- Slack channels cached with prefix-reuse heuristic

#### Secure Storage
- Keychain reads cached 30s (avoids spawn storm at startup)
- Async reads deduplicated (concurrent callers share promise)
- Prefetch paralelizes with module loading (hidden cost)
- Stale-while-error prevents cascading "Not logged in" messages

### Error Handling Philosophy

#### Suggestions
- Partial failures (one suggestion type fails): return [] or partial results
- Graceful degradation (shell history unavailable): skip shell suggestions
- Silent errors logged to debug console only

#### Secure Storage
- Read failures: stale-while-error (serve cached data if available)
- Write failures: fallback to secondary storage
- Delete failures: log and continue (non-blocking)
- Keychain locked: plaintext fallback seamless to user

### Typing & Data Flow

#### Suggestions
```
Raw user input string
  → parsePartialPath() / cleanWord() / findMidInputSlashCommand()
  → SuggestionItem[]
  → UI renders displayText, hooks metadata for selection
  → applyCommandSuggestion() / onDirectorySelect() / etc.
  → Final action
```

#### Secure Storage
```
Credential object (OAuth token, API key)
  → jsonStringify()
  → Hex-encode (Keychain) / write to disk (plaintext)
  ↓ (on read)
  → Read from Keychain / plaintext
  → Hex-decode (Keychain) / JSON parse
  → Credential object
```

---

## Part 4: Data Structures & Type Details

### Suggestion System Types

**SecureStorageData** (inferred from usage):
```typescript
{
  // OAuth credentials
  accessToken?: string
  refreshToken?: string
  expiresAt?: number

  // Legacy API key
  apiKey?: string

  // MCP server credentials (optional)
  [key: string]: any
}
```

**Command** (from commands.ts):
```typescript
{
  name: string
  type: 'local' | 'local-jsx' | 'prompt'
  description?: string
  aliases?: string[]
  isHidden?: boolean
  source?: 'userSettings' | 'localSettings' | 'projectSettings' | 'policySettings' | 'plugin'
  kind?: 'workflow'
  pluginInfo?: { repository: string, ... }
  argNames?: string[]
}
```

**SuggestionItem**:
```typescript
{
  id: string            // Unique ID for deduplication
  displayText: string   // User-facing text
  tag?: string          // Tag like "workflow"
  description?: string  // Help text
  metadata?: unknown    // Type-specific (Command, DirectoryEntry, etc.)
}
```

### Secure Storage Types

**SecureStorage** (interface satisfied by all backends):
```typescript
{
  name: string  // "keychain", "plaintext", "keychain-with-plaintext-fallback"
  read(): SecureStorageData | null
  readAsync(): Promise<SecureStorageData | null>
  update(data: SecureStorageData): { success: boolean; warning?: string }
  delete(): boolean
}
```

**SecureStorageData**:
```typescript
Record<string, any>  // Flexible structure for OAuth, API keys, MCP creds
```

---

## Part 5: Platform-Specific Behavior

### macOS (darwin)

**Keychain backend**:
- Encrypted by OS keychain
- Service names: "Claude Code-credentials", "Claude Code"
- Handles multi-instance via config dir hash
- Supports cache prefetch at startup
- Detects lock state (SSH sessions)

**Fallback to plaintext**:
- `~/.claude/.credentials.json` with 0o600 perms
- Used if Keychain locked, errors, or unavailable
- Migration on write: deletes plaintext once Keychain succeeds

**Keychain prefetch**:
- Two parallel spawns on startup
- Primes cache before main module evaluation
- Avoids 65ms startup blocking

### Linux

**Current**: plaintext only
- `~/.claude/.credentials.json` with 0o600 perms

**Planned**: libsecret support
- TODO in `index.ts`

### Windows

**Current**: plaintext only
- `%APPDATA%\.claude\.credentials.json` (via `getClaudeConfigHomeDir()`)

**Planned**: Credential Manager support
- TODO in `index.ts`

---

## Part 6: Key Design Decisions & Rationales

### Why Fuse.js for Commands?

- Fuzzy matching beats exact prefix matching (user may not remember exact spelling)
- Weighted fields allow ranking command names > aliases > descriptions
- Score-based sorting integrates with usage scoring
- Threshold of 0.3 prevents too-loose matches

### Why Exponential Decay for Usage Scoring?

- Half-life model (7 days) matches human recency bias
- 0.1 floor prevents old heavily-used skills from disappearing
- Debounce at 60s avoids sub-minute granularity (7-day half-life doesn't need it)

### Why Separate Module for keychainPrefetch?

- Avoids importing execa bundle (~58ms module init)
- Allows parallel spawn with module evaluation
- Minimal import chain (crypto, os, envUtils already loaded)

### Why 30s Keychain Cache TTL?

- OAuth tokens last hours; 30s staleness acceptable
- Avoids 500ms spawn per read (5.5s stall with 50+ connectors)
- Stale-while-error handles transient failures gracefully

### Why Hex-Encode in Keychain?

- Avoids shell escaping complexity
- Defeats naive plaintext-grep rules (process monitors)
- Fallback to argv if > 4KB (avoids silent truncation)

### Why Fallback Storage?

- Keychain can fail (locked, permission denied)
- Graceful degradation > complete auth failure
- Enables Docker/container sharing (separate volumes)
- Migration path (primary → secondary or vice versa)

### Why Cache by Query Prefix (Slack)?

- Typing "c" → "cl" → "cla" reuses cache (no new MCP call)
- MCP limit is 20; prefix filtering is local
- Avoids redundant server queries during typing

### Why 5-Minute Directory Cache TTL?

- Filesystem rarely changes during single session
- Avoids repeated scandir for frequently-accessed paths
- 500-entry LRU covers typical project structures

---

## Summary: Anthropic's Engineering Philosophy

### Suggestions System

1. **Context-aware ranking**: Usage, fuzzy match score, prefix preference
2. **Efficient caching**: Fuse index by reference, shell history 60s, directory entries 5m
3. **Graceful degradation**: Partial failures don't break UI
4. **Performance focus**: Avoid sorting per keystroke, deduplicate concurrent queries

### Secure Storage System

1. **Platform abstraction**: Single interface, multiple backends
2. **Secure by default**: Keychain on macOS, plaintext with 0o600 perms
3. **Reliability over security**: Fallback > data loss, stale-while-error > blocking
4. **Startup optimization**: Prefetch keychain in parallel, avoid blocking
5. **Multi-instance support**: Config dir hash prevents credential collision

### Overall Philosophy

- **Lazy evaluation**: Cache by identity, avoid unnecessary computation
- **Parallel execution**: Prefetch, async/await, non-blocking operations
- **Graceful failures**: Fallback mechanisms, stale-while-error, best-effort deletes
- **User transparency**: Warnings about plaintext storage, skip messages on hidden commands
- **Integration**: Both systems compose with command execution, authentication, plugins

---

## Files Analyzed

**Suggestions System** (5 files, ~1,213 lines):
1. `/src/utils/suggestions/commandSuggestions.ts` (568 lines)
2. `/src/utils/suggestions/skillUsageTracking.ts` (56 lines)
3. `/src/utils/suggestions/shellHistoryCompletion.ts` (120 lines)
4. `/src/utils/suggestions/directoryCompletion.ts` (264 lines)
5. `/src/utils/suggestions/slackChannelSuggestions.ts` (210 lines)

**Secure Storage System** (6 files, ~629 lines):
1. `/src/utils/secureStorage/index.ts` (18 lines)
2. `/src/utils/secureStorage/macOsKeychainStorage.ts` (232 lines)
3. `/src/utils/secureStorage/macOsKeychainHelpers.ts` (112 lines)
4. `/src/utils/secureStorage/keychainPrefetch.ts` (117 lines)
5. `/src/utils/secureStorage/plainTextStorage.ts` (84 lines)
6. `/src/utils/secureStorage/fallbackStorage.ts` (71 lines)

---

**End of Analysis Document**
