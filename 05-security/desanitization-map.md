# Desanitization / API String-Transformation Map

## Scope

This document only covers model/API-facing string rewrites and prompt/transcript fixups.
It excludes unrelated sanitizers for analytics, paths, filenames, headers, etc.

Self-contained reading note:

- file/path references in this document are provenance only
- the transferable artifact is the mapping itself and the classification of why each rewrite exists

## Bottom line

- The only explicit reverse-sanitization table is `DESANITIZATIONS` in the file-edit pipeline.
- The code comments attribute that table to strings being "sanitized in the API", so this is treated as an Anthropic API-side transformation from the client’s perspective.
- The rest split into three buckets:
  - Anthropic API validation/compatibility constraints
  - model-behavior workarounds
  - mixed cases where Anthropic’s server-side rendering creates a pattern that some models learn and reproduce

## 1. Full `DESANITIZATIONS` record

| Sanitized/emitted form | Desanitized target |
| --- | --- |
| `<fnr>` | `<function_results>` |
| `<n>` | `<name>` |
| `</n>` | `</name>` |
| `<o>` | `<output>` |
| `</o>` | `</output>` |
| `<e>` | `<error>` |
| `</e>` | `</error>` |
| `<s>` | `<system>` |
| `</s>` | `</system>` |
| `<r>` | `<result>` |
| `</r>` | `</result>` |
| `< META_START >` | `<META_START>` |
| `< META_END >` | `<META_END>` |
| `< EOT >` | `<EOT>` |
| `< META >` | `<META>` |
| `< SOS >` | `<SOS>` |
| `\n\nH:` | `\n\nHuman:` |
| `\n\nA:` | `\n\nAssistant:` |

### How the table is used

- `desanitizeMatchString()` runs `replaceAll()` for every entry in the table.
- `normalizeFileEditInput()` first tries the model’s `old_string` literally.
- If that fails, it retries with the desanitized version.
- If the desanitized `old_string` matches the file, it applies the exact same replacements to `new_string` before executing the edit.

### Why it exists

Direct code comment summary:

- "Contains replacements to de-sanitize strings from Claude"
- "Since Claude can't see any of these strings (sanitized in the API)"
- "It'll output the sanitized versions in the edit response"

### Classification

- `DESANITIZATIONS`: Anthropic API transformation/sanitization, as explicitly described by the code comment.

## 2. Other places where output/transcript content is fixed up before use

| Location | Transformation | Why the comment says it exists | Classification |
| --- | --- | --- | --- |
| `src/tools/FileEditTool/utils.ts:18-36`, `src/tools/FileEditTool/utils.ts:96-130`, `src/tools/FileEditTool/FileEditTool.ts:470-479` | Curly quotes are normalized to straight quotes for matching: `‘`/`’` -> `'`, `“`/`”` -> `"`; if a match succeeded through quote normalization, the replacement text is converted back to the file’s curly-quote style. | Comment says: "Claude can't output curly quotes", so the tool normalizes them and then preserves the file’s typography. | Model behavior, not API sanitization. |
| `src/tools/FileEditTool/utils.ts:44-63`, `src/tools/FileEditTool/utils.ts:595-612` | Trailing whitespace is stripped from each line of `new_string`, except for `.md`/`.mdx`. | Comment says markdown trailing double-spaces are semantic hard line breaks, so markdown is exempt. The code treats this as formatting normalization when exact matching/edit application would otherwise fail. | Model/edit normalization, not specifically Anthropic API. |
| `src/utils/toolResultStorage.ts:280-294` | Empty tool results are replaced with `(${toolName} completed with no output)`. | Comment says empty `tool_result` at prompt tail can make some models, "notably capybara", emit the `\n\nHuman:` stop sequence because the server renderer leaves a bare `</function_results>\n\n` turn-boundary pattern. | Mixed: Anthropic server rendering plus model behavior. |
| `src/utils/messages.ts:2139-2180` | If a user message contains `tool_reference`, the API-bound copy gets a sibling text block `Tool loaded.`. | Comment says server-side `tool_reference` expansion renders as `<functions>...</functions>`; at prompt tail, capybara samples the stop sequence. A sibling text block forces a clean `\n\nHuman:` boundary. | Mixed: Anthropic server rendering plus model behavior. |
| `src/utils/messages.ts:1910-1931`, `src/utils/messages.ts:1933-1986`, `src/utils/messages.ts:2295-2303` | Text siblings are moved off `tool_reference` messages onto a later non-`tool_reference` tool-result message. This is mostly structural, but it changes which literal text is adjacent to which tool output on the wire. | Comment says `[tool_result + text]` after a `tool_reference` becomes an anomalous second human-turn segment that the model later imitates, causing stop-sequence emission. | Mixed: Anthropic server rendering plus model behavior. |
| `src/utils/messages.ts:1789-1816`, `src/utils/messages.ts:1820-1871`, `src/utils/messages.ts:3097-3099`, `src/utils/messages.ts:2327-2338` | Attachment/system-reminder text is wrapped as `<system-reminder>\n...\n</system-reminder>` and later "smooshed" into adjacent `tool_result.content`; adjacent text is merged with `\n\n`. | Comments say the wrapper makes system-reminder text easy to detect, and the smoosh removes repeated sibling patterns that would otherwise create extra `Human:` teachers. The comment cites A/B results dropping the issue to `0%`. | Mixed prompt-shaping workaround, mostly model behavior induced by wire rendering. |
| `src/utils/messages.ts:1875-1905`, `src/utils/messages.ts:2340-2343`, `src/utils/messages.ts:2524-2553` | For `is_error` tool results, non-text blocks are stripped; remaining text blocks are rejoined with `\n\n`. `smooshIntoToolResult()` also filters non-text additions when `is_error` is true. | Comment says the API rejects `is_error` tool results unless all content blocks are text: `"all content must be type text if is_error is true"`. | Anthropic API validation constraint. |
| `src/utils/messages.ts:1672-1725`, `src/utils/messages.ts:2099-2106` | When tool search is off, `tool_reference` blocks are stripped from `tool_result` content. If nothing remains, the client inserts `[Tool references removed - tool search not enabled]`. | Comment says `tool_reference` is only valid when the tool-search beta is enabled; otherwise the API would error. | Anthropic API/beta compatibility constraint. |
| `src/utils/messages.ts:1535-1612`, `src/utils/messages.ts:2101-2110` | If saved history refers to tools that no longer exist, those `tool_reference` blocks are stripped. If nothing remains, the client inserts `[Tool references removed - tools no longer available]`. | Comment says otherwise the API rejects the request with `"Tool reference not found in available tools"`. | Anthropic API validation/compatibility constraint. |
| `src/utils/messages.ts:4857-4898`, `src/utils/conversationRecovery.ts:197-201` | Assistant messages containing only whitespace text are dropped entirely. | Comment says the API requires non-whitespace text blocks, and this situation happens when the model emits whitespace like `"\n\n"` before a thinking block and the user cancels mid-stream. | Mixed: model behavior plus Anthropic API validation constraint. |
| `src/utils/messages.ts:4925-4968`, `src/utils/messages.ts:2321-2325` | Non-final assistant messages with empty content arrays are replaced with a text block containing `(no content)`. | Comment says the API requires assistant messages to have content except for the optional final assistant prefill message; empty arrays can come from the model returning empty content. | Mixed: model behavior plus Anthropic API validation constraint. |
| `src/utils/messages.ts:411-432`, `src/utils/messages.ts:435-457`, `src/utils/messages.ts:460-507`, `src/constants/messages.ts:1` | Generic message constructors canonicalize empty content to `(no content)` for both assistant and user messages. | Inline comment in `createUserMessage()` says "Make sure we don't send empty messages". | Client-side API hygiene guard, not specific to Anthropic sanitization. |
| `src/utils/messages.ts:1615-1625`, `src/utils/messages.ts:2345-2360` | The API-bound copy of user messages gets `\n[id:...]` appended to the last text block. | Comment says this is so Claude can reference message IDs when using the snip tool, and it only mutates the API-bound copy. | API-bound string mutation, but not a sanitization/filtering workaround. |

## 3. Comments that explicitly explain the why

The strongest explanatory comments are these:

- `src/tools/FileEditTool/utils.ts:527-529`
  - The model "can't see" certain strings because they are "sanitized in the API", so it emits sanitized aliases.
- `src/utils/toolResultStorage.ts:280-286`
  - Empty tool results leave a bare `</function_results>\n\n` pattern, which some models interpret as a turn boundary and stop on.
- `src/utils/messages.ts:1912-1918`
  - `tool_reference` expands to a functions block, and trailing text siblings create a second human-turn segment that the model later imitates.
- `src/utils/messages.ts:2139-2147`
  - `tool_reference` at prompt tail needs a sibling text block because server rendering plus model behavior otherwise hits the stop sequence; text cannot be inserted inside the `tool_result` because that is a server `ValueError`.
- `src/utils/messages.ts:1827-1831`
  - Leaving real user input alone is fine, but removing system-reminder sibling "teachers" fixed the issue in A/B.
- `src/utils/messages.ts:1876-1882`
  - `is_error` plus non-text content creates an unrecoverable transcript that 400s forever on resume.
- `src/utils/messages.ts:1536-1539`
  - Unavailable `tool_reference` blocks cause `"Tool reference not found in available tools"`.
- `src/utils/messages.ts:1673-1675`
  - `tool_reference` blocks are only valid with the tool-search beta.
- `src/utils/messages.ts:4860-4865`
  - Whitespace-only assistant content comes from model behavior during interrupted thinking streams and is invalid for the API.
- `src/utils/messages.ts:4925-4929`
  - Empty assistant content arrays are repaired with a placeholder because the API allows emptiness only for the final assistant prefill.
- `src/tools/FileEditTool/utils.ts:18-20`
  - "Claude can't output curly quotes", so the edit pipeline normalizes them.

## 4. Anthropic API transformation vs model behavior

### Clearly Anthropic API-side transformations/constraints

- `DESANITIZATIONS` table: explicitly described as reversing strings "sanitized in the API".
- `is_error` text-only enforcement.
- stripping `tool_reference` when tool search beta is off.
- stripping unavailable `tool_reference` blocks when referenced tools no longer exist.
- placeholdering or rejecting empty/non-whitespace-invalid messages.

### Clearly model-behavior workarounds

- curly-quote normalization and quote-style preservation in the file edit tool.
- trailing-whitespace normalization in file edits.

### Mixed: Anthropic rendering behavior plus model behavior

- replacing empty tool results with `(${toolName} completed with no output)`.
- injecting `Tool loaded.` after `tool_reference` blocks.
- relocating text siblings away from `tool_reference` messages.
- wrapping/smooshing `<system-reminder>` text into `tool_result.content`.
- dropping whitespace-only assistant messages and repairing empty assistant content when the invalid shape originates from model output.

## 5. Practical conclusion

From the client’s point of view, the code knows about two distinct classes of transformation:

1. Explicit API sanitization aliases that must be reversed for file edits.
2. A wider set of prompt/transcript rewrites added because Anthropic’s wire rendering and validation rules interact badly with specific model behaviors, especially around:
   - `</function_results>`
   - `<functions>...</functions>`
   - `tool_reference`
   - empty tool results
   - empty/whitespace assistant content

The strongest evidence of true API-side content rewriting is the `DESANITIZATIONS` comment itself. The rest are mostly defensive client workarounds around API rendering/validation and model stop-sequence behavior, not evidence of a separate hidden "content filtering" system.
