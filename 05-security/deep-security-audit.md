# Deep Security Audit: Data Flows, Secrets, Anti-Distillation & Attack Surface

**Added by:**  blind-spot analysis (April 2, 2026)
**Technique:** Secret scanning, data flow tracing, credential audit, attack surface mapping

---

## Anti-Distillation System

**Entirely missed by prior analysis. One of the most significant security mechanisms.**

When `ANTI_DISTILLATION_CC` compile flag is enabled and `tengu_anti_distill_fake_tool_injection` GrowthBook gate is active, Claude Code sends `anti_distillation` requests instructing the server to silently inject **decoy tool definitions** into the system prompt.

**Purpose:** Defensive measure against competitors recording Claude Code's API traffic to train their own models. Decoy tools corrupt any training data derived from intercepted prompts.

**Second mechanism:** `tengu_attribution_header` — kill-switch for the billing attribution header (`x-anthropic-billing-header`) with placeholder `cch=9fb25` replaced by a Zig-level hash at build time.

---

## Undercover Mode

**Also entirely missed by prior analysis.**

Activated via `CLAUDE_CODE_UNDERCOVER=1`. No force-off in external builds (dead-code elimination removes the toggle). System prompt instructs:

> "You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE repository. Your commit messages, PR titles, and PR bodies MUST NOT contain ANY Anthropic-internal information. Do not blow your cover. NEVER mention you are an AI."

**Effects:**
- Strips Co-Authored-By git metadata
- Hides internal codenames (Capybara, Tengu)
- Removes reviewer requirements and changelog sections
- Prevents mention of internal Slack channels and repo names
- Only available when `USER_TYPE='ant'` (Anthropic employees)

---

## Secret Scan Results

**Finding: Zero hardcoded API keys, credentials, passwords, or tokens in source code.** The codebase properly manages sensitive data through environment variables and OAuth token flows.

### Credential Management
- OAuth tokens: Stored in macOS Keychain (with cache TTL) or plaintext fallback
- API keys: Via env var `ANTHROPIC_API_KEY` or settings file
- Session tokens: Read from file descriptor, stored in in-memory STATE singleton
- Subprocess tokens: Written to `~/.claude/remote/.oauth_token` (mode 0o600)

### Secret Scanning Integration
Built-in gitleaks-based detection for: `anthropic-api-key`, `anthropic-admin-api-key`, `aws-access-token`, `gcp-api-key`, `azure-ad-client-secret`, `github-oauth`, `github-pat`, `sentry-user-token`, `sentry-org-token`.

---

## OAuth Token Flow Analysis

### Flow Path (No Intermediate Encryption):
1. File descriptor → `authFileDescriptor.ts:readTokenFromWellKnownFile()`
2. → `bootstrap/state.ts:setOauthTokenFromFd()` (direct mutation, no audit trail)
3. → `utils/auth.ts:getOAuthTokenFromFileDescriptor()` → passed to Anthropic SDK
4. Subprocess fallback: written to disk at `~/.claude/remote/.oauth_token` (0o600)

**Risk:** OAuth tokens and API keys are mixed into the STATE singleton without separate encryption or access control. Both passed to external SDKs without intermediate wrapping.

### CCR Container Hardening:
- Reads session token from `/run/ccr/session_token`
- Calls `prctl(PR_SET_DUMPABLE, 0)` to block ptrace heap scraping
- Downloads CA certificate
- Starts local CONNECT→WebSocket relay
- Unlinks token file (token remains heap-only)
- Injects `HTTPS_PROXY` and `SSL_CERT_FILE` environment variables

---

## Permission System Deep-Dive

### Multi-Stage Decision Flow:
Config → Hooks → Classifier → Interactive Prompt

All decisions funnel through `logPermissionDecision()` for zero audit blind spots.
- Four approval sources: hook > config > classifier > user
- Two rejection paths: config denylist or user rejection

### Hook Types (Previously Undocumented):

| Type | Trigger | Key Features |
|------|---------|-------------|
| BashCommandHook | Shell execution | timeout, statusMessage, once, async, asyncRewake |
| PromptHook | LLM evaluation | model override, timeout, once |
| AgentHook | Agentic verifier (Haiku default) | timeout, model override |
| HttpHook | POST JSON with header env interpolation | allowedEnvVars whitelist for $VAR expansion |

**asyncRewake:** Exit code 2 from a BashCommandHook runs in background and wakes the model on completion. Enables hooks that block tool execution until an external condition is met.

### Swarm Permission Security:
- Workers submit permission requests to team lead via file-based mailbox
- Leader shows interactive prompt, writes response back
- Worker polls every 500ms for response
- **Workers cannot self-approve requests** — enforced by architecture

---

## Bash Security Deep Findings

### Known Bugs in Security Code:
- `bashPermissions.ts:1799` — `stripCommentLines` has known bugs affecting permission rule handling
- `bashPermissions.ts:2274` — shell-quote library has documented single-quote backslash misparsing bug
- `powershellSecurity.ts:963` — Set-Alias/New-Alias can hijack future command resolution (static analysis limitation)

### Blocked Zsh Builtins (18):
zmodload, sysopen, zpty, ztcp, and 14 others — prevents Zsh-specific bypass vectors.

### Specific Attack Mitigations:
- Zsh equals expansion (`=curl` permission bypass) — explicitly blocked
- Unicode zero-width space injection — detected and rejected
- IFS null-byte injection — defended
- Malformed token bypass — patched after HackerOne report
- ANSI-C shell quoting — marked "EXTREMELY CAREFUL" in source

### Validator Resource Protection:
`MAX_SUBCOMMANDS_FOR_SECURITY_CHECK = 50` — prevents exponential validator starvation via deeply nested command chains.

---

## MCP Validation Gap

**Critical finding:** MCP tools use `z.object({}).passthrough()` Zod schema, meaning **tool inputs are NOT validated**. Also, `entrypoints/mcp.ts:136` has a TODO to validate input types with Zod — currently missing.

This means external MCP servers can send arbitrary JSON as tool inputs without schema enforcement.

---

## Swallowed Error Patterns (Security Risk)

Silent failures that could mask security issues:
1. `ink/ink.tsx` — `catch { /* stream may be destroyed */ }`
2. `main.tsx` — `catch { // Silently ignore errors - this is just for analytics }`
3. MCP client — reconnection retries swallow connection failures, potentially masking persistent auth issues

---

## File Operation Security

- `O_NOFOLLOW` flag prevents symlink attacks on task output files (Unix-only, unavailable on Windows)
- Task IDs: 36^8 combinations (2.8 trillion) to resist symlink prediction
- Memory paths: validated against traversal, root dirs, UNC paths, null bytes
- Deep link CWD: absolute paths only, reject ASCII control chars (0x00-0x1F, 0x7F)
- Atomic file writes: temp file → datasync → chmod → rename pattern for configs

---

## Analytics PII Protection

- `_PROTO_*` prefix for BigQuery proto columns (restricted access)
- Marker types force explicit verification: `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` — forced classification before logging
- `stripProtoFields()` removes `_PROTO_*` before Datadog routing
