# Command, Tool, And Skill Surface Map

Generated: 2026-04-01
Extraction basis:
- command registry
- tool registry
- bundled skill registry
- built-in plugin registry
- plugin command loader

## Scope

This document maps the user-invocable surface that Claude Code assembles at runtime:

- slash commands
- tools exposed to the model
- bundled skills
- plugin skills
- dynamic skills discovered during file operations

It is a self-contained surface map. A reader should be able to understand what the product exposes, how those surfaces are assembled, and which parts are conditional without having to inspect the implementation.

## 1. Slash Command Assembly

The product ships a large base slash-command list before feature-gated additions and internal-only additions are considered.

Core user-facing built-in slash-command names:

- `add-dir`
- `advisor`
- `agents`
- `branch`
- `btw`
- `chrome`
- `clear`
- `color`
- `compact`
- `config`
- `context`
- `cost`
- `diff`
- `doctor`
- `effort`
- `exit`
- `export`
- `extra-usage`
- `fast`
- `feedback`
- `files`
- `heapdump`
- `help`
- `hooks`
- `ide`
- `init`
- `insights`
- `install-github-app`
- `install-slack-app`
- `keybindings`
- `mcp`
- `memory`
- `mobile`
- `model`
- `output-style`
- `passes`
- `permissions`
- `plan`
- `plugin`
- `pr-comments`
- `privacy-settings`
- `rate-limit-options`
- `release-notes`
- `reload-plugins`
- `remote-env`
- `rename`
- `resume`
- `review`
- `rewind`
- `sandbox`
- `security-review`
- `session`
- `skills`
- `stats`
- `status`
- `statusline`
- `stickers`
- `tag`
- `tasks`
- `terminal-setup`
- `theme`
- `think-back`
- `thinkback-play`
- `ultrareview`
- `upgrade`
- `usage`
- `vim`

Conditionally inserted built-in command names:

- `brief`
- `login`
- `logout`
- `remote-control`
- `voice`
- `web-setup`

Implementation details worth preserving:

- there are two separate `context` command implementations, but both publish the same command name `context`
- `usageReport` is the lazy implementation shim behind the user-facing command `insights`
- `sandboxToggle` is the implementation identifier for the user-facing command `sandbox`
- some feature-gated commands in `COMMANDS()` are only present as variables at this layer, so the loader composition matters as much as the static names

Feature-gated command additions in the same file:

- remote setup command when `CCR_REMOTE_SETUP` is on
- fork command when `FORK_SUBAGENT` is on
- buddy command when `BUDDY` is on
- proactive command when `PROACTIVE` or `KAIROS` is on
- brief command when `KAIROS` or `KAIROS_BRIEF` is on
- assistant command when `KAIROS` is on
- bridge command when `BRIDGE_MODE` is on
- remote-control server command when both `DAEMON` and `BRIDGE_MODE` are on
- voice command when `VOICE_MODE` is on
- peers command when `UDS_INBOX` is on
- workflows command when `WORKFLOW_SCRIPTS` is on
- torch command when `TORCH` is on

Auth/provider-dependent behavior:

- `login` and `logout` are only shown in certain auth/provider modes
- some commands are hidden when the current auth mode cannot support them

### Practical reading of the built-in command set

The built-in slash commands naturally cluster into a few product areas:

- session and transcript management:
  `add-dir`, `branch`, `clear`, `compact`, `context`, `cost`, `diff`, `export`, `files`, `memory`, `rename`, `resume`, `rewind`, `session`, `status`, `tag`, `usage`, `stats`
- configuration and local UX:
  `advisor`, `color`, `config`, `effort`, `fast`, `hooks`, `keybindings`, `model`, `output-style`, `permissions`, `privacy-settings`, `rate-limit-options`, `sandbox`, `statusline`, `terminal-setup`, `theme`, `vim`
- extensions and integrations:
  `agents`, `chrome`, `ide`, `install-github-app`, `install-slack-app`, `mcp`, `mobile`, `plugin`, `reload-plugins`, `remote-env`, `skills`, `tasks`
- planning, review, and higher-level workflows:
  `plan`, `review`, `security-review`, `ultrareview`, `pr-comments`
- account and product lifecycle:
  `extra-usage`, `feedback`, `help`, `login`, `logout`, `release-notes`, `upgrade`
- special or niche surfaces:
  `brief`, `btw`, `heapdump`, `insights`, `passes`, `remote-control`, `stickers`, `think-back`, `thinkback-play`, `voice`, `web-setup`

Even without the implementations, this inventory is enough to reconstruct the product surface:

- the CLI is not just a chat loop; it is a command-driven shell with explicit account, plugin, MCP, review, planning, and remote-control modes
- commands are a first-class UX layer, distinct from tools and skills
- several commands are workflow entrypoints rather than simple toggles

## 2. Internal-Only Commands

There is also a second command pool that is only appended when:

- `process.env.USER_TYPE === 'ant'`
- `!process.env.IS_DEMO`

This pool is clearly meant for internal or advanced operational use rather than the ordinary public surface.

Internal-only command names clearly visible from the registry:

- `commit`
- `commit-push-pr`
- `init-verifiers`
- `version`
- `bridge-kick`
- `ultraplan`

Additional internal-only command identities visible from the registry:

- `backfillSessions`
- `breakCache`
- `bughunter`
- `ctx_viz`
- `goodClaude`
- `issue`
- `forceSnip`
- `mockLimits`
- `subscribePr`
- `resetLimits`
- `resetLimitsNonInteractive`
- `onboarding`
- `share`
- `summary`
- `teleport`
- `antTrace`
- `perfIssue`
- `env`
- `oauthRefresh`
- `debugToolCall`
- `agentsPlatform`
- `autofixPr`

## 3. True Command Load Order

The load order matters because later stages can dedupe, hide, or shadow earlier ones.

Command sources are loaded in this order:

1. bundled skills
2. built-in plugin skills
3. skill-directory commands
4. workflow commands
5. plugin commands
6. plugin skills
7. hardcoded built-in commands

After assembly, the runtime applies:

1. availability filtering
2. `isEnabled()` checks
3. dynamic skill injection

Dynamic skills are inserted after plugin skills but before hardcoded built-ins.

### Why load order matters

This order tells us how the product thinks about extensibility:

1. skills and plugins are loaded first so they can contribute user-facing workflow surfaces
2. hardcoded slash commands are loaded last so the core product surface is always present
3. dynamic skills are injected late because they depend on the files and context actually seen in the current session

Operationally, this means a user-visible command list is not a static constant. It is the result of layered assembly.

## 4. Skill Sources

There are four distinct skill sources:

- bundled skills that ship with the binary
- built-in plugin skills that are toggleable extension surfaces
- user skill directories from `getSkillDirCommands(cwd)`
- plugin-provided skills loaded from markdown under plugin directories

Always-on bundled skills:

- `update-config`
- `keybindings-help`
- `verify`
- `debug`
- `lorem-ipsum`
- `skillify`
- `remember`
- `simplify`
- `batch`
- `stuck`

Feature-gated bundled skills:

- `dream`
- `hunter`
- `loop`
- `schedule`
- `claude-api`
- `claude-in-chrome` when auto-enabled by setup detection
- `run-skill-generator`

## 5. Tool Surface Summary

Slash commands and skills are only part of the runtime surface. The model-facing tool layer is assembled separately.

Always-present base tools:

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

Conditionally added tools include:

- `GlobTool` and `GrepTool` when embedded search helpers are not already available
- `ConfigTool` and `TungstenTool` for ant builds
- `LSPTool` behind `ENABLE_LSP_TOOL`
- `EnterWorktreeTool` and `ExitWorktreeTool` when worktree mode is enabled
- task CRUD tools when Todo V2 is enabled
- `REPLTool`, schedule/cron tools, remote trigger, monitor, push notifications, browser, PowerShell, workflow, snip, peer-list, and test-only tools behind various gates

Important exposure rules:

- blanket deny rules can hide a tool before the model ever sees it
- simple mode collapses the surface to a tiny subset, usually `Bash`, `Read`, and `Edit`
- REPL mode can hide primitive tools behind a wrapper tool rather than exposing them directly
- some tools exist for infrastructure and are stripped from the ordinary prompt-visible set

This means the command surface and the tool surface are related but not identical:

- slash commands are user-invoked control surfaces
- tools are model-invoked execution capabilities
- skills sit between them as reusable prompt workflows

## 6. Plugin Skill Naming And Discovery

Plugin skills and plugin commands follow a consistent naming rule:

- regular markdown command file: `plugin[:namespace]:command`
- `SKILL.md` directory skill: `plugin[:namespace]:skillDirectoryName`

Examples implied by the loader:

- `formatter:fix`
- `github:gh-address-comments`

## 7. Top-Level Tool Implementation Inventory

For source-free transfer, here is the top-level tool implementation family inventory:

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
- `testing`

This directory inventory is useful because it shows the real capability families the runtime was designed around:

- shell and file operations
- planning and task orchestration
- agents and teams
- MCP resource access
- browser, remote, and workflow automation
- configuration and user-interaction tools

Conditionally present tool families:

- search primitives: `GlobTool`, `GrepTool` unless embedded search is available
- ant-only tools: `ConfigTool`, `TungstenTool`, `REPLTool`, `SuggestBackgroundPRTool`
- browser / UI tools: `WebBrowserTool`, `TerminalCaptureTool`
- task tools: `TaskCreateTool`, `TaskGetTool`, `TaskUpdateTool`, `TaskListTool`
- worktree tools: `EnterWorktreeTool`, `ExitWorktreeTool`
- swarm tools: `TeamCreateTool`, `TeamDeleteTool`, `SendMessageTool`
- scheduling tools: `SleepTool`, `CronCreateTool`, `CronDeleteTool`, `CronListTool`, `RemoteTriggerTool`
- remote / notification tools: `MonitorTool`, `SendUserFileTool`, `PushNotificationTool`, `SubscribePRTool`
- specialist tools: `LSPTool`, `SnipTool`, `ToolSearchTool`, `PowerShellTool`

## 8. Tool Exposure Is Further Filtered At Runtime

The raw base list is not the final model-visible list.

Runtime filtering stages:

- deny-rule prefiltering removes blanket-denied tools before the model sees them
- `CLAUDE_CODE_SIMPLE` mode collapses the surface to `Bash`, `Read`, and `Edit`, or to `REPLTool` in REPL mode
- special tools such as MCP resource list/read and the synthetic output tool are removed from the ordinary prompt-time tool list
- REPL mode hides primitive tools that are wrapped behind the REPL VM

## 9. Surface-Level Conclusions

The notable product pattern is that Claude Code does not have a single flat command registry. It has a layered surface composed from:

- hardcoded built-ins
- feature-gated built-ins
- internal-only ant surfaces
- bundled skills
- built-in plugin skills
- user skill directories
- workflow-generated commands
- marketplace plugin commands
- dynamically discovered skills

The corresponding tool surface is equally compositional:

- raw tool registry
- permission pruning
- environment-based simplification
- REPL-mode hiding
- deferred-tool logic via `ToolSearchTool`

That composition model is a major source of extractable behavior not captured by the earlier architecture docs.
