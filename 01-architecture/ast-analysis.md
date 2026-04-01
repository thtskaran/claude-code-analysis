# AST Analysis — Claude Code Source

> 1884 TypeScript files analyzed

## File Size Hotspots (Top 30)

| File | Size | Lines |
|------|------|-------|
| screens/REPL.tsx | 875KB | 5006 |
| main.tsx | 785KB | 4684 |
| components/PromptInput/PromptInput.tsx | 347KB | 2339 |
| commands/plugin/ManagePlugins.tsx | 314KB | 2215 |
| components/Settings/Config.tsx | 265KB | 1822 |
| ink/ink.tsx | 246KB | 1723 |
| tools/AgentTool/AgentTool.tsx | 228KB | 1398 |
| utils/ansiToPng.ts | 210KB | 335 |
| cli/print.ts | 208KB | 5595 |
| hooks/useTypeahead.tsx | 208KB | 1385 |
| components/LogSelector.tsx | 196KB | 1575 |
| utils/messages.ts | 189KB | 5513 |
| utils/sessionStorage.ts | 176KB | 5106 |
| components/mcp/ElicitationDialog.tsx | 175KB | 1169 |
| utils/teleport.tsx | 172KB | 1226 |
| tools/BashTool/BashTool.tsx | 157KB | 1144 |
| utils/hooks.ts | 156KB | 5023 |
| components/Stats.tsx | 149KB | 1228 |
| components/ScrollKeybindingHandler.tsx | 146KB | 1012 |
| components/VirtualMessageList.tsx | 145KB | 1082 |
| components/Messages.tsx | 144KB | 834 |
| utils/processUserInput/processSlashCommand.tsx | 141KB | 922 |
| tools/PowerShellTool/PowerShellTool.tsx | 141KB | 1001 |
| utils/bash/bashParser.ts | 128KB | 4437 |
| commands/plugin/PluginSettings.tsx | 126KB | 1072 |
| utils/attachments.ts | 124KB | 3998 |
| tasks/RemoteAgentTask/RemoteAgentTask.tsx | 123KB | 856 |
| services/api/claude.ts | 123KB | 3420 |
| tools/AgentTool/UI.tsx | 122KB | 872 |
| components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx | 119KB | 768 |

## Public API Surface

- **Exported functions/consts:** 6552
- **Exported classes:** 99
- **Exported types/interfaces:** 1308
- **Exported enums:** 1

### Exported Classes

- `QueryEngine` — QueryEngine.ts
- `BridgeFatalError` — bridge/bridgeApi.ts
- `BridgeHeadlessPermanentError` — bridge/bridgeMain.ts
- `BoundedUUIDSet` — bridge/bridgeMessaging.ts
- `FlushGate` — bridge/flushGate.ts
- `RemoteIO` — cli/remoteIO.ts
- `StructuredIO` — cli/structuredIO.ts
- `HybridTransport` — cli/transports/HybridTransport.ts
- `SSETransport` — cli/transports/SSETransport.ts
- `RetryableError` — cli/transports/SerialBatchEventUploader.ts
- `SerialBatchEventUploader` — cli/transports/SerialBatchEventUploader.ts
- `WebSocketTransport` — cli/transports/WebSocketTransport.ts
- `WorkerStateUploader` — cli/transports/WorkerStateUploader.ts
- `CCRInitError` — cli/transports/ccrClient.ts
- `CCRClient` — cli/transports/ccrClient.ts
- `RedactedGithubToken` — commands/remote-setup/api.ts
- `OptionMap` — components/CustomSelect/option-map.ts
- `SentryErrorBoundary` — components/SentryErrorBoundary.ts
- `AbortError` — entrypoints/agentSdkTypes.ts
- `App` — ink/components/App.tsx
- `ClickEvent` — ink/events/click-event.ts
- `Dispatcher` — ink/events/dispatcher.ts
- `EventEmitter` — ink/events/emitter.ts
- `Event` — ink/events/event.ts
- `FocusEvent` — ink/events/focus-event.ts
- `InputEvent` — ink/events/input-event.ts
- `KeyboardEvent` — ink/events/keyboard-event.ts
- `TerminalEvent` — ink/events/terminal-event.ts
- `TerminalFocusEvent` — ink/events/terminal-focus-event.ts
- `FocusManager` — ink/focus.ts
- `Ink` — ink/ink.tsx
- `YogaLayoutNode` — ink/layout/yoga.ts
- `LogUpdate` — ink/log-update.ts
- `Output` — ink/output.ts
- `CharPool` — ink/screen.ts
- `HyperlinkPool` — ink/screen.ts
- `StylePool` — ink/screen.ts
- `TerminalQuerier` — ink/terminal-querier.ts
- `Parser` — ink/termio/parser.ts
- `PathTraversalError` — memdir/teamMemPaths.ts
- `ColorDiff` — native-ts/color-diff/index.ts
- `ColorFile` — native-ts/color-diff/index.ts
- `FileIndex` — native-ts/file-index/index.ts
- `Node` — native-ts/yoga-layout/index.ts
- `RemoteSessionManager` — remote/RemoteSessionManager.ts
- `SessionsWebSocket` — remote/SessionsWebSocket.ts
- `DirectConnectError` — server/createDirectConnectSession.ts
- `DirectConnectSessionManager` — server/directConnectManager.ts
- `FirstPartyEventLoggingExporter` — services/analytics/firstPartyEventLoggingExporter.ts
- `CannotRetryError` — services/api/withRetry.ts
- `FallbackTriggeredError` — services/api/withRetry.ts
- `DiagnosticTrackingService` — services/diagnosticTracking.ts
- `SdkControlClientTransport` — services/mcp/SdkControlTransport.ts
- `SdkControlServerTransport` — services/mcp/SdkControlTransport.ts
- `AuthenticationCancelledError` — services/mcp/auth.ts
- `ClaudeAuthProvider` — services/mcp/auth.ts
- `McpAuthError` — services/mcp/client.ts
- `McpToolCallError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` — services/mcp/client.ts
- `XaaTokenExchangeError` — services/mcp/xaa.ts
- `AuthCodeListener` — services/oauth/auth-code-listener.ts
- `OAuthService` — services/oauth/index.ts
- `StreamingToolExecutor` — services/tools/StreamingToolExecutor.ts
- `StopTaskError` — tasks/stopTask.ts
- `MaxFileReadTokenExceededError` — tools/FileReadTool/FileReadTool.ts
- `CircularBuffer` — utils/CircularBuffer.ts
- `Cursor` — utils/Cursor.ts
- `MeasuredText` — utils/Cursor.ts
- `QueryGuard` — utils/QueryGuard.ts
- `ActivityManager` — utils/activityManager.ts
- `AwsAuthStatusManager` — utils/awsAuthStatusManager.ts
- `RegexParsedCommand_DEPRECATED` — utils/bash/ParsedCommand.ts
- `ClaudeError` — utils/errors.ts
- `MalformedCommandError` — utils/errors.ts
- `AbortError` — utils/errors.ts
- `ConfigParseError` — utils/errors.ts
- `ShellError` — utils/errors.ts
- `TeleportOperationError` — utils/errors.ts
- `TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` — utils/errors.ts
- `FileStateCache` — utils/fileStateCache.ts
- `FpsTracker` — utils/fpsTracker.ts
- `WindowsToWSLConverter` — utils/idePathConversion.ts
- `ImageResizeError` — utils/imageResizer.ts
- `ImageSizeError` — utils/imageValidation.ts
- `Mailbox` — utils/mailbox.ts
- `WebSocketTransport` — utils/mcpWebSocketTransport.ts
- `FileTooLargeError` — utils/readFileInRange.ts
- `RipgrepTimeoutError` — utils/ripgrep.ts
- `Stream` — utils/stream.ts
- `EndTruncatingAccumulator` — utils/stringUtils.ts
- `ITermBackend` — utils/swarm/backends/ITermBackend.ts
- `InProcessBackend` — utils/swarm/backends/InProcessBackend.ts
- `PaneBackendExecutor` — utils/swarm/backends/PaneBackendExecutor.ts
- `TmuxBackend` — utils/swarm/backends/TmuxBackend.ts
- `TaskOutput` — utils/task/TaskOutput.ts
- `DiskTaskOutput` — utils/task/diskOutput.ts
- `BigQueryMetricsExporter` — utils/telemetry/bigqueryExporter.ts
- `ClaudeCodeDiagLogger` — utils/telemetry/logger.ts
- `UltraplanPollError` — utils/ultraplan/ccrSession.ts
- `ExitPlanModeScanner` — utils/ultraplan/ccrSession.ts

## Zod Schemas (Tool Contracts)

Found 14 Zod schemas:

- `BashCommandHookSchema` (z.object) — schemas/hooks.ts
- `PromptHookSchema` (z.object) — schemas/hooks.ts
- `HttpHookSchema` (z.object) — schemas/hooks.ts
- `AgentHookSchema` (z.object) — schemas/hooks.ts
- `multiAgentInputSchema` (z.object) — tools/AgentTool/AgentTool.tsx
- `asyncOutputSchema` (z.object) — tools/AgentTool/AgentTool.tsx
- `annotationSchema` (z.object) — tools/AskUserQuestionTool/AskUserQuestionTool.tsx
- `imageMediaTypes` (z.enum) — tools/FileReadTool/FileReadTool.ts
- `inlineOutputSchema` (z.object) — tools/SkillTool/SkillTool.ts
- `forkedOutputSchema` (z.object) — tools/SkillTool/SkillTool.ts
- `searchHitSchema` (z.object) — tools/WebSearchTool/WebSearchTool.ts
- `asyncHookResponseSchema` (z.object) — types/hooks.ts
- `schema` (z.object) — utils/settings/settings.ts
- `parseResult` (z.object) — utils/teleport.tsx

## Feature Flags Found in Source

90 unique flags:
- `ABLATION_BASELINE`
- `AGENT_MEMORY_SNAPSHOT`
- `AGENT_TRIGGERS`
- `AGENT_TRIGGERS_REMOTE`
- `ALLOW_TEST_VERSIONS`
- `ANTI_DISTILLATION_CC`
- `AUTO_THEME`
- `AWAY_SUMMARY`
- `BASH_CLASSIFIER`
- `BG_SESSIONS`
- `BREAK_CACHE_COMMAND`
- `BRIDGE_MODE`
- `BUDDY`
- `BUILDING_CLAUDE_APPS`
- `BUILTIN_EXPLORE_PLAN_AGENTS`
- `BYOC_ENVIRONMENT_RUNNER`
- `CACHED_MICROCOMPACT`
- `CCR_AUTO_CONNECT`
- `CCR_MIRROR`
- `CCR_REMOTE_SETUP`
- `CHICAGO_MCP`
- `COMMIT_ATTRIBUTION`
- `COMPACTION_REMINDERS`
- `CONNECTOR_TEXT`
- `CONTEXT_COLLAPSE`
- `COORDINATOR_MODE`
- `COWORKER_TYPE_TELEMETRY`
- `DAEMON`
- `DIRECT_CONNECT`
- `DOWNLOAD_USER_SETTINGS`
- `DUMP_SYSTEM_PROMPT`
- `ENHANCED_TELEMETRY_BETA`
- `EXPERIMENTAL_SKILL_SEARCH`
- `EXTRACT_MEMORIES`
- `FILE_PERSISTENCE`
- `FORK_SUBAGENT`
- `HARD_FAIL`
- `HISTORY_PICKER`
- `HISTORY_SNIP`
- `HOOK_PROMPTS`
- `IS_LIBC_GLIBC`
- `IS_LIBC_MUSL`
- `KAIROS`
- `KAIROS_BRIEF`
- `KAIROS_CHANNELS`
- `KAIROS_DREAM`
- `KAIROS_GITHUB_WEBHOOKS`
- `KAIROS_PUSH_NOTIFICATION`
- `LODESTONE`
- `MCP_RICH_OUTPUT`
- `MCP_SKILLS`
- `MEMORY_SHAPE_TELEMETRY`
- `MESSAGE_ACTIONS`
- `MONITOR_TOOL`
- `NATIVE_CLIENT_ATTESTATION`
- `NATIVE_CLIPBOARD_IMAGE`
- `NEW_INIT`
- `OVERFLOW_TEST_TOOL`
- `PERFETTO_TRACING`
- `POWERSHELL_AUTO_MODE`
- `PROACTIVE`
- `PROMPT_CACHE_BREAK_DETECTION`
- `QUICK_SEARCH`
- `REACTIVE_COMPACT`
- `REVIEW_ARTIFACT`
- `RUN_SKILL_GENERATOR`
- `SELF_HOSTED_RUNNER`
- `SHOT_STATS`
- `SKILL_IMPROVEMENT`
- `SKIP_DETECTION_WHEN_AUTOUPDATES_DISABLED`
- `SLOW_OPERATION_LOGGING`
- `SSH_REMOTE`
- `STREAMLINED_OUTPUT`
- `TEAMMEM`
- `TEMPLATES`
- `TERMINAL_PANEL`
- `TOKEN_BUDGET`
- `TORCH`
- `TRANSCRIPT_CLASSIFIER`
- `TREE_SITTER_BASH`
- `TREE_SITTER_BASH_SHADOW`
- `UDS_INBOX`
- `ULTRAPLAN`
- `ULTRATHINK`
- `UNATTENDED_RETRY`
- `UPLOAD_USER_SETTINGS`
- `VERIFICATION_AGENT`
- `VOICE_MODE`
- `WEB_BROWSER_TOOL`
- `WORKFLOW_SCRIPTS`

## Most-Imported Modules (Architectural Hotspots)

| Module | Import Count |
|--------|-------------|
| ../ink | 240 |
| debug | 179 |
| ../Tool | 165 |
| ink | 151 |
| ../commands | 127 |
| ../bootstrap/state | 126 |
| types | 121 |
| errors | 106 |
| envUtils | 103 |
| bootstrap/state | 98 |
| src/services/analytics/index | 98 |
| ../utils/errors | 95 |
| slowOperations | 91 |
| log | 89 |
| ../utils/log | 87 |
| ../utils/debug | 85 |
| ../types/message | 81 |
| ../utils/slowOperations | 80 |
| ../services/analytics/index | 78 |
| utils/debug | 76 |
| types/message | 75 |
| ../utils/envUtils | 69 |
| fsOperations | 66 |
| ../state/AppState | 64 |
| ../types/command | 61 |
| utils/config | 60 |
| ../utils/lazySchema | 58 |
| ../utils/messages | 57 |
| ../utils/config | 56 |
| state/AppState | 55 |
| services/analytics/index | 54 |
| config | 54 |
| settings/settings | 53 |
| Tool | 52 |
| ../keybindings/useKeybinding | 50 |
| ../utils/format | 50 |
| design-system/Dialog | 50 |
| ../utils/stringUtils | 49 |
| commands | 46 |
| services/analytics/growthbook | 45 |
| utils/errors | 44 |
| ../utils/auth | 44 |
| ../services/analytics/growthbook | 43 |
| utils/log | 42 |
| utils/envUtils | 41 |
| prompt | 41 |
| utils/settings/settings | 39 |
| constants | 39 |
| execFileNoThrow | 39 |
| ../utils/theme | 37 |

## Engineering Pattern Frequencies

| Pattern | Count |
|---------|-------|
| Async Generators | 23 |
| WeakRef usage | 2 |
| AbortController | 77 |
| EventEmitter/emit | 27 |
| Circuit Breaker | 8 |
| Retry/Backoff | 152 |
| Cache patterns | 741 |
| Memoization | 617 |
| Stream handling | 240 |

### Files Using Async Generators
- QueryEngine.ts
- cli/structuredIO.ts
- entrypoints/agentSdkTypes.ts
- history.ts
- hooks/useHistorySearch.ts
- query/stopHooks.ts
- query.ts
- services/api/claude.ts
- services/api/withRetry.ts
- services/tools/StreamingToolExecutor.ts
- services/tools/toolExecution.ts
- services/tools/toolHooks.ts
- services/tools/toolOrchestration.ts
- services/vcr.ts
- tools/AgentTool/agentToolUtils.ts
- tools/AgentTool/runAgent.ts
- tools/BashTool/BashTool.tsx
- tools/PowerShellTool/PowerShellTool.tsx
- utils/attachments.ts
- utils/fsOperations.ts
- utils/generators.ts
- utils/hooks.ts
- utils/queryHelpers.ts

## Prompt-Bearing Strings in Source (60 found)

### cli/print.ts
```
<system-reminder>
You are running in non-interactive mode and cannot return a response to the user until your team is shut down.

You MUST shut down your team before preparing your final response:
1. ...
```

### commands/init.ts
```
Please analyze this codebase and create a CLAUDE.md file, which will be given to future instances of Claude Code to operate in this repository.

What to add:
1. Commands that will be commonly used, su...
```

### commands/init.ts
```
Set up a minimal CLAUDE.md (and optionally skills and hooks) for this repo. CLAUDE.md is loaded into every Claude Code session, so it must be concise — only include what Claude would get wrong without...
```

### commands/insights.ts
```
Analyze this Claude Code session and extract structured facets.

CRITICAL GUIDELINES:

1. **goal_categories**: Count ONLY what the USER explicitly asked for.
   - DO NOT count Claude's autonomous code...
```

### commands/insights.ts
```
Summarize this portion of a Claude Code session transcript. Focus on:
1. What the user asked for
2. What Claude did (tools used, files modified)
3. Any friction or issues
4. The outcome

Keep it conci...
```

### commands/insights.ts
```
Analyze this Claude Code usage data and identify project areas.

RESPOND WITH ONLY A VALID JSON OBJECT:
{
  "areas": [
    {"name": "Area name", "session_count": N, "description": "2-3 sentences about...
```

### commands/insights.ts
```
Analyze this Claude Code usage data and describe the user's interaction style.

RESPOND WITH ONLY A VALID JSON OBJECT:
{
  "narrative": "2-3 paragraphs analyzing HOW the user interacts with Claude Cod...
```

### commands/insights.ts
```
Analyze this Claude Code usage data and identify what's working well for this user. Use second person ("you").

RESPOND WITH ONLY A VALID JSON OBJECT:
{
  "intro": "1 sentence of context",
  "impressi...
```

### commands/insights.ts
```
Analyze this Claude Code usage data and identify friction points for this user. Use second person ("you").

RESPOND WITH ONLY A VALID JSON OBJECT:
{
  "intro": "1 sentence summarizing friction pattern...
```

### commands/insights.ts
```
Analyze this Claude Code usage data and suggest improvements.

## CC FEATURES REFERENCE (pick from these for features_to_try):
1. **MCP Servers**: Connect Claude to external tools, databases, and APIs...
```

### commands/insights.ts
```
Analyze this Claude Code usage data and identify future opportunities.

RESPOND WITH ONLY A VALID JSON OBJECT:
{
  "intro": "1 sentence about evolving AI-assisted development",
  "opportunities": [
  ...
```

### commands/insights.ts
```
Analyze this Claude Code usage data and suggest product improvements for the CC team.

RESPOND WITH ONLY A VALID JSON OBJECT:
{
  "improvements": [
    {"title": "Product/tooling improvement", "detail...
```

### commands/insights.ts
```
Analyze this Claude Code usage data and suggest model behavior improvements.

RESPOND WITH ONLY A VALID JSON OBJECT:
{
  "improvements": [
    {"title": "Model behavior change", "detail": "3-4 sentenc...
```

### commands/insights.ts
```
Analyze this Claude Code usage data and find a memorable moment.

RESPOND WITH ONLY A VALID JSON OBJECT:
{
  "headline": "A memorable QUALITATIVE moment from the transcripts - not a statistic. Somethi...
```

### commands/review.ts
```
~10–20 min · Finds and verifies bugs in your branch. Runs in Claude Code on the web. See ${CCR_TERMS_URL}...
```

### commands/ultraplan.tsx
```
~10–30 min · Claude Code on the web drafts an advanced plan you can edit and approve. See ${CCR_TERMS_URL}...
```

### components/WorktreeExitDialog.tsx
```
Stays at ${worktreeSession.worktreePath}. Reattach with: tmux attach -t ${worktreeSession.tmuxSessionName}...
```

### components/agents/generateAgent.ts
```
You are an elite AI agent architect specializing in crafting high-performance agent configurations. Your expertise lies in translating user requirements into precisely-tuned agent specifications that ...
```

### components/agents/generateAgent.ts
```
Create an agent configuration based on this request: "${userPrompt}".${existingList}
  Return ONLY the JSON object, no other text....
```

### constants/outputStyles.ts
```

## Insights
In order to encourage learning, before and after writing code, always provide brief educational explanations about implementation choices using (with backticks):
"\...
```

### constants/outputStyles.ts
```
You are an interactive CLI tool that helps users with software engineering tasks. In addition to software engineering tasks, you should provide educational insights about the codebase along the way.

...
```

### constants/outputStyles.ts
```
You are an interactive CLI tool that helps users with software engineering tasks. In addition to software engineering tasks, you should help users learn more about the codebase through hands-on practi...
```

### constants/prompts.ts
```
You are an agent for Claude Code, Anthropic's official CLI for Claude. Given the user's message, you should use the tools available to complete the task. Complete the task fully—don't gold-plate, but ...
```

### memdir/findRelevantMemories.ts
```
You are selecting memories that will be useful to Claude Code as it processes a user's query. You will be given the user's query and a list of available memory files with their filenames and descripti...
```

### services/PromptSuggestion/promptSuggestion.ts
```
[SUGGESTION MODE: Suggest what the user might naturally type next into Claude Code.]

FIRST: Look at the user's recent messages and original request.

Your job is to predict what THEY would type - not...
```

### services/compact/prompt.ts
```
Your task is to create a detailed summary of the conversation so far, paying close attention to the user's explicit requests and your previous actions.
This summary should be thorough in capturing tec...
```

### services/compact/prompt.ts
```
Your task is to create a detailed summary of the RECENT portion of the conversation — the messages that follow earlier retained context. The earlier messages are being kept intact and do NOT need to b...
```

### services/compact/prompt.ts
```
Your task is to create a detailed summary of this conversation. This summary will be placed at the start of a continuing session; newer messages that build on this context will follow after your summa...
```

### services/toolUseSummary/toolUseSummaryGenerator.ts
```
Write a short summary label describing what these tool calls accomplished. It appears as a single-line row in a mobile app and truncates around 30 characters, so think git-commit-subject, not sentence...
```

### skills/bundled/debug.ts
```
# Debug Skill

Help the user debug an issue they're encountering in this current Claude Code session.
${justEnabledSection}
## Session Debug Log

The debug log for the current session is at: \...
```

### skills/bundled/remember.ts
```
# Memory Review

## Goal
Review the user's memory landscape and produce a clear report of proposed changes, grouped by action type. Do NOT apply changes — present proposals for user approval.

## Step...
```

### skills/bundled/simplify.ts
```
# Simplify: Code Review and Cleanup

Review all changed files for reuse, quality, and efficiency. Fix any issues found.

## Phase 1: Identify Changes

Run \...
```

### skills/bundled/skillify.ts
```
# Skillify {{userDescriptionBlock}}

You are capturing this session's repeatable process as a reusable skill.

## Your Session Context

Here is the session memory summary:
<session_memory>
{{sessionMe...
```

### skills/bundled/stuck.ts
```
# /stuck — diagnose frozen/slow Claude Code sessions

The user thinks another Claude Code session on this machine is frozen, stuck, or very slow. Investigate and post a report to #claude-code-feedback...
```

### skills/bundled/updateConfig.ts
```
# Update Config Skill

Modify Claude Code configuration by updating settings.json files.

## When Hooks Are Required (Not Memory)

If the user wants something to happen automatically in response to an...
```

### tools/AgentTool/built-in/statuslineSetup.ts
```
You are a status line setup agent for Claude Code. Your job is to create or update the statusLine command in the user's Claude Code settings.

When asked to convert the user's shell PS1 configuration,...
```

### tools/AgentTool/built-in/verificationAgent.ts
```
You are a verification specialist. Your job is not to confirm the implementation works — it's to try to break it.

You have two documented failure patterns. First, verification avoidance: when faced w...
```

### tools/AskUserQuestionTool/prompt.ts
```
Use this tool when you need to ask the user questions during execution. This allows you to:
1. Gather user preferences or requirements
2. Clarify ambiguous instructions
3. Get decisions on implementat...
```

### tools/BriefTool/prompt.ts
```
Send a message the user will read. Text outside this tool is visible in the detail view, but most won't open it — the answer lives here.

\...
```

### tools/ExitPlanModeTool/prompt.ts
```
Use this tool when you are in plan mode and have finished writing your plan to the plan file and are ready for user approval.

## How This Tool Works
- You should have already written your plan to the...
```

### tools/GlobTool/prompt.ts
```
- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
- Use this tool w...
```

### tools/LSPTool/prompt.ts
```
Interact with Language Server Protocol (LSP) servers to get code intelligence features.

Supported operations:
- goToDefinition: Find where a symbol is defined
- findReferences: Find all references to...
```

### tools/ListMcpResourcesTool/prompt.ts
```

Lists available resources from configured MCP servers.
Each resource object includes a 'server' field indicating which server it's from.

Usage examples:
- List all resources from all servers: \...
```

### tools/ListMcpResourcesTool/prompt.ts
```

List available resources from configured MCP servers.
Each returned resource will include all standard MCP resource fields plus a 'server' field 
indicating which server the resource belongs to.

Par...
```

### tools/NotebookEditTool/prompt.ts
```
Completely replaces the contents of a specific cell in a Jupyter notebook (.ipynb file) with new source. Jupyter notebooks are interactive documents that combine code, text, and visualizations, common...
```

### tools/ReadMcpResourceTool/prompt.ts
```

Reads a specific resource from an MCP server.
- server: The name of the MCP server to read from
- uri: The URI of the resource to read

Usage examples:
- Read a resource from a server: \...
```

### tools/ReadMcpResourceTool/prompt.ts
```

Reads a specific resource from an MCP server, identified by server name and resource URI.

Parameters:
- server (required): The name of the MCP server from which to read the resource
- uri (required)...
```

### tools/RemoteTriggerTool/prompt.ts
```
Call the claude.ai remote-trigger API. Use this instead of curl — the OAuth token is added automatically in-process and never exposed.

Actions:
- list: GET /v1/code/triggers
- get: GET /v1/code/trigg...
```

### tools/SleepTool/prompt.ts
```
Wait for a specified duration. The user can interrupt the sleep at any time.

Use this when the user tells you to sleep or rest, when you have nothing to do, or when you're waiting for something.

You...
```

### tools/TaskGetTool/prompt.ts
```
Use this tool to retrieve a task by its ID from the task list.

## When to Use This Tool

- When you need the full description and context before starting work on a task
- To understand task dependenc...
```

