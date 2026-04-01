# Claude Code v2.1.88 Plugin System: Deep-Dive Architecture Analysis

## Executive Summary

Claude Code's plugin system is a sophisticated infrastructure (20,521 lines across 44 files) that manages **discovery, validation, installation, execution, and updates** of plugins from distributed sources. The architecture separates concerns across **three data layers** (disk state, intent/settings, session memory), implements **five-scope priority merging** for per-repo control, and provides **enterprise-grade security** through manifest validation, path traversal prevention, homograph detection, and policy enforcement.

The system is engineered for **offline-first operation** with ZIP caching, supports **rich plugin composition** (commands, agents, skills, output styles, hooks, MCP servers), and includes **dependency resolution** via DFS with cycle detection and cross-marketplace boundary enforcement.

---

## Part 1: Plugin Lifecycle

### 1.1 Plugin Discovery Pipeline

Plugins originate from three sources:

1. **Marketplace-based plugins** (`plugin@marketplace` format in settings)
   - Registered in `known_marketplaces.json`
   - Downloaded/cloned into versioned cache: `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`
   - Loaded via `pluginLoader.ts` main pipeline

2. **Session-only plugins** (`--plugin-dir` CLI flag)
   - Loaded directly from user-provided directories
   - Register as `name@inline` (synthetic marketplace sentinel)
   - Bare dependencies resolved within same inline marketplace only

3. **Built-in plugins**
   - Hardcoded in `builtinPlugins.ts`
   - Marketplace name: `builtin` (reserved)
   - No external dependencies

**Discovery Order (precedence):**
```
Built-in plugins
  ↓
Inline plugins (--plugin-dir)
  ↓
Marketplace plugins (settings.enabledPlugins)
  ↓
Official marketplace (implicit fallback if any @claude-code-marketplace plugin enabled)
```

### 1.2 Plugin Source Types

The `PluginSource` type supports six installation methods:

```typescript
type PluginSource =
  | string                              // Local path
  | { source: 'npm', package, version?, registry? }
  | { source: 'github', repo, ref?, sha? }
  | { source: 'url', url, ref?, sha? } // Generic git URL
  | { source: 'git-subdir', url, path, ref?, sha? } // Sparse clone
  | { source: 'pip', ... }              // Planned, not implemented
```

**Key behaviors:**
- **Git cloning**: Uses `--depth 1 --recurse-submodules --shallow-submodules` for efficiency
- **Specific commits**: Falls back from shallow fetch → unshallow fetch if needed
- **Git subdir**: Partial clone via `--filter=tree:0` + `sparse-checkout --cone` for monorepos
- **npm packages**: Cached globally at `~/.claude/plugins/npm-cache/`, copied to destination
- **All sources**: `.git` directory removed from cache after installation

### 1.3 Plugin Validation & Manifest Loading

**Path:** `pluginLoader.ts:1348-1769` (`createPluginFromPath`)

Plugins are assembled into `LoadedPlugin` objects:

1. **Manifest loading** (from `.claude-plugin/plugin.json` or `plugin.json`)
   - Schema: `PluginManifestSchema()` (Zod validated)
   - Unknown keys stripped silently (lenient)
   - Fallback to default manifest if missing: `{ name, description: "Plugin from {source}" }`

2. **Component detection** (parallel validation):
   - **Commands**: `commands/` dir OR manifest `commands` field (supports inline content)
   - **Agents**: `agents/` dir OR manifest `agents` field
   - **Skills**: `skills/` dir OR manifest `skills` field
   - **Output styles**: `output-styles/` dir OR manifest `outputStyles` field
   - **Hooks**: `hooks/hooks.json` + manifest `hooks` array (merged, duplicate detection)

3. **Command metadata** (object-mapping format):
   ```json
   {
     "commands": {
       "build": { "source": "./build.md", "description": "Build project" },
       "inline": { "content": "# Inline command markdown" }
     }
   }
   ```
   - Supports both `source` (file path) and `content` (inline markdown)
   - Cannot have both simultaneously (schema enforced)

4. **Settings loading** (`.json` in plugin dir OR manifest.settings):
   - Filtered to allowlisted keys (`agent` only)
   - Non-sensitive plaintext config

### 1.4 Installation Flow

**Entry point:** `cachePlugin()` → cache-path computation → source installation

```
┌─ Local path ──→ copyDir()
├─ npm ────────→ npm install --prefix npm-cache → copyDir()
├─ GitHub ─────→ validateGitUrl() → gitClone()
├─ git URL ────→ validateGitUrl() → gitClone()
├─ git-subdir ─→ validateGitUrl() → sparse clone + subdir extraction
└─ pip ────────→ throw Error ("not yet supported")
```

**Cache organization:**
- **Versioned cache**: `~/.claude/plugins/cache/{marketplace}/{plugin}/{version}/`
- **Fallback to legacy**: `~/.claude/plugins/cache/{plugin-name}/` (backward compatibility)
- **ZIP mode**: Same paths but `.zip` extension; extracted to session temp on load
- **Seed cache probing**: Check configured seed directories first (read-only, no copy)

**Post-installation:**
- Parse manifest from `.claude-plugin/plugin.json` or `plugin.json`
- Create `LoadedPlugin` object with path/source/manifest
- Store in `installed_plugins.json` for future reference

### 1.5 Enable/Disable State Management

**State sources (five-layer scope merge):**
1. **Flag scope** (session-only, highest priority): `--plugin-disable`/`--plugin-enable` CLI flags
2. **Local scope** (project): `.claude/.claude/settings.json`
3. **Project scope**: `.claude/settings.json`
4. **User scope**: `~/.claude/settings.json`
5. **Managed scope** (lowest): Built-in defaults for managed plugins

**Schema (`PluginScope`):**
```typescript
type PluginScope = 'managed' | 'user' | 'project' | 'local' | 'flag'
```

**Settings format:**
```json
{
  "enabledPlugins": {
    "plugin-name@marketplace": true,
    "disabled@marketplace": false,
    "versioned@mkt": ["^1.0.0"]  // Version constraints
  }
}
```

**Merge algorithm** (`installedPluginsManager.ts`):
- Higher scopes override lower ones
- Explicit `false` disables regardless of lower scopes
- Default for unspecified plugins: disabled

### 1.6 Execution & Loading

**Load pipeline** (`pluginLoader.ts:1888-2000+`):

1. **Marketplace plugin discovery**
   - Read settings `enabledPlugins` entries in `plugin@marketplace` format
   - Filter to non-builtin entries
   - Load `known_marketplaces.json` once (not per-plugin)

2. **Enterprise policy checks** (optional, fail-closed):
   - Strict allowlist: plugin must be from allowed marketplace
   - Blocklist: plugin source must not be in denied list
   - Missing source + active policy → blocked

3. **Parallel marketplace loading**
   - Pre-load all marketplace catalogs (cache-only, no network)
   - Load all enabled plugins in parallel

4. **Component loading** (see Part 3)
   - Parse command markdown → namespaced commands
   - Load agents/skills
   - Load hooks with merge logic
   - Load MCP server configs

---

## Part 2: Marketplace Architecture

### 2.1 Known Marketplaces Configuration

**File:** `~/.claude/plugins/known_marketplaces.json`

**Schema:**
```typescript
type KnownMarketplacesFile = Record<string, KnownMarketplace>

type KnownMarketplace = {
  source: MarketplaceSource    // URL, GitHub, git, or local
  installLocation?: string      // Where marketplace is cached
  lastUpdated?: string         // ISO timestamp
  autoUpdate?: boolean         // Background refresh enabled
}

type MarketplaceSource =
  | { source: 'url', url: string }
  | { source: 'github', repo: string, ref?: string }
  | { source: 'git', url: string, ref?: string }
  | { source: 'local', path: string }
```

### 2.2 Marketplace Resolution

**Process:**
1. **Settings layer** (`getDeclaredMarketplaces()`)
   - Reads `extraKnownMarketplaces` from settings
   - Implicit official marketplace if any `@claude-code-marketplace` plugin enabled

2. **Reconciliation** (`reconciler.ts`)
   - Compares declared vs. actual (on-disk in `known_marketplaces.json`)
   - Detects: missing, changed source, version mismatch
   - Triggers download/clone/pull as needed

3. **Caching**
   - **URL source**: Downloaded as `marketplace.json` to `~/.claude/plugins/marketplaces/{name}.json`
   - **GitHub/git source**: Cloned to `~/.claude/plugins/marketplaces/{name}/` with `.git`
   - **Local source**: Symlinked or referenced directly
   - In-memory cache via memoization after first load

### 2.3 Marketplace Entry Format

**File:** `marketplace.json` (at marketplace root or `.claude-plugin/marketplace.json`)

```typescript
type PluginMarketplace = {
  version?: string              // Marketplace version
  name: string                 // Marketplace name
  plugins: PluginMarketplaceEntry[]
}

type PluginMarketplaceEntry = {
  id: string                    // Plugin identifier
  name: string                 // Display name
  version: string              // Semver
  description?: string
  source: string | LocalPluginSource  // Relative path or object
  category?: string
  tags?: string[]
  strict?: boolean              // v2 only; if false, skip policy checks
  author?: { name, email?, url? }
  dependencies?: string[]       // Plugin dependencies
  allowCrossMarketplaceDependenciesOn?: string[]
  commands?: ...                // Command metadata
  agents?: ...                 // Agent metadata
  mcpServers?: ...             // MCP server config
  userConfig?: ...             // Plugin-level user config
}

type LocalPluginSource = {
  source: 'directory'
  path: string                 // Relative path to plugin root
}
```

### 2.4 Marketplace Download & Caching

**`marketplaceManager.ts` key functions:**

- **`loadMarketplace(name, options)`**: Fetches/clones marketplace, parses `marketplace.json`
- **`getMarketplaceCacheOnly(name)`**: In-memory lookup without network
- **`gitPull(cwd, ref?, options)`**: Updates cloned marketplace with `git pull` + submodule sync
- **`registerSeedMarketplaces()`**: Ingests read-only seed dir marketplaces into primary config

**Update flow:**
- Disabled by default (except official marketplace)
- Background task: `pluginAutoupdate.ts` runs every 24h
- Timeout: 120s (override via `CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS`)
- Environment: Git credential prompts disabled (`GIT_TERMINAL_PROMPT=0`)

---

## Part 3: Manifest & Schema System

### 3.1 Core Validation Schemas (Zod)

**`schemas.ts` ~1,681 lines - Core validation layers:**

#### 3.1.1 Marketplace Name Validation

```typescript
const ALLOWED_OFFICIAL_MARKETPLACE_NAMES = new Set([
  'claude-code-marketplace',
  'claude-code-plugins',
  'claude-plugins-official',
  'anthropic-marketplace',
  'anthropic-plugins',
  'agent-skills',
  'life-sciences',
  'knowledge-work-plugins',
])

// Blocked pattern detection
const BLOCKED_OFFICIAL_NAME_PATTERN =
  /(?:official[^a-z0-9]*(anthropic|claude)|...)/i

function isBlockedOfficialName(name: string): boolean
  - Check allowlist first (fast path)
  - Block non-ASCII characters (homograph attack defense)
  - Test against blocked pattern
```

#### 3.1.2 Official Name Source Validation

Only reserved marketplace names can be used by official marketplaces:

```typescript
function validateOfficialNameSource(name: string, source: MarketplaceSource) {
  // Reserved names ONLY from official GitHub org (github.com/anthropics/)
  // GitHub SSH: git@github.com:anthropics/...
  // GitHub HTTPS: https://github.com/anthropics/...
}
```

#### 3.1.3 Plugin Manifest Schema

```typescript
type PluginManifest = {
  name: string              // Kebab-case, required
  version?: string         // Semver
  description?: string
  author?: PluginAuthor
  homepage?: string        // URL
  repository?: string      // URL
  license?: string         // SPDX
  keywords?: string[]
  dependencies?: DependencyRef[]  // Plugin dependencies

  // Component paths (multiple formats supported)
  commands?: string | string[] | Record<string, CommandMetadata>
  agents?: string | string[]
  skills?: string | string[]
  outputStyles?: string | string[]

  // Hooks (file or inline)
  hooks?: string | HooksSettings | (string | HooksSettings)[]

  // MCP servers (file, MCPB, or inline)
  mcpServers?: string | McpbPath | Record<string, McpServerConfig>
            | (string | McpbPath | Record<string, McpServerConfig>)[]

  // Plugin-level user config
  userConfig?: Record<string, PluginUserConfigOption>

  // Settings
  settings?: Record<string, unknown>
}

type CommandMetadata = {
  source?: string         // Path to .md file
  content?: string        // Inline markdown
  description?: string
  argumentHint?: string
  model?: string
  allowedTools?: string[]
}
```

#### 3.1.4 Dependency Reference Schema

```typescript
type DependencyRef = string  // "plugin-name" or "plugin-name@marketplace"
```

### 3.2 Path Security

**Validation functions:**

- **`validatePathWithinBase(baseDir, path)`**: Ensures resolved path stays within baseDir
  - Rejects `..` components, absolute paths, symlink escapes
  - Used for marketplace entry `source` field resolution

- **`checkPathTraversal(path, field, errors)`**: Security check in manifest validation
  - Detects `..` in component paths (commands, agents, skills, hooks)
  - Warns in `claude plugin validate` tool

**Attack defense:**
- Symlink resolution via `await realpath()` before comparison
- Circular symlink detection via visited-inode set
- NTFS edge case: Use `bigint: true` for `stat()` to avoid precision loss on high inode numbers

### 3.3 Homograph Attack Prevention

**Unicode-based marketplace name spoofing:**
```typescript
const NON_ASCII_PATTERN = /[^\u0020-\u007E]/

function isBlockedOfficialName(name: string): boolean {
  // Block non-ASCII: "аnthropіc" (Cyrillic а, і) → rejected
  if (NON_ASCII_PATTERN.test(name)) return true
  // Block direct impersonation patterns
  if (BLOCKED_OFFICIAL_NAME_PATTERN.test(name)) return true
}
```

---

## Part 4: Scope System & Settings Architecture

### 4.1 Five-Scope Priority Merge

**Scope precedence** (highest to lowest):
1. **Flag scope** (session-only): `--plugin-enable`/`--plugin-disable` CLI args
2. **Local scope** (project-local): `.claude/.claude/settings.json`
3. **Project scope**: `.claude/settings.json`
4. **User scope**: `~/.claude/settings.json`
5. **Managed scope** (defaults): Built-in for system plugins

**Merge semantics:**
- Higher scope values override lower ones
- Explicit `false` disables a plugin despite lower-scope `true`
- Version constraints (`["^1.0.0"]`) only apply within same scope
- Absent keys don't override — lower scope fills gaps

### 4.2 Installation Metadata (`installed_plugins.json`)

**Purpose:** Track global installation state (orthogonal to enable/disable)

**Schema (V2):**
```typescript
type InstalledPluginsFileV2 = {
  version: 2
  plugins: Record<string, PluginInstallationEntry[]>
}

type PluginInstallationEntry = {
  scope: PluginScope              // Where it was installed
  installPath: string             // Absolute path
  version: string                 // Installed version
  installedAt: string            // ISO timestamp
  lastUpdated?: string           // Last update check
  gitCommitSha?: string          // For git sources
}
```

**Why V2?** V1 used flat structure with one installation per plugin; V2 allows multiple installations per plugin (different scopes/versions).

**Migration (`migrateToSinglePluginFile()`):**
1. If `installed_plugins_v2.json` exists → rename to `installed_plugins.json`
2. If only `installed_plugins.json` (V1) exists → convert in-place
3. Clean up legacy non-versioned cache directories

### 4.3 Configuration Storage

**Non-sensitive config** (`pluginConfigs`):
```json
{
  "pluginConfigs": {
    "plugin@marketplace": {
      "agent": "some-value",
      "mcpServers": {
        "server-name": { "key": "value" }
      }
    }
  }
}
```
- Stored in plaintext `settings.json`
- Updated via `updateSettingsForSource()`

**Sensitive config** (`pluginSecrets`):
- Stored in secure storage (macOS keychain, `.credentials.json` 0600 on Linux/Windows)
- Via `getSecureStorage()` API
- Keyed as `${pluginId}/${serverName}` flat map

**Split logic** (`mcpbHandler.ts:193-341`):
- Schema field `sensitive: true` → secureStorage
- Everything else → plaintext settings
- On reconfigure: scrub old plaintext secrets and cross-store stale values

---

## Part 5: Plugin Content Loading

### 5.1 Command Loading Pipeline

**File:** `loadPluginCommands.ts` (~946 lines)

**Process:**
1. **Discover command sources**
   - `commands/` directory (auto-discovery)
   - Manifest `commands` field (explicit listing)
   - Command metadata object with `source`/`content`

2. **Parse markdown files** (each `.md` is a command)
   ```markdown
   ---
   name: build
   description: Build the project
   argumentHint: "[target]"
   ---

   # Command description in markdown
   ```

3. **Extract frontmatter** (YAML 1.1)
   - Keys: `name`, `description`, `argumentHint`, `model`, `allowedTools`
   - Override by manifest metadata if provided

4. **Namespace commands**
   - Global commands: `/command-name`
   - Plugin commands: `/plugin-name:command-name`
   - Namespace extracted from plugin `manifest.name` or fallback to `source`

5. **Inline content handling**
   - Manifest `commands: { "about": { "content": "# About..." } }`
   - No file parse needed; markdown used directly

6. **Duplicate detection**
   - Error if two command files resolve to same name
   - Error if manifest specifies same command twice

### 5.2 Agent & Skill Loading

**Agents** (`loadPluginAgents.ts`):
- Discovered from `agents/` directory
- Must be `.md` files (Markdown format)
- Frontmatter parsed for metadata (currently minimal)
- Name derived from filename (e.g., `code-reviewer.md` → agent `code-reviewer`)

**Skills** (`loadPluginOutputStyles.ts` pattern):
- Discovered from `skills/` directory
- SKILL.md convention for structured content
- Parallel to agents; skills are callable/composable variants

### 5.3 Hooks Loading

**File:** `loadPluginHooks.ts` (~387 lines)

**Sources:**
1. **Standard location**: `hooks/hooks.json` (auto-loaded)
2. **Manifest references**: `manifest.hooks` array (merged)

**Merge logic** (`createPluginFromPath()` lines 1614-1755):
- Load standard `hooks/hooks.json` first (if exists)
- Merge additional hooks from manifest
- Duplicate detection: Warn if manifest duplicates standard location
- Duplicate file detection: Track normalized realpath to prevent re-loading via symlink

**Hooks schema:**
```typescript
type HooksSettings = {
  onFileCreate?: HookMatcher[]
  onFileSave?: HookMatcher[]
  onSelectionChange?: HookMatcher[]
  // ... other event types
}

type HookMatcher = {
  path?: string                // Glob pattern
  language?: string           // File language
  action: 'run' | 'notify'
  command?: string
  // ...
}
```

### 5.4 Output Styles

**File:** `loadPluginOutputStyles.ts` (~374 lines)

- Discovered from `output-styles/` directory
- Can be directory OR file (both formats supported)
- Registered under plugin namespace
- Used for rendering markdown output with custom formatting

---

## Part 6: Dependency Resolution

### 6.1 DFS Closure Computation

**Function:** `resolveDependencyClosure()` (`dependencyResolver.ts:95-159`)

**Algorithm:** Install-time DFS walker

```typescript
async function resolveDependencyClosure(
  rootId: PluginId,
  lookup: (id) => Promise<DependencyLookupResult | null>,
  alreadyEnabled: ReadonlySet<PluginId>,
  allowedCrossMarketplaces?: ReadonlySet<string>
): Promise<ResolutionResult>
```

**Key behaviors:**
1. **Root always included** in closure (even if already-enabled)
2. **Dependencies skipped** if in `alreadyEnabled` (avoids surprise writes)
3. **Cycle detection** via stack tracking
4. **Cross-marketplace boundary**: Blocked by default
   - Root marketplace can allowlist via `allowCrossMarketplaceDependenciesOn`
   - Only applies to root; no transitive trust
   - Security boundary: trusted marketplace shouldn't pull from untrusted via auto-install

**Return types:**
```typescript
type ResolutionResult =
  | { ok: true, closure: PluginId[] }
  | { ok: false, reason: 'cycle', chain: PluginId[] }
  | { ok: false, reason: 'not-found', missing: PluginId, requiredBy: PluginId }
  | { ok: false, reason: 'cross-marketplace', dependency, requiredBy }
```

### 6.2 Load-time Verification

**Function:** `verifyAndDemote()` (`dependencyResolver.ts:177-234`)

**Purpose:** Session-local safety net (does NOT write settings)

**Algorithm:** Fixed-point loop demoting plugins with unsatisfied deps

1. Iterate over enabled plugins
2. For each, check all `manifest.dependencies` are enabled
3. If any missing/disabled: demote the plugin (remove from enabled set)
4. Re-iterate until no changes
5. Return set of demoted plugin IDs + errors

**Special handling for inline plugins:**
- Bare deps from `@inline` plugins matched by name only (`enabledByName` multiset)
- Full-qualified deps matched exactly
- Reason field distinguishes: `'not-enabled'` vs `'not-found'`

### 6.3 Dependency Qualification

**Function:** `qualifyDependency()` (`dependencyResolver.ts:38-46`)

Bare names inherit declaring plugin's marketplace:
```typescript
"my-dep" → "my-dep@{declaring-plugin-marketplace}"
```

**Exception:** `@inline` plugins can't inherit `@inline` (synthetic sentinel)
- Bare deps from `@inline` plugins returned unchanged
- Matched name-only via `verifyAndDemote`

### 6.4 Reverse Dependencies

**Function:** `findReverseDependents()` (`dependencyResolver.ts:244-263`)

Returns list of enabled plugins that depend on target:
- Used for uninstall/disable warnings ("required by X, Y")
- Name-only matching for `@inline` deps

---

## Part 7: MCP Integration

### 7.1 MCPB/DXT File Handling

**File:** `mcpbHandler.ts` (~968 lines)

**Format:** MCPB (Model Context Protocol Bundle) = ZIP archive with manifest

**Download & extraction:**
```typescript
async function downloadMcpb(url: string, destPath: string): Promise<Uint8Array>
  - Axios GET with 2min timeout, 5 redirects
  - Telemetry via logPluginFetch()

async function extractMcpb(source: string, extractPath: string): Promise<void>
  - Parse manifest from ZIP via parseAndValidateManifestFromBytes()
  - Extract to temp directory
  - Move to cache on success
```

**Manifest parsing:**
- `@anthropic-ai/mcpb` library handles serialization/validation
- Returns `McpbManifest` with `servers[]`, `user_config`, `channels`

### 7.2 Content Hash & Caching

**SHA256 hashing:**
```typescript
function generateContentHash(data: Uint8Array): string {
  return createHash('sha256').update(data).digest('hex').substring(0, 16)
}
```

**Cache metadata** (per MCPB source):
```typescript
type McpbCacheMetadata = {
  source: string              // URL or path
  contentHash: string        // SHA256 truncated
  extractedPath: string      // Where files are
  cachedAt: string          // ISO timestamp
  lastChecked: string       // Last update check
}
```

### 7.3 User Configuration Handling

**Load configuration** (`mcpbHandler.ts:141-172`):
```typescript
function loadMcpServerUserConfig(pluginId: string, serverName: string): UserConfigValues | null
  - Read non-sensitive from settings.pluginConfigs[pluginId].mcpServers[serverName]
  - Read sensitive from secureStorage.pluginSecrets[${pluginId}/${serverName}]
  - Merge: secureStorage wins on collision
```

**Save configuration** (`mcpbHandler.ts:193-341`):
- Split by `schema[key].sensitive` field
- Sensitive → secureStorage (keychain/0600 file)
- Non-sensitive → settings.json
- Scrub cross-store stale values on reconfigure

**Validation** (`mcpbHandler.ts:346-408`):
- Type checking: string, number, boolean, file, directory
- Range validation for numbers (min/max)
- Required field checking
- Error collection for UI display

### 7.4 MCP Server Generation

**Function:** `generateMcpConfig()` (`mcpbHandler.ts:413-438`)

```typescript
async function generateMcpConfig(
  manifest: McpbManifest,
  extractedPath: string,
  userConfig: UserConfigValues
): Promise<McpServerConfig>
```

- Lazy import of `@anthropic-ai/mcpb` (700KB of closures)
- Calls `getMcpConfigForManifest()` with system directories
- Returns validated `McpServerConfig`

---

## Part 8: Security Infrastructure

### 8.1 Path Traversal Prevention (Defense in Depth)

**Layer 1: Manifest validation** (`validatePlugin.ts`)
- Scan `commands`, `agents`, `skills` arrays for `..` sequences
- Block early with friendly error messages

**Layer 2: Runtime validation** (`pluginInstallationHelpers.ts`)
- `validatePathWithinBase(baseDir, relPath)` function
- Resolves symlinks via `realpath()`
- Compares: must start with baseDir prefix
- Used when accessing marketplace entry `source` fields

**Layer 3: ZIP cache handling** (`zipCache.ts:259-271`)
- Symlink cycle detection via inode tracking
- `bigint: true` stat() to avoid Windows precision loss
- Skip symlinked directories (follow files)

### 8.2 Official Marketplace Protection

**Homograph attacks:**
```typescript
// Cyrillic 'а' (U+0430) instead of Latin 'a'
isBlockedOfficialName("аnthropic")  // true ✓

// Multiple homograph sources tracked
BLOCKED_OFFICIAL_NAME_PATTERN = /(?:official...anthropic|claude.*official)/i

// Non-ASCII completely rejected
NON_ASCII_PATTERN = /[^\u0020-\u007E]/
```

**Reserved name source validation:**
- Reserved names: only from `github.com/anthropics/` (GitHub, HTTPS, SSH)
- Schema validation at marketplace registration
- Stored source checked before registration

### 8.3 Plugin Blocklist & Delisting

**Blocklist** (`pluginBlocklist.ts`):
- File: `~/.claude/plugins/delisted.json`
- Tracks plugins auto-removed from Marketplace
- Checked during load; disabled if listed
- Prevents re-enabling deleted plugins

**Flagging** (`pluginFlagging.ts`):
- Tracks plugins flagged for auto-removal
- Prepared list from Marketplace sync
- Session records which plugins were affected

### 8.4 Enterprise Policy Enforcement

**Allowlist** (strict):
```typescript
getStrictKnownMarketplaces(): string[] | null
  // null = no allowlist (open)
  // [] = empty allowlist (deny all)
  // ["official", "company"] = only these allowed
```

**Blocklist:**
```typescript
getBlockedMarketplaces(): string[] | null
  // null = no blocklist
  // [] = empty list (allow all)
  // ["untrusted"] = deny these
```

**Fail-closed guard** (`pluginLoader.ts:1967-1998`):
- If policy is active AND marketplace source unknown → block
- Prevents silent fail-open when config is corrupted
- Logged as `marketplace-blocked-by-policy` error

**Policy application:**
- Checked during `loadPluginsFromMarketplaces()`
- Per-plugin: blocks load if source doesn't match policy
- Doesn't affect already-cached plugins (can't retroactively uninstall)

---

## Part 9: Autoupdate System

### 9.1 Background Update Flow

**File:** `pluginAutoupdate.ts` (~284 lines)

**Triggers:**
- Marketplace autoupdate: enabled by default for official marketplaces
- Per-marketplace: `autoUpdate: true/false` in `known_marketplaces.json`
- Exclusion list: `NO_AUTO_UPDATE_OFFICIAL_MARKETPLACES` (currently `knowledge-work-plugins`)

**Process:**
1. On startup: Check if 24h since last update
2. Background task: `git pull` for GitHub/git sources
3. No blocking: UI shows "plugins updated" notification after restart
4. Disk-only: Changes written to cache, not settings
5. Next session: Reloaded from updated cache

**Git pull with timeout:**
```typescript
async function gitPull(cwd: string, ref?: string, options?: { ... })
  - Timeout: DEFAULT_PLUGIN_GIT_TIMEOUT_MS = 120s
  - Override: CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS env var
  - Credential prompts disabled: GIT_TERMINAL_PROMPT=0
  - Ref support: fetch + checkout specific branch/tag
  - Submodule sync: Update submodule working dirs after pull
```

### 9.2 Autoupdate Policy

**Which marketplaces auto-update by default?**

Marketplace names in `ALLOWED_OFFICIAL_MARKETPLACE_NAMES` EXCEPT those in `NO_AUTO_UPDATE_OFFICIAL_MARKETPLACES`:
```typescript
isMarketplaceAutoUpdate(marketplaceName: string, entry: { autoUpdate? })
  return entry.autoUpdate ?? (ALLOWED && !NO_AUTO_UPDATE)
```

**Current official marketplace list:**
- `claude-code-marketplace`
- `claude-code-plugins`
- `claude-plugins-official`
- `anthropic-marketplace`
- `anthropic-plugins`
- `agent-skills`
- `life-sciences`

---

## Part 10: ZIP Cache (Headless Mode)

### 10.1 Purpose & Activation

**Environment variables:**
```bash
CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE=1
CLAUDE_CODE_PLUGIN_CACHE_DIR=/mnt/plugins-cache
```

**Use case:** Headless/CI environments where plugins live on mounted Filestore

**Canonical format:** ZIP archives (not directories)

### 10.2 Directory Structure

```
/mnt/plugins-cache/
├── known_marketplaces.json          # Marketplace registry
├── installed_plugins.json           # Installation metadata
├── marketplaces/
│   ├── official-marketplace.json    # Cached marketplace.json
│   └── company-marketplace.json
└── plugins/
    ├── official-marketplace/
    │   └── my-plugin/
    │       └── 1.0.0.zip            # ZIP archive (canonical)
    └── company-marketplace/
        └── other-plugin/
            └── 2.1.3.zip
```

### 10.3 Session Extraction

**Process:**
1. Create session temp directory: `/tmp/claude-plugin-session-{random}/`
2. On load: Extract ZIPs to session temp
3. Use extracted paths during session
4. On shutdown: Cleanup session temp

**Function:** `getSessionPluginCachePath()` (`zipCache.ts:125-140`)

**Atomic ZIP write** (`atomicWriteToZipCache()`):
- Write to temp file in same dir
- Rename (atomic on POSIX)
- Cleanup temp on failure

### 10.4 ZIP Creation

**Function:** `createZipFromDirectory()` (`zipCache.ts:216-229`)

**Preserves:**
- File permissions (+x bits via external_attr)
- Symlinks (resolved to actual content, not links)
- Directory structure

**Skips:**
- `.git` directories (git clone metadata)
- Symlink cycles (tracked by dev+ino inode pairs)
- Broken symlinks (logged as warning)

**Compression:** Level 6 (fflate library)

---

## Part 11: Plugin Content Types & Constants

### 11.1 Plugin Identifier Format

**Canonical:** `{name}@{marketplace}`

**Parsing** (`parsePluginIdentifier.ts`):
```typescript
function parsePluginIdentifier(id: string) {
  const [name, marketplace] = id.split('@')
  return { name, marketplace }
}
```

**Validation** (`PluginIdSchema()`):
- Requires `@marketplace` suffix
- Name: alphanumeric, hyphens, underscores
- Marketplace: kebab-case, reserved names protected

### 11.2 Component Types

```typescript
type PluginComponent = 'commands' | 'agents' | 'skills' | 'output-styles' | 'hooks' | 'mcpServers'
```

**Per-component discovery:**
- Standard directory: `{component}s/` (except output-styles → `output-styles/`)
- Manifest field: `manifest.{component}s`
- Metadata format: Object-mapping for commands, arrays for others

### 11.3 Magic Strings & Constants

```typescript
BUILTIN_MARKETPLACE_NAME = 'builtin'  // Reserved
OFFICIAL_MARKETPLACE_NAME = 'claude-code-marketplace'
INLINE_MARKETPLACE = 'inline'  // Session-only, synthetic

OFFICIAL_GITHUB_ORG = 'anthropics'  // For source validation

NO_AUTO_UPDATE_OFFICIAL_MARKETPLACES = Set(['knowledge-work-plugins'])

DEFAULT_PLUGIN_GIT_TIMEOUT_MS = 120 * 1000  // 2 minutes

// Homograph detection
BLOCKED_OFFICIAL_NAME_PATTERN = /(?:official...anthropic|claude.*official)/i
NON_ASCII_PATTERN = /[^\u0020-\u007E]/
```

---

## Part 12: Error Handling & Reporting

### 12.1 Plugin Error Types

```typescript
type PluginError =
  | { type: 'path-not-found', source, plugin, path, component }
  | { type: 'hook-load-failed', source, plugin, hookPath, reason }
  | { type: 'dependency-unsatisfied', source, plugin, dependency, reason }
  | { type: 'marketplace-blocked-by-policy', source, plugin, marketplace, blockedByBlocklist, allowedSources }
  | { type: 'plugin-disabled-missing-deps', ... }
```

### 12.2 Fetch Telemetry

**Function:** `classifyFetchError()` (`fetchTelemetry.ts`)

**Categories:**
- `network_error`: Connection refused, timeout, DNS
- `auth_error`: 401, 403
- `not_found`: 404
- `server_error`: 5xx
- `unknown`: Other

**Logging:**
```typescript
logPluginFetch(
  eventType: 'plugin_clone' | 'plugin_install' | 'marketplace_url',
  url: string,
  status: 'success' | 'failure',
  durationMs: number,
  errorType?: string
)
```

### 12.3 Failure Recovery

**Fetch failures:**
- Marketplace URL unreachable → use cached version
- Git clone timeout → block plugin (not critical)
- Manifest invalid → log error, don't load plugin

**Installation failures:**
- Partial download → cleanup temp directory
- Manifest missing → create default manifest
- Cache collision → overwrite old version

---

## Part 13: Security Analysis & Attack Vectors

### 13.1 Supply Chain Attacks

**Vulnerability:** Marketplace hijacking via code in untrusted source

**Mitigations:**
- Enterprise allowlist/blocklist policies
- Official name protection (source validation)
- Homograph detection (non-ASCII blocking)
- Path traversal prevention (manifest validation)

**Remaining risk:** Trusted marketplace compromised
- Mitigation: No per-plugin signature verification (complexity vs. trust model)
- User assumes marketplace is trustworthy before adding to settings

### 13.2 Manifest Injection

**Attack:** Malicious marketplace.json with crafted plugin entries

**Defense:**
- Schema validation (Zod) — type/format checks
- Path traversal checks (validatePathWithinBase)
- Component path validation (no `..` sequences)
- Unknown keys stripped (lenient schema)

### 13.3 Marketplace Spoofing

**Attack:** Register "claude-plugins-official" as third-party marketplace

**Defense:**
- Blocked name pattern detection
- Homograph (non-ASCII) detection
- Source verification: reserved names only from `anthropics` GitHub org
- Marketplace name validation in settings schema

### 13.4 Dependency Cycle / Missing Dependency

**Attack:** Circular or broken dependency chain

**Defense:**
- Install-time cycle detection (stack-based DFS)
- Load-time verification + demotion (fixed-point loop)
- Cross-marketplace boundary enforcement (default deny)

**Remaining:** User can manually enable broken config in settings
- Mitigation: `/doctor` command surfaces dependency errors

### 13.5 Configuration Exfiltration

**Risk:** Sensitive config (API keys) stored in plaintext

**Mitigation:**
- Plugin schema field `sensitive: true` → secure storage
- MCPB server config `sensitive: true` → keychain/0600
- User visible in config dialog (masked input)

**Current gap:** User can edit settings.json directly
- Mitigation: Documentation warning, future: encrypted settings.json

### 13.6 Symlink Attacks

**Risk:** Plugin installation follows symlink to escape cache directory

**Defense:**
- `realpath()` resolution before path comparison
- Inode cycle detection (dev+ino tracking)
- Windows precision issue fixed: `bigint: true` in stat()

### 13.7 ZIP Archive Attacks

**Risk:** Malicious ZIP with directory traversal (`../../etc/passwd`)

**Defense:**
- fflate ZIP parser (not vulnerable to directory traversal)
- Manual directory traversal check not needed (fflate safe)
- Symlink resolution during ZIP creation

---

## Part 14: Data Flow Diagrams

### Plugin Installation Flow

```
User runs: claude plugin install @marketplace/my-plugin
           ↓
parsePluginIdentifier("my-plugin@marketplace")
           ↓
loadKnownMarketplaces() → get marketplace source
           ↓
getPluginByIdCacheOnly(pluginId) → fetch marketplace.json
           ↓
extractEntry from marketplace.json (source: path or URL)
           ↓
(Parallel: resolve dependencies via resolveDependencyClosure)
           ↓
For each plugin in closure:
  ├→ cachePlugin(source) → download/clone
  ├→ copyPluginToVersionedCache(sourcePath, version)
  ├→ createPluginFromPath(cachePath) → LoadedPlugin
  ├→ registerInstallation(pluginId, version, scope) → installed_plugins.json
  └→ updateSettings(enabledPlugins[pluginId] = true)
           ↓
Show installation summary (main + dependencies)
```

### Plugin Load Flow

```
Claude Code startup
           ↓
migrateToSinglePluginFile() (V1→V2 conversion, once per session)
           ↓
loadPluginsFromMarketplaces(cacheOnly: false)
  ├→ getInMemoryInstalledPlugins() (cache)
  ├→ loadKnownMarketplaces() (known_marketplaces.json)
  ├→ For each enabledPlugins entry:
  │   ├→ Check enterprise policy (allowlist/blocklist)
  │   ├→ getPluginByIdCacheOnly(pluginId)
  │   ├→ extractEntry.source → cache path
  │   ├→ resolvePluginPath(pluginId, version) → directory
  │   ├→ createPluginFromPath() → LoadedPlugin
  │   └→ Load components (commands, agents, hooks, MCP)
  └→ Return { plugins[], errors[] }
           ↓
verifyAndDemote(plugins) → demote plugins with unsatisfied deps
           ↓
loadComponentsInParallel(plugins)
  ├→ loadPluginCommands() → namespaced /commands
  ├→ loadPluginAgents() → agents
  ├→ loadPluginHooks() → hooks registry
  └→ mcpPluginIntegration.ts → MCP servers
           ↓
Ready for user interaction
```

### Marketplace Update Flow

```
Every 24h (background task via pluginAutoupdate.ts)
           ↓
For each marketplace with autoUpdate: true:
  ├→ seedDirFor(installLocation)? → skip if seed-managed
  ├→ gitPull(cwd, ref?, options) → background, non-blocking
  ├→ On success: logForDebugging("marketplace updated")
  └→ On failure: logForDebugging("update failed")
           ↓
On next session startup:
  └→ Reloaded from updated cache
```

---

## Part 15: Performance Optimizations

### 15.1 Memoization

**Functions using memoize:**
- `getMarketplace()` — in-memory marketplace cache
- Various schema loaders — Zod schema compilation cached

**Benefit:** Avoid re-parsing marketplace.json per plugin

### 15.2 Parallel Operations

**Parallelized:**
- Plugin path validation (pathExists checks)
- Marketplace catalog loading (Promise.all per marketplace)
- Component loading (agents, skills, hooks)

**Benefit:** Startup time reduced from ~500ms (sequential) to ~150ms

### 15.3 Lazy Imports

- `@anthropic-ai/mcpb` imported only when MCPB file needed
- Saves ~700KB memory for users without MCPB plugins

### 15.4 ZIP Cache for Headless

- Reduces disk footprint (compressed archives)
- Faster container startup (session extraction vs. full installation)
- Mounted Filestore integration (immutable, no write-back)

---

## Part 16: File Listing & Line Counts

| File | Lines | Purpose |
|------|-------|---------|
| pluginLoader.ts | 3,302 | Plugin discovery, caching, manifest loading |
| marketplaceManager.ts | 2,643 | Marketplace sources, config, updates |
| schemas.ts | 1,681 | Zod validation, homograph detection |
| installedPluginsManager.ts | 1,268 | Installation metadata, V1→V2 migration |
| mcpbHandler.ts | 968 | MCPB download, extraction, user config |
| loadPluginCommands.ts | 946 | Command markdown parsing, namespacing |
| validatePlugin.ts | 903 | CLI validation tool |
| mcpPluginIntegration.ts | 634 | MCP server loading, env expansion |
| pluginInstallationHelpers.ts | 595 | Path security, dependency closure |
| marketplaceHelpers.ts | 592 | Policy enforcement, source formatting |
| loadPluginHooks.ts | 387 | Hook loading, merging, validation |
| lspPluginIntegration.ts | 387 | LSP server integration |
| lspRecommendation.ts | 374 | Language-based plugin hints |
| zipCache.ts | 406 | ZIP archive caching, headless extraction |
| pluginAutoupdate.ts | 284 | Background marketplace updates |
| installCounts.ts | 292 | Installation analytics |
| dependencyResolver.ts | 305 | DFS closure, cycle detection, load-time verification |
| pluginDirectories.ts | 142 | Path management |
| pluginVersioning.ts | 128 | Semver parsing |
| pluginIdentifier.ts | 98 | Name/marketplace parsing |
| pluginBlocklist.ts | 127 | Delisted plugin detection |
| pluginFlagging.ts | 208 | Auto-removed plugin tracking |
| reconciler.ts | 265 | Marketplace reconciliation |
| officialMarketplaceStartupCheck.ts | 439 | Official marketplace availability |
| pluginStartupCheck.ts | 341 | Enabled plugin resolution |
| pluginOptionsStorage.ts | 400 | Config storage (non-sensitive) |
| headlessPluginInstall.ts | 210 | Headless mode installation |
| loadPluginAgents.ts | 378 | Agent loading |
| loadPluginOutputStyles.ts | 175 | Output styles loading |
| cacheUtils.ts | 239 | Cache management helpers |
| parseMarketplaceInput.ts | 186 | CLI marketplace parsing |
| orphanedPluginFilter.ts | 152 | Unused plugin detection |
| fetchTelemetry.ts | 166 | Network request logging |
| gitAvailability.ts | 87 | Git binary detection |
| managedPlugins.ts | 34 | Managed plugin list |
| officialMarketplace.ts | 73 | Official marketplace defaults |
| officialMarketplaceGcs.ts | 332 | GCS marketplace fetching |
| refresh.ts | 156 | Full marketplace refresh |
| addDirPluginSettings.ts | 121 | --plugin-dir settings |
| walkPluginMarkdown.ts | 142 | Markdown AST walking |
| hintRecommendation.ts | 181 | Plugin hints |
| performStartupChecks.tsx | 268 | UI startup checks |
| pluginPolicy.ts | 156 | Policy rule parsing |
| zipCacheAdapters.ts | 234 | ZIP cache integration layer |
| **TOTAL** | **20,521** | |

---

## Conclusion

Claude Code's plugin system is a **sophisticated, well-engineered infrastructure** balancing:

1. **Flexibility** (six installation methods, five scopes, rich composition)
2. **Security** (path traversal prevention, homograph detection, policy enforcement)
3. **Performance** (memoization, parallelization, ZIP caching for headless)
4. **Resilience** (fallback to cache on network failure, load-time demotion of broken deps)

The **three-layer data model** (disk state, settings intent, session memory) enables:
- Per-repository plugin selection (project/local scopes)
- Global installation tracking (installed_plugins.json)
- Backward compatibility (V1→V2 migration)

The **five-scope merge** provides fine-grained control without overwhelming users with settings dialogs.

The **dependency system** enforces marketplace boundaries by default while allowing explicit cross-marketplace dependencies when declared.

For future maintainers: Key complexity areas are **path resolution** (multiple fallback chains), **schema validation** (Zod with custom refinements), and **settings merge semantics** (scope precedence, undefined as delete).
