# Agent Subsystems & Multi-Agent Orchestration

Autonomous agent capabilities including swarm orchestration, coordinator system, daemon agents, session teleport, and remote bridge protocol. This section covers leader-follower team coordination, file-based IPC, permission brokering, and distributed agent communication.

## Documents

| Filename | Description | Lines |
|----------|-------------|-------|
| **swarm-orchestration-deep-dive.md** | Leader-follower swarm architecture, file-based mailbox IPC, three execution backends (tmux/iTerm2/in-process), permission brokering with team-lead approval, 1000ms inbox polling, teammate spawn lifecycle, color assignment and layout management | 848 |
| **coordinator-voice-moreright-deep-dive.md** | Coordinator mode for team leadership, voice pipeline integration, moreright system for policy enforcement, agent decision-making augmentation | 596 |
| **daemon-proactive-agent.md** | Persistent background agents, event-driven execution, proactive monitoring, async task management | 351 |
| **remote-bridge-protocol.md** | Remote bridge protocol for distributed coordination across network boundaries, message routing, session transparency | 296 |
| **session-teleport-deep-dive.md** | Session teleport for moving execution context between environments, state preservation, seamless switching | 1,246 |
| **swarm-bridge-sdk.md** | Bridge SDK for swarm communication, inter-agent messaging, team API | 141 |

## Architecture Diagrams

![Swarm Orchestration Architecture](../infographics/swarm-orchestration.svg)

Swarm orchestration: leader-follower team structure with file-based mailbox IPC and three execution backends (tmux, iTerm2, in-process).

![Agent Loop Flow](../infographics/agent-loop-flow.svg)

Agent loop execution: initialization → mailbox polling → permission checking → task execution → status update.

## Key Findings

### Swarm Orchestration

- **Leader-Follower Architecture**: Single leader coordinates multiple workers with distributed decision-making
- **Three Execution Backends**:
  - `TmuxBackend` — Terminal multiplexer integration for pane management
  - `ITermBackend` — Native iTerm2 split-pane API integration
  - `InProcessBackend` — Same-process execution with isolated context
- **File-Based Mailbox IPC**: Message passing via `~/.claude/teams/{team-name}/mailbox/{agent-name}/`
  - `messages.json` for message storage
  - Atomic updates with lockfile synchronization
  - 1000ms polling interval for inbox checks
- **Permission Brokering**: Only team-lead approved as request sender; delegation maintains audit trail
- **Teammate Spawn Lifecycle**: Registration → environment setup → agent loop initialization → team context injection
- **Layout Management**: Color assignment and pane delegation for visual team identification

### Coordinator System

- **Coordinator Mode**: Leadership delegation for policy-driven team decisions
- **Voice Pipeline**: Audio/voice integration for agent communication
- **Moreright System**: Policy enforcement and decision validation for safety-critical operations
- **Team Memory Sharing**: Context propagation across swarm agents

### Session Teleport

- **Cross-Environment Movement**: Transfer execution context between local and remote environments
- **State Preservation**: Maintain session state, memory, and task context during transition
- **Seamless Switching**: Transparent switchover without task interruption

### Daemon & Proactive Agents

- **Background Execution**: Persistent agents running independently of main session
- **Event-Driven Model**: React to file changes, timers, and external signals
- **Proactive Monitoring**: Autonomous checks and pre-emptive actions

### Remote Bridge Protocol

- **Distributed Coordination**: Network-transparent agent communication
- **Message Routing**: Reliable delivery across network boundaries
- **Security Isolation**: Sandboxed execution with capability-based access control

---

**Analysis Date**: 2026-04-02
**Codebase Path**: `/sessions/cool-friendly-einstein/mnt/claude-code/src/utils/swarm/`
