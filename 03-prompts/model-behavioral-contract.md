# Model Behavioral Contract

This codebase does not define the Claude contract in one place. It emerges from:

- parser expectations
- retry/recovery prompts
- tool-use repair logic
- stop-reason handling

Below is the behavioral contract the code implicitly expects the model to satisfy.

Self-contained reading note:

- the invariant descriptions are the transferred knowledge
- `Evidence:` lines are provenance only
- a reader can ignore the file references and still use this document as the model-behavior contract

## How to read this contract

- Each numbered invariant is a product expectation placed on model output.
- The `What happens when violated` line is the practical consequence and is usually more important than the provenance.
- The later parser inventory is a reference appendix showing which subsystems expect strict JSON, XML, tagged text, or plain text.

## Behavioral Invariants

1. **Tool calls must be expressed as actual `tool_use` blocks; `stop_reason` is not trusted.**
   - Expectation: If the model wants follow-up tool execution, it must emit real `tool_use` content blocks. `stop_reason === "tool_use"` is treated as optional/unreliable metadata.
   - Evidence: `src/query.ts:553-557`, `src/query.ts:829-835`, `src/utils/messages.ts:829-836`
   - What happens when violated: The loop only continues when it sees `tool_use` blocks. A missing block means no tool execution, even if `stop_reason` suggested otherwise.
   - Severity: **Critical**

2. **`tool_use.input` must be valid JSON and must match the tool schema with correct types.**
   - Expectation: Tool input is expected to arrive as parseable JSON, then as a shape that passes Zod validation and tool-specific validation.
   - Evidence: `src/services/api/claude.ts:1997-2007`, `src/services/api/claude.ts:2087-2112`, `src/utils/messages.ts:2651-2715`, `src/services/tools/toolExecution.ts:614-680`, `src/services/tools/toolExecution.ts:682-733`
   - What happens when violated: Input JSON is parsed to `{}` on failure, then rejected with an error `tool_result`; typed mismatches produce `InputValidationError`.
   - Severity: **Critical**

3. **Deferred tools must be loaded before the model emits typed arguments for them.**
   - Expectation: If a deferred tool was not loaded into prompt context, the model should first call `ToolSearch` with `select:<tool_name>`, then retry.
   - Evidence: `src/services/tools/toolExecution.ts:578-596`, `src/services/tools/toolExecution.ts:619-630`
   - What happens when violated: The model tends to emit strings for arrays/numbers/booleans; client-side schema validation rejects the call and sends back a corrective `tool_result`.
   - Severity: **High**

4. **Every `tool_use` id must be unique and must be paired with exactly one matching `tool_result`.**
   - Expectation: The model should produce one unique `tool_use.id` per call, and the next user-side turn should contain a matching `tool_result` for each emitted tool.
   - Evidence: `src/utils/messages.ts:5119-5133`, `src/utils/messages.ts:5139-5160`, `src/utils/messages.ts:5213-5243`, `src/utils/messages.ts:5270-5356`, `src/utils/messages.ts:5437-5442`
   - What happens when violated: The client repairs the transcript by stripping orphaned results, deduping duplicate ids, or injecting synthetic placeholder `tool_result`s. In strict mode it throws instead of repairing.
   - Severity: **Critical**

5. **Server-side tool blocks (`server_tool_use`, `mcp_tool_use`) also require matching result blocks.**
   - Expectation: Server-side tool uses are expected to have corresponding `*_tool_result` blocks.
   - Evidence: `src/utils/messages.ts:5205-5243`
   - What happens when violated: The client strips orphaned server-side tool-use blocks and may replace an emptied assistant message with `[Tool use interrupted]`.
   - Severity: **High**

6. **`tool_result` blocks must be formatted in API-safe order and shape.**
   - Expectation: In a user message, `tool_result` blocks must come first; if sibling text/images/documents exist, they should be folded into `tool_result.content` when legal.
   - Evidence: `src/utils/messages.ts:2467-2482`, `src/utils/messages.ts:2523-2597`, `src/utils/messages.ts:2604-2646`
   - What happens when violated: The normalizer hoists `tool_result` blocks, smooshes siblings, and tries to avoid server-side prompt-shape errors such as `"tool result must follow tool use"`.
   - Severity: **High**

7. **Errored `tool_result`s must contain only text.**
   - Expectation: If `is_error === true`, the model should not mix in images or other non-text blocks.
   - Evidence: `src/utils/messages.ts:1876-1905`, `src/utils/messages.ts:2545-2553`
   - What happens when violated: Non-text blocks are stripped during normalization; otherwise the next API call would 400.
   - Severity: **High**

8. **Tool-search-only format features are conditional: `tool_reference` and `caller` are not universally valid.**
   - Expectation: The model should only rely on `tool_reference` blocks and `tool_use.caller` when the relevant beta/tool-search path is active.
   - Evidence: `src/utils/messages.ts:1536-1600`, `src/utils/messages.ts:1673-1718`, `src/utils/messages.ts:1733-1771`, `src/utils/messages.ts:2526-2543`
   - What happens when violated: `tool_reference` blocks are stripped or relocated; `caller` is stripped from `tool_use`.
   - Severity: **Medium**

9. **Assistant messages must not end with thinking blocks, and orphaned thinking-only messages are invalid.**
   - Expectation: Thinking/redacted thinking may appear during a trajectory, but the final assistant content sent back to the API must not end with them, and split thinking-only remnants should not survive retries/resume.
   - Evidence: `src/query.ts:713-715`, `src/query.ts:924-929`, `src/utils/messages.ts:4778-4794`, `src/utils/messages.ts:4980-5057`
   - What happens when violated: The client tombstones partial streaming messages, strips signature-bearing blocks on retry/fallback, removes orphaned thinking-only messages, or truncates trailing thinking blocks.
   - Severity: **High**

10. **Assistant content must be non-empty and not whitespace-only, except for specific final-turn carve-outs.**
    - Expectation: Normal assistant messages should contain real content, not empty arrays or whitespace-only text.
    - Evidence: `src/utils/messages.ts:4921-4977`, `src/utils/messages.ts:4833-4918`, `src/utils/queryHelpers.ts:83-93`
    - What happens when violated: Empty/whitespace-only messages are filtered or replaced with placeholders. A special carve-out accepts `stop_reason=end_turn` with zero assistant text.
    - Severity: **Medium**

11. **Structured-output mode requires one final `StructuredOutput` tool call with schema-valid JSON.**
    - Expectation: In non-interactive structured-output flows, the model must call `StructuredOutput` exactly once at the end and provide data matching the requested JSON Schema.
    - Evidence: `src/tools/SyntheticOutputTool/SyntheticOutputTool.ts:20-25`, `src/tools/SyntheticOutputTool/SyntheticOutputTool.ts:44-52`, `src/tools/SyntheticOutputTool/SyntheticOutputTool.ts:142-155`, `src/utils/hooks/hookHelpers.ts:41-62`, `src/utils/hooks/hookHelpers.ts:66-80`, `src/QueryEngine.ts:1004-1044`
    - What happens when violated: A stop hook injects `You MUST call the StructuredOutput tool...`; repeated failure eventually terminates with `error_max_structured_output_retries`.
    - Severity: **Critical**

12. **Prompt-hook model responses must be strict JSON objects matching `{ ok: boolean, reason?: string }`.**
    - Expectation: The hook model must return parseable JSON matching the hook schema.
    - Evidence: `src/utils/hooks/execPromptHook.ts:64-99`, `src/utils/hooks/execPromptHook.ts:113-149`
    - What happens when violated: The hook degrades to a non-blocking error attachment instead of enforcing the hook decision.
    - Severity: **High**

13. **Permission-explainer responses must arrive as a forced `tool_use` block with schema-valid input.**
    - Expectation: The side query is forced onto `explain_command`; the model is expected to satisfy that tool call with structured input rather than plain prose.
    - Evidence: `src/utils/permissions/permissionExplainer.ts:177-199`
    - What happens when violated: No parsed explanation is returned; the feature silently falls back to `null`.
    - Severity: **Medium**

14. **The XML classifier must emit `<block>` first, optional `<reason>`, and no preamble.**
    - Expectation: Auto-mode XML classification expects responses like `<block>yes</block><reason>...</reason>` or `<block>no</block>`. Stage 2 may include `<thinking>`, but parsing strips it before reading decision tags.
    - Evidence: `src/utils/permissions/yoloClassifier.ts:573-603`, `src/utils/permissions/yoloClassifier.ts:648-662`, `src/utils/permissions/yoloClassifier.ts:800-848`, `src/utils/permissions/yoloClassifier.ts:886-926`
    - What happens when violated: The classifier treats the result as unparseable and blocks for safety.
    - Severity: **Critical**

15. **When output is cut off by token limits, the model should resume directly mid-thought without apology or recap.**
    - Expectation: Recovery explicitly asks the model to continue from the cut point, avoid apology/recap, and split remaining work into smaller pieces.
    - Evidence: `src/query.ts:1185-1251`, `src/services/api/claude.ts:2266-2291`, `src/services/api/claude.ts:3402-3409`
    - What happens when violated: The system retries up to three recovery turns; after exhaustion it surfaces the error.
    - Severity: **High**

16. **On user rejection/cancellation, the model must stop and wait rather than pretending the tool ran.**
    - Expectation: Rejection and cancellation messages explicitly tell the model to stop, wait, and avoid assuming the tool action succeeded.
    - Evidence: `src/utils/messages.ts:210-215`, `src/utils/messages.ts:622-630`
    - What happens when violated: The transcript now contains a contradictory synthetic `tool_result`; any continued action is behaviorally wrong even if the client does not hard-stop it.
    - Severity: **Critical**

17. **On permission denial, the model may try reasonable alternatives but must not bypass the denial’s intent.**
    - Expectation: The model may switch to naturally adjacent tools, but must not use unrelated capabilities as a covert bypass. If the permission is essential, it should stop and explain.
    - Evidence: `src/utils/messages.ts:223-238`, `src/utils/messages.ts:267-280`
    - What happens when violated: There is no parser repair here; this becomes a behavioral/policy failure.
    - Severity: **High**

18. **On temporary classifier outage, the model should wait briefly, retry, or continue with unrelated work.**
    - Expectation: The corrective message explicitly asks for a short wait-and-retry, with fallback to other tasks that do not require the blocked action.
    - Evidence: `src/utils/messages.ts:284-297`
    - What happens when violated: The model may thrash on the unavailable action instead of making progress elsewhere.
    - Severity: **Medium**

19. **Certain helper prompts expect exact tagged/JSON output, not markdown prose.**
    - Expectation:
      - Skill-improvement detection: JSON array inside `<updates>...</updates>`
      - Skill-improvement apply: full file inside `<updated_file>...</updated_file>`
      - Agent generation: bare JSON object, ideally with no wrapper text
    - Evidence: `src/utils/hooks/skillImprovement.ts:103-124`, `src/utils/hooks/skillImprovement.ts:134-143`, `src/utils/hooks/skillImprovement.ts:215-230`, `src/utils/hooks/skillImprovement.ts:252-259`, `src/components/agents/generateAgent.ts:133-180`, `src/utils/messages.ts:633-686`
    - What happens when violated: The feature returns no update, skips the write, or throws if no JSON object can be extracted.
    - Severity: **Medium**

20. **Some helper prompts expect plain-text outputs with no markdown or commentary.**
    - Expectation:
      - Date/time parsing: only ISO-8601 or exactly `INVALID`
      - Shell-prefix classification: a single prefix token / sentinel like `command_injection_detected`
      - Tool-use summaries / away summaries / issue titles: plain text is consumed directly
    - Evidence: `src/utils/mcp/dateTimeParser.ts:40-47`, `src/utils/mcp/dateTimeParser.ts:82-95`, `src/utils/shell/prefix.ts:248-285`, `src/services/toolUseSummary/toolUseSummaryGenerator.ts:13-17`, `src/services/toolUseSummary/toolUseSummaryGenerator.ts:76-82`, `src/services/awaySummary.ts:17-22`, `src/components/Feedback.tsx:449-468`
    - What happens when violated: These features fall back to null/defaults or misclassify the result.
    - Severity: **Low**

21. **`stop_reason` has special semantics in resume/fork logic: `null` usually means “in progress”, but `end_turn` can mean “done with no visible text.”**
    - Expectation:
      - In-progress assistant messages often have `stop_reason === null`
      - Persisted streaming assistants may still have `null` even after a normal completion
      - `end_turn` with no assistant content is a legitimate terminal state
    - Evidence: `src/services/api/claude.ts:2229-2247`, `src/utils/queryContext.ts:134-140`, `src/utils/conversationRecovery.ts:264-303`, `src/utils/queryHelpers.ts:83-93`
    - What happens when violated: Resume/fork/interruption detection can misclassify turns, inject phantom continuation prompts, or exclude/retain the wrong final assistant message.
    - Severity: **Medium**

## Parser Inventory

These are the model-output parse sites I found, grouped by expected format.

| Location | Parser / extraction | Expected output |
| --- | --- | --- |
| `src/utils/messages.ts:2651-2715` | `safeParseJSON(contentBlock.input)` | `tool_use` / `server_tool_use` JSON input |
| `src/services/tools/toolExecution.ts:614-733` | Zod + custom validation | schema-valid tool inputs |
| `src/utils/hooks/execPromptHook.ts:113-149` | `safeParseJSON` + `hookResponseSchema` | JSON object `{ok, reason?}` |
| `src/utils/permissions/permissionExplainer.ts:177-199` | forced `tool_use`, Zod parse of `toolUseBlock.input` | structured tool payload |
| `src/tools/SyntheticOutputTool/SyntheticOutputTool.ts:142-155` | AJV schema validation | final structured JSON |
| `src/components/agents/generateAgent.ts:172-180` | `jsonParse`, regex fallback `/\{[\s\S]*\}/` | bare JSON object |
| `src/utils/hooks/skillImprovement.ts:123-143` | `extractTag(..., "updates")` + `jsonParse` | `<updates>[...]</updates>` |
| `src/utils/hooks/skillImprovement.ts:230-259` | `extractTag(..., "updated_file")` | `<updated_file>...</updated_file>` |
| `src/utils/permissions/yoloClassifier.ts:578-603` | regex XML extraction | `<block>`, `<reason>`, `<thinking>` |
| `src/utils/sessionTitle.ts:56-117` | `safeParseJSON` + Zod | JSON `{"title": ...}` |
| `src/commands/rename/generateSessionName.ts:20-56` | `safeParseJSON` | JSON `{"name": ...}` |
| `src/utils/teleport.tsx:76-150` | `safeParseJSON` + Zod | JSON `{"title": ..., "branch": ...}` |
| `src/utils/mcp/dateTimeParser.ts:40-95` | text extraction + regex sanity check | ISO-8601 string or `INVALID` |
| `src/utils/shell/prefix.ts:248-285` | first text block | single prefix token / sentinel |
| `src/services/toolUseSummary/toolUseSummaryGenerator.ts:76-82` | text extraction | short plain-text label |
| `src/services/awaySummary.ts:17-22`, `src/services/awaySummary.ts:59-68` | text extraction | 1-3 short sentences |
| `src/components/Feedback.tsx:449-468` | first text block | plain-text issue title |

## Recovery / Retry Messages That Directly Shape Behavior

- `src/query.ts:1224-1228`: resume directly, no apology, no recap, pick up mid-thought, break work smaller.
- `src/utils/messages.ts:210-215`: stop and wait after cancel/reject.
- `src/utils/messages.ts:223-232`: do not maliciously bypass denied permissions; stop and explain if essential.
- `src/utils/messages.ts:277-280`: continue with other tasks after classifier denial when possible.
- `src/utils/messages.ts:293-296`: wait briefly and retry when classifier is unavailable.
- `src/services/tools/toolExecution.ts:592-596`: load deferred tool schema with `ToolSearch` and retry.
- `src/services/tools/toolExecution.ts:1093-1097`: permission-denied hook may explicitly authorize a retry.
