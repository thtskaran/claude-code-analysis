# Claude Code v2.1.88 - Architecture Analysis Collection

This directory contains comprehensive reverse-engineering analysis of Claude Code's internal systems.

## Documents

### 01-architecture/hooks-system-deep-dive.md (2,306 lines)

**The definitive guide to Claude Code's 104 custom React hooks.**

Complete architectural analysis covering:

- **Hook Architecture Overview**: 8 domain categories, full dependency graph
- **Typeahead Suggestion Engine** (1,384 lines): Unified suggestion system merging commands, files, shell completions, Slack channels, and history
- **File Suggestion System** (811 lines): Rust FileIndex integration, git tracking, ignore patterns, background refresh
- **Voice Pipeline** (1,820 lines): Hold-to-talk STT, language detection, audio visualization, focus-mode lifecycle
- **Team/Swarm Coordination** (969 + 330 lines): Mailbox polling, permission brokering, distributed agent async flows
- **Remote Sessions** (1,327 lines): Always-on claude.ai bridge, CCR WebSocket adapter, failure recovery
- **Permission State Machine** (1,389 lines): Hierarchical decisions (hook → classifier → user → swarm leader), audit logging
- **Notification System** (16 hooks, 1,342 lines): Lifecycle management for IDE status, rate limits, LSP, model migrations
- **State Patterns**: AppState singleton, useSyncExternalStore for fine-grained updates, Promise-based async initialization
- **Complete Hook Catalog**: All 104 hooks listed by category with line counts and purposes
- **Key Constants & Timeouts**: 200ms release detection, 1000ms inbox polling, 250ms suggestion debouncing, etc.
- **Security Analysis**: 10 attack vectors with mitigations (permission forging, classifier abuse, file index poisoning, task race conditions, etc.)

**Key Stats:**
- 104 custom React hooks across 19,204 lines of code
- 8 architectural domains
- 27 hooks for input/suggestions
- 2 hooks for voice (1,820 LOC)
- 7 hooks for team/swarm coordination
- 16 notification hooks
- 40+ utility hooks (<100 LOC each)

**Reverse-Engineering Focus:**

This document answers:

1. How does Anthropic implement a real-time suggestion engine that merges 5+ sources?
2. How is the Rust FileIndex integrated with git tracking and ignore patterns?
3. How does hold-to-talk voice STT work with Anthropic's voice_stream endpoint?
4. How do distributed team agents coordinate via file-based mailbox IPC?
5. How does the permission state machine enforce security across hooks, classifiers, users, and swarm leaders?
6. What state patterns avoid unnecessary re-renders in a complex React application?
7. How are async operations coordinated with AbortControllers and Promise-based initialization?

---

## Technical Highlights

### Architecture Patterns

- **Lazy Loading**: Voice module deferred until user activates hold-to-talk (avoids TCC permission prompts)
- **Singleton Stores**: FileIndex, TasksV2Store persist across re-renders
- **Promise-Based Async**: Background index rebuilds backed by Promises to prevent race conditions
- **External Store Subscriptions**: AppState accessed via useSyncExternalStore for fine-grained updates
- **Event Sourcing**: Permission decisions logged with full context for auditing
- **Async Context Propagation**: AsyncLocalStorage for teammate context across async boundaries

### Performance Optimizations

- **Path List Signature Hashing**: O(700 samples) instead of 346k paths on large repos
- **Git Index Mtime Monitoring**: Detects tracked file changes without spawning git ls-files
- **Debouncing Strategy**: 250ms debounce on suggestion generation with per-source cancellation
- **Selection Preservation**: Match by ID when regenerating suggestions to avoid UI jank
- **Untracked File Background Fetch**: Non-blocking async merge of untracked files into index

### Security-Conscious Design

- **Permission Verification**: Verify plan approval sender (only team-lead accepted)
- **Schema Validation**: Zod validation of all external responses (permission updates, task files)
- **Audit Logging**: Every permission decision logged with tool name, message ID, decision reason
- **Failure Fuses**: Bridge connection fuse (3 consecutive failures → auto-disable)
- **Subprocess Cleanup**: AbortController ensures killed processes on cancellation
- **Input Filtering**: Shell suggestions filtered before rendering (no raw output)

---

## How to Use This Analysis

1. **For Hook Architecture**: Read "Hook Architecture Overview" section
2. **For Suggestion Engine Details**: Read "The Typeahead Suggestion Engine" section
3. **For Permission System Deep-Dive**: Read "Permission State Machine" section
4. **For Team Coordination**: Read "Team/Swarm Hooks" + "Remote Session Management"
5. **For State Pattern Learning**: Read "State Patterns & AppState Integration"
6. **For Complete Hook Reference**: See "Complete Hook Catalog" (all 104 hooks listed)
7. **For Security Review**: See "Security Considerations" (10 attack vectors + mitigations)

---

## Document Statistics

- **Total Lines**: 2,306 lines of analysis
- **Words**: ~28,000
- **Code Examples**: 50+
- **Sections**: 14
- **Tables**: 12
- **Diagrams**: 3 (ASCII flow charts)
- **Hook Coverage**: 104/104 (100%)
- **Code Analysis Depth**: 19,204 lines of source code examined

---

## Reverse-Engineering Methodology

This analysis was generated by:

1. **Comprehensive File Enumeration**: Located all 104 hooks across 3 subdirectories
2. **Focused Deep-Reads**: Read complete implementations of key files:
   - useTypeahead.tsx (1,384 lines)
   - useVoice.ts (1,144 lines)
   - useInboxPoller.ts (969 lines)
   - fileSuggestions.ts (811 lines)
   - useReplBridge.tsx (722 lines)
   - useRemoteSession.ts (605 lines)
   - useVoiceIntegration.tsx (676 lines)
3. **Dependency Chain Tracing**: Mapped how hooks depend on each other
4. **Constant Extraction**: Catalogued all key timeouts, intervals, and thresholds
5. **Security Analysis**: Identified 10 attack vectors and verified mitigations
6. **Integration Pattern Recognition**: Identified singleton stores, lazy loading, async patterns

---

## About Claude Code v2.1.88

Claude Code is Anthropic's AI-powered command-line IDE that:

- Runs as a CLI application with React for UI (Ink framework)
- Integrates with claude.ai via always-on WebSocket bridge
- Supports distributed "team" agents via file-based IPC (mailbox protocol)
- Implements hold-to-talk voice input via Anthropic's voice_stream STT
- Uses Rust native FileIndex for O(n) fuzzy file matching
- Features 3-level permission hierarchy: hook allowlist → classifier → user prompt → swarm leader
- Manages real-time suggestions across 5 sources (commands, files, shell, history, Slack)

The hook system is the beating heart of this application, coordinating input, voice, permissions, notifications, and distributed team coordination in a single React tree.

---

**Generated**: 2026-04-02
**Scope**: Claude Code v2.1.88 (19,204 LOC across 104 hooks)
**Analysis Depth**: Complete architectural reverse-engineering with code examples
