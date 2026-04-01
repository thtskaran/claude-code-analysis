# Team Memory Sync: Security Analysis

## Executive Summary

Anthropic's team memory sync system demonstrates **security-first engineering**. Client-side secret scanning prevents credentials from ever leaving the user's machine, OAuth enforces auth, and path validation blocks directory traversal. However, **no PII scanning** and **server-side secret validation gaps** represent the primary security limitations.

---

## 1. AUTHENTICATION & AUTHORIZATION

### OAuth Enforcement

**Strong Points**:
- ✅ Requires first-party Anthropic OAuth with specific scopes
- ✅ Scopes validated on every request (`CLAUDE_AI_INFERENCE_SCOPE` + `CLAUDE_AI_PROFILE_SCOPE`)
- ✅ Token refresh on every API call (`checkAndRefreshOAuthTokenIfNeeded()`)
- ✅ Bearer token in Authorization header (no URL params, not logged)

**Code**:
```typescript
function isUsingOAuth(): boolean {
  return (
    getAPIProvider() === 'firstParty' &&
    isFirstPartyAnthropicBaseUrl() &&
    Boolean(
      tokens?.accessToken &&
      tokens.scopes?.includes(CLAUDE_AI_INFERENCE_SCOPE) &&
      tokens.scopes.includes(CLAUDE_AI_PROFILE_SCOPE)
    )
  )
}
```

**Weaknesses**:
- ⚠️ No documented token expiry validation
- ⚠️ `getClaudeAIOAuthTokens()` implementation not visible (assumes standard OAuth lib)
- ⚠️ No documented rate limiting at OAuth server level (only 429 on API)

### Scope-Based Access Control

**Current**:
- Per-org OAuth tokens grant access to all repo team memory in that org
- **No per-repo scope** — user can access all repos they're org members of (by design)
- **No per-user scope** — all org members see all team memory

**Expected Behavior**:
- User with GitHub access to repo X can sync team memory for repo X
- Server enforces per-repo ACL (not validated client-side)

**Trust Model**:
- Trust server to validate org membership + repo access
- Client assumes: valid OAuth token → can access this repo

### No_OAuth Suppression

**Behavior**:
```typescript
if (r.errorType === 'no_oauth') {
  pushSuppressedReason = 'no_oauth'
  // Watcher stops retrying until session restart
}
```

**Implication**: User without valid OAuth credentials will see watcher permanently suppressed. They must restart with auth.

---

## 2. DATA LEAKAGE & ISOLATION

### Cross-Tenant Data Leakage

**Risk**: Team memory from Organization A leaks to Organization B

**Mitigations**:
1. ✅ OAuth token scoped to specific org
2. ✅ API endpoint includes `organizationId` in response (server validation)
3. ✅ Client never allows cross-org sync (would require OAuth token from other org)
4. ✅ Per-repo isolation (organizationId + repo slug required on server)

**Attack Surface**:
- ⚠️ If OAuth token from Org A is stolen and used by Org B user → Org A memory accessible?
  - **Mitigated by**: OAuth issued per user per org; token is User+Org specific
  - **Not explicitly documented** in code (assumed handled by OAuth server)

**Code**:
```typescript
const auth = getAuthHeaders()  // Extracts accessToken
// Only this token is used; can't specify org/repo in auth header
const endpoint = getTeamMemorySyncEndpoint(repoSlug)
```

**Verdict**: ✅ **Cross-tenant isolation is adequate** (server-enforced, OAuth validates)

### Intra-Org Visibility

**Current Design**: All org members see all repo team memory (all-or-nothing)

**Implication**:
- Sensitive team memory shared with anyone in the org who has GitHub access to that repo
- No fine-grained access control (no per-team sharing)
- **Not a security bug**, but a **design choice** (transparency > granularity)

**User-Facing Risk**:
- Org might commit PII, customer data, or sensitive patterns to team memory
- Unaware that it's visible to all org members with repo access

---

## 3. SECRET SCANNING & CREDENTIAL PROTECTION (PSR M22174)

### Client-Side Secret Scanner

**Strength**: 44 high-confidence gitleaks rules, curated to avoid false positives

```typescript
const SECRET_RULES: SecretRule[] = [
  // Cloud providers
  { id: 'aws-access-token', source: '\\b((?:A3T[A-Z0-9]|AKIA|...)' },
  { id: 'gcp-api-key', source: '\\b(AIza[\\w-]{35})' },

  // AI APIs
  { id: 'anthropic-api-key', source: `\\b(${ANT_KEY_PFX}03-[a-zA-Z0-9_\\-]{93}AA)` },
  { id: 'openai-api-key', source: '\\b(sk-(?:proj|svcacct|...)' },

  // VCS
  { id: 'github-pat', source: 'ghp_[0-9a-zA-Z]{36}' },

  // ... 40+ more
]
```

**How It Works**:
1. Before each file is added to entries, `scanForSecrets(content)` is called
2. If ANY rule matches → entire file is **skipped** from upload
3. Matched rule ID + file path logged; secret value **never logged**
4. User receives warning message with file count and secret types

**Code**:
```typescript
const secretMatches = scanForSecrets(content)
if (secretMatches.length > 0) {
  const firstMatch = secretMatches[0]!
  skippedSecrets.push({
    path: relPath,
    ruleId: firstMatch.ruleId,
    label: firstMatch.label,
  })
  logForDebugging(`detected ${firstMatch.label}`)
  return  // Skip this file entirely
}
entries[relPath] = content
```

**Analytics** (secret VALUES never included):
```typescript
logEvent('tengu_team_mem_secret_skipped', {
  file_count: skippedSecrets.length,
  rule_ids: 'github-pat,aws-access-token'  // Comma-joined
})
```

### Strengths

✅ **Secrets never leave user's machine** — detected pre-upload
✅ **No false positive noise** — only 44 high-confidence rules (generic keyword rules excluded)
✅ **Runtime assembly of prefixes** — Anthropic API key prefix assembled as `['sk', 'ant', 'api'].join('-')` so literal doesn't appear in bundle
✅ **Values never logged** — only rule IDs logged for analytics
✅ **Entire file skipped** — conservative approach (avoids partial redaction)
✅ **Lazyly compiled** — regexes compiled on first scan, cached

### Weaknesses

⚠️ **Server-side gap**: No documented server-side secret scanning
- If client is downgraded or disabled (feature flag OFF), secrets pass through
- **Recommendation**: Server should also scan team memory entries before storing

⚠️ **Custom secrets not matched**: Only gitleaks rules
- Custom API key formats (internal services, non-standard prefixes)
- Weak passwords that aren't API keys
- Custom identifiers (user IDs, internal tokens)

⚠️ **False negatives possible**: High-confidence rules miss obscured secrets
- Example: API key split across lines (though team memory is flat UTF-8)
- Comments that reference credentials (e.g., "don't use key: xxx")

⚠️ **Redaction optional**: `redactSecrets()` function exists but isn't used in push flow
- Files are skipped (not redacted in-place)
- If wanted to redact instead of skip, would need different path

### User-Facing Issue

**Problem**: User commits secret → file silently skipped → user doesn't know
- User sees "5 files pushed" but their 6th file (with secret) silently dropped
- If user doesn't read logs, they're unaware team memory is incomplete

**Mitigation**: Watcher logs warning to debug output, also fires analytics event
- User must check debug logs to see which files were skipped
- **No prominent error** (would be noisy if user is iterating)

---

## 4. CREDENTIAL VALIDATION & REDACTION

### File Write Hook (FileWriteTool, FileEditTool)

**Validation**:
```typescript
export function checkTeamMemSecrets(
  filePath: string,
  content: string,
): string | null {
  if (!isTeamMemPath(filePath)) return null
  const matches = scanForSecrets(content)
  if (matches.length === 0) return null

  const labels = matches.map(m => m.label).join(', ')
  return `Content contains potential secrets (${labels}) and cannot be written to team memory.
           Team memory is shared with all repository collaborators.`
}
```

**Strength**: ✅ Prevents model from writing secrets to team memory (blocks at write time)

**Limitation**: ⚠️ Only runs if Claude tries to write to team memory path
- If Claude writes to non-team-memory file, then user moves it later → secret not detected
- Unlikely but possible

---

## 5. PATH TRAVERSAL & DIRECTORY ESCAPE

### Path Validation

**Function**: `validateTeamMemKey(relPath: string)`

**Requirements** (from teamMemPaths.js):
- ✅ No `../` or absolute paths
- ✅ No null bytes
- ✅ Returns validated absolute path or throws `PathTraversalError`

**Applied At**:
1. **Pull**: Every remote entry path validated before writing
2. **Push**: Local paths normalized (relative from team mem dir)

**Code Flow**:
```typescript
// On pull
async writeRemoteEntriesToLocal(entries: Record<string, string>) {
  for (const [relPath, content] of Object.entries(entries)) {
    try {
      validatedPath = await validateTeamMemKey(relPath)
    } catch (e) {
      if (e instanceof PathTraversalError) {
        logForDebugging(`${e.message}`, { level: 'warn' })
        return false  // Skip this entry
      }
    }
    // ... write to validatedPath
  }
}

// On push
const relPath = relative(teamDir, fullPath).replaceAll('\\', '/')
// relPath is automatically relative, but validated elsewhere before upload
```

**Attack Surface**:
- ⚠️ Server could send entries with paths like `../../../etc/passwd`
- ✅ Client validates and rejects them (skips with warning)
- **Verdict**: Safe (validation in place)

**Edge Case**:
- Subdirectories are allowed (e.g., `patterns/react.md`)
- fs.watch ({recursive: true}) watches all subdirs
- **Safe**: No traversal possible (validation gates all paths)

---

## 6. INPUT VALIDATION & PARSING

### Schema Validation (Zod)

**Strength**: ✅ All API responses validated against strict Zod schemas

```typescript
const TeamMemoryDataSchema = z.object({
  organizationId: z.string(),
  repo: z.string(),
  version: z.number(),
  lastModified: z.string(),  // ISO 8601
  checksum: z.string(),      // sha256:<hex>
  content: z.object({
    entries: z.record(z.string(), z.string()),
    entryChecksums: z.record(z.string(), z.string()).optional(),
  }),
})
```

**Validation Points**:
1. **GET /api/claude_code/team_memory**: Parsed via `TeamMemoryDataSchema()`
2. **GET ?view=hashes**: Validated that `entryChecksums` is object (not string or null)
3. **PUT error responses**: Structured 413 parsed via `TeamMemoryTooManyEntriesSchema()`

**Behavior on Validation Failure**:
```typescript
const parsed = TeamMemoryDataSchema().safeParse(response.data)
if (!parsed.success) {
  return {
    success: false,
    error: 'Invalid team memory response format',
    skipRetry: true,  // Don't retry
    errorType: 'parse',
  }
}
```

**Strength**: ✅ Invalid responses treated as permanent failure (skip retry)

### Size Validation

**Pre-checks**:
- ✅ `MAX_FILE_SIZE_BYTES = 250,000` — stat before read, skip if > limit
- ✅ `MAX_PUT_BODY_BYTES = 200,000` — batching divides large deltas
- ⚠️ No per-entry UTF-8 byte count validation (content is strings, not bytes)

**Issue**:
- A 100-character UTF-8 string could theoretically be > 100 bytes (multibyte chars)
- Actual check uses `Buffer.byteLength(content, 'utf8')`
- **Mitigation**: ✅ filesize checked pre-read, batching checked pre-send

### ETag Escaping

**Code**:
```typescript
if (ifMatchChecksum) {
  headers['If-Match'] = `"${ifMatchChecksum.replace(/"/g, '')}"`
}
```

**Strength**: ✅ Quotes stripped from checksum before adding headers (prevents injection)

**Defensive**: Checksum is `sha256:<hex>`, so quotes would be unusual anyway

---

## 7. OFFLINE & RACE CONDITIONS

### Local Data Persistence

**Strength**: ✅ All edits persisted to disk before push attempted

**Flow**:
1. User edits team memory file
2. Watcher debounces (2 seconds)
3. `readLocalTeamMemory()` reads from disk
4. Push attempted
5. Even if push fails → file still on local disk

**Result**: No data loss on network failure

### Concurrent Edits (Same Device, Multiple Processes)

**Scenario**: Two Claude instances in same repo editing same team memory file

**Expected Behavior**:
1. Both pull from server (same content)
2. Both read local disk (same initial state)
3. Instance A pushes → succeeds
4. Instance B tries to push → hits 412 conflict
5. Instance B probes hashes, refreshes state
6. Instance B's delta recomputed
7. If same key: Instance B's version overwrites Instance A's (local-wins)

**Result**: **Last writer wins** (neither instance knows about the other's edits)

**Severity**: Low in practice (users don't typically run multiple Claude processes in parallel)

### Concurrent Edits (Different Devices, Same Repo)

**Scenario**: User A and User B both edit same file, push concurrently

**Expected Behavior**:
1. User A pushes first → succeeds with new ETag
2. User B's push (in progress) still using old ETag
3. User B's push hits 412 conflict
4. User B probes hashes
5. User B's delta computed
6. If User A's changes are to different keys: merged (delta includes User B's unique changes)
7. If to same key: User B's version overwrites User A's (local-wins on conflict)

**Result**: **Content-level merge of keys** (same-key conflicts: local-wins)

**Severity**: Expected (documented as no-content-merge design)

---

## 8. MEMORY POISONING & MALICIOUS SERVER

### Server Sends Malicious Content

**Threat**: Attacker controls backend, sends malicious team memory

**Examples**:
- XSS payload in team memory file
- Binary data instead of UTF-8
- Excessively large entries to exhaust disk/memory

**Mitigations**:
1. ✅ All entries are written as UTF-8 text files (not parsed/executed)
   - Files written via `writeFile(path, content, 'utf8')`
   - Never evaluated as JavaScript or HTML

2. ✅ Size validation pre-write
   - `MAX_FILE_SIZE_BYTES = 250,000` checked before writing
   - Entries > limit skipped

3. ✅ Path validation pre-write
   - `validateTeamMemKey()` prevents `../` traversal
   - Malicious paths like `../../.bashrc` rejected

4. ✅ Content is never executed
   - Team memory is read-only context for Claude
   - Not run as scripts or parsed as code

**Verdict**: ✅ **Server poisoning via team memory content is not exploitable**

### Server Sends Malformed JSON

**Threat**: Attacker sends invalid JSON or schema mismatch

**Mitigation**:
- ✅ Zod validation on all responses
- ✅ Parse failure → `skipRetry: true` (don't retry malformed)
- No fallback to partial parsing

**Verdict**: ✅ **Safe**

---

## 9. NETWORK & TRANSPORT SECURITY

### HTTPS & TLS

**Assumed**: ✅ All API calls use HTTPS (not explicitly shown in code, assumed by axios defaults)

**Headers**:
- ✅ `Authorization: Bearer <token>` (not in URL)
- ✅ Bearer token sent in headers (standard)
- ✅ `User-Agent` header included (prevents some attacks)

### ETag Injection

**Threat**: Attacker modifies ETag in response to break conflict detection

**Mitigation**:
- ✅ HTTPS prevents tampering (ETag extracted from header, not user input)
- ✅ ETag is SHA256 hash, so modification obvious (would fail client hash verification)

### Token in URL

**Risk**: If token ever ends up in URL → logged in referer headers, proxy logs

**Current Code**:
- ✅ Bearer token always in `Authorization` header (never in URL)
- ✅ `repo` slug is only URL param (public info)

---

## 10. LOGGING & INFORMATION DISCLOSURE

### What Gets Logged

**OK (Safe)**:
- ✅ File paths (relative to team memory dir)
- ✅ File counts (number of files pushed/pulled)
- ✅ Gitleaks rule IDs (e.g., "github-pat")
- ✅ Error types (auth, timeout, network)
- ✅ HTTP status codes
- ✅ Duration metrics

**NEVER logged**:
- ✅ Secret values (only rule IDs)
- ✅ Entry content (never logged)
- ✅ OAuth token (never logged)
- ✅ User IDs or org names

**Potential Issue**:
- ⚠️ File paths logged on secret detection
- ⚠️ Example: `team-memory-sync: skipping "api_keys/production.md" — detected GitHub PAT`
  - File path reveals existence of `api_keys/production.md`
  - But if file contains a secret, discovering its existence is secondary risk

**Verdict**: ✅ **Logging is conservative** (errs on side of hiding data)

### Debug Logging

**Function**: `logForDebugging(message, {level})`

**Levels**: debug, info, warn

**Access**: Requires debug flag enabled (not shown in code, assumed restricted)

**Content**:
- Error messages with path context
- Success counts and metrics
- Conflict retry details

**Verdict**: ✅ **Debug logs may contain more details, but assumed not accessible to untrusted users**

---

## 11. ANALYTICS & TELEMETRY

### Events Sent

**Safe Events**:
- ✅ `tengu_team_mem_sync_pull`: {success, files_written, not_modified, duration, errorType}
- ✅ `tengu_team_mem_sync_push`: {success, files_uploaded, conflict, duration}
- ✅ `tengu_team_mem_secret_skipped`: {file_count, rule_ids}  // No file paths, no values

**No Sensitive Data**:
- ✅ No content hashes sent (only counts)
- ✅ No file names sent
- ✅ No organizational data sent
- ✅ No user IDs sent

**Verdict**: ✅ **Analytics is privacy-preserving**

---

## 12. FEATURE FLAG & ROLLOUT

### TEAMMEM Build Flag

**Gated At**:
1. `startTeamMemoryWatcher()` → returns early if flag off
2. `checkTeamMemSecrets()` → is inert if flag off

**Implications**:
- ✅ Can be disabled globally (no partial rollout risk)
- ✅ Old bundles won't sync team memory
- ✅ Fresh rollout doesn't require hot-fix immediately

---

## 13. DEPENDENCY SECURITY

### External Libraries Used

```typescript
import axios from 'axios'              // HTTP client
import { createHash } from 'crypto'    // Node built-in
import { z } from 'zod/v4'              // Schema validation
import { mkdir, readdir, readFile, ...} from 'fs/promises'  // Node built-in
```

**Risk Assessment**:
- ✅ `axios` — widely used, assumed secure
- ✅ `crypto` — Node.js built-in, trusted
- ✅ `zod` — actively maintained, popular
- ⚠️ No dependency pinning shown (assumed handled by package.json)

---

## SUMMARY OF FINDINGS

### ✅ Strong Security Practices

1. **OAuth enforcement** — scoped tokens, required on every request
2. **Client-side secret scanning** — 44 high-confidence rules, prevents leaks
3. **Path validation** — blocks directory traversal
4. **Schema validation** — strict Zod schemas on API responses
5. **No data execution** — content written as files, never eval'd
6. **Checksum-based conflicts** — prevents data loss on concurrent edits
7. **Conservative logging** — secrets never logged
8. **Privacy-preserving analytics** — no content, no paths, no values

### ⚠️ Limitations & Gaps

1. **No PII scanning** — can't block personally identifiable info (names, emails, IDs)
2. **No server-side secret validation** — client-side gate can be bypassed (downgrade feature flag)
3. **No fine-grained permissions** — all org members see all repo team memory
4. **No deletion propagation** — deleted local files restore on pull (expected, but confusing)
5. **No content merge** — same-key conflicts: local wins (data loss possible)
6. **Custom secrets not detected** — only gitleaks rules (custom API keys pass through)
7. **File path in logs** — secret detection logs file path (reveals structure)

### 🎯 Recommendations for Anthropic

1. **Server-side secret scanning**: Scan team memory entries before storing (defense-in-depth)
2. **PII warning**: Document that team memory is org-shared; advise against storing PII
3. **Deletion API**: Add soft_delete or archive endpoint so deletions propagate
4. **Merge strategy**: Option for CRDT or vector-clock semantics for better multi-user support
5. **File-level audit logging**: Track who edited what when (server-side)
6. **Quarantine suspicious entries**: If server detects secrets, store but mark as quarantined
7. **User warning in UI**: Prominently warn when files with secrets are skipped
8. **Rate limiting**: Document server-side rate limits for team memory syncs

---

## CONCLUSION

**Verdict**: Anthropic's team memory sync is **security-conscious** with **production-grade secret protection**.

**Confidence**: 🟢 **HIGH** that:
- Credentials won't leak via team memory (client-side scanning is robust)
- Cross-tenant isolation works (OAuth + server ACL)
- Path traversal is blocked (validation in place)
- Data isn't executed (written as files)

**Concerns**: 🟡 **MEDIUM** on:
- Server-side secret validation gap (client can be downgraded)
- No PII blocking (false sense of safety)
- Same-key conflict resolution (local-wins can lose data)

**Overall Risk**: 🟢 **LOW** for typical use cases (team sharing patterns, decisions, code snippets)
