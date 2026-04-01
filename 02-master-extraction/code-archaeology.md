# Code Archaeology: Developer Annotations, Dead Code & Technical Debt

**Added by:**  blind-spot analysis (April 2, 2026)
**Technique:** grep for TODO/FIXME/HACK/WORKAROUND/DEPRECATED/WARNING + dead code detection + orphan analysis

---

## Critical TODOs (Deferred Work Revealing Incomplete Systems)

### Unimplemented Features
- `utils/secureStorage/index.ts:14` — **Linux libsecret support not implemented** (macOS Keychain works, Linux falls back to plaintext)
- `utils/thinking.ts:88` — Model capability probing via API error detection not yet supported
- `entrypoints/mcp.ts:136` — TODO: validate input types with zod (currently missing — MCP inputs NOT validated)
- `services/mcp/xaa.ts:133,176,229` — XAA validation and token endpoint auth methods deferred to GA
- `services/mcp/auth.ts:1743` — Cross-process lockfile needed before GA

### Wire Protocol Fragility
- `tools/AgentTool/forkSubagent.ts:154` — Wire protocol issue: `[tool_result, text]` pattern creates undesired message structure
- `utils/messages.ts:2674` — Recursive field stringification can still leak through sanitization
- `tasks/RemoteAgentTask.tsx:459` — ExitPlanModeScanner needs folding into poller; uses deprecated startDetachedPoll

### Plugin System Gaps
- `utils/plugins/pluginLoader.ts:3242` — Installed plugins cache never cleared; npm package support missing
- `utils/plugins/marketplaceManager.ts:1619` — npm package support unimplemented
- `utils/plugins/pluginOptionsStorage.ts:156` — Merged settings return from deprecated method needs redesign

---

## Workarounds (Known External/Internal Issues Being Hacked Around)

| File | Issue | Workaround |
|------|-------|------------|
| `utils/proxy.ts:343` | axios/axios#4531 | Custom proxy handling for axios bug |
| `utils/skills/skillChangeDetector.ts:59` | Bun FSWatcher deadlock | stat() polling instead of native watcher |
| `utils/staticRender.tsx:8` | Ink doesn't support multiple `<Static>` elements | Custom render implementation |
| `services/analytics/growthbook.ts:51,79,330,378,694` | SDK bug in remote eval | Response format transformation |
| `utils/bash/ShellSnapshot.ts:129` | GNU find vs other implementations | Longer alternative command listed first |
| `utils/computerUse/executor.ts:109` | handleScroll mouse_full behavior | Mouse event workaround |
| `utils/api.ts:626` | Tokens Claude can't see | Special visibility handling |

---

## Known Bugs (Documented in Source)

- `tools/BashTool/bashPermissions.ts:1799` — **`stripCommentLines` has known bugs** affecting permission rule handling
- `tools/BashTool/bashPermissions.ts:2274` — **shell-quote library has documented single-quote backslash misparsing bug**
- `tools/PowerShellTool/powershellSecurity.ts:963` — Set-Alias/New-Alias can hijack future command resolution (static analysis limitation)
- `services/mcp/xaaIdpLogin.ts:197` — URL construction edge case with leading slashes

---

## Deprecated but Active Code

Active deprecated functions still in use (migration ongoing, not complete):
- `ink/log-update.ts:52` — `renderPreviousOutput_DEPRECATED()`
- `utils/settings/settings.js` — `getSettings_DEPRECATED()`
- `utils/slowOperations.js` — `writeFileSync_DEPRECATED()`
- `utils/config.ts:176` — `'emacs'` mode kept for backward compatibility
- Multiple `_DEPRECATED` suffixed functions across secure storage, settings, and config modules

---

## Performance-Instrumented Hotspots

The rendering pipeline is the most instrumented code in the codebase:
- `ink/ink.tsx:247-777` — Extensive `performance.now()` tracking for render → diff → optimize → write pipeline
- `ink/render-to-screen.ts:84-198` — Phase-by-phase timing breakdown
- `ink/reconciler.ts:287,311` — `SLOW_YOGA` and `SLOW_PAINT` diagnostic logging for layout bottleneck detection
- `ink/log-update.ts:460` — Slow render detection (>5ms threshold)

---

## Dead Code in External Builds

### Ant-Only Code (88 locations)
`"external" === 'ant'` guards code unreachable in public builds across:
- `main.tsx` (14 guards) — event loop stall detector, session data uploading, ANT org warnings, native auto-updater
- `screens/REPL.tsx` — ANT-specific UI elements
- `tools/` — task mode config, model overrides
- `components/` — 8 files with internal-only UI

### Non-Existent Module Reference
`main.tsx:429` — imports `./utils/eventLoopStallDetector.js` which **does not exist in codebase**. Dead code referencing a removed module.

### Feature-Gated Tools Never Enabled in Public Builds

| Tool | Feature Gate | Lines |
|------|-------------|-------|
| `RemoteTriggerTool` | AGENT_TRIGGERS_REMOTE | Full directory |
| `SleepTool` | PROACTIVE or KAIROS | Full directory |
| `MonitorTool` | MONITOR_TOOL | Full directory |
| `SubscribePRTool` | KAIROS_GITHUB_WEBHOOKS | Full directory |
| `SendUserFileTool` | KAIROS | Full directory |
| `PushNotificationTool` | KAIROS or KAIROS_PUSH_NOTIFICATION | Full directory |
| `WebBrowserTool` | WEB_BROWSER_TOOL | Full directory |
| `OverflowTestTool` | OVERFLOW_TEST_TOOL | Full directory |
| `TerminalCaptureTool` | TERMINAL_PANEL | Full directory |
| `CtxInspectTool` | CONTEXT_COLLAPSE | Full directory |
| `SnipTool` | HISTORY_SNIP | Full directory |
| `ListPeersTool` | UDS_INBOX | Full directory |
| `WorkflowTool` | WORKFLOW_SCRIPTS | Full directory |

### Feature Flags with No Corresponding Runtime Gate (Potentially Dead)
ABLATION_BASELINE, AGENT_MEMORY_SNAPSHOT, BG_SESSIONS, BUILDING_CLAUDE_APPS, BYOC_ENVIRONMENT_RUNNER, LODESTONE, OVERFLOW_TEST_TOOL, PERFETTO_TRACING, RUN_SKILL_GENERATOR

---

## Validation Gaps

- `entrypoints/mcp.ts:136` — MCP tool inputs **not validated with Zod** (TODO in source)
- `tools/MCPTool` — Uses `.passthrough()` Zod schema, **skipping all validation**
- `tools/SyntheticOutputTool` — Also uses `.passthrough()`, no validation
- `tools/PowerShellTool/powershellSecurity.ts:968` — Set-Alias effects can't be validated statically

---

## Security-Relevant Developer Notes

- `tools/WebFetchTool/preapproved.ts:5` — **SECURITY WARNING: preapproved domains ONLY for GET requests**
- `tools/AskUserQuestionTool:257` — onclick handlers possible; consumers must sanitize
- `tools/SendMessageTool:592` — Cross-machine prompt injection must stay bypass-immune
- `tools/FileEditTool/utils.ts:527-636` — Sanitization/desanitization pattern: strings sanitized by API that Claude can't see must be de-sanitized in edit operations. Critical but fragile API boundary.
- `tools/BashTool/bashSecurity.ts:195` — "EXTREMELY CAREFUL" note about ANSI-C shell quoting

---

## Future Plans (Roadmap Hints from Comments)

- `constants/apiLimits.ts:10` — Dynamic limits fetching from server (#13240)
- `services/mcp/auth.ts:1743` — Cross-process lockfile needed before MCP OAuth GA
- `services/lsp/LSPServerManager.ts:374` — LSP integration with compact system pending
- `tools/SkillTool:528` — Default to requiring permission in future
- `utils/plugins/schemas.ts:432,463` — Glob support in plugin config, gist support, single file support planned
