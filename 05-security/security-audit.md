# Security Audit

Scope: targeted review of filesystem/path handling, shell execution, prompt/context re-entry, MCP/network surfaces, transcript parsing, and permission boundaries.

Self-contained reading note:

- file/path references in findings are provenance only
- each finding is written to stand on its own without requiring the original code

Assessment summary: no obvious critical remote-code-execution path jumped out in this pass, but there is one substantial permission-boundary weakness around subagents, plus a few medium/low integrity and network-hardening gaps. The codebase also contains several explicit, well-documented defenses against path traversal, symlink abuse, prompt injection, and OAuth mix-up attacks.

## Findings

### 1. High | gap | subagent permission widening
Description: `AgentTool.checkPermissions()` auto-allows subagent spawning outside auto mode, and normal worker subagents rebuild their tool pool under `selectedAgent.permissionMode ?? 'acceptEdits'`. The built-in general-purpose agent exposes `tools: ['*']` and does not define a stricter permission mode. In practice, a parent agent running under a tighter mode can spawn a child that receives an `acceptEdits` tool pool and broad tool access.

Exploitation scenario: if the parent model is constrained by a mode that would normally require approval for edits, it can call `AgentTool`, spawn a general-purpose subagent, and have that child edit files inside the working tree without the parent-level approval boundary. This is a genuine permission-boundary bypass, not just a UX inconsistency.

### 2. Medium | gap | non-interactive managed-settings acceptance
Description: dangerous remote-managed-settings changes are only surfaced through an interactive dialog. In non-interactive mode, `checkManagedSettingsSecurity()` returns `no_check_needed`, and the caller proceeds to apply the fetched settings.

Exploitation scenario: a headless, daemon, CI, or otherwise non-interactive session that receives newly-dangerous remote settings will accept them silently. If the remote settings source is compromised or misconfigured, policy-tightening assumptions made in interactive mode do not hold in unattended mode.

### 3. Medium | gap | weakened WebFetch hostname safety when preflight is skipped
Description: when `skipWebFetchPreflight` is enabled, WebFetch bypasses the `api.anthropic.com` domain safety check entirely. The remaining URL validation is shallow: parseable URL, no credentials, and a hostname containing at least one dot. That does not itself reject numeric IP literals, reserved ranges, or private/internal hostnames.

Exploitation scenario: in an enterprise deployment that enables `skipWebFetchPreflight`, a model with WebFetch permission to a specific hostname could request URLs like `https://169.254.169.254/...` or internal hostnames if network egress allows them. The permission layer still gates by hostname, but the public-resolvability safety property described in the comments is no longer enforced locally once the preflight is disabled.

### 4. Low | gap | silent malformed-JSONL dropping
Description: JSONL parsing silently drops malformed lines. That behavior is used in transcript and metadata loading paths via `parseJSONL`.

Exploitation scenario: a local actor who can tamper with transcript or analytics files can corrupt selected JSONL lines to hide or suppress entries instead of forcing a hard failure. This is primarily an integrity/auditability weakness, not a privilege-escalation path, because the attacker already needs write access to the underlying files.

### 5. Low | gap | cross-process XAA refresh races
Description: XAA silent refresh currently deduplicates only within a single process. The code explicitly notes the absence of a cross-process lockfile and the resulting race on shared token storage.

Exploitation scenario: multiple Claude Code processes refreshing the same expiring MCP/XAA credentials can race on token exchange and storage updates. The comment notes this is not a token-bricking condition today, but it remains a cross-process auth-state integrity gap and can amplify token churn or inconsistent auth state.

### 6. Informational | defense | `src/tools/BashTool/pathValidation.ts:110-117`
Description: the shell path validator explicitly handles POSIX `--` end-of-options semantics so path extraction is not bypassed by adversarial arguments that begin with `-`. The comment documents the exact attack shape (`rm -- -/../...`) being defended against.

### 7. Informational | defense | `src/utils/permissions/pathValidation.ts:150-210`
Description: path permission evaluation is ordered defensively: deny rules first, then internal-path exceptions, then safety checks, and only after that working-directory/`acceptEdits` fast paths. This prevents `acceptEdits` from bypassing dangerous-file and Claude-config protections.

### 8. Informational | defense | `src/utils/Shell.ts:299-312`
Description: shell task output files are opened with `O_NOFOLLOW` on POSIX, explicitly preventing symlink-following attacks against the task-output path.

### 9. Informational | defense | `src/utils/messages.ts:1797-1855`, `src/utils/transcriptSearch.ts:44-59`, `src/utils/transcriptSearch.ts:117-126`, `src/utils/queryHelpers.ts:430-438`
Description: the code has multiple prompt-injection mitigations around `<system-reminder>` content. Attachment-origin text is wrapped in `<system-reminder>`, reminder siblings are folded into tool results to avoid malformed wire framing, transcript search strips reminder content because it is model-facing rather than user-visible, and read-result reconstruction strips reminder blocks before caching file content.

### 10. Informational | defense | `src/utils/permissions/filesystem.ts:352-369`
Description: bundled-skill extraction uses a per-process random nonce in its temp-root path specifically to defend against shared-`/tmp` pre-creation, intermediate-directory symlink attacks, and post-write prompt-injection swaps.

### 11. Informational | defense | `src/utils/hooks/execAgentHook.ts:138-150`
Description: hook agents do not receive broad filesystem bypasses for transcript access. They are given a narrow session rule scoped to `Read(/${transcriptPath})`, which is a better design than inheriting blanket read access.

### 12. Informational | defense | `src/services/mcp/xaa.ts:132-200`
Description: the XAA/OAuth discovery flow adds explicit mix-up defenses beyond the SDK defaults: protected-resource resource matching, authorization-server issuer matching, and mandatory HTTPS token endpoints before any token exchange occurs.

## Notes By Review Topic

### Path traversal and filesystem
- Strong on command/path parsing and write-path validation. The code explicitly documents prior bypass classes and places safety checks before `acceptEdits` auto-allow logic.
- Symlink handling is defense-in-depth rather than a single mechanism: `realpath` checks, `O_NOFOLLOW`, randomized temp roots, and deny-first permission evaluation all show up in the relevant paths.

### Command injection
- The Bash validation surface is mature and unusually adversarial in its parsing assumptions. I did not find an obvious shell-metacharacter or wrapper-based escape in the files reviewed here.

### Prompt injection and context re-entry
- The codebase is aware that model-facing tool-result serialization differs from user-visible rendering and actively strips or wraps `<system-reminder>` blocks in several places.
- Residual risk remains wherever arbitrary tool output is intentionally reintroduced into context, but the specific reminder-tag re-entry class appears consciously mitigated.

### Secrets, logging, and deserialization
- The largest issue here is integrity, not direct secret exfiltration: JSONL consumers prefer partial recovery over strict failure, which is operationally convenient but weakens tamper visibility.

### MCP / supply chain / network
- Remote MCP/XAA handling contains solid protocol-level defenses, especially around issuer/resource matching and HTTPS enforcement.
- The softer spots are operational: token refresh races across processes and the fact that WebFetch can be locally downgraded from “externally validated hostname” to “syntactically plausible URL” by a settings flag intended for restrictive enterprises.

## Highest-Risk Pairs

1. `AgentTool` + permission modes
The current composition allows a model to step around a stricter parent mode by moving work into a child with `acceptEdits`.

2. Remote managed settings + non-interactive sessions
The human-approval control disappears exactly in the unattended environments where surprise policy changes are hardest to notice quickly.

3. WebFetch + `skipWebFetchPreflight`
The opt-out setting removes the only explicit public-domain safety check without replacing it with a local reserved-address or private-host denial layer.

## Bottom Line

Most of the low-level hardening is good: path parsing, symlink resistance, prompt-wrapper handling, and MCP/XAA validation are all above average. The main security concern in this pass is policy composition: subagents currently look capable of widening permissions relative to the caller, which undercuts the surrounding permission model.
