# Constants & Thresholds Encyclopedia

Generated from product-wide constant declarations plus inline timer literals. Nearby comments are included when available to preserve intent.

Self-contained reading note:

- the `File:Line` column is provenance only
- the transferable artifact is the constant name, value, category, and nearby intent
- a reader does not need the original source tree to use this document

## How to read this reference

- The constant name and value are exact extraction artifacts.
- The `Category` and `Context/Comment` columns are the important human-readable layer.
- If a name is duplicated, treat each row as a separate usage contract rather than assuming a single global definition.
- If a name is opaque, auto-generated, or implementation-shaped, use the context text as the canonical meaning.
- Rows named like `inline_setTimeout@...` or `inline_setInterval@...` capture operational magic numbers that matter even when they were not declared as named constants in code.

| Name | Value | File:Line | Category | Context/Comment |
|------|-------|-----------|----------|-----------------|
| HISTORY_PAGE_SIZE | `100` | src/assistant/sessionHistory.ts:7 | file-handling | export const HISTORY_PAGE_SIZE = 100 |
| MAX_IN_MEMORY_ERRORS | `100` | src/bootstrap/state.ts:1219 | feature-threshold | const MAX_IN_MEMORY_ERRORS = 100 |
| MAX_SLOW_OPERATIONS | `10` | src/bootstrap/state.ts:1566 | feature-threshold | Slow operations tracking for dev bar |
| SLOW_OPERATION_TTL_MS | `10000` | src/bootstrap/state.ts:1567 | timeout | Slow operations tracking for dev bar |
| EMPTY_POLL_LOG_INTERVAL | `100` | src/bridge/bridgeApi.ts:74 | timeout | const EMPTY_POLL_LOG_INTERVAL = 100 |
| STATUS_UPDATE_INTERVAL_MS | `1_000` | src/bridge/bridgeMain.ts:82 | timeout | const STATUS_UPDATE_INTERVAL_MS = 1_000 |
| MAX_ATTEMPTS | `3` | src/bridge/bridgeMain.ts:1634 | retry | const MAX_ATTEMPTS = 3 |
| TITLE_MAX_LEN | `80` | src/bridge/bridgeMain.ts:1953 | feature-threshold | const TITLE_MAX_LEN = 80 |
| MAX_WORKTREE_FANOUT | `50` | src/bridge/bridgePointer.ts:19 | file-handling | (50 is a LOT), but this caps the parallel stat() burst and guards against pathological setups. Above this, --continue falls back to current-dir-only. / |
| BRIDGE_POINTER_TTL_MS | `4 * 60 * 60 * 1000` | src/bridge/bridgePointer.ts:40 | timeout | concurrent bridges in different repos don't clobber each other. / |
| SHIMMER_INTERVAL_MS | `150` | src/bridge/bridgeStatusUtil.ts:21 | timeout | export const SHIMMER_INTERVAL_MS = 150 |
| DEBUG_MSG_LIMIT | `2000` | src/bridge/debugUtils.ts:9 | feature-threshold | const DEBUG_MSG_LIMIT = 2000 |
| REDACT_MIN_LENGTH | `16` | src/bridge/debugUtils.ts:24 | security | const REDACT_MIN_LENGTH = 16 |
| DOWNLOAD_TIMEOUT_MS | `30_000` | src/bridge/inboundAttachments.ts:25 | timeout | const DOWNLOAD_TIMEOUT_MS = 30_000 |
| TITLE_MAX_LEN | `50` | src/bridge/initReplBridge.ts:547 | feature-threshold | const TITLE_MAX_LEN = 50 |
| TOKEN_REFRESH_BUFFER_MS | `5 * 60 * 1000` | src/bridge/jwtUtils.ts:52 | timeout | const TOKEN_REFRESH_BUFFER_MS = 5 * 60 * 1000 |
| FALLBACK_REFRESH_INTERVAL_MS | `30 * 60 * 1000 // 30 minutes` | src/bridge/jwtUtils.ts:55 | timeout | const FALLBACK_REFRESH_INTERVAL_MS = 30 * 60 * 1000 // 30 minutes |
| MAX_REFRESH_FAILURES | `3` | src/bridge/jwtUtils.ts:58 | timeout | const MAX_REFRESH_FAILURES = 3 |
| REFRESH_RETRY_DELAY_MS | `60_000` | src/bridge/jwtUtils.ts:61 | timeout | const REFRESH_RETRY_DELAY_MS = 60_000 |
| POLL_INTERVAL_MS_NOT_AT_CAPACITY | `2000` | src/bridge/pollConfigDefaults.ts:13 | timeout | Governs user-visible "connecting…" latency on initial work pickup and recovery speed after the server re-dispatches a work item. / |
| POLL_INTERVAL_MS_AT_CAPACITY | `600_000` | src/bridge/pollConfigDefaults.ts:30 | timeout | failures, so poll is not the recovery path — it's strictly a liveness signal plus a backstop for permanent close. / |
| MULTISESSION_POLL_INTERVAL_MS_NOT_AT_CAPACITY | `` | src/bridge/pollConfigDefaults.ts:38 | timeout | preserve current behavior. Ops can tune these independently via the tengu_bridge_poll_interval_config GB flag. / |
| MULTISESSION_POLL_INTERVAL_MS_PARTIAL_CAPACITY | `` | src/bridge/pollConfigDefaults.ts:40 | timeout | / |
| MULTISESSION_POLL_INTERVAL_MS_AT_CAPACITY | `POLL_INTERVAL_MS_AT_CAPACITY` | src/bridge/pollConfigDefaults.ts:42 | timeout | const MULTISESSION_POLL_INTERVAL_MS_AT_CAPACITY = POLL_INTERVAL_MS_AT_CAPACITY |
| POLL_ERROR_INITIAL_DELAY_MS | `2_000` | src/bridge/replBridge.ts:244 | timeout | is truly dead. As long as the server accepts our poll, we keep waiting for it to re-dispatch the work item. / |
| POLL_ERROR_MAX_DELAY_MS | `60_000` | src/bridge/replBridge.ts:245 | timeout | for it to re-dispatch the work item. / |
| POLL_ERROR_GIVE_UP_MS | `15 * 60 * 1000` | src/bridge/replBridge.ts:246 | timeout | / |
| MAX_ENVIRONMENT_RECREATIONS | `3` | src/bridge/replBridge.ts:583 | feature-threshold | Shared counter for environment re-creations, used by both onEnvironmentLost and the abnormal-close handler. |
| MAX_ENVIRONMENT_RECREATIONS | `3` | src/bridge/replBridge.ts:1920 | feature-threshold | / |
| MAX_ACTIVITIES | `10` | src/bridge/sessionRunner.ts:16 | file-handling | const MAX_ACTIVITIES = 10 |
| MAX_STDERR_LINES | `10` | src/bridge/sessionRunner.ts:17 | file-handling | const MAX_STDERR_LINES = 10 |
| DEFAULT_SESSION_TIMEOUT_MS | `24 * 60 * 60 * 1000` | src/bridge/types.ts:2 | timeout | export const DEFAULT_SESSION_TIMEOUT_MS = 24 * 60 * 60 * 1000 |
| FADE_WINDOW | `6; // last ~3s the bubble dims so you know it's about to go` | src/buddy/CompanionSprite.tsx:18 | feature-threshold | const FADE_WINDOW = 6; // last ~3s the bubble dims so you know it's about to go |
| MIN_COLS_FOR_FULL_SPRITE | `100;` | src/buddy/CompanionSprite.tsx:152 | ui-rendering | export const MIN_COLS_FOR_FULL_SPRITE = 100; |
| SPRITE_BODY_WIDTH | `12;` | src/buddy/CompanionSprite.tsx:153 | ui-rendering | const SPRITE_BODY_WIDTH = 12; |
| BUBBLE_WIDTH | `36; // SpeechBubble box (34) + tail column` | src/buddy/CompanionSprite.tsx:156 | ui-rendering | const BUBBLE_WIDTH = 36; // SpeechBubble box (34) + tail column |
| NARROW_QUIP_CAP | `24;` | src/buddy/CompanionSprite.tsx:157 | ui-rendering | const NARROW_QUIP_CAP = 24; |
| inline_setInterval@CompanionSprite.tsx:202 | `const timer = setInterval(setT => setT((t: number) => t + 1), TICK_MS, setTick);` | src/buddy/CompanionSprite.tsx:202 | timeout | Inline timer magic number |
| JS_LINE_TERMINATORS | `/\u2028\|\u2029/g` | src/cli/ndjsonSafeStringify.ts:16 | feature-threshold | Single regex with alternation: the callback's one dispatch per match is cheaper than two full-string scans. |
| MAX_RECEIVED_UUIDS | `10_000` | src/cli/print.ts:394 | ui-rendering | Track message UUIDs received during the current session runtime |
| inline_setInterval@print.ts:549 | `const gcTimer = setInterval(Bun.gc, 1000)` | src/cli/print.ts:549 | timeout | Periodically force a full GC to keep memory usage in check |
| inline_setTimeout@print.ts:1832 | `// setTimeout(0) yields to the event loop so pending stdin messages` | src/cli/print.ts:1832 | timeout | Proactive mode: schedule a tick to keep the model looping autonomously. |
| POLL_INTERVAL_MS | `500` | src/cli/print.ts:2510 | timeout | Poll for messages while teammates are active This is needed because teammates may send messages while we're waiting Keep polling until the team is shut down |
| MAX_RESOLVED_TOOL_USE_IDS | `1000` | src/cli/structuredIO.ts:133 | timeout | Maximum number of resolved tool_use IDs to track. Once exceeded, the oldest entry is evicted. This bounds memory in very long sessions while keeping enough history to catch duplicate control_response deliveries. |
| DEFAULT_HEARTBEAT_INTERVAL_MS | `20_000` | src/cli/transports/ccrClient.ts:33 | timeout | const DEFAULT_HEARTBEAT_INTERVAL_MS = 20_000 |
| STREAM_EVENT_FLUSH_INTERVAL_MS | `100` | src/cli/transports/ccrClient.ts:42 | timeout | snapshot per flush — each emitted event is self-contained so a client connecting mid-stream sees complete text, not a fragment. / |
| MAX_CONSECUTIVE_AUTH_FAILURES | `10` | src/cli/transports/ccrClient.ts:68 | security | exp is in the future but server says 401 (userauth down, KMS hiccup, clock skew). 10 × 20s heartbeat ≈ 200s to ride it out. / |
| BATCH_FLUSH_INTERVAL_MS | `100` | src/cli/transports/HybridTransport.ts:12 | timeout | const BATCH_FLUSH_INTERVAL_MS = 100 |
| POST_TIMEOUT_MS | `15_000` | src/cli/transports/HybridTransport.ts:15 | timeout | Per-attempt POST timeout. Bounds how long a single stuck POST can block the serialized queue. Without this, a hung connection stalls all writes. |
| RECONNECT_BASE_DELAY_MS | `1000` | src/cli/transports/SSETransport.ts:16 | timeout | Configuration --------------------------------------------------------------------------- |
| RECONNECT_MAX_DELAY_MS | `30_000` | src/cli/transports/SSETransport.ts:17 | timeout | --------------------------------------------------------------------------- |
| LIVENESS_TIMEOUT_MS | `45_000` | src/cli/transports/SSETransport.ts:21 | timeout | const LIVENESS_TIMEOUT_MS = 45_000 |
| POST_MAX_RETRIES | `10` | src/cli/transports/SSETransport.ts:30 | retry | POST retry configuration (matches HybridTransport) |
| POST_BASE_DELAY_MS | `500` | src/cli/transports/SSETransport.ts:31 | timeout | POST retry configuration (matches HybridTransport) |
| POST_MAX_DELAY_MS | `8000` | src/cli/transports/SSETransport.ts:32 | timeout | POST retry configuration (matches HybridTransport) |
| DEFAULT_MAX_BUFFER_SIZE | `1000` | src/cli/transports/WebSocketTransport.ts:22 | buffer-size | const DEFAULT_MAX_BUFFER_SIZE = 1000 |
| DEFAULT_BASE_RECONNECT_DELAY | `1000` | src/cli/transports/WebSocketTransport.ts:23 | timeout | const DEFAULT_BASE_RECONNECT_DELAY = 1000 |
| DEFAULT_MAX_RECONNECT_DELAY | `30000` | src/cli/transports/WebSocketTransport.ts:24 | timeout | const DEFAULT_MAX_RECONNECT_DELAY = 30000 |
| DEFAULT_PING_INTERVAL | `10000` | src/cli/transports/WebSocketTransport.ts:27 | timeout | const DEFAULT_PING_INTERVAL = 10000 |
| DEFAULT_KEEPALIVE_INTERVAL | `300_000 // 5 minutes` | src/cli/transports/WebSocketTransport.ts:28 | timeout | const DEFAULT_KEEPALIVE_INTERVAL = 300_000 // 5 minutes |
| SLEEP_DETECTION_THRESHOLD_MS | `DEFAULT_MAX_RECONNECT_DELAY * 2 // 60s` | src/cli/transports/WebSocketTransport.ts:36 | retry | the reconnection budget and retry — the server will reject with permanent close codes (4001/1002) if the session was reaped during sleep. / |
| inline_setTimeout@add-dir.tsx:26 | `const timer = setTimeout(onDone, 0);` | src/commands/add-dir/add-dir.tsx:26 | timeout | Inline timer magic number |
| CHROME_ROWS | `5;` | src/commands/btw/btw.tsx:33 | ui-rendering | const CHROME_ROWS = 5; |
| OUTER_CHROME_ROWS | `6;` | src/commands/btw/btw.tsx:34 | ui-rendering | const OUTER_CHROME_ROWS = 6; |
| SCROLL_LINES | `3;` | src/commands/btw/btw.tsx:35 | feature-threshold | const SCROLL_LINES = 3; |
| MAX_LOOKBACK | `20;` | src/commands/copy/copy.tsx:25 | feature-threshold | const MAX_LOOKBACK = 20; |
| IDE_CONNECTION_TIMEOUT_MS | `35000;` | src/commands/ide/ide.tsx:507 | timeout | Connection timeout slightly longer than the 30s MCP connection timeout |
| SUMMARIZE_CHUNK_PROMPT | ``Summarize this portion of a Claude Code session transcript. Focus on: 1. What the user asked for 2. What Claude did (tools used, files modified) 3. Any friction or issues 4. The outcome` | src/commands/insights.ts:870 | buffer-size | const SUMMARIZE_CHUNK_PROMPT = `Summarize this portion of a Claude Code session transcript. Focus on: |
| CHUNK_SIZE | `25000` | src/commands/insights.ts:917 | buffer-size | For long transcripts, split into chunks and summarize in parallel |
| OVERLAP_WINDOW_MS | `30 * 60000` | src/commands/insights.ts:1072 | feature-threshold | const OVERLAP_WINDOW_MS = 30 * 60000 |
| inline_setTimeout@insights.ts:2396 | `setTimeout(() => { btn.textContent = 'Copy'; }, 2000);` | src/commands/insights.ts:2396 | timeout | Inline timer magic number |
| inline_setTimeout@insights.ts:2405 | `if (btn) { btn.textContent = 'Copied!'; setTimeout(() => { btn.textContent = 'Copy'; }, 2000); }` | src/commands/insights.ts:2405 | timeout | Inline timer magic number |
| inline_setTimeout@insights.ts:2421 | `setTimeout(() => { btn.textContent = 'Copy All Checked'; btn.classList.remove('copied'); }, 2000);` | src/commands/insights.ts:2421 | timeout | Inline timer magic number |
| META_BATCH_SIZE | `50` | src/commands/insights.ts:2820 | token-management | Phase 2: Load SessionMeta — use cache where available, parse only uncached Read cached metas in parallel batches to avoid blocking the event loop |
| MAX_SESSIONS_TO_LOAD | `200` | src/commands/insights.ts:2821 | token-management | Phase 2: Load SessionMeta — use cache where available, parse only uncached Read cached metas in parallel batches to avoid blocking the event loop |
| LOAD_BATCH_SIZE | `10` | src/commands/insights.ts:2864 | token-management | Load uncached sessions in batches to yield to event loop between batches |
| MAX_FACET_EXTRACTIONS | `50` | src/commands/insights.ts:2932 | token-management | Phase 3: Facet extraction — only for sessions without cached facets |
| CONCURRENCY | `50` | src/commands/insights.ts:2953 | file-handling | Extract facets for sessions that need them (50 concurrent) |
| inline_setTimeout@install-github-app.tsx:277 | `setTimeout(openGitHubAppInstallation, 0);` | src/commands/install-github-app/install-github-app.tsx:277 | timeout | Inline timer magic number |
| inline_setTimeout@install-github-app.tsx:337 | `setTimeout(openGitHubAppInstallation, 0);` | src/commands/install-github-app/install-github-app.tsx:337 | timeout | Inline timer magic number |
| inline_setTimeout@OAuthFlowStep.tsx:111 | `const timer_0 = setTimeout(setShowPastePrompt, 3000, true);` | src/commands/install-github-app/OAuthFlowStep.tsx:111 | timeout | Inline timer magic number |
| inline_setTimeout@OAuthFlowStep.tsx:137 | `const timer2 = setTimeout(onSuccess_0, 1000, accessToken);` | src/commands/install-github-app/OAuthFlowStep.tsx:137 | timeout | Auto-continue after brief delay to show success |
| inline_setTimeout@OAuthFlowStep.tsx:179 | `urlCopiedTimerRef.current = setTimeout(setUrlCopied, 2000, false);` | src/commands/install-github-app/OAuthFlowStep.tsx:179 | timeout | Inline timer magic number |
| inline_setTimeout@install.tsx:184 | `setTimeout(setState, 2000, {` | src/commands/install.tsx:184 | timeout | Still mark as success but show both setup messages and cleanup warnings |
| inline_setTimeout@install.tsx:213 | `setTimeout(onDone, 2000, 'Claude Code installation completed successfully', {` | src/commands/install.tsx:213 | timeout | Give success message time to render before exiting |
| inline_setTimeout@install.tsx:218 | `setTimeout(onDone, 3000, 'Claude Code installation failed', {` | src/commands/install.tsx:218 | timeout | Give error message time to render before exiting |
| inline_setTimeout@ManageMarketplaces.tsx:143 | `setTimeout(applyChanges, 100, newStates);` | src/commands/plugin/ManageMarketplaces.tsx:143 | timeout | Apply the change immediately |
| inline_setTimeout@ManageMarketplaces.tsx:322 | `setTimeout(setViewState, 2000, {` | src/commands/plugin/ManageMarketplaces.tsx:322 | timeout | Otherwise show result and exit to menu |
| DEFAULT_MAX_VISIBLE | `5` | src/commands/plugin/usePagination.ts:3 | ui-rendering | const DEFAULT_MAX_VISIBLE = 5 |
| inline_setTimeout@release-notes.ts:25 | `setTimeout(rej => rej(new Error('Timeout')), 500, reject)` | src/commands/release-notes/release-notes.ts:25 | timeout | Inline timer magic number |
| inline_setTimeout@resume.tsx:50 | `const timer = setTimeout(onDone, 0);` | src/commands/resume/resume.tsx:50 | timeout | Inline timer magic number |
| ULTRAPLAN_TIMEOUT_MS | `30 * 60 * 1000;` | src/commands/ultraplan.tsx:24 | timeout | consider refresh. Multi-agent exploration is slow; 30min timeout. |
| inline_setTimeout@upgrade.tsx:22 | `setTimeout(onDone, 0, 'You are already on the highest Max subscription plan. For additional usage, run /login to switch to an API usage-billed account.');` | src/commands/upgrade/upgrade.tsx:22 | timeout | Inline timer magic number |
| inline_setTimeout@upgrade.tsx:34 | `setTimeout(onDone, 0, 'Failed to open browser. Please visit https://claude.ai/upgrade/max to upgrade.');` | src/commands/upgrade/upgrade.tsx:34 | timeout | Inline timer magic number |
| LANG_HINT_MAX_SHOWS | `2` | src/commands/voice/voice.ts:14 | feature-threshold | const LANG_HINT_MAX_SHOWS = 2 |
| inline_setTimeout@ConsoleOAuthFlow.tsx:106 | `const timer = setTimeout(setOAuthStatus, 1000, oauthStatus.nextState);` | src/components/ConsoleOAuthFlow.tsx:106 | timeout | Retry logic |
| inline_setTimeout@ConsoleOAuthFlow.tsx:150 | `setTimeout(setUrlCopied, 2000, false);` | src/components/ConsoleOAuthFlow.tsx:150 | timeout | Inline timer magic number |
| inline_setTimeout@ConsoleOAuthFlow.tsx:199 | `setTimeout(setShowPastePrompt, 3000, true);` | src/components/ConsoleOAuthFlow.tsx:199 | timeout | Inline timer magic number |
| DEFAULT_VISIBLE | `8;` | src/components/design-system/FuzzyPicker.tsx:63 | ui-rendering | const DEFAULT_VISIBLE = 8; |
| CHROME_ROWS | `10;` | src/components/design-system/FuzzyPicker.tsx:66 | ui-rendering | Pane (paddingTop + Divider) + title + 3 gaps + SearchBox (rounded border = 3 rows) + hints. matchLabel adds +1 when present, accounted for separately. |
| MIN_VISIBLE | `2;` | src/components/design-system/FuzzyPicker.tsx:67 | ui-rendering | Pane (paddingTop + Divider) + title + 3 gaps + SearchBox (rounded border = 3 rows) + hints. matchLabel adds +1 when present, accounted for separately. |
| inline_setTimeout@DesktopHandoff.tsx:94 | `setTimeout(_temp2, 500, onDone);` | src/components/DesktopHandoff.tsx:94 | timeout | Inline timer magic number |
| MAX_VISIBLE_FILES | `5;` | src/components/diff/DiffFileList.tsx:9 | ui-rendering | const MAX_VISIBLE_FILES = 5; |
| MAX_RENDERED_LINES | `10;` | src/components/FallbackToolUseErrorMessage.tsx:11 | ui-rendering | const MAX_RENDERED_LINES = 10; |
| GITHUB_URL_LIMIT | `7250;` | src/components/Feedback.tsx:34 | ui-rendering | This value was determined experimentally by testing the URL length limit |
| DEFAULT_DEBOUNCE_MS | `400` | src/components/FeedbackSurvey/useDebouncedDigitInput.ts:8 | timeout | submissions when users start messages with numbers (e.g., numbered lists). Short enough to feel instant for intentional presses, long enough to cancel when the user types more characters. |
| SURVEY_PROBABILITY | `0.2;` | src/components/FeedbackSurvey/useMemorySurvey.tsx:21 | ui-rendering | const SURVEY_PROBABILITY = 0.2; |
| SURVEY_PROBABILITY | `0.2; // Show survey 20% of the time after compaction` | src/components/FeedbackSurvey/usePostCompactSurvey.tsx:15 | token-management | const SURVEY_PROBABILITY = 0.2; // Show survey 20% of the time after compaction |
| MAX_LINES_TO_RENDER | `10;` | src/components/FileEditToolUseRejectedMessage.tsx:11 | ui-rendering | const MAX_LINES_TO_RENDER = 10; |
| VISIBLE_RESULTS | `12;` | src/components/GlobalSearchDialog.tsx:27 | ui-rendering | const VISIBLE_RESULTS = 12; |
| DEBOUNCE_MS | `100;` | src/components/GlobalSearchDialog.tsx:28 | timeout | const DEBOUNCE_MS = 100; |
| PREVIEW_CONTEXT_LINES | `4;` | src/components/GlobalSearchDialog.tsx:29 | ui-rendering | const PREVIEW_CONTEXT_LINES = 4; |
| MAX_MATCHES_PER_FILE | `10;` | src/components/GlobalSearchDialog.tsx:31 | ui-rendering | rg -m is per-file; we also cap the parsed array to keep memory bounded. |
| MAX_TOTAL_MATCHES | `500;` | src/components/GlobalSearchDialog.tsx:32 | ui-rendering | rg -m is per-file; we also cap the parsed array to keep memory bounded. |
| DEFAULT_WIDTH | `80;` | src/components/HighlightedCode.tsx:17 | ui-rendering | const DEFAULT_WIDTH = 80; |
| HL_CACHE_MAX | `500;` | src/components/HighlightedCode/Fallback.tsx:19 | token-management | Module-level highlight cache — hl.highlight() is the hot cost on virtual- scroll remounts. useMemo doesn't survive unmount→remount. Keyed by hash of code+language to avoid retaining full source strings (#24180 RSS fix). |
| PREVIEW_ROWS | `6;` | src/components/HistorySearchDialog.tsx:18 | ui-rendering | const PREVIEW_ROWS = 6; |
| AGE_WIDTH | `8;` | src/components/HistorySearchDialog.tsx:19 | ui-rendering | const AGE_WIDTH = 8; |
| SETTLED_GREY | `toRGBColor({ r: 153, g: 153, b: 153 });` | src/components/LogoV2/AnimatedAsterisk.tsx:10 | timeout | const SETTLED_GREY = toRGBColor({ |
| CLAWD_HEIGHT | `3;` | src/components/LogoV2/AnimatedClawd.tsx:47 | ui-rendering | const CLAWD_HEIGHT = 3; |
| LEFT_PANEL_MAX_WIDTH | `50;` | src/components/LogoV2/LogoV2.tsx:46 | ui-rendering | const LEFT_PANEL_MAX_WIDTH = 50; |
| MAX_SHOW_COUNT | `6;` | src/components/LogoV2/Opus1mMergeNotice.tsx:9 | ui-rendering | const MAX_SHOW_COUNT = 6; |
| MAX_IMPRESSIONS | `3;` | src/components/LogoV2/OverageCreditUpsell.tsx:10 | ui-rendering | const MAX_IMPRESSIONS = 3; |
| MAX_SHOW_COUNT | `3;` | src/components/LogoV2/VoiceModeNotice.tsx:11 | ui-rendering | const MAX_SHOW_COUNT = 3; |
| WELCOME_V2_WIDTH | `58;` | src/components/LogoV2/WelcomeV2.tsx:5 | ui-rendering | const WELCOME_V2_WIDTH = 58; |
| PARENT_PREFIX_WIDTH | `2; // '▼ ' or '▶ '` | src/components/LogSelector.tsx:68 | ui-rendering | Width of prefixes that TreeSelect will add |
| CHILD_PREFIX_WIDTH | `4; // ' ▸ '` | src/components/LogSelector.tsx:69 | ui-rendering | Width of prefixes that TreeSelect will add |
| DEEP_SEARCH_MAX_MESSAGES | `2000;` | src/components/LogSelector.tsx:72 | ui-rendering | Deep search constants |
| DEEP_SEARCH_CROP_SIZE | `1000;` | src/components/LogSelector.tsx:73 | ui-rendering | Deep search constants |
| DEEP_SEARCH_MAX_TEXT_LENGTH | `50000; // Cap searchable text per session` | src/components/LogSelector.tsx:74 | ui-rendering | Deep search constants |
| FUSE_THRESHOLD | `0.3;` | src/components/LogSelector.tsx:75 | ui-rendering | const FUSE_THRESHOLD = 0.3; |
| DATE_TIE_THRESHOLD_MS | `60 * 1000; // 1 minute - use relevance as tie-breaker within this window` | src/components/LogSelector.tsx:76 | ui-rendering | const DATE_TIE_THRESHOLD_MS = 60 * 1000; // 1 minute - use relevance as tie-breaker within this window |
| inline_setTimeout@LogSelector.tsx:290 | `const timeoutId = setTimeout(setDebouncedDeepSearchQuery, 300, deferredSearchQuery);` | src/components/LogSelector.tsx:290 | timeout | Inline timer magic number |
| inline_setTimeout@LogSelector.tsx:471 | `const timeoutId_0 = setTimeout(_temp5, 0, null, debouncedDeepSearchQuery, setDeepSearchResults, setIsSearching);` | src/components/LogSelector.tsx:471 | timeout | Inline timer magic number |
| TOKEN_CACHE_MAX | `500;` | src/components/Markdown.tsx:22 | token-management | scrolling back to a previously-visible message re-parses. Messages are immutable in history; same content → same tokens. Keyed by hash to avoid retaining full content strings (turn50→turn99 RSS regression, #24180). |
| MIN_COLUMN_WIDTH | `3;` | src/components/MarkdownTable.tsx:18 | ui-rendering | const MIN_COLUMN_WIDTH = 3; |
| MAX_ROW_LINES | `4;` | src/components/MarkdownTable.tsx:25 | timeout | When wrapping would make rows taller than this, vertical (key-value) format provides better readability. / |
| inline_setInterval@ElicitationDialog.tsx:56 | `const timer = setInterval(setFrame, 80, advanceSpinnerFrame);` | src/components/mcp/ElicitationDialog.tsx:56 | timeout | Inline timer magic number |
| inline_setTimeout@ElicitationDialog.tsx:463 | `ta_0.timer = setTimeout(resetTypeahead, 2000, ta_0);` | src/components/mcp/ElicitationDialog.tsx:463 | timeout | Inline timer magic number |
| LINES_PER_FIELD | `3;` | src/components/mcp/ElicitationDialog.tsx:754 | token-management | To generalize: track per-field height (3 for collapsed, N+3 for expanded multi-select) and compute a pixel-budget window instead of a simple item-count window. |
| inline_setTimeout@MCPRemoteServerMenu.tsx:205 | `copyTimeoutRef.current = setTimeout(setUrlCopied, 2000, false);` | src/components/mcp/MCPRemoteServerMenu.tsx:205 | timeout | Inline timer magic number |
| MAX_MESSAGES_TO_SHOW_IN_TRANSCRIPT_MODE | `30;` | src/components/Messages.tsx:276 | ui-rendering | Measured Mar 2026: 538-msg session, 20 slices → −55% plateau RSS. */ |
| MAX_MESSAGES_WITHOUT_VIRTUALIZATION | `200;` | src/components/Messages.tsx:307 | ui-rendering | slice roughly where it was instead of resetting to 0 — which would jump from ~200 rendered messages to the full history, orphaning in-progress badge snapshots in scrollback. |
| MESSAGE_CAP_STEP | `50;` | src/components/Messages.tsx:308 | ui-rendering | jump from ~200 rendered messages to the full history, orphaning in-progress badge snapshots in scrollback. |
| MAX_API_ERROR_CHARS | `1000;` | src/components/messages/AssistantTextMessage.tsx:19 | ui-rendering | const MAX_API_ERROR_CHARS = 1000; |
| MIN_HINT_DISPLAY_MS | `700;` | src/components/messages/CollapsedReadSearchContent.tsx:29 | ui-rendering | Hold each ⤿ hint for a minimum duration so fast-completing tool calls (bash commands, file reads, search patterns) are actually readable instead of flickering past in a single frame. |
| MAX_API_ERROR_CHARS | `1000;` | src/components/messages/SystemAPIErrorMessage.tsx:10 | ui-rendering | const MAX_API_ERROR_CHARS = 1000; |
| TRUNCATE_AT | `60;` | src/components/messages/UserChannelMessage.tsx:25 | ui-rendering | const TRUNCATE_AT = 60; |
| MAX_DISPLAY_CHARS | `10_000;` | src/components/messages/UserPromptMessage.tsx:28 | ui-rendering | avoids this via <Static> (print-and-forget to terminal scrollback). Head+tail because `{ cat file; echo prompt; } \| claude` puts the user's actual question at the end. |
| TRUNCATE_HEAD_CHARS | `2_500;` | src/components/messages/UserPromptMessage.tsx:29 | ui-rendering | Head+tail because `{ cat file; echo prompt; } \| claude` puts the user's actual question at the end. |
| TRUNCATE_TAIL_CHARS | `2_500;` | src/components/messages/UserPromptMessage.tsx:30 | ui-rendering | actual question at the end. |
| MAX_VISIBLE_MESSAGES | `7;` | src/components/MessageSelector.tsx:45 | ui-rendering | const MAX_VISIBLE_MESSAGES = 7; |
| MIN_CONTENT_HEIGHT | `12;` | src/components/permissions/AskUserQuestionPermissionRequest/AskUserQuestionPermissionRequest.tsx:26 | ui-rendering | const MIN_CONTENT_HEIGHT = 12; |
| MIN_CONTENT_WIDTH | `40;` | src/components/permissions/AskUserQuestionPermissionRequest/AskUserQuestionPermissionRequest.tsx:27 | ui-rendering | const MIN_CONTENT_WIDTH = 40; |
| LEFT_PANEL_WIDTH | `30;` | src/components/permissions/AskUserQuestionPermissionRequest/PreviewQuestionView.tsx:233 | ui-rendering | The right panel's available width is terminal minus the left panel and gap. |
| inline_setTimeout@ExitPlanModePermissionRequest.tsx:222 | `const timer = setTimeout(setShowSaveMessage, 5000, false);` | src/components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx:222 | timeout | Auto-hide save message after 5 seconds |
| TRUNCATION_THRESHOLD | `10000 // Characters before we truncate` | src/components/PromptInput/inputPaste.ts:4 | ui-rendering | const TRUNCATION_THRESHOLD = 10000 // Characters before we truncate |
| PREVIEW_LENGTH | `1000 // Characters to show at start and end` | src/components/PromptInput/inputPaste.ts:5 | ui-rendering | const PREVIEW_LENGTH = 1000 // Characters to show at start and end |
| FOOTER_TEMPORARY_STATUS_TIMEOUT | `5000;` | src/components/PromptInput/Notifications.tsx:40 | timeout | export const FOOTER_TEMPORARY_STATUS_TIMEOUT = 5000; |
| PROMPT_FOOTER_LINES | `5;` | src/components/PromptInput/PromptInput.tsx:192 | ui-rendering | Bottom slot has maxHeight="50%"; reserve lines for footer, border, status. |
| MIN_INPUT_VIEWPORT_LINES | `3;` | src/components/PromptInput/PromptInput.tsx:193 | ui-rendering | Bottom slot has maxHeight="50%"; reserve lines for footer, border, status. |
| MAX_VOICE_HINT_SHOWS | `3;` | src/components/PromptInput/PromptInputFooterLeftSide.tsx:51 | ui-rendering | const MAX_VOICE_HINT_SHOWS = 3; |
| inline_setInterval@PromptInputFooterLeftSide.tsx:91 | `const interval = setInterval(update, 1000);` | src/components/PromptInput/PromptInputFooterLeftSide.tsx:91 | timeout | Inline timer magic number |
| OVERLAY_MAX_ITEMS | `5;` | src/components/PromptInput/PromptInputFooterSuggestions.tsx:18 | ui-rendering | export const OVERLAY_MAX_ITEMS = 5; |
| MAX_VISIBLE_NOTIFICATIONS | `3;` | src/components/PromptInput/PromptInputQueuedCommands.tsx:30 | buffer-size | Maximum number of task notification lines to show |
| inline_setTimeout@SandboxPromptFooterHint.tsx:30 | `timerRef.current = setTimeout(setRecentViolationCount, 5000, 0);` | src/components/PromptInput/SandboxPromptFooterHint.tsx:30 | timeout | Inline timer magic number |
| NUM_TIMES_QUEUE_HINT_SHOWN | `3` | src/components/PromptInput/usePromptInputPlaceholder.ts:22 | buffer-size | const NUM_TIMES_QUEUE_HINT_SHOWN = 3 |
| MAX_TEAMMATE_NAME_LENGTH | `20` | src/components/PromptInput/usePromptInputPlaceholder.ts:23 | ui-rendering | const MAX_TEAMMATE_NAME_LENGTH = 20 |
| VISIBLE_RESULTS | `8;` | src/components/QuickOpenDialog.tsx:21 | ui-rendering | const VISIBLE_RESULTS = 8; |
| PREVIEW_LINES | `20;` | src/components/QuickOpenDialog.tsx:22 | ui-rendering | const PREVIEW_LINES = 20; |
| WHEEL_ACCEL_WINDOW_MS | `40;` | src/components/ScrollKeybindingHandler.tsx:47 | ui-rendering | iTerm2 "faster scroll" similar) — base=1 is correct there. Others send 1 event/notch — users on those can set CLAUDE_CODE_SCROLL_SPEED=3 to match vim/nvim/opencode app-side defaults. We can't detect which, so knob it. |
| WHEEL_ACCEL_MAX | `6;` | src/components/ScrollKeybindingHandler.tsx:49 | ui-rendering | vim/nvim/opencode app-side defaults. We can't detect which, so knob it. |
| WHEEL_BOUNCE_GAP_MAX_MS | `200; // flip-back must arrive within this` | src/components/ScrollKeybindingHandler.tsx:65 | timeout | threshold needed, large gaps just have m≈0 → mult→1. Wheel mode is STICKY: once a bounce confirms it's a mouse, the decay curve applies until an idle gap or trackpad-flick-burst signals a possible device switch. |
| WHEEL_MODE_CAP | `15;` | src/components/ScrollKeybindingHandler.tsx:69 | ui-rendering | Mouse is ~9 events/sec vs VS Code's ~30 — STEP is 3× xterm.js's 5 to compensate. At gap=100ms (m≈0.63): one click gives 1+15*0.63≈10.5. |
| WHEEL_DECAY_CAP_SLOW | `3; // gap ≥ GAP_MS: precision` | src/components/ScrollKeybindingHandler.tsx:100 | ui-rendering | Cap boundary: slow events (≥GAP_MS) cap low for short smooth drains; fast events cap higher for throughput (adaptive drain handles backlog). |
| WHEEL_DECAY_CAP_FAST | `6; // gap < GAP_MS: throughput` | src/components/ScrollKeybindingHandler.tsx:101 | ui-rendering | fast events cap higher for throughput (adaptive drain handles backlog). |
| AUTOSCROLL_LINES | `2;` | src/components/ScrollKeybindingHandler.tsx:344 | timeout | Drag-to-scroll: when dragging past the viewport edge, scroll by this many rows every AUTOSCROLL_INTERVAL_MS. Mode 1002 mouse tracking only fires on cell change, so a timer is needed to continue scrolling while stationary. |
| AUTOSCROLL_INTERVAL_MS | `50;` | src/components/ScrollKeybindingHandler.tsx:345 | timeout | rows every AUTOSCROLL_INTERVAL_MS. Mode 1002 mouse tracking only fires on cell change, so a timer is needed to continue scrolling while stationary. |
| AUTOSCROLL_MAX_TICKS | `200; // 10s @ 50ms` | src/components/ScrollKeybindingHandler.tsx:351 | ui-rendering | pointer and drop the release), isDragging stays true and the timer would run until a scroll boundary. Cap bounds the damage; any new drag motion event restarts the count via check()→start(). |
| MAX_JSON_FORMAT_LENGTH | `10_000;` | src/components/shell/OutputLine.tsx:32 | ui-rendering | const MAX_JSON_FORMAT_LENGTH = 10_000; |
| inline_setTimeout@Spinner.tsx:147 | `clearStatusTimer = setTimeout(setThinkingStatus, 2000, null);` | src/components/Spinner.tsx:147 | timeout | Clear after 2s |
| SEP_WIDTH | `stringWidth(' · ');` | src/components/Spinner/SpinnerAnimationRow.tsx:17 | ui-rendering | const SEP_WIDTH = stringWidth(' · '); |
| THINKING_BARE_WIDTH | `stringWidth('thinking');` | src/components/Spinner/SpinnerAnimationRow.tsx:18 | ui-rendering | const THINKING_BARE_WIDTH = stringWidth('thinking'); |
| THINKING_DELAY_MS | `3000;` | src/components/Spinner/SpinnerAnimationRow.tsx:34 | timeout | const THINKING_DELAY_MS = 3000; |
| RGB_CACHE | `new Map<string, RGBColorType \| null>()` | src/components/Spinner/utils.ts:68 | token-management | const RGB_CACHE = new Map<string, RGBColorType \| null>() |
| inline_setTimeout@Stats.tsx:1066 | `setTimeout(setStatus, 2000, null);` | src/components/Stats.tsx:1066 | timeout | Clear status after 2 seconds |
| COL1_LABEL_WIDTH | `18;` | src/components/Stats.tsx:1103 | ui-rendering | Two-column helper with fixed spacing Column 1: label (18 chars) + value + padding to reach col 2 Column 2 starts at character position 40 |
| COL2_LABEL_WIDTH | `18;` | src/components/Stats.tsx:1105 | ui-rendering | Column 2 starts at character position 40 |
| RENDER_CACHE | `new WeakMap<StructuredPatchHunk, Map<string, CachedRender>>();` | src/components/StructuredDiff.tsx:41 | token-management | const RENDER_CACHE = new WeakMap<StructuredPatchHunk, Map<string, CachedRender>>(); |
| CHANGE_THRESHOLD | `0.4;` | src/components/StructuredDiff/Fallback.tsx:80 | ui-rendering | Threshold for when we show a full-line diff instead of word-level diffing |
| HASH_PREFIX_LENGTH | `1; // "#" prefix for non-All tabs` | src/components/TagTabs.tsx:9 | ui-rendering | Constants for width calculations - derived from actual rendered strings |
| MAX_OVERFLOW_DIGITS | `2; // Assume max 99 hidden tabs for width calculation` | src/components/TagTabs.tsx:14 | ui-rendering | const MAX_OVERFLOW_DIGITS = 2; // Assume max 99 hidden tabs for width calculation |
| LEFT_ARROW_WIDTH | `LEFT_ARROW_PREFIX.length + MAX_OVERFLOW_DIGITS + 1; // "← NN " with gap` | src/components/TagTabs.tsx:17 | ui-rendering | Computed widths |
| RIGHT_HINT_WIDTH_WITH_COUNT | `RIGHT_HINT_WITH_COUNT_PREFIX.length + MAX_OVERFLOW_DIGITS + RIGHT_HINT_SUFFIX.length; // "→NN (tab to cycle)"` | src/components/TagTabs.tsx:18 | ui-rendering | Computed widths |
| RIGHT_HINT_WIDTH_NO_COUNT | `RIGHT_HINT_NO_COUNT.length;` | src/components/TagTabs.tsx:19 | ui-rendering | Computed widths |
| RECENT_COMPLETED_TTL_MS | `30_000;` | src/components/TaskListV2.tsx:21 | timeout | const RECENT_COMPLETED_TTL_MS = 30_000; |
| inline_setTimeout@TaskListV2.tsx:83 | `const timer = setTimeout(forceUpdate_0 => forceUpdate_0((n: number) => n + 1), earliestExpiry - currentNow, forceUpdate);` | src/components/TaskListV2.tsx:83 | timeout | Inline timer magic number |
| VISIBLE_TURNS | `6;` | src/components/tasks/DreamDetailDialog.tsx:21 | ui-rendering | How many recent turns to render. Earlier turns collapse to a count. |
| inline_setInterval@ShellDetailDialog.tsx:76 | `const timer = setInterval(_temp, 1000, setOutputPromise, shell);` | src/components/tasks/ShellDetailDialog.tsx:76 | timeout | Inline timer magic number |
| CURSOR_WAVEFORM_WIDTH | `1;` | src/components/TextInput.tsx:19 | ui-rendering | Mini waveform cursor width |
| SILENCE_THRESHOLD | `0.15;` | src/components/TextInput.tsx:33 | ui-rendering | Raw audio level threshold (pre-boost) below which the cursor is grey. computeLevel returns sqrt(rms/2000), so ambient mic noise typically sits at 0.05-0.15. Speech starts around 0.2+. |
| HEADROOM | `3;` | src/components/VirtualMessageList.tsx:15 | buffer-size | Rows of breathing room above the target when we scrollTo. |
| STICKY_TEXT_CAP | `500;` | src/components/VirtualMessageList.tsx:43 | ui-rendering | 2 rows via overflow:hidden — this just bounds the React prop size. */ |
| CHUNK | `500;` | src/components/VirtualMessageList.tsx:800 | buffer-size | const CHUNK = 500; |
| API_IMAGE_MAX_BASE64_SIZE | `5 * 1024 * 1024 // 5 MB` | src/constants/apiLimits.ts:22 | feature-threshold | The API rejects images where the base64 string length exceeds this value. Note: This is the base64 length, NOT raw bytes. Base64 increases size by ~33%. / |
| IMAGE_TARGET_RAW_SIZE | `(API_IMAGE_MAX_BASE64_SIZE * 3) / 4 // 3.75 MB` | src/constants/apiLimits.ts:29 | feature-threshold | Base64 encoding increases size by 4/3, so we derive the max raw size: raw_size * 4/3 = base64_size → raw_size = base64_size * 3/4 / |
| IMAGE_MAX_WIDTH | `2000` | src/constants/apiLimits.ts:42 | ui-rendering | The API_IMAGE_MAX_BASE64_SIZE (5MB) is the actual hard limit that causes API errors if exceeded. / |
| IMAGE_MAX_HEIGHT | `2000` | src/constants/apiLimits.ts:43 | ui-rendering | API errors if exceeded. / |
| PDF_TARGET_RAW_SIZE | `20 * 1024 * 1024 // 20 MB` | src/constants/apiLimits.ts:54 | feature-threshold | The API has a 32MB total request size limit. Base64 encoding increases size by ~33% (4/3), so 20MB raw → ~27MB base64, leaving room for conversation context. / |
| API_PDF_MAX_PAGES | `100` | src/constants/apiLimits.ts:59 | feature-threshold | Maximum number of pages in a PDF accepted by the API. / |
| PDF_EXTRACT_SIZE_THRESHOLD | `3 * 1024 * 1024 // 3 MB` | src/constants/apiLimits.ts:66 | feature-threshold | instead of being sent as base64 document blocks. This applies to first-party API only; non-first-party always uses extraction. / |
| PDF_MAX_EXTRACT_SIZE | `100 * 1024 * 1024 // 100 MB` | src/constants/apiLimits.ts:72 | file-handling | Maximum PDF file size for the page extraction path. PDFs larger than this are rejected to avoid processing extremely large files. / |
| PDF_MAX_PAGES_PER_READ | `20` | src/constants/apiLimits.ts:77 | feature-threshold | Max pages the Read tool will extract in a single call with the pages parameter. / |
| PDF_AT_MENTION_INLINE_THRESHOLD | `10` | src/constants/apiLimits.ts:83 | feature-threshold | PDFs with more pages than this get the reference treatment on @ mention instead of being inlined into context. / |
| API_MAX_MEDIA_PER_REQUEST | `100` | src/constants/apiLimits.ts:94 | feature-threshold | The API rejects requests exceeding this limit with a confusing error. We validate client-side to provide a clear error message. / |
| CLAUDE_CODE_20250219_BETA_HEADER | `'claude-code-20250219'` | src/constants/betas.ts:3 | feature-threshold | export const CLAUDE_CODE_20250219_BETA_HEADER = 'claude-code-20250219' |
| CONTEXT_1M_BETA_HEADER | `'context-1m-2025-08-07'` | src/constants/betas.ts:6 | feature-threshold | export const CONTEXT_1M_BETA_HEADER = 'context-1m-2025-08-07' |
| CONTEXT_MANAGEMENT_BETA_HEADER | `'context-management-2025-06-27'` | src/constants/betas.ts:7 | feature-threshold | export const CONTEXT_MANAGEMENT_BETA_HEADER = 'context-management-2025-06-27' |
| STRUCTURED_OUTPUTS_BETA_HEADER | `'structured-outputs-2025-12-15'` | src/constants/betas.ts:8 | feature-threshold | export const STRUCTURED_OUTPUTS_BETA_HEADER = 'structured-outputs-2025-12-15' |
| WEB_SEARCH_BETA_HEADER | `'web-search-2025-03-05'` | src/constants/betas.ts:9 | feature-threshold | export const WEB_SEARCH_BETA_HEADER = 'web-search-2025-03-05' |
| TOOL_SEARCH_BETA_HEADER_1P | `'advanced-tool-use-2025-11-20'` | src/constants/betas.ts:13 | feature-threshold | Tool search beta headers differ by provider: - Claude API / Foundry: advanced-tool-use-2025-11-20 - Vertex AI / Bedrock: tool-search-tool-2025-10-19 |
| TOOL_SEARCH_BETA_HEADER_3P | `'tool-search-tool-2025-10-19'` | src/constants/betas.ts:14 | feature-threshold | - Claude API / Foundry: advanced-tool-use-2025-11-20 - Vertex AI / Bedrock: tool-search-tool-2025-10-19 |
| EFFORT_BETA_HEADER | `'effort-2025-11-24'` | src/constants/betas.ts:15 | feature-threshold | - Vertex AI / Bedrock: tool-search-tool-2025-10-19 |
| TASK_BUDGETS_BETA_HEADER | `'task-budgets-2026-03-13'` | src/constants/betas.ts:16 | token-management | export const TASK_BUDGETS_BETA_HEADER = 'task-budgets-2026-03-13' |
| FAST_MODE_BETA_HEADER | `'fast-mode-2026-02-01'` | src/constants/betas.ts:19 | feature-threshold | export const FAST_MODE_BETA_HEADER = 'fast-mode-2026-02-01' |
| REDACT_THINKING_BETA_HEADER | `'redact-thinking-2026-02-12'` | src/constants/betas.ts:20 | ui-rendering | export const REDACT_THINKING_BETA_HEADER = 'redact-thinking-2026-02-12' |
| ADVISOR_BETA_HEADER | `'advisor-tool-2026-03-01'` | src/constants/betas.ts:31 | feature-threshold | export const ADVISOR_BETA_HEADER = 'advisor-tool-2026-03-01' |
| E_TOOL_USE_SUMMARY_GENERATION_FAILED | `344` | src/constants/errorIds.ts:15 | feature-threshold | Next ID: 346 / |
| EFFORT_MAX | `'◉' // \u25c9 - effort level: max (Opus 4.6 only)` | src/constants/figures.ts:13 | feature-threshold | export const EFFORT_MAX = '◉' // \u25c9 - effort level: max (Opus 4.6 only) |
| BINARY_CHECK_SIZE | `8192` | src/constants/files.ts:125 | file-handling | Number of bytes to read for binary content detection. / |
| OAUTH_BETA_HEADER | `'oauth-2025-04-20' as const` | src/constants/oauth.ts:36 | security | export const OAUTH_BETA_HEADER = 'oauth-2025-04-20' as const |
| EXPLANATORY_FEATURE_PROMPT | `` ## Insights In order to encourage learning, before and after writing code, always provide brief educational explanations about implementation choices using (with backticks): "\`${figures.star} Insight ─────────────────` | src/constants/outputStyles.ts:30 | feature-threshold | Used in both the Explanatory and Learning modes |
| CLAUDE_AI_LOCAL_BASE_URL | `'http://localhost:4000'` | src/constants/product.ts:6 | file-handling | Claude Code Remote session URLs |
| FRONTIER_MODEL_NAME | `'Claude Opus 4.6'` | src/constants/prompts.ts:118 | feature-threshold | @[MODEL LAUNCH]: Update the latest frontier model. |
| CLAUDE_4_5_OR_4_6_MODEL_IDS | `{ opus: 'claude-opus-4-6', sonnet: 'claude-sonnet-4-6', haiku: 'claude-haiku-4-5-20251001', }` | src/constants/prompts.ts:121 | feature-threshold | @[MODEL LAUNCH]: Update the model family IDs below to the latest in each tier. |
| DEFAULT_MAX_RESULT_SIZE_CHARS | `50_000` | src/constants/toolLimits.ts:13 | feature-threshold | Individual tools may declare a lower maxResultSizeChars, but this constant acts as a system-wide cap regardless of what tools declare. / |
| MAX_TOOL_RESULT_TOKENS | `100_000` | src/constants/toolLimits.ts:22 | token-management | This is approximately 400KB of text (assuming ~4 bytes per token). / |
| BYTES_PER_TOKEN | `4` | src/constants/toolLimits.ts:28 | token-management | Bytes per token estimate for calculating token count from byte size. This is a conservative estimate - actual token count may vary. / |
| MAX_TOOL_RESULT_BYTES | `MAX_TOOL_RESULT_TOKENS * BYTES_PER_TOKEN` | src/constants/toolLimits.ts:33 | token-management | Maximum size for tool results in bytes (derived from token limit). / |
| MAX_TOOL_RESULTS_PER_MESSAGE_CHARS | `200_000` | src/constants/toolLimits.ts:49 | token-management | Overridable at runtime via GrowthBook flag tengu_hawthorn_window — see getPerMessageBudgetLimit() in toolResultStorage.ts. / |
| TOOL_SUMMARY_MAX_LENGTH | `50` | src/constants/toolLimits.ts:56 | ui-rendering | Used by getToolUseSummary() implementations to truncate long inputs for display in grouped agent rendering. / |
| TERMINAL_OUTPUT_TAGS | `[ BASH_INPUT_TAG, BASH_STDOUT_TAG, BASH_STDERR_TAG, LOCAL_COMMAND_STDOUT_TAG, LOCAL_COMMAND_STDERR_TAG, LOCAL_COMMAND_CAVEAT_TAG, ] as const` | src/constants/xml.ts:16 | feature-threshold | All terminal-related tags that indicate a message is terminal output, not a user prompt |
| MAX_STATUS_CHARS | `2000` | src/context.ts:20 | feature-threshold | const MAX_STATUS_CHARS = 2000 |
| DEFAULT_TIMEOUT_MS | `8000;` | src/context/notifications.tsx:34 | timeout | const DEFAULT_TIMEOUT_MS = 8000; |
| RESERVOIR_SIZE | `1024;` | src/context/stats.tsx:20 | feature-threshold | const RESERVOIR_SIZE = 1024; |
| READ_FILE_STATE_CACHE_SIZE | `100` | src/entrypoints/mcp.ts:42 | token-management | Use size-limited LRU cache for readFileState to prevent unbounded memory growth 100 files and 25MB limit should be sufficient for MCP server operations |
| MAX_HISTORY_ITEMS | `100` | src/history.ts:19 | file-handling | const MAX_HISTORY_ITEMS = 100 |
| MAX_PASTED_CONTENT_LENGTH | `1024` | src/history.ts:20 | buffer-size | const MAX_PASTED_CONTENT_LENGTH = 1024 |
| MAX_SUGGESTIONS | `15` | src/hooks/fileSuggestions.ts:617 | file-handling | Find matching files and folders for a given query using the TS file index / |
| REFRESH_THROTTLE_MS | `5_000` | src/hooks/fileSuggestions.ts:635 | timeout | has actually changed. This prevents every keystroke from spawning git ls-files and rebuilding the nucleo index. / |
| MAX_SHOW_COUNT | `3;` | src/hooks/notifs/useCanSwitchToExistingSubscription.tsx:8 | feature-threshold | const MAX_SHOW_COUNT = 3; |
| MAX_IDE_HINT_SHOW_COUNT | `5;` | src/hooks/notifs/useIDEStatusIndicator.tsx:11 | feature-threshold | const MAX_IDE_HINT_SHOW_COUNT = 5; |
| inline_setTimeout@useIDEStatusIndicator.tsx:61 | `const timeoutId = setTimeout(_temp2, 3000, hasShownHintRef, addNotification);` | src/hooks/notifs/useIDEStatusIndicator.tsx:61 | timeout | Inline timer magic number |
| LSP_POLL_INTERVAL_MS | `5000;` | src/hooks/notifs/useLspInitializationNotification.tsx:11 | timeout | const LSP_POLL_INTERVAL_MS = 5000; |
| MAX_UNIFIED_SUGGESTIONS | `15` | src/hooks/unifiedSuggestions.ts:70 | feature-threshold | const MAX_UNIFIED_SUGGESTIONS = 15 |
| DESCRIPTION_MAX_LENGTH | `60` | src/hooks/unifiedSuggestions.ts:71 | feature-threshold | const DESCRIPTION_MAX_LENGTH = 60 |
| HISTORY_CHUNK_SIZE | `10;` | src/hooks/useArrowKeyHistory.tsx:13 | buffer-size | Load history entries in chunks to reduce disk reads on rapid keypresses |
| PREFETCH_THRESHOLD_ROWS | `40` | src/hooks/useAssistantHistory.ts:38 | ui-rendering | const PREFETCH_THRESHOLD_ROWS = 40 |
| MAX_FILL_PAGES | `10` | src/hooks/useAssistantHistory.ts:42 | ui-rendering | events convert to zero visible messages (everything filtered). */ |
| BLUR_DELAY_MS | `5 * 60_000` | src/hooks/useAwaySummary.ts:12 | timeout | const BLUR_DELAY_MS = 5 * 60_000 |
| BLINK_INTERVAL_MS | `600` | src/hooks/useBlink.ts:3 | timeout | const BLINK_INTERVAL_MS = 600 |
| KILL_AGENTS_CONFIRM_WINDOW_MS | `3000` | src/hooks/useCancelRequest.ts:38 | feature-threshold | const KILL_AGENTS_CONFIRM_WINDOW_MS = 3000 |
| inline_setTimeout@useCanUseTool.tsx:193 | `return setTimeout(res, 2000, {` | src/hooks/useCanUseTool.tsx:193 | timeout | Inline timer magic number |
| FOCUS_CHECK_DEBOUNCE_MS | `1000` | src/hooks/useClipboardImageHint.ts:8 | timeout | Small debounce to batch rapid focus changes |
| MAX_LINES_PER_FILE | `400` | src/hooks/useDiffData.ts:10 | file-handling | const MAX_LINES_PER_FILE = 400 |
| DOUBLE_PRESS_TIMEOUT_MS | `800` | src/hooks/useDoublePress.ts:6 | timeout | export const DOUBLE_PRESS_TIMEOUT_MS = 800 |
| INBOX_POLL_INTERVAL_MS | `1000` | src/hooks/useInboxPoller.ts:107 | timeout | const INBOX_POLL_INTERVAL_MS = 1000 |
| MIN_SUBMIT_COUNT | `3` | src/hooks/useIssueFlagBanner.ts:89 | feature-threshold | const MIN_SUBMIT_COUNT = 3 |
| TIMEOUT_THRESHOLD_MS | `28_000;` | src/hooks/useLspPluginRecommendation.tsx:29 | timeout | Threshold for detecting timeout vs explicit dismiss (ms) Menu auto-dismisses at 30s, so anything over 28s is likely timeout |
| HIGH_MEMORY_THRESHOLD | `1.5 * 1024 * 1024 * 1024 // 1.5GB in bytes` | src/hooks/useMemoryUsage.ts:11 | feature-threshold | const HIGH_MEMORY_THRESHOLD = 1.5 * 1024 * 1024 * 1024 // 1.5GB in bytes |
| CRITICAL_MEMORY_THRESHOLD | `2.5 * 1024 * 1024 * 1024 // 2.5GB in bytes` | src/hooks/useMemoryUsage.ts:12 | feature-threshold | const CRITICAL_MEMORY_THRESHOLD = 2.5 * 1024 * 1024 * 1024 // 2.5GB in bytes |
| DEFAULT_INTERACTION_THRESHOLD_MS | `6000` | src/hooks/useNotifyAfterTimeout.ts:9 | timeout | The time threshold in milliseconds for considering an interaction "recent" (6 seconds) |
| CLIPBOARD_CHECK_DEBOUNCE_MS | `50` | src/hooks/usePasteHandler.ts:15 | timeout | const CLIPBOARD_CHECK_DEBOUNCE_MS = 50 |
| PASTE_COMPLETION_TIMEOUT_MS | `100` | src/hooks/usePasteHandler.ts:16 | timeout | const PASTE_COMPLETION_TIMEOUT_MS = 100 |
| POLL_INTERVAL_MS | `60_000` | src/hooks/usePrStatus.ts:5 | timeout | const POLL_INTERVAL_MS = 60_000 |
| SLOW_GH_THRESHOLD_MS | `4_000` | src/hooks/usePrStatus.ts:6 | feature-threshold | const SLOW_GH_THRESHOLD_MS = 4_000 |
| RESPONSE_TIMEOUT_MS | `60000 // 60 seconds` | src/hooks/useRemoteSession.ts:37 | timeout | How long to wait for a response before showing a warning |
| COMPACTION_TIMEOUT_MS | `180000 // 3 minutes` | src/hooks/useRemoteSession.ts:41 | timeout | Extended timeout during compaction — compact API calls take 5-30s and block other SDK messages, so the normal 60s timeout isn't enough when compaction itself runs close to the edge. |
| MAX_CONSECUTIVE_INIT_FAILURES | `3;` | src/hooks/useReplBridge.tsx:40 | feature-threshold | top stuck client generated 2,879 × 401/day alone (17% of all 401s on the route). / |
| POLL_INTERVAL_MS | `500` | src/hooks/useSwarmPermissionPoller.ts:28 | timeout | const POLL_INTERVAL_MS = 500 |
| DEBOUNCE_MS | `1000` | src/hooks/useTaskListWatcher.ts:14 | timeout | const DEBOUNCE_MS = 1000 |
| HIDE_DELAY_MS | `5000` | src/hooks/useTasksV2.ts:16 | timeout | const HIDE_DELAY_MS = 5000 |
| DEBOUNCE_MS | `50` | src/hooks/useTasksV2.ts:17 | timeout | const DEBOUNCE_MS = 50 |
| FALLBACK_POLL_MS | `5000 // Fallback in case fs.watch misses events` | src/hooks/useTasksV2.ts:18 | timeout | const FALLBACK_POLL_MS = 5000 // Fallback in case fs.watch misses events |
| OVERSCAN_ROWS | `80` | src/hooks/useVirtualScroll.ts:24 | ui-rendering | Extra rows rendered above and below the viewport. Generous because real heights can be 10x the estimate for long tool results. / |
| PESSIMISTIC_HEIGHT | `1` | src/hooks/useVirtualScroll.ts:45 | ui-rendering | regardless of how small items actually are — at the cost of over-mounting when items are larger (which is fine, overscan absorbs it). / |
| MAX_MOUNTED_ITEMS | `300` | src/hooks/useVirtualScroll.ts:47 | feature-threshold | / |
| MAX_SPAN_ROWS | `viewportH * 3` | src/hooks/useVirtualScroll.ts:385 | ui-rendering | clamp (setClampBounds) shows edge-of-mounted during catch-up so there's no blank screen — scroll reaches target over a few frames instead of freezing once for seconds. |
| RELEASE_TIMEOUT_MS | `200` | src/hooks/useVoice.ts:160 | timeout | Gap (ms) between auto-repeat key events that signals key release. Terminal auto-repeat typically fires every 30-80ms; 200ms comfortably covers jitter while still feeling responsive. |
| FOCUS_SILENCE_TIMEOUT_MS | `5_000` | src/hooks/useVoice.ts:177 | timeout | How long (ms) to keep a focus-mode session alive without any speech before tearing it down to free the WebSocket connection. Re-arms on the next focus cycle (blur → refocus). |
| HOLD_THRESHOLD | `5;` | src/hooks/useVoiceIntegration.tsx:51 | timeout | Number of rapid consecutive key events required to activate voice. Only applies to bare-char bindings (space, v, etc.) where a single press could be normal typing. Modifier combos activate on the first press. |
| WARMUP_THRESHOLD | `2;` | src/hooks/useVoiceIntegration.tsx:54 | analytics | Number of rapid key events to start showing warmup feedback. |
| CURSOR_HOME_WINDOWS | `csi(0, 'f')` | src/ink/clearTerminal.ts:14 | ui-rendering | HVP (Horizontal Vertical Position) - legacy Windows cursor home |
| MULTI_CLICK_TIMEOUT_MS | `500;` | src/ink/components/App.tsx:92 | timeout | Multi-click detection thresholds. 500ms is the macOS default; a small position tolerance allows for trackpad jitter between clicks. |
| inline_setTimeout@Button.tsx:102 | `activeTimer.current = setTimeout(_temp, 100, setIsActive);` | src/ink/components/Button.tsx:102 | timeout | Inline timer magic number |
| BLURRED_TICK_INTERVAL_MS | `FRAME_INTERVAL_MS * 2;` | src/ink/components/ClockContext.tsx:70 | timeout | const BLURRED_TICK_INTERVAL_MS = FRAME_INTERVAL_MS * 2; |
| FRAME_INTERVAL_MS | `16` | src/ink/constants.ts:2 | timeout | Shared frame interval for render throttling and animations (~60fps) |
| MAX_FOCUS_STACK | `32` | src/ink/focus.ts:4 | ui-rendering | const MAX_FOCUS_STACK = 32 |
| inline_setTimeout@ink.tsx:758 | `this.drainTimer = setTimeout(() => this.onRender(), FRAME_INTERVAL_MS >> 2);` | src/ink/ink.tsx:758 | timeout | quarter interval (~250fps, setTimeout practical floor) for max scroll speed. Regular renders stay at FRAME_INTERVAL_MS via the throttle. |
| MAX_CACHE_SIZE | `4096` | src/ink/line-width-cache.ts:8 | token-management | unchanged lines on every token (~50x reduction in stringWidth calls). |
| SCROLL_MIN_PER_FRAME | `4` | src/ink/render-node-to-output.ts:110 | ui-rendering | Minimum rows applied per frame. Above this, drain is proportional (~3/4 of remaining) so big bursts catch up in log₄ frames while the tail decelerates smoothly. Hard cap is innerHeight-1 so DECSTBM hint fires. |
| SCROLL_INSTANT_THRESHOLD | `5 // ≤ this: drain all at once` | src/ink/render-node-to-output.ts:117 | ui-rendering | instant (click → visible jump → done), not micro-stutter 1-row frames. Higher pending drains at a small fixed step so fast-scroll animation stays smooth (no big jumps). Pending >MAX snaps excess. |
| SCROLL_MAX_PENDING | `30 // snap excess beyond this` | src/ink/render-node-to-output.ts:121 | ui-rendering | const SCROLL_MAX_PENDING = 30 // snap excess beyond this |
| VISIBLE_ON_SPACE | `new Set([ '\x1b[49m', // background color '\x1b[27m', // inverse '\x1b[24m', // underline '\x1b[29m', // strikethrough '\x1b[55m', // overline ])` | src/ink/screen.ts:263 | ui-rendering | endCodes that produce visible effects on space characters |
| WIDTH_MASK | `3 // 2 bits` | src/ink/screen.ts:339 | ui-rendering | const WIDTH_MASK = 3 // 2 bits |
| BUN_STRING_WIDTH_OPTS | `{ ambiguousIsNarrow: true } as const` | src/ink/stringWidth.ts:218 | ui-rendering | const BUN_STRING_WIDTH_OPTS = { ambiguousIsNarrow: true } as const |
| ADDITIONAL_HYPERLINK_TERMINALS | `[ 'ghostty', 'Hyper', 'kitty', 'alacritty', 'iTerm.app', 'iTerm2', ]` | src/ink/supports-hyperlinks.ts:5 | ui-rendering | Additional terminals that support OSC 8 hyperlinks but aren't detected by supports-hyperlinks. Checked against both TERM_PROGRAM and LC_TERMINAL (the latter is preserved inside tmux). |
| DEFAULT_TAB_INTERVAL | `8` | src/ink/tabstops.ts:7 | timeout | const DEFAULT_TAB_INTERVAL = 8 |
| EXTENDED_KEYS_TERMINALS | `[ 'iTerm.app', 'kitty', 'WezTerm', 'ghostty', 'tmux', 'windows-terminal', ]` | src/ink/terminal.ts:156 | ui-rendering | in xterm.js-based terminals like VS Code). tmux is allowlisted because it accepts modifyOtherKeys and doesn't forward the kitty sequence to the outer terminal. |
| CLEAR_TERMINAL_TITLE | ``${OSC_PREFIX}${OSC.SET_TITLE_AND_ICON};${BEL}`` | src/ink/termio/osc.ts:450 | ui-rendering | Clear terminal title sequence (OSC 0 with empty string + BEL). Uses BEL terminator for cleanup — safe on all terminals. / |
| SUPPORTS_TERMINAL_VT_MODE | `` | src/keybindings/defaultBindings.ts:21 | feature-threshold | See: https://github.com/microsoft/terminal/issues/879#issuecomment-618801651 Node enabled VT mode in 24.2.0 / 22.17.0: https://github.com/nodejs/node/pull/58358 Bun enabled VT mode in 1.2.23: https://github.com/oven-sh/bun/pull/21161 |
| CHORD_TIMEOUT_MS | `1000;` | src/keybindings/KeybindingProviderSetup.tsx:30 | timeout | Timeout for chord sequences in milliseconds. If the user doesn't complete the chord within this time, it's cancelled. / |
| FILE_STABILITY_THRESHOLD_MS | `500` | src/keybindings/loadUserBindings.ts:51 | file-handling | Time in milliseconds to wait for file writes to stabilize. / |
| FILE_STABILITY_POLL_INTERVAL_MS | `200` | src/keybindings/loadUserBindings.ts:56 | timeout | Polling interval for checking file stability. / |
| CLAUDE_AI_MCP_TIMEOUT_MS | `5_000;` | src/main.tsx:2738 | timeout | climbed to 76s. If fetch+connect doesn't finish in time, proceed; the promise keeps running and updates headlessStore in the background so turn 2+ still sees connectors. |
| MAX_ENTRYPOINT_LINES | `200` | src/memdir/memdir.ts:35 | feature-threshold | export const MAX_ENTRYPOINT_LINES = 200 |
| MAX_ENTRYPOINT_BYTES | `25_000` | src/memdir/memdir.ts:38 | feature-threshold | ~125 chars/line at 200 lines. At p97 today; catches long-line indexes that slip past the line cap (p100 observed: 197KB under 200 lines). |
| MAX_MEMORY_FILES | `200` | src/memdir/memoryScan.ts:21 | file-handling | const MAX_MEMORY_FILES = 200 |
| FRONTMATTER_MAX_LINES | `30` | src/memdir/memoryScan.ts:22 | feature-threshold | const FRONTMATTER_MAX_LINES = 30 |
| CHANGE_THRESHOLD | `0.4` | src/native-ts/color-diff/index.ts:546 | feature-threshold | const CHANGE_THRESHOLD = 0.4 |
| TOP_LEVEL_CACHE_LIMIT | `100` | src/native-ts/file-index/index.ts:32 | token-management | const TOP_LEVEL_CACHE_LIMIT = 100 |
| MAX_QUERY_LEN | `64` | src/native-ts/file-index/index.ts:33 | file-handling | const MAX_QUERY_LEN = 64 |
| CHUNK_MS | `4` | src/native-ts/file-index/index.ts:38 | buffer-size | time-based (not count-based) so slow machines get smaller chunks and stay responsive — 5k paths is ~2ms on M-series but could be 15ms+ on older Windows hardware. |
| CACHE_SLOTS | `4` | src/native-ts/yoga-layout/index.ts:968 | token-management | const CACHE_SLOTS = 4 |
| MAX_OUTPUT_TOKENS_RECOVERY_LIMIT | `3` | src/query.ts:164 | token-management | the rules of thinking are the rules of the universe. If ye does not heed these rules, ye will be punished with an entire day of debugging and hair pulling. / |
| inline_setTimeout@stopHooks.ts:130 | `new Promise<void>(r => setTimeout(r, 60_000).unref()),` | src/query/stopHooks.ts:130 | timeout | eslint-disable-next-line no-restricted-syntax -- sleep() has no .unref(); timer must not block exit |
| COMPLETION_THRESHOLD | `0.9` | src/query/tokenBudget.ts:3 | token-management | const COMPLETION_THRESHOLD = 0.9 |
| DIMINISHING_THRESHOLD | `500` | src/query/tokenBudget.ts:4 | token-management | const DIMINISHING_THRESHOLD = 500 |
| RECONNECT_DELAY_MS | `2000` | src/remote/SessionsWebSocket.ts:17 | timeout | const RECONNECT_DELAY_MS = 2000 |
| MAX_RECONNECT_ATTEMPTS | `5` | src/remote/SessionsWebSocket.ts:18 | retry | const MAX_RECONNECT_ATTEMPTS = 5 |
| PING_INTERVAL_MS | `30000` | src/remote/SessionsWebSocket.ts:19 | timeout | const PING_INTERVAL_MS = 30000 |
| MAX_SESSION_NOT_FOUND_RETRIES | `3` | src/remote/SessionsWebSocket.ts:26 | retry | server may briefly consider the session stale; a short retry window lets the client recover without giving up permanently. / |
| RECENT_SCROLL_REPIN_WINDOW_MS | `3000;` | src/screens/REPL.tsx:305 | timeout | repin to bottom. Josh Rosen's workflow: Claude emits long output → scroll up to read the start → start typing → before this fix, snapped to bottom. https://anthropic.slack.com/archives/C07VBSHV7EV/p1773545449871739 |
| inline_setTimeout@REPL.tsx:430 | `setTimeout(() => alive && setIndexStatus(null), 2000);` | src/screens/REPL.tsx:430 | timeout | Inline timer magic number |
| TITLE_ANIMATION_INTERVAL_MS | `960;` | src/screens/REPL.tsx:475 | timeout | const TITLE_ANIMATION_INTERVAL_MS = 960; |
| inline_setTimeout@REPL.tsx:4327 | `editorTimerRef.current = setTimeout(s => s(''), 4000, setEditorStatus);` | src/screens/REPL.tsx:4327 | timeout | Inline timer magic number |
| inline_setTimeout@ResumeConversation.tsx:393 | `const timeout = setTimeout(_temp2, 100);` | src/screens/ResumeConversation.tsx:393 | timeout | Inline timer magic number |
| SUMMARY_INTERVAL_MS | `30_000` | src/services/AgentSummary/agentSummary.ts:26 | timeout | const SUMMARY_INTERVAL_MS = 30_000 |
| DEFAULT_FLUSH_INTERVAL_MS | `15000` | src/services/analytics/datadog.ts:15 | timeout | const DEFAULT_FLUSH_INTERVAL_MS = 15000 |
| MAX_BATCH_SIZE | `100` | src/services/analytics/datadog.ts:16 | buffer-size | const MAX_BATCH_SIZE = 100 |
| NETWORK_TIMEOUT_MS | `5000` | src/services/analytics/datadog.ts:17 | timeout | const NETWORK_TIMEOUT_MS = 5000 |
| BATCH_CONFIG_NAME | `'tengu_1p_event_batch_config'` | src/services/analytics/firstPartyEventLogger.ts:87 | buffer-size | const BATCH_CONFIG_NAME = 'tengu_1p_event_batch_config' |
| DEFAULT_LOGS_EXPORT_INTERVAL_MS | `10000` | src/services/analytics/firstPartyEventLogger.ts:300 | timeout | const DEFAULT_LOGS_EXPORT_INTERVAL_MS = 10000 |
| DEFAULT_MAX_EXPORT_BATCH_SIZE | `200` | src/services/analytics/firstPartyEventLogger.ts:301 | buffer-size | const DEFAULT_MAX_EXPORT_BATCH_SIZE = 200 |
| DEFAULT_MAX_QUEUE_SIZE | `8192` | src/services/analytics/firstPartyEventLogger.ts:302 | buffer-size | const DEFAULT_MAX_QUEUE_SIZE = 8192 |
| BATCH_UUID | `randomUUID()` | src/services/analytics/firstPartyEventLoggingExporter.ts:38 | buffer-size | Unique ID for this process run - used to isolate failed event files between runs |
| GROWTHBOOK_REFRESH_INTERVAL_MS | `` | src/services/analytics/growthbook.ts:1013 | timeout | Periodic refresh interval (matches Statsig's 6-hour interval) |
| TOOL_INPUT_STRING_TRUNCATE_AT | `512` | src/services/analytics/metadata.ts:236 | analytics | const TOOL_INPUT_STRING_TRUNCATE_AT = 512 |
| TOOL_INPUT_STRING_TRUNCATE_TO | `128` | src/services/analytics/metadata.ts:237 | analytics | const TOOL_INPUT_STRING_TRUNCATE_TO = 128 |
| TOOL_INPUT_MAX_JSON_CHARS | `4 * 1024` | src/services/analytics/metadata.ts:238 | analytics | const TOOL_INPUT_MAX_JSON_CHARS = 4 * 1024 |
| TOOL_INPUT_MAX_COLLECTION_ITEMS | `20` | src/services/analytics/metadata.ts:239 | analytics | const TOOL_INPUT_MAX_COLLECTION_ITEMS = 20 |
| TOOL_INPUT_MAX_DEPTH | `2` | src/services/analytics/metadata.ts:240 | analytics | const TOOL_INPUT_MAX_DEPTH = 2 |
| MAX_FILE_EXTENSION_LENGTH | `10` | src/services/analytics/metadata.ts:311 | file-handling | (e.g., hash-based filenames like "key-hash-abcd-123-456") and will be replaced with 'other'. / |
| STREAM_IDLE_TIMEOUT_MS | `` | src/services/api/claude.ts:1877 | timeout | const STREAM_IDLE_TIMEOUT_MS = |
| STALL_THRESHOLD_MS | `30_000 // 30 seconds` | src/services/api/claude.ts:1936 | timeout | stream in and accumulate state |
| MAX_NON_STREAMING_TOKENS | `64_000` | src/services/api/claude.ts:3354 | timeout | https://platform.claude.com/docs/en/api/errors#long-requests The SDK's 21333-token cap is derived from 10min × 128k tokens/hour, but we bypass it by setting a client-level timeout, so we can cap higher. |
| MAX_CACHED_REQUESTS | `5` | src/services/api/dumpPrompts.ts:14 | token-management | Cache last few API requests for ant users (e.g., for /issue command) |
| API_TIMEOUT_ERROR_MESSAGE | `'Request timed out'` | src/services/api/errors.ts:169 | timeout | export const API_TIMEOUT_ERROR_MESSAGE = 'Request timed out' |
| MAX_RETRIES | `3` | src/services/api/filesApi.ts:80 | file-handling | const MAX_RETRIES = 3 |
| BASE_DELAY_MS | `500` | src/services/api/filesApi.ts:81 | timeout | const BASE_DELAY_MS = 500 |
| MAX_FILE_SIZE_BYTES | `500 * 1024 * 1024 // 500MB` | src/services/api/filesApi.ts:82 | file-handling | const MAX_FILE_SIZE_BYTES = 500 * 1024 * 1024 // 500MB |
| DEFAULT_CONCURRENCY | `5` | src/services/api/filesApi.ts:270 | file-handling | Default concurrency limit for parallel downloads |
| GROVE_CACHE_EXPIRATION_MS | `24 * 60 * 60 * 1000` | src/services/api/grove.ts:23 | token-management | Cache expiration: 24 hours |
| CACHE_TTL_MS | `60 * 60 * 1000` | src/services/api/metricsOptOut.ts:22 | timeout | In-memory TTL — dedupes calls within a single process |
| DISK_CACHE_TTL_MS | `24 * 60 * 60 * 1000` | src/services/api/metricsOptOut.ts:27 | timeout | Disk TTL — org settings rarely change. When disk cache is fresher than this, we skip the network entirely (no background refresh). This is what collapses N `claude -p` invocations into ~1 API call/day. |
| CACHE_TTL_MS | `60 * 60 * 1000 // 1 hour` | src/services/api/overageCreditGrant.ts:22 | timeout | const CACHE_TTL_MS = 60 * 60 * 1000 // 1 hour |
| MAX_TRACKED_SOURCES | `10` | src/services/api/promptCacheBreakDetection.ts:107 | token-management | Each entry stores a ~300KB+ diffableContent string (serialized system prompt + tool schemas). Without a cap, spawning many subagents (each with a unique agentId key) causes the map to grow indefinitely. |
| MIN_CACHE_MISS_TOKENS | `2_000` | src/services/api/promptCacheBreakDetection.ts:120 | token-management | Minimum absolute token drop required to trigger a cache break warning. Small drops (e.g., a few thousand tokens) can happen due to normal variation and aren't worth alerting on. |
| CACHE_TTL_5MIN_MS | `5 * 60 * 1000` | src/services/api/promptCacheBreakDetection.ts:125 | timeout | Anthropic's server-side prompt cache TTL thresholds to test. Cache breaks after these durations are likely due to TTL expiration rather than client-side changes. |
| CACHE_TTL_1HOUR_MS | `60 * 60 * 1000` | src/services/api/promptCacheBreakDetection.ts:126 | timeout | Cache breaks after these durations are likely due to TTL expiration rather than client-side changes. |
| CACHE_EXPIRATION_MS | `24 * 60 * 60 * 1000` | src/services/api/referral.ts:21 | token-management | Cache expiration time: 24 hours (eligibility changes only on subscription/experiment changes) |
| MAX_RETRIES | `10` | src/services/api/sessionIngress.ts:25 | file-handling | Module-level state |
| BASE_DELAY_MS | `500` | src/services/api/sessionIngress.ts:26 | timeout | const BASE_DELAY_MS = 500 |
| DEFAULT_MAX_RETRIES | `10` | src/services/api/withRetry.ts:52 | retry | const DEFAULT_MAX_RETRIES = 10 |
| MAX_529_RETRIES | `3` | src/services/api/withRetry.ts:54 | retry | const MAX_529_RETRIES = 3 |
| BASE_DELAY_MS | `500` | src/services/api/withRetry.ts:55 | timeout | export const BASE_DELAY_MS = 500 |
| FOREGROUND_529_RETRY_SOURCES | `new Set<QuerySource>([ 'repl_main_thread', 'repl_main_thread:outputStyle:custom', 'repl_main_thread:outputStyle:Explanatory', 'repl_main_thread:outputStyle:Learning', 'sdk', 'agent:custom', 'agent:default',` | src/services/api/withRetry.ts:62 | retry | bails immediately: during a capacity cascade each retry is 3-10× gateway amplification, and the user never sees those fail anyway. New sources default to no-retry — add here only if the user is waiting on the result. |
| PERSISTENT_MAX_BACKOFF_MS | `5 * 60 * 1000` | src/services/api/withRetry.ts:96 | timeout | environment does not mark the session idle mid-wait. TODO(ANT-344): the keep-alive via SystemAPIErrorMessage yields is a stopgap until there's a dedicated keep-alive channel. |
| PERSISTENT_RESET_CAP_MS | `6 * 60 * 60 * 1000` | src/services/api/withRetry.ts:97 | retry | TODO(ANT-344): the keep-alive via SystemAPIErrorMessage yields is a stopgap until there's a dedicated keep-alive channel. |
| HEARTBEAT_INTERVAL_MS | `30_000` | src/services/api/withRetry.ts:98 | timeout | until there's a dedicated keep-alive channel. |
| SHORT_RETRY_THRESHOLD_MS | `20 * 1000 // 20 seconds` | src/services/api/withRetry.ts:800 | retry | const SHORT_RETRY_THRESHOLD_MS = 20 * 1000 // 20 seconds |
| MIN_COOLDOWN_MS | `10 * 60 * 1000 // 10 minutes` | src/services/api/withRetry.ts:801 | retry | const MIN_COOLDOWN_MS = 10 * 60 * 1000 // 10 minutes |
| SESSION_SCAN_INTERVAL_MS | `10 * 60 * 1000` | src/services/autoDream/autoDream.ts:56 | timeout | Scan throttle: when time-gate passes but session-gate doesn't, the lock mtime doesn't advance, so the time-gate keeps passing every turn. |
| RECENT_MESSAGE_WINDOW | `30` | src/services/awaySummary.ts:16 | file-handling | Recap only needs recent context — truncate to avoid "prompt too long" on large sessions. 30 messages ≈ ~15 exchanges, plenty for "where we left off." |
| DEFAULT_MAX_INPUT_TOKENS | `180_000 // Typical warning threshold` | src/services/compact/apiMicrocompact.ts:16 | token-management | Default values for context management strategies Match client-side microcompact token values |
| MAX_OUTPUT_TOKENS_FOR_SUMMARY | `20_000` | src/services/compact/autoCompact.ts:30 | token-management | Reserve this many tokens for output during compaction Based on p99.99 of compact summary output being 17,387 tokens. |
| AUTOCOMPACT_BUFFER_TOKENS | `13_000` | src/services/compact/autoCompact.ts:62 | token-management | export const AUTOCOMPACT_BUFFER_TOKENS = 13_000 |
| WARNING_THRESHOLD_BUFFER_TOKENS | `20_000` | src/services/compact/autoCompact.ts:63 | token-management | export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000 |
| ERROR_THRESHOLD_BUFFER_TOKENS | `20_000` | src/services/compact/autoCompact.ts:64 | token-management | export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000 |
| MANUAL_COMPACT_BUFFER_TOKENS | `3_000` | src/services/compact/autoCompact.ts:65 | token-management | export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000 |
| MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES | `3` | src/services/compact/autoCompact.ts:70 | token-management | Stop trying autocompact after this many consecutive failures. BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272) in a single session, wasting ~250K API calls/day globally. |
| POST_COMPACT_MAX_FILES_TO_RESTORE | `5` | src/services/compact/compact.ts:122 | token-management | export const POST_COMPACT_MAX_FILES_TO_RESTORE = 5 |
| POST_COMPACT_TOKEN_BUDGET | `50_000` | src/services/compact/compact.ts:123 | token-management | export const POST_COMPACT_TOKEN_BUDGET = 50_000 |
| POST_COMPACT_MAX_TOKENS_PER_FILE | `5_000` | src/services/compact/compact.ts:124 | token-management | export const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000 |
| POST_COMPACT_MAX_TOKENS_PER_SKILL | `5_000` | src/services/compact/compact.ts:129 | timeout | unbounded on every compact → 5-10K tok/compact. Per-skill truncation beats dropping — instructions at the top of a skill file are usually the critical part. Budget sized to hold ~5 skills at the per-skill cap. |
| POST_COMPACT_SKILLS_TOKEN_BUDGET | `25_000` | src/services/compact/compact.ts:130 | timeout | dropping — instructions at the top of a skill file are usually the critical part. Budget sized to hold ~5 skills at the per-skill cap. |
| MAX_COMPACT_STREAMING_RETRIES | `2` | src/services/compact/compact.ts:131 | token-management | part. Budget sized to hold ~5 skills at the per-skill cap. |
| MAX_PTL_RETRIES | `3` | src/services/compact/compact.ts:227 | token-management | const MAX_PTL_RETRIES = 3 |
| PTL_RETRY_MARKER | `'[earlier conversation truncated for compaction retry]'` | src/services/compact/compact.ts:228 | retry | const PTL_RETRY_MARKER = '[earlier conversation truncated for compaction retry]' |
| SKILL_TRUNCATION_MARKER | `` | src/services/compact/compact.ts:1657 | token-management | const SKILL_TRUNCATION_MARKER = |
| IMAGE_MAX_TOKEN_SIZE | `2000` | src/services/compact/microCompact.ts:38 | token-management | Drift is caught by a test asserting equality with the source-of-truth. |
| MAX_DIAGNOSTICS_SUMMARY_CHARS | `4000` | src/services/diagnosticTracking.ts:12 | feature-threshold | const MAX_DIAGNOSTICS_SUMMARY_CHARS = 4000 |
| MAX_DIAGNOSTICS_PER_FILE | `10` | src/services/lsp/LSPDiagnosticRegistry.ts:42 | file-handling | / Volume limiting constants |
| MAX_TOTAL_DIAGNOSTICS | `30` | src/services/lsp/LSPDiagnosticRegistry.ts:43 | feature-threshold | Volume limiting constants |
| MAX_DELIVERED_FILES | `500` | src/services/lsp/LSPDiagnosticRegistry.ts:46 | file-handling | Max files to track for deduplication - prevents unbounded memory growth |
| MAX_RETRIES_FOR_TRANSIENT_ERRORS | `3` | src/services/lsp/LSPServerInstance.ts:22 | feature-threshold | Maximum number of retries for transient LSP errors like "content modified". / |
| RETRY_BASE_DELAY_MS | `500` | src/services/lsp/LSPServerInstance.ts:28 | timeout | Base delay in milliseconds for exponential backoff on transient errors. Actual delays: 500ms, 1000ms, 2000ms / |
| AUTH_REQUEST_TIMEOUT_MS | `30000` | src/services/mcp/auth.ts:65 | timeout | Timeout for individual OAuth requests (metadata discovery, token refresh, etc.) / |
| MAX_LOCK_RETRIES | `5` | src/services/mcp/auth.ts:94 | security | const MAX_LOCK_RETRIES = 5 |
| MAX_ATTEMPTS | `3` | src/services/mcp/auth.ts:2180 | retry | const MAX_ATTEMPTS = 3 |
| FETCH_TIMEOUT_MS | `5000` | src/services/mcp/claudeai.ts:30 | timeout | const FETCH_TIMEOUT_MS = 5000 |
| DEFAULT_MCP_TOOL_TIMEOUT_MS | `100_000_000` | src/services/mcp/client.ts:211 | timeout | Default timeout for MCP tool calls (effectively infinite - ~27.8 hours). / |
| MAX_MCP_DESCRIPTION_LENGTH | `2048` | src/services/mcp/client.ts:218 | timeout | OpenAPI-generated MCP servers have been observed dumping 15-60KB of endpoint docs into tool.description; this caps the p95 tail without losing the intent. / |
| MCP_AUTH_CACHE_TTL_MS | `15 * 60 * 1000 // 15 min` | src/services/mcp/client.ts:257 | timeout | const MCP_AUTH_CACHE_TTL_MS = 15 * 60 * 1000 // 15 min |
| MCP_REQUEST_TIMEOUT_MS | `60000` | src/services/mcp/client.ts:463 | timeout | Default timeout for individual MCP requests (auth, tool calls, etc.) / |
| MAX_ERRORS_BEFORE_RECONNECT | `3` | src/services/mcp/client.ts:1228 | retry | which CC uses to trigger reconnection. We bridge this gap by tracking consecutive terminal errors and manually closing after MAX_ERRORS_BEFORE_RECONNECT failures. |
| MCP_FETCH_CACHE_SIZE | `20` | src/services/mcp/client.ts:1726 | retry | Max cache size for fetch* caches. Keyed by server name (stable across reconnects), bounded to prevent unbounded growth with many MCP servers. |
| MAX_SESSION_RETRIES | `1` | src/services/mcp/client.ts:1859 | file-handling | const MAX_SESSION_RETRIES = 1 |
| MAX_URL_ELICITATION_RETRIES | `3` | src/services/mcp/client.ts:2850 | mcp | const MAX_URL_ELICITATION_RETRIES = 3 |
| MAX_RECONNECT_ATTEMPTS | `5` | src/services/mcp/useManageMCPConnections.ts:88 | retry | Constants for reconnection with exponential backoff |
| INITIAL_BACKOFF_MS | `1000` | src/services/mcp/useManageMCPConnections.ts:89 | retry | Constants for reconnection with exponential backoff |
| MAX_BACKOFF_MS | `30000` | src/services/mcp/useManageMCPConnections.ts:90 | retry | Constants for reconnection with exponential backoff |
| MCP_BATCH_FLUSH_MS | `16` | src/services/mcp/useManageMCPConnections.ts:207 | timeout | in a single setAppState call via setTimeout. Using a time-based window (instead of queueMicrotask) ensures updates are batched even when connection callbacks arrive at different times due to network I/O. |
| XAA_REQUEST_TIMEOUT_MS | `30000` | src/services/mcp/xaa.ts:29 | timeout | const XAA_REQUEST_TIMEOUT_MS = 30000 |
| IDP_LOGIN_TIMEOUT_MS | `5 * 60 * 1000` | src/services/mcp/xaaIdpLogin.ts:51 | timeout | const IDP_LOGIN_TIMEOUT_MS = 5 * 60 * 1000 |
| IDP_REQUEST_TIMEOUT_MS | `30000` | src/services/mcp/xaaIdpLogin.ts:52 | timeout | const IDP_REQUEST_TIMEOUT_MS = 30000 |
| ID_TOKEN_EXPIRY_BUFFER_S | `60` | src/services/mcp/xaaIdpLogin.ts:53 | token-management | const ID_TOKEN_EXPIRY_BUFFER_S = 60 |
| VALID_INSTALLABLE_SCOPES | `['user', 'project', 'local'] as const` | src/services/plugins/pluginOperations.ts:72 | timeout | export const VALID_INSTALLABLE_SCOPES = ['user', 'project', 'local'] as const |
| CACHE_FILENAME | `'policy-limits.json'` | src/services/policyLimits/index.ts:55 | token-management | Constants |
| FETCH_TIMEOUT_MS | `10000 // 10 seconds` | src/services/policyLimits/index.ts:56 | timeout | Constants |
| DEFAULT_MAX_RETRIES | `5` | src/services/policyLimits/index.ts:57 | feature-threshold | Constants |
| POLLING_INTERVAL_MS | `60 * 60 * 1000 // 1 hour` | src/services/policyLimits/index.ts:58 | timeout | const POLLING_INTERVAL_MS = 60 * 60 * 1000 // 1 hour |
| LOADING_PROMISE_TIMEOUT_MS | `30000 // 30 seconds` | src/services/policyLimits/index.ts:69 | timeout | Timeout for the loading promise to prevent deadlocks |
| CAFFEINATE_TIMEOUT_SECONDS | `300 // 5 minutes` | src/services/preventSleep.ts:21 | timeout | Caffeinate timeout in seconds. Process auto-exits after this duration. We restart it before expiry to maintain continuous sleep prevention. |
| RESTART_INTERVAL_MS | `4 * 60 * 1000` | src/services/preventSleep.ts:25 | timeout | Restart interval - restart caffeinate before it expires. Use 4 minutes to give plenty of buffer before the 5 minute timeout. |
| MAX_PARENT_UNCACHED_TOKENS | `10_000` | src/services/PromptSuggestion/promptSuggestion.ts:239 | token-management | const MAX_PARENT_UNCACHED_TOKENS = 10_000 |
| MAX_SPECULATION_TURNS | `20` | src/services/PromptSuggestion/speculation.ts:58 | feature-threshold | const MAX_SPECULATION_TURNS = 20 |
| MAX_SPECULATION_MESSAGES | `100` | src/services/PromptSuggestion/speculation.ts:59 | feature-threshold | const MAX_SPECULATION_MESSAGES = 100 |
| RATE_LIMIT_ERROR_PREFIXES | `[ "You've hit your", "You've used", "You're now using extra usage", "You're close to", "You're out of extra usage", ] as const` | src/services/rateLimitMessages.ts:21 | ui-rendering | All possible rate limit error message prefixes Export this to avoid fragile string matching in UI components / |
| WARNING_THRESHOLD | `0.7` | src/services/rateLimitMessages.ts:72 | analytics | Only show warnings when utilization is above threshold (70%) This prevents false warnings after week reset when API may send allowed_warning with stale data at low usage levels |
| SETTINGS_TIMEOUT_MS | `10000 // 10 seconds for settings fetch` | src/services/remoteManagedSettings/index.ts:52 | timeout | Constants |
| DEFAULT_MAX_RETRIES | `5` | src/services/remoteManagedSettings/index.ts:53 | feature-threshold | Constants |
| POLLING_INTERVAL_MS | `60 * 60 * 1000 // 1 hour` | src/services/remoteManagedSettings/index.ts:54 | timeout | Constants |
| LOADING_PROMISE_TIMEOUT_MS | `30000 // 30 seconds` | src/services/remoteManagedSettings/index.ts:66 | timeout | Timeout for the loading promise to prevent deadlocks if loadRemoteManagedSettings() is never called (e.g., in Agent SDK tests that don't go through main.tsx) |
| MAX_SECTION_LENGTH | `2000` | src/services/SessionMemory/prompts.ts:8 | file-handling | const MAX_SECTION_LENGTH = 2000 |
| MAX_TOTAL_SESSION_MEMORY_TOKENS | `12000` | src/services/SessionMemory/prompts.ts:9 | token-management | const MAX_TOTAL_SESSION_MEMORY_TOKENS = 12000 |
| EXTRACTION_WAIT_TIMEOUT_MS | `15000` | src/services/SessionMemory/sessionMemoryUtils.ts:12 | timeout | const EXTRACTION_WAIT_TIMEOUT_MS = 15000 |
| EXTRACTION_STALE_THRESHOLD_MS | `60000 // 1 minute` | src/services/SessionMemory/sessionMemoryUtils.ts:13 | file-handling | const EXTRACTION_STALE_THRESHOLD_MS = 60000 // 1 minute |
| SETTINGS_SYNC_TIMEOUT_MS | `10000 // 10 seconds` | src/services/settingsSync/index.ts:51 | timeout | const SETTINGS_SYNC_TIMEOUT_MS = 10000 // 10 seconds |
| DEFAULT_MAX_RETRIES | `3` | src/services/settingsSync/index.ts:52 | feature-threshold | const DEFAULT_MAX_RETRIES = 3 |
| MAX_FILE_SIZE_BYTES | `500 * 1024 // 500 KB per file (matches backend limit)` | src/services/settingsSync/index.ts:53 | file-handling | const MAX_FILE_SIZE_BYTES = 500 * 1024 // 500 KB per file (matches backend limit) |
| TEAM_MEMORY_SYNC_TIMEOUT_MS | `30_000` | src/services/teamMemorySync/index.ts:71 | timeout | const TEAM_MEMORY_SYNC_TIMEOUT_MS = 30_000 |
| MAX_FILE_SIZE_BYTES | `250_000` | src/services/teamMemorySync/index.ts:75 | ui-rendering | Per-entry size cap — server default from anthropic/anthropic#293258. Pre-filtering oversized entries saves bandwidth: the structured 413 for this case doesn't give us anything to learn (one file is just too big). |
| MAX_PUT_BODY_BYTES | `200_000` | src/services/teamMemorySync/index.ts:89 | buffer-size | and keeps a single-entry-at-MAX_FILE_SIZE_BYTES solo batch (~250KB) just under the real gateway limit. Batches larger than this are split into sequential PUTs — server upsert-merge semantics make that safe. |
| MAX_RETRIES | `3` | src/services/teamMemorySync/index.ts:90 | buffer-size | under the real gateway limit. Batches larger than this are split into sequential PUTs — server upsert-merge semantics make that safe. |
| MAX_CONFLICT_RETRIES | `2` | src/services/teamMemorySync/index.ts:91 | retry | sequential PUTs — server upsert-merge semantics make that safe. |
| DEBOUNCE_MS | `2000 // Wait 2s after last change before pushing` | src/services/teamMemorySync/watcher.ts:35 | timeout | const DEBOUNCE_MS = 2000 // Wait 2s after last change before pushing |
| TOKEN_COUNT_THINKING_BUDGET | `1024` | src/services/tokenEstimation.ts:32 | token-management | Minimal values for token counting with thinking enabled API constraint: max_tokens must be greater than thinking.budget_tokens |
| TOKEN_COUNT_MAX_TOKENS | `2048` | src/services/tokenEstimation.ts:33 | token-management | Minimal values for token counting with thinking enabled API constraint: max_tokens must be greater than thinking.budget_tokens |
| HOOK_TIMING_DISPLAY_THRESHOLD_MS | `500` | src/services/tools/toolExecution.ts:134 | feature-threshold | export const HOOK_TIMING_DISPLAY_THRESHOLD_MS = 500 |
| SLOW_PHASE_LOG_THRESHOLD_MS | `2000` | src/services/tools/toolExecution.ts:137 | feature-threshold | BashTool's PROGRESS_THRESHOLD_MS — the collapsed view feels stuck past this. */ |
| SILENCE_THRESHOLD | `'3%'` | src/services/voice.ts:45 | feature-threshold | SoX silence detection: stop after this duration of silence |
| MAX_KEYTERMS | `50` | src/services/voiceKeyterms.ts:55 | feature-threshold | ─── Public API ───────────────────────────────────────────────────── |
| KEEPALIVE_INTERVAL_MS | `8_000` | src/services/voiceStreamSTT.ts:38 | timeout | const KEEPALIVE_INTERVAL_MS = 8_000 |
| FINALIZE_TIMEOUTS_MS | `{ safety: 5_000, noData: 1_500, }` | src/services/voiceStreamSTT.ts:44 | timeout | arrives post-CloseStream — the server has nothing; don't wait out the full ~3-5s WS teardown to confirm emptiness. `safety` is the last- resort cap if the WS hangs. Exported so tests can shorten them. |
| inline_setTimeout@voiceStreamSTT.ts:434 | `// inside the setTimeout(0) that actually sends CloseStream, so` | src/services/voiceStreamSTT.ts:434 | timeout | accumulated buffer immediately (~300ms) instead of waiting for the WebSocket close event (~3-5s of server teardown). `finalized` (not `finalizing`) is the right gate: it flips |
| MIN_AGENTS | `5` | src/skills/bundled/batch.ts:9 | buffer-size | const MIN_AGENTS = 5 |
| MAX_AGENTS | `30` | src/skills/bundled/batch.ts:10 | buffer-size | const MAX_AGENTS = 30 |
| DEFAULT_DEBUG_LINES_READ | `20` | src/skills/bundled/debug.ts:9 | feature-threshold | const DEFAULT_DEBUG_LINES_READ = 20 |
| DEFAULT_INTERVAL | `'10m'` | src/skills/bundled/loop.ts:9 | timeout | const DEFAULT_INTERVAL = '10m' |
| MAX_TURNS | `30` | src/tasks/DreamTask/DreamTask.ts:12 | feature-threshold | Keep only the N most recent turns for live display. |
| TEAMMATE_MESSAGES_UI_CAP | `50` | src/tasks/InProcessTeammateTask/types.ts:101 | ui-rendering | 9a990de8 launched 292 agents in 2 minutes and reached 36.8GB. The dominant cost is this array holding a second full copy of every message. / |
| MAX_RECENT_ACTIVITIES | `5;` | src/tasks/LocalAgentTask/LocalAgentTask.tsx:40 | feature-threshold | const MAX_RECENT_ACTIVITIES = 5; |
| MAX_RECENT_ACTIVITIES | `5` | src/tasks/LocalMainSessionTask.ts:325 | file-handling | Max recent activities to keep for display |
| STALL_CHECK_INTERVAL_MS | `5_000;` | src/tasks/LocalShellTask/LocalShellTask.tsx:24 | timeout | const STALL_CHECK_INTERVAL_MS = 5_000; |
| STALL_THRESHOLD_MS | `45_000;` | src/tasks/LocalShellTask/LocalShellTask.tsx:25 | timeout | const STALL_THRESHOLD_MS = 45_000; |
| STALL_TAIL_BYTES | `1024;` | src/tasks/LocalShellTask/LocalShellTask.tsx:26 | timeout | const STALL_TAIL_BYTES = 1024; |
| POLL_INTERVAL_MS | `1000;` | src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:540 | timeout | / |
| REMOTE_REVIEW_TIMEOUT_MS | `30 * 60 * 1000;` | src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:541 | timeout | const REMOTE_REVIEW_TIMEOUT_MS = 30 * 60 * 1000; |
| STABLE_IDLE_POLLS | `5;` | src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:545 | timeout | Remote sessions flip to 'idle' between tool turns. With 100+ rapid turns, a 1s poll WILL catch a transient idle mid-run. Require stable idle (no log growth for N consecutive polls) before believing it. |
| PROGRESS_THRESHOLD_MS | `2000; // Show background hint after 2 seconds` | src/tools/AgentTool/AgentTool.tsx:63 | feature-threshold | Progress display constants (for showing background hint) |
| MAX_WAIT_MS | `30_000;` | src/tools/AgentTool/AgentTool.tsx:378 | feature-threshold | const MAX_WAIT_MS = 30_000; |
| POLL_INTERVAL_MS | `500;` | src/tools/AgentTool/AgentTool.tsx:379 | timeout | const POLL_INTERVAL_MS = 500; |
| EXPLORE_AGENT_MIN_QUERIES | `3` | src/tools/AgentTool/built-in/exploreAgent.ts:59 | ui-rendering | export const EXPLORE_AGENT_MIN_QUERIES = 3 |
| SHARED_GUIDELINES | ``Your strengths: - Searching for code, configurations, and patterns across large codebases - Analyzing multiple files to understand system architecture - Investigating complex questions that require exploring many files ` | src/tools/AgentTool/built-in/generalPurposeAgent.ts:5 | ui-rendering | const SHARED_GUIDELINES = `Your strengths: |
| MAX_PROGRESS_MESSAGES_TO_SHOW | `3;` | src/tools/AgentTool/UI.tsx:33 | ui-rendering | const MAX_PROGRESS_MESSAGES_TO_SHOW = 3; |
| ESTIMATED_LINES_PER_TOOL | `9;` | src/tools/AgentTool/UI.tsx:181 | ui-rendering | const ESTIMATED_LINES_PER_TOOL = 9; |
| TERMINAL_BUFFER_LINES | `7;` | src/tools/AgentTool/UI.tsx:182 | buffer-size | const TERMINAL_BUFFER_LINES = 7; |
| ASK_USER_QUESTION_TOOL_CHIP_WIDTH | `12` | src/tools/AskUserQuestionTool/prompt.ts:5 | ui-rendering | export const ASK_USER_QUESTION_TOOL_CHIP_WIDTH = 12 |
| MAX_SUBCOMMANDS_FOR_SECURITY_CHECK | `50` | src/tools/BashTool/bashPermissions.ts:103 | timeout | strace showed /proc/self/stat reads at ~127Hz with no epoll_wait. Fifty is generous: legitimate user commands don't split that wide. Above the cap we fall back to 'ask' (safe default — we can't prove safety, so we prompt). |
| MAX_SUGGESTED_RULES_FOR_COMPOUND | `5` | src/tools/BashTool/bashPermissions.ts:110 | security | degrades to "similar commands" anyway, and saving 10+ rules from one prompt is more likely noise than intent. Users chaining this many write commands in one && list are rare; they can always approve once and add rules manually. |
| TIMEOUT_FLAG_VALUE_RE | `/^[A-Za-z0-9_.+-]+$/` | src/tools/BashTool/bashPermissions.ts:620 | timeout | SECURITY: allowlist for timeout flag VALUES (signals are TERM/KILL/9, durations are 5/5s/10.5). Rejects $ ( ) ` \| ; & and newlines that previously matched via [^ \t]+ — `timeout -k$(id) 10 ls` must NOT strip. |
| PROGRESS_THRESHOLD_MS | `2000; // Show progress after 2 seconds` | src/tools/BashTool/BashTool.tsx:55 | feature-threshold | Progress display constants |
| ASSISTANT_BLOCKING_BUDGET_MS | `15_000;` | src/tools/BashTool/BashTool.tsx:57 | token-management | Progress display constants In assistant mode, blocking bash auto-backgrounds after this many ms in the main agent |
| MAX_PERSISTED_SIZE | `64 * 1024 * 1024;` | src/tools/BashTool/BashTool.tsx:732 | buffer-size | stdout already contains the first chunk (from getStdout()). Copy the output file to the tool-results dir so the model can read it via FileRead. If > 64 MB, truncate after copying. |
| TIMEOUT_FLAG_VALUE_RE | `/^[A-Za-z0-9_.+-]+$/` | src/tools/BashTool/pathValidation.ts:1177 | timeout | SECURITY: allowlist for timeout flag VALUES (signals are TERM/KILL/9, durations are 5/5s/10.5). Rejects $ ( ) ` \| ; & and newlines that previously matched via [^ \t]+ — `timeout -k$(id) 10 ls` must NOT strip. |
| DANGEROUS_CAPABILITIES | `new Set([ 'init', 'reset', 'rs1', 'rs2', 'rs3', 'is1', 'is2',` | src/tools/BashTool/readOnlyValidation.ts:990 | buffer-size | (redirect output to printer device). smcup/rmcup manipulate screen buffer. pfkey/pfloc/pfx/pfxl program function keys — pfloc executes strings locally. rf is reset file (analogous to if/init_file). |
| ESCAPED_AMP_PLACEHOLDER | ``___ESCAPED_AMPERSAND_${salt}___`` | src/tools/BashTool/sedEditParser.ts:304 | analytics | Convert \n to newline, & to $& (match), etc. Use a unique placeholder with random salt to prevent injection attacks |
| MAX_COMMAND_DISPLAY_LINES | `2;` | src/tools/BashTool/UI.tsx:26 | ui-rendering | Constants for command display |
| MAX_COMMAND_DISPLAY_CHARS | `160;` | src/tools/BashTool/UI.tsx:27 | ui-rendering | Constants for command display |
| MAX_IMAGE_FILE_SIZE | `20 * 1024 * 1024` | src/tools/BashTool/utils.ts:96 | file-handling | Cap file reads to 20 MB — any image data URI larger than this is well beyond what the API accepts (5 MB base64) and would OOM if read into memory. |
| MAX_UPLOAD_BYTES | `30 * 1024 * 1024` | src/tools/BriefTool/upload.ts:32 | file-handling | Matches the private_api backend limit |
| UPLOAD_TIMEOUT_MS | `30_000` | src/tools/BriefTool/upload.ts:34 | timeout | Matches the private_api backend limit |
| MAX_EDIT_FILE_SIZE | `1024 * 1024 * 1024 // 1 GiB (stat bytes)` | src/tools/FileEditTool/FileEditTool.ts:84 | file-handling | ≈ 1 billion characters ≈ the runtime string limit. Multi-byte UTF-8 files can be larger on disk per character, but 1 GiB is a safe byte-level guard that prevents OOM without being unnecessarily restrictive. |
| DIFF_SNIPPET_MAX_BYTES | `8192` | src/tools/FileEditTool/utils.ts:355 | token-management | Cap on edited_text_file attachment snippets. Format-on-save of a large file previously injected the entire file per turn (observed max 16.1KB, ~14K tokens/session). 8KB preserves meaningful context while bounding worst case. |
| CONTEXT_LINES | `4` | src/tools/FileEditTool/utils.ts:408 | file-handling | const CONTEXT_LINES = 4 |
| CYBER_RISK_MITIGATION_REMINDER | `` | src/tools/FileReadTool/FileReadTool.ts:729 | file-handling | export const CYBER_RISK_MITIGATION_REMINDER = |
| DEFAULT_MAX_OUTPUT_TOKENS | `25000` | src/tools/FileReadTool/limits.ts:18 | token-management | export const DEFAULT_MAX_OUTPUT_TOKENS = 25000 |
| MAX_LINES_TO_READ | `2000` | src/tools/FileReadTool/prompt.ts:10 | file-handling | export const MAX_LINES_TO_READ = 2000 |
| MAX_LINES_TO_RENDER | `10;` | src/tools/FileWriteTool/UI.tsx:26 | ui-rendering | const MAX_LINES_TO_RENDER = 10; |
| DEFAULT_HEAD_LIMIT | `250` | src/tools/GrepTool/GrepTool.ts:108 | token-management | greps can fill up to the 20KB persist threshold (~6-24K tokens/grep-heavy session). 250 is generous enough for exploratory searches while preventing context bloat. Pass head_limit=0 explicitly for unlimited. |
| MAX_LSP_FILE_SIZE_BYTES | `10_000_000` | src/tools/LSPTool/LSPTool.ts:53 | file-handling | const MAX_LSP_FILE_SIZE_BYTES = 10_000_000 |
| BATCH_SIZE | `50` | src/tools/LSPTool/LSPTool.ts:580 | buffer-size | Batch check paths with git check-ignore Exit code 0 = at least one path is ignored, 1 = none ignored, 128 = not a git repo |
| MAX_READ_BYTES | `64 * 1024` | src/tools/LSPTool/symbolContext.ts:6 | feature-threshold | const MAX_READ_BYTES = 64 * 1024 |
| MCP_OUTPUT_WARNING_THRESHOLD_TOKENS | `10_000;` | src/tools/MCPTool/UI.tsx:21 | token-management | Threshold for displaying warning about large MCP responses |
| MAX_INPUT_VALUE_CHARS | `80;` | src/tools/MCPTool/UI.tsx:26 | timeout | In non-verbose mode, truncate individual input values to keep the header compact. Matches BashTool's philosophy of showing enough to identify the call without dumping the entire payload inline. |
| MAX_FLAT_JSON_KEYS | `12;` | src/tools/MCPTool/UI.tsx:30 | ui-rendering | Max number of top-level keys before we fall back to raw JSON display. Beyond this a flat k:v list is more noise than help. |
| MAX_FLAT_JSON_CHARS | `5_000;` | src/tools/MCPTool/UI.tsx:33 | retry | Don't attempt flat-object parsing for large blobs. |
| MAX_JSON_PARSE_CHARS | `200_000;` | src/tools/MCPTool/UI.tsx:36 | retry | Don't attempt to parse JSON blobs larger than this (perf safety). |
| UNWRAP_MIN_STRING_LEN | `200;` | src/tools/MCPTool/UI.tsx:40 | timeout | A string value is "dominant text payload" if it has newlines or is long enough that inline display would be worse than unwrapping. |
| MAX_DIRS_TO_LIST | `5` | src/tools/PowerShellTool/pathValidation.ts:46 | file-handling | const MAX_DIRS_TO_LIST = 5 |
| PROGRESS_THRESHOLD_MS | `2000;` | src/tools/PowerShellTool/PowerShellTool.tsx:159 | feature-threshold | Progress display constants |
| PROGRESS_INTERVAL_MS | `1000;` | src/tools/PowerShellTool/PowerShellTool.tsx:160 | timeout | Progress display constants |
| ASSISTANT_BLOCKING_BUDGET_MS | `15_000;` | src/tools/PowerShellTool/PowerShellTool.tsx:162 | token-management | In assistant mode, blocking commands auto-background after this many ms in the main agent |
| WINDOWS_SANDBOX_POLICY_REFUSAL | `'Enterprise policy requires sandboxing, but sandboxing is not available on native Windows. Shell command execution is blocked on this platform by policy.';` | src/tools/PowerShellTool/PowerShellTool.tsx:219 | feature-threshold | (covers direct callers like promptShellExecution.ts that skip validateInput). The call() guard is the load-bearing one. / |
| MAX_PERSISTED_SIZE | `64 * 1024 * 1024;` | src/tools/PowerShellTool/PowerShellTool.tsx:596 | file-handling | ordering, where persistence is post-try/finally): a failing command that also produced >maxOutputLength bytes would otherwise do 3-4 disk syscalls, store to tool-results/, then throw — orphaning the file. |
| WINDOWS_PATHEXT | `/\.(exe\|cmd\|bat\|com)$/` | src/tools/PowerShellTool/readOnlyValidation.ts:973 | file-handling | .ps1 is intentionally excluded — a script named git.ps1 is not the git binary and does not trigger git's hook mechanism. / |
| MAX_COMMAND_DISPLAY_LINES | `2;` | src/tools/PowerShellTool/UI.tsx:17 | ui-rendering | Constants for command display |
| MAX_COMMAND_DISPLAY_CHARS | `160;` | src/tools/PowerShellTool/UI.tsx:18 | ui-rendering | Constants for command display |
| MAX_JOBS | `50` | src/tools/ScheduleCronTool/CronCreateTool.ts:25 | feature-threshold | const MAX_JOBS = 50 |
| DEFAULT_MAX_AGE_DAYS | `` | src/tools/ScheduleCronTool/prompt.ts:8 | feature-threshold | export const DEFAULT_MAX_AGE_DAYS = |
| SKILL_BUDGET_CONTEXT_PERCENT | `0.01` | src/tools/SkillTool/prompt.ts:21 | token-management | Skill listing gets 1% of the context window (in characters) |
| DEFAULT_CHAR_BUDGET | `8_000 // Fallback: 1% of 200k × 4` | src/tools/SkillTool/prompt.ts:23 | token-management | Skill listing gets 1% of the context window (in characters) |
| MAX_LISTING_DESC_CHARS | `250` | src/tools/SkillTool/prompt.ts:29 | token-management | full content on invoke, so verbose whenToUse strings waste turn-1 cache_creation tokens without improving match rate. Applies to all entries, including bundled, since the cap is generous enough to preserve the core use case. |
| MIN_DESC_LENGTH | `20` | src/tools/SkillTool/prompt.ts:68 | feature-threshold | const MIN_DESC_LENGTH = 20 |
| MAX_PROGRESS_MESSAGES_TO_SHOW | `3;` | src/tools/SkillTool/UI.tsx:18 | ui-rendering | const MAX_PROGRESS_MESSAGES_TO_SHOW = 3; |
| MAX_COMMAND_DISPLAY_LINES | `2;` | src/tools/TaskStopTool/UI.tsx:10 | ui-rendering | const MAX_COMMAND_DISPLAY_LINES = 2; |
| MAX_COMMAND_DISPLAY_CHARS | `160;` | src/tools/TaskStopTool/UI.tsx:11 | ui-rendering | const MAX_COMMAND_DISPLAY_CHARS = 160; |
| CACHE_TTL_MS | `15 * 60 * 1000 // 15 minutes` | src/tools/WebFetchTool/utils.ts:63 | timeout | Cache with 15-minute TTL and 50MB size limit LRUCache handles automatic expiration and eviction |
| MAX_CACHE_SIZE_BYTES | `50 * 1024 * 1024 // 50MB` | src/tools/WebFetchTool/utils.ts:64 | timeout | Cache with 15-minute TTL and 50MB size limit LRUCache handles automatic expiration and eviction |
| URL_CACHE | `new LRUCache<string, CacheEntry>({ maxSize: MAX_CACHE_SIZE_BYTES, ttl: CACHE_TTL_MS, })` | src/tools/WebFetchTool/utils.ts:66 | token-management | const URL_CACHE = new LRUCache<string, CacheEntry>({ |
| DOMAIN_CHECK_CACHE | `new LRUCache<string, true>({ max: 128, ttl: 5 * 60 * 1000, // 5 minutes — shorter than URL_CACHE TTL })` | src/tools/WebFetchTool/utils.ts:75 | retry | fetching two paths on the same domain triggers two identical preflight HTTP round-trips to api.anthropic.com. This hostname-keyed cache avoids that. Only 'allowed' is cached — blocked/failed re-check on next attempt. |
| MAX_URL_LENGTH | `2000` | src/tools/WebFetchTool/utils.ts:106 | security | which provides a primary security boundary. In addition, Claude Code has other data exfil channels, and this one does not seem relatively high risk, so I'm removing that length restriction. -ab |
| MAX_HTTP_CONTENT_LENGTH | `10 * 1024 * 1024` | src/tools/WebFetchTool/utils.ts:112 | buffer-size | "Implement resource consumption controls because setting limits on CPU, memory, and network usage for the Web Fetch tool can prevent a single request or user from overwhelming the system." |
| FETCH_TIMEOUT_MS | `60_000` | src/tools/WebFetchTool/utils.ts:116 | timeout | Timeout for the main HTTP fetch request (60 seconds). Prevents hanging indefinitely on slow/unresponsive servers. |
| DOMAIN_CHECK_TIMEOUT_MS | `10_000` | src/tools/WebFetchTool/utils.ts:119 | timeout | Timeout for the domain blocklist preflight check (10 seconds). |
| MAX_REDIRECTS | `10` | src/tools/WebFetchTool/utils.ts:125 | timeout | a redirect loop (/a → /b → /a …) and the per-request FETCH_TIMEOUT_MS resets on every hop, hanging the tool until user interrupt. 10 matches common client defaults (axios=5, follow-redirects=21, Chrome=20). |
| MAX_MARKDOWN_LENGTH | `100_000` | src/tools/WebFetchTool/utils.ts:128 | token-management | Truncate to not spend too many tokens |
| MAX_CHUNK_BYTES | `512 * 1024` | src/upstreamproxy/relay.ts:51 | buffer-size | Envoy per-request buffer cap. Week-1 Datadog payloads won't hit this, but design for it so git-push doesn't need a relay rewrite. |
| PING_INTERVAL_MS | `30_000` | src/upstreamproxy/relay.ts:54 | timeout | Sidecar idle timeout is 50s; ping well inside that. |
| DEFAULT_MAX_LISTENERS | `50` | src/utils/abortController.ts:6 | feature-threshold | Default max listeners for standard operations / |
| MAX_TRANSCRIPT_CHARS | `2000 // Max chars of transcript per session` | src/utils/agenticSessionSearch.ts:11 | file-handling | Limits for transcript extraction |
| MAX_MESSAGES_TO_SCAN | `100 // Max messages to scan from start/end` | src/utils/agenticSessionSearch.ts:12 | file-handling | Limits for transcript extraction |
| MAX_SESSIONS_TO_SEARCH | `100 // Max sessions to send to the API` | src/utils/agenticSessionSearch.ts:13 | file-handling | Limits for transcript extraction |
| MANUAL_COMPACT_BUFFER_NAME | `'Compact buffer'` | src/utils/analyzeContext.ts:66 | token-management | const MANUAL_COMPACT_BUFFER_NAME = 'Compact buffer' |
| GRID_WIDTH | `` | src/utils/analyzeContext.ts:1180 | ui-rendering | For narrow screens (< 80 cols), use 5x5 for 200k models, 5x10 for 1M+ models For normal screens, use 10x10 for 200k models, 20x10 for 1M+ models |
| GRID_HEIGHT | `contextWindow >= 1000000 ? 10 : isNarrowScreen ? 5 : 10` | src/utils/analyzeContext.ts:1188 | ui-rendering | const GRID_HEIGHT = contextWindow >= 1000000 ? 10 : isNarrowScreen ? 5 : 10 |
| TODO_REMINDER_CONFIG | `{ TURNS_SINCE_WRITE: 10, TURNS_BETWEEN_REMINDERS: 10, } as const` | src/utils/attachments.ts:254 | file-handling | export const TODO_REMINDER_CONFIG = { |
| MAX_MEMORY_LINES | `200` | src/utils/attachments.ts:269 | file-handling | const MAX_MEMORY_LINES = 200 |
| MAX_MEMORY_BYTES | `4096` | src/utils/attachments.ts:277 | file-handling | readFileInRange's truncateOnByteLimit option. Truncation means the most-relevant memory still surfaces: the frontmatter + opening context is usually what matters. |
| VERIFY_PLAN_REMINDER_CONFIG | `{ TURNS_BETWEEN_REMINDERS: 10, } as const` | src/utils/attachments.ts:291 | file-handling | export const VERIFY_PLAN_REMINDER_CONFIG = { |
| inline_setTimeout@attachments.ts:767 | `const timeoutId = setTimeout(ac => ac.abort(), 1000, abortController)` | src/utils/attachments.ts:767 | timeout | TODO: Compute attachments as the user types, not here (though we use this function for slash command prompts too) |
| MAX_DIR_ENTRIES | `1000` | src/utils/attachments.ts:1922 | file-handling | const MAX_DIR_ENTRIES = 1000 |
| FILTERED_LISTING_MAX | `30` | src/utils/attachments.ts:2641 | timeout | When skill-search is enabled and the filtered (bundled + MCP) listing exceeds this count, fall back to bundled-only. Protects MCP-heavy users (100+ servers) from truncation while keeping the turn-0 guarantee for typical setups. |
| DEFAULT_API_KEY_HELPER_TTL | `5 * 60 * 1000` | src/utils/auth.ts:81 | timeout | const DEFAULT_API_KEY_HELPER_TTL = 5 * 60 * 1000 |
| DEFAULT_AWS_STS_TTL | `60 * 60 * 1000` | src/utils/auth.ts:606 | timeout | const DEFAULT_AWS_STS_TTL = 60 * 60 * 1000 |
| AWS_AUTH_REFRESH_TIMEOUT_MS | `3 * 60 * 1000` | src/utils/auth.ts:648 | timeout | Timeout for AWS auth refresh command (3 minutes). Long enough for browser-based SSO flows, short enough to prevent indefinite hangs. |
| GCP_CREDENTIALS_CHECK_TIMEOUT_MS | `5_000` | src/utils/auth.ts:841 | timeout | credential source exists (no ADC file, no env var), google-auth-library falls through to the GCE metadata server which hangs ~12s outside GCP. */ |
| DEFAULT_GCP_CREDENTIAL_TTL | `60 * 60 * 1000` | src/utils/auth.ts:869 | timeout | const DEFAULT_GCP_CREDENTIAL_TTL = 60 * 60 * 1000 |
| GCP_AUTH_REFRESH_TIMEOUT_MS | `3 * 60 * 1000` | src/utils/auth.ts:915 | timeout | Timeout for GCP auth refresh command (3 minutes). Long enough for browser-based auth flows, short enough to prevent indefinite hangs. |
| MAX_RETRIES | `5` | src/utils/auth.ts:1451 | security | const MAX_RETRIES = 5 |
| DEFAULT_OTEL_HEADERS_DEBOUNCE_MS | `29 * 60 * 1000 // 29 minutes` | src/utils/auth.ts:1768 | timeout | Cache for debouncing otelHeadersHelper calls |
| MAX_DENIALS | `20` | src/utils/autoModeDenials.ts:17 | feature-threshold | const MAX_DENIALS = 20 |
| LOCK_TIMEOUT_MS | `5 * 60 * 1000 // 5 minute timeout for locks` | src/utils/autoUpdater.ts:162 | timeout | Lock file for auto-updater to prevent concurrent updates |
| RECURRING_CLEANUP_INTERVAL_MS | `24 * 60 * 60 * 1000` | src/utils/backgroundHousekeeping.ts:26 | timeout | 24 hours in milliseconds |
| DELAY_VERY_SLOW_OPERATIONS_THAT_HAPPEN_EVERY_SESSION | `10 * 60 * 1000` | src/utils/backgroundHousekeeping.ts:29 | timeout | 10 minutes after start. |
| PARSE_TIMEOUT_MS | `50` | src/utils/bash/bashParser.ts:29 | timeout | Pass `Infinity` via `parse(src, Infinity)` to disable (e.g. correctness tests, where CI jitter would otherwise cause spurious null returns). / |
| MAX_NODES | `50_000` | src/utils/bash/bashParser.ts:32 | feature-threshold | const MAX_NODES = 50_000 |
| MAX_COMMAND_LENGTH | `10000` | src/utils/bash/parser.ts:19 | feature-threshold | const MAX_COMMAND_LENGTH = 10000 |
| MAX_SHELL_COMPLETIONS | `15` | src/utils/bash/shellCompletion.ts:12 | feature-threshold | Constants |
| SHELL_COMPLETION_TIMEOUT_MS | `1000` | src/utils/bash/shellCompletion.ts:13 | timeout | Constants |
| SNAPSHOT_CREATION_TIMEOUT | `10000 // 10 seconds` | src/utils/bash/ShellSnapshot.ts:24 | timeout | const SNAPSHOT_CREATION_TIMEOUT = 10000 // 10 seconds |
| MAX_SANITIZED_LENGTH | `200` | src/utils/cachePaths.ts:12 | token-management | sessionStoragePortable.ts which uses Bun.hash (wyhash) when available. Cache directory names must remain stable across upgrades so existing cache data (error logs, MCP logs) is not orphaned. |
| CACHE_PATHS | `{ baseLogs: () => join(paths.cache, getProjectDir(getFsImplementation().cwd())), errors: () => join(paths.cache, getProjectDir(getFsImplementation().cwd()), 'errors'), messages: () => join(paths.cache, getProjectDir(getF` | src/utils/cachePaths.ts:25 | token-management | export const CACHE_PATHS = { |
| MAX_MESSAGE_SIZE | `1024 * 1024 // 1MB - Max message size that can be sent to Chrome` | src/utils/claudeInChrome/chromeNativeHost.ts:27 | feature-threshold | const MAX_MESSAGE_SIZE = 1024 * 1024 // 1MB - Max message size that can be sent to Chrome |
| MAX_TRACKED_TABS | `200` | src/utils/claudeInChrome/common.ts:415 | feature-threshold | const MAX_TRACKED_TABS = 200 |
| CLAUDE_IN_CHROME_SKILL_HINT_WITH_WEBBROWSER | ``**Browser Automation**: Use WebBrowser for development (dev servers, JS eval, console, screenshots). Use claude-in-chrome for the user's real Chrome when you need logged-in sessions, OAuth, or computer-use — invoke Skil` | src/utils/claudeInChrome/prompt.ts:83 | ui-rendering | dev-loop tasks to WebBrowser and reserve the extension for the user's authenticated Chrome (logged-in sites, OAuth, computer-use). / |
| MAX_MEMORY_CHARACTER_COUNT | `40000` | src/utils/claudemd.ts:92 | file-handling | Recommended max character count for a memory file |
| MAX_INCLUDE_DEPTH | `5` | src/utils/claudemd.ts:537 | feature-threshold | const MAX_INCLUDE_DEPTH = 5 |
| NPM_CACHE_RETENTION_COUNT | `5` | src/utils/cleanup.ts:462 | token-management | const NPM_CACHE_RETENTION_COUNT = 5 |
| MAX_HINT_CHARS | `300` | src/utils/collapseReadSearch.ts:118 | ui-rendering | ~5 lines × ~60 cols. Generous static cap — the renderer lets Ink wrap. |
| APP_NAME_MAX_LEN | `40` | src/utils/computerUse/appNames.ts:109 | feature-threshold | Still bars quotes, angle brackets, backticks, pipes, colons. / |
| APP_NAME_MAX_COUNT | `50` | src/utils/computerUse/appNames.ts:110 | feature-threshold | / |
| UNHIDE_TIMEOUT_MS | `5000` | src/utils/computerUse/cleanup.ts:15 | timeout | timeout — unhide should be ~instant; if it takes 5s something is wrong and proceeding is better than waiting. The Swift call continues in the background regardless; we just stop blocking on it. |
| CLI_CU_CAPABILITIES | `{ screenshotFiltering: 'native' as const, platform: 'darwin' as const, }` | src/utils/computerUse/common.ts:54 | ui-rendering | by `executor.ts` per `ComputerExecutor.capabilities`. `buildComputerUseTools` takes this shape (no `hostBundleId`, no `teachMode`). / |
| inline_setInterval@drainRunLoop.ts:27 | `pump = setInterval(drainTick, 1, requireComputerUseSwift())` | src/utils/computerUse/drainRunLoop.ts:27 | timeout | Inline timer magic number |
| TIMEOUT_MS | `30_000` | src/utils/computerUse/drainRunLoop.ts:42 | timeout | const TIMEOUT_MS = 30_000 |
| MOVE_SETTLE_MS | `50` | src/utils/computerUse/executor.ts:111 | timeout | decomposed mouseDown/moveMouse path, emitting stray `.leftMouseDragged` events (toolCalls.ts handleScroll's mouse_full workaround). / |
| APP_ENUM_TIMEOUT_MS | `1000` | src/utils/computerUse/mcpServer.ts:18 | timeout | const APP_ENUM_TIMEOUT_MS = 1000 |
| CONFIG_WRITE_DISPLAY_THRESHOLD | `20` | src/utils/config.ts:887 | feature-threshold | export const CONFIG_WRITE_DISPLAY_THRESHOLD = 20 |
| CONFIG_FRESHNESS_POLL_MS | `1000` | src/utils/config.ts:992 | timeout | fs.watchFile poll interval for detecting writes from other instances (ms) |
| MIN_BACKUP_INTERVAL_MS | `60_000` | src/utils/config.ts:1263 | timeout | backup already exists. During startup, many saveGlobalConfig calls fire within milliseconds of each other; without this check, each call creates a new backup file that accumulates on disk. |
| MAX_BACKUPS | `5` | src/utils/config.ts:1284 | timeout | Clean up old backups, keeping only the 5 most recent |
| MODEL_CONTEXT_WINDOW_DEFAULT | `200_000` | src/utils/context.ts:9 | token-management | Model context window size (200k tokens for all models right now) |
| COMPACT_MAX_OUTPUT_TOKENS | `20_000` | src/utils/context.ts:12 | token-management | Maximum output tokens for compact operations |
| MAX_OUTPUT_TOKENS_DEFAULT | `32_000` | src/utils/context.ts:15 | token-management | Default max output tokens |
| MAX_OUTPUT_TOKENS_UPPER_LIMIT | `64_000` | src/utils/context.ts:16 | token-management | Default max output tokens |
| CAPPED_DEFAULT_MAX_TOKENS | `8_000` | src/utils/context.ts:24 | token-management | (see query.ts max_output_tokens_escalate). Cap is applied in claude.ts:getMaxOutputTokensForModel to avoid the growthbook→betas→context import cycle. |
| ESCALATED_MAX_TOKENS | `64_000` | src/utils/context.ts:25 | token-management | claude.ts:getMaxOutputTokensForModel to avoid the growthbook→betas→context import cycle. |
| NEAR_CAPACITY_PERCENT | `80` | src/utils/contextSuggestions.ts:25 | feature-threshold | const NEAR_CAPACITY_PERCENT = 80 |
| CHECK_INTERVAL_MS | `1000` | src/utils/cronScheduler.ts:40 | timeout | const CHECK_INTERVAL_MS = 1000 |
| LOCK_PROBE_INTERVAL_MS | `5000` | src/utils/cronScheduler.ts:44 | timeout | How often a non-owning session re-probes the scheduler lock. Coarse because takeover only matters when the owning session has crashed. |
| KILL_RING_MAX_SIZE | `10` | src/utils/Cursor.ts:16 | feature-threshold | Consecutive kills accumulate in the kill ring until the user types some other key. Alt+Y cycles through previous kills after a yank. / |
| LONG_PREFILL_THRESHOLD | `1000` | src/utils/deepLink/banner.ts:30 | ui-rendering | carefully" to an explicit "scroll to review the entire prompt" so a malicious tail buried past line 60 isn't silently off-screen. / |
| MAX_QUERY_LENGTH | `5000` | src/utils/deepLink/parseDeepLink.ts:70 | ui-rendering | launch error, not a security issue — so we don't penalize real users for an implausible input. / |
| MAX_CWD_LENGTH | `4096` | src/utils/deepLink/parseDeepLink.ts:77 | ui-rendering | opt-in). No real path approaches this; a cwd over 4096 is malformed or malicious. / |
| WINDOWS_REG_KEY | ``HKEY_CURRENT_USER\\Software\\Classes\\${DEEP_LINK_PROTOCOL}`` | src/utils/deepLink/registerProtocol.ts:51 | ui-rendering | const WINDOWS_REG_KEY = `HKEY_CURRENT_USER\\Software\\Classes\\${DEEP_LINK_PROTOCOL}` |
| WINDOWS_COMMAND_KEY | ``${WINDOWS_REG_KEY}\\shell\\open\\command`` | src/utils/deepLink/registerProtocol.ts:52 | ui-rendering | const WINDOWS_COMMAND_KEY = `${WINDOWS_REG_KEY}\\shell\\open\\command` |
| FAILURE_BACKOFF_MS | `24 * 60 * 60 * 1000` | src/utils/deepLink/registerProtocol.ts:54 | retry | const FAILURE_BACKOFF_MS = 24 * 60 * 60 * 1000 |
| LINUX_TERMINALS | `[ 'ghostty', 'kitty', 'alacritty', 'wezterm', 'gnome-terminal', 'konsole', 'xfce4-terminal',` | src/utils/deepLink/terminalLauncher.ts:46 | ui-rendering | Linux terminals in preference order (command name) |
| MIN_DESKTOP_VERSION | `'1.1.2396'` | src/utils/desktopDeepLink.ts:11 | ui-rendering | const MIN_DESKTOP_VERSION = '1.1.2396' |
| CONTEXT_LINES | `3` | src/utils/diff.ts:9 | feature-threshold | export const CONTEXT_LINES = 3 |
| DIFF_TIMEOUT_MS | `5_000` | src/utils/diff.ts:10 | timeout | export const DIFF_TIMEOUT_MS = 5_000 |
| MCP_TOOLS_THRESHOLD | `25_000 // 15k tokens` | src/utils/doctorContextWarnings.ts:21 | mcp | Thresholds (matching status notices and existing patterns) |
| LIMITS | `{ MAX_FILE_SIZE: 512 * 1024 * 1024, // 512MB per file MAX_TOTAL_SIZE: 1024 * 1024 * 1024, // 1024MB total uncompressed MAX_FILE_COUNT: 100000, // Maximum number of files MAX_COMPRESSION_RATIO: 50, // Anything above 50:1 ` | src/utils/dxt/zip.ts:7 | feature-threshold | const LIMITS = { |
| SECONDS_IN_MINUTE | `60` | src/utils/execFileNoThrow.ts:12 | file-handling | const SECONDS_IN_MINUTE = 60 |
| SECONDS_IN_MINUTE | `60` | src/utils/execFileNoThrowPortable.ts:6 | file-handling | const SECONDS_IN_MINUTE = 60 |
| PREFETCH_MIN_INTERVAL_MS | `30_000` | src/utils/fastMode.ts:383 | timeout | const PREFETCH_MIN_INTERVAL_MS = 30_000 |
| MAX_OUTPUT_SIZE | `0.25 * 1024 * 1024 // 0.25MB in bytes` | src/utils/file.ts:48 | file-handling | export const MAX_OUTPUT_SIZE = 0.25 * 1024 * 1024 // 0.25MB in bytes |
| MAX_SNAPSHOTS | `100` | src/utils/fileHistory.ts:54 | file-handling | const MAX_SNAPSHOTS = 100 |
| MAX_CONTENT_HASH_SIZE | `100 * 1024` | src/utils/fileOperationAnalytics.ts:32 | file-handling | Maximum content size to hash (100KB) Prevents memory exhaustion when hashing large files (e.g., base64-encoded images) |
| READ_FILE_STATE_CACHE_SIZE | `100` | src/utils/fileStateCache.ts:18 | token-management | Default max entries for read file state caches |
| DEFAULT_MAX_CACHE_SIZE_BYTES | `25 * 1024 * 1024` | src/utils/fileStateCache.ts:22 | token-management | Default size limit for file state caches (25MB) This prevents unbounded memory growth from large file contents |
| CHUNK_SIZE | `1024 * 4` | src/utils/fsOperations.ts:725 | buffer-size | const CHUNK_SIZE = 1024 * 4 |
| GH_TIMEOUT_MS | `5000` | src/utils/ghPrStatus.ts:19 | timeout | const GH_TIMEOUT_MS = 5000 |
| MAX_FILE_SIZE_BYTES | `500 * 1024 * 1024 // 500MB per file` | src/utils/git.ts:548 | file-handling | Size limits for untracked file capture |
| MAX_TOTAL_SIZE_BYTES | `5 * 1024 * 1024 * 1024 // 5GB total` | src/utils/git.ts:549 | file-handling | Size limits for untracked file capture |
| MAX_FILE_COUNT | `20000` | src/utils/git.ts:550 | file-handling | Size limits for untracked file capture |
| SNIFF_BUFFER_SIZE | `64 * 1024` | src/utils/git.ts:556 | buffer-size | most source files in a single read; isBinaryContent() internally scans only its first 8KB for the binary heuristic, so the extra bytes are purely for avoiding a second read when the file turns out to be text. |
| WATCH_INTERVAL_MS | `process.env.NODE_ENV === 'test' ? 10 : 1000` | src/utils/git/gitFilesystem.ts:331 | timeout | const WATCH_INTERVAL_MS = process.env.NODE_ENV === 'test' ? 10 : 1000 |
| GIT_TIMEOUT_MS | `5000` | src/utils/gitDiff.ts:35 | timeout | const GIT_TIMEOUT_MS = 5000 |
| MAX_FILES | `50` | src/utils/gitDiff.ts:36 | file-handling | const MAX_FILES = 50 |
| MAX_DIFF_SIZE_BYTES | `1_000_000 // 1 MB - skip files larger than this` | src/utils/gitDiff.ts:37 | feature-threshold | const MAX_DIFF_SIZE_BYTES = 1_000_000 // 1 MB - skip files larger than this |
| MAX_LINES_PER_FILE | `400 // GitHub's auto-load limit` | src/utils/gitDiff.ts:38 | file-handling | const MAX_LINES_PER_FILE = 400 // GitHub's auto-load limit |
| MAX_FILES_FOR_DETAILS | `500 // Skip per-file details if more files than this` | src/utils/gitDiff.ts:39 | file-handling | const MAX_FILES_FOR_DETAILS = 500 // Skip per-file details if more files than this |
| SINGLE_FILE_DIFF_TIMEOUT_MS | `3000` | src/utils/gitDiff.ts:384 | timeout | const SINGLE_FILE_DIFF_TIMEOUT_MS = 3000 |
| GROUPING_CACHE | `new WeakMap<Tools, Set<string>>()` | src/utils/groupToolUses.ts:23 | timeout | tools array reference. The tools array is stable across renders (only replaced on MCP connect/disconnect), so this avoids rebuilding the set on every call. WeakMap lets old entries be GC'd when the array is replaced. |
| TOOL_HOOK_EXECUTION_TIMEOUT_MS | `10 * 60 * 1000` | src/utils/hooks.ts:166 | timeout | const TOOL_HOOK_EXECUTION_TIMEOUT_MS = 10 * 60 * 1000 |
| SESSION_END_HOOK_TIMEOUT_MS_DEFAULT | `1500` | src/utils/hooks.ts:175 | timeout | parallel, so one value suffices). Overridable via env var for users whose teardown scripts need more time. / |
| MAX_AGENT_TURNS | `50` | src/utils/hooks/execAgentHook.ts:119 | feature-threshold | const MAX_AGENT_TURNS = 50 |
| DEFAULT_HTTP_HOOK_TIMEOUT_MS | `10 * 60 * 1000 // 10 minutes (matches TOOL_HOOK_EXECUTION_TIMEOUT_MS)` | src/utils/hooks/execHttpHook.ts:12 | timeout | const DEFAULT_HTTP_HOOK_TIMEOUT_MS = 10 * 60 * 1000 // 10 minutes (matches TOOL_HOOK_EXECUTION_TIMEOUT_MS) |
| MAX_PENDING_EVENTS | `100` | src/utils/hooks/hookEvents.ts:20 | analytics | / |
| TURN_BATCH_SIZE | `5` | src/utils/hooks/skillImprovement.ts:31 | buffer-size | const TURN_BATCH_SIZE = 5 |
| PASTE_THRESHOLD | `800` | src/utils/imagePaste.ts:30 | file-handling | Threshold in characters for when to consider text a "large paste" |
| ERROR_TYPE_PIXEL_LIMIT | `4` | src/utils/imageResizer.ts:28 | feature-threshold | const ERROR_TYPE_PIXEL_LIMIT = 4 |
| ERROR_TYPE_TIMEOUT | `6` | src/utils/imageResizer.ts:30 | timeout | const ERROR_TYPE_TIMEOUT = 6 |
| MAX_STORED_IMAGE_PATHS | `200` | src/utils/imageStore.ts:10 | file-handling | const MAX_STORED_IMAGE_PATHS = 200 |
| PARSE_CACHE_MAX_KEY_BYTES | `8 * 1024` | src/utils/json.ts:29 | token-management | so a 200KB config file would pin ~10MB in #keyList across 50 slots. Large inputs like ~/.claude.json also change between reads (numStartups bumps on every CC startup), so the cache never hits anyway. |
| MAX_JSONL_READ_BYTES | `100 * 1024 * 1024` | src/utils/json.ts:192 | feature-threshold | const MAX_JSONL_READ_BYTES = 100 * 1024 * 1024 |
| READ_BATCH_SIZE | `32` | src/utils/listSessionsImpl.ts:224 | buffer-size | --------------------------------------------------------------------------- |
| MAX_IN_MEMORY_ERRORS | `100` | src/utils/log.ts:66 | feature-threshold | In-memory error log for recent errors Moved from bootstrap/state.ts to break import cycle |
| MAX_LEFT_WIDTH | `50` | src/utils/logoV2Utils.ts:18 | ui-rendering | Layout constants |
| MAX_USERNAME_LENGTH | `20` | src/utils/logoV2Utils.ts:19 | feature-threshold | Layout constants |
| DIVIDER_WIDTH | `1` | src/utils/logoV2Utils.ts:21 | ui-rendering | const DIVIDER_WIDTH = 1 |
| MCP_TOKEN_COUNT_THRESHOLD_FACTOR | `0.5` | src/utils/mcpValidation.ts:14 | token-management | export const MCP_TOKEN_COUNT_THRESHOLD_FACTOR = 0.5 |
| DEFAULT_MAX_MCP_OUTPUT_TOKENS | `25000` | src/utils/mcpValidation.ts:16 | token-management | const DEFAULT_MAX_MCP_OUTPUT_TOKENS = 25000 |
| IS_WINDOWS | `process.platform === 'win32'` | src/utils/memoryFileDetection.ts:22 | file-handling | const IS_WINDOWS = process.platform === 'win32' |
| PLAN_PHASE4_CAP | ``### Phase 4: Final Plan Goal: Write your final plan to the plan file (the only file you can edit). - Do NOT write a Context, Background, or Overview section. The user just told you what they want. - Do NOT restate the u` | src/utils/messages.ts:3181 | feature-threshold | const PLAN_PHASE4_CAP = `### Phase 4: Final Plan |
| DEFAULT_STALL_TIMEOUT_MS | `60000 // 60 seconds` | src/utils/nativeInstaller/download.ts:272 | timeout | Stall timeout: abort if no bytes received for this duration |
| MAX_DOWNLOAD_RETRIES | `3` | src/utils/nativeInstaller/download.ts:273 | timeout | Stall timeout: abort if no bytes received for this duration |
| STALL_TIMEOUT_MS | `DEFAULT_STALL_TIMEOUT_MS` | src/utils/nativeInstaller/download.ts:522 | timeout | Exported for testing |
| LARGE_OUTPUT_THRESHOLD | `10000` | src/utils/notebook.ts:20 | feature-threshold | const LARGE_OUTPUT_THRESHOLD = 10000 |
| TERMINAL_CAPTURE_TOOL_NAME | `feature('TERMINAL_PANEL')` | src/utils/permissions/classifierDecision.ts:27 | ui-rendering | Ant-only tool names: conditional require so Bun can DCE these in external builds. Gates mirror tools.ts. Keeps the tool name strings out of cli.js. |
| DENIAL_LIMITS | `{ maxConsecutive: 3, maxTotal: 20, } as const` | src/utils/permissions/denialTracking.ts:12 | security | export const DENIAL_LIMITS = { |
| MAX_DIRS_TO_LIST | `5` | src/utils/permissions/pathValidation.ts:24 | security | const MAX_DIRS_TO_LIST = 5 |
| WINDOWS_DRIVE_ROOT_REGEX | `/^[A-Za-z]:\/?$/` | src/utils/permissions/pathValidation.ts:318 | security | const WINDOWS_DRIVE_ROOT_REGEX = /^[A-Za-z]:\/?$/ |
| WINDOWS_DRIVE_CHILD_REGEX | `/^[A-Za-z]:\/[^/]+$/` | src/utils/permissions/pathValidation.ts:319 | security | const WINDOWS_DRIVE_CHILD_REGEX = /^[A-Za-z]:\/[^/]+$/ |
| NO_CACHED_AUTO_MODE_CONFIG | `Symbol('no-cached-auto-mode-config')` | src/utils/permissions/permissionSetup.ts:1335 | token-management | const NO_CACHED_AUTO_MODE_CONFIG = Symbol('no-cached-auto-mode-config') |
| ESCAPED_STAR_PLACEHOLDER | `'\x00ESCAPED_STAR\x00'` | src/utils/permissions/shellRuleMatching.ts:14 | timeout | Null-byte sentinel placeholders for wildcard pattern escaping — module-level so the RegExp objects are compiled once instead of per permission check. |
| ESCAPED_BACKSLASH_PLACEHOLDER | `'\x00ESCAPED_BACKSLASH\x00'` | src/utils/permissions/shellRuleMatching.ts:15 | timeout | Null-byte sentinel placeholders for wildcard pattern escaping — module-level so the RegExp objects are compiled once instead of per permission check. |
| ESCAPED_STAR_PLACEHOLDER_RE | `new RegExp(ESCAPED_STAR_PLACEHOLDER, 'g')` | src/utils/permissions/shellRuleMatching.ts:16 | security | so the RegExp objects are compiled once instead of per permission check. |
| ESCAPED_BACKSLASH_PLACEHOLDER_RE | `new RegExp( ESCAPED_BACKSLASH_PLACEHOLDER, 'g', )` | src/utils/permissions/shellRuleMatching.ts:17 | security | const ESCAPED_BACKSLASH_PLACEHOLDER_RE = new RegExp( |
| MAX_SLUG_RETRIES | `10` | src/utils/plans.ts:25 | feature-threshold | const MAX_SLUG_RETRIES = 10 |
| MAX_SHOWN_PLUGINS | `100` | src/utils/plugins/hintRecommendation.ts:39 | feature-threshold | plugin appends one slug; past this point we stop prompting (and stop appending) rather than let the config grow without limit. / |
| INSTALL_COUNTS_CACHE_VERSION | `1` | src/utils/plugins/installCounts.ts:23 | timeout | const INSTALL_COUNTS_CACHE_VERSION = 1 |
| INSTALL_COUNTS_CACHE_FILENAME | `'install-counts-cache.json'` | src/utils/plugins/installCounts.ts:24 | timeout | const INSTALL_COUNTS_CACHE_FILENAME = 'install-counts-cache.json' |
| INSTALL_COUNTS_URL | `` | src/utils/plugins/installCounts.ts:25 | timeout | const INSTALL_COUNTS_URL = |
| CACHE_TTL_MS | `24 * 60 * 60 * 1000 // 24 hours in milliseconds` | src/utils/plugins/installCounts.ts:27 | timeout | const CACHE_TTL_MS = 24 * 60 * 60 * 1000 // 24 hours in milliseconds |
| MAX_IGNORED_COUNT | `5` | src/utils/plugins/lspRecommendation.ts:41 | feature-threshold | Maximum number of times user can ignore recommendations before we stop showing |
| DEFAULT_PLUGIN_GIT_TIMEOUT_MS | `120 * 1000` | src/utils/plugins/marketplaceManager.ts:515 | timeout | const DEFAULT_PLUGIN_GIT_TIMEOUT_MS = 120 * 1000 |
| RETRY_CONFIG | `{ MAX_ATTEMPTS: 10, INITIAL_DELAY_MS: 60 * 60 * 1000, // 1 hour BACKOFF_MULTIPLIER: 2, MAX_DELAY_MS: 7 * 24 * 60 * 60 * 1000, // 1 week }` | src/utils/plugins/officialMarketplaceStartupCheck.ts:56 | retry | Configuration for retry logic / |
| DEFAULT_PARSE_TIMEOUT_MS | `5_000` | src/utils/powershell/parser.ts:207 | timeout | attackVectors F1 hit 2×5s timeout → valid:false → 'ask' instead of 'deny'). Override via env for tests. Read inside parsePowerShellCommandImpl, not top-level, per CLAUDE.md (globalSettings.env ordering). |
| WINDOWS_ARGV_CAP | `32_767` | src/utils/powershell/parser.ts:611 | file-handling | If the Windows limit becomes too restrictive, switch to -File with a temp file for large inputs. --------------------------------------------------------------------------- |
| SCRIPT_CHARS_BUDGET | `((WINDOWS_ARGV_CAP - FIXED_ARGV_OVERHEAD) * 3) / 8` | src/utils/powershell/parser.ts:622 | token-management | estimation drift. Multibyte expansion is NOT absorbed here — the gate measures actual UTF-8 bytes (Buffer.byteLength), not code units. |
| CMD_B64_BUDGET | `` | src/utils/powershell/parser.ts:623 | token-management | measures actual UTF-8 bytes (Buffer.byteLength), not code units. |
| WINDOWS_MAX_COMMAND_LENGTH | `Math.max( 0, Math.floor((CMD_B64_BUDGET * 3) / 4) - SAFETY_MARGIN, )` | src/utils/powershell/parser.ts:627 | buffer-size | Exported for drift-guard tests (the drift-prone value is the Windows one). Unit: UTF-8 BYTES. Compare against Buffer.byteLength, not .length. |
| UNIX_MAX_COMMAND_LENGTH | `4_500` | src/utils/powershell/parser.ts:636 | security | commands (the common case) bytes==chars so no regression; for multibyte commands this is slightly tighter but still far below Unix ARG_MAX (~128KB per-arg), so the argv spawn cannot overflow. |
| MAX_COMMAND_LENGTH | `` | src/utils/powershell/parser.ts:638 | security | per-arg), so the argv spawn cannot overflow. Unit: UTF-8 BYTES (see SECURITY note above). |
| inline_setTimeout@preflightChecks.tsx:113 | `const timer = setTimeout(_temp, 100);` | src/utils/preflightChecks.tsx:113 | timeout | Inline timer magic number |
| MCP_SETTLE_POLL_MS | `200;` | src/utils/processUserInput/processSlashCommand.tsx:56 | timeout | Poll interval and deadline for MCP settle before launching a background forked subagent. MCP servers typically connect within 1-3s of startup; 10s headroom covers slow SSE handshakes. |
| MCP_SETTLE_TIMEOUT_MS | `10_000;` | src/utils/processUserInput/processSlashCommand.tsx:57 | timeout | forked subagent. MCP servers typically connect within 1-3s of startup; 10s headroom covers slow SSE handshakes. |
| MAX_HOOK_OUTPUT_LENGTH | `10000` | src/utils/processUserInput/processUserInput.ts:272 | feature-threshold | const MAX_HOOK_OUTPUT_LENGTH = 10000 |
| ASK_READ_FILE_STATE_CACHE_SIZE | `10` | src/utils/queryHelpers.ts:46 | token-management | Small cache size for ask operations which typically access few files during permission prompts or limited tool operations |
| MAX_TOOL_PROGRESS_TRACKING_ENTRIES | `100` | src/utils/queryHelpers.ts:98 | analytics | Track last sent time for tool progress messages per tool use ID Keep only the last 100 entries to prevent unbounded growth |
| TOOL_PROGRESS_THROTTLE_MS | `30000` | src/utils/queryHelpers.ts:99 | timeout | Track last sent time for tool progress messages per tool use ID Keep only the last 100 entries to prevent unbounded growth |
| CHUNK_SIZE | `8 * 1024` | src/utils/readEditContext.ts:4 | buffer-size | export const CHUNK_SIZE = 8 * 1024 |
| MAX_SCAN_BYTES | `10 * 1024 * 1024` | src/utils/readEditContext.ts:5 | feature-threshold | export const MAX_SCAN_BYTES = 10 * 1024 * 1024 |
| FAST_PATH_MAX_SIZE | `10 * 1024 * 1024 // 10 MB` | src/utils/readFileInRange.ts:44 | file-handling | const FAST_PATH_MAX_SIZE = 10 * 1024 * 1024 // 10 MB |
| MAX_RELEASE_NOTES_SHOWN | `5` | src/utils/releaseNotes.ts:13 | feature-threshold | const MAX_RELEASE_NOTES_SHOWN = 5 |
| MAX_BUFFER_SIZE | `20_000_000 // 20MB; large monorepos can have 200k+ files` | src/utils/ripgrep.ts:80 | buffer-size | const MAX_BUFFER_SIZE = 20_000_000 // 20MB; large monorepos can have 200k+ files |
| inline_setTimeout@ripgrep.ts:180 | `killTimeoutId = setTimeout(c => c.kill('SIGKILL'), 5_000, child)` | src/utils/ripgrep.ts:180 | timeout | Inline timer magic number |
| MAX_ITERATIONS | `10 // Safety limit to prevent infinite loops` | src/utils/sanitization.ts:29 | feature-threshold | const MAX_ITERATIONS = 10 // Safety limit to prevent infinite loops |
| MAX_QUEUE_SIZE | `1000` | src/utils/sdkEventQueue.ts:74 | buffer-size | const MAX_QUEUE_SIZE = 1000 |
| KEYCHAIN_PREFETCH_TIMEOUT_MS | `10_000` | src/utils/secureStorage/keychainPrefetch.ts:33 | timeout | const KEYCHAIN_PREFETCH_TIMEOUT_MS = 10_000 |
| KEYCHAIN_CACHE_TTL_MS | `30_000` | src/utils/secureStorage/macOsKeychainHelpers.ts:69 | timeout | prime it without pulling in execa. Wrapped in an object because ES module `let` bindings aren't writable across module boundaries — both this file and macOsKeychainStorage.ts need to mutate all three fields. |
| SECURITY_STDIN_LINE_LIMIT | `4096 - 64` | src/utils/secureStorage/macOsKeychainStorage.ts:24 | buffer-size | storage then reads as stale. See #30337. Headroom of 64B below the limit guards against edge-case line-terminator accounting differences. |
| SESSION_ACTIVITY_INTERVAL_MS | `30_000` | src/utils/sessionActivity.ts:18 | timeout | const SESSION_ACTIVITY_INTERVAL_MS = 30_000 |
| MAX_TOMBSTONE_REWRITE_BYTES | `50 * 1024 * 1024` | src/utils/sessionStorage.ts:123 | file-handling | / 50MB — prevents OOM in the tombstone slow path which reads + rewrites the entire session file. Session files can grow to multiple GB (inc-3930). |
| MAX_TRANSCRIPT_READ_BYTES | `50 * 1024 * 1024` | src/utils/sessionStorage.ts:229 | file-handling | 50 MB — session JSONL can grow to multiple GB (inc-3930). Callers that read the raw transcript must bail out above this threshold to avoid OOM. |
| REMOTE_FLUSH_INTERVAL_MS | `10` | src/utils/sessionStorage.ts:530 | timeout | const REMOTE_FLUSH_INTERVAL_MS = 10 |
| LITE_READ_BUF_SIZE | `65536` | src/utils/sessionStoragePortable.ts:17 | file-handling | export const LITE_READ_BUF_SIZE = 65536 |
| MAX_SANITIZED_LENGTH | `200` | src/utils/sessionStoragePortable.ts:293 | ui-rendering | Most filesystems (ext4, APFS, NTFS) limit individual components to 255 bytes. We use 200 to leave room for the hash suffix and separator. / |
| TRANSCRIPT_READ_CHUNK_SIZE | `1024 * 1024` | src/utils/sessionStoragePortable.ts:473 | buffer-size | --------------------------------------------------------------------------- |
| SKIP_PRECOMPACT_THRESHOLD | `5 * 1024 * 1024` | src/utils/sessionStoragePortable.ts:480 | token-management | Large sessions (>5 MB) almost always have compact boundaries — they got big because of many turns triggering auto-compact. / |
| CHUNK_SIZE | `TRANSCRIPT_READ_CHUNK_SIZE` | src/utils/sessionStoragePortable.ts:726 | buffer-size | const CHUNK_SIZE = TRANSCRIPT_READ_CHUNK_SIZE |
| MAX_CONVERSATION_TEXT | `1000` | src/utils/sessionTitle.ts:26 | file-handling | const MAX_CONVERSATION_TEXT = 1000 |
| FILE_STABILITY_THRESHOLD_MS | `1000` | src/utils/settings/changeDetector.ts:31 | file-handling | Time in milliseconds to wait for file writes to stabilize before processing. This helps avoid processing partial writes or rapid successive changes. / |
| FILE_STABILITY_POLL_INTERVAL_MS | `500` | src/utils/settings/changeDetector.ts:38 | timeout | Used by chokidar's awaitWriteFinish option. Must be lower than FILE_STABILITY_THRESHOLD_MS. / |
| INTERNAL_WRITE_WINDOW_MS | `5000` | src/utils/settings/changeDetector.ts:45 | file-handling | If a file change occurs within this window after markInternalWrite() is called, it's assumed to be from Claude Code itself and won't trigger a notification. / |
| MDM_POLL_INTERVAL_MS | `30 * 60 * 1000 // 30 minutes` | src/utils/settings/changeDetector.ts:51 | timeout | Poll interval for MDM settings (registry/plist) changes. These can't be watched via filesystem events, so we poll periodically. / |
| WINDOWS_REGISTRY_KEY_PATH_HKLM | `` | src/utils/settings/mdm/constants.ts:23 | file-handling | redirected and 32-bit processes would silently read from WOW6432Node. See: https://learn.microsoft.com/en-us/windows/win32/winprog64/shared-registry-keys / |
| WINDOWS_REGISTRY_KEY_PATH_HKCU | `` | src/utils/settings/mdm/constants.ts:25 | file-handling | / |
| WINDOWS_REGISTRY_VALUE_NAME | `'Settings'` | src/utils/settings/mdm/constants.ts:29 | feature-threshold | export const WINDOWS_REGISTRY_VALUE_NAME = 'Settings' |
| MDM_SUBPROCESS_TIMEOUT_MS | `5000` | src/utils/settings/mdm/constants.ts:38 | timeout | export const MDM_SUBPROCESS_TIMEOUT_MS = 5000 |
| DEFAULT_TIMEOUT | `30 * 60 * 1000 // 30 minutes` | src/utils/Shell.ts:44 | timeout | const DEFAULT_TIMEOUT = 30 * 60 * 1000 // 30 minutes |
| BASH_MAX_OUTPUT_UPPER_LIMIT | `150_000` | src/utils/shell/outputLimits.ts:3 | feature-threshold | export const BASH_MAX_OUTPUT_UPPER_LIMIT = 150_000 |
| BASH_MAX_OUTPUT_DEFAULT | `30_000` | src/utils/shell/outputLimits.ts:4 | feature-threshold | export const BASH_MAX_OUTPUT_DEFAULT = 30_000 |
| URL_PROTOCOLS | `['http://', 'https://', 'ftp://']` | src/utils/shell/specPrefix.ts:16 | ui-rendering | const URL_PROTOCOLS = ['http://', 'https://', 'ftp://'] |
| SIZE_WATCHDOG_INTERVAL_MS | `5_000` | src/utils/ShellCommand.ts:54 | timeout | Background tasks write stdout/stderr directly to a file fd (no JS involvement), so a stuck append loop can fill the disk. Poll file size and kill when exceeded. |
| FILE_STABILITY_THRESHOLD_MS | `1000` | src/utils/skills/skillChangeDetector.ts:27 | file-handling | Time in milliseconds to wait for file writes to stabilize before processing. / |
| FILE_STABILITY_POLL_INTERVAL_MS | `500` | src/utils/skills/skillChangeDetector.ts:32 | timeout | Polling interval in milliseconds for checking file stability. / |
| RELOAD_DEBOUNCE_MS | `300` | src/utils/skills/skillChangeDetector.ts:42 | timeout | clearCommandsCache() + listener notification cycle, which can deadlock the event loop when dozens of events fire in rapid succession. / |
| POLLING_INTERVAL_MS | `2000` | src/utils/skills/skillChangeDetector.ts:49 | timeout | Skill files change rarely (manual edits, git operations), so a 2s interval trades negligible latency for far fewer stat() calls than the default 100ms. / |
| USE_POLLING | `typeof Bun !== 'undefined'` | src/utils/skills/skillChangeDetector.ts:62 | timeout | Workaround: use stat() polling under Bun. No FSWatcher = no deadlock. The fix is pending upstream; remove this once the Bun PR lands. / |
| SLOW_OPERATION_THRESHOLD_MS | `(() => {` | src/utils/slowOperations.ts:29 | ui-rendering | - Dev builds: 20ms (lower threshold for development) - Ants: 300ms (enabled for all internal users) / |
| inline_setTimeout@staticRender.tsx:30 | `const timer = setTimeout(exit, 0);` | src/utils/staticRender.tsx:30 | timeout | Inline timer magic number |
| BATCH_SIZE | `20` | src/utils/stats.ts:138 | buffer-size | Process session files in parallel batches for better performance |
| STATS_CACHE_VERSION | `3` | src/utils/statsCache.ts:14 | token-management | export const STATS_CACHE_VERSION = 3 |
| MIN_MIGRATABLE_VERSION | `1` | src/utils/statsCache.ts:15 | token-management | const MIN_MIGRATABLE_VERSION = 1 |
| STATS_CACHE_FILENAME | `'stats-cache.json'` | src/utils/statsCache.ts:16 | token-management | const STATS_CACHE_FILENAME = 'stats-cache.json' |
| AGENT_DESCRIPTIONS_THRESHOLD | `15_000` | src/utils/statusNoticeHelpers.ts:4 | feature-threshold | export const AGENT_DESCRIPTIONS_THRESHOLD = 15_000 |
| MAX_STRING_LENGTH | `2 ** 25` | src/utils/stringUtils.ts:88 | feature-threshold | Keep in-memory accumulation modest to avoid blowing up RSS. Overflow beyond this limit is spilled to disk by ShellCommand. |
| CACHE_SIZE | `500` | src/utils/suggestions/directoryCompletion.ts:37 | token-management | Cache configuration |
| CACHE_TTL | `5 * 60 * 1000 // 5 minutes` | src/utils/suggestions/directoryCompletion.ts:38 | timeout | Cache configuration |
| CACHE_TTL_MS | `60000 // 60 seconds - history won't change while typing` | src/utils/suggestions/shellHistoryCompletion.ts:18 | timeout | History only changes when user submits a command, so a long TTL is fine |
| SKILL_USAGE_DEBOUNCE_MS | `60_000` | src/utils/suggestions/skillUsageTracking.ts:3 | timeout | const SKILL_USAGE_DEBOUNCE_MS = 60_000 |
| PANE_SHELL_INIT_DELAY_MS | `200` | src/utils/swarm/backends/TmuxBackend.ts:33 | timeout | Delay after pane creation to allow shell initialization (loading rc files, prompts, etc.) 200ms is enough for most shell configurations including slow ones like starship/oh-my-zsh |
| SWARM_VIEW_WINDOW_NAME | `'swarm-view'` | src/utils/swarm/constants.ts:3 | feature-threshold | export const SWARM_VIEW_WINDOW_NAME = 'swarm-view' |
| PERMISSION_POLL_INTERVAL_MS | `500` | src/utils/swarm/inProcessRunner.ts:114 | timeout | const PERMISSION_POLL_INTERVAL_MS = 500 |
| POLL_INTERVAL_MS | `500` | src/utils/swarm/inProcessRunner.ts:697 | timeout | const POLL_INTERVAL_MS = 500 |
| inline_setTimeout@It2SetupPrompt.tsx:75 | `setTimeout(onDone, 1500, "installed" as const);` | src/utils/swarm/It2SetupPrompt.tsx:75 | timeout | Inline timer magic number |
| inline_setTimeout@It2SetupPrompt.tsx:284 | `setTimeout(onDone, 1500, "installed" as const);` | src/utils/swarm/It2SetupPrompt.tsx:284 | timeout | Inline timer magic number |
| ENCODED_LENGTH | `22` | src/utils/taggedId.ts:14 | feature-threshold | ceil(128 / log2(58)) = 22 |
| DEFAULT_MAX_READ_BYTES | `8 * 1024 * 1024 // 8MB` | src/utils/task/diskOutput.ts:23 | feature-threshold | O_NOFOLLOW is not available on Windows, but the sandbox attack vector is Unix-only. |
| MAX_TASK_OUTPUT_BYTES | `5 * 1024 * 1024 * 1024` | src/utils/task/diskOutput.ts:30 | buffer-size | file size and kills the process. In pipe mode (hooks), DiskTaskOutput drops chunks past this limit. Shared so both caps stay in sync. / |
| MAX_TASK_OUTPUT_BYTES_DISPLAY | `'5GB'` | src/utils/task/diskOutput.ts:31 | buffer-size | drops chunks past this limit. Shared so both caps stay in sync. / |
| POLL_INTERVAL_MS | `1000` | src/utils/task/framework.ts:22 | timeout | Standard polling interval for all tasks |
| TASK_MAX_OUTPUT_UPPER_LIMIT | `160_000` | src/utils/task/outputFormatting.ts:4 | feature-threshold | export const TASK_MAX_OUTPUT_UPPER_LIMIT = 160_000 |
| TASK_MAX_OUTPUT_DEFAULT | `32_000` | src/utils/task/outputFormatting.ts:5 | feature-threshold | export const TASK_MAX_OUTPUT_DEFAULT = 32_000 |
| DEFAULT_MAX_MEMORY | `8 * 1024 * 1024 // 8MB` | src/utils/task/TaskOutput.ts:9 | feature-threshold | const DEFAULT_MAX_MEMORY = 8 * 1024 * 1024 // 8MB |
| POLL_INTERVAL_MS | `1000` | src/utils/task/TaskOutput.ts:10 | timeout | const POLL_INTERVAL_MS = 1000 |
| MAX_PROGRESS_BYTES | `4096` | src/utils/task/TaskOutput.ts:208 | timeout | Only used in pipe mode (hooks). File mode uses the shared poller. / |
| MAX_PROGRESS_LINES | `100` | src/utils/task/TaskOutput.ts:209 | feature-threshold | / |
| MAX_CONTENT_SIZE | `60 * 1024 // 60KB (Honeycomb limit is 64KB, staying safe)` | src/utils/telemetry/betaSessionTracing.ts:70 | file-handling | const MAX_CONTENT_SIZE = 60 * 1024 // 60KB (Honeycomb limit is 64KB, staying safe) |
| SYSTEM_REMINDER_REGEX | `` | src/utils/telemetry/betaSessionTracing.ts:142 | file-handling | Regex to detect content wrapped in <system-reminder> tags |
| DEFAULT_METRICS_EXPORT_INTERVAL_MS | `60000` | src/utils/telemetry/instrumentation.ts:69 | timeout | const DEFAULT_METRICS_EXPORT_INTERVAL_MS = 60000 |
| DEFAULT_LOGS_EXPORT_INTERVAL_MS | `5000` | src/utils/telemetry/instrumentation.ts:70 | timeout | const DEFAULT_LOGS_EXPORT_INTERVAL_MS = 5000 |
| DEFAULT_TRACES_EXPORT_INTERVAL_MS | `5000` | src/utils/telemetry/instrumentation.ts:71 | timeout | const DEFAULT_TRACES_EXPORT_INTERVAL_MS = 5000 |
| MAX_EVENTS | `100_000` | src/utils/telemetry/perfettoTracing.ts:106 | file-handling | does not truncate — it writes the full snapshot). At ~300B/event this is ~30MB, enough trace history for any debugging session. Eviction drops the oldest half when hit, amortized O(1). |
| STALE_SPAN_TTL_MS | `30 * 60 * 1000 // 30 minutes` | src/utils/telemetry/perfettoTracing.ts:121 | timeout | Periodic write interval handle |
| STALE_SPAN_CLEANUP_INTERVAL_MS | `60 * 1000 // 1 minute` | src/utils/telemetry/perfettoTracing.ts:122 | timeout | const STALE_SPAN_CLEANUP_INTERVAL_MS = 60 * 1000 // 1 minute |
| MAX_EVENTS_FOR_TESTING | `MAX_EVENTS` | src/utils/telemetry/perfettoTracing.ts:1117 | analytics | export const MAX_EVENTS_FOR_TESTING = MAX_EVENTS |
| SPAN_TTL_MS | `30 * 60 * 1000 // 30 minutes` | src/utils/telemetry/sessionTracing.ts:79 | timeout | const SPAN_TTL_MS = 30 * 60 * 1000 // 30 minutes |
| MAX_EVENT_PAGES | `50;` | src/utils/teleport.tsx:658 | analytics | Cap is a safety valve against stuck cursors; steady-state is 0–1 pages. |
| TELEPORT_RETRY_DELAYS | `[2000, 4000, 8000, 16000] // 4 retries with exponential backoff` | src/utils/teleport/api.ts:16 | timeout | Retry configuration for teleport API requests |
| MAX_TELEPORT_RETRIES | `TELEPORT_RETRY_DELAYS.length` | src/utils/teleport/api.ts:17 | retry | Retry configuration for teleport API requests |
| DEFAULT_BUNDLE_MAX_BYTES | `100 * 1024 * 1024` | src/utils/teleport/gitBundle.ts:26 | feature-threshold | Tunable via tengu_ccr_bundle_max_bytes. |
| MAX_LINES_TO_SHOW | `3` | src/utils/terminal.ts:7 | ui-rendering | Text rendering utilities for terminal display |
| DEFAULT_TIMEOUT_MS | `120_000 // 2 minutes` | src/utils/timeouts.ts:2 | timeout | Constants for timeout values |
| MAX_TIMEOUT_MS | `600_000 // 10 minutes` | src/utils/timeouts.ts:3 | timeout | Constants for timeout values |
| THRESHOLD | `200_000` | src/utils/tokens.ts:162 | token-management | const THRESHOLD = 200_000 |
| PERSIST_THRESHOLD_OVERRIDE_FLAG | `'tengu_satin_quoll'` | src/utils/toolResultStorage.ts:43 | feature-threshold | Tools absent from the map use the hardcoded fallback. Flag default is {} (no overrides == behavior unchanged). / |
| PREVIEW_SIZE_BYTES | `2000` | src/utils/toolResultStorage.ts:109 | feature-threshold | Preview size in bytes for the reference message |
| TOOL_SCHEMA_CACHE | `new Map<string, CachedSchema>()` | src/utils/toolSchemaCache.ts:18 | token-management | const TOOL_SCHEMA_CACHE = new Map<string, CachedSchema>() |
| SYSTEM_REMINDER_CLOSE | `'</system-reminder>'` | src/utils/transcriptSearch.ts:7 | feature-threshold | const SYSTEM_REMINDER_CLOSE = '</system-reminder>' |
| POLL_INTERVAL_MS | `3000` | src/utils/ultraplan/ccrSession.ts:21 | timeout | const POLL_INTERVAL_MS = 3000 |
| MAX_CONSECUTIVE_FAILURES | `5` | src/utils/ultraplan/ccrSession.ts:24 | timeout | pollRemoteSessionEvents doesn't retry. A 30min poll makes ~600 calls; at any nonzero 5xx rate one blip would kill the run. |
| MAX_WARNING_KEYS | `1000` | src/utils/warningHandler.ts:11 | analytics | Track warnings to avoid spam — bounded to prevent unbounded memory growth |
| MAX_WORKTREE_SLUG_LENGTH | `64` | src/utils/worktree.ts:49 | feature-threshold | const MAX_WORKTREE_SLUG_LENGTH = 64 |
| MAX_VIM_COUNT | `10000` | src/vim/types.ts:182 | feature-threshold | export const MAX_VIM_COUNT = 10000 |

## Why These Values

- MAX_OUTPUT_TOKENS_RECOVERY_LIMIT (src/query.ts:164): This caps automatic continuation attempts after max-output truncation. Doubling it would let longer tool-heavy turns self-recover more often, but it also increases the risk of runaway continuation loops and duplicated work. Halving it would reduce token burn and latency on pathological turns, but would surface more incomplete answers back to the user.
- COMPLETION_THRESHOLD (src/query/tokenBudget.ts:3): At 0.9, the agent keeps working until it has consumed about 90% of a user-specified token budget. Raising it closer to 1.0 would squeeze more work out of the budget, but increases the chance of overrun and low-value filler near the end. Lowering it would stop earlier and feel safer, but it would underdeliver on explicit “use this many tokens” instructions.
- DIMINISHING_THRESHOLD (src/query/tokenBudget.ts:4): The 500-token cutoff is a heuristic for “not enough incremental progress anymore.” Doubling it would stop long-budget runs sooner, which protects cost but may cut off useful finishing work. Halving it would permit more continuations with smaller gains, improving completeness at the cost of efficiency.
- STREAM_IDLE_TIMEOUT_MS (src/services/api/claude.ts:1877): The 90-second default is a watchdog for hung streams. Doubling it would reduce false positives on slow or bursty responses, but users would wait much longer on dead connections. Halving it would make failure detection snappier, but risks aborting legitimate long-pauses in streaming output.
- STALL_THRESHOLD_MS (src/services/api/claude.ts:1936): Thirty seconds marks a “stall” without necessarily killing the stream. Doubling it would reduce warning noise but miss slow-path diagnostics. Halving it would improve observability, but could generate noisy telemetry on otherwise healthy long generations.
- MAX_NON_STREAMING_TOKENS (src/services/api/claude.ts:3354): The 64k cap bounds fallback non-streaming requests. Doubling it would make fallback requests more capable, but much more expensive and slower, and would raise the blast radius when a request hangs. Halving it would keep fallback lighter and safer, but increase truncation risk in recovery paths.
- MAX_HTTP_CONTENT_LENGTH (src/tools/WebFetchTool/utils.ts:112): A 10MB fetch cap is a resource-consumption guardrail. Doubling it would support larger documents and binaries, but increases memory, bandwidth, and tokenization pressure. Halving it would better protect the client from abuse, but would reject more legitimate large pages and docs.
- MAX_CACHE_SIZE_BYTES (src/tools/WebFetchTool/utils.ts:59): A 50MB cache balances repeated fetch performance against resident memory use. Doubling it improves hit rate for repeated browsing sessions, but keeps more content in memory. Halving it reduces memory footprint, but more often re-fetches and re-processes pages.
- MAX_FILE_SIZE_BYTES (src/services/api/filesApi.ts:74): The 500MB Files API limit prevents impractically large uploads/downloads from dominating the session. Doubling it expands supported workloads, but sharply increases retry cost and disk/network pressure. Halving it is safer operationally, but would exclude legitimate large-model and dataset workflows.
- BASE_DELAY_MS (src/services/api/filesApi.ts:73): A 500ms retry base delay is conservative enough to avoid immediate hammering while keeping failures recoverable. Doubling it would reduce burst pressure on the API, but noticeably slow user-visible retries. Halving it would feel faster, but risks synchronized retry storms and more repeated failures.
- DEFAULT_MAX_EXPORT_BATCH_SIZE (src/services/analytics/firstPartyEventLogger.ts:301): A batch size of 200 trades throughput against flush latency and memory. Doubling it improves network efficiency, but increases loss/duplication impact on exporter failure and keeps more events buffered. Halving it reduces burst memory and time-to-export, but increases overhead per event.
- DEFAULT_MAX_QUEUE_SIZE (src/services/analytics/firstPartyEventLogger.ts:302): An 8192-event queue is a backpressure cushion for analytics spikes. Doubling it makes bursts safer but increases memory use and worst-case replay work. Halving it reduces memory cost, but drops events sooner under stress.
- AUTH_REQUEST_TIMEOUT_MS (src/services/mcp/auth.ts:65): Thirty seconds is long enough for metadata discovery and token refresh over imperfect networks without trapping the user indefinitely. Doubling it tolerates slower enterprise networks, but makes OAuth failures feel stuck. Halving it makes auth failures surface faster, but risks breaking legitimate slow providers.
- MAX_PUT_BODY_BYTES (src/services/teamMemorySync/index.ts:89): The 200KB upload cap is explicitly below observed gateway 413 thresholds. Doubling it would reduce the number of split PUTs, but would push more requests into opaque gateway rejections. Halving it would be safer against gateway limits, but would fragment sync into many more sequential uploads.
- MAX_RECONNECT_ATTEMPTS (src/remote/SessionsWebSocket.ts:18): Five reconnect attempts gives transient remote-session failures a chance to heal without looping forever. Doubling it would help on shaky networks, but prolong dead-session recovery and background churn. Halving it would fail fast, but make transient outages much more visible to users.
