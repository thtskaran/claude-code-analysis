# Settings And Policy Schema Encyclopedia

Generated: 2026-04-01
Extraction basis:
- user-facing config tool settings
- full settings schema
- settings merge and policy behavior

## Bottom Line

There are two different settings surfaces:

1. `ConfigTool` exposes a small user-facing writable subset.
2. `SettingsSchema` defines the full settings contract across user, project, local, and managed policy sources.

The user-facing config panel exposes a small writable subset. The full settings contract is much broader and includes enterprise policy, plugins, marketplaces, MCP policy, hooks, remote mode, updater policy, memory, and assistant-mode settings.

This document is intentionally self-contained. It summarizes not just the names of settings, but also the merge rules and trust rules that determine which values actually win at runtime.

## 0. Full Top-Level Settings Key Inventory

For self-contained transfer, here is the full top-level key inventory exposed by the settings contract:

- `$schema`
- `apiKeyHelper`
- `awsCredentialExport`
- `awsAuthRefresh`
- `gcpAuthRefresh`
- `xaaIdp`
- `fileSuggestion`
- `respectGitignore`
- `cleanupPeriodDays`
- `env`
- `attribution`
- `includeCoAuthoredBy`
- `includeGitInstructions`
- `permissions`
- `model`
- `availableModels`
- `modelOverrides`
- `enableAllProjectMcpServers`
- `enabledMcpjsonServers`
- `disabledMcpjsonServers`
- `allowedMcpServers`
- `deniedMcpServers`
- `hooks`
- `worktree`
- `disableAllHooks`
- `defaultShell`
- `allowManagedHooksOnly`
- `allowedHttpHookUrls`
- `httpHookAllowedEnvVars`
- `allowManagedPermissionRulesOnly`
- `allowManagedMcpServersOnly`
- `strictPluginOnlyCustomization`
- `statusLine`
- `enabledPlugins`
- `extraKnownMarketplaces`
- `strictKnownMarketplaces`
- `blockedMarketplaces`
- `forceLoginMethod`
- `forceLoginOrgUUID`
- `otelHeadersHelper`
- `outputStyle`
- `language`
- `skipWebFetchPreflight`
- `sandbox`
- `feedbackSurveyRate`
- `spinnerTipsEnabled`
- `spinnerVerbs`
- `spinnerTipsOverride`
- `syntaxHighlightingDisabled`
- `terminalTitleFromRename`
- `alwaysThinkingEnabled`
- `effortLevel`
- `advisorModel`
- `fastMode`
- `fastModePerSessionOptIn`
- `promptSuggestionEnabled`
- `showClearContextOnPlanAccept`
- `agent`
- `companyAnnouncements`
- `pluginConfigs`
- `remote`
- `autoUpdatesChannel`
- `disableDeepLinkRegistration`
- `minimumVersion`
- `plansDirectory`
- `classifierPermissionsEnabled`
- `minSleepDurationMs`
- `maxSleepDurationMs`
- `voiceEnabled`
- `assistant`
- `assistantName`
- `channelsEnabled`
- `allowedChannelPlugins`
- `defaultView`
- `prefersReducedMotion`
- `autoMemoryEnabled`
- `autoMemoryDirectory`
- `autoDreamEnabled`
- `showThinkingSummaries`
- `skipDangerousModePermissionPrompt`
- `skipAutoPermissionPrompt`
- `useAutoModeDuringPlan`
- `autoMode`
- `disableAutoMode`
- `sshConfigs`
- `claudeMdExcludes`
- `pluginTrustMessage`

## 1. ConfigTool Surface

The interactive config tool exposes this user-facing writable subset:

- `theme`
- `editorMode`
- `verbose`
- `preferredNotifChannel`
- `autoCompactEnabled`
- `autoMemoryEnabled`
- `autoDreamEnabled`
- `fileCheckpointingEnabled`
- `showTurnDuration`
- `terminalProgressBarEnabled`
- `todoFeatureEnabled`
- `model`
- `alwaysThinkingEnabled`
- `permissions.defaultMode`
- `language`
- `teammateMode`
- `classifierPermissionsEnabled` when `USER_TYPE === 'ant'`
- `voiceEnabled` when `VOICE_MODE`
- `remoteControlAtStartup` when `BRIDGE_MODE`
- `taskCompleteNotifEnabled` when `KAIROS` or `KAIROS_PUSH_NOTIFICATION`
- `inputNeededNotifEnabled` when `KAIROS` or `KAIROS_PUSH_NOTIFICATION`
- `agentPushNotifEnabled` when `KAIROS` or `KAIROS_PUSH_NOTIFICATION`

Notable characteristics:

- each key declares `source: 'global' | 'settings'`
- some keys sync directly to AppState for immediate UI effect
- `model` uses dynamic options and async validation
- `permissions.defaultMode` changes enum membership when transcript-classifier features are enabled

## 2. Source Precedence And Merge Rules

The settings system is layered. Effective settings are assembled from multiple sources in this order:

1. plugin-provided base settings
2. `userSettings`
3. `projectSettings`
4. `localSettings`
5. `flagSettings`
6. `policySettings`

Higher layers override lower ones.

Important merge semantics:

- objects deep-merge rather than replace wholesale
- arrays concatenate and deduplicate rather than last-write-wins replace
- unknown fields are often preserved on disk instead of being stripped
- invalid fields are usually ignored at runtime but left in place for repair
- `permissions` is intentionally permissive and preserves unknown keys for compatibility

Policy settings have their own internal precedence. Only one policy source wins:

1. remote managed settings
2. admin-only MDM / HKLM / macOS plist
3. file-based managed settings
4. HKCU fallback

That is different from ordinary settings merging. For policy, it is "first source wins", not "merge all policy layers".

Managed file settings also support a drop-in directory:

- `managed-settings.json` is the base
- files in `managed-settings.d/` merge on top alphabetically
- later drop-ins win

## 3. Full SettingsSchema Principles

Important meta-rules in the settings system:

- backward-compatible schema evolution is expected
- unknown fields are often preserved instead of destroyed
- invalid fields are usually ignored rather than stripped from disk
- `permissions` uses `.passthrough()` to keep unknown keys

This is a deliberate compatibility strategy, not an accident.

## 4. Category Map

### 4A. Auth And Provider Configuration

- `$schema`
- `apiKeyHelper`
- `awsCredentialExport`
- `awsAuthRefresh`
- `gcpAuthRefresh`
- `xaaIdp` when `CLAUDE_CODE_ENABLE_XAA`
- `forceLoginMethod`
- `forceLoginOrgUUID`
- `otelHeadersHelper`

These settings mostly configure how Claude Code acquires credentials, not how the model behaves.

### 4B. Session And UX Behavior

- `fileSuggestion`
- `respectGitignore`
- `cleanupPeriodDays`
- `env`
- `attribution.commit`
- `attribution.pr`
- `includeCoAuthoredBy`
- `includeGitInstructions`
- `outputStyle`
- `language`
- `feedbackSurveyRate`
- `spinnerTipsEnabled`
- `spinnerVerbs`
- `spinnerTipsOverride`
- `syntaxHighlightingDisabled`
- `terminalTitleFromRename`
- `alwaysThinkingEnabled`
- `effortLevel`
- `fastMode`
- `fastModePerSessionOptIn`
- `promptSuggestionEnabled`
- `showClearContextOnPlanAccept`
- `companyAnnouncements`
- `prefersReducedMotion`
- `showThinkingSummaries`

### 4C. Permissions, Hooks, And Safety Policy

- `permissions.allow`
- `permissions.deny`
- `permissions.ask`
- `permissions.defaultMode`
- `permissions.disableBypassPermissionsMode`
- `permissions.disableAutoMode` when classifier is enabled
- `permissions.additionalDirectories`
- `hooks`
- `disableAllHooks`
- `defaultShell`
- `allowManagedHooksOnly`
- `allowedHttpHookUrls`
- `httpHookAllowedEnvVars`
- `allowManagedPermissionRulesOnly`

Important policy detail:

- managed settings can force only managed hooks or only managed permission rules to apply
- this is stronger than ordinary user/project merge semantics
- `projectSettings` is intentionally excluded from some trust-sensitive acknowledgements so a checked-in repo cannot silently opt a user into dangerous behavior

### 4D. MCP, Connectors, And Remote Policy

- `enableAllProjectMcpServers`
- `enabledMcpjsonServers`
- `disabledMcpjsonServers`
- `allowedMcpServers`
- `deniedMcpServers`
- `allowManagedMcpServersOnly`
- `remote.defaultEnvironmentId`
- `channelsEnabled`
- `allowedChannelPlugins`

Notable policy semantics:

- `deniedMcpServers` takes precedence over `allowedMcpServers`
- `allowManagedMcpServersOnly` restricts the allowlist source, not the denylist source
- channel notifications are opt-in and separately allowlisted
- MCP allow and deny arrays merge across settings sources, so users and policy can both contribute entries

### 4E. Worktree And Shell Execution Controls

- `worktree.symlinkDirectories`
- `worktree.sparsePaths`
- `defaultShell`

This is where the repo encodes operational tradeoffs for large repos and multi-worktree setups.

### 4F. Plugins And Marketplaces

- `strictPluginOnlyCustomization`
- `statusLine`
- `enabledPlugins`
- `extraKnownMarketplaces`
- `strictKnownMarketplaces`
- `blockedMarketplaces`
- `pluginConfigs[pluginId].mcpServers`
- `pluginConfigs[pluginId].options`

Two especially important admin controls:

- `strictPluginOnlyCustomization`
- `strictKnownMarketplaces`

Together they can force customization to come only from approved plugin sources.

### 4G. Model Selection And Provider Routing

- `model`
- `availableModels`
- `modelOverrides`
- `effortLevel`
- `advisorModel`
- `agent`

`availableModels` is an enterprise allowlist. `modelOverrides` remaps Anthropic model IDs to provider-specific IDs.

### 4H. Memory, Dream, And Planning

- `autoMemoryEnabled`
- `autoMemoryDirectory`
- `autoDreamEnabled`
- `plansDirectory`
- `minSleepDurationMs` when proactive/Kairos
- `maxSleepDurationMs` when proactive/Kairos

### 4I. Assistant / Kairos / View Mode

- `assistant`
- `assistantName`
- `defaultView`

These are product-mode settings, not just presentation toggles.

### 4J. Update And Versioning Policy

- `autoUpdatesChannel`
- `minimumVersion`
- `disableDeepLinkRegistration` when `LODESTONE`

### 4K. Auto-Mode, SSH, And Final-Tail Settings

- `skipAutoPermissionPrompt`
- `useAutoModeDuringPlan`
- `autoMode.allow`
- `autoMode.soft_deny`
- `autoMode.deny` as ant-only back-compat alias
- `autoMode.environment`
- `disableAutoMode`
- `sshConfigs[]`
- `claudeMdExcludes`
- `pluginTrustMessage`

These were easy to miss because they appear after the long main body of the schema, but they are part of the same public settings contract.

## 5. Trusted Versus Untrusted Sources

The settings system does not treat all sources as equally trustworthy.

Admin-trusted sources:

- managed policy
- plugin-provided customization
- built-in or bundled product surfaces

Potentially user-controlled or repo-controlled sources:

- user settings
- project settings
- local settings
- flag settings

This matters because some policy switches do not merely override values; they disable entire customization surfaces from untrusted sources.

`strictPluginOnlyCustomization` currently supports these surfaces:

- `skills`
- `agents`
- `hooks`
- `mcp`

When enabled by policy:

- user and project customization for those surfaces is skipped
- managed policy and plugin-provided definitions still load
- the system intentionally degrades toward "less locked" if it encounters future unknown surface names rather than rejecting the whole policy file

## 6. Enterprise Control Surfaces

The major managed-policy surfaces are:

- `allowManagedHooksOnly`
- `allowManagedPermissionRulesOnly`
- `allowManagedMcpServersOnly`
- `strictPluginOnlyCustomization`
- `strictKnownMarketplaces`
- `blockedMarketplaces`
- `availableModels`
- `modelOverrides`
- `allowedChannelPlugins`

This means the settings system is also the policy system.

## 7. `strictPluginOnlyCustomization` Is More Careful Than It Looks

The `strictPluginOnlyCustomization` policy uses unusually defensive preprocessing logic:

- unknown future surface names are filtered out rather than breaking the whole managed-settings file
- invalid non-array values are `.catch(undefined)`-dropped instead of invalidating the entire file

The design goal is clear in comments:

- degrade toward "less locked"
- never degrade toward "whole managed settings file rejected"

The currently recognized customization surfaces are:

- `skills`
- `agents`
- `hooks`
- `mcp`

## 8. Marketplace Integrity Constraints In Settings

`extraKnownMarketplaces` has an important consistency check:

- for `source: 'settings'`, the object key must match `source.name`

This exists to keep marketplace reconciliation idempotent and avoid endless cache churn.

## 9. Operationally Important Exceptions

Several settings have extra trust or safety handling beyond simple merge precedence:

- `skipDangerousModePermissionPrompt` is only trusted from user, local, and managed settings, not project settings
- `skipAutoPermissionPrompt` follows the same trusted-source pattern
- `useAutoModeDuringPlan` is effectively opt-out across trusted sources; any trusted `false` disables it
- `autoMode` is merged only from trusted sources because a checked-in project file should not silently widen a user's automatic permission envelope
- `autoMemoryDirectory` is ignored from checked-in project settings for security reasons

These exceptions are important because they show the team explicitly defending against hostile or surprising repository-level configuration.

## 10. What The Earlier Analysis Missed

The earlier reports covered feature flags and some policy behavior, but not the full shape of the settings contract.

The source code reveals that settings are simultaneously:

- user preferences
- runtime feature toggles
- enterprise policy controls
- plugin/marketplace trust configuration
- remote session defaults
- updater/version policy
- memory and assistant-mode configuration

That makes `SettingsSchema` one of the highest-value source-extraction targets in the repo.
