# Swarm Orchestration, Bridge Protocol & SDK Entrypoints

**Added by:**  blind-spot analysis (April 2, 2026)
**Technique:** Full reads of bridge/ (31 files), remote/ (4 files), entrypoints/sdk/ (2 files), utils/swarm/ (all files)

---

## Swarm Orchestration (src/utils/swarm/)

Multi-agent coordination system using AsyncLocalStorage for context isolation.

### Architecture
- **In-process teammates:** Run within main process using `runWithTeammateContext()`
- **Pane backends:** tmux or iTerm2 for process-based teammates
- **IPC:** File-based mailbox at `~/.claude/teams/{teamName}/`

### Permission Synchronization
- Workers send `permission_request` messages via JSON files
- Leader reads from mailbox, shows interactive prompt, writes response
- Worker polls every `PERMISSION_POLL_INTERVAL_MS = 500`ms
- **Workers cannot self-approve** — enforced by architecture
- Callback registration handles async resolution with cleanup functions

### Key Constants
- `TEAM_LEAD_NAME = 'team-lead'`
- Lock retry: 30 attempts, 5-100ms exponential backoff (~2.6s max wait)
- `O_NOFOLLOW` flag prevents symlink attacks (Unix-only)

### Features
- Plan mode approval flow: teammates forced to enter plan mode before implementation
- Conversation compaction: in-process teammates auto-compact long histories
- Idle notification: `createIdleNotification()` tells leader when available
- Task claiming: auto-claims unblocked team tasks for distribution
- Formatted messages as XML `<teammate-message>` for transcript rendering

---

## Bridge System (src/bridge/ — 31 files)

Two-tier dispatch connecting local CLI to claude.ai web interface.

### v1 (Env-Based Bridge)
1. Environments API → Poll/Ack/Stop work items
2. Session spawn from work item
3. HybridTransport: WebSocket + POST fallback
4. Exponential backoff reconnection (2s initial → 2min cap)

### v2 (Env-Less Bridge)
1. Direct OAuth → POST /bridge
2. Worker JWT + epoch for stale worker discrimination
3. SSE stream + CCRClient write path
4. Epoch counter prevents stale worker interference

### Three Token Types
| Token | Purpose | Refresh |
|-------|---------|---------|
| OAuth | claude.ai login | Refresh on 401 |
| JWT session ingress | Session authentication | Proactive refresh 5min before expiry |
| Worker Epoch | Stale worker discrimination | Incremented per connection |

### Bridge Debug Commands (/bridge-kick — ANT-ONLY)
- `close <code>` — Fire ws_closed with WebSocket code
- `poll <status> [type]` — Inject poll failures
- `register fail [N]` / `register fatal` — Inject register failures
- `reconnect-session fail` — Fail reconnection
- `heartbeat <status>` — Inject heartbeat failure
- `reconnect` — Force reconnect
- `status` — Print bridge state

### Message Adaptation
Bidirectional format translation between SDK messages (used by web UI) and local Claude Code format. Permission request mediation between bridge and local approval system.

---

## Remote Session Management (src/remote/)

### Remote Session Lifecycle
1. Check eligibility (policy gate → login → env availability → git → GitHub app)
2. Create session via API
3. Subscribe via WebSocket: `wss://api.anthropic.com/v1/sessions/ws/{id}/subscribe`
4. Send events via HTTP: POST `/v1/sessions/{id}/events`
5. Poll for results with 100-event pages, backward iteration

### CCR Proxy URL Unwrapping
```javascript
CCR_PROXY_PATH_MARKERS = [
  '/v2/session_ingress/shttp/mcp/',
  '/v2/ccr-sessions/',
]
```
Extracts `mcp_url` query param from proxy URLs to recover original vendor endpoints.

---

## SDK Entrypoints (src/entrypoints/)

### Public API Surface
- `query()` — Primary query function
- `unstable_v2_createSession()` — Session creation (unstable API)
- Session management utilities

### SDK Schema (coreSchemas.ts — 1,889 lines)
Generated TypeScript types from Zod schemas defining:
- Control protocol for permission/model/interrupt requests
- Tool input/output schemas
- Message format schemas
- Event stream schemas

### MCP Entrypoint (entrypoints/mcp.ts)
**Validation gap:** `TODO — validate input types with zod` at line 136. MCP tool inputs currently NOT validated.

---

## Upstream Proxy Security (CCR Containers)

Hardened execution environment for cloud-hosted sessions:
1. Read session token from `/run/ccr/session_token`
2. `prctl(PR_SET_DUMPABLE, 0)` — block ptrace heap scraping
3. Download CA certificate from `/v1/code/upstreamproxy/ca-cert`
4. Start local CONNECT→WebSocket relay
5. Unlink token file (token remains heap-only)
6. Inject `HTTPS_PROXY` and `SSL_CERT_FILE` environment variables

---

## MCP Integration (8 Scopes)

5 transports: `stdio`, `sse`/`http`/`ws`, `sdk`, `claudeai-proxy`

Configuration scope priority:
1. `local` (highest)
2. `project`
3. `user`
4. `userSettings`
5. `policySettings`
6. `enterprise`
7. `claudeai`
8. `dynamic` (lowest)

Duplicate detection via dedup signature across sources.
Atomic writes: temp file → datasync → chmod → rename.
