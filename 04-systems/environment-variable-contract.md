# Environment Variable Contract

Generated: 2026-04-01
Extraction basis:
- environment-variable reads across the product codebase

## Bottom Line

The product references 473 unique environment variable names.

That does not mean all 473 are stable public knobs. Many are:

- test-only
- internal rollout/debug flags
- environment-detection signals
- provider-specific credentials
- bridge/session plumbing

Still, the source clearly exposes a large operational contract that is worth cataloging.

Companion raw inventory in this analysis bundle:

- `analysis/environment-variable-inventory.md`

## 1. High-Frequency Variables

Most frequently referenced environment variables:

| Variable | Approx. Ref Count | Main Meaning |
|---|---:|---|
| `USER_TYPE` | 357 | internal/external build behavior, ant-only surfaces |
| `NODE_ENV` | 43 | dev/test/prod branching |
| `CLAUDE_CODE_ENTRYPOINT` | 41 | mode of invocation: cli, sdk, local-agent, remote, desktop, etc. |
| `CLAUDE_CODE_REMOTE` | 27 | remote/session-bridge mode |
| `TERM_PROGRAM` | 17 | terminal integration behavior |
| `CLAUDE_CODE_SIMPLE` | 16 | reduced tool and prompt surface |
| `ANTHROPIC_API_KEY` | 16 | direct API auth |
| `CLAUDE_CODE_USE_BEDROCK` | 14 | provider routing |
| `CLAUDE_CODE_USE_VERTEX` | 13 | provider routing |
| `ANTHROPIC_BASE_URL` | 13 | API endpoint override |

This is enough to say the real execution mode is heavily environment-driven.

## 2. Major Categories

### 2A. Runtime Identity And Entry Mode

Examples:

- `USER_TYPE`
- `NODE_ENV`
- `CLAUDE_CODE_ENTRYPOINT`
- `CLAUDE_CODE_ACTION`
- `CLAUDE_CODE_ENVIRONMENT_KIND`
- `CLAUDE_CODE_SESSION_KIND`
- `CLAUDE_CODE_IS_COWORK`

These determine which product surface is active:

- interactive CLI
- SDK CLI
- local-agent
- desktop
- bridge
- remote
- GitHub Action

### 2B. Provider And Auth Routing

Examples:

- `ANTHROPIC_API_KEY`
- `ANTHROPIC_AUTH_TOKEN`
- `ANTHROPIC_BASE_URL`
- `ANTHROPIC_MODEL`
- `ANTHROPIC_SMALL_FAST_MODEL`
- `CLAUDE_CODE_USE_BEDROCK`
- `CLAUDE_CODE_USE_VERTEX`
- `CLAUDE_CODE_USE_FOUNDRY`
- `CLAUDE_CODE_SKIP_BEDROCK_AUTH`
- `CLAUDE_CODE_SKIP_VERTEX_AUTH`
- `AWS_BEARER_TOKEN_BEDROCK`
- `MCP_CLIENT_SECRET`
- `MCP_XAA_IDP_CLIENT_SECRET`

The provider contract is broader than “Anthropic API key”. The codebase actively supports multiple backends and proxying strategies.

### 2C. Remote / Bridge / Session Plumbing

Examples:

- `CLAUDE_CODE_REMOTE`
- `CLAUDE_CODE_REMOTE_SESSION_ID`
- `CLAUDE_CODE_REMOTE_MEMORY_DIR`
- `CLAUDE_CODE_SESSION_ACCESS_TOKEN`
- `CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR`
- `CLAUDE_BRIDGE_SESSION_INGRESS_URL`
- `CLAUDE_CODE_USE_CCR_V2`
- `CLAUDE_CODE_REMOTE_ENVIRONMENT_TYPE`
- `CLAUDE_CODE_WORKER_EPOCH`

These variables are a strong sign that session transport and remote execution are wired through environment handoff between cooperating processes.

### 2D. UX And Surface-Toggle Knobs

Examples:

- `CLAUDE_CODE_SIMPLE`
- `CLAUDE_CODE_BRIEF`
- `CLAUDE_CODE_PROACTIVE`
- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION`
- `CLAUDE_CODE_DISABLE_FAST_MODE`
- `CLAUDE_CODE_DISABLE_THINKING`
- `CLAUDE_CODE_DISABLE_TERMINAL_TITLE`
- `CLAUDE_CODE_STREAMLINED_OUTPUT`
- `CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER`
- `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`

These are often internal or automation-oriented, but they materially change the user-visible harness behavior.

### 2E. Tooling And Safety Overrides

Examples:

- `ENABLE_LSP_TOOL`
- `CLAUDE_CODE_DISABLE_COMMAND_INJECTION_CHECK`
- `CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR`
- `CLAUDE_CODE_USE_POWERSHELL_TOOL`
- `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING`
- `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`

These are important because they alter tool exposure, safety checks, or execution semantics.

### 2F. Plugin / Marketplace / Skill Control

Examples:

- `CLAUDE_CODE_SYNC_PLUGIN_INSTALL`
- `CLAUDE_CODE_SYNC_PLUGIN_INSTALL_TIMEOUT_MS`
- `CLAUDE_CODE_PLUGIN_CACHE_DIR`
- `CLAUDE_CODE_PLUGIN_SEED_DIR`
- `CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL`
- `CLAUDE_CODE_DISABLE_POLICY_SKILLS`
- `ENABLE_TOOL_SEARCH`

### 2G. Telemetry / Debug / Profiling

Examples:

- `CLAUDE_CODE_ENABLE_TELEMETRY`
- `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA`
- `OTEL_EXPORTER_OTLP_ENDPOINT`
- `OTEL_EXPORTER_OTLP_HEADERS`
- `OTEL_LOG_TOOL_DETAILS`
- `OTEL_LOG_USER_PROMPTS`
- `CLAUDE_CODE_PROFILE_QUERY`
- `CLAUDE_CODE_PROFILE_STARTUP`
- `CLAUDE_CODE_PERFETTO_TRACE`
- `CLAUDE_CODE_DEBUG_LOG_LEVEL`

This is a major operational contract in its own right.

### 2H. Test / Fixture / Internal Diagnostics

Examples:

- `CLAUDE_CODE_TEST_FIXTURES_ROOT`
- `VCR_RECORD`
- `FORCE_VCR`
- `CLAUDE_CODE_STALL_TIMEOUT_MS_FOR_TESTING`
- `CLAUDE_CODE_VERIFY_PLAN`
- `CLAUDE_CODE_FRAME_TIMING_LOG`
- `CLAUDE_CODE_COMMIT_LOG`
- `DEBUG_SDK`

## 3. Important Source-Derived Behaviors

### `CLAUDE_CODE_ENTRYPOINT`

This variable is used to label major execution modes, including:

- `cli`
- `sdk-cli`
- `sdk-ts`
- `sdk-py`
- `claude-vscode`
- `local-agent`
- `claude-desktop`
- `remote`
- `mcp`
- `claude-code-github-action`

This is one of the most important env vars in the repo because analytics, UI behavior, and remote handling all branch on it.

### `CLAUDE_CODE_SIMPLE`

This variable radically shrinks the tool surface. In simple mode, the system exposes only:

- `Bash`
- `Read`
- `Edit`

or `REPLTool` in REPL mode.

### `CLAUDE_CODE_REMOTE`

This appears across setup, analytics, printing, upstream proxying, memdir handling, and transport code. It is not cosmetic. It changes transport and session behavior throughout the app.

### `ENABLE_LSP_TOOL`

This variable enables the LSP tool surface.

### `OTEL_LOG_TOOL_DETAILS`

This variable gates detailed tool-name and tool-input logging. This is privacy-sensitive and intentionally off by default.

## 4. Stability Guidance

The source strongly suggests three stability tiers:

### Probably More Stable

- provider auth vars like `ANTHROPIC_API_KEY`
- provider routing vars like `CLAUDE_CODE_USE_BEDROCK`
- major mode vars like `CLAUDE_CODE_ENTRYPOINT` and `CLAUDE_CODE_REMOTE`
- telemetry endpoint vars under `OTEL_*`

### Likely Internal But Operationally Important

- `CLAUDE_CODE_USE_CCR_V2`
- `CLAUDE_CODE_SYNC_PLUGIN_INSTALL`
- `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION`
- `CLAUDE_CODE_DISABLE_*` flags
- bridge/session ingress vars

### Clearly Internal / Test / Dev

- `CLAUDE_CODE_VERIFY_PLAN`
- `CLAUDE_CODE_TEST_FIXTURES_ROOT`
- `CLAUDE_CODE_FRAME_TIMING_LOG`
- `FORCE_VCR`
- `VCR_RECORD`

## 5. Practical Extraction Value

Even without documenting all 473 names individually, the source yields several useful artifacts:

- a stable-enough “ops knobs” shortlist
- a provider-routing contract
- a remote-session handoff contract
- a telemetry/debugging contract
- a list of dangerous override flags that weaken safety or change behavior

If you want the exhaustive raw list, the command used here was:

```bash
rg -o --no-filename "process\\.env\\.[A-Z0-9_]+" src \
  | sed 's/process.env.//' \
  | sort | uniq
```

That currently returns 473 unique names.

For self-contained transfer, the full alphabetical inventory is preserved separately in:

- `analysis/environment-variable-inventory.md`
