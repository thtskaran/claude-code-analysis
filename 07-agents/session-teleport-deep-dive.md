# Claude Code Session Teleportation System - Deep Dive Analysis

## Executive Summary

Claude Code's session teleportation system enables moving conversations and code sessions between devices and execution contexts. The system uses OAuth authentication, REST APIs, and Git bundles to transfer complete session state including message history, file references, and uncommitted changes. Teleportation consists of two main flows: teleporting **to remote** (sending local work to cloud) and **resuming from remote** (pulling cloud work locally).

**Key Architecture:**
- **API-first:** Sessions API (`/v1/sessions`) serves as the central control plane
- **Multi-mode source strategy:** GitHub cloning (preferred) with automatic fallback to git bundles (for local-only repos)
- **Stateless transfer protocol:** Messages and state are serialized as JSON and transmitted via REST
- **Authentication:** OAuth2 tokens with organization UUID scoping
- **Serialization:** JSON for messages/metadata, Git native format for repository state

---

## Part 1: File-by-File Architecture

### 1.1 `api.ts` - Core API Types & Utilities (467 lines)

**Primary Purpose:** Session API client library with types, headers, and retry logic.

#### Functions:
- **`prepareApiRequest()`** (lines 181-198)
  - Validates OAuth token and fetches org UUID
  - Returns: `{accessToken: string, orgUUID: string}`
  - Error: Throws if no auth or org UUID

- **`fetchCodeSessionsFromSessionsAPI()`** (lines 204-269)
  - Lists all sessions accessible to the authenticated user
  - Transforms `SessionResource` → `CodeSession` format
  - Extracts git repo from `session_context.sources` (GitSource type)
  - Uses retry-enabled GET with exponential backoff (2s, 4s, 8s, 16s)
  - Error: Logs and re-throws on failure

- **`fetchSession(sessionId: string)`** (lines 289-327)
  - GET `/v1/sessions/{sessionId}` with 15s timeout
  - Returns full `SessionResource` including `session_context`
  - Handles 404 (not found), 401 (expired token), other errors
  - No retry here (unlike the list endpoint)

- **`getBranchFromSession(session)`** (lines 334-342)
  - Extracts first branch name from GitRepositoryOutcome
  - Path: `session.session_context.outcomes[]` → find `type: 'git_repository'` → branches[0]

- **`sendEventToRemoteSession()`** (lines 361-417)
  - POST `{uuid, session_id, type: 'user', message}` as event
  - 30s timeout (accounts for CCR worker cold-start, ~2.6s observed)
  - De-duplication: callers can pass UUID of a local message for echo filtering
  - Returns boolean (success/failure, no exceptions)

- **`updateSessionTitle()`** (lines 425-466)
  - PATCH `/v1/sessions/{sessionId}` with new title
  - Returns boolean success/failure

#### Critical Types:

```typescript
type SessionStatus = 'requires_action' | 'running' | 'idle' | 'archived'

type SessionContext = {
  sources: SessionContextSource[]  // [{ type: 'git_repository', url, revision?, allow_unrestricted_git_push? }]
  cwd: string
  outcomes: Outcome[] | null       // git_info with branches
  custom_system_prompt: string | null
  append_system_prompt: string | null
  model: string | null
  seed_bundle_file_id?: string     // For git-bundle seeding
  github_pr?: { owner, repo, number }
  reuse_outcome_branches?: boolean // Reuse caller's branch instead of claude/xxx
}

type SessionResource = {
  type: 'session'
  id: string
  title: string | null
  session_status: SessionStatus
  environment_id: string
  created_at: string
  updated_at: string
  session_context: SessionContext
}
```

#### Retry Strategy:
- **Transient errors retried:** Network errors (no response), 5xx status
- **Not retried:** Client errors (4xx) — permanent failures
- **Exponential backoff:** 2s, 4s, 8s, 16s (4 retries = 5 total attempts)
- **Beta header:** `'anthropic-beta': 'ccr-byoc-2025-07-29'` signals CCR support
- **Org scoping:** `'x-organization-uuid': orgUUID` in all headers

#### Security:
- OAuth tokens required (no API key fallback)
- 15-30s timeouts prevent hanging
- Error messages reveal minimal internal state
- No sensitive data logged by default (logForDebugging used)

---

### 1.2 `environments.ts` - Execution Environment Discovery (121 lines)

**Primary Purpose:** List and create compute environments for remote sessions.

#### Functions:
- **`fetchEnvironments()`** (lines 32-70)
  - GET `/v1/environment_providers` listing all available environments
  - Requires OAuth + org UUID
  - Returns `EnvironmentResource[]` with `kind: 'anthropic_cloud' | 'byoc' | 'bridge'`

- **`createDefaultCloudEnvironment(name)`** (lines 76-120)
  - POST `/v1/environment_providers/cloud/create`
  - Creates a default anthropic_cloud environment with preset config:
    - cwd: `/home/user`
    - languages: Python 3.11, Node 20
    - network: default hosts allowed, no IP restrictions
  - Returns new environment resource

#### Env Types:
```typescript
type EnvironmentKind = 'anthropic_cloud' | 'byoc' | 'bridge'
  // anthropic_cloud: standard Anthropic-managed compute
  // byoc: customer's own compute environment
  // bridge: lightweight bridging environment
```

#### Use Case:
`teleportToRemote()` fetches environments to select where the session runs. Preference ladder: user's configured default → anthropic_cloud (preferred) → first available (excluding bridge).

---

### 1.3 `environmentSelection.ts` - User Env Configuration (78 lines)

**Primary Purpose:** Merge environment list with user settings to determine the selected environment.

#### Functions:
- **`getEnvironmentSelectionInfo()`** (lines 24-77)
  - Queries `fetchEnvironments()` then checks merged settings
  - Respects `settings.remote.defaultEnvironmentId` if configured
  - Iterates SETTING_SOURCES in reverse (highest priority last)
  - Returns object:
    ```typescript
    {
      availableEnvironments: EnvironmentResource[],
      selectedEnvironment: EnvironmentResource | null,  // env that would be used
      selectedEnvironmentSource: SettingSource | null   // where the config came from
    }
    ```

#### Design:
Used by CLI to let users choose their default compute target without hard-coding env IDs.

---

### 1.4 `gitBundle.ts` - Git State Packaging (293 lines)

**Primary Purpose:** Create and upload Git bundles for "seed bundle" seeding — allows sessions to clone from a local snapshot instead of GitHub.

#### The Flow:
1. **Stash uncommitted changes:** `git stash create` → dangling commit (untracked files excluded)
2. **Make stash reachable:** `git update-ref refs/seed/stash <sha>`
3. **Bundle --all:** Packs all refs including `refs/seed/stash`
4. **Upload:** POST to Files API as `_source_seed.bundle`
5. **Cleanup:** Delete refs/seed/stash and refs/seed/root (don't pollute user's repo)

#### Key Functions:

**`_bundleWithFallback()`** (lines 50-146)
- Tries bundle in order: `--all` → HEAD → squashed-root
- **Scope tiers:**
  - **`all`**: Full history, all refs (default, smallest when possible)
  - **`head`**: Current branch only + stash (if present)
  - **`squashed`**: Single parentless commit of HEAD's tree (or stash tree if WIP exists) — no history, just the snapshot
- Falls back when bundle exceeds `maxBytes` (default 100MB, tunable via `tengu_ccr_bundle_max_bytes` feature gate)
- **Why squashed?** Large repos: a single-commit tree is tiny; history is dropped entirely
- **Stash handling:** When WIP exists, squashed tier bakes uncommitted changes in (can't bundle stash ref separately because its parents drag history back)

**`createAndUploadGitBundle()`** (lines 152-292)
- Main entry point; called from `teleportToRemote()` when bundle mode is active
- **Pre-flight:**
  - Sweeps stale refs from prior crashed runs (`refs/seed/stash`, `refs/seed/root`)
  - Checks for any refs with `git for-each-ref --count=1 refs/` (rejects empty repos)
- **Stash capture:**
  - `git stash create` writes a dangling commit (doesn't touch reflog or working tree)
  - Untracked files intentionally excluded
- **Bundle creation:**
  - Calls `_bundleWithFallback()` with signal support for cancellation
- **Upload:**
  - POST to `/v1/files` with fixed `relativePath: '_source_seed.bundle'`
  - CCR reads this path to locate the bundle
- **Cleanup:**
  - Always delete temp bundle file and refs (in finally block, non-fatal if fails)
- **Returns:**
  ```typescript
  {
    success: true,
    fileId: string,         // File API ID for seed_bundle_file_id
    bundleSizeBytes: number,
    scope: 'all' | 'head' | 'squashed',
    hasWip: boolean
  } | {
    success: false,
    error: string,
    failReason: 'git_error' | 'too_large' | 'empty_repo'
  }
  ```

#### Bundle Config:
- **maxBytes:** Tunable per feature gate; default 100MB
- **Gate:** `tengu_ccr_bundle_max_bytes` (GrowthBook feature)
- **Trigger:** CCR_FORCE_BUNDLE env var or `tengu_ccr_bundle_seed_enabled` gate

#### Limitation:
- Untracked files not captured (only tracked + stash)
- Very large repos may fail all tiers (then fall back to GitHub if available)

#### Security:
- Bundle is a standard Git format; no custom serialization
- Files API handles access control (via session_id and token)
- No credentials embedded in bundle

---

## Part 2: What IS Teleportation?

### 2.1 The Two Flows

**Teleport To Remote:**
- User runs `claude --remote` (or `--teleport`) with a description
- CLI bundles local repo + uncommitted changes
- Creates a session on Anthropic's cloud (CCR)
- Returns session ID (can be resumed later from another machine)
- Use case: Offload heavy compute or switch devices mid-task

**Teleport From Remote (Resume):**
- User runs `claude --teleport <session-id>` from another machine/checkout
- CLI fetches session history and metadata from API
- Validates current checkout matches session's repo
- Pulls branch created by remote session
- Resumes conversation locally
- Use case: Continue work started in cloud on a different device

### 2.2 Session Identity Model

A session is a **durable object** on Anthropic's infrastructure:
- **Created by:** REST API call (POST /v1/sessions)
- **Identified by:** UUID string (`session_id`)
- **Persisted across:** Device restarts, network outages, multiple resumptions
- **Owned by:** Organization (org_uuid scoped)
- **Lifecycle:**
  - `idle` → user resumes or sends events
  - `running` → CCR processing
  - `requires_action` → waiting for user input
  - `archived` → no new events accepted (via POST /archive)

### 2.3 Data Transfer Mechanics

**What Transfers:**

| Data | Format | Transport | Source → Dest |
|------|--------|-----------|---------------|
| Messages | JSON SDK format (SDKMessage[]) | REST API /v1/sessions/{id}/events | Serialized thread store |
| Git history + uncommitted | Git native (bundle format) | Files API + seed_bundle_file_id | `git bundle --all` |
| Metadata (branch, model) | JSON | REST API /v1/sessions/{id} | session_context |
| File references | URLs/IDs in message content | Not directly transferred | Resolved by receiver |
| Working directory | File paths + cwd | Not transferred | Reconstructed from git checkout |

**What Does NOT Transfer:**
- File system state outside git (except stash via bundle)
- MCP connections (rebuilt on destination)
- Tool permissions context (reset/revalidated)
- Local environment variables (not in session)
- SSH keys, OAuth tokens (re-authenticated locally)

### 2.4 Authorization Model

**Authentication:**
- User must have OAuth credentials (Claude.ai account, not API key)
- OAuth token scoped to organization UUID
- Token validated on every API request

**Session Access:**
- Sessions are **org-scoped**, not globally shareable
- User can only access sessions in their org
- No per-session access tokens (auth delegates to OAuth token)
- Session IDs are UUIDs but **not secret** — brute-force risk is org-wide OAuth scope

**Risk:** If an attacker gets your OAuth token, they can:
- List all your sessions (`/v1/sessions`)
- Fetch any session's history (`/v1/sessions/{id}`)
- Send messages to running sessions (`/v1/sessions/{id}/events`)
- Archive sessions (prevent resumption)

They **cannot:**
- Create sessions without your auth context
- Access sessions from other orgs
- Execute in your repo (remote only; no code execution on client)

### 2.5 Protocol Details

**Session Creation (Teleport To Remote):**

```
POST /v1/sessions
Headers:
  Authorization: Bearer <oauth_token>
  x-organization-uuid: <org-uuid>
  anthropic-beta: ccr-byoc-2025-07-29
  Content-Type: application/json

Body:
{
  title: string,
  events: [
    {
      type: 'event',
      data: {
        type: 'control_request',
        request: { subtype: 'set_permission_mode', mode: 'plan' | 'execute', ... }
      }
    },
    {
      type: 'event',
      data: {
        type: 'user',
        message: { role: 'user', content: string | ContentBlock[] }
      }
    }
  ],
  session_context: {
    sources: [{ type: 'git_repository', url: 'https://github.com/...', revision: 'branch' }],
    seed_bundle_file_id: 'file-xyz',  // optional; takes precedence
    outcomes: [{ type: 'git_repository', git_info: { branches: ['claude/xxx'] } }],
    model: 'claude-3-5-sonnet-20241022',
    cwd: '/home/user'
  },
  environment_id: 'env-abc123'
}

Response (201 Created):
{
  id: 'session-uuid',
  title: string,
  session_status: 'running' | 'idle',
  environment_id: string,
  session_context: { ... },
  created_at: ISO8601,
  updated_at: ISO8601
}
```

**Session Fetch (Teleport Resume):**

```
GET /v1/sessions/{sessionId}
Headers:
  Authorization: Bearer <oauth_token>
  x-organization-uuid: <org-uuid>
  anthropic-beta: ccr-byoc-2025-07-29

Response (200 OK):
{
  id: 'session-uuid',
  title: string,
  session_status: ...,
  environment_id: string,
  session_context: {
    sources: [...],
    outcomes: [{ git_info: { branches: ['claude/task'] } }],
    ...
  },
  ...
}
```

**Events Polling (Remote Work Sync):**

```
GET /v1/sessions/{sessionId}/events?after_id=<cursor>
Headers: (same as above)

Response (200 OK):
{
  data: [
    { type: 'user', message: {...} },
    { type: 'assistant', message: {...} },
    { type: 'result', subtype: 'success' | 'error_during_execution', ... }
  ],
  has_more: boolean,
  first_id: string | null,
  last_id: string | null
}
```

---

## Part 3: Deep Dives on Key Mechanisms

### 3.1 Git State Transfer via Bundles

**Problem:** GitHub clone requires OAuth setup and fails for local-only repos. Bundle solves this.

**Mechanism:**

1. **Stash uncommitted changes** (tracked files only):
   ```bash
   git stash create  # → SHA of dangling commit with staged + unstaged
   git update-ref refs/seed/stash <SHA>  # Make it reachable
   ```

2. **Create bundle** (fallback ladder):
   ```bash
   # Tier 1: Full history
   git bundle create seed.bundle --all [refs/seed/stash]

   # Tier 2: Current branch only
   git bundle create seed.bundle HEAD [refs/seed/stash]

   # Tier 3: Single snapshot commit
   git commit-tree refs/seed/stash^{tree} -m 'seed'  # Orphan commit
   git update-ref refs/seed/root <new-sha>
   git bundle create seed.bundle refs/seed/root
   ```

3. **Upload via Files API:**
   ```
   POST /v1/files
   file: seed.bundle (binary)
   relativePath: '_source_seed.bundle'
   ```

4. **Cleanup** (always):
   ```bash
   git update-ref -d refs/seed/stash  # Delete temp ref
   rm seed.bundle
   ```

5. **CCR side:**
   - `git clone --bundle <file-id>` (or similar internal logic)
   - Restores full tree state
   - If stash present, checkout tree from stash (baked into squashed tier)

**Why This Works:**
- Bundle is Git's native format; no custom serialization needed
- Survives full git history (--all tier) when repo is small enough
- Falls back to just the tree snapshot when history is huge
- Untracked files are not included (intentional; they're local config/build artifacts)

**Fallback Triggers:**
- If --all bundle > 100MB, try HEAD
- If HEAD bundle > 100MB, try squashed (single orphan commit)
- If squashed > 100MB, fail and fall back to GitHub (if available)
- If no GitHub remote, user gets empty sandbox

### 3.2 File References & Path Reconstruction

**Challenge:** Messages contain file paths/references that exist on source but not destination.

**How It's Handled:**

1. **Messages are serialized as-is:**
   - Content blocks include file paths: `"file": "/path/to/file.ts"`
   - These are strings in the message; no resolution happens during transfer
   - Model has no knowledge that file was transferred

2. **Destination Reconstruction:**
   ```
   Teleport From Remote:
   1. Fetch session history (messages with file paths)
   2. Validate repo matches (session.sources[0].url == current repo)
   3. Checkout branch (fetch + git checkout)
   4. Messages become available in local session
   5. File paths are now valid (same repo, same files exist)

   Teleport To Remote:
   1. Create session with session_context.sources = [github_url]
   2. CCR clones repo before session starts
   3. Initial message sent with file paths
   4. Paths are relative to repo root, valid in CCR container
   ```

3. **For Bundle Mode:**
   - Bundle contains full tree snapshot (or history)
   - CCR extracts it into a fresh working directory
   - File paths in messages now point to extracted files
   - No special handling needed; it just works

4. **Unresolvable References:**
   - If destination repo differs from session's source, files may not exist
   - Teleport validates this before resuming (`validateSessionRepository()`)
   - Throws error: "must run from checkout of owner/repo"

**Data Leakage Risk:**
- File paths leaked in messages if repo access changes
- Mitigation: org-scoped auth; can't access other orgs' sessions
- No URLs/IDs exposed; just paths relative to repo root

### 3.3 MCP Connections & Tool State Across Teleport

**How Teleportation Affects MCP:**

1. **Teleport To Remote:**
   - MCP connections are **local** (to CLI's MCP server)
   - Not transferred to remote
   - Remote session has its own MCP setup (in the container)
   - Initial message sent with raw text (no tool context)

2. **Teleport Resume:**
   - Message history fetched; includes tool calls from remote
   - Local MCP is initialized fresh
   - Tool-use blocks from remote are replayed as-is
   - Local tooling may differ (tools available in remote ≠ tools available locally)
   - Model continuation works because message format is standard

3. **Tool Permission Context:**
   - Permissions are **re-evaluated** on the receiving end
   - No permission tokens transferred
   - Each environment (local, remote) checks policy independently
   - Risk: Different policy enforcement on each end could cause tool calls to fail

**Mitigation:**
- Teleport validates preconditions before resuming (git repo, GitHub app, etc.)
- Policy checks in precondition validation
- User sees errors early if remote work can't continue locally

### 3.4 Authentication & Secrets Handling

**What Transfers in Session Context:**

```typescript
session_context = {
  sources: [{ url: 'https://github.com/...' }],  // Public URL
  outcomes: [...],
  model: 'claude-3-5-sonnet-20241022',
  custom_system_prompt: string | null,            // User-written; may contain instructions
  append_system_prompt: string | null,
  // These do NOT transfer:
  // - OAuth tokens
  // - SSH keys
  // - API credentials
  // - Environment variables (except explicitly sent in environmentVariables)
}
```

**Credential Flow:**
1. User authenticates locally with OAuth (`/login`)
2. OAuth token stored in local state
3. Token used to create remote session (passed in Authorization header)
4. Remote session **does not receive** the token
5. Remote gets `CLAUDE_CODE_OAUTH_TOKEN` injected (fresh token for the session, not user's token)
6. When resuming, user's local OAuth token required (not the session's token)

**Implications:**
- Tokens are **transient** and **non-transferable**
- Each environment must authenticate independently
- No shared credentials across machines
- Reduces blast radius if one token compromised

### 3.5 Progress Indication & Error Handling

**Progress Steps** (from `TeleportProgressStep`):

```typescript
'validating'     // Checking repo, auth, preconditions
'fetching_logs'  // Pulling session history via /v1/sessions/{id}/events
'fetching_branch'// Extracting git branch from session metadata
'checking_out'   // Running git fetch/checkout locally
'done'           // Complete
```

Callback: `(step: TeleportProgressStep) => void`

**Error Handling:**

1. **Network Errors (api.ts):**
   - Retried with exponential backoff (2s, 4s, 8s, 16s)
   - 5xx errors trigger retry
   - 4xx errors thrown immediately (permanent failures)

2. **Partial Transfers:**
   - Bundle upload fails → fall back to GitHub (if available)
   - Session creation fails → return null (caller decides action)
   - Events fetch fails → throw error (can't resume)

3. **Repo Mismatch:**
   - Validates session.sources[0].url == current repo
   - If mismatch: throws `TeleportOperationError` with formatted message
   - User directed to correct checkout

4. **Branch Checkout Failures:**
   - Fetch fails: logs and attempts without mapping
   - Checkout fails: throws error, attempts alternative strategies (with/without -b)
   - Upstream setup fails: non-fatal; logged but doesn't block

5. **Session Not Found:**
   - 404 on GET /v1/sessions/{id} → specific error message
   - 401 on any request → suggests `/login` to re-authenticate

**Structured Analytics:**
- `tengu_teleport_error_*` events logged for each error type
- `tengu_teleport_bundle_mode` events for bundle success/fail
- `tengu_teleport_source_decision` logged with reason (github_preflight_ok, bundle_fallback, etc.)

### 3.6 Session Lifecycle & Archival

**State Transitions:**

```
CREATE → running
              ↓
        (user sends event)
              ↓
        running → idle
              ↓
        (user sends event or resumes)
              ↓
        running → requires_action (wait for user input)
              ↓
        (waiting for browser approval, etc.)
              ↓
        running → idle/requires_action (cycle)
              ↓
        (user calls archive or session errors)
              ↓
        archived (no new events accepted)
```

**Archival:**

```typescript
// From teleport.tsx line ~1192+
export async function archiveRemoteSession(sessionId: string): Promise<boolean> {
  POST /v1/sessions/{sessionId}/archive
  // 409 if already archived (treated as success)
  // No running-status check (unlike DELETE)
  // Fire-and-forget; failure leaves visible session until GC
}
```

**Why Archival Instead of Delete?**
- Archive immediately stops accepting new events
- Remote session stops on next write attempt
- Soft delete; visible in history
- GC cleaner removes archived sessions after retention period

---

## Part 4: Security Analysis

### 4.1 Attack Surface

**Vector 1: OAuth Token Compromise**
- Attacker obtains user's OAuth token
- Can fetch any session in the org
- Can send events (commands) to running sessions
- **Cannot** execute code on client (remote only)
- **Mitigation:** Token rotation, short expiry, secure storage

**Vector 2: Session ID Enumeration**
- Session IDs are UUIDs; not random enough to prevent brute-force
- Attacker tries 2^64 session IDs to find valid ones
- Each valid session reveals history (via GET /v1/sessions/{id})
- **Mitigation:** API rate-limiting, IP-based throttling, audit logging
- **Assessment:** Low risk if OAuth scoped (can't cross orgs)

**Vector 3: Repository Mismatch Exploitation**
- Attacker creates session with different repo (e.g., phishing repo)
- Sends resume session ID to victim
- Victim runs `claude --teleport <id>` and gets wrong branch
- **Vulnerability:** No signature on session; history could be forged if API is compromised
- **Mitigation:** Session validation checks current repo matches; user must initiate resume

**Vector 4: File Path Injection in Messages**
- Attacker includes malicious file paths in initial message
- Model generates code that reads/writes those paths
- **Risk:** Limited to files in checked-out repo (no directory traversal with git)
- **Mitigation:** Same as any model interaction; user must review code

**Vector 5: Git Bundle Tampering**
- Attacker modifies uploaded bundle
- CCR extracts malicious history
- **Mitigation:** Files API access control; needs valid OAuth token + session_id
- **Assessment:** Bundle is Git-verified on extract (git clone validates SHA1, though SHA1 is weak)

**Vector 6: Untracked File Loss**
- Bundle excludes untracked files
- User teleports, untracked files disappear remotely
- **Risk:** Low (expected behavior; documented)
- **Mitigation:** Documentation; user can `.gitignore` files to include them

### 4.2 Privilege Escalation

**Risk: No**
- Sessions run with user's own credentials
- No elevation of privilege
- CCR container isolated from user's machine
- OAuth token scoped to user's org

### 4.3 Data Leakage Vectors

**1. Session History Exposure:**
- GET /v1/sessions/{sessionId}/events requires OAuth token
- Token scope: org-wide (not per-session)
- If token compromised: all org's sessions visible
- **Mitigation:** Org-wide access control; audit logging

**2. File Paths in Messages:**
- Message content includes file paths
- Path reveals directory structure
- **Mitigation:** Paths are relative to repo root; same visibility as GitHub

**3. Bundle Contents:**
- Bundle is Git format (standard); contents are repo history
- Uploaded to Files API with `relativePath: '_source_seed.bundle'`
- Files API enforces org-scoped access
- **Mitigation:** Org-scoped auth; no cross-org leaks

**4. Metadata Leakage:**
- Session title, branch names, model choice, environment ID visible
- Doesn't reveal secrets but exposes task intent
- **Mitigation:** None by design (intended for session management)

### 4.4 Hijacking Scenarios

**Scenario 1: Attacker Gets Session ID from Victim**
- Victim runs `claude --remote` and gets back session ID
- Victim shares ID in chat (or it leaks in logs)
- Attacker runs `claude --teleport <id>` from their machine
- **Result:** Attacker must be in correct repo; gets same code but can't execute
- **Mitigation:** Session IDs are not secret; recommendation to not share in untrusted channels

**Scenario 2: OAuth Token Leaked in Session History**
- User accidentally includes token in a message to Claude
- Session history retrieved later
- **Result:** Attacker can hijack user's account
- **Mitigation:** User responsibility; model should refuse tokens
- **Note:** This is a general Claude issue, not teleport-specific

**Scenario 3: MITM on API Calls**
- Attacker intercepts HTTPS between CLI and API
- Captures OAuth token
- **Result:** Full account compromise
- **Mitigation:** TLS; no custom auth mechanism (delegates to OAuth)

---

## Part 5: Serialization & Data Format

### 5.1 Session Context Serialization

```json
{
  "sources": [
    {
      "type": "git_repository",
      "url": "https://github.com/anthropic-ai/anthropic-sdk-python",
      "revision": "main",
      "allow_unrestricted_git_push": false
    }
  ],
  "outcomes": [
    {
      "type": "git_repository",
      "git_info": {
        "type": "github",
        "repo": "anthropic-ai/anthropic-sdk-python",
        "branches": ["claude/fix-retry-logic"]
      }
    }
  ],
  "cwd": "/home/user/repo",
  "custom_system_prompt": null,
  "append_system_prompt": null,
  "model": "claude-3-5-sonnet-20241022",
  "seed_bundle_file_id": "file-abc123xyz",
  "github_pr": null,
  "reuse_outcome_branches": false
}
```

### 5.2 Messages Serialization

```typescript
type SDKMessage =
  | { type: 'user'; message: { role: 'user'; content: string | ContentBlock[] } }
  | { type: 'assistant'; message: { role: 'assistant'; content: ContentBlock[] } }
  | { type: 'result'; subtype: 'success' | 'error_during_execution' | ... }
  | { type: 'tool_result'; tool_use_id: string; ... }

// Sent as event in create:
{
  "type": "event",
  "data": {
    "uuid": "uuid-string",
    "session_id": "session-uuid",
    "type": "user",
    "message": {
      "role": "user",
      "content": "Fix the login button"
    }
  }
}

// Fetched from /events:
[
  {
    "type": "user",
    "message": { "role": "user", "content": "..." },
    "uuid": "...",
    "session_id": "..."
  },
  {
    "type": "assistant",
    "message": { "role": "assistant", "content": [...] },
    "uuid": "...",
    "session_id": "..."
  }
]
```

### 5.3 Git Bundle Format

```
Binary Git format:
- Standard `git bundle` output
- Contains packfile (objects) + refs (heads, tags, custom)
- Verifiable: SHA1 hashes embedded (weak but standard)
- Portable: works with any Git implementation
- Size: varies (100MB typical limit)

Structure:
refs/seed/stash → dangling commit with WIP
refs/heads/main → branch HEAD
refs/heads/feature → other branches
...objects... (packfile)
```

---

## Part 6: Implementation Details

### 6.1 Teleport To Remote Flow (Detailed Steps)

```javascript
async function teleportToRemote(options) {
  1. Check auth
     - getClaudeAIOAuthTokens() → error if missing
     - getOrganizationUUID() → error if missing

  2. Generate title & branch name (unless provided)
     - Call Haiku API (queryHaiku)
     - Parse JSON schema response
     - Fallback to description truncated + "claude/task"

  3. Detect current repository
     - parseGitRemote(git config --get remote.origin.url)
     - Result: { owner, repo, host }

  4. GitHub Preflight Check (if not forced to bundle)
     - checkGithubAppInstalled(owner, repo)
     - Result: boolean (is CCR authorized on this repo?)
     - If false and bundle gate on: skip GitHub, use bundle
     - If false and bundle gate off: proceed optimistically (fail in CCR)

  5. Create/Upload Git Bundle (if GitHub not viable)
     - createAndUploadGitBundle({ oauthToken, sessionId, baseUrl })
     - Flow:
       a. Find .git/ or throw "not in git"
       b. Sweep stale refs (cleanup from prior crashes)
       c. Check for commits (reject empty repos)
       d. git stash create (capture WIP)
       e. git update-ref refs/seed/stash (make reachable)
       f. git bundle create --all (Tier 1)
       g. If > 100MB, try HEAD (Tier 2)
       h. If > 100MB, try squashed (Tier 3)
       i. Upload to /v1/files with relativePath '_source_seed.bundle'
       j. Cleanup refs + temp file
     - Result: { fileId, bundleSizeBytes, scope, hasWip }

  6. Fetch environments
     - GET /v1/environment_providers
     - Filter: anthropic_cloud preferred, skip bridge
     - Result: selected environment_id

  7. Build session context
     - sources: [{ type: 'git_repository', url, revision }]
     - seed_bundle_file_id: (if bundle created)
     - outcomes: [{ type: 'git_repository', git_info: { branches: ['claude/xxx'] } }]
     - model: options.model || getMainLoopModel()
     - cwd: /home/user (default)

  8. Create session
     - POST /v1/sessions
     - Body: { title, events: [], session_context, environment_id }
     - Response: { id, title, session_context, ... }

  9. Return session ID
     - { id: 'session-uuid', title: 'Generated title' }
}
```

### 6.2 Teleport Resume Flow (Detailed Steps)

```javascript
async function teleportResumeCodeSession(sessionId) {
  1. Check auth
     - getClaudeAIOAuthTokens() → error if missing
     - getOrganizationUUID() → error if missing

  2. Fetch session metadata
     - GET /v1/sessions/{sessionId}
     - Response: SessionResource with session_context

  3. Validate repository
     - currentRepo = detectCurrentRepositoryWithHost()
     - sessionRepo = parseGitRemote(session_context.sources[0].url)
     - If repos differ: throw TeleportOperationError
     - If no repo required: proceed

  4. Fetch session history
     - getTeleportEvents(sessionId) → new v2 endpoint
     - Fallback: getSessionLogsViaOAuth(sessionId) → old endpoint
     - Both return Message[] (transcript messages only, no sidechain)

  5. Fetch branch name
     - getBranchFromSession(session) → first branch from outcomes
     - Result: 'claude/task' or similar

  6. Update progress callback
     - onProgress?.('fetching_branch')
     - onProgress?.('checking_out')

  7. Checkout branch locally
     - git fetch origin (or specific branch)
     - git checkout -b <branch> --track origin/<branch>
     - ensureUpstreamIsSet(branch)

  8. Reconstruct messages
     - Deserialize fetched messages
     - Add teleport-resume notice (system + user message)
     - Messages are now ready for local replay

  9. Return result
     - { log: Message[], branch: 'claude/task' }
}
```

### 6.3 Events Polling (For Remote-Agent Sync)

```javascript
async function pollRemoteSessionEvents(sessionId, afterId = null) {
  const MAX_EVENT_PAGES = 50  // Safety cap
  let cursor = afterId
  let sdkMessages = []

  for (let page = 0; page < MAX_EVENT_PAGES; page++) {
    // GET /v1/sessions/{sessionId}/events?after_id=<cursor>
    const response = await axios.get(eventsUrl, {
      params: cursor ? { after_id: cursor } : undefined,
      timeout: 30000
    })

    // Parse response
    const eventsData = response.data
    for (const event of eventsData.data) {
      // Skip internal events
      if (event.type === 'env_manager_log' || event.type === 'control_response') continue

      // Collect SDK messages
      if (event.type in SDKMessage types) {
        sdkMessages.push(event)
      }
    }

    // Pagination
    if (!eventsData.has_more) break
    cursor = eventsData.last_id
  }

  // Fetch metadata if needed
  if (!opts?.skipMetadata) {
    const sessionData = await fetchSession(sessionId)
    const branch = getBranchFromSession(sessionData)
    const status = sessionData.session_status
  }

  return {
    newEvents: sdkMessages,
    lastEventId: cursor,
    branch,
    sessionStatus
  }
}
```

---

## Part 7: Known Limitations & Future Work

### 7.1 Current Limitations

1. **Untracked Files Not Transferred**
   - Bundle only includes tracked files + stash
   - Local config files, build artifacts, node_modules not transferred
   - Workaround: gitignore them (so they're tracked), or manually sync

2. **No Per-Session Access Tokens**
   - Session ID is not secret (UUID brute-forceable in theory)
   - OAuth token provides org-wide access (can't revoke per-session)
   - Improvement: per-session short-lived tokens (future work)

3. **No File Diff Indication**
   - Teleport validates repo matches but doesn't show diffs
   - User doesn't know if files changed since session creation
   - Improvement: show file diffs on resume

4. **Large Repo Limitations**
   - 100MB bundle limit (fallback to squashed or GitHub)
   - Very large monorepos may fail all tiers
   - Improvement: streaming bundles, sparse checkout

5. **No Selective File Transfer**
   - Bundle is all-or-nothing
   - Can't request subset of repo
   - Improvement: sparse bundles, partial fetch

6. **MCP Mismatch on Resume**
   - Tools available locally ≠ tools in remote
   - Tool calls from remote may fail locally (if tool not available)
   - Improvement: validate tool compatibility before resuming

### 7.2 Future Improvements (Based on Code Comments)

1. **Streaming Events**
   - Events poll is paginated (50 pages max)
   - Streaming API could reduce latency

2. **Delta Sync**
   - Full history always fetched on resume
   - Could sync only new events since last resume

3. **Compression**
   - Bundle could be compressed (gzip)
   - Saves bandwidth for large repos

4. **Retry Mechanisms**
   - Events poll has no retry; single failure aborts
   - Could add exponential backoff (like getCodeSessionsFromSessionsAPI)

5. **Session Expiry**
   - No explicit TTL on sessions
   - Old sessions may accumulate (GC cleans up)
   - Could add user-initiated cleanup

---

## Part 8: Test Vectors & Validation

### 8.1 Teleport To Remote Preconditions

```typescript
checkBackgroundRemoteSessionEligibility() validates:
- Policy: allow_remote_sessions (policy_blocked)
- Auth: OAuth token present (not_logged_in)
- Environment: at least one available (no_remote_environment)
- Repo: in .git/ or has remote (not_in_git_repo, no_git_remote)
- GitHub: app installed if no bundle gate (github_app_not_installed)
- Bundle: .git/ exists if bundle mode (not_in_git_repo)
```

### 8.2 Teleport Resume Preconditions

```typescript
validateSessionRepository() checks:
- Current repo matches session's source URL (mismatch → error)
- Host matches (github.com vs GHE instance)
- Port normalization (8443 dropped)
- Case-insensitive comparison
```

### 8.3 Sample Teleport Scenarios

**Scenario A: Local Repo to Cloud**
```
$ cd ~/my-feature
$ claude --remote "Fix login button"
→ Creates session, bundles repo, returns ID
→ User can switch to another machine and resume
```

**Scenario B: Resume on Different Machine**
```
$ cd ~/my-feature  # Same repo, same files
$ claude --teleport session-uuid
→ Validates repo matches
→ Fetches history
→ Checks out claude/fix-login-button
→ Resumes conversation
```

**Scenario C: Large Repo Bundle Fallback**
```
$ cd ~/monorepo (1GB history)
$ claude --remote "Refactor config"
→ git bundle --all fails (> 100MB)
→ git bundle HEAD fails (> 100MB)
→ git bundle squashed succeeds (10MB)
→ Session created with squashed bundle
```

**Scenario D: GitHub Preflight Failure, Bundle Fallback**
```
$ cd ~/private-repo
$ checkGithubAppInstalled returns false
$ CCR_ENABLE_BUNDLE=1
→ Skip GitHub, create bundle
→ Session created with bundle seed
```

---

## Part 9: Summary of Findings

### Security Posture

**Strong:**
- OAuth-based authentication (no hardcoded credentials)
- Org-scoped access control (no cross-org leaks)
- No secrets in session context (tokens, keys not transferred)
- Git bundle format is standard (no custom crypto)

**Weak:**
- Session ID enumeration risk (UUID; no rate-limiting mentioned)
- No per-session tokens (org-wide scope on OAuth token)
- Untracked file handling (silently dropped; could surprise users)

**Recommendations:**
1. Add rate-limiting on /v1/sessions/{id} to prevent brute-force
2. Implement short-lived per-session tokens (future work)
3. Document untracked file behavior in CLI help
4. Add warning when resuming if files changed since creation

### Teleportation Design Quality

**Well-Engineered:**
- Graceful fallbacks (GitHub → bundle → empty sandbox)
- Feature gates allow gradual rollout (tengu_ccr_bundle_seed_enabled)
- Comprehensive error handling with analytics
- Progress callbacks for long operations

**Areas for Improvement:**
- No streaming (all-at-once fetch/upload)
- Limited debugging visibility into bundle failures
- No diff/summary on resume (what changed?)

### Data Transfer Integrity

**What's Verified:**
- Repository validation (session source == current repo)
- Session existence (404 handling)
- Bundle validity (Git format verified on extraction)

**What's Not Verified:**
- Session history authenticity (no signature; relies on API auth)
- File integrity beyond Git's SHA1 (weak by modern standards)

### Completeness

**Fully Implemented:**
- Session creation with git source + bundle support
- Event polling and history fetch
- Branch checkout with upstream setup
- Precondition validation
- Error handling and analytics

**Not Covered (Outside Scope):**
- Browser UI for session management
- Real-time collaboration features
- Session merging/conflict resolution
- Persistent local session cache

---

## Appendix: Code Statistics

| File | Lines | Functions | Purpose |
|------|-------|-----------|---------|
| api.ts | 467 | 6 | Core API client, retry logic, types |
| environments.ts | 121 | 2 | Environment listing & creation |
| environmentSelection.ts | 78 | 1 | Environment preference resolution |
| gitBundle.ts | 293 | 2 main + 1 helper | Bundle creation & upload |
| teleport.tsx | ~1200+ | 10+ | Orchestration & validation |

**Total teleport module: ~955 lines**

---

## Appendix: Key Constants & Feature Gates

```typescript
// Retry configuration
TELEPORT_RETRY_DELAYS = [2000, 4000, 8000, 16000]  // 4 retries
MAX_TELEPORT_RETRIES = 4

// Bundle limits
DEFAULT_BUNDLE_MAX_BYTES = 100 * 1024 * 1024  // 100MB
Tunable via: 'tengu_ccr_bundle_max_bytes' (GrowthBook)

// Feature gates
'tengu_ccr_bundle_seed_enabled'  // Enable bundle fallback
'tengu_ccr_bundle_max_bytes'     // Bundle size limit

// Environment variables
CCR_FORCE_BUNDLE = 1              // Force bundle mode
CCR_ENABLE_BUNDLE = 1             // Enable bundle fallback
CLAUDE_CODE_OAUTH_TOKEN           // Passed to session (injected)

// Beta header
'anthropic-beta': 'ccr-byoc-2025-07-29'  // CCR support signaling

// Timeouts
Session creation: 30s (CCR worker cold-start)
Session fetch: 15s
Events poll: 30s per page
```

---

## Appendix: API Endpoints

```
POST   /v1/sessions                           Create session
GET    /v1/sessions                           List sessions
GET    /v1/sessions/{sessionId}               Fetch session
PATCH  /v1/sessions/{sessionId}               Update title
POST   /v1/sessions/{sessionId}/events        Send event
GET    /v1/sessions/{sessionId}/events        Poll events
POST   /v1/sessions/{sessionId}/archive       Archive session

GET    /v1/environment_providers              List environments
POST   /v1/environment_providers/cloud/create Create cloud env

POST   /v1/files                              Upload file (bundle)
```

---

End of Analysis
