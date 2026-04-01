# Remote And Bridge Protocol Notes

Generated: 2026-04-01
Extraction basis:
- remote session client behavior
- bridge supervisor behavior

## Scope

This document covers the visible protocol and state-machine behavior exposed by the source.

It does not attempt to reconstruct the entire server-side API. It focuses on what the client code guarantees and assumes.

It is written to be source-free: the message types, flows, and operational semantics below are the contract a reader needs in order to understand remote sessions and bridge mode.

## 1. RemoteSessionManager Contract

The remote session manager is the clearest high-level client protocol surface.

### Remote session config

`RemoteSessionConfig` (`50-62`) contains:

- `sessionId`
- `getAccessToken()`
- `orgUuid`
- `hasInitialPrompt?`
- `viewerOnly?`

`viewerOnly` is important. The comments explicitly say it disables:

- Ctrl+C / Escape interrupt forwarding
- the 60-second reconnect timeout
- session-title updates

and is used by `claude assistant`.

### Callback contract

`RemoteSessionCallbacks` (`64-85`) exposes:

- `onMessage`
- `onPermissionRequest`
- `onPermissionCancelled`
- `onConnected`
- `onDisconnected`
- `onReconnecting`
- `onError`

This confirms that permission handling is part of the remote session protocol, not just local UI logic.

### Message classes

The manager distinguishes:

- `SDKMessage`
- `control_request`
- `control_response`
- `control_cancel_request`

See `isSDKMessage()` at `22-34` and `handleMessage()` at `146-184`.

### Permission request flow

When a remote `control_request` arrives (`189-213`):

- if subtype is `can_use_tool`, it is stored in `pendingPermissionRequests`
- callbacks are fired with the request and `request_id`
- unsupported control subtypes get an explicit error response rather than being dropped

This is a good protocol hygiene detail: the client avoids leaving the server hanging on unknown requests.

### Permission response shape

`RemotePermissionResponse` (`40-48`) is intentionally simplified:

- allow: `{ behavior: 'allow', updatedInput }`
- deny: `{ behavior: 'deny', message }`

Permission responses are wrapped in a `control_response` envelope with subtype `success`.

### End-to-end remote permission sequence

The visible remote permission round-trip is:

1. remote side sends `control_request`
2. subtype is `can_use_tool`
3. client stores the request by `request_id`
4. UI callback decides allow or deny
5. client sends `control_response`
6. response payload is either:
   - allow:
     `behavior: allow` plus `updatedInput`
   - deny:
     `behavior: deny` plus user-facing `message`
7. server may also send `control_cancel_request` if the request is no longer relevant

This is important because it shows remote sessions are not "fire and forget". They preserve the same permission mediation model as local sessions.

### Transport split

The manager explicitly coordinates two transports (`87-94`):

- WebSocket for incoming session messages and control traffic
- HTTP POST for sending user messages to the remote session

Outbound user messages go over HTTP POST rather than the WebSocket channel.

### Interrupt contract

Interrupt is an explicit control message with subtype `interrupt`.

### Connection lifecycle surface

The visible client lifecycle operations are:

- `connect()`
- `isConnected()`
- `cancelSession()`
- `disconnect()`
- `reconnect()`

Viewer-only sessions alter that lifecycle:

- they do not forward Ctrl+C / Escape interrupts
- they do not use the normal 60-second reconnect timeout behavior
- they do not update the session title

That means "viewer" is a real protocol role, not just a cosmetic UI mode.

## 2. Bridge Loop Contract

The bridge loop is the lower-level supervisor side.

### Backoff model

`BackoffConfig` (`59-70`) separates:

- connection retry timing
- general retry timing
- shutdown grace timing
- stop-work retry base delay

Default values (`72-79`):

- connection initial: 2s
- connection cap: 120s
- connection give-up: 10m
- general initial: 500ms
- general cap: 30s
- general give-up: 10m

### Session-management state

The bridge loop tracks:

- active session handles
- session start times
- work IDs
- compat session IDs
- ingress JWTs
- session timers
- completed work IDs
- worktree info
- timed-out sessions
- titled sessions
- cleanup promises

This is strong evidence that bridge mode is a session supervisor, not a thin transport shim.

### Multi-session support

Multi-session spawn is feature-gated.

The comments make the feature scope explicit:

- multiple sessions per environment
- staged rollout
- blocking gate fetch to avoid stale-cache false negatives

### Sleep/wake detection

Sleep/wake detection uses a threshold larger than normal retry backoff so ordinary reconnect delays do not look like host sleep.

That is a small but important resilience detail.

### Child spawn behavior

Child spawning handles the difference between:

- bundled binaries, where `process.execPath` is the Claude binary
- npm installs, where `node cli.js` requires the script path before CLI flags

This is a protocol detail between parent and child process invocation, not just a portability helper.

### What the bridge actually supervises

The bridge loop is responsible for:

- polling for work from the remote control service
- spawning local Claude child sessions
- associating work IDs with sessions
- keeping per-session ingress JWTs for heartbeat auth
- refreshing or re-dispatching expired work leases
- tracking worktrees and cleanup
- deciding when a session is complete, interrupted, failed, or timed out
- shutting down cleanly without leaking active work or temporary worktrees

## 3. Heartbeat And Token Refresh

The bridge loop contains unusually explicit comments around lease and JWT behavior.

### Heartbeats

Heartbeat of active work returns one of:

- `ok`
- `auth_failed`
- `fatal`
- `failed`

Its logic shows:

- a heartbeat authenticates with an ingress JWT, not necessarily the current OAuth token
- 401/403 triggers re-queue behavior
- 404/410 is treated as fatal because the environment is gone

### Re-dispatch on auth failure

When heartbeat auth fails:

- the bridge calls `api.reconnectSession(environmentId, sessionId)`
- comments explain why: otherwise work stays ACKed out of the queue and polling returns empty forever

### v2 session behavior

The bridge distinguishes v2 sessions from earlier ones.

The refresh behavior differs:

- v1 children can receive refreshed OAuth directly
- v2 children cannot use OAuth tokens and must be re-dispatched for fresh JWT-bearing work

This is one of the clearest protocol differences visible in the source.

### Poll / heartbeat model

The bridge is not a simple long-poll worker. It has two coordinated loops:

- a poll loop for acquiring work
- a heartbeat loop for keeping already-accepted work alive

Important semantics:

- heartbeats authenticate with ingress JWTs, not just the current OAuth token
- auth failure triggers `reconnectSession(...)` so work is re-queued with fresh credentials
- 404/410 on heartbeat is treated as fatal because the environment is gone
- a special heartbeat-only mode exists when the bridge is at capacity and cannot accept more work yet

Operationally, this means the bridge is designed for long-lived leased work, not short stateless requests.

## 4. Remote Protocol Conclusions

At a high level, the remote/bridge stack looks like this:

### High-level remote client

- `RemoteSessionManager`
- WebSocket subscription for messages and control
- HTTP POST for outbound user events
- explicit permission round-trips

### Lower-level bridge supervisor

- polls for work
- spawns child Claude sessions
- tracks work leases and ingress JWTs
- heartbeats active work
- reconnects sessions on token expiry
- manages worktrees and cleanup

That is enough source evidence to treat bridge mode as a durable orchestration subsystem with its own protocol and state machine.

## 5. Concrete Mental Model

The cleanest way to understand the subsystem is:

- `RemoteSessionManager` is the session client used by an interactive consumer
- the bridge loop is the worker daemon that acquires, supervises, and heartbeats remote work
- WebSocket traffic carries inbound messages and control requests
- HTTP POST carries outbound user events
- permission requests are explicit protocol messages, not implicit UI events
- session leases are maintained by heartbeat using ingress tokens
- token expiry does not just refresh credentials locally; in some modes it requires server-side re-dispatch

That is a much stronger contract than "remote terminal access". It is a real session-orchestration protocol.
