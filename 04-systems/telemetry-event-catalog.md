# Telemetry Event Catalog

Generated: 2026-04-01
Extraction basis:
- analytics event pipeline
- analytics metadata sanitization
- generated event envelope type
- all event logging callsites

## Bottom Line

The product references 666 unique analytics event names.

The telemetry system is not a side concern. It is effectively a product map for:

- agent behavior
- permissions
- OAuth
- MCP
- remote/bridge
- plugin lifecycle
- compaction
- session memory
- updater behavior
- worktree and file history

Companion raw inventory in this analysis bundle:

- `analysis/telemetry-event-inventory.md`

## 1. Event Pipeline

The analytics entrypoint is intentionally dependency-light:

- events queue before a sink is attached
- `attachAnalyticsSink()` drains the queue asynchronously
- the public API is just `logEvent()` and `logEventAsync()`

Important design details in the analytics entrypoint:

- strings are intentionally discouraged in raw metadata types to reduce accidental code/path logging
- `_PROTO_*` fields are stripped before general-access sinks and routed only to privileged first-party columns

## 2. Event Envelope

The generated proto type `ClaudeCodeInternalEvent` includes:

- `event_name`
- `session_id`
- `additional_metadata`
- `auth`
- `parent_session_id`

This tells us the durable event contract is:

- one named event
- one session identity
- one auth block
- one freeform metadata JSON payload
- optional parent-session linkage

## 3. Metadata Privacy Model

The metadata layer shows deliberate PII controls:

- MCP tool names default to `mcp_tool` unless logging is explicitly allowed
- skill names are only extracted in controlled ways
- detailed tool-input serialization is gated behind `OTEL_LOG_TOOL_DETAILS`
- nested tool inputs are truncated and depth-limited before serialization

Important privacy gates:

- tool names are sanitized before analytics logging in privacy-sensitive cases
- detailed tool-name logging is separately gated
- detailed tool-input logging is separately gated
- tool-input extraction is bounded and truncated before serialization

## 4. Largest Event Families

Grouping the 666 unique events by the first token after `tengu_` produces these largest families:

| Family | Approx. Unique Events |
|---|---:|
| `mcp_*` | 50 |
| `bash_*` | 47 |
| `bridge_*` | 46 |
| `oauth_*` | 43 |
| `file_*` | 39 |
| `session_*` | 29 |
| `tool_*` | 23 |
| `native_*` | 22 |
| `agent_*` | 21 |
| `auto_*` | 20 |
| `plugin_*` | 20 |
| `teleport_*` | 18 |
| `streaming_*` | 17 |
| `api_*` | 16 |

This gives a reliable picture of where the product team has invested the most operational instrumentation.

## 5. Notable Event Families

### 5A. API And Model Calls

Representative events:

- `tengu_api_query`
- `tengu_api_success`
- `tengu_api_error`
- `tengu_api_retry`
- `tengu_api_opus_fallback_triggered`
- `tengu_model_fallback_triggered`
- `tengu_model_whitespace_response`

These events instrument both request lifecycle and model-behavior anomalies.

### 5B. Tool And Permission Flow

Representative events:

- `tengu_tool_use_can_use_tool_allowed`
- `tengu_tool_use_can_use_tool_rejected`
- `tengu_tool_use_show_permission_request`
- `tengu_tool_use_success`
- `tengu_tool_use_error`
- `tengu_tool_result_pairing_repaired`
- `tengu_tool_input_json_parse_fail`
- `tengu_permission_request_option_selected`

This is direct evidence that the model-tool contract is heavily observed in production.

### 5C. Bash Safety

Representative events:

- `tengu_bash_security_check_triggered`
- `tengu_bash_ast_too_complex`
- `tengu_bash_command_explicitly_backgrounded`
- `tengu_bash_tool_command_executed`

The bash surface is one of the most instrumented subsystems in the product.

### 5D. MCP And Connector Lifecycle

Representative events:

- `tengu_mcp_add`
- `tengu_mcp_delete`
- `tengu_mcp_server_connection_succeeded`
- `tengu_mcp_server_connection_failed`
- `tengu_mcp_server_needs_auth`
- `tengu_mcp_oauth_flow_start`
- `tengu_mcp_oauth_flow_success`
- `tengu_mcp_tool_call_auth_error`
- `tengu_mcp_large_result_handled`
- `tengu_mcp_channel_message`

This confirms that MCP is first-class product infrastructure, not just optional extension code.

### 5E. Bridge / Remote / Teleport

Representative events:

- `tengu_bridge_started`
- `tengu_bridge_session_started`
- `tengu_bridge_session_done`
- `tengu_bridge_repl_ws_connected`
- `tengu_bridge_repl_reconnected_in_place`
- `tengu_bridge_heartbeat_error`
- `tengu_remote_create_session`
- `tengu_remote_setup_started`
- `tengu_teleport_resume_session`
- `tengu_teleport_error_git_not_clean`

This family is strong evidence that remote execution and session mobility are major product areas.

### 5F. Plugin And Marketplace Lifecycle

Representative events:

- `tengu_plugin_install_command`
- `tengu_plugin_installed`
- `tengu_plugin_enabled_cli`
- `tengu_plugin_disabled_cli`
- `tengu_plugin_update_command`
- `tengu_marketplace_added`
- `tengu_marketplace_updated`
- `tengu_official_marketplace_auto_install`

### 5G. Memory And Compaction

Representative events:

- `tengu_compact`
- `tengu_compact_failed`
- `tengu_compact_ptl_retry`
- `tengu_cached_microcompact`
- `tengu_time_based_microcompact`
- `tengu_session_memory_init`
- `tengu_session_memory_extraction`
- `tengu_sm_compact_threshold_exceeded`
- `tengu_memdir_loaded`

### 5H. Updater / Installer / Native Runtime

Representative events:

- `tengu_auto_updater_success`
- `tengu_auto_updater_fail`
- `tengu_native_auto_updater_start`
- `tengu_native_install_package_success`
- `tengu_native_install_binary_failure`
- `tengu_version_lock_acquired`
- `tengu_version_lock_failed`

## 6. Signals About Product Priorities

The telemetry families imply the following product priorities:

- shell safety and permissioning are operationally important
- MCP and remote/bridge are deeply monitored
- plugin/marketplace behavior is now productized
- model behavior is watched for malformed tool input, whitespace responses, fallback cases, and stall conditions
- compaction and memory systems are important enough to warrant dedicated failure and recovery events

## 7. Reconstructing The Full Event Set

The unique event names in this document were extracted with:

```bash
rg -o --no-filename "logEvent(?:Async)?\\(\\s*'[^']+'" src \
  | sed "s/.*('//; s/'$//" \
  | sort | uniq
```

That command currently yields 666 unique names.

For self-contained transfer, the full alphabetical inventory is preserved separately in:

- `analysis/telemetry-event-inventory.md`

## 8. Highest-Value Future Extracts

If you want to go even deeper, the next telemetry-focused artifacts would be:

- per-event file/callsite index
- event family to subsystem map
- `_PROTO_*` field catalog for privileged first-party storage
- metadata field taxonomy by event family

Even without those, the current source already exposes a large amount of product intent through telemetry naming alone.
