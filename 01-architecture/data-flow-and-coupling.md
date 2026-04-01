# Data Flow, Coupling & Architectural Pressure Points

**Added by:**  blind-spot analysis (April 2, 2026)
**Technique:** Import graph analysis, data flow tracing, singleton detection, circular dependency audit

---

## Import Graph Hotspots (Top 20 Most-Imported Modules)

These are the architectural pillars — the modules everything depends on. Breaking changes here cascade everywhere.

| Rank | Module | Import Count | Role |
|------|--------|-------------|------|
| 1 | `react` | 753 | UI framework |
| 2 | `react/compiler-runtime` | 395 | React compiler optimization |
| 3 | `path` | 250 | File path manipulation |
| 4 | `ink.js` | 240+145 = 385 | Terminal UI rendering (two path variants) |
| 5 | `bun:bundle` | 196 | Feature flag system (build-time DCE) |
| 6 | `Tool.js` | 157 | Core tool definition/execution framework |
| 7 | `fs/promises` | 136 | Async file operations |
| 8 | `zod/v4` | 125 | Schema validation |
| 9 | `commands.js` | 124 | Command registry |
| 10 | `crypto` | 120 | Cryptographic operations |
| 11 | `bootstrap/state.js` | 105 | Global session state singleton |
| 12 | `debug.js` | 96+85 = 181 | Debug logging (multiple path variants) |
| 13 | `utils/errors.js` | 93 | Error handling utilities |
| 14 | `figures` | 89 | Unicode box drawing characters |
| 15 | `utils/log.js` | 87 | Logging infrastructure |
| 16 | `utils/slowOperations.js` | 80 | Performance measurement wrapper |
| 17 | `utils/lazySchema.js` | 58 | Deferred schema validation |
| 18 | `axios` | 57 | HTTP client |

**Key insight:** `ink.js` (terminal rendering), `Tool.js` (execution), `bootstrap/state.js` (globals), and `commands.js` form the core architectural quadrant. Multiple path variants for `ink.js` and `debug.js` indicate path aliasing complexity.

---

## The State Singleton Problem (bootstrap/state.ts)

`bootstrap/state.ts` is the single point of truth for ~250+ mutable fields. 30+ files import and call setters on this state. No centralized transaction log. No undo capability.

### Module-Level Mutable State (15+ `let` variables):
- `interactionTimeDirty: boolean`
- `outputTokensAtTurnStart: number`
- `currentTurnTokenBudget: number | null`
- `budgetContinuationCount: number`
- `scrollDraining: boolean`
- `scrollDrainTimer: ReturnType<typeof setTimeout> | undefined`

### STATE Object (~180+ fields):
- Session metrics (cost, duration, tool counts)
- Auth tokens: `oauthTokenFromFd`, `apiKeyFromFd`, `sessionIngressToken`
- Telemetry counters (meter, sessionCounter, costCounter)
- Plugin metadata and hook registrations
- Slow operation traces

### Mutation Pattern:
Direct field assignment — no middleware, no event log:
```typescript
export function setOauthTokenFromFd(token: string | null): void {
  STATE.oauthTokenFromFd = token  // Direct mutation, no audit trail
}
```

### Concurrency Risk:
All mutable singletons across the codebase use simple boolean or null flags, not proper locks/mutexes. History flush (`history.ts:isWriting`) uses a boolean flag, not a real lock.

---

## Sensitive Data Flow Paths

### OAuth Token Lifecycle:
1. **Entry:** File descriptor → `utils/authFileDescriptor.ts:readTokenFromWellKnownFile()`
2. **Storage:** `bootstrap/state.ts:setOauthTokenFromFd()` → in-memory STATE
3. **Retrieval:** `utils/auth.ts:getOAuthTokenFromFileDescriptor()` → returns to Anthropic SDK
4. **Subprocess Fallback:** Writes to disk at `~/.claude/remote/.oauth_token` (mode 0o600)

### API Key Flow:
- Entry: `setApiKeyFromFd()` or env var `ANTHROPIC_API_KEY`
- Validation: `utils/auth.ts:getAuthToken()` checks: OAuth FD → apiKeyHelper (settings) → env var
- Passed to: `@anthropic-ai/sdk` with no intermediate encryption or access control wrapper

**Finding:** OAuth tokens and API keys are mixed into the STATE singleton without separate encryption or access control. Both are passed to external SDKs without intermediate wrapping.

---

## Error Propagation Architecture

### Files with Most Try/Catch Blocks:
| File | Try/Catch Count | Pattern |
|------|-----------------|---------|
| `utils/nativeInstaller/installer.ts` | 50 | Sequential retries with fallbacks |
| `utils/sessionStorage.ts` | 35 | Recovery from corrupted state |
| `services/mcp/client.ts` | 29 | MCP server connection recovery |
| `utils/plugins/marketplaceManager.ts` | 30 | Plugin download/install fallbacks |
| `main.tsx` | 26 | Bootstrap-stage error handling |

### Swallowed Errors (Silent Failures):
1. `ink/ink.tsx`: `} catch { /* stream may be destroyed */ }`
2. `ink/components/App.tsx`: `} catch (error) { // In Bun, uncaught throw...`
3. `main.tsx`: `} catch { // Silently ignore errors - this is just for analytics }`
4. `upstreamproxy/relay.ts`: `} catch { // already closing }`

MCP client errors use exponential backoff but swallow connection failures silently, potentially masking persistent auth issues.

---

## Pub/Sub Signal System

Centralized in `utils/signal.ts` with 15+ channels:

| Signal | File | Purpose |
|--------|------|---------|
| `sessionSwitched` | bootstrap/state.ts | Session lifecycle |
| `keybindingsChanged` | keybindings/loadUserBindings.ts | File watcher on ~/.claude/keybindings.json |
| `settingsChanged` | utils/settings/changeDetector.ts | Config change propagation |
| `cooldownExpired` | utils/fastMode.ts | Rate limiter expiry |
| `queueChanged` | utils/messageQueueManager.ts | Message dispatch |
| `classifierChecking` | utils/classifierApprovals.ts | Auto-mode classifier UI |
| `skillsChanged` | utils/skills/skillChangeDetector.ts | Skill file watcher |

`settingsChanged` is subscribed by both plugin loader and permissionsLoader — implicit coupling.

---

## Circular Dependency Mitigations

**Primary pattern:** `bootstrap/state.ts` → Tools → Tool.ts → state/AppState.tsx

Mitigated by type-only imports:
```typescript
// Tool.ts line 78
import type { AppState } from './state/AppState.js'  // Type-only, no runtime cycle
```

Additional mitigation via `utils/lazySchema.js` (58 importers) — defers heavy Zod schema binding to prevent circular dependency at module evaluation time.

---

## Implicit Singleton Inventory

| File | Mutable State | Risk |
|------|--------------|------|
| `history.ts` | `pendingEntries[]`, `isWriting`, `currentFlushPromise` | Race condition on multi-flush |
| `bridge/bridgeDebug.ts` | `debugHandle`, `faultQueue[]` | Cross-process without serialization |
| `ink/terminal-focus-state.ts` | `focusState`, `resolvers`, `subscribers` | Promise race on multiple resolvers |
| `utils/sessionActivity.ts` | `activityCallback` | Last setter wins |
| `context/notifications.tsx` | `currentTimeoutId` | No queue, simplistic timeout |

All use boolean/null flags instead of proper synchronization primitives.
