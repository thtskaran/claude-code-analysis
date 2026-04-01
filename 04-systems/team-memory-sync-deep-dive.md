# Reverse Engineering: Claude Code Team Memory Sync System

## Executive Summary

This is Anthropic's **per-repository, cross-team memory synchronization system** that shares development context (Markdown notes, patterns, decisions) across all authenticated organization members. The architecture uses **optimistic locking with delta uploads, conflict-free merging via checksums, and client-side secret scanning**. Memory is scoped to git repositories and shared across all org members with OAuth access.

---

## 1. TEAM MEMORY DATA STRUCTURE

### What IS a "Team Memory"?

**Location**: Team memory is stored in `.claudeai/memory/` directory locally, scoped per-repository on the server.

**Format**: A flat key-value store with optional per-key SHA-256 checksums. Keys are relative file paths (e.g., "MEMORY.md", "patterns.md", "subdirs/notes.md").

### Data Schema (from types.ts)

```typescript
// Server response structure
TeamMemoryData {
  organizationId: string                        // Org owning the memory
  repo: string                                  // GitHub repo slug (owner/repo)
  version: number                               // Monotonic version counter
  lastModified: string                          // ISO 8601 timestamp
  checksum: string                              // SHA256 with 'sha256:' prefix
  content: {
    entries: Record<string, string>             // Key→UTF-8 content map
    entryChecksums?: Record<string, string>     // Optional: Per-key SHA256 hashes
  }
}

// Error response for too-many-entries (413)
TeamMemoryTooManyEntriesSchema {
  error: {
    details: {
      error_code: 'team_memory_too_many_entries'
      max_entries: number                       // Server-enforced cap (GB-tunable)
      received_entries: number                  // How many were sent
    }
  }
}
```

### Per-File Size Limits

- **MAX_FILE_SIZE_BYTES = 250,000 bytes** per entry (pre-checked client-side)
- **MAX_PUT_BODY_BYTES = 200,000 bytes** per PUT request (batching splits large deltas)
- **Gateway hard limit**: ~256-512 KB for entire PUT body
- Files exceeding limits are **silently skipped** with debug logging

### Entry Cap (Per-Organization)

- Server-enforced `max_entries` is **GB-tunable per-org** (no compile-time client default)
- First learned from a 413 response's `extra_details.max_entries`
- Cached in `SyncState.serverMaxEntries` for subsequent pushes
- Alphabetically-last files that exceed the cap **never sync**

---

## 2. SYNC STATE (Persistent Client-Side State)

Defined in `index.ts`:

```typescript
export type SyncState = {
  // Last known server checksum (ETag value)
  lastKnownChecksum: string | null

  // Per-key content hashes of server state
  // Map<string, string> where string = 'sha256:<hex>'
  serverChecksums: Map<string, string>

  // Server-enforced max_entries cap (null until first 413)
  serverMaxEntries: number | null
}
```

**Created once per session** by the watcher (`createSyncState()`), threaded through all sync operations. **No module-level mutable state** — tests get fresh instances per test for isolation.

---

## 3. SYNC MECHANISM: Push/Pull/Conflict Resolution

### 3.1 PULL (Server → Local)

**Endpoint**: `GET /api/claude_code/team_memory?repo={owner/repo}`

**Semantics**:
- Server entries **fully overwrite** local files (last-write-wins by server)
- Uses ETag-based conditional requests (304 Not Modified optimizes repeat pulls)
- Returns 404 if server has no data yet (fresh repo)

**Flow**:
1. `pullTeamMemory(state)` with optional `skipEtagCache` flag
2. If ETag matches (`state.lastKnownChecksum`), returns 304 → **no disk writes**
3. If 200 response:
   - Parses `content.entries` (UTF-8 strings)
   - **Validates every path** against team memory boundary (`validateTeamMemKey()`)
   - **Skips oversized entries** (> 250 KB)
   - Skips entries whose on-disk content already matches (preserves mtime)
   - Writes via parallel `Promise.all()` — safe concurrent `mkdir` with `recursive: true`
   - Clears stale `serverChecksums` on fresh pull
   - Returns count of **actually written** files (not all returned entries)

**Checksum Refresh**:
- Requires backend PR #283027 (entryChecksums in response)
- If missing, `serverChecksums` stays empty and next push becomes full (non-delta)
- Self-corrects on next successful push

### 3.2 PUSH (Local → Server)

**Endpoint**: `PUT /api/claude_code/team_memory?repo={owner/repo}` with optional `If-Match` header

**Semantics**:
- **Delta upload only** — compares local file hashes against `serverChecksums`
- Only keys with differing hashes are uploaded
- Server uses **upsert**: keys not in PUT are preserved
- File deletions **do NOT propagate** (deleting local file won't remove from server)
- Next pull restores deleted file locally

**Flow**:
1. `pushTeamMemory(state)` reads local directory once at start
2. **Secret scanning** (PSR M22174) happens here:
   - Uses gitleaks rule patterns (high-confidence only)
   - Files containing detected secrets are **completely skipped** from upload
   - Detected rule IDs and file paths logged, secret VALUES never logged
3. Compute local content hashes (SHA256, `sha256:<hex>` format)
4. Build delta: keys where `localHash !== serverChecksums[key]`
5. If delta empty → returns success (0 files uploaded)
6. **Batch the delta** into PUT-sized chunks (greedy bin-packing by byte size)
   - Deterministic batching via alphabetical sort
   - Each batch independently upserted on server
   - Updates `state.lastKnownChecksum` ETag chain through batches
7. **Upload each batch sequentially** with `If-Match: "<lastKnownChecksum>"` header
8. On **batch success**: update `serverChecksums` for those keys, continue
9. On **batch failure**: return immediately with partial count in `filesUploaded`

### 3.3 CONFLICT RESOLUTION (412 Precondition Failed)

**Trigger**: ETag mismatch — server state changed since our last pull/successful push

**Strategy**: **Local-wins-on-conflict**. Opposite of pull semantics intentionally.
- When user is actively editing locally, silently discarding their edit is unacceptable
- Content-level merge (same key, both sides changed) is NOT attempted
- Local version simply overwrites the server version for that key

**Conflict Loop** (max 2 retries, so 3 total attempts):
1. Upload delta with `If-Match: lastKnownChecksum` → 412 response
2. **Cheap probe**: `GET ?view=hashes` — fetch **only** per-key checksums, no bodies
   - Avoids downloading ~300 KB of content just to see what changed
   - Requires backend PR #283027
3. **Refresh** `serverChecksums` from probe response
4. **Recompute delta** — naturally excludes keys where teammate's push matches ours
5. **Retry** the push with tighter delta
6. After 2 conflict retries, **give up** and fail the push
7. Watcher retries on next local edit

**No merge, no disk writes during conflict resolution** — server-only keys from teammate's concurrent push propagate on next pull.

---

## 4. API ENDPOINTS & AUTHENTICATION

### Endpoints

```
GET  /api/claude_code/team_memory?repo={owner/repo}
     Returns: TeamMemoryData (200), 404 (no data), 304 (not modified)
     Caching: If-None-Match header with ETag from lastKnownChecksum

GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes
     Returns: {version, checksum, entryChecksums} (200), 404
     Purpose: Conflict resolution probe (no entry bodies)

PUT  /api/claude_code/team_memory?repo={owner/repo}
     Body: {"entries": {key: content, ...}}
     Response: TeamMemorySyncUploadResult with new checksum
     Conditional: If-Match header with lastKnownChecksum
     Errors: 412 (conflict), 413 (too many entries), 403/401 (auth)
```

### Authentication

**Required**: First-party Anthropic OAuth with specific scopes

```typescript
function isUsingOAuth(): boolean {
  return (
    getAPIProvider() === 'firstParty' &&
    isFirstPartyAnthropicBaseUrl() &&
    tokenHasScopes([
      CLAUDE_AI_INFERENCE_SCOPE,    // Required
      CLAUDE_AI_PROFILE_SCOPE        // Required
    ])
  )
}
```

**Headers**:
```
Authorization: Bearer {accessToken}
anthropic-beta: {OAUTH_BETA_HEADER}
User-Agent: {getClaudeCodeUserAgent()}
```

**Token Refresh**: `checkAndRefreshOAuthTokenIfNeeded()` called before every request

**Base URL**: `process.env.TEAM_MEMORY_SYNC_URL` or `getOauthConfig().BASE_API_URL`

### HTTP Status Codes & Semantics

| Status | Meaning | Retryable |
|--------|---------|-----------|
| 200 | Success | — |
| 304 | Not Modified (ETag match) | — |
| 404 | No data exists yet | No |
| 412 | ETag mismatch (conflict) | Yes (via probe) |
| 413 | Too many entries or too large | No (capped) |
| 403 | Forbidden (permission denied) | No |
| 401 | Unauthorized (auth failure) | No |
| 429 | Rate limited | Yes (backoff) |
| 5xx | Server error | Yes (exponential backoff) |

---

## 5. SYNC PROTOCOL & CONFLICT RESOLUTION

### Push vs. Pull Conflict Semantics

| Operation | Conflict Handling | Merge Strategy |
|-----------|-------------------|-----------------|
| **Pull** | Server wins | Full overwrite per-key |
| **Push** | Local wins | Delta + probe + retry |
| **Sync** | Pull first (server), then push (local overwrite) | Sequential |

### Conflict Resolution Protocol

```
User edits local file
     ↓
pushTeamMemory() reads disk, builds delta
     ↓
PUT delta with If-Match: lastKnownChecksum
     ↓
[412 Conflict]
     ↓
GET ?view=hashes (cheap probe, no bodies)
     ↓
Refresh serverChecksums from probe
     ↓
Recompute delta (teammate's keys excluded)
     ↓
PUT new delta (retry)
     ↓
[Success] → Update serverChecksums, return
[412 again] → Repeat probe & retry (max 2 times)
[Non-412 failure] → Give up, return error
```

### Why This Works

1. **Deterministic**: Same set of retries produce same batches (alphabetical sort)
2. **Converges**: After probe, overlapping changes are excluded from delta
3. **Atomic**: Server upsert is atomic per PUT — partial batch failures leave disk in usable state
4. **Lost Update Prevention**: ETag ensures we see what changed, not just lose our changes

---

## 6. SECRET SCANNING & DATA PROTECTION (PSR M22174)

### Client-Side Secret Scanner

**Location**: `secretScanner.ts`

**Scanning Method**:
- 44 high-confidence gitleaks rules (curated subset)
- Only rules with distinctive prefixes (near-zero false-positives)
- Generic keyword-context rules omitted
- Rules include: AWS, GitHub, OpenAI, Slack, Anthropic API keys, private keys, etc.

**Rule List** (from secretScanner.ts):
```
AWS:              aws-access-token
GCP:              gcp-api-key
Azure:            azure-ad-client-secret
DigitalOcean:     digitalocean-pat, digitalocean-access-token

AI APIs:
  Anthropic:      anthropic-api-key, anthropic-admin-api-key
  OpenAI:         openai-api-key
  HuggingFace:    huggingface-access-token

GitHub:           github-pat, github-fine-grained-pat, github-app-token,
                  github-oauth, github-refresh-token
GitLab:           gitlab-pat, gitlab-deploy-token

Slack:            slack-bot-token, slack-user-token, slack-app-token
Twilio:           twilio-api-key
SendGrid:         sendgrid-api-token

npm:              npm-access-token
PyPI:             pypi-upload-token
Databricks:       databricks-api-token
HashiCorp TF:     hashicorp-tf-api-token
Pulumi:           pulumi-api-token
Postman:          postman-api-token

Grafana:          grafana-api-key, grafana-cloud-api-token,
                  grafana-service-account-token
Sentry:           sentry-user-token, sentry-org-token

Stripe:           stripe-access-token
Shopify:          shopify-access-token, shopify-shared-secret

Private Keys:     private-key (PEM format)
```

### Secret Detection Flow

1. **scanForSecrets(content: string)** → returns array of `SecretMatch[]`
   - One match per rule that fired (deduplicated by rule ID)
   - **Never returns matched text** — only rule ID and label
   - Lazily compiles patterns on first scan

2. **During push**:
   - Each local file scanned before inclusion in entries
   - If any secret detected: file **completely skipped** from upload
   - `skippedSecrets` array collects `{path, ruleId, label}`
   - Secret VALUES never logged, only file paths and rule IDs
   - User receives warning with file list and secret types

3. **During file write** (FileWriteTool/FileEditTool):
   - `checkTeamMemSecrets()` validates before write
   - Prevents model from writing secrets into team memory
   - Error message: "Content contains potential secrets (...) and cannot be written to team memory"

4. **Secret Prefix Assembly**:
   - Anthropic key prefix assembled at runtime (`ANT_KEY_PFX = ['sk', 'ant', 'api'].join('-')`)
   - Literal byte sequence not in external bundle (excluded-strings check)

### Redaction (Optional)

```typescript
redactSecrets(content: string): string
  // Replace matched secrets with [REDACTED]
  // Only replaces captured group, preserves boundary chars
```

---

## 7. FILE WATCHER & AUTOMATIC SYNC

**Location**: `watcher.ts`

### Architecture

Uses **fs.watch({recursive: true})** on directory level, not chokidar:
- **Why not chokidar?** v4+ dropped fsevents, fallback kqueue requires 1 fd per file
- With 500+ team memory files → 500+ open fds (verified via lsof)
- fs.watch on macOS uses FSEvents → O(1) fds regardless of tree size (2 fds for 60 files)
- On Linux uses inotify → O(subdirs) fds (still fine)

### Watcher State Machine

```typescript
let watcher: FSWatcher | null = null
let debounceTimer: ReturnType<typeof setTimeout> | null = null
let pushInProgress: boolean = false
let hasPendingChanges: boolean = false
let currentPushPromise: Promise<void> | null = null
let watcherStarted: boolean = false
let pushSuppressedReason: string | null = null  // Permanent failure tracking
```

### Lifecycle

```
startTeamMemoryWatcher()
  ↓
[Check feature flag, auth, git remote]
  ↓
createSyncState()
  ↓
Initial pull from server (before watcher starts)
  ↓
Start fs.watch({recursive: true, persistent: true})
  ↓
Log telemetry event: tengu_team_mem_sync_started
```

### Push Suppression Logic

**Permanent failures** stop the watcher from retrying on every edit:

```typescript
function isPermanentFailure(r: TeamMemorySyncPushResult): boolean {
  if (r.errorType === 'no_oauth' || r.errorType === 'no_repo') return true
  if (r.httpStatus >= 400 && r.httpStatus < 500 &&
      r.httpStatus !== 409 && r.httpStatus !== 429) {
    return true  // 4xx except 409 (transient conflict) and 429 (rate limit)
  }
  return false
}
```

**When suppressed**:
- `fs.watch` fires but `schedulePush()` returns early
- Suppression clears on **file unlink** (recovery path for too-many-entries)
- Clears on session restart

**Example**: User hits 413 too-many-entries, deletes files, suppression clears, next edit retries.

### Debounce Logic

- **DEBOUNCE_MS = 2000** (2 second wait after last file change)
- Multiple writes coalesce into single push
- If push in progress when debounce fires: reschedule debounce
- Ensures 2s+ quiet period before uploading

### Explicit Notification

```typescript
notifyTeamMemoryWrite()
  // Called from PostToolUse hooks when Claude writes to team memory
  // Explicitly schedules push in case fs.watch misses the write
  // (file written same tick watcher starts, rapid successive writes)
```

### Shutdown

```typescript
stopTeamMemoryWatcher()
  // Clear debounce timer
  // Close watcher
  // Await in-flight push
  // Flush any pending (debounced but not yet pushed) changes
  // Within 2s graceful shutdown budget (best-effort)
```

---

## 8. LOCAL STORAGE & FILE OPERATIONS

### Directory Structure

```
.claudeai/memory/              # Team memory root
  ├── MEMORY.md               # Example: flat file
  ├── patterns.md
  ├── decisions/              # Subdirectories supported
  │   └── 2024-arch.md
  └── patterns/
      └── react-hooks.md
```

### Key Validation

**Function**: `validateTeamMemKey(relPath: string)`
- Validates path is within team memory boundary (no `../` traversal)
- Returns validated absolute path
- Throws `PathTraversalError` on invalid paths
- Relative paths use forward slashes (normalized via `.replaceAll('\\', '/')`)

### Read Operations

```typescript
async function readLocalTeamMemory(maxEntries: number | null): Promise<{
  entries: Record<string, string>
  skippedSecrets: SkippedSecretFile[]
}>
```

**Walk**:
- Recursive walk of team memory directory
- Parallel reads via `Promise.all()`
- Skips unreadable files (ENOENT, EACCES, EPERM ignored)

**Per-file processing**:
1. Stat file → skip if size > 250 KB
2. Read as UTF-8
3. Scan for secrets (PSR M22174)
4. If secret found: add to `skippedSecrets`, skip file entirely
5. Add to entries map

**Truncation** (if serverMaxEntries learned):
- Alphabetically sort all keys
- If count > maxEntries: drop alphabetically-last files
- Log warning + analytics event
- Deterministic order ensures same N keys sync on retry

### Write Operations

```typescript
async function writeRemoteEntriesToLocal(
  entries: Record<string, string>
): Promise<number>  // Returns count actually written
```

**Per-entry processing**:
1. Validate path against boundary
2. Stat target → skip if size > 250 KB
3. Read existing content → skip if identical (preserves mtime)
4. Create parent directories (parallel-safe with `recursive: true`)
5. Write UTF-8 content
6. Return boolean (wrote or skipped)

**Parallel**: Each entry independently validates, reads, and writes. Safe concurrent `mkdir`.

---

## 9. HASHING & CHECKSUM INTEGRITY

### Hash Function

```typescript
export function hashContent(content: string): string {
  return 'sha256:' + createHash('sha256').update(content, 'utf8').digest('hex')
}
```

**Format**: `sha256:<hex>` (matches server's entryChecksums format)

### Checksum Usage

1. **ETag** (`lastKnownChecksum`):
   - Full-dataset checksum
   - Used in `If-Match` / `If-None-Match` headers
   - Sent as `"<hex>"` in header (quotes stripped on read)

2. **Per-entry checksums** (`serverChecksums: Map<string, string>`):
   - One hash per key
   - Used to compute delta on push
   - Populated from server's `content.entryChecksums` on pull
   - Updated locally after successful push

3. **Delta computation**:
   ```typescript
   for (const [key, localHash] of localHashes) {
     if (state.serverChecksums.get(key) !== localHash) {
       delta[key] = entries[key]!  // Include in upload
     }
   }
   ```

---

## 10. RATE LIMITING & QUOTAS

### Per-Request Limits

| Limit | Value | Notes |
|-------|-------|-------|
| **MAX_FILE_SIZE_BYTES** | 250,000 bytes | Per entry |
| **MAX_PUT_BODY_BYTES** | 200,000 bytes | Per PUT request (soft cap) |
| **MAX_RETRIES** | 3 | For transient failures |
| **MAX_CONFLICT_RETRIES** | 2 | Total 3 conflict attempts |
| **DEBOUNCE_MS** | 2,000 ms | Wait time after last file change |
| **TEAM_MEMORY_SYNC_TIMEOUT_MS** | 30,000 ms | HTTP request timeout |

### Entry Cap (Per-Organization)

- **Server-enforced** `max_entries` is GB-tunable per-org
- First learned from structured 413 response
- Cached in `state.serverMaxEntries` for next push
- Pre-client-side truncation during push if known

### Retry Strategy

**Exponential backoff** (from `getRetryDelay(attempt)` in withRetry.js):
- Transient failures (timeout, network, 5xx) retry up to 3 times
- Non-transient (4xx except 409/429) skip retry
- 429 (rate limit) retries with backoff

---

## 11. TEAM MEMBERSHIP & AUTHORIZATION

### Who Can Access Team Memory?

- **All authenticated org members** with valid OAuth tokens
- Scoped to **specific Git repository** (owner/repo)
- No per-user or per-team access control (all-or-nothing at org level)

### Scope Checking

```typescript
function isTeamMemorySyncAvailable(): boolean {
  return (
    getAPIProvider() === 'firstParty' &&
    isFirstPartyAnthropicBaseUrl() &&
    getClaudeAIOAuthTokens()?.accessToken &&
    tokens.scopes.includes(CLAUDE_AI_INFERENCE_SCOPE) &&
    tokens.scopes.includes(CLAUDE_AI_PROFILE_SCOPE)
  )
}
```

### What Happens on Auth Failure?

- **no_oauth** error type → **permanent suppression** of watcher retries
- User must restart with valid OAuth
- **no_repo** error type → repos without github.com remote can't sync (skipped early)

---

## 12. ERROR HANDLING & FAILURE MODES

### Error Classification (from classifyAxiosError)

```typescript
type ErrorKind = 'auth' | 'timeout' | 'network' | 'http' | 'other'
```

### Error Types in Results

**Push result**:
```typescript
type TeamMemorySyncPushResult = {
  success: boolean
  filesUploaded: number
  error?: string
  errorType?: 'auth' | 'timeout' | 'network' | 'conflict' | 'unknown' | 'no_oauth' | 'no_repo'
  httpStatus?: number
  conflict?: boolean  // 412
  skippedSecrets?: SkippedSecretFile[]
  serverErrorCode?: 'team_memory_too_many_entries'
  serverMaxEntries?: number
  serverReceivedEntries?: number
}
```

**Fetch result**:
```typescript
type TeamMemorySyncFetchResult = {
  success: boolean
  data?: TeamMemoryData
  isEmpty?: boolean  // 404
  notModified?: boolean  // 304
  error?: string
  errorType?: 'auth' | 'timeout' | 'network' | 'parse' | 'unknown'
  httpStatus?: number
  skipRetry?: boolean  // Don't retry this error
}
```

### Partial Batch Failures

When pushing multiple batches:
- If batch N fails: batches 1..N-1 already committed server-side
- `filesUploaded` reflects partial success
- `serverChecksums` updated only for committed batches
- Push returns failure, but disk state is consistent

### Corrupt Data & Version Mismatches

**Parsing failures**:
- Response validation via Zod schemas
- Invalid response format → `skipRetry: true` (don't retry malformed)
- Old server (missing `entryChecksums`) handled gracefully (full sync next push)

**Path traversal attacks**:
- Every remote path validated via `validateTeamMemKey()`
- Throws `PathTraversalError` if `../` detected
- Remote entries with traversal attempts skipped, logged as warn

---

## 13. TELEMETRY & ANALYTICS

### Events Logged

```typescript
// Session start
'tengu_team_mem_sync_started' {
  initial_pull_success: boolean
  initial_files_pulled: number
  watcher_started: boolean  // Always true
  server_has_content: boolean
}

// Pull completion
'tengu_team_mem_sync_pull' {
  success: boolean
  files_written: number
  not_modified: boolean
  duration_ms: number
  errorType?: string
  status?: number
}

// Push completion
'tengu_team_mem_sync_push' {
  success: boolean
  files_uploaded: number
  conflict: boolean
  conflict_retries: number
  duration_ms: number
  put_batches?: number
  errorType?: string
  status?: number
  error_code?: string  // e.g., 'team_memory_too_many_entries'
  server_max_entries?: number
  server_received_entries?: number
}

// Suppression of watcher retries
'tengu_team_mem_push_suppressed' {
  reason: string  // 'no_oauth', 'http_403', etc.
  status?: number
}

// Secret detection
'tengu_team_mem_secret_skipped' {
  file_count: number
  rule_ids: string  // Comma-joined, e.g. "github-pat,aws-access-token"
}

// Entry cap hit
'tengu_team_mem_entries_capped' {
  total_entries: number
  dropped_count: number
  max_entries: number
}
```

### Datadog Filtering

- Can filter by `@error_code:team_memory_too_many_entries`
- Can filter by `@errorType:conflict`, `@errorType:auth`, etc.
- Duration tracking helps identify slow syncs

---

## 14. SECURITY ANALYSIS

### Cross-Tenant Data Leakage

**Risk**: Team memory from Org A leaks to Org B

**Mitigation**:
- Server enforces **per-org scope** via OAuth tokens
- Client never allows cross-org team memory
- Team memory scoped to `organizationId` + `repo`
- Validated early: `isUsingOAuth()` checks scopes

**Remaining Risk**: If OAuth token from Org A is used by Org B user, team memory could leak. Mitigated by:
- OAuth token issued per user per org
- Token refresh validates scopes on every request

### Memory Poisoning

**Risk**: Malicious server sends harmful data (e.g., XSS payloads in JSON)

**Mitigation**:
- All remote entries are **written as UTF-8 text files**, not parsed/executed
- Zod schema validation on parse before any use
- Path traversal validation (`validateTeamMemKey`)
- No eval(), no code injection possible
- Content is only read by Claude for context (not interpreted as code)

### Unauthorized Memory Access

**Risk**: User without access to repo reads team memory

**Mitigation**:
- OAuth check: `isTeamMemorySyncAvailable()` before any sync
- GitHub repo validation: `getGithubRepo()` confirms github.com remote exists
- Server enforces per-repo ACL on backend
- No team memory URL is exposed; endpoints require OAuth

### PII in Shared Memories

**Risk**: User accidentally commits PII (names, emails, phone numbers) to team memory

**Mitigation**:
- Client-side secret scanning blocks known credential patterns
- gitleaks rules don't match generic PII (by design, to avoid false positives)
- No PII scanning on client side (would be noisy)
- Server-side: no additional validation documented here

**Recommendation**: Document that team memory is **shared across org** and warn users not to add PII.

### Secret Poisoning

**Risk**: User commits real API keys to team memory

**Mitigation**:
- `scanForSecrets()` scans every local file before upload
- 44 high-confidence gitleaks rules (very low false-positive rate)
- Files with detected secrets are **completely skipped** from upload
- Secrets never leave the user's machine
- User receives warning with file paths and secret types
- Secret VALUES never logged (only rule IDs and paths)

**Limitation**: Secrets not matching any rule pass through (e.g., custom API keys, weak passwords).

### Downgrade Attacks

**Risk**: Attacker downgrades client to older version without secret scanning

**Mitigation**:
- Secret scanning is client-side only (no server validation)
- If client is downgraded or disabled, secrets could be uploaded
- Recommended: Server-side secret scanning as defense-in-depth

### Offline/Disconnected State

**Risk**: Local edits made offline are lost if conflict on next connect

**Mitigation**:
- All local edits are **persisted to disk** before push attempted
- Push failure doesn't delete local files
- On reconnect, watcher triggers push
- If 412 conflict: local version wins (user's edit survives)

**No automatic reconciliation**: If both user and teammate edit same file offline, local user's version overwrites on push (conflict resolution via local-wins).

### Gateway Body-Size Bypass

**Risk**: Client bypasses MAX_PUT_BODY_BYTES soft limit, hits gateway hard reject

**Mitigation**:
- Batching divides large deltas into multiple sequential PUTs
- Each batch carefully sized under 200 KB (soft cap)
- Large single entries (up to 250 KB) are allowed in solo batches
- Gateway rejects ~256-512 KB; 200 KB + 250 KB entry = ~450 KB, just under limit
- If batching fails to prevent: push fails with unstructured 413 (distinguishable from structured app 413 by latency ~750ms)

---

## 15. ARCHITECTURE INSIGHTS

### Why This Design?

1. **Optimistic Locking (ETag)**:
   - Avoids round-trip latency of pessimistic locks (request → acquire → release)
   - Most pushes succeed on first try (concurrent edits rare)
   - On 412: cheap probe (hashes only) instead of redownloading everything

2. **Delta Upload**:
   - Reduces bandwidth (most edits are 1-3 files)
   - Pre-computed locally via hash comparison
   - Batching handles large repos (500+ files)

3. **Per-Key Hashes**:
   - Enables cheap conflict probes (GET ?view=hashes)
   - Teammates' edits automatically excluded from retry delta
   - Deterministic (not CRDT or vector clocks needed)

4. **File Watcher**:
   - Automatic sync without user action
   - Debounce coalesces rapid edits
   - fs.watch ({recursive: true}) avoids fd exhaustion

5. **Secret Scanning**:
   - Gitleaks rules curated (high-confidence only)
   - Client-side (secrets never leave machine)
   - Logged separately (file count + rule IDs, not paths/values)

6. **Server-Side Upsert**:
   - Allows partial pushes (batching) without manual merge
   - Keys not in PUT preserved on server
   - Enables simple "last-write-wins" semantics per-key

### What's NOT Implemented

- **CRDT** (Conflict-free Replicated Data Types): Overkill for this use case
- **Vector clocks**: Not needed (per-key last-write-wins is simpler)
- **Three-way merge**: Content-level merging deferred to user (CLI shows conflicts)
- **Permissions**: All org members see all repo team memory
- **Encryption**: In-transit via HTTPS + OAuth; at-rest on server assumed secured
- **Soft delete**: Files aren't marked deleted, just removed from local dir

---

## 16. OPERATIONAL ISSUES & EDGE CASES

### Too-Many-Entries Recovery

**Scenario**: User has 1000 files, server cap is 500

**Flow**:
1. Push fails with 413 (structured)
2. Client learns `serverMaxEntries: 500` from response
3. Next push pre-truncates to 500 files (alphabetically)
4. Push succeeds (but last 500 files don't sync)
5. User sees warning: "Consider consolidating or removing team memory files"

**Resolution**: User must delete files or consolidate to get under cap

### ETag Staleness

**Scenario**: Server restarted, ETag validation broken?

**Mitigation**:
- Server's ETag should be stable for same content
- If ETag changes on same content: client falls back to full fetch (304 not returned)
- Next push computes delta from scratch

### Parallel Writes

**Scenario**: Two Claude processes both in same repo, both editing team memory?

**Current Behavior**:
- Both pull from server
- Both compute independent deltas
- First push wins, second push hits 412
- Second push retries, pulls hashes, recomputes
- Second push's changes either merge (if different keys) or lose (if same key)

**Not optimal** but acceptable (rare in practice, second push retries).

### Disk Full

**Scenario**: Writing team memory to disk fails (ENOSPC)

**Current Behavior**:
- `writeFile` throws, caught in promise chain
- File write logged as warn, not included in return count
- Pull partially succeeds (some files written)
- Next pull retries

### Massive Repos (Lots of Files)

**Performance**:
- Initial pull p99 was ~22s (50 entries, serial on old code; now parallel)
- fs.watch ({recursive: true}) on macOS: 2 fds for 60 files (FSEvents O(1))
- watcher debounce: 2s after last write → ~2s push latency

### Unicode & Encoding

**Handling**:
- Content read/written as UTF-8 (no encoding option in readFile/writeFile)
- Hashes computed on UTF-8 bytes
- Keys stored with forward slashes (normalized via replaceAll)

---

## 17. CONCLUSION

**Anthropic's team memory sync is a well-engineered, pragmatic system**:

- **Simple semantics**: Pull overwrites, push delta, conflicts resolved locally-wins
- **Efficient**: ETag + per-key hashes + batching reduce bandwidth
- **Robust**: Validation, secret scanning, error handling
- **Observable**: Rich telemetry for debugging and monitoring
- **Safe**: OAuth-scoped, path-validated, no XSS surface

**Key engineering decisions**:
1. Optimistic locking instead of pessimistic
2. Delta uploads with per-key checksums
3. Client-side secret scanning (prevents leaks)
4. fs.watch ({recursive: true}) avoids fd exhaustion
5. Local-wins conflict resolution (respects user's current edit)
6. Per-org cap on entries (no client-side default)

**Known limitations**:
- No file deletion propagation (delete local, file returns on next pull)
- No content-level merge (same-key edits by different users: local wins)
- No permissions (all org members see all repo memory)
- Secret scanning only matches gitleaks rules (custom secrets pass through)

This is production-grade code with security considerations built in from the start.
