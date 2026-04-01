# Feature Flag Encyclopedia

Method:
- Build-time flags were extracted from build-time feature checks.
- Runtime flags were extracted from GrowthBook helper reads plus constant-backed helper calls.
- Build-time default is `false` unless Bun's bundle enables the feature.
- Runtime default is the default argument shown below; when GrowthBook is unavailable, helper fallbacks are described in the final sections.

Self-contained reading note:

- file paths in the `Notes` column are provenance only
- the transferable artifact is the flag name, default, and behavioral meaning
- treat this document as a product capability/control catalog, not as a file index

## How to interpret flags in this document

- Many runtime flag names are opaque rollout codenames rather than descriptive product names.
- In those cases, the `What It Controls` column is the canonical meaning and should be preferred over the flag name itself.
- `Kill Switch?` means the flag is explicitly usable as a rollback or disable lever for an already-shipped behavior.
- Build-time flags remove or include code at build time; runtime flags change behavior in a built client.

## Build-time (DCE)

| Flag Name | Type | Default | What It Controls | Kill Switch? | Notes |
|---|---|---|---|---|---|
| `ABLATION_BASELINE` | Build-time (DCE) | `false` | CLI ablation/baseline entrypoint code | No | `src/entrypoints/cli.tsx` |
| `AGENT_MEMORY_SNAPSHOT` | Build-time (DCE) | `false` | Agent memory snapshot loading/persistence | No | `src/main.tsx`, `src/tools/AgentTool/loadAgentsDir.ts` |
| `AGENT_TRIGGERS` | Build-time (DCE) | `false` | Scheduled task / cron-trigger tools, UI, and bundled loop skill support | No | `src/tools.ts`, `src/tools/ScheduleCronTool/prompt.ts`, `src/screens/REPL.tsx` |
| `AGENT_TRIGGERS_REMOTE` | Build-time (DCE) | `false` | Remote-facing scheduled trigger surfaces | No | `src/tools.ts`, `src/skills/bundled/index.ts` |
| `ALLOW_TEST_VERSIONS` | Build-time (DCE) | `false` | Native installer acceptance of `99.99.x` test builds | No | `src/utils/nativeInstaller/download.ts` |
| `ANTI_DISTILLATION_CC` | Build-time (DCE) | `false` | Anti-distillation Claude Code API behavior | No | `src/services/api/claude.ts` |
| `AUTO_THEME` | Build-time (DCE) | `false` | Automatic theme setting/provider support | No | `src/components/design-system/ThemeProvider.tsx`, `src/tools/ConfigTool/supportedSettings.ts` |
| `AWAY_SUMMARY` | Build-time (DCE) | `false` | Away summary hook and REPL surfaces | No | `src/hooks/useAwaySummary.ts`, `src/screens/REPL.tsx` |
| `BASH_CLASSIFIER` | Build-time (DCE) | `false` | Bash command classifier, explanations, and permission UI | No | `src/utils/permissions/*`, `src/components/permissions/*`, `src/cli/structuredIO.ts` |
| `BG_SESSIONS` | Build-time (DCE) | `false` | Background session lifecycle and concurrency features | No | `src/main.tsx`, `src/query.ts`, `src/utils/concurrentSessions.ts` |
| `BREAK_CACHE_COMMAND` | Build-time (DCE) | `false` | Context/cache breaking slash-command support | No | `src/context.ts` |
| `BRIDGE_MODE` | Build-time (DCE) | `false` | IDE/extension bridge subsystem | No | `src/bridge/*`, `src/commands/bridge/index.ts`, `src/main.tsx` |
| `BUDDY` | Build-time (DCE) | `false` | Buddy sprite, prompts, notifications, and related UI | No | `src/buddy/*`, `src/components/PromptInput/PromptInput.tsx` |
| `BUILDING_CLAUDE_APPS` | Build-time (DCE) | `false` | Bundled Claude Apps authoring/building skill exposure | No | `src/skills/bundled/index.ts` |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Build-time (DCE) | `false` | Built-in Explore/Plan agent definitions | No | `src/tools/AgentTool/builtInAgents.ts` |
| `BYOC_ENVIRONMENT_RUNNER` | Build-time (DCE) | `false` | Bring-your-own environment runner wiring | No | `src/entrypoints/cli.tsx` |
| `CACHED_MICROCOMPACT` | Build-time (DCE) | `false` | Cached microcompact pipeline and prompt reuse paths | No | `src/services/compact/microCompact.ts`, `src/services/api/claude.ts` |
| `CCR_AUTO_CONNECT` | Build-time (DCE) | `false` | Auto-connect behavior for CCR/bridge startup | No | `src/bridge/bridgeEnabled.ts`, `src/utils/config.ts` |
| `CCR_MIRROR` | Build-time (DCE) | `false` | CCR mirror mode | No | `src/bridge/bridgeEnabled.ts`, `src/bridge/remoteBridgeCore.ts` |
| `CCR_REMOTE_SETUP` | Build-time (DCE) | `false` | Remote setup command registration | No | `src/commands.ts` |
| `CHICAGO_MCP` | Build-time (DCE) | `false` | Chicago/Computer Use MCP tools and UI | No | `src/services/mcp/client.ts`, `src/services/mcp/config.ts`, `src/query.ts` |
| `COMMIT_ATTRIBUTION` | Build-time (DCE) | `false` | Commit attribution capture, restore, and post-compact cleanup | No | `src/utils/attribution.ts`, `src/utils/sessionRestore.ts`, `src/screens/REPL.tsx` |
| `COMPACTION_REMINDERS` | Build-time (DCE) | `false` | Attachment-time compaction reminders | No | `src/utils/attachments.ts` |
| `CONNECTOR_TEXT` | Build-time (DCE) | `false` | Connector-text message blocks and related beta plumbing | No | `src/services/api/claude.ts`, `src/utils/messages.ts`, `src/constants/betas.ts` |
| `CONTEXT_COLLAPSE` | Build-time (DCE) | `false` | Context collapse commands, restore paths, and token warning behavior | No | `src/commands/context/*`, `src/services/compact/*`, `src/screens/*` |
| `COORDINATOR_MODE` | Build-time (DCE) | `false` | Coordinator-specific tools, env mode, and UI paths | No | `src/coordinator/coordinatorMode.ts`, `src/main.tsx`, `src/cli/print.ts` |
| `COWORKER_TYPE_TELEMETRY` | Build-time (DCE) | `false` | Coworker-type analytics fields | No | `src/services/analytics/metadata.ts` |
| `DAEMON` | Build-time (DCE) | `false` | Daemon command/startup flow | No | `src/commands.ts`, `src/entrypoints/cli.tsx` |
| `DIRECT_CONNECT` | Build-time (DCE) | `false` | Direct-connect remote transport flow | No | `src/main.tsx` |
| `DOWNLOAD_USER_SETTINGS` | Build-time (DCE) | `false` | Pulling/downloading synced user settings | No | `src/services/settingsSync/index.ts`, `src/cli/print.ts` |
| `DUMP_SYSTEM_PROMPT` | Build-time (DCE) | `false` | System-prompt dump entrypoint | No | `src/entrypoints/cli.tsx` |
| `ENHANCED_TELEMETRY_BETA` | Build-time (DCE) | `false` | Enhanced telemetry instrumentation code paths | No | `src/utils/telemetry/sessionTracing.ts` |
| `EXPERIMENTAL_SKILL_SEARCH` | Build-time (DCE) | `false` | Skill discovery/search UX and MCP integration | No | `src/query.ts`, `src/utils/attachments.ts`, `src/services/mcp/useManageMCPConnections.ts` |
| `EXTRACT_MEMORIES` | Build-time (DCE) | `false` | Automatic memory extraction pipeline | No | `src/services/extractMemories/*`, `src/utils/backgroundHousekeeping.ts` |
| `FILE_PERSISTENCE` | Build-time (DCE) | `false` | File persistence timing and persistence-specific UI hooks | No | `src/utils/filePersistence/filePersistence.ts`, `src/cli/print.ts` |
| `FORK_SUBAGENT` | Build-time (DCE) | `false` | Fork-subagent command and agent fork behavior | No | `src/commands/branch/index.ts`, `src/tools/AgentTool/forkSubagent.ts` |
| `HARD_FAIL` | Build-time (DCE) | `false` | Hard-fail logging/development behavior | No | `src/main.tsx`, `src/utils/log.ts` |
| `HISTORY_PICKER` | Build-time (DCE) | `false` | Prompt history picker UI | No | `src/components/PromptInput/PromptInput.tsx`, `src/hooks/useHistorySearch.ts` |
| `HISTORY_SNIP` | Build-time (DCE) | `false` | History snipping and related attachment/context behavior | No | `src/query.ts`, `src/utils/messages.ts`, `src/utils/collapseReadSearch.ts` |
| `HOOK_PROMPTS` | Build-time (DCE) | `false` | REPL hook prompts | No | `src/screens/REPL.tsx` |
| `IS_LIBC_GLIBC` | Build-time (DCE) | `false` | Compile-time glibc classification | No | `src/utils/envDynamic.ts` |
| `IS_LIBC_MUSL` | Build-time (DCE) | `false` | Compile-time musl classification | No | `src/utils/envDynamic.ts` |
| `KAIROS` | Build-time (DCE) | `false` | Core Kairos/assistant mode surfaces across commands, UI, storage, and transport | No | Very broad: `src/main.tsx`, `src/commands.ts`, `src/screens/REPL.tsx`, `src/cli/print.ts` |
| `KAIROS_BRIEF` | Build-time (DCE) | `false` | Brief-mode command/UI/tooling | No | `src/commands/brief.ts`, `src/components/*Brief*`, `src/screens/REPL.tsx` |
| `KAIROS_CHANNELS` | Build-time (DCE) | `false` | Kairos channel UI/MCP notification surfaces | No | `src/services/mcp/channelNotification.ts`, `src/main.tsx`, `src/components/LogoV2/*` |
| `KAIROS_DREAM` | Build-time (DCE) | `false` | Bundled dream skill entry | No | `src/skills/bundled/index.ts` |
| `KAIROS_GITHUB_WEBHOOKS` | Build-time (DCE) | `false` | GitHub webhook subscription command/tooling | No | `src/commands.ts`, `src/hooks/useReplBridge.tsx`, `src/tools.ts` |
| `KAIROS_PUSH_NOTIFICATION` | Build-time (DCE) | `false` | Push notification settings/tooling | No | `src/components/Settings/Config.tsx`, `src/tools.ts` |
| `LODESTONE` | Build-time (DCE) | `false` | Deep-link registration and related background startup hooks | No | `src/utils/deepLink/registerProtocol.ts`, `src/utils/backgroundHousekeeping.ts` |
| `MCP_RICH_OUTPUT` | Build-time (DCE) | `false` | Rich MCP output rendering | No | `src/tools/MCPTool/UI.tsx` |
| `MCP_SKILLS` | Build-time (DCE) | `false` | MCP-backed skills/resource loading | No | `src/services/mcp/client.ts`, `src/services/mcp/useManageMCPConnections.ts` |
| `MEMORY_SHAPE_TELEMETRY` | Build-time (DCE) | `false` | Memory access-shape telemetry | No | `src/memdir/findRelevantMemories.ts`, `src/utils/sessionFileAccessHooks.ts` |
| `MESSAGE_ACTIONS` | Build-time (DCE) | `false` | Message actions UI and keybindings | No | `src/screens/REPL.tsx`, `src/keybindings/defaultBindings.ts` |
| `MONITOR_TOOL` | Build-time (DCE) | `false` | Monitor/background task tools and related permission UI | No | `src/tasks/*`, `src/tools/BashTool/*`, `src/components/tasks/*` |
| `NATIVE_CLIENT_ATTESTATION` | Build-time (DCE) | `false` | Native client attestation header support | No | `src/constants/system.ts` |
| `NATIVE_CLIPBOARD_IMAGE` | Build-time (DCE) | `false` | Native clipboard image path | No | `src/utils/imagePaste.ts` |
| `NEW_INIT` | Build-time (DCE) | `false` | New `/init` flow | No | `src/commands/init.ts` |
| `OVERFLOW_TEST_TOOL` | Build-time (DCE) | `false` | Overflow test tool registration | No | `src/tools.ts`, `src/utils/permissions/classifierDecision.ts` |
| `PERFETTO_TRACING` | Build-time (DCE) | `false` | Perfetto tracing support | No | `src/utils/telemetry/perfettoTracing.ts` |
| `POWERSHELL_AUTO_MODE` | Build-time (DCE) | `false` | PowerShell-specific auto-mode guidance/classifier behavior | No | `src/utils/permissions/permissions.ts`, `src/utils/permissions/yoloClassifier.ts` |
| `PROMPT_CACHE_BREAK_DETECTION` | Build-time (DCE) | `false` | Prompt cache break detection and related compact/API logic | No | `src/services/api/claude.ts`, `src/services/compact/*`, `src/commands/compact/compact.ts` |
| `PROACTIVE` | Build-time (DCE) | `false` | Proactive assistant behavior and related prompts/UI | No | `src/main.tsx`, `src/constants/prompts.ts`, `src/components/Messages.tsx` |
| `QUICK_SEARCH` | Build-time (DCE) | `false` | Quick search triggers and keybindings | No | `src/components/PromptInput/PromptInput.tsx`, `src/keybindings/defaultBindings.ts` |
| `REACTIVE_COMPACT` | Build-time (DCE) | `false` | Reactive-only compaction paths and token buffer logic | No | `src/services/compact/autoCompact.ts`, `src/utils/analyzeContext.ts` |
| `REVIEW_ARTIFACT` | Build-time (DCE) | `false` | Review artifact surfaces | No | `src/components/permissions/PermissionRequest.tsx`, `src/skills/bundled/index.ts` |
| `RUN_SKILL_GENERATOR` | Build-time (DCE) | `false` | Bundled skill-generator exposure | No | `src/skills/bundled/index.ts` |
| `SELF_HOSTED_RUNNER` | Build-time (DCE) | `false` | Self-hosted runner entrypoint code | No | `src/entrypoints/cli.tsx` |
| `SHOT_STATS` | Build-time (DCE) | `false` | Shot stats collection/rendering | No | `src/utils/stats.ts`, `src/utils/statsCache.ts`, `src/components/Stats.tsx` |
| `SKILL_IMPROVEMENT` | Build-time (DCE) | `false` | Skill improvement hook registration | No | `src/utils/hooks/skillImprovement.ts` |
| `SLOW_OPERATION_LOGGING` | Build-time (DCE) | `false` | Slow-operation logging variants | No | `src/utils/slowOperations.ts` |
| `SSH_REMOTE` | Build-time (DCE) | `false` | SSH remote mode | No | `src/main.tsx` |
| `STREAMLINED_OUTPUT` | Build-time (DCE) | `false` | Streamlined CLI output mode | No | `src/cli/print.ts` |
| `TEAMMEM` | Build-time (DCE) | `false` | Team memory files, sync, and UI | No | `src/memdir/*`, `src/services/teamMemorySync/*`, `src/components/messages/*teamMem*` |
| `TEMPLATES` | Build-time (DCE) | `false` | Template file support in query and permission paths | No | `src/utils/markdownConfigLoader.ts`, `src/query.ts`, `src/utils/permissions/filesystem.ts` |
| `TERMINAL_PANEL` | Build-time (DCE) | `false` | Terminal panel tool, keybinding, and related UI | No | `src/tools.ts`, `src/hooks/useGlobalKeybindings.tsx` |
| `TOKEN_BUDGET` | Build-time (DCE) | `false` | Token budget UI, prompts, and query handling | No | `src/query.ts`, `src/components/PromptInput/PromptInput.tsx`, `src/screens/REPL.tsx` |
| `TORCH` | Build-time (DCE) | `false` | Torch command | No | `src/commands.ts` |
| `TRANSCRIPT_CLASSIFIER` | Build-time (DCE) | `false` | Auto-mode transcript classifier, prompts, permission flow, and UI | No | Very broad: `src/utils/permissions/*`, `src/main.tsx`, `src/services/api/claude.ts` |
| `TREE_SITTER_BASH` | Build-time (DCE) | `false` | Native tree-sitter Bash parser | No | `src/utils/bash/parser.ts` |
| `TREE_SITTER_BASH_SHADOW` | Build-time (DCE) | `false` | Shadow-mode tree-sitter Bash parse path | No | `src/tools/BashTool/bashPermissions.ts`, `src/utils/bash/parser.ts` |
| `UDS_INBOX` | Build-time (DCE) | `false` | UDS inbox/send-message/peer tooling | No | `src/tools/SendMessageTool/*`, `src/main.tsx`, `src/cli/print.ts` |
| `ULTRAPLAN` | Build-time (DCE) | `false` | Ultraplan command, dialogs, and trigger detection | No | `src/commands/ultraplan.tsx`, `src/components/PromptInput/PromptInput.tsx`, `src/screens/REPL.tsx` |
| `ULTRATHINK` | Build-time (DCE) | `false` | Ultrathink mode support | No | `src/utils/thinking.ts` |
| `UNATTENDED_RETRY` | Build-time (DCE) | `false` | Unattended retry code path in API retries | No | `src/services/api/withRetry.ts` |
| `UPLOAD_USER_SETTINGS` | Build-time (DCE) | `false` | Pushing/syncing user settings upward | No | `src/main.tsx`, `src/services/settingsSync/index.ts` |
| `VERIFICATION_AGENT` | Build-time (DCE) | `false` | Verification-agent instructions, agent list, and completion nudges | No | `src/constants/prompts.ts`, `src/tools/AgentTool/builtInAgents.ts`, `src/tools/TodoWriteTool/TodoWriteTool.ts` |
| `VOICE_MODE` | Build-time (DCE) | `false` | Voice mode UI, transport, keybindings, and STT integration | No | `src/screens/REPL.tsx`, `src/hooks/useVoiceIntegration.tsx`, `src/services/voiceStreamSTT.ts` |
| `WEB_BROWSER_TOOL` | Build-time (DCE) | `false` | Browser/web tool registration | No | `src/tools.ts`, `src/main.tsx`, `src/screens/REPL.tsx` |
| `WORKFLOW_SCRIPTS` | Build-time (DCE) | `false` | Workflow script tools and permission surfaces | No | `src/tools.ts`, `src/constants/tools.ts`, `src/tasks.ts` |

## Runtime (GrowthBook)

| Flag Name | Type | Default | What It Controls | Kill Switch? | Notes |
|---|---|---|---|---|---|
| `enhanced_telemetry_beta` | Runtime boolean | `false` | Enables enhanced session tracing for external users | No | `src/utils/telemetry/sessionTracing.ts` |
| `tengu-off-switch` | Runtime config | `{ activated: false }` | Global API-side off switch checked before specific Claude API behavior | Yes | `src/services/api/claude.ts`; blocking dynamic config |
| `tengu_1p_event_batch_config` | Runtime config | `{}` | Batch exporter tuning for first-party event logging (`scheduledDelayMillis`, queue/batch sizes, retry/path/baseUrl) | No | Constant-backed in `src/services/analytics/firstPartyEventLogger.ts` |
| `tengu_agent_list_attach` | Runtime boolean | `false` | Whether agent lists are attached in messages/prompts | No | `src/tools/AgentTool/prompt.ts` |
| `tengu_amber_flint` | Runtime boolean | `true` | External gate for experimental agent swarms/agent teams | Yes | Explicit kill switch in `src/utils/agentSwarmsEnabled.ts` |
| `tengu_amber_json_tools` | Runtime boolean | `false` | Enables JSON tool-use beta/header when strict tools are off | No | `src/utils/betas.ts` |
| `tengu_amber_prism` | Runtime boolean | `false` | Adds memory correction hint text when auto memory is enabled | No | `src/utils/messages.ts` |
| `tengu_amber_quartz_disabled` | Runtime boolean | `false` | Emergency remote kill switch for voice mode | Yes | `src/voice/voiceModeEnabled.ts` |
| `tengu_amber_stoat` | Runtime boolean | `true` | Keeps built-in Explore/Plan agents enabled for external builds unless experiment removes them | No | `src/tools/AgentTool/builtInAgents.ts` |
| `tengu_ant_model_override` | Runtime config | `null` | Ant-only model alias/override config, including always-on-thinking model metadata | No | `src/utils/model/antModels.ts`, `src/hooks/useMainLoopModel.ts` |
| `tengu_anti_distill_fake_tool_injection` | Runtime boolean | `false` | Enables anti-distillation fake-tool injection behavior in Claude API calls | No | `src/services/api/claude.ts` |
| `tengu_attribution_header` | Runtime boolean | `true` | Controls sending the attribution header when not disabled by env | No | `src/constants/system.ts` |
| `tengu_auto_background_agents` | Runtime boolean | `false` | Turns on auto-backgrounding of agent tasks after a delay | No | `src/tools/AgentTool/AgentTool.tsx` |
| `tengu_auto_mode_config` | Runtime config | `{}` | Master auto-mode config: availability (`enabled`), fast-mode breaker, force-external-permissions, model override, two-stage classifier, JSONL transcript, allowlisted models | Yes | Used across `src/utils/permissions/*`, `src/utils/betas.ts`, `src/services/mcp/vscodeSdkMcp.ts` |
| `tengu_basalt_3kr` | Runtime boolean | `false` | Enables MCP instruction delta behavior for non-ant users | No | `src/utils/mcpInstructionsDelta.ts` |
| `tengu_birch_trellis` | Runtime boolean | `true` | Shadow-mode tree-sitter Bash parsing gate | Yes | Explicit kill switch in `src/tools/BashTool/bashPermissions.ts` |
| `tengu_bramble_lintel` | Runtime numeric config | `null` | Throttles extract-memories to every N eligible turns | No | `src/services/extractMemories/extractMemories.ts`; `null` falls back to `1` |
| `tengu_bridge_initial_history_cap` | Runtime numeric config | `200` | Caps initial bridge history sent during bridge startup | No | `src/bridge/initReplBridge.ts` |
| `tengu_bridge_min_version` | Runtime config | `{ minVersion: '0.0.0' }` | Minimum required bridge version for v1 bridge path | Yes | `src/bridge/bridgeEnabled.ts` |
| `tengu_bridge_poll_interval_config` | Runtime config | `DEFAULT_POLL_CONFIG` | Bridge polling interval/backoff configuration | No | `src/bridge/pollConfig.ts` |
| `tengu_bridge_repl_v2` | Runtime boolean | `false` | Enables env-less/remote REPL bridge v2 path | No | `src/bridge/bridgeEnabled.ts`, `src/bridge/initReplBridge.ts` |
| `tengu_bridge_repl_v2_config` | Runtime config | `DEFAULT_ENV_LESS_BRIDGE_CONFIG` | Config blob for bridge REPL v2 (including min version) | No | `src/bridge/envLessBridgeConfig.ts` |
| `tengu_bridge_repl_v2_cse_shim_enabled` | Runtime boolean | `true` | Keeps the bridge CSE shim enabled for REPL v2 | Yes | Default-on rollback lever in `src/bridge/bridgeEnabled.ts` |
| `tengu_bridge_system_init` | Runtime boolean | `false` | Enables bridge system-init path in REPL bridge setup | No | `src/hooks/useReplBridge.tsx` |
| `tengu_ccr_bridge` | Runtime boolean/gate | `false` | Main CCR bridge entitlement gate | No | Fast cached path plus blocking check in `src/bridge/bridgeEnabled.ts` |
| `tengu_ccr_bridge_multi_session` | Runtime gate | `false` | Allows multiple concurrent CCR bridge sessions | No | `src/bridge/bridgeMain.ts` |
| `tengu_ccr_bundle_max_bytes` | Runtime numeric config | `null` | Max bytes allowed for CCR git bundle upload | No | `src/utils/teleport/gitBundle.ts`; `null` means use built-in limit |
| `tengu_ccr_bundle_seed_enabled` | Runtime gate | `false` | Enables bundle seeding for remote/background sessions and teleport | No | `src/utils/background/remote/remoteSession.ts`, `src/utils/teleport.tsx` |
| `tengu_ccr_mirror` | Runtime boolean | `false` | Enables CCR mirror mode | No | `src/bridge/bridgeEnabled.ts` |
| `tengu_chair_sermon` | Runtime gate | `false` | Enables “chair sermon” message cleanup/repair behavior | No | `src/utils/messages.ts` |
| `tengu_chomp_inflection` | Runtime boolean | `false` | Enables prompt suggestions and exposes its config toggle | No | `src/services/PromptSuggestion/promptSuggestion.ts`, `src/components/Settings/Config.tsx` |
| `tengu_chrome_auto_enable` | Runtime boolean | `false` | Auto-enables Claude in Chrome integration when the extension is installed | No | `src/utils/claudeInChrome/setup.ts` |
| `tengu_cicada_nap_ms` | Runtime numeric config | `0` | Startup prefetch throttle in milliseconds | No | `src/main.tsx` |
| `tengu_cobalt_frost` | Runtime boolean | `false` | Switches voice streaming STT to Deepgram Nova 3 / conversation engine path | No | `src/services/voiceStreamSTT.ts` |
| `tengu_cobalt_harbor` | Runtime boolean | `false` | Auto-connects sessions to CCR/harbor path | No | `src/bridge/bridgeEnabled.ts` |
| `tengu_cobalt_lantern` | Runtime boolean | `false` | Enables remote setup/web setup prerequisites for remote agents | No | `src/commands/remote-setup/index.ts`, `src/utils/background/remote/preconditions.ts` |
| `tengu_cobalt_raccoon` | Runtime boolean | `false` | Puts reactive compaction into “reactive-only” mode by skipping reserved buffer / session-memory attempt | No | `src/services/compact/autoCompact.ts`, `src/utils/analyzeContext.ts`, `src/components/TokenWarning.tsx` |
| `tengu_collage_kaleidoscope` | Runtime boolean | `true` | Uses native macOS clipboard image reader instead of osascript fallback | Yes | Explicit default-on kill switch in `src/utils/imagePaste.ts` |
| `tengu_compact_cache_prefix` | Runtime boolean | `true` | Enables prompt-cache-sharing prefix reuse during compaction | Yes | Explicit rollback lever in `src/services/compact/compact.ts` |
| `tengu_compact_line_prefix_killswitch` | Runtime boolean | `false` | Disables compact line-prefix rendering when flipped on | Yes | `src/utils/file.ts` |
| `tengu_compact_streaming_retry` | Runtime boolean | `false` | Retries regular streaming compaction path on failure | No | `src/services/compact/compact.ts` |
| `tengu_copper_bridge` | Runtime boolean | `false` | Enables Claude-in-Chrome MCP bridge URL/path | No | `src/utils/claudeInChrome/mcpServer.ts` |
| `tengu_copper_panda` | Runtime boolean | `false` | Enables post-sampling skill-improvement detector/hook | No | `src/utils/hooks/skillImprovement.ts` |
| `tengu_coral_fern` | Runtime boolean | `false` | Adds the “Searching past context” memory section | No | `src/memdir/memdir.ts` |
| `tengu_cork_m4q` | Runtime boolean | `false` | Uses system-prompt policy spec in shell prefix handling | No | `src/utils/shell/prefix.ts` |
| `tengu_desktop_upsell` | Runtime config | `{ enable_shortcut_tip: false, enable_startup_dialog: false }` | Desktop upsell startup dialog and shortcut-tip rollout | No | `src/components/DesktopUpsell/DesktopUpsellStartup.tsx` |
| `tengu_destructive_command_warning` | Runtime boolean | `false` | Shows destructive-command warning text in Bash/PowerShell permission prompts | No | `src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx`, `src/components/permissions/PowerShellPermissionRequest/PowerShellPermissionRequest.tsx` |
| `tengu_disable_bypass_permissions_mode` | Runtime gate/security gate | `false` | Disables bypass-permissions mode / acts as a circuit breaker after auth changes | Yes | `src/utils/permissions/permissionSetup.ts` |
| `tengu_disable_keepalive_on_econnreset` | Runtime boolean | `false` | Disables HTTP keepalive reuse after `ECONNRESET` in retry logic | No | `src/services/api/withRetry.ts` |
| `tengu_disable_streaming_to_non_streaming_fallback` | Runtime boolean | `false` | Disables fallback from streaming API path to non-streaming path | No | `src/services/api/claude.ts` |
| `tengu_dunwich_bell` | Runtime boolean | `false` | Enables memory feedback survey | No | Constant-backed in `src/components/FeedbackSurvey/useMemorySurvey.tsx` |
| `tengu_enable_settings_sync_push` | Runtime boolean | `false` | Enables pushing settings sync updates upstream | No | `src/services/settingsSync/index.ts` |
| `tengu_event_sampling_config` | Runtime config | `{}` | Per-event analytics sampling config (`sample_rate` per event name) | No | Constant-backed in `src/services/analytics/firstPartyEventLogger.ts` |
| `tengu_fgts` | Runtime boolean | `false` | Enables fine-grained tool streaming in first-party API calls | No | `src/utils/api.ts` |
| `tengu_frond_boric` | Runtime config | `{}` | Per-sink analytics kill map (`datadog`, `firstParty`) | Yes | Constant-backed in `src/services/analytics/sinkKillswitch.ts`; fail-open on malformed config |
| `tengu_glacier_2xr` | Runtime boolean | `false` | Enables deferred-tools delta messaging/prompting | No | `src/utils/toolSearch.ts`, `src/tools/ToolSearchTool/prompt.ts` |
| `tengu_grey_step2` | Runtime config | `OPUS_DEFAULT_EFFORT_CONFIG_DEFAULT` | Controls Opus default effort rollout, especially for Max/Team | No | `src/utils/effort.ts`, `src/components/EffortCallout.tsx` |
| `tengu_harbor` | Runtime boolean/gate | `false` | Overall channels on/off gate | No | `src/services/mcp/channelAllowlist.ts`, `src/interactiveHelpers.tsx` |
| `tengu_harbor_ledger` | Runtime list config | `[]` | Allowlist of approved channel plugins (`{ marketplace, plugin }[]`) | No | `src/services/mcp/channelAllowlist.ts` |
| `tengu_harbor_permissions` | Runtime boolean | `false` | Enables channel-based permission approvals | No | `src/services/mcp/channelPermissions.ts` |
| `tengu_hawthorn_steeple` | Runtime boolean | `false` | Enables content-replacement bookkeeping for persisted tool results | No | `src/utils/toolResultStorage.ts` |
| `tengu_hawthorn_window` | Runtime numeric config | `null` | Overrides per-message aggregate persisted tool-result budget | No | `src/utils/toolResultStorage.ts`; `null` falls back to hardcoded limit |
| `tengu_herring_clock` | Runtime boolean | `false` | Enables Team Memory cohort / TeamMem pathing | No | `src/memdir/teamMemPaths.ts`, `src/memdir/memdir.ts` |
| `tengu_hive_evidence` | Runtime boolean | `false` | Enables verification-agent contract, built-in verifier agent, and completion nudges | No | `src/constants/prompts.ts`, `src/tools/TaskUpdateTool/TaskUpdateTool.ts`, `src/tools/TodoWriteTool/TodoWriteTool.ts` |
| `tengu_immediate_model_command` | Runtime boolean | `false` | Makes inference-config/model command immediate for non-ant users | No | `src/utils/immediateCommand.ts` |
| `tengu_iron_gate_closed` | Runtime boolean | `true` | Whether classifier unavailability fails closed in auto-mode permissioning | Yes | `src/utils/permissions/permissions.ts` |
| `tengu_jade_anvil_4` | Runtime boolean | `false` | Reorders rate-limit options UI to put buy/upgrade path first | No | `src/commands/rate-limit-options/rate-limit-options.tsx` |
| `tengu_kairos_brief` | Runtime boolean | `false` | Enables Brief tool/layout at runtime for opted-in sessions | Yes | Explicit kill-switch semantics in `src/tools/BriefTool/BriefTool.ts` |
| `tengu_kairos_brief_config` | Runtime config | `DEFAULT_BRIEF_CONFIG` | Runtime config for brief behavior/eligibility | No | `src/commands/brief.ts` |
| `tengu_kairos_cron` | Runtime boolean | `true` | Master runtime gate for cron scheduling after build-time `AGENT_TRIGGERS` | Yes | `src/tools/ScheduleCronTool/prompt.ts`, `src/utils/cronScheduler.ts` |
| `tengu_kairos_cron_config` | Runtime config | `DEFAULT_CRON_JITTER_CONFIG` | Cron scheduler jitter/expiry tuning knobs | No | `src/utils/cronJitterConfig.ts`, `src/utils/cronTasks.ts` |
| `tengu_kairos_cron_durable` | Runtime boolean | `true` | Enables durable cron persistence/behavior | No | `src/tools/ScheduleCronTool/prompt.ts` |
| `tengu_keybinding_customization_release` | Runtime boolean | `false` | Enables keybinding customization feature | No | `src/keybindings/loadUserBindings.ts` |
| `tengu_lapis_finch` | Runtime boolean | `false` | Enables plugin hint recommendation recording | No | `src/utils/plugins/hintRecommendation.ts` |
| `tengu_lodestone_enabled` | Runtime boolean | `false` | Enables deep-link protocol registration | No | `src/utils/deepLink/registerProtocol.ts` |
| `tengu_log_datadog_events` | Runtime gate | `false` | Enables Datadog analytics sink | No | Constant-backed in `src/services/analytics/sink.ts`; falls back to cached gate |
| `tengu_marble_fox` | Runtime boolean | `false` | Enables attachment-time behavior guarded in `attachments.ts` | No | `src/utils/attachments.ts`; gate name is opaque in this snapshot |
| `tengu_marble_sandcastle` | Runtime boolean | `false` | Enables part of fast-mode availability/behavior | No | `src/utils/fastMode.ts` |
| `tengu_max_version_config` | Runtime config | `{}` | Auto-update max-version cap and user-facing incident message | Yes | `src/utils/autoUpdater.ts` |
| `tengu_miraculo_the_bard` | Runtime boolean | `false` | Kill switch for startup fast-mode status prefetch | Yes | Explicit comment in `src/main.tsx` |
| `tengu_moth_copse` | Runtime boolean | `false` | Skips injecting `MEMORY.md` index because memory prefetch attachments already surface it | No | `src/utils/claudemd.ts`, `src/utils/attachments.ts`, `src/services/extractMemories/extractMemories.ts` |
| `tengu_onyx_plover` | Runtime config | `null` | Auto-dream enablement and scheduling thresholds (`enabled`, `minHours`, `minSessions`) | No | `src/services/autoDream/config.ts`, `src/services/autoDream/autoDream.ts` |
| `tengu_otk_slot_v1` | Runtime boolean | `false` | Enables OTK slot / one-time key-cap related API/query behavior | No | `src/query.ts`, `src/services/api/claude.ts` |
| `tengu_paper_halyard` | Runtime boolean | `false` | Skips project-level memory injection in attachments/CLAUDE.md context assembly | No | `src/utils/attachments.ts`, `src/utils/claudemd.ts` |
| `tengu_passport_quail` | Runtime boolean | `false` | Master gate for extract-memories mode | No | `src/memdir/paths.ts`, `src/services/extractMemories/extractMemories.ts` |
| `tengu_pebble_leaf_prune` | Runtime boolean | `false` | Prunes non-leaf conversation nodes in session graph rebuilding | No | `src/utils/sessionStorage.ts` |
| `tengu_penguins_off` | Runtime config | `null` | Statsig/GrowthBook-supplied reason string for disabling fast mode | No | `src/utils/fastMode.ts`; `null` means no remote reason |
| `tengu_pewter_ledger` | Runtime config | `null` | Plan-mode V2 remote config/variant hook | No | `src/utils/planModeV2.ts`; opaque name in this snapshot |
| `tengu_pid_based_version_locking` | Runtime boolean | `false` | Enables PID-based native installer version lock coordination | No | `src/utils/nativeInstaller/pidLock.ts` |
| `tengu_plan_mode_interview_phase` | Runtime boolean | `false` | Enables plan-mode interview phase for external users | No | `src/utils/planModeV2.ts` |
| `tengu_plugin_official_mkt_git_fallback` | Runtime boolean | `true` | Allows official marketplace to fall back to Git fetches when the primary path fails | Yes | Explicit kill switch in marketplace startup/refresh paths |
| `tengu_plum_vx3` | Runtime boolean | `false` | Makes web search tool use Haiku for its internal path | No | `src/tools/WebSearchTool/WebSearchTool.ts` |
| `tengu_post_compact_survey` | Runtime gate | `false` | Enables post-compaction feedback survey | No | Constant-backed in `src/components/FeedbackSurvey/usePostCompactSurvey.tsx` |
| `tengu_prompt_cache_1h_config` | Runtime config | `{}` | 1-hour prompt-cache allowlist config | No | `src/services/api/claude.ts` |
| `tengu_quartz_lantern` | Runtime boolean | `false` | Fetches single-file git diffs for remote file edit/write permission UI | No | `src/tools/FileEditTool/FileEditTool.ts`, `src/tools/FileWriteTool/FileWriteTool.ts` |
| `tengu_quiet_fern` | Runtime boolean | `false` | Browser-support gate forwarded to VS Code SDK | No | `src/services/mcp/vscodeSdkMcp.ts` |
| `tengu_read_dedup_killswitch` | Runtime boolean | `false` | Disables file-read dedup when flipped on | Yes | `src/tools/FileReadTool/FileReadTool.ts` |
| `tengu_remote_backend` | Runtime boolean | `false` | Enables `--remote` TUI/backend mode without a required initial prompt | No | `src/main.tsx` |
| `tengu_sage_compass` | Runtime config | `{}` | Advisor tool config (`enabled`, user-configurable, base/advisor model pairing) | No | `src/utils/advisor.ts` |
| `tengu_sandbox_disabled_commands` | Runtime config | `{ commands: [], substrings: [] }` | Disables sandbox for matching commands/substrings | No | `src/tools/BashTool/shouldUseSandbox.ts` |
| `tengu_satin_quoll` | Runtime config | `{}` | Per-tool persistence-threshold overrides; also overrides MCP output token cap via `mcp_tool` | No | Constant-backed in `src/utils/toolResultStorage.ts`, `src/utils/mcpValidation.ts` |
| `tengu_scratch` | Runtime gate | `false` | Scratch / coordinator-related Statsig migration gate used in permissions and coordinator mode | No | `src/coordinator/coordinatorMode.ts`, `src/utils/permissions/filesystem.ts` |
| `tengu_sedge_lantern` | Runtime boolean | `false` | Enables away-summary experience | No | `src/hooks/useAwaySummary.ts` |
| `tengu_session_memory` | Runtime boolean | `false` | Enables session memory feature | No | `src/services/SessionMemory/sessionMemory.ts`, `src/services/compact/sessionMemoryCompact.ts` |
| `tengu_slate_heron` | Runtime config | `TIME_BASED_MC_CONFIG_DEFAULTS` | Time-based microcompact configuration | No | `src/services/compact/timeBasedMCConfig.ts` |
| `tengu_slate_prism` | Runtime boolean | `false` / `true` | Used for connector-text summarization beta rollout, and also defaults SDK agent progress summaries on when requested | No | `src/utils/betas.ts`, `src/cli/print.ts`; defaults differ by caller |
| `tengu_slate_thimble` | Runtime boolean | `false` | Allows non-interactive extract-memories mode | No | `src/memdir/paths.ts`, `src/cli/print.ts` |
| `tengu_slim_subagent_claudemd` | Runtime boolean | `true` | Omits inherited `CLAUDE.md` hierarchy from read-only subagents to save context | Yes | Explicit default-on kill switch in `src/tools/AgentTool/runAgent.ts` |
| `tengu_sm_compact` | Runtime boolean | `false` | Enables session-memory compaction path | No | `src/services/compact/sessionMemoryCompact.ts` |
| `tengu_strap_foyer` | Runtime boolean | `false` | Enables settings sync behavior that is separately checked from push rollout | No | `src/services/settingsSync/index.ts` |
| `tengu_streaming_tool_execution2` | Runtime gate | `false` | Enables streaming tool execution in query config | No | `src/query/config.ts` |
| `tengu_surreal_dali` | Runtime boolean | `false` | Remote trigger / scheduled remote agent capability gate | No | `src/skills/bundled/scheduleRemoteAgents.ts`, `src/tools/RemoteTriggerTool/RemoteTriggerTool.ts` |
| `tengu_terminal_panel` | Runtime boolean | `false` | Enables runtime terminal panel toggle/shortcut visibility | No | `src/hooks/useGlobalKeybindings.tsx`, `src/components/PromptInput/PromptInputHelpMenu.tsx` |
| `tengu_terminal_sidebar` | Runtime boolean | `false` | Enables terminal-tab status indicator and exposes its config toggle | No | `src/screens/REPL.tsx`, `src/components/Settings/Config.tsx` |
| `tengu_tern_alloy` | Runtime enum | `'off'` | Tip-copy experiment for subagent/fan-out recommendations | No | `src/services/tips/tipRegistry.ts` |
| `tengu_thinkback` | Runtime gate | `false` | Enables thinkback / thinkback-play commands | No | `src/commands/thinkback/index.ts`, `src/commands/thinkback-play/index.ts` |
| `tengu_tide_elm` | Runtime enum | `'off'` | Tip-copy experiment for `/effort high` recommendation | No | `src/services/tips/tipRegistry.ts` |
| `tengu_timber_lark` | Runtime enum | `'off'` | Tip-copy experiment for `/loop` / recurring-schedule messaging | No | `src/services/tips/tipRegistry.ts` |
| `tengu_tool_pear` | Runtime gate | `false` | Enables strict tool schemas / structured outputs beta | No | `src/utils/api.ts`, `src/utils/betas.ts`, `src/Tool.ts` |
| `tengu_tool_search_unsupported_models` | Runtime list config | `null` | Model-pattern denylist for tool-reference support/tool search | No | `src/utils/toolSearch.ts` |
| `tengu_toolref_defer_j8m` | Runtime gate | `false` | Enables deferred tool-reference messaging path | No | `src/utils/messages.ts` |
| `tengu_trace_lantern` | Runtime boolean | `false` | Enables beta session tracing for allowlisted orgs | No | `src/utils/telemetry/betaSessionTracing.ts` |
| `tengu_turtle_carbon` | Runtime boolean | `true` | Enables ultrathink runtime path when build-time `ULTRATHINK` is present | No | `src/utils/thinking.ts` |
| `tengu_ultraplan_model` | Runtime config | `ALL_MODEL_CONFIGS.opus46.firstParty` | Default model selection for `/ultraplan` | No | `src/commands/ultraplan.tsx` |
| `tengu_version_config` | Runtime config | `{ minVersion: '0.0.0' }` | Minimum required app version | Yes | `src/utils/autoUpdater.ts` |
| `tengu_vscode_cc_auth` | Runtime boolean | `false` | Enables in-band Claude Code auth flow in VS Code SDK | No | `src/services/mcp/vscodeSdkMcp.ts` |
| `tengu_vscode_onboarding` | Runtime gate | `false` | VS Code onboarding gate forwarded to the extension | No | `src/services/mcp/vscodeSdkMcp.ts` |
| `tengu_vscode_review_upsell` | Runtime gate | `false` | VS Code review upsell gate forwarded to the extension | No | `src/services/mcp/vscodeSdkMcp.ts` |
| `tengu_willow_mode` | Runtime enum | `'off'` | Idle-return UX mode for large/cold conversations (`dialog`, `hint`, `hint_v2`, `off`) | No | `src/screens/REPL.tsx` |
| `tengu_sessions_elevated_auth_enforcement` | Runtime boolean/gate | `false` | Sends trusted-device token and enrolls device for elevated bridge sessions | Yes | Constant-backed in `src/bridge/trustedDevice.ts` |

## GrowthBook Attribute Schema

User attributes sent to GrowthBook are assembled in `src/services/analytics/growthbook.ts`:

| Attribute | Source |
|---|---|
| `id` | `user.deviceId` |
| `sessionId` | `user.sessionId` |
| `deviceID` | `user.deviceId` |
| `platform` | `user.platform` |
| `apiBaseUrlHost` | Hostname from `ANTHROPIC_BASE_URL`, only when non-default |
| `organizationUUID` | `user.organizationUuid` when present |
| `accountUUID` | `user.accountUuid` when present |
| `userType` | `user.userType` when present |
| `subscriptionType` | `user.subscriptionType` when present |
| `rateLimitTier` | `user.rateLimitTier` when present |
| `firstTokenTime` | `user.firstTokenTime` when present |
| `email` | `user.email`, or OAuth email fallback for ants |
| `appVersion` | `user.appVersion` when present |
| `githubActionsMetadata` | `user.githubActionsMetadata` when present |

Notes:
- `cacheKeyAttributes` for remote eval are only `id` and `organizationUUID`.
- The declared `GrowthBookUserAttributes` type mentions `github`, but the implementation currently sends `githubActionsMetadata`.
- Trust/auth gating matters: if trust is not established yet, the client is created without auth headers and relies on cached values.

## Flag Evaluation Caching

- Override precedence is: `CLAUDE_INTERNAL_FC_OVERRIDES` env var (ants only) -> `/config Gates` overrides in `growthBookOverrides` -> in-memory remote-eval cache -> disk cache -> provided default.
- The in-memory authoritative cache is `remoteEvalFeatureValues` in `src/services/analytics/growthbook.ts`.
- Successful remote-eval payloads are wholesale-synced to `cachedGrowthBookFeatures` in global config (`~/.claude.json`-backed config).
- Periodic refresh is global, not per-flag:
  - ants: every 20 minutes
  - non-ants: every 6 hours
- `getFeatureValue_CACHED_MAY_BE_STALE()` is a pure read:
  - memory cache first
  - disk cache second
  - default on miss/read failure
- `getFeatureValue_CACHED_WITH_REFRESH()` is currently just an alias to `_CACHED_MAY_BE_STALE`; the old per-flag TTL parameter is ignored.
- `checkGate_CACHED_OR_BLOCKING()` is asymmetric:
  - cached `true` returns immediately
  - cached `false` or missing awaits initialization and fetches the fresh value
- `checkSecurityRestrictionGate()` waits for auth-triggered reinitialization, then prefers `cachedStatsigGates`, then `cachedGrowthBookFeatures`, then `false`.
- Exposure logging is deduped per session with `loggedExposures`; pre-init accesses are queued in `pendingExposures`.

## Fallback When GrowthBook Is Unreachable

- If GrowthBook is effectively disabled (`is1PEventLoggingEnabled()` is false), getters do not initialize the client:
  - value getters return the supplied default
  - gate helpers return `false`
- If trust/auth is missing, the client is created without HTTP init and the process relies on cached disk values only.
- `getFeatureValue_CACHED_MAY_BE_STALE()` falls back to:
  - env/config override if present
  - disk cache if readable
  - supplied default otherwise
- Blocking config/value getters (`getDynamicConfig_BLOCKS_ON_INIT`, `getFeatureValueInternal`) return the supplied default if init fails, times out, or no client is available.
- `checkGate_CACHED_OR_BLOCKING()` falls back to `false` when no fresh value can be fetched.
- `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` falls back to:
  - GrowthBook disk cache first
  - then `cachedStatsigGates`
  - then `false`
- Empty/malformed remote-eval payloads do not clear persisted flags; the code explicitly refuses to sync an empty feature map to disk.
- Sink kill/config readers are intentionally fail-open in a few places:
  - `tengu_frond_boric`: malformed/missing config leaves sinks enabled
  - `tengu_event_sampling_config`: malformed/missing event config means unsampled logging
