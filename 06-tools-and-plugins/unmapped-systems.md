# Unmapped Systems: Tasks, Commands, Services, Hooks & Hidden Tools

**Added by:**  blind-spot analysis (April 2, 2026)
**Technique:** Full directory reads of unmapped modules + command registry analysis

---

## Task System (7 Types — Entirely Missed)

The task system was completely absent from prior analysis. It manages all background work through a polymorphic framework.

| Task Type | Purpose | Key Mechanism |
|-----------|---------|---------------|
| LocalShellTask | Bash command execution | Zombie process cleanup, exit code tracking |
| LocalAgentTask | Core sub-agent execution | 50-message UI cap, progress tracking, abort control |
| InProcessTeammateTask | Async swarm teammates | Plan mode gates, conversation compaction |
| LocalMainSessionTask | Backgrounding main query | Isolated transcripts, session continuity |
| DreamTask | Auto-memory consolidation | Surfaces in UI, max 30 turns, 5 recent activities |
| RemoteAgentTask | Remote/workflow tracking | WebSocket subscription, polling lifecycle |
| Base Task | Polymorphic framework | Immutable state, terminal guards, 36^8 IDs |

**Security:** Task IDs use 36^8 combinations (2.8T) to resist symlink attacks. O_NOFOLLOW prevents symlink following. TEAMMATE_MESSAGES_UI_CAP = 50 prevents 36.8GB RSS in 292-agent swarm scenarios.

---

## Undocumented Slash Commands (40+)

### Stub Commands (Disabled — isEnabled: false):
bughunter, teleport, good-claude, ant-trace, ctx_viz, mock-limits, perf-issue, summary

### Feature-Gated Commands:

| Command | Purpose | Feature Gate |
|---------|---------|-------------|
| /brief | Toggle brief-only mode | KAIROS or KAIROS_BRIEF |
| /voice | Toggle voice mode | VOICE_MODE |
| /remote-control (/rc) | Connect for remote control | BRIDGE_MODE |
| /bridge-kick | ANT: inject bridge failures | BRIDGE_MODE + USER_TYPE='ant' |
| /buddy | AI companion pet | BUDDY |
| /fork | Subagent forking | FORK_SUBAGENT |
| /peers | User messaging/inbox | UDS_INBOX |
| /workflows | Workflow script execution | WORKFLOW_SCRIPTS |
| /ultraplan | Remote plan optimization | ULTRAPLAN (ANT-ONLY) |
| /subscribe-pr | GitHub webhook subscriptions | KAIROS_GITHUB_WEBHOOKS (ANT-ONLY) |

### Notable Visible Commands:

| Command | Description |
|---------|-------------|
| /stickers | Opens stickermule.com/claudecode (swag marketing) |
| /mobile (/ios, /android) | QR code for Claude mobile app |
| /btw | Side question without interrupting main conversation |
| /passes | Share free week + earn extra usage (referral) |
| /rewind (/checkpoint) | Restore code/conversation to checkpoint |
| /desktop (/app) | Continue session in Claude Desktop (macOS + Win x64) |
| /fast | Toggle fast mode (lighter model) |
| /effort | Set effort: low/medium/high/max/auto |
| /sandbox | Toggle sandbox + exclusion patterns |
| /advisor | Set advisor model for suggestions |

### /insights (3,200 lines — Largest Command)
Session analysis reports using Opus model. ANT-ONLY: Can SSH into remote Coder homespaces. Extracts session metadata, tool usage stats, language breakdown, friction points. Generates narrative HTML output.

### /security-review
Focuses on HIGH-CONFIDENCE exploitable vulnerabilities only: SQL/NoSQL/command/XXE/template/path-traversal injection, auth bypass, crypto flaws, RCE, deserialization, SSRF, CORS. Explicitly excludes DoS and rate-limiting issues.

---

## 16 Unmapped Service Directories

| Service | Purpose | Key Constants |
|---------|---------|---------------|
| autoDream | Background memory consolidation (forked subagent) | minHours=24, minSessions=5, stale lock=60min |
| extractMemories | Session transcript → auto-memory files | Max 5 turns, coalesced runs, mutual exclusion |
| SessionMemory | Periodic session progress documentation | Token count thresholds, background summarization |
| MagicDocs | Auto-maintain files with `# MAGIC DOC:` header | Sonnet model, FileEdit-only tool access |
| PromptSuggestion | Next-step suggestions via forked Haiku | Parallel speculation, abort control |
| AgentSummary | 30-sec background sub-agent summaries | 3-5 word present-tense with filenames |
| lsp | Language Server Protocol integration | Per-extension routing, diagnostic caching |
| oauth | OAuth/PKCE token management | Local redirect server, param redaction |
| plugins | Plugin install/uninstall/enable/disable | 4 scopes: user/project/local/managed |
| policyLimits | Org-level policy enforcement | 1hr cache, ETag, fail-open, SHA-256 dedup |
| remoteManagedSettings | Enterprise remote settings sync | Atomic writes, permission preservation |
| settingsSync | Cross-environment settings sync | Push/pull, 500KB max per file |
| teamMemorySync | Team memory per git-remote | Secret scanning, 250KB/entry, server-wins |
| tips | Contextual tips and suggestions | IDE detection, file pattern matching |
| toolUseSummary | Git-commit-style tool batch summaries | Haiku model, 300 char truncation |
| analytics | Central event logging with PII protection | Queue-before-sink, _PROTO_* tagging |

---

## 17 Unmapped Utility Subsystems

| Utility | Purpose | Key Finding |
|---------|---------|-------------|
| swarm | Multi-agent coordination | File-based mailbox IPC, 500ms poll, O_NOFOLLOW |
| ultraplan | Remote planning with browser approval | 3sec poll, 30min timeout, sentinel for local exec |
| teleport | Session migration to cloud (CCR) | Exponential backoff, git bundle seeding |
| computerUse | MCP bridge for desktop automation | 9 macOS terminal bundle IDs, screenshot exclusion |
| claudeInChrome | Native messaging to 7 browsers | Unix sockets/named pipes, MAX_TRACKED_TABS=200 |
| deepLink | claude-cli://open protocol handler | 5000 char query limit, control char rejection |
| dxt | MCPB manifest validation | Lazy imports defer ~700KB from startup heap |
| filePersistence | BYOC file sync | 5GB cap, concurrent upload, FILE_COUNT_LIMIT |
| suggestions | Fuse.js fuzzy search engine | 3-tier weighting (name:3, parts:2, desc:0.5) |
| secureStorage | Platform credential storage | macOS Keychain, Linux libsecret (TODO), plaintext fallback |
| settings/mdm | MDM policy reading | macOS plutil, Windows registry |
| todo | TodoWrite tool backing | Task state management and persistence |
| task | Disk task output management | Queue-based async writes, 5GB cap, O_NOFOLLOW |
| model | Model selection & pricing | 12 codename guards via excluded-strings.txt |
| memory | Memory type definitions | 6 types: User/Project/Local/Managed/AutoMem/TeamMem |
| git | Git state inspection | INI-style config parser, gitignore matching |
| background/remote | Remote session eligibility | 5-step precondition chain |

---

## Buddy System (src/buddy/ — Entirely Missed)

Tamagotchi companion with procedural generation:
- **18 species** with 3-frame ASCII animations (names encoded via String.fromCharCode to avoid excluded-strings.txt)
- **Rarity tiers:** common=60, uncommon=25, rare=10, epic=4, legendary=1
- **RPG stats:** DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK (one peak, one dump stat)
- **Deterministic:** Mulberry32 PRNG seeded from hash(userId + 'friend-2026-401')
- **Shininess:** 1% chance
- **Persistence:** Bones regenerated from userId on read; soul (name/personality) stored in config
- **Teaser window:** April 1-7, 2026; lives forever after

---

## Memory Directory System (src/memdir/ — Entirely Missed)

Semantic memory with AI selection:
- **4 memory types:** user, feedback, project, reference
- **Sonnet-powered selector:** Ranks up to 5 most-relevant files per query
- **Index cap:** 200 lines / 25KB (MEMORY.md loaded into system prompt)
- **Path security:** Rejects relative paths, root dirs, UNC paths, null bytes
- **Base path:** `~/.claude/projects/{sanitized-git-root}/memory/`
- **Feature gate:** feature('EXTRACT_MEMORIES') + tengu_passport_quail

---

## Coordinator Mode (src/coordinator/ — Under-Analyzed)

Multi-agent orchestration with 63-line system prompt enforcing 4-phase workflow:
1. **Research** — Gather information via workers
2. **Synthesis** — Coordinator MUST understand findings before delegating
3. **Implementation** — Direct workers with specific specs
4. **Verification** — Validate results

Workers communicate via `<task-notification>` XML wrappers. Workers cannot poll — they push completion notifications. Internal tools hidden from coordinators: TEAM_CREATE, TEAM_DELETE, SEND_MESSAGE, SYNTHETIC_OUTPUT.
