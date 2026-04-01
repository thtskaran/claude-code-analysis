# Claude Code Settings, Policy, and Managed Settings Infrastructure Deep Dive

## Executive Summary

Claude Code implements a sophisticated multi-layered enterprise settings and policy management system designed for organizations deploying Claude Code at scale. The infrastructure uses a "first-source-wins" priority model across four distinct layers: remote managed settings (push from API), MDM/registry (admin enforcement), file-based settings, and user preferences. The system employs aggressive fail-open patterns, ETag-based HTTP caching, background polling, and security dialogs to balance enterprise control with graceful degradation and user autonomy.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Layer 1: Policy Limits Service](#layer-1-policy-limits-service)
3. [Layer 2: Remote Managed Settings](#layer-2-remote-managed-settings)
4. [Layer 3: Settings Sync (User Settings Cross-Device)](#layer-3-settings-sync-user-settings-cross-device)
5. [Layer 4: MDM Integration (macOS/Windows)](#layer-4-mdm-integration-macos-windows)
6. [System Composition and Priority](#system-composition-and-priority)
7. [Authentication and Eligibility](#authentication-and-eligibility)
8. [Error Handling and Resilience](#error-handling-and-resilience)
9. [Caching Strategies](#caching-strategies)
10. [Background Polling and Hot Reload](#background-polling-and-hot-reload)

---

## Architecture Overview

### Design Philosophy: "First-Source-Wins"

The settings infrastructure uses a tiered priority model where the **highest-priority source that has content wins** and provides all settings. This prevents layering (e.g., an empty plist shouldn't fall back to HKCU; if an admin explicitly sets a plist, their intent is final even if incomplete).

**Priority Order (highest to lowest):**
1. **Remote Managed Settings** — Push from backend API (org admin configured)
2. **Admin MDM** — macOS `/Library/Managed Preferences/` (plist) or Windows HKLM registry
3. **File-Based Managed Settings** — `/etc/claude-code/managed-settings.json` (Linux) or local file
4. **User HKCU** — Windows `HKCU` registry (user-writable, lowest priority)
5. **User Preferences** — `~/.claude/settings.json` (local CLI user settings)

### Fail-Open Philosophy

All policy systems are **non-blocking** and fail gracefully:
- API failures → falls back to cached settings or continues without restrictions
- Missing cache files → continues without settings (no hard failure)
- Network timeouts → uses stale cache if available, otherwise continues
- Malformed settings → ignored, other sources still applied
- Auth errors → skips retry, doesn't block startup

This ensures the CLI remains functional even if the backend is unavailable, network is degraded, or enterprise settings fail to load.

---

## Layer 1: Policy Limits Service

**Location:** `/src/services/policyLimits/`

**Purpose:** Enforce org-level policy restrictions that **disable** CLI features. Examples: `allow_remote_sessions`, `allow_product_feedback`, etc.

### Type Definitions (`types.ts`)

```typescript
PolicyLimitsResponse = {
  restrictions: {
    [policyKey: string]: { allowed: boolean }
  }
}

PolicyLimitsFetchResult = {
  success: boolean
  restrictions?: restrictions | null  // null = 304 Not Modified
  etag?: string
  error?: string
  skipRetry?: boolean  // true = don't retry (auth errors)
}
```

**Key Design:** Only **blocked** policies are included in the response. If a policy key is absent, it's **allowed by default** (fail-open). This minimizes payload size and makes the policy list additive.

### Eligibility (`index.ts`: `isPolicyLimitsEligible()`)

Users eligible for policy limits:

**Console (API key):**
- Must have an actual Anthropic API key (not apiKeyHelper)
- Must use first-party Anthropic endpoints

**OAuth (Claude.ai):**
- Only Team and Enterprise/C4E subscribers eligible
- Must have `user:inference` scope
- Checks `subscriptionType` field

**Ineligible:**
- Third-party provider users (Bedrock, custom endpoints)
- Free tier Claude.ai users

### Fetching and Caching (`index.ts`: Main Functions)

#### Initialization Phase
```typescript
initializePolicyLimitsLoadingPromise()
// Creates a promise that other systems can await
// Includes 30-second timeout to prevent deadlocks
// Called early in bootstrap
```

#### Fetch with Retry
```typescript
fetchWithRetry(cachedChecksum?: string)
// Up to 5 retries with exponential backoff
// Uses getRetryDelay() from api/withRetry.ts
// Returns on success, 304, 404, or non-retryable error (auth)
```

#### Fetch Single Attempt
```typescript
fetchPolicyLimits(cachedChecksum?: string)
// GET /api/claude_code/policy_limits
// Conditional ETag header: If-None-Match: "checksum"
// 200 OK = new restrictions
// 304 Not Modified = cached version still valid
// 404 Not Found = no restrictions exist
```

#### HTTP Caching
- **Checksum computation:** SHA256 of sorted, normalized JSON (consistent with server)
- **Format:** `sha256:hexdigest`
- **Header:** `If-None-Match: "sha256:..."`
- **Response 304:** Returns `restrictions: null` (client keeps cached version)

#### Cache File
- **Location:** `~/.claude/policy-limits.json`
- **Permissions:** 0o600 (user-only)
- **Content:** Full `PolicyLimitsResponse` (including restrictions object)

#### Special Handling: Essential-Traffic-Only Mode
```typescript
const ESSENTIAL_TRAFFIC_DENY_ON_MISS = new Set(['allow_product_feedback'])
```

If `isEssentialTrafficOnly()` (HIPAA/compliance mode) is active AND policy cache is unavailable, certain policies default to **denied** instead of allowed. Prevents silent re-enablement of features that must stay disabled.

### Checking Policy Status

```typescript
isPolicyAllowed(policy: string): boolean
// Returns true if:
//   - Policy unknown (fail-open)
//   - Policy explicitly allowed
//   - Cache unavailable AND policy not in ESSENTIAL_TRAFFIC_DENY_ON_MISS
// Returns false if:
//   - Policy explicitly denied
//   - Cache unavailable AND essential-traffic-only AND policy is essential
```

### Session-Level Cache

```typescript
let sessionCache: PolicyLimitsResponse['restrictions'] | null = null

getRestrictionsFromCache()
// Sync lookup: checks session cache, then file cache
// Called by isPolicyAllowed() — must be synchronous
```

### Background Polling

```typescript
startBackgroundPolling()  // Every 60 minutes
stopBackgroundPolling()

pollPolicyLimits()
// Fires asyncly every hour to pick up mid-session changes
// Compares cached restrictions; logs if changed
// Doesn't fail closed if network fails
```

### Lifecycle

1. **Init phase:** `initializePolicyLimitsLoadingPromise()` creates promise, sets 30s timeout
2. **Startup:** `loadPolicyLimits()` fires fetch with retry, starts background polling, resolves promise
3. **Runtime:** `isPolicyAllowed(policy)` checks cache (sync)
4. **Auth change:** `refreshPolicyLimits()` clears cache and refetches
5. **Shutdown:** `stopBackgroundPolling()` + cleanup registered

---

## Layer 2: Remote Managed Settings

**Location:** `/src/services/remoteManagedSettings/`

**Purpose:** Allow org admins to push settings to Claude Code instances remotely. Example: enforce specific `max_tokens`, disable plugins, enforce env var mappings, etc.

### Type Definitions (`types.ts`)

```typescript
RemoteManagedSettingsResponse = {
  uuid: string              // Settings UUID (opaque)
  checksum: string          // SHA256 checksum
  settings: SettingsJson    // Full settings object
}

RemoteManagedSettingsFetchResult = {
  success: boolean
  settings?: SettingsJson | null  // null = 304 Not Modified
  checksum?: string
  error?: string
  skipRetry?: boolean
}
```

### Eligibility (`syncCache.ts`: `isRemoteManagedSettingsEligible()`)

**Console (API key):**
- All users with actual Anthropic API key eligible

**OAuth (Claude.ai):**
- Enterprise and Team subscribers
- Or externally-injected tokens (CCD, CCR, Agent SDK) with `subscriptionType: null` — API decides via 204/404

**Ineligible:**
- Third-party provider users
- Free tier Claude.ai users
- Local-agent (Cowork) entrypoint (different permission model)

### State Architecture

**`syncCacheState.ts`** (Leaf Module)
- Minimal imports (no auth.ts) to break circular dependency with settings.ts
- Holds session and file caches
- Called by settings.ts directly

**`syncCache.ts`** (Auth-Aware Wrapper)
- Imports auth.ts
- Runs eligibility check once, caches result
- Re-exported functions for callers

**`index.ts`** (Fetch/Sync Logic)
- HTTP fetch, retry, security checks
- Manages loading promise and background polling

### Fetching and Caching

#### Endpoint
```
GET /api/claude_code/settings
```

#### HTTP Caching
- **Checksum computation:** SHA256 of sorted, normalized JSON
- **Must match server's:** `json.dumps(settings, sort_keys=True, separators=(",", ":"))`
- **Header:** `If-None-Match: "sha256:..."`
- **Response codes:**
  - 200 OK = new settings
  - 304 Not Modified = cached settings still valid
  - 204 No Content / 404 Not Found = no settings configured

#### Cache-First Bootstrap
```typescript
loadRemoteManagedSettings()
// 1. Check disk cache; if exists, apply to session and resolve waiters immediately
// 2. Fire async fetch in parallel
// 3. After fetch: trigger settingsChangeDetector.notifyChange('policySettings')
```

This minimizes startup latency — if settings are cached, CLI doesn't wait for network round-trip.

#### Fetch with Retry
```typescript
fetchWithRetry(cachedChecksum?: string)
// Up to 5 retries with exponential backoff
// Returns on success, 304, 404, or auth error (skipRetry: true)
```

#### File Cache
- **Location:** `~/.claude/remote-settings.json`
- **Permissions:** 0o600 (user-only)
- **Content:** Raw `SettingsJson` (checksum computed on-demand)

### Security: Dangerous Settings Dialog

**File:** `securityCheck.tsx`

When remote settings contain potentially dangerous changes (e.g., disabling privacy features, changing API endpoints), a blocking dialog prompts the user to approve:

```typescript
checkManagedSettingsSecurity(cachedSettings, newSettings)
// Returns 'approved', 'rejected', or 'no_check_needed'

handleSecurityCheckResult(result)
// If 'rejected': gracefulShutdownSync(1) — exits CLI
// Otherwise: continues
```

**No check needed if:**
- No dangerous settings in new settings
- Dangerous settings haven't changed from cached
- Non-interactive mode (e.g., headless/print mode)

**Dialog shown if:**
- Dangerous settings detected AND changed AND interactive mode

**Logged as:**
- `tengu_managed_settings_security_dialog_shown`
- `tengu_managed_settings_security_dialog_accepted`
- `tengu_managed_settings_security_dialog_rejected`

### Validation

Full `SettingsSchema` validation after parse:
```typescript
const settingsValidation = SettingsSchema().safeParse(parsed.data.settings)
if (!settingsValidation.success) {
  // Invalid settings structure — return error, don't apply
}
```

### Session Cache
```typescript
let sessionCache: SettingsJson | null = null

setSessionCache(value: SettingsJson | null)
getRemoteManagedSettingsSyncFromCache(): SettingsJson | null
```

### Background Polling

```typescript
startBackgroundPolling()  // Every 60 minutes
stopBackgroundPolling()

pollRemoteSettings()
// Compares cached settings; triggers notifyChange if changed
// Allows mid-session updates without restart
```

### Lifecycle

1. **Init phase:** `initializeRemoteManagedSettingsLoadingPromise()` creates promise, 30s timeout
2. **Startup phase:**
   - Load disk cache (if available, apply immediately)
   - Fire async fetch with retry
   - Trigger change detection if new settings loaded
   - Start background polling
3. **Runtime:** Settings merged in `getSettings()` via settings.ts pipeline
4. **Auth change:** `refreshRemoteManagedSettings()` clears cache, refetches, notifies
5. **Shutdown:** `stopBackgroundPolling()` registered in cleanup

---

## Layer 3: Settings Sync (User Settings Cross-Device)

**Location:** `/src/services/settingsSync/`

**Purpose:** Sync user settings and memory across Claude Code environments (interactive CLI ↔ CCR/headless agent).

### Type Definitions (`types.ts`)

```typescript
UserSyncData = {
  userId: string
  version: number
  lastModified: string      // ISO 8601
  checksum: string          // MD5
  content: {
    entries: {
      [key: string]: string // UTF-8 content
    }
  }
}

SYNC_KEYS = {
  USER_SETTINGS: '~/.claude/settings.json',
  USER_MEMORY: '~/.claude/CLAUDE.md',
  projectSettings: (id) => 'projects/{id}/.claude/settings.local.json',
  projectMemory: (id) => 'projects/{id}/CLAUDE.local.md',
}
```

### Endpoint

```
GET/PUT /api/claude_code/user_settings
```

### Two Modes of Operation

#### 1. Upload (Interactive CLI) — Push

**When:** `uploadUserSettingsInBackground()` called from main.tsx preAction

**Gating:**
- Feature flag `UPLOAD_USER_SETTINGS` enabled
- Growth book flag `tengu_enable_settings_sync_push` enabled
- Interactive mode (`getIsInteractive()`)
- Using OAuth

**Flow:**
1. Fetch current remote settings (to detect changes)
2. Build local entries from files:
   - Global: `~/.claude/settings.json`, `~/.claude/CLAUDE.md`
   - Project: `projects/{id}/.claude/settings.local.json`, `projects/{id}/CLAUDE.local.md`
3. Diff: compare local vs remote
4. Upload only changed entries via `PUT /api/claude_code/user_settings`

**File Size Limit:** 500 KB per file (matches backend limit)

**Background:** Fire-and-forget, caller doesn't await

#### 2. Download (CCR/Agent) — Pull

**When:**
- CCR startup (`print.ts` runHeadless)
- Manual reload (`/reload-plugins` command in CCR)

**Gating:**
- Feature flag `DOWNLOAD_USER_SETTINGS` enabled
- Growth book flag `tengu_strap_foyer` enabled
- OAuth tokens available

**Flow:**
1. `downloadUserSettings()` — returns cached promise (coalesces multiple calls)
2. Fetch remote settings
3. Apply to local files (only matching keys)
4. Invalidate caches (`resetSettingsCache()`, `clearMemoryFileCaches()`)
5. Caller fires `settingsChangeDetector.notifyChange('policySettings')`

**Manual reload:** `redownloadUserSettings()` clears cache, forces fresh fetch (0 retries)

### Auth and Eligibility

```typescript
isUsingOAuth(): boolean
// Checks:
//   - First-party provider
//   - First-party base URL
//   - OAuth token + user:inference scope
```

Note: Requires `user:inference` scope specifically (not `user:profile`), because CCR's file-descriptor token hardcodes `['user:inference']`.

### Fetch Implementation

```typescript
fetchUserSettingsOnce()
// GET /api/claude_code/user_settings
// Returns 200 (full data) or 404 (no settings yet)

fetchUserSettings(maxRetries = DEFAULT_MAX_RETRIES)
// Retry wrapper: 3 retries default, exponential backoff
// Stops on auth error (skipRetry: true)
```

### Upload Implementation

```typescript
uploadUserSettings(entries: Record<string, string>)
// PUT /api/claude_code/user_settings
// Body: { entries }
// Returns checksum and lastModified timestamp
```

### File I/O and Cache Invalidation

**Read for sync:**
```typescript
tryReadFileForSync(filePath: string): Promise<string | null>
// Checks size limit (500 KB)
// Skips empty/whitespace-only files
// Graceful on not-found
```

**Write for sync:**
```typescript
writeFileForSync(filePath: string, content: string): Promise<boolean>
// Creates parent directories recursively
// Validates size limit
```

**Cache reset after write:**
- Settings: `resetSettingsCache()` — forces re-read
- Memory: `clearMemoryFileCaches()` — CLAUDE.md cache

**Internal write marking:**
```typescript
markInternalWrite(filePath)
// Marks file as "internal" so change detector doesn't trigger during startup
// Mid-session changes (e.g., /reload-plugins) caller is responsible
```

### Logging

All operations logged via:
- `logForDiagnosticsNoPII()` — structured diagnostic logs (no PII)
- `logEvent()` — analytics events

**Events:**
- `tengu_settings_sync_upload_skipped_ineligible`
- `tengu_settings_sync_upload_success` { entryCount }
- `tengu_settings_sync_download_success` { entryCount }
- `tengu_settings_sync_download_failed`

### Timeout and Limits

- **Request timeout:** 10 seconds
- **File size limit:** 500 KB per file
- **Max retries (upload background):** Implicit (1 attempt, fire-and-forget)
- **Max retries (download startup):** 3 retries
- **Max retries (manual reload):** 0 retries (user-initiated, let them retry manually)

---

## Layer 4: MDM Integration (macOS/Windows)

**Location:** `/src/utils/settings/mdm/`

**Purpose:** Read enterprise settings from OS-level MDM profiles (macOS) or registry (Windows). Requires admin privileges to set.

### Architecture

Three-file modular design to avoid circular imports:

**`constants.ts`**
- macOS domain: `com.anthropic.claudecode`
- Windows keys: `HKLM\SOFTWARE\Policies\ClaudeCode`, `HKCU\SOFTWARE\Policies\ClaudeCode`
- plist path builder (handles per-user vs device-level)
- Zero heavy imports

**`rawRead.ts`**
- Subprocess I/O only (child_process, fs)
- Fires at module evaluation time (main.tsx)
- No heavy imports

**`settings.ts`**
- Parsing, caching, "first-source-wins" logic

### macOS: plist Paths (Priority Order)

```typescript
getMacOSPlistPaths()
// Returns (highest to lowest priority):
// 1. /Library/Managed Preferences/{username}/{domain}.plist (per-user managed)
// 2. /Library/Managed Preferences/{domain}.plist (device-level managed)
// 3. ~/Library/Preferences/{domain}.plist (ant-only, local testing)
```

**Execution:** `plutil -convert json -o - -- {path}`
- All paths queried in parallel
- First to succeed wins
- Paths that don't exist are skipped (fast-path: no subprocess if file missing)

### Windows: Registry Paths

Two registry locations queried in parallel:

```
HKLM\SOFTWARE\Policies\ClaudeCode\Settings  (REG_SZ, admin-only)
HKCU\SOFTWARE\Policies\ClaudeCode\Settings  (REG_SZ, user-writable, lowest priority)
```

**Note:** Paths live under `SOFTWARE\Policies` (shared list between 32-bit and 64-bit), not `SOFTWARE\ClaudeCode` (would be redirected on 32-bit processes).

**Execution:** `reg query {key} /v Settings`
- Parsed via regex: `Settings\s+REG_(?:EXPAND_)?SZ\s+(.*)`

### First-Source-Wins Logic

```typescript
consumeRawReadResult(raw: RawReadResult)
// 1. If macOS plist found with content → use it, ignore HKCU
// 2. Else if Windows HKLM found with content → use it, ignore HKCU
// 3. Else if managed-settings.json exists with content → use it, ignore HKCU
// 4. Else if Windows HKCU found → use it
// 5. Else return empty
```

Key: An empty/missing high-priority source doesn't fall back. If admin set HKLM explicitly, that's the final decision (even if empty).

### Settings Validation and Error Handling

```typescript
parseCommandOutputAsSettings(stdout: string, sourcePath: string)
// 1. Parse JSON
// 2. Filter invalid permission rules (defense-in-depth)
// 3. Validate against SettingsSchema
// 4. Return { settings, errors }
// Errors don't block application — partial settings applied
```

### Startup Load

```typescript
startMdmSettingsLoad()
// Called as early as possible (main.tsx evaluation)
// Fires subprocess asynchronously
// Results cached, awaited before first settings read

ensureMdmSettingsLoaded()
// Await the in-flight load
// Called before settings pipeline begins
```

### Session Cache

```typescript
getMdmSettings(): MdmResult
// Admin-only MDM (plist/HKLM)

getHkcuSettings(): MdmResult
// User-writable HKCU (lowest priority)

clearMdmSettingsCache()
// Force re-read on next load

setMdmSettingsCache(mdm, hkcu)
// Update cache (used by change detector)
```

### Refresh (Background Polling)

```typescript
refreshMdmSettings()
// Fire fresh subprocess read and parse
// Returns { mdm: MdmResult, hkcu: MdmResult }
// Doesn't update cache (caller decides)
```

Called by change detector every 30 minutes to pick up admin-pushed changes.

### Linux

No MDM equivalent. Falls through to file-based managed settings (`/etc/claude-code/managed-settings.json`).

### Subprocess Timeout

- **Timeout:** 5 seconds (MDM_SUBPROCESS_TIMEOUT_MS)
- **Failure mode:** Empty result, fail-open

### Diagnostic Logging

```typescript
logForDiagnosticsNoPII('info', 'mdm_settings_loaded', {
  duration_ms,
  key_count,
  error_count,
})
```

---

## System Composition and Priority

### Complete Priority Chain

When `getSettings()` is called, layers are merged in this order (highest to lowest):

1. **Remote Managed Settings** — org admin push (API)
   - Fetched during startup with retry
   - Background polling every 60 minutes
   - ETag-based caching with file backup
   - Security dialog for dangerous changes

2. **Admin MDM** — OS-enforced (requires admin privilege)
   - macOS: `/Library/Managed Preferences/{domain}.plist`
   - Windows: `HKLM\SOFTWARE\Policies\ClaudeCode`
   - Subprocess read at startup (5s timeout)
   - Background polling every 30 minutes

3. **File-Based Managed Settings** — IT-deployed files
   - macOS/Linux: `/etc/claude-code/managed-settings.json`
   - Checked before HKCU to implement priority

4. **User HKCU** — Windows user-writable registry (lowest priority)
   - `HKCU\SOFTWARE\Policies\ClaudeCode`
   - Falls back only if no higher source has content

5. **User Preferences** — Local CLI user settings
   - `~/.claude/settings.json`
   - User-created via CLI commands

### Composition in `settings.ts`

```typescript
// Simplified pseudocode
getSettings()
  = mergeSettings(
      remoteManaged,     // Layer 1 (highest)
      mdmSettings,       // Layer 2
      fileManaged,       // Layer 3
      hkcu,              // Layer 4
      userSettings       // Layer 5 (lowest)
    )
```

Each higher layer **completely overrides** lower layers (not partial merge). If a key exists in Remote, it doesn't fall back to MDM.

### Special Case: Eligibility Checks

Eligibility determined once per startup:

```typescript
isRemoteManagedSettingsEligible()   // Cached after first call
isPolicyLimitsEligible()             // Cached after first call
isUsingOAuth()                       // Dynamic (token-based)
```

If a user logs in/out, eligibility is re-evaluated via:
- `refreshPolicyLimits()`
- `refreshRemoteManagedSettings()`

---

## Authentication and Eligibility

### API Key Authentication (Console)

**Format:** `x-api-key: sk-ant-...`

**Eligibility:**
- Policy Limits: All API key users
- Remote Settings: All API key users
- Settings Sync: OAuth only (API key users cannot sync)

**Source:** `getAnthropicApiKeyWithSource()` with `skipRetrievingKeyFromApiKeyHelper: true` to avoid circular dependency with settings.ts

### OAuth Authentication (Claude.ai)

**Format:**
```
Authorization: Bearer {accessToken}
anthropic-beta: claude-code-oauth-v1
```

**Token Source:** `getClaudeAIOAuthTokens()`
- Keychain (macOS), Credential Manager (Windows)
- Environment: `CLAUDE_CODE_OAUTH_TOKEN`, `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`

**Eligibility Tiers:**

| User Type | Policy Limits | Remote Settings | Settings Sync |
|-----------|---------------|-----------------|---------------|
| Free CLI (API key) | Yes | Yes | No |
| Free Claude.ai | No | No | No |
| Team Claude.ai | Yes | Yes | Yes |
| Enterprise Claude.ai | Yes | Yes | Yes |
| Externally-injected (subscriptionType: null) | Only checks API | Let API decide (204/404) | Yes (if OAuth) |

**Scope Required:** `user:inference` (includes all scopes Claude.ai infers)

### Circular Dependency Prevention

**Problem:** settings.ts needs auth.ts to check eligibility. If auth.ts imports settings.ts, circular import.

**Solution:** Three-layer pattern:

1. **`policyLimits/index.ts`** imports auth directly
2. **`remoteManagedSettings/syncCache.ts`** imports auth, holds eligibility check
3. **`remoteManagedSettings/syncCacheState.ts`** (leaf) imports only path, file I/O, types — no auth
4. **`settings.ts`** imports from syncCacheState only (no auth cycle)

Similar pattern used for MDM: rawRead.ts fires subprocess early, before heavy imports.

---

## Error Handling and Resilience

### Fail-Open Philosophy

All enterprise policies are **non-blocking**. Failures never prevent CLI startup:

```typescript
// Policy Limits
if (!result.success) {
  // Use cached or continue without restrictions
  return null
}

// Remote Settings
if (!result.success) {
  // Use cached or continue without settings
  return null
}

// MDM
if (parseError) {
  // Ignore this source, try next layer
  return EMPTY_RESULT
}

// Settings Sync
if (!result.success) {
  // Fail-open, CLI starts without synced settings
  return false
}
```

### Retry Logic

**Pattern:**
```typescript
for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
  const result = await fetch()
  if (result.success) return result
  if (result.skipRetry) return result  // Auth error
  if (attempt > maxRetries) return result
  const delay = getRetryDelay(attempt)
  await sleep(delay)
}
```

**Exponential backoff:** `getRetryDelay()` from `api/withRetry.ts`

**Retry Counts:**
- Policy Limits: 5 retries
- Remote Settings: 5 retries
- Settings Sync (upload background): Fire-and-forget (implicit 1 attempt)
- Settings Sync (download startup): 3 retries
- Settings Sync (manual reload): 0 retries
- MDM subprocess: No retry (5s timeout)

**Non-Retryable Errors:**
- 401/403 Auth errors (`skipRetry: true`)
- Malformed responses (schema validation failure)

**Retryable Errors:**
- Network timeout
- Connection refused
- Temporary server errors

### HTTP Status Code Handling

**Policy Limits:**
- 200 OK → Apply new restrictions
- 304 Not Modified → Use cached version
- 404 Not Found → No restrictions (return {})
- Other → Retry

**Remote Settings:**
- 200 OK → Apply new settings
- 204 No Content → No settings (return {})
- 304 Not Modified → Use cached version
- 404 Not Found → No settings (return {})
- Other → Retry

**Settings Sync:**
- 200 OK → Data present
- 404 Not Found → No data yet (isEmpty: true)
- Other → Retry

### Validation Errors

**Settings validation failure:**
```typescript
const validation = SettingsSchema().safeParse(data)
if (!validation.success) {
  // Log error, don't apply partial settings
  return { success: false, error: 'Invalid schema' }
}
```

**Permission rules validation:**
```typescript
const ruleWarnings = filterInvalidPermissionRules(data, sourcePath)
// Filters bad rules before schema validation
// Allows partial application (e.g., 2/3 rules valid)
```

### Diagnostic Logging

All systems log via `logForDiagnosticsNoPII()` and `logEvent()`:

**Policy Limits:**
- `mdm_settings_loaded` (if found)

**Remote Settings:**
- `tengu_managed_settings_security_dialog_shown`
- `tengu_managed_settings_security_dialog_accepted`
- `tengu_managed_settings_security_dialog_rejected`

**Settings Sync:**
- `tengu_settings_sync_upload_skipped_ineligible`
- `tengu_settings_sync_upload_success`
- `tengu_settings_sync_upload_failed`
- `tengu_settings_sync_download_success`
- `tengu_settings_sync_download_failed`

---

## Caching Strategies

### HTTP Caching (ETag)

**Pattern:** If-None-Match with SHA256 checksum

**Computation (must be deterministic):**
```typescript
function computeChecksum(data: unknown): string {
  const sorted = sortKeysDeep(data)
  const json = jsonStringify(sorted)  // Compact separators
  const hash = createHash('sha256').update(json).digest('hex')
  return `sha256:${hash}`
}
```

**Validation:**
- Server must use **identical** normalization (Python: `json.dumps(sort_keys=True, separators=(",", ":")`)
- Any JSON formatting difference → different checksum → unnecessary re-fetch

**304 Response:**
- Server: checksum matches
- Client: Use cached version (local + session)
- Bandwidth saved: Fetch headers only, no body

**Uses:**
- Policy Limits: Policy checksum caching
- Remote Settings: Settings checksum caching
- Settings Sync: Metadata only (no ETag, uses version/lastModified)

### File-Based Caching

**Policy Limits:**
- File: `~/.claude/policy-limits.json`
- Permissions: 0o600
- Format: `{ restrictions: { [policy]: { allowed: boolean } } }`
- Fallback: Used if fetch fails and cache exists (graceful degradation)

**Remote Settings:**
- File: `~/.claude/remote-settings.json`
- Permissions: 0o600
- Format: Raw `SettingsJson` object
- Fallback: Used if fetch fails and cache exists

**Settings Sync:**
- No file cache (uses remote as source of truth)
- Local files are the cache (upload pulls changes from them)

### Session-Level Caching

**In-Memory Cache (Reset on Load Failure):**

```typescript
let sessionCache: SettingsJson | null = null

// Loaded from file/API, kept in memory for fast access
// Reset by settingsChangeDetector.notifyChange()
// Survives transient fetch failures (can fall back to disk)
```

**Eligibility Caching:**
```typescript
let cached: boolean | undefined  // syncCache.ts

isRemoteManagedSettingsEligible()
// Computed once, cached for startup
// Re-evaluated on auth change
```

### Settings Cache (`settingsCache.ts`)

Central cache for merged settings object (across all layers):

```typescript
resetSettingsCache()
// Called when any layer changes (file, remote, MDM, sync)
// Forces re-merge on next getSettings() call
```

**Notified by:**
- Remote Settings fetch completes
- Settings Sync applies files
- MDM change detected
- User settings file modified

---

## Background Polling and Hot Reload

### Polling Intervals

| Source | Interval | Purpose |
|--------|----------|---------|
| Policy Limits | 60 minutes | Pick up policy changes mid-session |
| Remote Settings | 60 minutes | Pick up setting changes mid-session |
| MDM (change detector) | 30 minutes | Pick up admin changes mid-session |

### Change Detection Mechanism

**`settingsChangeDetector.notifyChange('policySettings')`**

Fired when:
- Remote settings fetch completes (diff detected)
- Policy limits fetch completes (diff detected)
- Settings Sync applies changes
- MDM change detected (via `refreshMdmSettings()`)

**Effect:**
1. Resets merged settings cache (`resetSettingsCache()`)
2. Calls listeners (env vars updated, permissions re-checked, telemetry reconfigured)
3. If running in interactive mode, triggers display update

### Policy Limits Polling

```typescript
startBackgroundPolling()
// setInterval(() => void pollPolicyLimits(), POLLING_INTERVAL_MS)
// pollingIntervalId.unref() — doesn't keep process alive

async function pollPolicyLimits() {
  const previousCache = sessionCache
  await fetchAndLoadPolicyLimits()  // Retry logic + caching
  if (jsonStringify(previousCache) !== jsonStringify(sessionCache)) {
    logForDebugging('Policy limits: Changed during background poll')
  }
}

stopBackgroundPolling()
// clearInterval(pollingIntervalId)
// Registered in cleanup() for graceful shutdown
```

### Remote Settings Polling

```typescript
startBackgroundPolling()
// setInterval(() => void pollRemoteSettings(), POLLING_INTERVAL_MS)

async function pollRemoteSettings() {
  const prevCache = getRemoteManagedSettingsSyncFromCache()
  await fetchAndLoadRemoteManagedSettings()  // Same fetch logic
  const newCache = getRemoteManagedSettingsSyncFromCache()
  if (jsonStringify(prevCache) !== jsonStringify(newCache)) {
    logForDebugging('Remote settings: Changed during background poll')
    settingsChangeDetector.notifyChange('policySettings')
  }
}
```

### MDM Change Detection

**Change Detector** (`changeDetector.ts`, not fully analyzed):
- Polls every 30 minutes via `refreshMdmSettings()`
- Compares new vs cached
- If different, updates cache and notifies listeners

### Failures Don't Stop Polling

```typescript
catch {
  // Don't fail closed for background polling
  // Errors logged but swallowed; timer continues
}
```

Background polling failures don't block the CLI or cascade into service degradation.

---

## Summary of Key Design Patterns

### 1. Non-Blocking Initialization

All async systems (remote settings, policy limits, MDM) create a promise early:
```typescript
initializeRemoteManagedSettingsLoadingPromise()
```

This allows other systems to `await` before using settings without risking deadlocks.

### 2. Dual-Mode Fetching

- **Startup:** With retries (resilient, may take ~100ms)
- **Background polling:** Fire-and-forget, no retry (non-blocking)
- **Manual reload:** No retry, let user retry (interactive command)

### 3. Checksum-Based Caching

Minimize bandwidth via SHA256 checksums. Works because:
- Client computes checksum from cached file
- Server returns 304 if match
- Only new/changed settings transferred

### 4. First-Source-Wins

Highest-priority source that has content wins completely:
```
if (remote has settings) → use remote only
else if (admin MDM has settings) → use MDM only
else if (file has settings) → use file only
else → fall to lower layers
```

Prevents unintended layering where empty sources cause fallback.

### 5. Security via Dialog

Dangerous settings (e.g., disabling privacy) require explicit user approval:
```typescript
checkManagedSettingsSecurity()
// Shows blocking dialog if dangerous changes detected
// User can reject, causes gracefulShutdownSync(1)
```

### 6. Circular Dependency Breaking

Use three-layer pattern:
- Leaf module (imports only path, file, types)
- Wrapper module (imports auth, holds eligibility)
- Main module (orchestrates logic)

### 7. Graceful Degradation

Every failure mode has a fallback:
- Fetch fails → use cached version
- Cache missing → continue without settings (fail-open)
- Validation fails → use next layer
- Subprocess timeout → empty result (fail-open)

### 8. Metadata Logging

All operations logged via `logForDiagnosticsNoPII()` (no credentials, API keys, PII):
- Success/failure
- Timing info (duration_ms)
- Counts (entryCount, key_count)
- Sources (which MDM path succeeded)

---

## Files and Locations

### Policy Limits
- `/src/services/policyLimits/types.ts` — 28 lines
- `/src/services/policyLimits/index.ts` — 664 lines

### Remote Managed Settings
- `/src/services/remoteManagedSettings/types.ts` — 32 lines
- `/src/services/remoteManagedSettings/index.ts` — 639 lines
- `/src/services/remoteManagedSettings/syncCache.ts` — 113 lines
- `/src/services/remoteManagedSettings/syncCacheState.ts` — 97 lines
- `/src/services/remoteManagedSettings/securityCheck.tsx` — 74 lines

### Settings Sync
- `/src/services/settingsSync/types.ts` — 68 lines
- `/src/services/settingsSync/index.ts` — 582 lines

### MDM Integration
- `/src/utils/settings/mdm/constants.ts` — 82 lines
- `/src/utils/settings/mdm/rawRead.ts` — 131 lines
- `/src/utils/settings/mdm/settings.ts` — 317 lines

**Total:** ~2,839 lines across 12 files

---

## Conclusion

Claude Code's settings and policy infrastructure is a production-grade enterprise system designed to balance organizational control with user autonomy and reliability. Key achievements:

1. **Resilience:** Fail-open throughout; network failures don't prevent CLI operation
2. **Performance:** ETag caching, cache-first bootstrap, parallel subprocess spawning
3. **Security:** Dangerous setting dialogs, checksum validation, file permissions (0o600)
4. **Observability:** Comprehensive diagnostic logging without PII leakage
5. **Flexibility:** Pluggable priority layers (Remote > MDM > File > HKCU > User)
6. **Maintainability:** Clear separation of concerns (constants → raw I/O → parsing → caching)

The system is well-engineered for organizations deploying Claude Code at scale, with fine-grained policy control, secure settings distribution, and graceful fallbacks for any failure scenario.
