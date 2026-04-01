# Tool Contract Inventory

Generated: 2026-04-01
Extraction basis:
- tool type contract
- tool registry
- tool exposure rules

## Bottom Line

The tool type contract and the tool registry together define the model-tool contract:

- how tools are typed
- what context they receive
- how permission state is represented
- how results and progress are emitted
- which concrete tools can be exposed

This document is self-contained. It describes both the abstract tool API and the concrete exposure rules, so a reader can understand what the model can call and under what constraints.

## 1. Type-Level Tool Contract

### Input schema

`ToolInputJSONSchema` is the minimal schema shape exported for API-facing tool schemas:

- `type: 'object'`
- arbitrary properties

This is the bridge type between internal tool implementations and Anthropic API tool declarations.

### Permission context

The permission context passed into tool execution includes:

- `mode`
- additional working directories
- always-allow rules
- always-deny rules
- always-ask rules
- bypass-permissions availability
- auto-mode availability
- stripped dangerous rules
- “avoid permission prompts” flag
- “await automated checks before dialog” flag
- `prePlanMode`

This means permission state is richer than a single boolean or mode enum.

### Tool execution context

The tool-use context is the real runtime contract passed to tools.

It includes:

- model/tool/options bundle
- abort controller
- read-file state
- AppState getters/setters
- MCP clients and resources
- prompt request callback
- tool JSX hooks
- notification hooks
- message list
- file-history and attribution state updaters
- per-tool and per-thread bookkeeping
- content replacement state for tool-result budgeting
- rendered system prompt snapshot for cache-sharing forks

This is one of the densest and most informative source artifacts in the repo.

### Tool results

A tool result can return:

- `data`
- optional `newMessages`
- optional `contextModifier`
- optional MCP metadata (`structuredContent`, `_meta`)

This shows tool execution can mutate transcript state, not just return a blob.

### Progress model

The tool progress contract includes:

- `ToolProgress`
- `ToolCallProgress`
- `Progress = ToolProgressData | HookProgress`

So progress reporting is part of the first-class tool API, alongside final results.

### What a tool implementation must declare

At minimum, a tool definition carries:

- primary `name`
- optional aliases
- input schema
- human-readable `description(...)`
- `call(...)`
- `isEnabled()`
- `isReadOnly(...)`
- `isConcurrencySafe(...)`

Optional but important fields and hooks include:

- output schema
- input-equivalence check
- destructive-operation marker
- interrupt behavior
- UI-collapse hints
- search hint for deferred tool discovery

This is closer to a typed RPC contract than a loose callback interface.

## 2. Concrete Tool Registry

### Core imports

The file imports the standard tool implementations directly:

- `AgentTool`
- `SkillTool`
- `BashTool`
- `FileEditTool`
- `FileReadTool`
- `FileWriteTool`
- `GlobTool`
- `NotebookEditTool`
- `WebFetchTool`
- `TaskStopTool`
- `BriefTool`
- `TaskOutputTool`
- `WebSearchTool`
- `TodoWriteTool`
- `GrepTool`
- `AskUserQuestionTool`
- `LSPTool`
- `ListMcpResourcesTool`
- `ReadMcpResourceTool`
- `ToolSearchTool`
- `EnterPlanModeTool`
- `EnterWorktreeTool`
- `ExitWorktreeTool`
- `ConfigTool`
- task CRUD tools

### Feature-gated and lazy tools

The registry also conditionally loads:

- `REPLTool`
- `SuggestBackgroundPRTool`
- `SleepTool`
- cron tools
- `RemoteTriggerTool`
- `MonitorTool`
- `SendUserFileTool`
- `PushNotificationTool`
- `SubscribePRTool`
- `OverflowTestTool`
- `CtxInspectTool`
- `TerminalCaptureTool`
- `WebBrowserTool`
- `SnipTool`
- `ListPeersTool`
- `WorkflowTool`
- `PowerShellTool`
- testing-only tools

## 3. Base Tool Pool

`getAllBaseTools()` is the source of truth for the exhaustive base pool in the current environment.

Always-present entries in that return list:

- `AgentTool`
- `TaskOutputTool`
- `BashTool`
- `ExitPlanModeV2Tool`
- `FileReadTool`
- `FileEditTool`
- `FileWriteTool`
- `NotebookEditTool`
- `WebFetchTool`
- `TodoWriteTool`
- `WebSearchTool`
- `TaskStopTool`
- `AskUserQuestionTool`
- `SkillTool`
- `EnterPlanModeTool`
- `BriefTool`
- `ListMcpResourcesTool`
- `ReadMcpResourceTool`

Conditional entries include search, config, LSP, worktree, team, schedule, browser, remote-trigger, notification, REPL, and ToolSearch surfaces.

### Capability families represented by the base pool

The base pool covers these major execution families:

- shell and file operations:
  Bash, Read, Edit, Write, NotebookEdit, Glob, Grep, PowerShell
- knowledge retrieval:
  WebFetch, WebSearch, MCP resource list/read, ToolSearch
- planning and workflow control:
  EnterPlanMode, ExitPlanMode, Brief, TaskOutput, TaskStop
- user interaction:
  AskUserQuestion, Config
- agents and orchestration:
  Agent, Skill, SendMessage, team tools, worktree tools
- automation and scheduling:
  sleep, cron, workflow, remote trigger, monitor, push notification

## 4. Filtering Rules Before Model Exposure

The runtime does not expose `getAllBaseTools()` directly to the model.

### Blanket deny-rule pruning

`filterToolsByDenyRules()` (`262-269`) removes tools before the model ever sees them if there is a blanket deny match.

Important comment:

- MCP server-prefix deny rules can hide an entire server’s tool family before prompt assembly

### Simple mode

When `CLAUDE_CODE_SIMPLE` is truthy (`271-298`):

- normal simple mode exposes only `BashTool`, `FileReadTool`, `FileEditTool`
- REPL-enabled simple mode exposes `REPLTool` instead
- coordinator mode can add `AgentTool`, `TaskStopTool`, and `SendMessageTool`

### Special-tool stripping

Before final exposure (`300-307`), some special tools are excluded from the ordinary prompt-visible tool list:

- `ListMcpResourcesTool`
- `ReadMcpResourceTool`
- synthetic output tool

### REPL wrapping

When REPL mode is enabled (`312-320`):

- primitive tools are hidden from direct use
- they remain accessible behind the REPL wrapper

### Effective exposure matrix

The same installed binary can expose very different tool surfaces depending on mode:

- normal mode:
  broad tool set from `getAllBaseTools()`, minus deny rules and special hidden tools
- simple mode:
  minimal execution set, usually `Bash`, `Read`, `Edit`
- REPL-enabled simple mode:
  wrapper-style REPL tool instead of primitive tools
- coordinator mode:
  regains some orchestration tools even when the rest of the surface is narrow
- REPL mode generally:
  primitive tools may be hidden and invoked indirectly through the REPL wrapper

This is why tool availability cannot be understood from the registry alone; exposure rewriting is part of the contract.

## 5. Tool Presets

`TOOL_PRESETS` currently contains only one preset:

- `default`

That is interesting because the code is preset-ready even though only one preset currently exists.

## 6. What This Tells Us About The Product

The tool system is not just “a list of tools”.

The contract has at least five layers:

1. type-level interface
2. concrete registry
3. environment and feature gating
4. permission-context pruning
5. execution-mode rewriting such as simple mode and REPL mode

That layering is exactly why a separate tool-contract document is useful even though the earlier analysis already covered some specific tools.

## 7. Implementation Inventory

Top-level tool implementation families present in the product:

- `AgentTool`
- `AskUserQuestionTool`
- `BashTool`
- `BriefTool`
- `ConfigTool`
- `EnterPlanModeTool`
- `EnterWorktreeTool`
- `ExitPlanModeTool`
- `ExitWorktreeTool`
- `FileEditTool`
- `FileReadTool`
- `FileWriteTool`
- `GlobTool`
- `GrepTool`
- `LSPTool`
- `ListMcpResourcesTool`
- `MCPTool`
- `McpAuthTool`
- `NotebookEditTool`
- `PowerShellTool`
- `REPLTool`
- `ReadMcpResourceTool`
- `RemoteTriggerTool`
- `ScheduleCronTool`
- `SendMessageTool`
- `SkillTool`
- `SleepTool`
- `SyntheticOutputTool`
- `TaskCreateTool`
- `TaskGetTool`
- `TaskListTool`
- `TaskOutputTool`
- `TaskStopTool`
- `TaskUpdateTool`
- `TeamCreateTool`
- `TeamDeleteTool`
- `TodoWriteTool`
- `ToolSearchTool`
- `WebFetchTool`
- `WebSearchTool`

That directory inventory is useful because it reveals the real product capability model, even for tools that are feature-gated in a given build.
