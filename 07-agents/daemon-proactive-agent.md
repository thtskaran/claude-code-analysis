# Daemon Mode And Proactive Agent

## Scope

This document explains daemon mode, proactive ticking, and scheduled/remote trigger behavior from the public surfaces that are visible in the product code.

Some of the deepest internal daemon implementations were not available in the extracted snapshot used for this analysis. That means this document is strongest on externally visible contracts, lifecycle, and orchestration behavior, and weaker on the exact internal worker registry implementation.

Because of that, some answers below are directly verified and some are inferred from public types, comments, and surrounding integration contracts.

Self-contained reading note:

- file/path references are provenance only
- the operational descriptions below are intended to stand on their own

## 1. What is daemon mode?

Best summary:

- "Daemon mode" is an internal long-lived supervisor process.
- The supervisor appears to own durable infrastructure like scheduling and remote-control connectivity.
- Individual daemon workers are spawned as `claude --daemon-worker <kind>`.
- The worker subprocesses are intentionally lightweight and only initialize what they need.

Operational implication:

- The product is designed so the always-on coordination logic can live separately from an interactive terminal session.
- That separation is what makes scheduled triggers, remote wakeups, and background control surfaces plausible without forcing a full UI session to stay attached.

## 2. What daemon worker types exist?

Not fully recoverable from the extracted snapshot.

Directly verified:

- The dispatch point exists: `runDaemonWorker(kind)` in `src/entrypoints/cli.tsx`.
- `connectRemoteControl()` accepts an optional `workerType`.
- Bridge metadata uses worker types like:
  - `claude_code`
  - `claude_code_assistant`

Strong inference:

- There is at least a remote-control/assistant-oriented worker, because the SDK types explicitly document daemon-owned remote control and assistant mode.
- There is likely a cron/scheduled-task worker, because the SDK exposes `watchScheduledTasks()` specifically "for daemon architectures that own the scheduler externally and spawn the agent via query()".

What is missing:

- the concrete daemon worker registry implementation, so the exact `runDaemonWorker` switch cases cannot be enumerated here
- the document therefore captures the recoverable worker categories, not a definitive exhaustive list

## 3. How does the proactive agent work?

Directly verified behavior:

- Proactive mode is enabled by `--proactive` or `CLAUDE_CODE_PROACTIVE`.
- `main.tsx` injects a proactive system prompt telling the model it will receive periodic `<tick>` prompts and should either do useful work or call `Sleep`.
- `Sleep` is only included when proactive/KAIROS is active.
- In headless mode, `runHeadless()` injects a hidden meta `<tick>` message whenever:
  - proactive is active
  - proactive is not paused
  - the queue is empty
- In the REPL, `useProactive(...)` submits or queues the tick as a hidden meta prompt.
- Tick prompts are hidden from the transcript via `isMeta: true`.

Model-facing prompt rules:

- `src/constants/prompts.ts` says `<tick>` means "you're awake, what now?"
- On the first wake-up, greet briefly and ask what the user wants.
- On later wake-ups, look for useful work.
- If nothing useful exists, the model **must** call `Sleep`.

Sleep tool semantics:

- `src/tools/SleepTool/prompt.ts` explicitly says:
  - use it when waiting / idle
  - user input can interrupt it
  - it is preferred over shell `sleep`
  - ticks are periodic check-ins

## 4. What triggers are supported?

### Local triggers

Verified:

- `CronCreate`
- `CronDelete`
- `CronList`

Behavior:

- Durable tasks persist to `.claude/scheduled_tasks.json`
- Session-only tasks live only in in-memory bootstrap state
- One-shot tasks auto-delete after firing
- Recurring tasks auto-expire after 7 days by default unless marked permanent
- A per-project lock file prevents two Claude sessions from double-firing the same disk-backed cron

Key files:

- `src/tools/ScheduleCronTool/CronCreateTool.ts`
- `src/tools/ScheduleCronTool/CronDeleteTool.ts`
- `src/tools/ScheduleCronTool/CronListTool.ts`
- `src/utils/cronTasks.ts`
- `src/utils/cronScheduler.ts`
- `src/utils/cronTasksLock.ts`

### Remote triggers

Verified:

- `RemoteTrigger` supports:
  - `list`
  - `get`
  - `create`
  - `update`
  - `run`

Behavior:

- It calls the claude.ai CCR trigger API under `/v1/code/triggers`
- OAuth is handled in-process
- The bundled `schedule` skill documents these as scheduled remote agents that run in Anthropic cloud environments
- Deletion is explicitly not supported from the tool

Key files:

- `src/tools/RemoteTriggerTool/RemoteTriggerTool.ts`
- `src/tools/RemoteTriggerTool/prompt.ts`
- `src/skills/bundled/scheduleRemoteAgents.ts`

## 5. How are background sessions managed?

Partially verified from the shared registry, but the command handlers themselves are missing.

Verified:

- `claude ps|logs|attach|kill` are wired as fast paths in `src/entrypoints/cli.tsx`
- Session metadata is stored under `~/.claude/sessions/<pid>.json`
- The registry records:
  - `pid`
  - `sessionId`
  - `cwd`
  - `startedAt`
  - `kind` = `interactive | bg | daemon | daemon-worker`
  - `entrypoint`
  - optional `messagingSocketPath`
  - optional `name`
  - optional `logPath`
  - optional `agent`
  - optional `bridgeSessionId`
  - optional activity state (`status`, `waitingFor`, `updatedAt`)

Runtime behavior:

- `registerSession()` writes the PID file and registers cleanup to remove it on exit.
- `updateSessionActivity()` keeps status fresh for `claude ps`.
- `updateSessionName()` and `updateSessionBridgeId()` enrich what peer/session listings can show.
- In `--bg` sessions, `/exit`, Ctrl+C, and Ctrl+D detach the tmux client instead of terminating the process.

What is missing:

- `src/cli/bg.js`, so the exact implementation of `ps`, `logs`, `attach`, and `kill` is not present here.

Key files:

- `src/entrypoints/cli.tsx`
- `src/utils/concurrentSessions.ts`
- `src/commands/exit/exit.tsx`
- `src/screens/REPL.tsx`

## 6. What is the tmux integration?

There are two tmux stories here.

### Worktree tmux (`--worktree --tmux`)

Verified:

- `--tmux` requires `--worktree`
- it is rejected on Windows
- it verifies tmux is installed
- there is an early fast path `execIntoTmuxWorktree(args)` before the full CLI loads

Behavior:

- creates or reuses a worktree
- creates or attaches to a tmux session named from repo + worktree branch
- sets `CLAUDE_CODE_TMUX_SESSION`, `CLAUDE_CODE_TMUX_PREFIX`, and `CLAUDE_CODE_TMUX_PREFIX_CONFLICTS`
- when launched from iTerm2 and not already inside tmux, it prefers `tmux -CC` unless `--tmux=classic` is used
- when already inside tmux, it creates a detached sibling session and `switch-client`s into it rather than nesting

Key files:

- `src/main.tsx`
- `src/utils/worktree.ts`
- `src/setup.ts`

### tmux isolation for Claude-managed commands

Verified:

- `src/utils/tmuxSocket.ts` creates an isolated Claude-specific tmux socket so Claude-issued tmux commands do not mutate the user's own tmux server.

## 7. How does the agent work without a terminal?

Verified path:

- `-p/--print` is the headless/non-interactive mode
- `runHeadless(...)` is the core runner
- In headless mode:
  - there is no Ink/React terminal tree
  - plugins are installed via headless-specific code
  - cron tasks are run by `createCronScheduler(...)`
  - proactive ticks are injected by the headless loop
  - UDS inbox messages can wake the loop
  - permission prompts are avoided; headless agents either use PermissionRequest hooks or auto-deny

Important details:

- `main.tsx` explicitly says trust is implicit in non-interactive `--print` mode
- `SessionStart` hooks can synthesize the initial user turn when stdin is empty
- headless `kairosEnabled` is preserved so scheduled tasks and async agent/tool calls do not accidentally become synchronous
- headless/team mode adds a reminder that the team must shut down before the final response

Key files:

- `src/main.tsx`
- `src/cli/print.ts`
- `src/utils/permissions/permissions.ts`
- `src/utils/forkedAgent.ts`

## 8. What is the KAIROS feature flag about?

KAIROS is the assistant-mode bundle.

Verified signals:

- `settings.assistant` is described as: "Start Claude in assistant mode (custom system prompt, brief view, scheduled check-in skills)"
- `--assistant` is hidden and documented as "Agent SDK daemon use"
- KAIROS gates:
  - assistant mode
  - brief mode integration
  - proactive/Sleep integration
  - extra assistant commands
  - channel/push features
  - assistant-specific system prompt addenda
- `fullRemoteControl = remoteControl || getRemoteControlAtStartup() || kairosEnabled`

Important nuance:

- Cron scheduling is **not** purely a KAIROS feature. The local cron system is gated by `AGENT_TRIGGERS` and is designed to ship independently.
- KAIROS seems to be the higher-level "assistant persona / perpetual assistant / remote-control assistant" package layered on top of those lower-level mechanisms.

Key files:

- `src/main.tsx`
- `src/tools.ts`
- `src/commands.ts`
- `src/utils/settings/types.ts`

## Full Daemon Lifecycle

## Start

Verified:

1. CLI entrypoint sees `claude daemon ...`
2. It enables configs and logging sinks
3. It dispatches to `daemonMain(...)`
4. The supervisor can spawn worker subprocesses as `claude --daemon-worker <kind>`

Missing:

- The internal `daemonMain` and `workerRegistry` implementations are not in this checkout.

## Receive Work

Direct evidence points to three work sources:

1. Remote-control inbound prompts
   - `connectRemoteControl()` returns `inboundPrompts()`, `controlRequests()`, and `permissionResponses()`
   - comments say user messages typed on claude.ai are read from `inboundPrompts()` and fed into `query()`

2. Scheduled tasks
   - `watchScheduledTasks()` is documented as the daemon-oriented watcher for `.claude/scheduled_tasks.json`
   - `createCronScheduler(...)` is the concrete scheduler implementation used by headless/REPL paths

3. Local message injection
   - headless mode can be kicked by UDS inbox messages
   - queued commands are the common async boundary for system-generated work

## Persist Across Terminal Disconnection

Two different persistence models exist:

### Daemon/assistant persistence

Strongly inferred from public SDK types:

- the **parent daemon** owns the remote-control WebSocket/session
- if the child agent subprocess crashes, the parent respawns it
- claude.ai keeps the same session
- this is called out explicitly in the `connectRemoteControl()` docs

### Local background-session persistence

Verified:

- background sessions register themselves in `~/.claude/sessions`
- `kind` distinguishes `bg`, `daemon`, and `daemon-worker`
- bg sessions detach from tmux instead of exiting, so they survive terminal disconnects

## Deliver Results To The User

Verified from public bridge handle docs:

1. Agent output is streamed into the remote-control handle via `write(msg)`
2. End-of-turn is signaled with `sendResult()`
3. Remote-control handle exposes:
   - `sessionUrl`
   - `environmentId`
   - `bridgeSessionId`
4. The bridge/transport code writes result events before teardown so the remote side sees the final state

For local background sessions:

- results are also reflected via session registry state, transcript/log persistence, and the `logs/attach` command surface, though those command bodies are not present in this checkout.

## Bottom Line

What is fully visible here:

- the daemon entrypoints
- the headless runner
- the proactive tick/Sleep contract
- the local cron scheduler
- the remote trigger API tool
- the session registry used by background/daemon processes
- the worktree/tmux integration
- the permission model for headless agents

What is not visible here:

- the concrete daemon supervisor
- the worker dispatch table
- the proactive module implementation itself
- the assistant module implementation
- the `ps/logs/attach/kill` command implementations

So the architecture is clear, but the final daemon-specific glue is missing from this repo snapshot.
