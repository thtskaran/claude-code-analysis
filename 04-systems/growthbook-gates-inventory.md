# GrowthBook Gates Inventory (tengu_* namespace)

**Added by:**  blind-spot analysis (April 2, 2026)
**Technique:** Exhaustive grep for tengu_ patterns across all source files

The original analysis documented 90 build-time feature() flags but missed the entire GrowthBook runtime experiment layer. 600+ `tengu_*` gates control A/B testing, gradual rollouts, and dynamic feature enablement.

---

## Critical Gates

| Gate | Purpose | Default |
|------|---------|---------|
| `tengu_anti_distill_fake_tool_injection` | Anti-distillation: inject decoy tools into system prompt | Off |
| `tengu_attribution_header` | Kill-switch for billing attribution header | On |
| `tengu_onyx_plover` | autoDream: background memory consolidation config | Off |
| `tengu_passport_quail` | Memory extraction enablement | Off |
| `tengu_bramble_lintel` | Memory extraction throttle (every N eligible turns) | 1 |
| `tengu_chomp_inflection` | Prompt suggestion enablement | Off |
| `tengu_session_memory` | Session memory feature | Off |
| `tengu_kairos_brief_config` | Kairos brief mode configuration | Off |
| `tengu_voice_mode` | Voice mode enablement | Off |
| `tengu_thinkback` | Year-in-review feature | Off |
| `tengu_ccr_bundle_seed_enabled` | CCR bundle seeding for remote sessions | Off |
| `tengu_cached_microcompact` | Cached microcompaction optimization | Off |
| `tengu_amber_quartz_disabled` | Voice mode kill-switch (false = enabled) | false |
| `tengu_ant_model_override` | ANT-only model override | ANT-only |
| `tengu_log_datadog_events` | Datadog event logging | Off |
| `tengu_enable_settings_sync_push` | Settings sync push enablement | Off |
| `tengu_scratch` | Scratchpad support in coordinator mode | Off |

---

## Infrastructure Endpoints Discovered

### Primary API
- `https://api.anthropic.com` — Main API (19 references)
- `https://api-staging.anthropic.com` — Staging (3 references)
- `https://claude.ai` — Web app origin (8 references)

### Internal APIs
- `https://api.anthropic.com/api/claude_code/metrics` — Metrics
- `https://api.anthropic.com/api/claude_code/policy_limits` — Policy limits
- `https://api.anthropic.com/api/claude_code/team_memory` — Team memory sync
- `https://api.anthropic.com/api/claude_code/user_sync` — Settings sync
- `https://api.anthropic.com/api/event_logging/batch` — Event batching
- `https://api.anthropic.com/api/oauth/profile` — OAuth profile
- `https://api.anthropic.com/api/oauth/account/settings` — Account settings
- `https://api.anthropic.com/api/oauth/claude_cli/create_api_key` — API key creation
- `https://api.anthropic.com/api/oauth/claude_cli/roles` — User roles
- `https://api.anthropic.com/mcp-registry/v0/servers` — MCP registry

### WebSocket
- `wss://api.anthropic.com/v1/sessions/ws/{sessionId}/subscribe` — Remote session subscription

### Telemetry
- `https://http-intake.logs.us5.datadoghq.com/api/v2/logs` — Datadog logs (US5 region)

### Cloud Provider Scopes
- `https://www.googleapis.com/auth/cloud-platform` — GCP OAuth scope
- `https://cognitiveservices.azure.com/.default` — Azure identity scope

---

## Additional Environment Variables (Not in Prior Analysis)

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_UNDERCOVER` | Enable stealth mode for public repos |
| `CLAUDE_CODE_COORDINATOR_MODE` | Enable multi-agent coordinator |
| `CLAUDE_CODE_DISABLE_SESSION_DATA_UPLOAD` | Disable ANT-ONLY session upload |
| `CLAUDE_CODE_UNATTENDED_RETRY` | Infinite retry on 429/529 errors |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | Disable auto-memory extraction |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS` | Disable experimental features |
| `CLAUDE_CODE_ATTRIBUTION_HEADER` | Disable billing attribution header |
| `CLAUDE_CODE_TEAMMATE_COMMAND` | Override swarm teammate command |
| `CLAUDE_CODE_AGENT_COLOR` | Set agent display color |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | Force plan mode before implementation |
| `CLAUDE_CODE_DEBUG_REPAINTS` | Show component owner chain in UI |
| `CLAUDE_CODE_COMMIT_LOG` | Log slow renders to file |
| `CLAUDE_CODE_PROFILE_STARTUP` | Full startup profiling with memory snapshots |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | Force-enable prompt suggestions |
| `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` | Override memory directory path |
| `CLAUDE_CODE_GB_BASE_URL` | Override GrowthBook endpoint |
| `CLAUDE_INTERNAL_FC_OVERRIDES` | Feature flag overrides (ANT-only) |
| `COO_RUNNING_ON_HOMESPACE` | Homespace detection |
| `USER_TYPE` | 'ant' = Anthropic employee mode |

---

## Build-Time Feature Flags (Complete — 95 found)

Extending the prior 90 with 5 newly discovered:

| New Flag | Purpose |
|----------|---------|
| `ABLATION_BASELINE` | A/B testing baseline control |
| `AGENT_MEMORY_SNAPSHOT` | Agent memory state persistence |
| `BG_SESSIONS` | Background session management |
| `CONTEXT_COLLAPSE` | Context window collapse tool |
| `PERFETTO_TRACING` | Google Perfetto-based tracing |
