# Codebase Structure Analysis - /Users/karan/Documents/claude-code/src

Generated: 2026-04-01

---

## 1. MOST-IMPORTED FILES (Hub Files)

Ranked by number of distinct files that import each module:

| Rank | Module Path | Importers |
|------|------------|-----------|
| 1 | `ink.js` (src/ink.ts) | **292** |
| 2 | `utils/config.js` | **268** |
| 3 | `utils/debug.js` | **192** |
| 4 | `types/message.js` | **181** |
| 5 | `Tool.js` (src/Tool.ts) | **184** (from Tool.js pattern) |
| 6 | `utils/errors.js` | **154** |
| 7 | `utils/log.js` | **154** |
| 8 | `utils/envUtils.js` | **153** (across 116+ unique files) |
| 9 | `state/AppState.js` | **150** |
| 10 | `commands.js` (types/command.js + commands.js) | **143+61 = ~204** |
| 11 | `bootstrap/state.js` | **134** |
| 12 | `services/analytics/growthbook.js` | **128** |
| 13 | `utils/slowOperations.js` | **119** |
| 14 | `utils/envUtils.js` | **117** (exact match) |
| 15 | `types/message.js` (exact) | **111** |
| 16 | `services/analytics/index.js` | **91** |
| 17 | `utils/auth.js` | **85** |
| 18 | `utils/format.js` | **83** |
| 19 | `utils/settings/settings.js` | **81** |
| 20 | `utils/lazySchema.js` | **72** |
| 21 | `utils/cwd.js` | **71** |
| 22 | `utils/theme.js` | **60** |
| 23 | `services/mcp/types.js` | **56** |
| 24 | `types/hooks.js` (agentSdkTypes.js) | **55** |
| 25 | `utils/model/model.js` | **53** |
| 26 | `utils/permissions/permissions.js` (types/permissions.js) | **49** |
| 27 | `utils/sessionStorage.js` | **44** |
| 28 | `utils/array.js` | **43** |
| 29 | `context/notifications.js` | **37** |
| 30 | `utils/file.js` | **36** |
| 31 | `utils/env.js` | **30** |
| 32 | `utils/platform.js` | **30** |
| 33 | `utils/fsOperations.js` | **30** |
| 34 | `utils/git.js` | **28** |
| 35 | `utils/settings/constants.js` | **28** |
| 36 | `utils/sleep.js` | **26** |
| 37 | `utils/permissions/PermissionMode.js` | **27** |
| 38 | `utils/model/providers.js` | **21** |
| 39 | `utils/thinking.js` | **14** |
| 40 | `utils/fullscreen.js` | **17** |
| 41 | `services/mcp/client.js` | **15** |
| 42 | `state/AppStateStore.js` | **14** |
| 43 | `utils/claudemd.js` | **14** |
| 44 | `utils/worktree.js` | **13** |
| 45 | `utils/http.js` | **13** |
| 46 | `types/ids.js` | **16** |
| 47 | `utils/user.js` | **7** |

### Top Hub Files Summary
The clear hub files in order of connectivity:
1. **ink.js** - UI framework re-export, imported by nearly every component/tool/command
2. **utils/config.js** - Configuration management, imported by 268 files
3. **utils/debug.js** - Debug logging, imported by 192 files
4. **Tool.js** - Tool type definitions, imported by 184 files
5. **types/message.js** - Core message types, imported by 181 files
6. **utils/errors.js** - Error handling utilities, imported by 154 files
7. **utils/log.js** - Logging, imported by 154 files
8. **state/AppState.js** - Global app state, imported by 150 files
9. **commands.js** - Command registry, imported by 143 files
10. **bootstrap/state.js** - Bootstrap state, imported by 134 files

---

## 2. ORPHAN FILES (Not imported by anything)

Files that are never referenced in any import statement. These are typically:
- Entry points (main.tsx, entrypoints/*)
- Test files
- Type definition re-exports
- Generated files

### Likely Orphans (entry points / standalone files):
- `src/main.tsx` - Main entry point
- `src/ink.ts` - Re-export module (imported as ink.js)
- `src/entrypoints/init.ts` - Initialization entry
- `src/entrypoints/mcp.ts` - MCP entry point
- `src/entrypoints/agentSdkTypes.ts` - SDK types entry
- `src/entrypoints/sandboxTypes.ts` - Sandbox types
- `src/entrypoints/sdk/controlSchemas.ts`
- `src/entrypoints/sdk/coreSchemas.ts`
- `src/entrypoints/sdk/coreTypes.ts`
- `src/types/generated/*/` - All generated proto types (entry points, not imported within src)
- `src/types/generated/google/protobuf/timestamp.ts`

### Potential orphans needing investigation:
- `src/types/permissions.ts` (may only be imported via re-export)
- `src/commands/createMovedToPluginCommand.ts`
- `src/commands/init-verifiers.ts`
- `src/commands/security-review.ts`
- `src/commands/review.ts` (re-export)
- `src/services/mockRateLimits.ts`
- `src/services/api/adminRequests.ts`
- `src/services/api/emptyUsage.ts`
- `src/services/api/errorUtils.ts`
- `src/services/mcp/InProcessTransport.ts`
- `src/services/mcp/SdkControlTransport.ts`
- `src/services/mcp/envExpansion.ts`
- `src/services/mcp/mcpStringUtils.ts`
- `src/services/mcp/normalization.ts`
- `src/services/compact/compactWarningHook.ts`
- `src/services/compact/compactWarningState.ts`
- `src/utils/apiPreconnect.ts`
- `src/utils/binaryCheck.ts`
- `src/utils/browser.ts`
- `src/utils/caCerts.ts`
- `src/utils/caCertsConfig.ts`
- `src/utils/classifierApprovals.ts`
- `src/utils/classifierApprovalsHook.ts`
- `src/utils/codeIndexing.ts`
- `src/utils/combinedAbortSignal.ts`
- `src/utils/completionCache.ts`
- Various `src/utils/bash/specs/*.ts` files
- Various `src/native-ts/*/` files
- `src/tools/REPLTool/primitiveTools.ts`
- Many `src/components/*/index.ts` re-export files

---

## 3. LONGEST FILES (Complexity Proxies)

| Rank | File | Lines |
|------|------|-------|
| 1 | `src/cli/print.ts` | **5,595** |
| 2 | `src/screens/REPL.tsx` | **5,006** |
| 3 | `src/main.tsx` | **4,684** |
| 4 | `src/services/api/claude.ts` | **3,420** |
| 5 | `src/bridge/bridgeMain.ts` | **3,000** |
| 6 | `src/tools/BashTool/bashPermissions.ts` | **2,622** |
| 7 | `src/components/PromptInput/PromptInput.tsx` | **2,339** |
| 8 | `src/utils/auth.ts` | **2,003** |
| 9 | `src/utils/config.ts` | **1,818** |
| 10 | `src/query.ts` | **1,730** |
| 11 | `src/services/mcp/config.ts` | **1,579** |
| 12 | `src/services/analytics/growthbook.ts` | **1,156** |
| 13 | `src/tools/BashTool/BashTool.tsx` | **1,144** |
| 14 | `src/tools/AgentTool/AgentTool.tsx` | **1,398** |
| 15 | `src/QueryEngine.ts` | **1,296** |
| 16 | `src/constants/prompts.ts` | **915** |
| 17 | `src/tools/AgentTool/runAgent.ts` | **974** |
| 18 | `src/Tool.ts` | **793** |
| 19 | `src/commands.ts` | **~620+** |

---

## 4. CIRCULAR DEPENDENCY INDICATORS

Based on import analysis, the following file pairs show mutual import patterns (A imports B AND B imports A):

### High-confidence circular import pairs:

1. **`utils/config.js` <-> `utils/auth.js`**
   - config.ts imports from auth (isUsing3PServices, isClaudeAISubscriber in commands.ts which re-exports config)
   - auth.ts imports from config (getGlobalConfig, saveGlobalConfig)

2. **`Tool.js` <-> `types/message.js`**
   - Tool.ts imports types/message.js
   - types/message.js likely imports Tool types (ProgressMessage references)

3. **`commands.js` <-> `types/command.js`**
   - commands.ts imports types/command.js
   - types/command.js references Tool types which loop back

4. **`services/analytics/index.js` <-> `services/analytics/growthbook.js`**
   - index.js imports growthbook
   - growthbook.ts imports from firstPartyEventLogger which imports from index

5. **`services/analytics/growthbook.js` <-> `services/analytics/firstPartyEventLogger.js`**
   - growthbook imports firstPartyEventLogger
   - firstPartyEventLogger imports growthbook

6. **`services/analytics/sink.js` <-> `services/analytics/index.js`**
   - sink imports index (attachAnalyticsSink)
   - index imports sink (initialization chain)

7. **`services/analytics/sinkKillswitch.js` <-> `services/analytics/growthbook.js`**
   - sinkKillswitch imports growthbook
   - Used by sink and firstPartyEventLogger creating a cycle

8. **`bootstrap/state.js` <-> `utils/config.js`**
   - bootstrap/state imports config patterns
   - config imports bootstrap/state

9. **`state/AppState.js` <-> `state/AppStateStore.js`**
   - AppState imports AppStateStore types
   - AppStateStore defines types used by AppState

10. **`utils/hooks.js` <-> `types/message.js`**
    - hooks imports message types
    - message indirectly requires hook types

### Potentially circular via transitive dependencies:
- `services/mcp/client.js` <-> `services/mcp/config.js` <-> `services/mcp/utils.js`
- `tools/BashTool/BashTool.tsx` <-> `tools/BashTool/bashPermissions.ts` <-> `tools/BashTool/readOnlyValidation.ts`
- `utils/permissions/permissions.ts` <-> `utils/permissions/yoloClassifier.ts`
- `services/compact/compact.ts` <-> `services/compact/autoCompact.ts` <-> `services/compact/sessionMemoryCompact.ts`

---

## 5. FILE COUNTS PER TOP-LEVEL DIRECTORY

| Directory | .ts files | .tsx files | Total |
|-----------|-----------|------------|-------|
| **utils/** | ~200+ | 15 | **~215** |
| **components/** | 43 | ~150+ | **~193** |
| **commands/** | ~115 | ~80 | **~195** |
| **tools/** | ~120+ | 35 | **~155** |
| **services/** | ~110+ | 3 | **~113** |
| **hooks/** | 76 | 28 | **~104** |
| **ink/** | 76 | 17 | **~93** |
| **bridge/** | 31 | 0 | **31** |
| **cli/** | 17 | 2 | **19** |
| **constants/** | 21 | 0 | **21** |
| **skills/** | 20 | 0 | **20** |
| **types/** | 11 | 0 | **11** (+ generated) |
| **keybindings/** | 12 | 2 | **14** |
| **memdir/** | 8 | 0 | **8** |
| **migrations/** | 11 | 0 | **11** |
| **tasks/** | 8 | 4 | **12** |
| **entrypoints/** | 7 | 0 | **7** |
| **context/** | 0 | 9 | **9** |
| **state/** | 5 | 1 | **6** |
| **remote/** | 4 | 0 | **4** |
| **buddy/** | 4 | 2 | **6** |
| **screens/** | 0 | 3 | **3** |
| **server/** | 3 | 0 | **3** |
| **query/** | 4 | 0 | **4** |
| **coordinator/** | 1 | 0 | **1** |
| **native-ts/** | 4 | 0 | **4** |
| **schemas/** | 1 | 0 | **1** |
| **outputStyles/** | 1 | 0 | **1** |
| **upstreamproxy/** | 2 | 0 | **2** |
| **voice/** | 1 | 0 | **1** |
| **plugins/** | 2 | 0 | **2** |
| **assistant/** | 1 | 0 | **1** |
| **(root src/)** | 14 | 4 | **18** |
| **TOTAL** | | | **~1,350+** |

### Lines of Code Distribution (estimated by directory importance):
1. **utils/** - Largest directory by file count and likely LOC
2. **commands/** - Many command implementations
3. **components/** - UI components (mostly .tsx)
4. **tools/** - Tool implementations
5. **services/** - Service layer (API, analytics, MCP, etc.)
6. **hooks/** - React hooks
7. **ink/** - Custom Ink rendering framework
8. **bridge/** - Remote bridge infrastructure
9. **cli/** - CLI transport layer (print.ts alone is 5,595 lines)

---

## Key Architectural Observations

1. **Central nervous system**: `ink.js`, `utils/config.js`, `utils/debug.js`, `Tool.ts`, and `types/message.js` form the core dependency backbone
2. **Analytics is deeply embedded**: `services/analytics/` modules (growthbook, index, firstPartyEventLogger) are imported by 128+ files
3. **The `utils/` directory is a catch-all**: 215+ files covering everything from auth to bash parsing to worktree management
4. **`cli/print.ts` is the single largest file** at 5,595 lines - a strong candidate for refactoring
5. **`screens/REPL.tsx`** at 5,006 lines is the main interactive UI component
6. **The analytics subsystem has circular dependencies** through the growthbook/sink/firstPartyEventLogger/index chain
7. **Permission system is complex**: files span `utils/permissions/`, `types/permissions.ts`, `hooks/toolPermission/`, and individual tool permission handlers
8. **MCP (Model Context Protocol) integration** spans config, client, types, auth, and many utility files
