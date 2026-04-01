# Claude Code OAuth 2.0 Lifecycle: Deep Reverse Engineering Analysis

**Scope:** Complete OAuth implementation in `/src/services/oauth/` (5 files, ~1,051 lines)
**Date:** 2026-04-02
**Focus:** How Anthropic engineered OAuth; token lifecycle, storage, refresh, revocation, and security vectors

---

## Executive Summary

Claude Code implements OAuth 2.0 Authorization Code flow with PKCE (Proof Key for Code Exchange) for secure CLI authentication to Anthropic services. The implementation follows modern OAuth security practices with explicit attention to:

- **PKCE Protection:** Prevents authorization code interception attacks
- **State Parameter CSRF Protection:** Validates authorization code origin
- **Secure Token Storage:** macOS Keychain + file-based secure storage with encryption
- **Token Refresh:** Automatic expiration detection with 5-minute buffer; distributed lock-based refresh coordination
- **Distributed Lock Mechanism:** Cross-process token refresh synchronization prevents concurrent refreshes
- **Scope-Based Access Control:** Multiple OAuth scopes enable fine-grained permissions (inference, profile, MCP servers, file upload)
- **Fallback Support:** Manual auth code flow for non-interactive environments

---

## File-by-File Breakdown

### 1. **crypto.ts** - Cryptographic Utilities

**Purpose:** Generate PKCE parameters and state token for OAuth security.

**Functions:**

1. **`generateCodeVerifier(): string`**
   - Creates a random 32-byte code verifier
   - Base64-URL encodes the randomBytes(32) buffer
   - Removes padding characters (`=`), replaces `+` with `-`, `/` with `_`
   - Output: 43-44 character string (RFC 7636 compliant)
   - Used once per OAuth flow; stored in OAuthService instance memory

2. **`generateCodeChallenge(verifier: string): string`**
   - SHA256 hash of the code verifier
   - Base64-URL encodes the hash digest
   - Implements S256 method (only supported method, not plain)
   - Sent in authorization request as `code_challenge` param
   - Server stores this; later validates that `code_verifier` hashes to it

3. **`generateState(): string`**
   - Creates a random 32-byte state token
   - Base64-URL encodes the randomBytes(32) buffer
   - Used for CSRF protection: sent to auth server, returned in callback
   - Validated in AuthCodeListener.validateAndRespond()

**Security Model:**
- Uses `crypto.randomBytes()` for cryptographic randomness
- PKCE S256 (SHA256) is hardcoded; no fallback to plain
- All values are Base64-URL encoded to be URL-safe

---

### 2. **auth-code-listener.ts** - Local HTTP Callback Server

**Purpose:** Temporary localhost HTTP server that captures OAuth authorization code redirects.

**Architecture:**

The server listens on `localhost:PORT/callback` for the OAuth provider's redirect:
```
http://localhost:PORT/callback?code=AUTH_CODE&state=STATE
```

**Key Classes & Methods:**

1. **`AuthCodeListener` Class**
   - Lifecycle: Created in OAuthService.startOAuthFlow()
   - Destroyed: After auth code exchange or error

2. **`start(port?: number): Promise<number>`**
   - Calls `http.createServer()` with no request handler yet
   - Listens on OS-assigned port (port=0) or specified port
   - Returns the actual port number allocated by OS
   - Resolves before request handlers are attached

3. **`waitForAuthorization(state: string, onReady: () => Promise<void>): Promise<string>`**
   - Stores expected state for CSRF validation
   - Calls `onReady()` (triggers browser open + displays manual URL)
   - Returns promise that resolves when auth code is received
   - Attaches request handlers on first call

4. **`handleRedirect(req, res): void`**
   - Routes all requests; checks if pathname === `/callback`
   - Parses URL search params: `code` and `state`
   - Delegates to `validateAndRespond()`

5. **`validateAndRespond(authCode, state, res): void`**
   - **CSRF Check:** Validates `state === expectedState`; rejects if mismatch
   - **Code Validation:** Checks `authCode` is present; returns 400 if missing
   - Stores `res` object for later success/error redirect
   - Resolves the promise with the auth code
   - **Does NOT close the response yet** — response held open for success redirect

6. **`handleSuccessRedirect(scopes: string[], customHandler?): void`**
   - Called after token exchange succeeds (in OAuthService)
   - Routes based on scopes: if has `user:inference` scope → claude.ai success page; else → console success page
   - Sends 302 redirect to success page
   - Only executes if `pendingResponse` is set (automatic flow only)

7. **`handleErrorRedirect(): void`**
   - Called if token exchange fails
   - Sends 302 redirect to success page (TODO: should be error page)
   - Cleans up pending response

8. **`close(): void`**
   - Removes all event listeners (prevents memory leaks)
   - Closes the HTTP server
   - If pending response exists, calls handleErrorRedirect() first

**Security Properties:**

- **State Validation:** Line 164-169 in handleRedirect: Rejects if `state !== expectedState`
- **No Authentication Required:** This is not an OAuth server, just a redirect receiver
- **Localhost Binding:** Binds to `localhost` (not `0.0.0.0`); prevents network-based attacks
- **Callback Path:** Hardcoded to `/callback` (configurable in constructor but immutable after)
- **No HTTPS:** Runs on HTTP because it's localhost-only; acceptable per OAuth spec

**Token Leakage Vector:**
- Authorization code is visible in browser history (code is short-lived, used once)
- Response object held in memory after redirect; could theoretically leak if process crashes before redirect

---

### 3. **client.ts** - OAuth Client Implementation

**Purpose:** Build authorization URLs, exchange codes for tokens, refresh tokens, fetch profile info.

**Constants & Scopes:**

```typescript
CLAUDE_AI_INFERENCE_SCOPE = 'user:inference'
CLAUDE_AI_PROFILE_SCOPE = 'user:profile'
CONSOLE_SCOPE = 'org:create_api_key'

CONSOLE_OAUTH_SCOPES = ['org:create_api_key', 'user:profile']
CLAUDE_AI_OAUTH_SCOPES = ['user:profile', 'user:inference', 'user:sessions:claude_code', 'user:mcp_servers', 'user:file_upload']
ALL_OAUTH_SCOPES = union of all above
```

**OAuth Server Endpoints:**

```
Production:
  CONSOLE_AUTHORIZE_URL: https://platform.claude.com/oauth/authorize
  CLAUDE_AI_AUTHORIZE_URL: https://claude.com/cai/oauth/authorize (redirects to claude.ai)
  TOKEN_URL: https://platform.claude.com/v1/oauth/token
  ROLES_URL: https://api.anthropic.com/api/oauth/claude_cli/roles
  API_KEY_URL: https://api.anthropic.com/api/oauth/claude_cli/create_api_key
  MANUAL_REDIRECT_URL: https://platform.claude.com/oauth/code/callback
  CLIENT_ID: 9d1c250a-e61b-44d9-88ed-5944d1962f5e

Staging/Local:
  Different CLIENT_ID and base URLs (environment-configurable)
```

**Functions:**

1. **`buildAuthUrl({codeChallenge, state, port, isManual, ...options}): string`**
   - Builds URL to `CONSOLE_AUTHORIZE_URL` or `CLAUDE_AI_AUTHORIZE_URL`
   - Query parameters:
     - `code=true` (tells login page to show Claude Max upsell)
     - `client_id` (hardcoded or env override)
     - `response_type=code` (standard OAuth)
     - `redirect_uri`: automatic flow → `http://localhost:PORT/callback`; manual flow → `https://platform.claude.com/oauth/code/callback`
     - `scope` (space-separated; either all scopes or just `user:inference` if `inferenceOnly=true`)
     - `code_challenge` (PKCE S256 hash)
     - `code_challenge_method=S256` (always S256)
     - `state` (CSRF protection)
     - Optional: `orgUUID`, `login_hint`, `login_method`
   - Returns fully-formed URL string

2. **`exchangeCodeForTokens(authCode, state, codeVerifier, port, useManualRedirect, expiresIn?): Promise<OAuthTokenExchangeResponse>`**
   - **POST to TOKEN_URL**
   - Body:
     ```json
     {
       "grant_type": "authorization_code",
       "code": "AUTH_CODE",
       "redirect_uri": "http://localhost:PORT/callback" or manual URL,
       "client_id": "CLIENT_ID",
       "code_verifier": "FULL_43-CHAR_VERIFIER",
       "state": "STATE"
     }
     ```
   - Response: Axios POST with 15s timeout
   - Validates response.status === 200
   - Logs `tengu_oauth_token_exchange_success` event
   - Returns: `{ access_token, refresh_token, expires_in, scope, account: {...}, organization: {...} }`

3. **`refreshOAuthToken(refreshToken, {scopes?: string[]}): Promise<OAuthTokens>`**
   - **POST to TOKEN_URL** with grant_type=refresh_token
   - Body:
     ```json
     {
       "grant_type": "refresh_token",
       "refresh_token": "REFRESH_TOKEN",
       "client_id": "CLIENT_ID",
       "scope": "SCOPES or default CLAUDE_AI_OAUTH_SCOPES"
     }
     ```
   - Implements **Scope Expansion:** Backend allows requesting scopes beyond the original grant
   - Returns access token, new refresh token (or same if unchanged), expires_in
   - **Profile Fetch Optimization:** Skips `/api/oauth/profile` call if config already has billing info + secure storage has subscription data
     - Reduces ~7M requests/day fleet-wide
     - Falls back to fetch if either is missing
   - Updates config with new profile info: displayName, hasExtraUsageEnabled, billingType, accountCreatedAt, subscriptionCreatedAt
   - Merges new profile info with existing cached data (null coalescing)
   - Logs `tengu_oauth_token_refresh_success` or `tengu_oauth_token_refresh_failure`
   - **Token Lifetime Calculation:** `expiresAt = Date.now() + expiresIn * 1000`

4. **`fetchProfileInfo(accessToken): Promise<{subscriptionType, rateLimitTier, ...}>`**
   - Calls `getOauthProfileFromOauthToken(accessToken)` (see getOauthProfile.ts)
   - Parses `organization.organization_type` → subscription type mapping:
     - `claude_max` → `max`
     - `claude_pro` → `pro`
     - `claude_enterprise` → `enterprise`
     - `claude_team` → `team`
     - Unknown → `null`
   - Extracts: `rate_limit_tier`, `has_extra_usage_enabled`, `billing_type`, `display_name`, `created_at`, `subscription_created_at`
   - Logs `tengu_oauth_profile_fetch_success`

5. **`isOAuthTokenExpired(expiresAt: number | null): boolean`**
   - **5-Minute Buffer:** `now + 5*60*1000 >= expiresAt`
   - Returns `false` if expiresAt is null (env var tokens with unknown expiry)
   - Used before calling refreshOAuthToken()

6. **`createAndStoreApiKey(accessToken): Promise<string | null>`**
   - **POST to API_KEY_URL** (requires `org:create_api_key` scope)
   - Response: `{ raw_key: "api_key_string" }`
   - Calls `saveApiKey(apiKey)` to store in macOS Keychain or config
   - Logs `tengu_oauth_api_key` success/failure

7. **`fetchAndStoreUserRoles(accessToken): Promise<void>`**
   - **GET to ROLES_URL** with Bearer token
   - Response: `{ organization_role, workspace_role, organization_name }`
   - Saves to config.oauthAccount
   - Logs `tengu_oauth_roles_stored`

8. **`getOrganizationUUID(): Promise<string | null>`**
   - Checks config first (no network call if cached)
   - Falls back to profile fetch if config missing
   - Returns `profile.organization.uuid` or null

9. **`populateOAuthAccountInfoIfNeeded(): Promise<boolean>`**
   - Checks env vars first: `CLAUDE_CODE_ACCOUNT_UUID`, `CLAUDE_CODE_USER_EMAIL`, `CLAUDE_CODE_ORGANIZATION_UUID`
   - If env vars present, stores to config immediately (eliminates race condition with early telemetry)
   - Waits for any in-flight token refresh to complete
   - Checks if config already has full profile (billing type, account created, subscription created)
   - If missing, fetches profile and stores
   - Returns true if freshly populated, false if already cached

10. **`storeOAuthAccountInfo({accountUuid, emailAddress, organizationUuid, ...}): void`**
    - Saves to global config; used by profile fetch, env var population
    - Idempotent: checks if config already matches before updating

**Token Lifetime:**

- Access tokens: Issued with `expires_in` (seconds); converted to `expiresAt` (ms since epoch)
- Refresh tokens: No explicit expiry; assumed valid until server rejects
- Expiry check: 5-minute buffer means tokens are refreshed 5 minutes before actual expiry

---

### 4. **getOauthProfile.ts** - Profile Fetching

**Purpose:** Fetch OAuth profile info using access token or API key.

**Functions:**

1. **`getOauthProfileFromApiKey(): Promise<OAuthProfileResponse | undefined>`**
   - Requires: config.oauthAccount.accountUuid + API key
   - **GET to `/api/claude_cli_profile`**
   - Headers: `x-api-key`, `anthropic-beta: oauth-2025-04-20`
   - Params: `account_uuid`
   - Used as fallback if OAuth token unavailable

2. **`getOauthProfileFromOauthToken(accessToken): Promise<OAuthProfileResponse | undefined>`**
   - **GET to `/api/oauth/profile`**
   - Headers: `Authorization: Bearer {accessToken}`, `Content-Type: application/json`
   - 10s timeout
   - Silently logs errors (no throwing)
   - Returns undefined on network failures

**Response Structure (OAuthProfileResponse):**
```typescript
{
  account: {
    uuid: string
    email: string
    display_name?: string
    created_at?: string
  }
  organization: {
    uuid: string
    organization_type: 'claude_max' | 'claude_pro' | 'claude_enterprise' | 'claude_team'
    rate_limit_tier?: string
    has_extra_usage_enabled?: boolean
    billing_type?: string
    subscription_created_at?: string
  }
}
```

---

### 5. **index.ts** - OAuthService Main Orchestrator

**Purpose:** Coordinates entire OAuth 2.0 Authorization Code flow with PKCE.

**OAuthService Class:**

1. **Constructor**
   - Generates code verifier once
   - Stored in instance variable (lives until service destroyed)

2. **`startOAuthFlow(authURLHandler, options?): Promise<OAuthTokens>`**
   - **High-level orchestration of entire auth flow**
   - Options:
     - `loginWithClaudeAi`: Route to claude.ai or Console
     - `inferenceOnly`: Request only `user:inference` scope
     - `expiresIn`: Custom token lifetime
     - `orgUUID`: Pre-populate organization
     - `loginHint`, `loginMethod`: Pre-fill login form
     - `skipBrowserOpen`: Let caller manage URLs (SDK control protocol)

   **Flow Steps:**

   1. Create and start AuthCodeListener on random port
   2. Generate code challenge from verifier (PKCE)
   3. Generate state token (CSRF)
   4. Build two URLs:
      - Manual flow: `https://platform.claude.com/oauth/code/callback`
      - Automatic flow: `http://localhost:PORT/callback`
   5. Call authURLHandler with both URLs:
      - If `skipBrowserOpen=true`: Let caller decide where to open
      - Else: Show manual URL to user; open automatic URL in browser
   6. Wait for auth code via `waitForAuthorizationCode(state, onReady)`
   7. Log whether code came from automatic or manual flow
   8. Exchange code for tokens:
      - Calls `client.exchangeCodeForTokens(code, state, codeVerifier, port, isManual)`
      - Passes code verifier (proves we created the code challenge)
   9. Fetch profile info (subscription, rate limit tier)
   10. Handle success redirect (for automatic flow)
   11. Format and return OAuthTokens object
   12. Clean up AuthCodeListener on error or success

3. **`waitForAuthorizationCode(state, onReady): Promise<string>`**
   - Sets up manual auth code resolver
   - Delegates to AuthCodeListener.waitForAuthorization()
   - Handles both automatic and manual flows
   - Automatic: Browser hits `http://localhost:PORT/callback?code=...&state=...`
   - Manual: User pastes code via handleManualAuthCodeInput()

4. **`handleManualAuthCodeInput({authorizationCode, state}): void`**
   - Called when user pastes auth code in CLI prompt
   - Resolves the auth code promise
   - Closes auth code listener (no need for HTTP server)

5. **`formatTokens(response, subscriptionType, rateLimitTier, profile?): OAuthTokens`**
   - Transforms API response to OAuthTokens
   - Calculates `expiresAt = Date.now() + expires_in * 1000`
   - Parses scopes
   - Stores account UUID, email, org UUID if present in response

6. **`cleanup(): void`**
   - Closes AuthCodeListener
   - Clears manual auth code resolver

**OAuth Flow Sequence:**

```
User → cli /login
         ↓
    OAuthService.startOAuthFlow()
         ↓
    Start AuthCodeListener on localhost
    Generate PKCE: verifier + challenge
    Generate state token
         ↓
    Build URLs:
    - Manual: https://platform.claude.com/oauth/code/callback
    - Auto: http://localhost:PORT/callback
         ↓
    Call authURLHandler(manualUrl, autoUrl)
         ↓
    If not skipBrowserOpen:
    - Show manual URL to user
    - Open browser to auto URL
         ↓
    User logs in at OAuth provider
    Provider redirects to manual or auto URL
         ↓
    If auto: AuthCodeListener captures redirect
    If manual: User receives code, pastes in CLI
         ↓
    OAuthService receives auth code
         ↓
    Exchange code → tokens:
    POST /v1/oauth/token {
      grant_type: authorization_code
      code: ...
      code_verifier: ... (PKCE)
      state: ... (CSRF validation)
      redirect_uri: ... (matches auth request)
    }
         ↓
    Receive: {
      access_token
      refresh_token
      expires_in
      scope
      account: {uuid, email_address}
      organization: {uuid}
    }
         ↓
    Fetch profile info (subscription type, rate limit)
    GET /api/oauth/profile (Bearer token)
         ↓
    Format OAuthTokens:
    {
      accessToken
      refreshToken
      expiresAt: now + expires_in*1000
      scopes: parsed scope string
      subscriptionType: claude_max|pro|enterprise|team|null
      rateLimitTier: rate_limit_tier
      profile: full OAuth profile
      tokenAccount: {uuid, email, orgUuid}
    }
         ↓
    Handle success redirect (auto flow only)
    Redirect browser to success page
         ↓
    Return OAuthTokens to caller
```

---

## Token Storage & Secure Storage Architecture

### Storage Layers (src/utils/auth.ts & secureStorage/)

**File Storage Hierarchy:**

```
~/.claude/
  .credentials.json (Secure Storage backed by macOS Keychain or encrypted file)
    {
      claudeAiOauth: {
        accessToken: "..."
        refreshToken: "..."
        expiresAt: 1234567890000
        scopes: ["user:inference", "user:profile", ...]
        subscriptionType: "max" | "pro" | "enterprise" | "team" | null
        rateLimitTier: "..." | null
      }
    }
```

**Token Storage Function: `saveOAuthTokensIfNeeded(tokens): {success, warning?}`**

1. **Guards:**
   - Skips if scopes don't include `user:inference` (not Claude.ai auth)
   - Skips if `refreshToken` or `expiresAt` missing (inference-only env tokens)

2. **Smart Merge Logic:**
   - Preserves existing subscription data on refresh
   - Falls back to existing value if profile fetch returned null
   - Prevents losing subscription type on transient profile fetch failures

3. **Storage Update:**
   - Reads current secure storage
   - Merges new tokens with existing data
   - Calls `secureStorage.update(storageData)`
   - Clears memoization cache on success

4. **Backend Options:**
   - macOS: Keychain (via SecureStorage wrapper)
   - Linux: Encrypted file (libsecret or similar)
   - Windows: Credential Manager

### Token Retrieval: `getClaudeAIOAuthTokens(): OAuthTokens | null`

**Memoized function** (caches forever until explicit clear):

1. Check `--bare` mode: Return null if bare-only auth
2. Check env var `CLAUDE_CODE_OAUTH_TOKEN` (e.g., from parent process)
   - Returns inference-only token: `{accessToken, refreshToken: null, expiresAt: null, scopes: ['user:inference']}`
3. Check file descriptor `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`
   - Same inference-only format
4. Read secure storage
   - Returns `storageData.claudeAiOauth` or null
5. Logs errors; never throws

**Async Variant: `getClaudeAIOAuthTokensAsync(): Promise<OAuthTokens | null>`**

- Avoids blocking keychain reads
- Delegates to sync version for env/FD tokens
- Uses async `secureStorage.readAsync()` for file-based storage

### Cache Invalidation

**Problem:** Multiple Claude Code instances (terminal 1, 2, ...) read from same keychain. Instance 1 calls `/login` and refreshes tokens; instance 2's memoize cache is stale.

**Solution:**

1. **File Modification Time Check:** `invalidateOAuthCacheIfDiskChanged()`
   - Checks mtime of `~/.claude/.credentials.json`
   - Clears memoize cache if file changed (different process wrote)
   - Called at start of `checkAndRefreshOAuthTokenIfNeeded()`

2. **Keychain Cache TTL:** macOS keychain storage has 30s TTL
   - Sync reads bounce off memoize (forever) → keychain cache (30s) → actual keychain (slow)
   - If memoize misses, keychain cache ensures fresh reads every 30s

---

## Token Refresh Lifecycle

### Expiry Detection & Refresh Trigger

**Expiration Check:** `isOAuthTokenExpired(expiresAt: number | null): boolean`
- Returns true if `now + 5*60*1000 >= expiresAt` (5-minute buffer)
- Returns false if expiresAt is null (env tokens with unknown expiry)
- Prevents "token suddenly invalid" surprise when server time differs slightly

### Refresh Flow: `checkAndRefreshOAuthTokenIfNeeded(retryCount?, force?): Promise<boolean>`

**Architecture: Distributed Lock-Based Refresh**

Problem: Multiple Claude Code instances hit 401 simultaneously; N instances shouldn't all refresh concurrently (thundering herd, duplicate profile fetches).

Solution: File-based distributed lock in `~/.claude/`:

```
1. Check if token expired (local check with 5-min buffer)
2. Re-read from keychain async (another process might have refreshed already)
3. If still expired, acquire file lock: lockfile.lock(~/.claude/)
4. Check AGAIN after acquiring lock (prevent TOCTOU race)
5. If still expired:
   - Call refreshOAuthToken(refreshToken, {scopes?})
   - Save new tokens: saveOAuthTokensIfNeeded()
   - Clear memoize cache
6. Release lock
7. Return true if refresh succeeded, false otherwise
```

**Retry Logic:**
- Max 5 retries if lock is held by another process
- Exponential backoff: 1s + random(0-1s) per retry
- Gives other process time to finish refresh and release lock

**Concurrent Call Deduplication:**
- In-flight promise deduplication via `pendingRefreshCheck`
- Multiple threads calling `checkAndRefreshOAuthTokenIfNeeded()` without retryCount/force share same promise
- Prevents thrashing keychain on startup when multiple components initialize

### Refresh Error Handling

**On refresh failure:**
- Logs error
- Clears cache and re-reads from keychain
- If different token in keychain, another process succeeded → return true
- If same token still expired → return false
- Caller must handle token unavailable case

**Token Refresh Failure Scenario:**
1. User's subscription expires or is revoked server-side
2. Refresh returns 400 "invalid_grant" (refresh token revoked)
3. Must re-authenticate: `claude /login`

---

## Token Revocation & Logout

### Logout Handler (src/cli/handlers/auth.ts - referenced but not included in scope)

Logical flow (inferred from OAuthService cleanup):

1. **performLogout()**
   - Clears secure storage (removes tokens from keychain)
   - Clears memoize cache
   - Removes API key if stored

2. **Post-logout behavior:**
   - Next API call without token → 401 error
   - Auth layer catches 401 → prompts `/login`

### Manual Token Revocation

Not directly implemented in OAuth client. Server-side revocation happens when:
- User logs out on claude.ai
- User changes password
- User revokes CLI access in account settings
- Subscription cancelled or downgraded

Result: Refresh token becomes invalid → `invalid_grant` on next refresh attempt.

---

## PKCE Deep Dive

**Problem PKCE Solves:**

Authorization code grant vulnerable to interception in mobile/desktop apps (no secure backend):

```
Attacker intercepts authorization code in redirect
Attacker exchanges code for tokens (if they know client_id + secret)
```

PKCE prevents this by:

1. **Client generates code verifier:** Random 43-44 character string
   ```typescript
   codeVerifier = base64URLEncode(randomBytes(32))
   ```

2. **Client computes challenge:** SHA256 hash of verifier
   ```typescript
   codeChallenge = base64URLEncode(sha256(codeVerifier))
   ```

3. **Authorization request includes challenge:**
   ```
   GET /oauth/authorize?...&code_challenge=...&code_challenge_method=S256
   ```
   Server stores code_challenge in authorization record

4. **Code exchange includes verifier:**
   ```json
   POST /oauth/token {
     code: "AUTH_CODE",
     code_verifier: "ORIGINAL_43_CHAR_STRING"
   }
   ```

5. **Server validates verifier:**
   ```
   sha256(code_verifier) === stored_code_challenge
   ```
   If attacker has code but not verifier, they can't exchange it

**Implementation Details:**
- Code verifier generated once in OAuthService constructor
- Stored in instance memory (not persisted)
- Passed to `exchangeCodeForTokens()` which includes it in POST body
- If exchange fails, verifier is lost (new flow creates new verifier)
- Server uses S256 (SHA256), not plain method

---

## OAuth Endpoint Details

### Authorization Endpoint Behavior

**Request:**
```
GET https://platform.claude.com/oauth/authorize?
  client_id=9d1c250a-e61b-44d9-88ed-5944d1962f5e
  response_type=code
  redirect_uri=http://localhost:PORT/callback (or manual)
  scope=user:inference+user:profile+...
  code_challenge=HASH_OF_VERIFIER
  code_challenge_method=S256
  state=RANDOM_STATE
  code=true (show Max upsell)
  login_hint=user@example.com (optional)
  login_method=sso (optional)
  orgUUID=ORG_ID (optional)
```

**Redirect Response:**
- Automatic flow: Browser redirects to `http://localhost:PORT/callback?code=...&state=...`
- Manual flow: User receives code, pastes in CLI

### Token Endpoint

**POST** `https://platform.claude.com/v1/oauth/token`

Request Body (Authorization Code Grant):
```json
{
  "grant_type": "authorization_code",
  "code": "AUTHORIZATION_CODE",
  "redirect_uri": "http://localhost:PORT/callback",
  "client_id": "9d1c250a-e61b-44d9-88ed-5944d1962f5e",
  "code_verifier": "FULL_43_CHAR_VERIFIER",
  "state": "RANDOM_STATE"
}
```

Response (200 OK):
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "refresh_token": "LONG_STRING",
  "expires_in": 3600,
  "scope": "user:inference user:profile user:sessions:claude_code user:mcp_servers user:file_upload",
  "account": {
    "uuid": "ACCOUNT_UUID",
    "email_address": "user@example.com"
  },
  "organization": {
    "uuid": "ORG_UUID"
  }
}
```

Request Body (Refresh Token Grant):
```json
{
  "grant_type": "refresh_token",
  "refresh_token": "REFRESH_TOKEN_STRING",
  "client_id": "9d1c250a-e61b-44d9-88ed-5944d1962f5e",
  "scope": "user:inference user:profile ..."
}
```

### Profile Endpoint

**GET** `https://api.anthropic.com/api/oauth/profile`
- Header: `Authorization: Bearer {accessToken}`
- Response: Full OAuth profile with account, organization, subscription info

**GET** `https://api.anthropic.com/api/claude_cli_profile?account_uuid=...`
- Header: `x-api-key: {apiKey}`, `anthropic-beta: oauth-2025-04-20`
- Fallback if OAuth token unavailable

---

## Security Analysis: Threat Vectors & Mitigations

### 1. Authorization Code Interception (PKCE Mitigates)

**Threat:**
- Attacker intercepts authorization code in browser redirect
- Attacker exchanges code for tokens using their own client_id

**PKCE Mitigation:**
- Code verifier generated client-side; never transmitted
- Attacker without verifier cannot exchange code
- SHA256 hash is one-way; can't reverse from code_challenge

**Residual Risk:** Attacker controls the OAuth server (out of scope)

### 2. State Parameter CSRF Attack (STATE Mitigates)

**Threat:**
- Attacker tricks user into visiting `oauth/authorize` with attacker's parameters
- User approves, redirect to attacker's app with valid code
- Attacker uses code with their client_id

**Mitigation:**
- OAuthService generates random state token
- AuthCodeListener validates incoming state matches generated state
- Mismatch → 400 error, auth code rejected
- Attacker cannot predict state value

**Implementation:** Line 164-169 in auth-code-listener.ts:
```typescript
if (state !== this.expectedState) {
  res.writeHead(400)
  res.end('Invalid state parameter')
  this.reject(new Error('Invalid state parameter'))
  return
}
```

### 3. Access Token Leakage in Browser History

**Threat:**
- Access token is short-lived but visible in browser URL during redirect
- Browser history, logs, proxy logs could leak token

**Mitigation:**
- Authorization code (not access token) is in redirect URL
- Code is single-use, short-lived (minutes)
- Tokens are exchanged server-side via POST body (not URL)
- No sensitive data in URLs except temporary authorization code

**Residual Risk:** Replay of authorization code by attacker
- Mitigated by PKCE (attacker lacks code_verifier)
- Mitigated by redirect_uri validation (code only valid for registered URI)

### 4. Token Storage Compromise (Keychain/Secure Storage Mitigates)

**Threat:**
- Tokens stored in plaintext on disk
- Malware reads `~/.claude/.credentials.json`
- Attacker has access token + refresh token

**Mitigation:**
- macOS: Uses Keychain (encrypted by OS, per-user)
- Linux: Uses libsecret or encrypted file storage
- Windows: Uses Credential Manager
- Tokens never logged (checked via `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` type)
- Code verifier only in memory, never stored

**Residual Risk:**
- If attacker has local machine access (root), can read keychain
- If malware compromises process memory, can steal tokens mid-refresh
- Encrypted at-rest; encrypted in transit (HTTPS)

### 5. Token Replay Attack (Token Lifetime Mitigates)

**Threat:**
- Attacker captures token in transit or from storage
- Uses token to call API

**Mitigation:**
- Tokens are short-lived: 1 hour typical
- Tokens are issued with HTTPS-only transmission
- Tokens bound to specific OAuth client (client_id)
- Tokens bound to specific user account (stored in JWT claims)
- Server validates token signature (JWT)

**Residual Risk:**
- Attacker with token can use it for 1 hour
- Mitigated by OAuth application scope restrictions (can't do everything with inference token)

### 6. Refresh Token Compromise (Rotation + Revocation Mitigates)

**Threat:**
- Attacker steals refresh token
- Uses it to get new access tokens indefinitely

**Mitigation:**
- Refresh token can be revoked server-side (user logs out, changes password)
- Server-side revocation prevents future token exchanges
- Client validates token not expired before using
- Refresh tokens stored in Keychain (OS-level encryption)

**Residual Risk:**
- Until revocation, attacker can refresh indefinitely
- Mitigation requires user awareness (log out when suspicious)

### 7. Cross-Device Token Sharing (No Mitigation)

**Threat:**
- User copies refresh token to another device
- Both devices use same token simultaneously

**Note:** Not explicitly mitigated. Tokens are personal; not designed for sharing.

### 8. Localhost Binding Security

**Threat:**
- Attacker program listens on localhost:9999
- Tricks Claude Code into redirecting to attacker's localhost

**Mitigation:**
- AuthCodeListener binds to `localhost` (not `0.0.0.0`)
- Localhost is loopback; not accessible from network
- OS assigns port randomly; can't be pre-registered by attacker
- Attacker must already have code execution on machine

**Residual Risk:** If attacker has local code execution, can intercept redirects anyway

### 9. CSRF on Manual Flow

**Threat:**
- User visits attacker's website
- Website displays code prompt asking user to paste code
- User unknowingly enters code for attacker

**Mitigation:**
- State parameter still validated by AuthCodeListener
- Code must match expected state
- User must paste code voluntarily

**Residual Risk:** User social engineering (out of scope)

### 10. Clock Skew in Token Expiry

**Threat:**
- Client thinks token valid, server thinks it's expired (system clock ahead)
- Token accepted by server, later rejected

**Mitigation:**
- 5-minute buffer in expiry check
- If server rejects token with 401, client forces refresh immediately
- `handleOAuth401Error()` detects 401 and refreshes token
- Distributed lock ensures only one refresh happens
- Deduplication prevents race conditions

**Implementation:** Lines 1360-1392 in auth.ts

---

## Integration with Rest of System

### API Request Flow with OAuth

```
HTTP Request Handler (http.ts)
    ↓
checkAndRefreshOAuthTokenIfNeeded()
    ↓ (async, non-blocking)
getClaudeAIOAuthTokens()
    ↓
Access Token Valid?
    ├─ No → Refresh Token
    │       POST /v1/oauth/token + refreshToken
    │       Save new tokens
    │       Clear caches
    └─ Yes → Use token
                 ↓
            axios.request({
              headers: {
                Authorization: `Bearer ${accessToken}`
              }
            })
    ↓
Receive 200 or 401
    ├─ 200 → Return response
    └─ 401 → handleOAuth401Error(token)
             Force refresh + retry
```

### Keychain Prefetch Optimization

**Problem:** Keychain read on macOS is slow (~15-100ms per access).

**Solution:** Prefetch at startup (src/utils/secureStorage/keychainPrefetch.ts):
- Background spawn of `security find-generic-password` at main.tsx top-level
- Completes while imports are running (parallelism)
- Result cached for first getApiKeyFromConfigOrMacOSKeychain() call
- Avoids blocking initial API request

---

## Environment Variables & Configuration

### OAuth Configuration Overrides

```
USER_TYPE=ant → Allow staging/local OAuth
USE_LOCAL_OAUTH=true → Use localhost:8000 (developer only)
USE_STAGING_OAUTH=true → Use staging endpoints

CLAUDE_CODE_OAUTH_TOKEN=<token> → Force env-var token (service key)
CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR=N → Token from FD N
CLAUDE_CODE_ACCOUNT_UUID=<uuid> → Pre-populate account (SDK)
CLAUDE_CODE_USER_EMAIL=<email> → Pre-populate email (SDK)
CLAUDE_CODE_ORGANIZATION_UUID=<uuid> → Pre-populate org (SDK)

CLAUDE_CODE_OAUTH_CLIENT_ID=<client_id> → Override client ID (Xcode integration)
CLAUDE_CODE_CUSTOM_OAUTH_URL=<base_url> → Override all OAuth URLs (FedStart only)
```

### Allowlisted Custom OAuth Bases (Security Gating)

```
https://beacon.claude-ai.staging.ant.dev
https://claude.fedstart.com
https://claude-staging.fedstart.com
```

Only FedStart/PubSec deployments supported to prevent credential leakage.

---

## Token Lifecycle Summary Table

| Stage | Location | TTL | Format | Scope |
|-------|----------|-----|--------|-------|
| **Code Verifier** | Memory (OAuthService) | Flow duration (~5min) | 43-44 char base64url | N/A |
| **Auth Code** | Browser redirect | Minutes (backend configured) | Random string | Single-use |
| **Access Token** | Secure storage + memory (memoized) | 1 hour (typical) | JWT | User:inference + others |
| **Refresh Token** | Secure storage only (Keychain) | Months (no client-side expiry) | Opaque string | Refresh only |
| **State Token** | Memory (AuthCodeListener) | Flow duration | 32 bytes base64url | CSRF validation |

---

## Code Verifier / Challenge Flow Recap

```
┌─────────────────────────────────────────────────────────────────┐
│ Client (Claude Code)                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Step 1: Generate Verifier (one-time, never transmitted)       │
│ ──────────────────────────────────────────────────────────────│
│ verifier = base64URLEncode(randomBytes(32))                   │
│ → 43-44 character random string                               │
│                                                                 │
│ Step 2: Compute Challenge (sent to server)                    │
│ ──────────────────────────────────────────────────────────────│
│ challenge = base64URLEncode(sha256(verifier))                 │
│ → 43 character hash                                            │
│                                                                 │
│ Step 3: Authorization Request (browser)                        │
│ ──────────────────────────────────────────────────────────────│
│ GET /oauth/authorize?                                          │
│   ...&code_challenge=CHALLENGE&code_challenge_method=S256    │
│                                                                 │
│ → Server stores: challenge, method=S256                       │
│                                                                 │
│ Step 4: User Approves, Receives Code                          │
│ ──────────────────────────────────────────────────────────────│
│ Redirect: localhost/callback?code=AUTH_CODE&state=STATE      │
│                                                                 │
│ → Code tied to specific challenge in server                   │
│                                                                 │
│ Step 5: Token Exchange (PKCE Validation)                      │
│ ──────────────────────────────────────────────────────────────│
│ POST /oauth/token {                                            │
│   grant_type: authorization_code                              │
│   code: AUTH_CODE                                              │
│   code_verifier: ORIGINAL_43_CHAR_VERIFIER                    │
│ }                                                              │
│                                                                 │
│ → Server: sha256(verifier) === stored_challenge ✓            │
│ → If mismatch: Reject with 400 invalid_grant                 │
│ → If match: Return {access_token, refresh_token}             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Multi-Account / Organization Support

### Account Switching

Not fully supported in current implementation. However:

1. **Environment-based account switching:**
   - Set `CLAUDE_CODE_OAUTH_TOKEN` to different token
   - System uses that token instead of keychain token
   - Allows running CLI with different credentials per invocation

2. **Org UUID support:**
   - OAuth request can include `orgUUID` parameter
   - Pre-selects organization on login page
   - Useful for CLI in managed environments

3. **Account info storage:**
   - Only one account cached in `~/.claude/.credentials.json`
   - Multiple accounts not supported (would require account selection UI)
   - Token refresh always applies to stored account

---

## Error Handling & Resilience

### Network Failures During OAuth Flow

1. **AuthCodeListener socket error:**
   - Rejects with Error: "Failed to start OAuth callback server"
   - User sees error in CLI

2. **Token exchange network error:**
   - Axios 15s timeout
   - Retries handled by caller (auth.ts)
   - If persistent: User must retry `/login`

3. **Profile fetch failure:**
   - Silently logs error
   - Returns undefined
   - Falls back to existing profile data in config
   - Prevents total failure due to transient API issues

4. **Token refresh failure:**
   - Logs error
   - Retries with exponential backoff + distributed lock
   - Max 5 retries over ~5 seconds
   - If all fail: Returns false, caller handles token unavailable

### Stale Cache Prevention

**Problem:** Multiple CLI instances; instance 1 refreshes, instance 2 doesn't know.

**Solution:**
1. File mtime check: Detect if `~/.claude/.credentials.json` changed
2. Keychain cache TTL: macOS cache refreshed every 30s
3. Memoize cache: Explicitly cleared after refresh + on explicit clear commands

---

## Scope System

### Scope Meanings

1. **`user:inference`** - Use Claude API (read-only)
2. **`user:profile`** - Read account profile (name, org, subscription)
3. **`org:create_api_key`** - Create API keys for Console
4. **`user:sessions:claude_code`** - Manage CLI sessions (?)
5. **`user:mcp_servers`** - Manage MCP servers
6. **`user:file_upload`** - Upload files to Claude

### Scope Expansion on Refresh

Backend allows requesting scopes beyond original grant:
- User logs in with inference-only scope
- Later, feature needs file_upload scope
- Refresh request includes new scope
- Backend honors expansion (per `ALLOWED_SCOPE_EXPANSIONS` config)
- User never sees consent dialog again

Implementation (client.ts:159-162):
```typescript
scope: (requestedScopes?.length
  ? requestedScopes
  : CLAUDE_AI_OAUTH_SCOPES
).join(' ')
```

---

## Conclusion: Security Posture

**Strengths:**
1. ✓ PKCE prevents authorization code interception
2. ✓ State parameter prevents CSRF
3. ✓ Tokens stored in OS Keychain (encrypted)
4. ✓ 5-minute expiry buffer prevents surprise expirations
5. ✓ Distributed lock prevents concurrent refresh thundering herd
6. ✓ Localhost binding prevents network-level attacks
7. ✓ Scope expansion prevents re-auth friction

**Weaknesses:**
1. ✗ No token rotation on refresh (refresh token reused unless revoked)
2. ✗ Manual flow vulnerable to social engineering (user pastes code)
3. ✗ No explicit device binding (tokens valid on any machine)
4. ✗ Error redirect uses same page for success/error (TODO comment)
5. ✗ Profile fetch errors silently logged (could hide auth issues)

**Operational:**
- Solid cache invalidation strategy for multi-instance scenarios
- Exponential backoff + lock timeout prevents distributed deadlock
- Comprehensive error logging for debugging auth issues
- Analytics instrumentation for monitoring OAuth health

---

**End of Analysis**
