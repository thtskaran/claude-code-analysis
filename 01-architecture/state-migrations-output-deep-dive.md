# State Management, Migrations, and Output Styling Deep Dive

## Executive Summary

Claude Code implements a sophisticated multi-layered state management system designed for reactive updates, configuration persistence, and session continuity. The architecture separates concerns across:

1. **State Management** (`src/state/`): A lightweight custom store pattern with React integration via context and hooks, enabling fine-grained subscriptions and efficient updates
2. **Migrations** (`src/migrations/`): Version-aware transformations that handle model upgrades, settings reorganization, and feature flag transitions
3. **Output Styling** (`src/outputStyles/`): Pluggable prompt-based styling system that customizes model behavior without code changes

---

## Part 1: State Management Architecture

### 1.1 Core Store Implementation

**File**: `src/state/store.ts`

The foundation is a minimal, reactive store pattern:

```typescript
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```

**Key Design Decisions**:

- **Immutable Updates**: `setState` accepts an updater function that receives the previous state and returns a new state. Object.is equality check prevents updates when state doesn't change.
- **Event-Driven Listeners**: Simple Set<Listener> maintains subscribers; each update notifies all listeners.
- **onChange Hook**: Optional callback fires after state mutation completes, enabling side effects like persistence and external sync.
- **No Selectors in Store**: The store itself knows nothing about memoization or derived state—that's the consumer's responsibility.

**Update Flow**:
1. Call `setState(updater)`
2. Store mutates internal state: `state = updater(prev)`
3. If Object.is equality check passes (state changed), proceed; otherwise return early
4. Call `onChange({ newState: state, oldState: prev })`
5. Notify all listeners synchronously

### 1.2 AppState Type Definition

**File**: `src/state/AppStateStore.ts`

AppState is a DeepImmutable TypeScript union of two parts:

**Immutable Properties** (565+ properties):
- Settings (SettingsJson)
- UI state (verbose, statusLineText, expandedView, footerSelection, etc.)
- Models (mainLoopModel, mainLoopModelForSession)
- Permission/auth context (toolPermissionContext, authVersion)
- MCP state (clients[], tools[], commands[], resources)
- Plugin state (enabled[], disabled[], errors[], installationStatus)
- Task state (foregroundedTaskId, viewingAgentTaskId)
- Remote session state (remoteSessionUrl, remoteConnectionStatus)
- Bridge/CCR integration (replBridgeEnabled, replBridgeConnected, replBridgeSessionUrl, etc.)
- Agent/team context (agent name, color, teammates, standalone context)
- Specialized tool state:
  - Tmux (tungstenActiveSession, tungstenPanelVisible)
  - WebBrowser/Bagel (bagelActive, bagelUrl, bagelPanelVisible)
  - Chicago MCP (computerUseMcpState with app allowlist, display targeting)
  - REPL (replContext with VM, registered tools, console capture)
- Plan mode (initialMessage, pendingPlanVerification, ultraplanLaunchPending, ultraplanSessionUrl, isUltraplanMode)
- Denial tracking (denialTracking for classifier modes)
- Other features (thinkingEnabled, promptSuggestion, speculation, skillImprovement)

**Mutable Properties** (excluded from DeepImmutable because they contain functions):
- `tasks: { [taskId: string]: TaskState }` – excludes function types
- `agentNameRegistry: Map<string, AgentId>` – runtime agent name → ID registry
- `mcp.pluginReconnectKey: number` – incremented by /reload-plugins to trigger re-subscriptions
- `notifications: { current: Notification | null, queue: Notification[] }`
- `elicitation: { queue: ElicitationRequestEvent[] }`
- `sessionHooks: SessionHooksState` – Map<string, ...> with hook handlers
- `teamContext?.teammates: { [id]: { ... } }` – teammate state with tmux pane IDs
- `inbox.messages[]` – inbox items with pending/processing status
- `fileHistory: FileHistoryState` – snapshots and tracked files (Set<string>)
- `attribution: AttributionState` – commit attribution tracking

**Initialization**:

`getDefaultAppState()` returns a fresh AppState with:
- Settings from `getInitialSettings()`
- Empty task map and agent registry
- Initial permission mode determined by teammate status (plan mode if `isPlanModeRequired()`)
- Default UI state (expandedView: 'none', footerSelection: null, etc.)
- Empty MCP, plugin, notification, and elicitation queues
- Thinking/prompt suggestion enabled if feature flags permit

### 1.3 React Integration

**File**: `src/state/AppState.tsx` (compiled React)

#### Provider Pattern

`AppStateProvider` wraps the app and prevents nesting:

```typescript
export function AppStateProvider({ children, initialState, onChangeAppState }: Props): React.ReactNode {
  // Prevent nested providers (enforced via HasAppStateContext)
  const [store] = useState(() =>
    createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  )

  // Check on mount if bypass permissions should be disabled
  useEffect(() => {
    if (isBypassPermissionsModeDisabled()) {
      store.setState(prev => ({
        ...prev,
        toolPermissionContext: createDisabledBypassPermissionsContext(...)
      }))
    }
  }, [])

  // Subscribe to settings changes via file watcher
  const onSettingsChange = useEffectEvent((source: SettingSource) =>
    applySettingsChange(source, store.setState)
  )
  useSettingsChange(onSettingsChange)

  return (
    <HasAppStateContext.Provider value={true}>
      <AppStoreContext.Provider value={store}>
        <MailboxProvider>
          <VoiceProvider>{children}</VoiceProvider>
        </MailboxProvider>
      </AppStoreContext.Provider>
    </HasAppStateContext.Provider>
  )
}
```

**Key Points**:
- Store created once via useState with initializer function (stable across re-renders)
- Provider never triggers re-renders because context value (the store object) never changes
- Mount effect checks for race condition: remote settings loaded before provider mounts
- Settings change listener shared with headless/SDK paths via useEffectEvent

#### Hooks for Consumers

1. **useAppState(selector)**: Subscribe to state slice
   ```typescript
   const verbose = useAppState(s => s.verbose)
   const { text } = useAppState(s => s.promptSuggestion)
   ```
   - Uses `useSyncExternalStore` for efficient re-renders
   - Only re-renders when selected value changes (Object.is)
   - Selector must not return new objects on every call

2. **useSetAppState()**: Get updater without subscribing
   ```typescript
   const setAppState = useSetAppState()
   setAppState(prev => ({ ...prev, verbose: true }))
   ```
   - Returns stable reference (never changes)
   - Components using only this hook never re-render

3. **useAppStateStore()**: Direct store access for non-React code
   ```typescript
   const store = useAppStateStore()
   store.setState(...)
   store.getState()
   ```

4. **useAppStateMaybeOutsideOfProvider(selector)**: Safe fallback
   - Returns undefined if called outside provider context
   - Useful for optional components

### 1.4 AppState Change Handler

**File**: `src/state/onChangeAppState.ts`

This is the onChange callback passed to the store. It implements critical side effects triggered by state mutations:

#### Permission Mode Sync (toolPermissionContext.mode)

```typescript
if (prevMode !== newMode) {
  const prevExternal = toExternalPermissionMode(prevMode)
  const newExternal = toExternalPermissionMode(newMode)
  if (prevExternal !== newExternal) {
    // Notify CCR external_metadata
    notifySessionMetadataChanged({
      permission_mode: newExternal,
      is_ultraplan_mode: isUltraplan ? true : null
    })
  }
  notifyPermissionModeChanged(newMode) // SDK status stream
}
```

**Why this matters**: Permission mode is single source of truth for 8+ mutation paths (Shift+Tab cycling, dialog options, /plan command, rewind, REPL bridge, etc.). Previously scattered paths didn't sync with CCR, leaving the web UI out of sync.

#### Model Selection (mainLoopModel)

```typescript
if (newState.mainLoopModel !== oldState.mainLoopModel) {
  if (newState.mainLoopModel === null) {
    // Remove from settings
    updateSettingsForSource('userSettings', { model: undefined })
    setMainLoopModelOverride(null)
  } else if (newState.mainLoopModel !== null) {
    // Save to settings
    updateSettingsForSource('userSettings', { model: newState.mainLoopModel })
    setMainLoopModelOverride(newState.mainLoopModel)
  }
}
```

**Result**: /model changes persist immediately to disk and in-memory model resolution.

#### Expanded View Persistence (expandedView)

Maps to legacy global config fields for backwards compat:
- expandedView: 'tasks' → showExpandedTodos: true
- expandedView: 'teammates' → showSpinnerTree: true

#### Settings-Driven Effects

When `settings` object changes:
- Clear API key, AWS, and GCP credential caches
- Re-apply environment variables from settings.env

### 1.5 Selectors

**File**: `src/state/selectors.ts`

Pure functions for derived state:

```typescript
export function getViewedTeammateTask(
  appState: Pick<AppState, 'viewingAgentTaskId' | 'tasks'>
): InProcessTeammateTaskState | undefined
```

Returns the task being viewed (if any). Used by input routing to direct messages to the correct agent.

```typescript
export function getActiveAgentForInput(appState: AppState): ActiveAgentForInput
```

Discriminated union: { type: 'leader' } | { type: 'viewed', task } | { type: 'named_agent', task }

### 1.6 Teammate View Helpers

**File**: `src/state/teammateViewHelpers.ts`

State mutations for switching between teammate transcript views:

```typescript
export function enterTeammateView(taskId: string, setAppState): void {
  setAppState(prev => {
    // 1. Release previous teammate (if any)
    if (switching) tasks[prevId] = release(prevTask)
    // 2. Set retain=true for new task (prevents eviction, enables disk load)
    if (needsRetain) tasks[taskId] = { ...task, retain: true, evictAfter: undefined }
    // 3. Update viewing state
    return { ...prev, viewingAgentTaskId: taskId, viewSelectionMode: 'viewing-agent', tasks }
  })
}

export function exitTeammateView(setAppState): void {
  // Clears viewingAgentTaskId, releases task back to stub (messages: undefined, retain: false)
  // If terminal, sets evictAfter = Date.now() + 30s for graceful removal
}

export function stopOrDismissAgent(taskId: string, setAppState): void {
  // Running → abort(). Terminal → dismiss (evictAfter: 0 for instant removal)
}
```

**Pattern**: These helpers encapsulate multi-step state transitions to prevent inconsistencies.

---

## Part 2: Migrations System

### 2.1 Overview

Migrations transform settings/config across Anthropic releases. They are **idempotent, source-specific, and completion-flagged**.

**Execution Points**:
- Called early in main.tsx bootstrap before appState initialization
- Triggered when detecting old config/settings formats
- Each migration responsible for its own completion tracking

**Design Principle**: Read/write same source (userSettings, globalConfig, project config) to avoid silently promoting limited scopes to global defaults.

### 2.2 Migration Catalog

#### 1. **migrateAutoUpdatesToSettings.ts**

**Purpose**: Preserve user's explicit auto-update disable preference.

**Flow**:
1. Check if `globalConfig.autoUpdates === false` AND NOT protected-for-native
2. Migrate to `settings.json: env.DISABLE_AUTOUPDATER = '1'`
3. Remove from globalConfig
4. Set process.env.DISABLE_AUTOUPDATER immediately

**Idempotence**: Check already_had_env_var flag in analytics.

#### 2. **migrateBypassPermissionsAcceptedToSettings.ts**

**Purpose**: Move permission prompt suppression flag to better home (settings.json).

**Flow**:
1. Check `globalConfig.bypassPermissionsModeAccepted`
2. If true, set `userSettings.skipDangerousModePermissionPrompt = true`
3. Remove from globalConfig

**Backwards Compat**: Reads from legacy location, writes to new standard location.

#### 3. **migrateEnableAllProjectMcpServersToSettings.ts**

**Purpose**: Migrate MCP server approval lists from project config to local settings.

**Fields Migrated**:
- `projectConfig.enableAllProjectMcpServers` → `localSettings.enableAllProjectMcpServers`
- `projectConfig.enabledMcpjsonServers` → `localSettings.enabledMcpjsonServers` (merges with existing)
- `projectConfig.disabledMcpjsonServers` → `localSettings.disabledMcpjsonServers` (merges with existing)

**Key Detail**: Uses Set to deduplicate when merging server lists, preserving unions of old/new.

#### 4. **migrateFennecToOpus.ts** (ANT-only)

**Purpose**: Migrate ANT users off removed Fennec model aliases to Opus equivalents.

**Mappings**:
- `fennec-latest` → `opus`
- `fennec-latest[1m]` → `opus[1m]`
- `fennec-fast-latest` → `opus[1m]` + fastMode: true
- `opus-4-5-fast` → `opus[1m]` + fastMode: true

**Source**: Only touches `userSettings.model` (not merged settings).

#### 5. **migrateLegacyOpusToCurrent.ts** (FirstParty only)

**Purpose**: Clean up explicit Opus 4.0/4.1 pins that predate 4.5.

**Condition**: Only if:
- Provider is firstParty AND
- isLegacyModelRemapEnabled() (feature flag gate)

**Migrations**:
- `claude-opus-4-20250514`, `claude-opus-4-1-20250805`, `claude-opus-4-0`, `claude-opus-4-1` → `opus`

**Side Effect**: Sets `globalConfig.legacyOpusMigrationTimestamp` so REPL can show one-time notification.

#### 6. **migrateOpusToOpus1m.ts** (FirstParty only)

**Purpose**: Upgrade eligible users to merged Opus 1M (Max/Team Premium only, not Pro).

**Condition**:
- isOpus1mMergeEnabled() (subscription-gated)
- userSettings.model === 'opus'

**Flow**:
1. Resolve both aliases and check if they resolve to same model
2. If so (user is on default), clear model setting
3. Otherwise set to `opus[1m]`

**Rationale**: Idempotent—only updates if exactly 'opus'. Respects CLI --model overrides (not in settings).

#### 7. **migrateReplBridgeEnabledToRemoteControlAtStartup.ts**

**Purpose**: Rename config key to better reflect feature (remote control mode).

**Flow**:
1. Read `globalConfig.replBridgeEnabled` (old, no longer typed)
2. If exists and `remoteControlAtStartup` not set, copy value and remove old key

**Pattern**: Untyped cast to access legacy key, idempotent (checks both old and new).

#### 8. **migrateSonnet1mToSonnet45.ts**

**Purpose**: Pin Sonnet 1M users to explicit 4.5 variant (Sonnet 4.6 1M rollout different).

**Condition**: Tracked by `globalConfig.sonnet1m45MigrationComplete` flag (single-run).

**Migrations**:
- `userSettings.model === 'sonnet[1m]'` → `sonnet-4-5-20250929[1m]`
- Also checks in-memory override (from setMainLoopModelOverride)

**Why**: Sonnet 4.6 and 4.5 1M had different rollout phases; existing users need pinned to 4.5.

#### 9. **migrateSonnet45ToSonnet46.ts** (FirstParty, Pro/Max/Team Premium)

**Purpose**: Upgrade Pro/Max/Team Premium users off explicit Sonnet 4.5 to new 4.6 alias.

**Condition**:
- Provider is firstParty AND
- User is Pro OR Max OR TeamPremium

**Migrations**:
- `claude-sonnet-4-5-20250929` → `sonnet`
- `claude-sonnet-4-5-20250929[1m]` → `sonnet[1m]`
- `sonnet-4-5-20250929` → `sonnet`
- `sonnet-4-5-20250929[1m]` → `sonnet[1m]`

**Side Effect**: Sets `globalConfig.sonnet45To46MigrationTimestamp` (only if numStartups > 1, skip for new users).

#### 10. **resetAutoModeOptInForDefaultOffer.ts**

**Purpose**: Re-surface auto mode permission dialog for users who accepted old 2-option dialog but haven't made it default.

**Condition**:
- Feature TRANSCRIPT_CLASSIFIER enabled AND
- globalConfig.hasResetAutoModeOptInForDefaultOffer not set AND
- getAutoModeEnabledState() === 'enabled'

**Flow**:
1. Check if `skipAutoPermissionPrompt` is true AND `permissions.defaultMode !== 'auto'`
2. If so, clear skipAutoPermissionPrompt (resurfaces dialog)
3. Set completion flag

**Guard**: Only runs for 'enabled' mode (not 'opt-in') because opt-in users can't reach the dialog if we clear the flag.

#### 11. **resetProToOpusDefault.ts**

**Purpose**: Notify Pro users upgraded to Opus 4.5 (old Pro default was Claude 3.5).

**Condition**: FirstParty + Pro subscription.

**Flow**:
1. Check if `settings.model === undefined` (user on default)
2. If so, set `opusProMigrationTimestamp` (tells REPL to show notification)
3. Always set completion flag

**Idempotence**: Tracked via `opusProMigrationComplete` global config flag.

### 2.3 Execution Pattern

All migrations follow this pattern:

```typescript
export function migrationName(): void {
  // 1. Early return if conditions not met
  if (some_condition) return

  // 2. Read from source (userSettings, localSettings, globalConfig, projectConfig)
  const value = getSetting(...)

  // 3. Transform
  const newValue = transform(value)

  // 4. Write back to same source (idempotent: reading/writing same source)
  updateSetting(..., newValue)

  // 5. Set completion flag in globalConfig if one-time migration
  saveGlobalConfig(c => ({ ...c, completionFlag: true }))

  // 6. Log event for analytics
  logEvent('tengu_migration_name', { details })
}
```

**Error Handling**: Wrapped in try/catch, errors logged but not thrown (don't break startup).

### 2.4 Execution Order

Migrations are called sequentially in `src/bootstrap/state.ts` (assumed from patterns). Order matters:

1. Auto-update migration (clears from global config)
2. Bypass permissions (moves to settings)
3. MCP server approvals (moves to settings)
4. Fennec→Opus (ANT)
5. Legacy Opus remap (1P, legacy remap gate)
6. Opus→Opus1m (1P, subscription gate)
7. REPL bridge rename
8. Sonnet 1M pin (single-run)
9. Sonnet 4.5→4.6 (1P, Pro/Max/Premium)
10. Auto mode opt-in reset
11. Pro→Opus notification

**Rationale**: Config cleanup (1-3) before model migrations (4-6, 8-9) ensures settings are in expected format.

---

## Part 3: Output Styling System

### 3.1 Overview

Output styles customize Claude's behavior via prompt injection without code changes. They are:
- **User-defined**: markdown files in `.claude/output-styles/*.md`
- **Project-scoped**: project `.claude/output-styles/` override user `~/.claude/output-styles/`
- **Plugin-supplied**: plugins contribute styles (with optional force-for-plugin flag)
- **Selectable**: user picks style via /model-style or settings

### 3.2 Type System

**File**: `src/constants/outputStyles.ts`

```typescript
export type OutputStyleConfig = {
  name: string
  description: string
  prompt: string
  source: SettingSource | 'built-in' | 'plugin'
  keepCodingInstructions?: boolean
  forceForPlugin?: boolean
}

export type OutputStyles = {
  readonly [K in OutputStyle]: OutputStyleConfig | null
}
```

**OutputStyle** is a branded string (enum of style names).

**Sources**:
- `'built-in'`: Explanatory, Learning (hard-coded)
- `'userSettings'`, `'localSettings'`, `'policySettings'`: Custom markdown files
- `'plugin'`: Contributed by plugin with force-for-plugin flag

### 3.3 Built-In Styles

#### Default
- No override; model behavior unchanged
- Null config

#### Explanatory
```
Description: "Claude explains its implementation choices and codebase patterns"
keepCodingInstructions: true
prompt: [injected prompt asking for "Insights" blocks with educational explanations]
```

**Feature**: Asks for brief insights using figure.star before/after coding, explaining implementation choices specific to the codebase.

#### Learning
```
Description: "Claude pauses and asks you to write small pieces of code for hands-on practice"
keepCodingInstructions: true
prompt: [injected prompt requesting human collaboration on 2-10 line pieces]
```

**Feature**: Requests human code contributions for design decisions (error handling, data structures, key algorithms). Includes:
- Format for "Learn by Doing" requests
- Guidelines for meaningful contributions
- TODO(human) markers in codebase
- Examples with context + guidance
- Post-contribution insights

### 3.4 Loading Mechanism

**File**: `src/outputStyles/loadOutputStylesDir.ts`

```typescript
export const getOutputStyleDirStyles = memoize(
  async (cwd: string): Promise<OutputStyleConfig[]> => {
    // Load markdown from .claude/output-styles and ~/.claude/output-styles
    const markdownFiles = await loadMarkdownFilesForSubdir('output-styles', cwd)

    return markdownFiles.map(({ filePath, frontmatter, content, source }) => {
      const fileName = basename(filePath)
      const styleName = fileName.replace(/\.md$/, '')

      // Extract from frontmatter or content
      const name = frontmatter['name'] || styleName
      const description = frontmatter['description'] || extractDescription(content)

      // Parse keep-coding-instructions flag
      const keepCodingInstructions = frontmatter['keep-coding-instructions'] === true || === 'true'

      // Warn if force-for-plugin on non-plugin style
      if (frontmatter['force-for-plugin'] !== undefined) {
        logForDebugging(`Output style has force-for-plugin but isn't plugin style`)
      }

      return {
        name,
        description,
        prompt: content.trim(),
        source,
        keepCodingInstructions,
      }
    }).filter(s => s !== null)
  }
)
```

**Markdown Structure**:
```markdown
---
name: "My Style"
description: "What this style does"
keep-coding-instructions: true
---

# Detailed prompt for the model
Your behavior changes here...
```

**Memoization**: Same cwd returns cached result (cleared on /reload-plugins).

### 3.5 Style Resolution

**File**: `src/constants/outputStyles.ts` (getAllOutputStyles function)

```typescript
export async function getAllOutputStyles(cwd: string): Promise<{ [styleName: string]: OutputStyleConfig | null }> {
  const customStyles = await getOutputStyleDirStyles(cwd)
  const pluginStyles = await loadPluginOutputStyles()

  // 1. Start with built-in (Default, Explanatory, Learning)
  const allStyles = { ...OUTPUT_STYLE_CONFIG }

  // 2. Filter managed styles (policySettings source)
  const managedStyles = customStyles.filter(s => s.source === 'policySettings')

  // 3. Merge custom styles (project > user, later files override earlier)
  // 4. Merge plugin styles with force-for-plugin handling
  // 5. Return map where null = style not available
}

export async function getOutputStyleConfig(): Promise<OutputStyleConfig | null> {
  const cwd = getCwd()
  const currentStyle = getSettings_DEPRECATED()?.outputStyle || DEFAULT_OUTPUT_STYLE_NAME
  const allStyles = await getAllOutputStyles(cwd)
  return allStyles[currentStyle] ?? null
}
```

**Precedence** (highest to lowest):
1. policySettings (enforced styles)
2. Project `.claude/output-styles/` (project-specific)
3. User `~/.claude/output-styles/` (personal)
4. Built-in (Default, Explanatory, Learning)
5. Plugin styles (with force-for-plugin flag as override)

### 3.6 Plugin Output Styles

**File**: `src/utils/plugins/loadPluginOutputStyles.ts` (implied)

Plugins contribute output styles via:
```javascript
{
  "outputStyles": [
    {
      "name": "PluginStyle",
      "description": "...",
      "prompt": "...",
      "forceForPlugin": true  // Auto-apply when plugin enabled
    }
  ]
}
```

**forceForPlugin Behavior**:
- Automatically selects the style when plugin is enabled
- If multiple plugins have forced styles, only one chosen (logged via debug)
- Non-plugin styles should not have this flag set (warning logged)

### 3.7 keepCodingInstructions Flag

**Purpose**: Preserve model's coding instructions when style prompt is injected.

**False** (default): Style prompt completely replaces default system behavior.

**True**: Style prompt is ADDED to coding instructions; model retains knowledge of /edit, /bash, /shell, etc.

**Usage**: Explanatory and Learning styles set this true (users still need to code).

### 3.8 Caching and Invalidation

```typescript
export function clearOutputStyleCaches(): void {
  getOutputStyleDirStyles.cache?.clear?.()
  loadMarkdownFilesForSubdir.cache?.clear?.()
  clearPluginOutputStyleCache()
}
```

Called when:
- `/reload-plugins` command executed
- Plugin state changes
- File watcher detects `.claude/output-styles/*.md` changes

---

## Part 4: State Initialization and Bootstrap Integration

### 4.1 Bootstrap Sequence

(Inferred from code structure):

1. **Parse CLI arguments** and environment (--model, --agent, --verbose, etc.)
2. **Load globalConfig** from `~/.claude.json`
3. **Run migrations** (all 11 migrations in sequence)
4. **Load settings** (userSettings, localSettings, policySettings merged)
5. **Initialize AppState** via `getDefaultAppState()`
6. **Create AppStateProvider** with initialState and onChangeAppState callback
7. **Mount React tree** (REPL, prompt input, UI components)
8. **Initialize MCP connections** (useMCPConnections hook)
9. **Load output styles** (getAllOutputStyles called for current setting)

### 4.2 State Initialization Relative to Bootstrap

**AppState Initialization**:

```typescript
export function getDefaultAppState(): AppState {
  const initialMode = isTeammate() && isPlanModeRequired() ? 'plan' : 'default'

  return {
    settings: getInitialSettings(),  // Merged from all sources
    mainLoopModel: null,  // Model override, null = use settings
    verbose: false,
    expandedView: 'none',
    toolPermissionContext: {
      ...getEmptyToolPermissionContext(),
      mode: initialMode
    },
    // ... 500+ other properties initialized to defaults
  }
}
```

**Relationship**:
- Migrations run BEFORE state initialization
- State defaults pulled from already-migrated settings
- Model settings populated from globalConfig (after migrations complete)
- Permission mode determined by teammate/plan-mode context at init time

### 4.3 Settings Hierarchy

After migrations, settings are merged from (highest to lowest precedence):

1. **policySettings** (enforced by org, can't be overridden)
2. **localSettings** (project-scoped, ~/.claude/local-settings.json)
3. **userSettings** (global user prefs, ~/.claude/settings.json)
4. **Hardcoded defaults**

**Mutations**:
- `onChangeAppState` writes back via `updateSettingsForSource` (respects source)
- UI changes typically write to userSettings
- Project config changes go to localSettings

---

## Part 5: Error Handling and Recovery

### 5.1 Migration Error Handling

All migrations wrapped in try/catch:

```typescript
try {
  // migration logic
} catch (error) {
  logError(new Error(`Failed to migrate X: ${error}`))
  logEvent('tengu_migrate_x_error', {})
  // Don't throw—don't break startup
}
```

**Strategy**: Fail gracefully. If migration fails, user continues with old format (runtime remapping may apply). Errors logged for diagnostics.

### 5.2 Settings Corruption Recovery

**In AppStateProvider mount effect**:

```typescript
useEffect(() => {
  const { toolPermissionContext } = store.getState()
  if (
    toolPermissionContext.isBypassPermissionsModeAvailable &&
    isBypassPermissionsModeDisabled()  // From remote settings
  ) {
    store.setState(prev => ({
      ...prev,
      toolPermissionContext: createDisabledBypassPermissionsContext(...)
    }))
  }
}, [])
```

Handles race condition: remote settings load before provider mounts → state initialized with stale mode → mount effect corrects it.

### 5.3 Output Style Loading Errors

```typescript
return markdownFiles
  .map(({ filePath, ... }) => {
    try {
      // Parse and validate style
      return { name, description, prompt, source, keepCodingInstructions }
    } catch (error) {
      logError(error)
      return null  // Skip malformed styles
    }
  })
  .filter(style => style !== null)  // Remove failures
```

Malformed markdown files logged but don't crash style loading.

### 5.4 State Consistency Checks

The DeepImmutable type system prevents accidental mutations at compile time. Runtime checks in some consumers:

```typescript
// In useAppState selector
if (state === selected) {
  throw new Error(`Selector returned original state—must select a property`)
}
```

Prevents common mistake of selecting the entire state object.

---

## Part 6: Key Architectural Patterns

### 6.1 Source-Aware Persistence

All migrations and mutations specify which source they read/write:

```typescript
getSettingsForSource('userSettings')
updateSettingsForSource('userSettings', { model: 'opus' })
getSettingsForSource('localSettings')
saveGlobalConfig(c => ({ ...c, flag: true }))
```

**Benefit**: Idempotent. Reading and writing the same source prevents silent promotion of limited scopes to global defaults.

### 6.2 Immutability via TypeScript

```typescript
export type AppState = DeepImmutable<{
  // immutable properties...
}> & {
  // mutable properties excluded from DeepImmutable
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
}
```

Compiler enforces immutability on 90% of state; exemptions documented.

### 6.3 Listener-Based Reactivity

No hooks, selectors, or computed values in the store itself. Consumers subscribe to changes:

```typescript
const unsubscribe = store.subscribe(() => {
  console.log('Something changed')
})
```

React integration via `useSyncExternalStore` provides fine-grained re-renders.

### 6.4 Completion Flags for One-Time Migrations

```typescript
if (globalConfig.hasResetAutoModeOptInForDefaultOffer) return
// ... do migration ...
saveGlobalConfig(c => ({ ...c, hasResetAutoModeOptInForDefaultOffer: true }))
```

Flags stored in globalConfig (persistent across sessions, survive settings resets).

### 6.5 Async Style Loading with Memoization

```typescript
export const getOutputStyleDirStyles = memoize(
  async (cwd: string): Promise<OutputStyleConfig[]> => { ... }
)
```

Memoization prevents redundant file I/O. Cache cleared on /reload-plugins.

---

## Part 7: Data Flow Examples

### Example 1: User Sets Model via /model Command

1. `/model opus[1m]` typed in REPL
2. Command handler calls `setAppState(prev => ({ ...prev, mainLoopModel: 'opus[1m]' }))`
3. Store mutates state
4. `onChangeAppState` fires (sees mainLoopModel changed from null → 'opus[1m]')
5. Updates `userSettings.model = 'opus[1m]'` and calls `setMainLoopModelOverride`
6. Model selection UI components subscribed via `useAppState(s => s.mainLoopModel)` re-render

### Example 2: Remote Settings Load Before Provider Mounts

1. Background process loads remote settings (async)
2. Remote settings contain disabled bypass mode
3. Provider component hasn't mounted yet
4. Global state somehow initialized from remote settings (before provider aware)
5. Provider mounts → useEffect checks `isBypassPermissionsModeDisabled()` (from remote)
6. State was initialized with stale mode → useEffect corrects it
7. Components using `toolPermissionContext.mode` see corrected value

### Example 3: Migration Path - Sonnet 1M Pin

1. App starts → migrations run
2. Check `globalConfig.sonnet1m45MigrationComplete` (not set)
3. Read `userSettings.model` → 'sonnet[1m]'
4. Update to 'sonnet-4-5-20250929[1m]' (explicit pin)
5. Check in-memory override (may have been set by /model in previous session)
6. If override also 'sonnet[1m]', update to explicit version
7. Set completion flag → next app start skips migration
8. Analytics event logged with migration details

### Example 4: Enter Teammate View with Message Loading

1. User clicks teammate in sidebar
2. UI calls `enterTeammateView(taskId, setAppState)`
3. State mutation:
   - Release previous teammate (evictAfter set if terminal)
   - Set retain: true on new teammate (blocks eviction)
   - Clear evictAfter (cancel pending dismissal)
   - Set viewingAgentTaskId and viewSelectionMode
4. Downstream effects:
   - Task stream listener detects retain changed → loads messages from disk
   - UI components subscribed to viewingAgentTaskId re-render with new transcript
5. User Shift+Escape to exit
6. `exitTeammateView` called → resets viewing state, releases task
7. Task drops retain → eviction allowed again

---

## Part 8: Performance Considerations

### 8.1 Selector Memoization

```typescript
// ❌ Bad—creates new object every render, always changes
const expanded = useAppState(s => ({ view: s.expandedView }))

// ✅ Good—selects existing property, only updates when it changes
const expandedView = useAppState(s => s.expandedView)

// ✅ Good—selects existing sub-object reference
const { text } = useAppState(s => s.promptSuggestion)
```

Store only notifies listeners when internal state changes; selector is consumer's responsibility.

### 8.2 Store Creation

Store created once in useState initializer (not recreated on re-renders). Context value (the store object) never changes → provider never triggers cascading re-renders.

### 8.3 Output Style Caching

Memoized async loader prevents repeated file I/O:

```typescript
const styles = await getOutputStyleDirStyles(cwd)  // Cached
// ... later ...
const styles2 = await getOutputStyleDirStyles(cwd)  // Returns from cache, no I/O
```

Cache cleared only on /reload-plugins or plugin state change.

---

## Part 9: Integration Points

### 9.1 Migration Execution

Called in bootstrap before AppStateProvider mounts. Results inform initial state.

### 9.2 onChangeAppState Callback

Wired into AppStateProvider props at mount time. Fires on every state mutation after store updates.

### 9.3 Output Style Selection

User selects via /model-style command. Selection persisted to settings. Style prompt injected into system context at completion time.

### 9.4 MCP and Plugin State

AppState holds:
- `mcp.clients[]`, `mcp.tools[]`, `mcp.commands[]` – active connections
- `mcp.pluginReconnectKey` – incremented to re-trigger MCP effect subscriptions
- `plugins.enabled[]`, `plugins.disabled[]` – loaded plugin manifests
- `plugins.installationStatus` – progress on background plugin install

Watchers subscribe to these via `useAppState(s => s.mcp)` / `useAppState(s => s.plugins)`.

---

## Conclusion

Claude Code's architecture prioritizes:

1. **Immutability** – TypeScript enforces, store verifies via Object.is
2. **Reactivity** – Fine-grained subscriptions prevent unnecessary re-renders
3. **Persistence** – All changes flow through onChangeAppState for durability
4. **Evolution** – Migrations handle model upgrades, settings reorganization, feature transitions
5. **Customization** – Output styles inject prompts without code changes
6. **Safety** – One-time migration flags, source-aware persistence, graceful error recovery

The system is engineered to scale: 500+ state properties, 11 migrations, 100+ components subscribing to slices, multiple concurrent feature flags, and seamless integration with remote sessions, plugins, and tools.
