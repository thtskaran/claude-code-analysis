# Deep Extraction — Model-Facing Strings & Contracts

## 1. Tool Input Schemas (Zod Contracts)

  - Field description: "A short (3-5 word) description of the task"
  - Field description: "The task for the agent to perform"
  - Field description: "The type of specialized agent to use for this task"
  - Field description: "Set to true to run this agent in the background. You will be notified when it completes."
  - Field description: "Name for the spawned agent. Makes it addressable via SendMessage({to: name}) while running."
  - Field description: "Team name for spawning. Uses current team context if omitted."
  - Field description: "The ID of the async agent"
  - Field description: "The description of the task"
  - Field description: "The prompt for the agent"
  - Field description: "Path to the output file for checking agent progress"
  - Field description: "Whether the calling agent has Read/Bash tools to check progress"
  - Field description: "The display text for this option that the user will see and select. Should be concise (1-5 words) and clearly describe the choice."
  - Field description: "Explanation of what this option means or what will happen if chosen. Useful for providing context about trade-offs or implications."
  - Field description: "Optional preview content rendered when this option is focused. Use for mockups, code snippets, or visual comparisons that help users compare options. See the tool description for the expected content format."
  - Field description: "Set to true to allow the user to select multiple options instead of just one. Use when choices are not mutually exclusive."
  - Field description: "The preview content of the selected option, if the question used previews."
  - Field description: "Free-text notes the user added to their selection."
  - Field description: "Optional per-question annotations from the user (e.g., notes on preview selections). Keyed by question text."
  - Field description: "User answers collected by the permission component"
  - Field description: "Optional metadata for tracking and analytics purposes. Not displayed to user."
  - Field description: "Questions to ask the user (1-4 questions)"
  - Field description: "The questions that were asked"
  - Field description: "The answers provided by the user (question text -> answer string; multi-select answers are comma-separated)"
  - Field description: "The command to execute"
  - Field description: "Optional timeout in milliseconds (max ${getMaxTimeoutMs()})"
  - Field description: "Set to true to run this command in the background. Use Read to read the output later."
  - Field description: "Set this to true to dangerously override sandbox mode and run commands without sandboxing."
  - Field description: "Internal: pre-computed sed edit result from preview"
  - Field description: "The standard output of the command"
  - Field description: "The standard error output of the command"
  - Field description: "Path to raw output file for large MCP tool outputs"
  - Field description: "Whether the command was interrupted"
  - Field description: "Flag to indicate if stdout contains image data"
  - Field description: "ID of the background task if command is running in background"
  - Field description: "True if the user manually backgrounded the command with Ctrl+B"
  - Field description: "True if assistant-mode auto-backgrounded a long-running blocking command"
  - Field description: "Flag to indicate if sandbox mode was overridden"
  - Field description: "Semantic interpretation for non-error exit codes with special meaning"
  - Field description: "Whether the command is expected to produce no output on success"
  - Field description: "Structured content blocks"
  - Field description: "Path to the persisted full output in tool-results dir (set when output is too large for inline)"
  - Field description: "Total size of the output in bytes (set when output is too large for inline)"
  - Field description: "The message for the user. Supports markdown formatting."
  - Field description: "The message"
  - Field description: "Resolved attachment metadata"
  - Field description: "The new value. Omit to get current value."
  - Field description: "Confirmation that plan mode was entered"
  - Field description: "The tool this prompt applies to"
  - Field description: "The plan content (injected by normalizeToolInput from disk)"
  - Field description: "The plan file path (injected by normalizeToolInput)"
  - Field description: "The plan that was presented to the user"
  - Field description: "The file path where the plan was saved"
  - Field description: "Whether the Agent tool is available in the current context"
  - Field description: "Unique identifier for the plan approval request"
  - Field description: "The absolute path to the file to modify"
  - Field description: "The text to replace"
  - Field description: "Replace all occurrences of old_string (default false)"
  - Field description: "GitHub owner/repo when available"
  - Field description: "The file path that was edited"
  - Field description: "The original string that was replaced"
  - Field description: "The new string that replaced it"
  - Field description: "The original file contents before editing"
  - Field description: "Diff patch showing the changes"
  - Field description: "Whether the user modified the proposed changes"
  - Field description: "Whether all occurrences were replaced"
  - Field description: "The absolute path to the file to read"
  - Field description: "The path to the file that was read"
  - Field description: "The content of the file"
  - Field description: "Number of lines in the returned content"
  - Field description: "The starting line number"
  - Field description: "Total number of lines in the file"
  - Field description: "Base64-encoded image data"
  - Field description: "The MIME type of the image"
  - Field description: "Original file size in bytes"
  - Field description: "Original image width in pixels"
  - Field description: "Original image height in pixels"
  - Field description: "Displayed image width in pixels (after resizing)"
  - Field description: "Displayed image height in pixels (after resizing)"
  - Field description: "Image dimension info for coordinate mapping"
  - Field description: "The path to the notebook file"
  - Field description: "Array of notebook cells"
  - Field description: "The path to the PDF file"
  - Field description: "Base64-encoded PDF data"
  - Field description: "Original file size in bytes"
  - Field description: "The path to the PDF file"
  - Field description: "Original file size in bytes"
  - Field description: "Number of pages extracted"
  - Field description: "Directory containing extracted page images"
  - Field description: "The path to the file"
  - Field description: "The content to write to the file"
  - Field description: "The path to the file that was written"
  - Field description: "The content that was written to the file"
  - Field description: "Diff patch showing the changes"
  - Field description: "The glob pattern to match files against"
  - Field description: "Time taken to execute the search in milliseconds"
  - Field description: "Total number of files found"
  - Field description: "Array of file paths that match the pattern"
  - Field description: "Whether results were truncated (limited to 100 files)"
  - Field description: "Alias for context."
  - Field description: "The LSP operation to perform"
  - Field description: "The absolute or relative path to the file"
  - Field description: "The line number (1-based, as shown in editors)"
  - Field description: "The character offset (1-based, as shown in editors)"
  - Field description: "The LSP operation that was performed"
  - Field description: "The formatted result of the LSP operation"
  - Field description: "The file path the operation was performed on"
  - Field description: "Number of results (definitions, references, symbols)"
  - Field description: "Number of files containing results"
  - Field description: "Optional server name to filter resources by"
  - Field description: "Resource URI"
  - Field description: "Resource name"
  - Field description: "MIME type of the resource"
  - Field description: "Resource description"
  - Field description: "Server that provides this resource"
  - Field description: "MCP tool execution result"
  - Field description: "The new source for the cell"
  - Field description: "The new source code that was written to the cell"
  - Field description: "The ID of the cell that was edited"
  - Field description: "The type of the cell"
  - Field description: "The programming language of the notebook"
  - Field description: "The edit mode that was used"
  - Field description: "Error message if the operation failed"
  - Field description: "The path to the notebook file"
  - Field description: "The original notebook content before modification"
  - Field description: "The updated notebook content after modification"
  - Field description: "The PowerShell command to execute"
  - Field description: "Optional timeout in milliseconds (max ${getMaxTimeoutMs()})"
  - Field description: "Clear, concise description of what this command does in active voice."
  - Field description: "Set to true to run this command in the background. Use Read to read the output later."
  - Field description: "Set this to true to dangerously override sandbox mode and run commands without sandboxing."
  - Field description: "The standard output of the command"
  - Field description: "The standard error output of the command"
  - Field description: "Whether the command was interrupted"
  - Field description: "Semantic interpretation for non-error exit codes with special meaning"
  - Field description: "Flag to indicate if stdout contains image data"
  - Field description: "Path to persisted full output when too large for inline"
  - Field description: "Total output size in bytes when persisted"
  - Field description: "ID of the background task if command is running in background"
  - Field description: "True if the user manually backgrounded the command with Ctrl+B"
  - Field description: "True if the command was auto-backgrounded by the assistant-mode blocking budget"
  - Field description: "The MCP server name"
  - Field description: "The resource URI to read"
  - Field description: "Resource URI"
  - Field description: "MIME type of the content"
  - Field description: "Text content of the resource"
  - Field description: "Path where binary blob content was saved"
  - Field description: "Required for get, update, and run"
  - Field description: "JSON body for create and update"
  - Field description: "The prompt to enqueue at each fire time."
  - Field description: "Job ID returned by CronCreate."
  - Field description: "Plain text message content"
  - Field description: "Optional arguments for the skill"
  - Field description: "Whether the skill is valid"
  - Field description: "The name of the skill"
  - Field description: "Tools allowed by this skill"
  - Field description: "Model override if specified"
  - Field description: "Execution status"
  - Field description: "Whether the skill completed successfully"
  - Field description: "The name of the skill"
  - Field description: "Execution status"
  - Field description: "The ID of the sub-agent that executed the skill"
  - Field description: "The result from the forked skill execution"
  - Field description: "Structured output tool result"
  - Field description: "A brief title for the task"
  - Field description: "What needs to be done"
  - Field description: "Arbitrary metadata to attach to the task"
  - Field description: "The ID of the task to retrieve"
  - Field description: "The task ID to get output from"
  - Field description: "Whether to wait for completion"
  - Field description: "Max wait time in ms"
  - Field description: "The ID of the background task to stop"
  - Field description: "Deprecated: use task_id instead"
  - Field description: "Status message about the operation"
  - Field description: "The ID of the task that was stopped"
  - Field description: "The type of the task that was stopped"
  - Field description: "The command or description of the stopped task"
  - Field description: "The ID of the task to update"
  - Field description: "New subject for the task"
  - Field description: "New description for the task"
  - Field description: "Task IDs that this task blocks"
  - Field description: "Task IDs that block this task"
  - Field description: "New owner for the task"
  - Field description: "Name for the new team to create."
  - Field description: "Team description/purpose."
  - Field description: "The updated todo list"
  - Field description: "The todo list before the update"
  - Field description: "The todo list after the update"
  - Field description: "Maximum number of results to return (default: 5)"
  - Field description: "The URL to fetch content from"
  - Field description: "The prompt to run on the fetched content"
  - Field description: "Size of the fetched content in bytes"
  - Field description: "HTTP response code"
  - Field description: "HTTP response code text"
  - Field description: "Processed result from applying the prompt to the content"
  - Field description: "Time taken to fetch and process the content"
  - Field description: "The URL that was fetched"
  - Field description: "The search query to use"
  - Field description: "Only include search results from these domains"
  - Field description: "Never include search results from these domains"
  - Field description: "The title of the search result"
  - Field description: "The URL of the search result"
  - Field description: "ID of the tool use"
  - Field description: "Array of search hits"
  - Field description: "The search query that was executed"
  - Field description: "Search results and/or text commentary from the model"
  - Field description: "Time taken to complete the search operation"

## 2. Error & Correction Messages (Model-Facing)

### bridge/bridgeMain.ts [Imperative instruction]
```
You must be logged in with a Claude account that has a subscription
  - Run \`claude\` first in the directory to accept the workspace trust dialog
${serverNote}`
  // biome-ignore lint/suspicious/noConsole: intentional help output
  console.
```

### bridge/types.ts [Imperative instruction]
```
You must be logged in to use Remote Control.
```

### commands/commit.ts [Imperative instruction]
```
CRITICAL: ALWAYS create NEW commits.
```

### commands/install-github-app/setupGitHubActions.ts [Permission denial]
```
Permission denied → Run: gh auth refresh -h github.
```

### commands/install-github-app/setupGitHubActions.ts [Permission denial]
```
Permission denied → Run: gh auth refresh -h github.
```

### components/Onboarding.tsx [Imperative instruction]
```
You should always review Claude&apos;s responses, especially when
              <Newline />
              running code.
```

### components/WorkflowMultiselectDialog.tsx [Imperative instruction]
```
You must select at least one workflow to continue</Text></Box>;
    $[9] = showError;
    $[10] = t7;
  } else {
    t7 = $[10];
  }
  let t8;
  if ($[11] !
```

### constants/outputStyles.ts [Imperative instruction]
```
You should generally focus on interesting insights that are specific to the codebase or the code you just wrote, rather than general programming concepts.
```

### constants/outputStyles.ts [Imperative instruction]
```
You should be clear and educational, providing helpful explanations while remaining focused on the task.
```

### constants/outputStyles.ts [Imperative instruction]
```
You should be collaborative and encouraging.
```

### constants/prompts.ts [Imperative instruction]
```
You must NEVER generate or guess URLs for the user unless you are confident that the URLs are for helping the user with programming.
```

### constants/prompts.ts [Imperative instruction]
```
You should defer to user judgement about whether a task is too large to attempt.
```

### constants/prompts.ts [Imperative instruction]
```
you MUST call ${SLEEP_TOOL_NAME}.
```

### main.tsx [Imperative instruction]
```
You must run claude --teleport ${teleport} from a checkout of ${sessionRepo}.
```

### main.tsx [Imperative instruction]
```
You must run claude --teleport ${teleport} from a checkout of ${chalk.
```

### services/SessionMemory/prompts.ts [Imperative instruction]
```
CRITICAL: The session memory file is currently ~${totalTokens} tokens, which exceeds the maximum of ${MAX_TOTAL_SESSION_MEMORY_TOKENS} tokens.
```

### services/api/grove.ts [Imperative instruction]
```
You must run `claude` to review the updated terms.
```

### services/compact/prompt.ts [Imperative instruction]
```
CRITICAL: Respond with TEXT ONLY.
```

### services/tools/StreamingToolExecutor.ts [tool_use_error]
```
Error: No such tool available: ${block.name}
```

### services/tools/StreamingToolExecutor.ts [tool_use_error]
```
Error: Streaming fallback - tool execution discarded
```

### services/tools/toolExecution.ts [tool_use_error]
```
Error: No such tool available: ${toolName}
```

### services/tools/toolExecution.ts [tool_use_error]
```
InputValidationError: ${errorContent}
```

### services/tools/toolExecution.ts [tool_use_error]
```
${isValidCall.message}
```

### skills/bundled/updateConfig.ts [Imperative instruction]
```
CRITICAL: Read Before Write

**Always read the existing settings file before making changes.
```

### state/AppState.tsx [Imperative instruction]
```
You must instead return a property for optimised rendering.
```

### tools/AgentTool/built-in/exploreAgent.ts [Imperative instruction]
```
CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
This is a READ-ONLY exploration task.
```

### tools/AgentTool/built-in/planAgent.ts [Imperative instruction]
```
CRITICAL: READ-ONLY MODE - NO FILE MODIFICATIONS ===
This is a READ-ONLY planning task.
```

### tools/AgentTool/built-in/verificationAgent.ts [Imperative instruction]
```
CRITICAL: This is a VERIFICATION-ONLY task.
```

### tools/AgentTool/prompt.ts [Imperative instruction]
```
you MUST send a single message with multiple ${AGENT_TOOL_NAME} tool use content blocks.
```

### tools/BashTool/prompt.ts [Imperative instruction]
```
CRITICAL: Always create NEW commits rather than amending, unless the user explicitly requests a git amend.
```

### tools/BashTool/prompt.ts [Imperative instruction]
```
You should always default to running commands within the sandbox.
```

### tools/BashTool/readOnlyValidation.ts [Imperative instruction]
```
CRITICAL: date positional args in format MMDDhhmm[[CC]YY][.
```

### tools/EnterPlanModeTool/EnterPlanModeTool.ts [Imperative instruction]
```
You should now focus on exploring the codebase and designing an implementation approach.
```

### tools/FileEditTool/prompt.ts [Imperative instruction]
```
You must use your \`${FILE_READ_TOOL_NAME}\` tool at least once in the conversation before editing.
```

### tools/FileReadTool/FileReadTool.ts [Imperative instruction]
```
you MUST refuse to improve or augment the code.
```

### tools/FileReadTool/prompt.ts [Imperative instruction]
```
you MUST provide the pages parameter to read specific page ranges (e.
```

### tools/NotebookEditTool/NotebookEditTool.ts [is_error result]
```
Updated cell ${cell_id} with ${new_source}
```

### tools/WebFetchTool/prompt.ts [Imperative instruction]
```
You should then make a new WebFetch request with the redirect URL to fetch the content.
```

### utils/agenticSessionSearch.ts [Imperative instruction]
```
CRITICAL: Be VERY inclusive in your matching.
```

### utils/api.ts [Imperative instruction]
```
You should not respond to this context unless it is highly relevant to your task.
```

### utils/bash/heredoc.ts [Imperative instruction]
```
CRITICAL: We do this BEFORE the closingLineIndex === -1 check.
```

### utils/claudeInChrome/prompt.ts [Imperative instruction]
```
You must ALWAYS:
* Capture extra frames before and after taking actions to ensure smooth playback
* Name the file meaningfully to help the user identify it later (e.
```

### utils/claudeInChrome/prompt.ts [Imperative instruction]
```
you MUST first load them using ToolSearch.
```

### utils/claudemd.ts [Imperative instruction]
```
you MUST follow them exactly as written.
```

### utils/mcpOutputStorage.ts [Imperative instruction]
```
you MUST explicitly describe what portion of the content you have read.
```

### utils/mcpOutputStorage.ts [Imperative instruction]
```
you MUST explicitly state this.
```

### utils/messages.ts [STOP instruction]
```
STOP what you are doing and wait for the user to tell you how to proceed.
```

### utils/messages.ts [STOP instruction]
```
STOP what you are doing and wait for the user to tell you how to proceed.
```

### utils/messages.ts [Imperative instruction]
```
You should only try to work around this restriction in reasonable ways that do not attempt to bypass the intent behind this denial.
```

### utils/messages.ts [Imperative instruction]
```
You should create your plan at ${attachment.
```

### utils/messages.ts [Imperative instruction]
```
you MUST NOT make any edits (with the exception of the plan file mentioned below), run any non-readonly tools (including changing configs or making commits), or otherwise make any changes to the system.
```

### utils/messages.ts [Imperative instruction]
```
You should build your plan incrementally by writing to or editing this file.
```

### utils/messages.ts [Imperative instruction]
```
You should create your plan at ${attachment.
```

### utils/messages.ts [Imperative instruction]
```
you MUST NOT make any edits (with the exception of the plan file mentioned below), run any non-readonly tools (including changing configs or making commits), or otherwise make any changes to the system.
```

### utils/messages.ts [Imperative instruction]
```
You should create your plan at ${attachment.
```

### utils/messages.ts [Imperative instruction]
```
you MUST NOT make any edits, run any non-readonly tools (including changing configs or making commits), or otherwise make any changes to the system.
```

### utils/messages.ts [Imperative instruction]
```
You should build your plan incrementally by writing to or editing this file.
```

### utils/messages.ts [Imperative instruction]
```
You must not share secrets (e.
```

### utils/messages.ts [Imperative instruction]
```
You should ask clarifying questions when the approach is ambiguous rather than making assumptions.
```

### utils/sideQuestion.ts [Imperative instruction]
```
You must answer this question directly in a single response.
```

### utils/swarm/teammatePromptAddendum.ts [Imperative instruction]
```
you MUST use the SendMessage tool.
```

### utils/telemetry/sessionTracing.ts [Imperative instruction]
```
you MUST pass the specific span
 *   to ensure responses are attached to the correct request.
```

### utils/teleport.tsx [Imperative instruction]
```
You should keep it short and simple, ideally no more than 6 words.
```

### utils/teleport.tsx [Imperative instruction]
```
You should keep it short and simple, ideally no more than 4 words.
```

### utils/teleport.tsx [Imperative instruction]
```
You must run claude --teleport ${sessionId} from a checkout of ${notInRepoDisplay}.
```

### utils/teleport.tsx [Imperative instruction]
```
You must run claude --teleport ${sessionId} from a checkout of ${chalk.
```

### utils/teleport.tsx [Imperative instruction]
```
You must run claude --teleport ${sessionId} from a checkout of ${sessionDisplay}.
```

### utils/teleport.tsx [Imperative instruction]
```
You must run claude --teleport ${sessionId} from a checkout of ${chalk.
```

### utils/tmuxSocket.ts [Imperative instruction]
```
CRITICAL: This value is used by Shell.
```

### utils/toolResultStorage.ts [Permission denial]
```
Permission denied: ${nodeError.
```


## 3. System-Reminder Injection Patterns

- **Tool.ts**: system-reminder
- **bridge/replBridgeTransport.ts**: system-reminder
- **cli/print.ts**: <system, </system
- **commands/brief.ts**: <system, </system
- **commands/ultraplan.tsx**: <system, system-reminder
- **components/ContextVisualization.tsx**: <collapsed
- **components/VirtualMessageList.tsx**: <system, system-reminder
- **components/messageActions.tsx**: </system, <system
- **constants/prompts.ts**: <user-prompt, <system
- **memdir/memoryAge.ts**: <system, system-reminder, </system
- **services/tools/StreamingToolExecutor.ts**: <tool_use_error
- **services/tools/toolExecution.ts**: <tool_use_error
- **services/vcr.ts**: </system, system-reminder
- **tools/AgentTool/prompt.ts**: <system
- **tools/FileEditTool/utils.ts**: <function_results, <system, </system
- **tools/FileReadTool/FileReadTool.ts**: <system, </system
- **tools/FileReadTool/UI.tsx**: <tool_use_error
- **tools/SkillTool/prompt.ts**: system-reminder
- **tools/ToolSearchTool/prompt.ts**: system-reminder, <system
- **types/logs.ts**: <collapsed
- **utils/api.ts**: <system, </system
- **utils/attachments.ts**: <system
- **utils/hooks.ts**: system-reminder
- **utils/messages.ts**: <system, system-reminder, <collapsed, </system
- **utils/queryHelpers.ts**: system-reminder, <system
- **utils/sideQuestion.ts**: <system, </system, system-reminder
- **utils/telemetry/betaSessionTracing.ts**: <system, system-reminder
- **utils/transcriptSearch.ts**: </system, system-reminder, <system

## 4. Behavioral Constants & Magic Strings

### constants/apiLimits.ts
- `API_IMAGE_MAX_BASE64_SIZE` = `5 * 1024 * 1024 // 5 MB`
- `IMAGE_TARGET_RAW_SIZE` = `(API_IMAGE_MAX_BASE64_SIZE * 3) / 4 // 3.75 MB`
- `IMAGE_MAX_WIDTH` = `2000`
- `IMAGE_MAX_HEIGHT` = `2000`
- `PDF_TARGET_RAW_SIZE` = `20 * 1024 * 1024 // 20 MB`
- `API_PDF_MAX_PAGES` = `100`
- `PDF_EXTRACT_SIZE_THRESHOLD` = `3 * 1024 * 1024 // 3 MB`
- `PDF_MAX_EXTRACT_SIZE` = `100 * 1024 * 1024 // 100 MB`
- `PDF_MAX_PAGES_PER_READ` = `20`
- `PDF_AT_MENTION_INLINE_THRESHOLD` = `10`
- `API_MAX_MEDIA_PER_REQUEST` = `100`

### constants/betas.ts
- `CLAUDE_CODE_20250219_BETA_HEADER` = `'claude-code-20250219'`
- `INTERLEAVED_THINKING_BETA_HEADER` = `'interleaved-thinking-2025-05-14'`
- `CONTEXT_1M_BETA_HEADER` = `'context-1m-2025-08-07'`
- `CONTEXT_MANAGEMENT_BETA_HEADER` = `'context-management-2025-06-27'`
- `STRUCTURED_OUTPUTS_BETA_HEADER` = `'structured-outputs-2025-12-15'`
- `WEB_SEARCH_BETA_HEADER` = `'web-search-2025-03-05'`
- `TOOL_SEARCH_BETA_HEADER_1P` = `'advanced-tool-use-2025-11-20'`
- `TOOL_SEARCH_BETA_HEADER_3P` = `'tool-search-tool-2025-10-19'`
- `EFFORT_BETA_HEADER` = `'effort-2025-11-24'`
- `TASK_BUDGETS_BETA_HEADER` = `'task-budgets-2026-03-13'`
- `PROMPT_CACHING_SCOPE_BETA_HEADER` = `'prompt-caching-scope-2026-01-05'`
- `FAST_MODE_BETA_HEADER` = `'fast-mode-2026-02-01'`
- `REDACT_THINKING_BETA_HEADER` = `'redact-thinking-2026-02-12'`
- `TOKEN_EFFICIENT_TOOLS_BETA_HEADER` = `'token-efficient-tools-2026-03-28'`
- `SUMMARIZE_CONNECTOR_TEXT_BETA_HEADER` = `feature('CONNECTOR_TEXT')`
- `AFK_MODE_BETA_HEADER` = `feature('TRANSCRIPT_CLASSIFIER')`
- `CLI_INTERNAL_BETA_HEADER` = `process.env.USER_TYPE === 'ant' ? 'cli-internal-2026-02-09' : ''`
- `ADVISOR_BETA_HEADER` = `'advisor-tool-2026-03-01'`
- `BEDROCK_EXTRA_PARAMS_HEADERS` = `new Set([`
- `VERTEX_COUNT_TOKENS_ALLOWED_BETAS` = `new Set([`

### constants/common.ts
- `getSessionStartDate` = `memoize(getLocalISODate)`

### constants/cyberRiskInstruction.ts
- `CYBER_RISK_INSTRUCTION` = ``IMPORTANT: Assist with authorized security testing, defensive security, CTF challenges, and educational contexts. Refuse requests for destructive techniques, DoS attacks, mass targeting, supply chain`

### constants/errorIds.ts
- `E_TOOL_USE_SUMMARY_GENERATION_FAILED` = `344`

### constants/figures.ts
- `BLACK_CIRCLE` = `env.platform === 'darwin' ? '⏺' : '●'`
- `BULLET_OPERATOR` = `'∙'`
- `TEARDROP_ASTERISK` = `'✻'`
- `UP_ARROW` = `'\u2191' // ↑ - used for opus 1m merge notice`
- `DOWN_ARROW` = `'\u2193' // ↓ - used for scroll hint`
- `LIGHTNING_BOLT` = `'↯' // \u21af - used for fast mode indicator`
- `EFFORT_LOW` = `'○' // \u25cb - effort level: low`
- `EFFORT_MEDIUM` = `'◐' // \u25d0 - effort level: medium`
- `EFFORT_HIGH` = `'●' // \u25cf - effort level: high`
- `EFFORT_MAX` = `'◉' // \u25c9 - effort level: max (Opus 4.6 only)`
- `PLAY_ICON` = `'\u25b6' // ▶`
- `PAUSE_ICON` = `'\u23f8' // ⏸`
- `REFRESH_ARROW` = `'\u21bb' // ↻ - used for resource update indicator`
- `CHANNEL_ARROW` = `'\u2190' // ← - inbound channel message indicator`
- `INJECTED_ARROW` = `'\u2192' // → - cross-session injected message indicator`
- `FORK_GLYPH` = `'\u2442' // ⑂ - fork directive indicator`
- `DIAMOND_OPEN` = `'\u25c7' // ◇ - running`
- `DIAMOND_FILLED` = `'\u25c6' // ◆ - completed/failed`
- `REFERENCE_MARK` = `'\u203b' // ※ - komejirushi, away-summary recap marker`
- `FLAG_ICON` = `'\u2691' // ⚑ - used for issue flag banner`
- `BLOCKQUOTE_BAR` = `'\u258e' // ▎ - left one-quarter block, used as blockquote line prefix`
- `HEAVY_HORIZONTAL` = `'\u2501' // ━ - heavy box-drawing horizontal`
- `BRIDGE_SPINNER_FRAMES` = `[`
- `BRIDGE_READY_INDICATOR` = `'\u00b7\u2714\ufe0e\u00b7'`
- `BRIDGE_FAILED_INDICATOR` = `'\u00d7'`

### constants/files.ts
- `BINARY_EXTENSIONS` = `new Set([`

### constants/github-app.ts
- `PR_TITLE` = `'Add Claude Code GitHub Workflow'`
- `GITHUB_ACTION_SETUP_DOCS_URL` = `'https://github.com/anthropics/claude-code-action/blob/main/docs/setup.md'`
- `WORKFLOW_CONTENT` = ``name: Claude Code`
- `PR_BODY` = ``## 🤖 Installing Claude Code GitHub App`
- `CODE_REVIEW_PLUGIN_WORKFLOW_CONTENT` = ``name: Claude Code Review`

### constants/keys.ts

### constants/messages.ts
- `NO_CONTENT_MESSAGE` = `'(no content)'`

### constants/oauth.ts
- `CLAUDE_AI_INFERENCE_SCOPE` = `'user:inference' as const`
- `CLAUDE_AI_PROFILE_SCOPE` = `'user:profile' as const`
- `OAUTH_BETA_HEADER` = `'oauth-2025-04-20' as const`
- `CONSOLE_OAUTH_SCOPES` = `[`
- `CLAUDE_AI_OAUTH_SCOPES` = `[`
- `ALL_OAUTH_SCOPES` = `Array.from(`
- `MCP_CLIENT_METADATA_URL` = `'https://claude.ai/oauth/claude-code-client-metadata'`

### constants/outputStyles.ts
- `DEFAULT_OUTPUT_STYLE_NAME` = `'default'`
- `getAllOutputStyles` = `memoize(async function getAllOutputStyles(`

### constants/product.ts
- `PRODUCT_URL` = `'https://claude.com/claude-code'`
- `CLAUDE_AI_BASE_URL` = `'https://claude.ai'`
- `CLAUDE_AI_STAGING_BASE_URL` = `'https://claude-ai.staging.ant.dev'`
- `CLAUDE_AI_LOCAL_BASE_URL` = `'http://localhost:4000'`

### constants/prompts.ts
- `CLAUDE_CODE_DOCS_MAP_URL` = `'https://code.claude.com/docs/en/claude_code_docs_map.md'`
- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` = `'__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'`
- `DEFAULT_AGENT_PROMPT` = ``You are an agent for Claude Code, Anthropic's official CLI for Claude. Given the user's message, you should use the tools available to complete the task. Complete the task fully—don't gold-plate, but`

### constants/spinnerVerbs.ts
- `SPINNER_VERBS` = `[`

### constants/system.ts

### constants/systemPromptSections.ts

### constants/toolLimits.ts
- `DEFAULT_MAX_RESULT_SIZE_CHARS` = `50_000`
- `MAX_TOOL_RESULT_TOKENS` = `100_000`
- `BYTES_PER_TOKEN` = `4`
- `MAX_TOOL_RESULT_BYTES` = `MAX_TOOL_RESULT_TOKENS * BYTES_PER_TOKEN`
- `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` = `200_000`
- `TOOL_SUMMARY_MAX_LENGTH` = `50`

### constants/tools.ts
- `ALL_AGENT_DISALLOWED_TOOLS` = `new Set([`
- `CUSTOM_AGENT_DISALLOWED_TOOLS` = `new Set([`
- `ASYNC_AGENT_ALLOWED_TOOLS` = `new Set([`
- `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS` = `new Set([`
- `COORDINATOR_MODE_ALLOWED_TOOLS` = `new Set([`

### constants/turnCompletionVerbs.ts
- `TURN_COMPLETION_VERBS` = `[`

### constants/xml.ts
- `COMMAND_NAME_TAG` = `'command-name'`
- `COMMAND_MESSAGE_TAG` = `'command-message'`
- `COMMAND_ARGS_TAG` = `'command-args'`
- `BASH_INPUT_TAG` = `'bash-input'`
- `BASH_STDOUT_TAG` = `'bash-stdout'`
- `BASH_STDERR_TAG` = `'bash-stderr'`
- `LOCAL_COMMAND_STDOUT_TAG` = `'local-command-stdout'`
- `LOCAL_COMMAND_STDERR_TAG` = `'local-command-stderr'`
- `LOCAL_COMMAND_CAVEAT_TAG` = `'local-command-caveat'`
- `TERMINAL_OUTPUT_TAGS` = `[`
- `TICK_TAG` = `'tick'`
- `TASK_NOTIFICATION_TAG` = `'task-notification'`
- `TASK_ID_TAG` = `'task-id'`
- `TOOL_USE_ID_TAG` = `'tool-use-id'`
- `TASK_TYPE_TAG` = `'task-type'`
- `OUTPUT_FILE_TAG` = `'output-file'`
- `STATUS_TAG` = `'status'`
- `SUMMARY_TAG` = `'summary'`
- `REASON_TAG` = `'reason'`
- `WORKTREE_TAG` = `'worktree'`
- `WORKTREE_PATH_TAG` = `'worktreePath'`
- `WORKTREE_BRANCH_TAG` = `'worktreeBranch'`
- `ULTRAPLAN_TAG` = `'ultraplan'`
- `REMOTE_REVIEW_TAG` = `'remote-review'`
- `REMOTE_REVIEW_PROGRESS_TAG` = `'remote-review-progress'`
- `TEAMMATE_MESSAGE_TAG` = `'teammate-message'`
- `CHANNEL_MESSAGE_TAG` = `'channel-message'`
- `CHANNEL_TAG` = `'channel'`
- `CROSS_SESSION_MESSAGE_TAG` = `'cross-session-message'`
- `FORK_BOILERPLATE_TAG` = `'fork-boilerplate'`
- `FORK_DIRECTIVE_PREFIX` = `'Your directive: '`
- `COMMON_HELP_ARGS` = `['help', '-h', '--help']`
- `COMMON_INFO_ARGS` = `[`


## 5. Recovery & Continuation Messages

### QueryEngine.ts
```
resume fails with "
```

### QueryEngine.ts
```
resume after
    // kill-mid-request.
```

### QueryEngine.ts
```
pick up updated messages and
    // model (from slash commands).
```

### QueryEngine.ts
```
resume loads full pre-compact history.
```

### Tool.ts
```
ContinueAnimation?: true
    showSpinner?: boolean
    isLocalJSXCommand?: boolean
    isImmediate?: boolean
    /** Set to true to clear a local JSX command (e.
```

### Tool.ts
```
resumeAgentBackground threads one reconstructed from sidechain records.
```

### bridge/bridgeApi.ts
```
resume),
              // send it back so the backend reattaches instead of creating
              // a new env.
```

### bridge/bridgeMain.ts
```
continue
      }
      try {
        await api.
```

### bridge/bridgeMain.ts
```
resume message (resume is impossible after env expiry/auth
  // failure/sustained connection errors).
```

### bridge/bridgeMain.ts
```
continue
      }

      // At capacity — we polled to keep the heartbeat alive, but cannot
      // accept new work right now.
```

### bridge/bridgeMain.ts
```
continue
      }

      // Decode the work secret for session spawning and to extract the JWT
      // used for the ack call below.
```

### bridge/bridgeMain.ts
```
continue
      }

      // Explicitly acknowledge after committing to handle the work — NOT
      // before.
```

### bridge/bridgeMain.ts
```
continue
                }
                logger.
```

### bridge/bridgeMain.ts
```
resume command a lie — deregister deletes Firestore + Redis stream.
```

### bridge/bridgeMain.ts
```
resume is impossible in those cases and the message would contradict the
  // error already printed.
```

### bridge/bridgeMain.ts
```
Resume this session by running \`
```

### bridge/bridgeMain.ts
```
resume of session ${initialSessionId}`
```

### bridge/bridgeMain.ts
```
Resume an existing session instead of creating a new one.
```

### bridge/bridgeMain.ts
```
Resume the last session in this directory (reads bridge-pointer.
```

### bridge/bridgeMain.ts
```
continueSession: boolean
  help: boolean
  error?: string
}

const SPAWN_FLAG_VALUES = ['
```

### bridge/bridgeMain.ts
```
continueSession = false

  for (let i = 0; i < args.
```

### bridge/bridgeMain.ts
```
continueSession = true
    } else if (arg === '
```

### bridge/bridgeMain.ts
```
continue resume a specific session on its original
  // environment; incompatible with spawn-related flags (which configure
  // fresh session creation), and mutually exclusive with each other.
```

### bridge/bridgeMain.ts
```
continueSession) &&
    (spawnMode !== undefined ||
      capacity !== undefined ||
      createSessionInDir !== undefined)
  ) {
    return makeError(
      `
```

### bridge/bridgeMain.ts
```
continue cannot be used with --spawn, --capacity, or --create-session-in-dir.
```

### bridge/bridgeMain.ts
```
continueSession) {
    return makeError(`
```

### bridge/bridgeMain.ts
```
continue cannot be used together.
```

### bridge/bridgeMain.ts
```
continueSession,
    help,
  }

  function makeError(error: string): ParsedArgs {
    return {
      verbose,
      sandbox,
      debugFile,
      sessionTimeoutMs,
      permissionMode,
      name,
      spawnMode,
      capacity,
      createSessionInDir,
      sessionId,
      continueSession,
      help,
      error,
    }
  }
}

async function printHelp(): Promise<void> {
  // Use EXTERNAL_P
```

### bridge/bridgeMain.ts
```
continue                   Resume the last session in this directory
  --session-id <id>                Resume a specific session by ID (cannot be
                                   used with spawn flags or --continue)
`
```

### bridge/bridgeMain.ts
```
continueSession,
  } = parsed
  // Mutable so --continue can set it from the pointer file.
```

### bridge/bridgeMain.ts
```
resume flow below then treats it the same as an explicit --session-id.
```

### bridge/bridgeMain.ts
```
resumeSessionId = parsedSessionId
  // When --continue found a pointer, this is the directory it came from
  // (may be a worktree sibling, not `
```

### bridge/bridgeMain.ts
```
resume-flow deterministic
  // failure, clear THIS file so --continue doesn'
```

### bridge/bridgeMain.ts
```
resumePointerDir: string | undefined

  const usedMultiSessionFeature =
    parsedSpawnMode !== undefined ||
    parsedCapacity !== undefined ||
    parsedCreateSessionInDir !== undefined

  // Validate permission mode early so the user gets an error before
  // the bridge starts polling for work.
```

### bridge/bridgeMain.ts
```
pick up where you left off on any device.
```

### bridge/bridgeMain.ts
```
continue: resolve the most recent session from the crash-recovery
  // pointer and chain into the #20460 --session-id flow.
```

### bridge/bridgeMain.ts
```
continueSession is always false in external
  // builds, so this block tree-shakes.
```

### bridge/bridgeMain.ts
```
continueSession) {
    const { readBridgePointerAcrossWorktrees } = await import(
      '
```

### bridge/bridgeMain.ts
```
resumeSessionId = pointer.
```

### bridge/bridgeMain.ts
```
continue
    // would keep hitting the same dead session.
```

### bridge/bridgeMain.ts
```
resumePointerDir = pointerDir
  }

  // In production, baseUrl is the Anthropic API (from OAuth config).
```

### bridge/bridgeMain.ts
```
resumeSessionId &&
    process.
```

### bridge/bridgeMain.ts
```
resume > explicit --spawn > saved project pref > gate default
  // - resuming via --continue / --session-id: always single-session (resume
  //   targets one specific session in its original directory)
  // - explicit --spawn flag: use that value directly (does not persist)
  // - saved ProjectConfig.
```

### bridge/bridgeMain.ts
```
resumeSessionId) {
    spawnMode = '
```

### bridge/bridgeMain.ts
```
ResumeSessionId guard at the creation site handles the
  // resume case (skip creation when resume succeeded; fall through to
  // fresh creation on env-mismatch fallback).
```

### bridge/bridgeMain.ts
```
continue: a leftover pointer means the previous run didn'
```

### bridge/bridgeMain.ts
```
resumeSessionId) {
    const { clearBridgePointer } = await import('
```

### bridge/bridgeMain.ts
```
resumeSessionId is always
  // undefined here in external builds — this guard is for tree-shaking.
```

### bridge/bridgeMain.ts
```
resumeSessionId) {
    try {
      validateBridgeId(resumeSessionId, '
```

### bridge/bridgeMain.ts
```
resumeSessionId}"
```

### bridge/bridgeMain.ts
```
resumeSessionId, {
      baseUrl,
      getAccessToken: getBridgeAccessToken,
    })
    if (!session) {
      // Session gone on server → pointer is stale.
```

### bridge/bridgeMain.ts
```
resumePointerDir may be a worktree sibling — clear THAT file.
```

### bridge/bridgeMain.ts
```
resumePointerDir) {
        const { clearBridgePointer } = await import('
```

### bridge/bridgeMain.ts
```
resumePointerDir)
      }
      // biome-ignore lint/suspicious/noConsole: intentional error output
      console.
```

### bridge/bridgeMain.ts
```
resumeSessionId} not found.
```

### bridge/bridgeMain.ts
```
resumePointerDir) {
        const { clearBridgePointer } = await import('
```

### bridge/bridgeMain.ts
```
resumePointerDir)
      }
      // biome-ignore lint/suspicious/noConsole: intentional error output
      console.
```

### bridge/bridgeMain.ts
```
resumeSessionId} has no environment_id.
```

### bridge/bridgeMain.ts
```
resumeSessionId} on environment ${reuseEnvironmentId}`
```

### bridge/bridgeMain.ts
```
resume flow completed successfully.
```

### bridge/bridgeMain.ts
```
ResumeSessionId: string | undefined
  if (feature('
```

### bridge/bridgeMain.ts
```
resumeSessionId) {
    if (reuseEnvironmentId && environmentId !== reuseEnvironmentId) {
      // Backend returned a different environment_id — the original env
      // expired or was reaped.
```

### bridge/bridgeMain.ts
```
resume env mismatch: requested ${reuseEnvironmentId}, backend returned ${environmentId}.
```

### bridge/bridgeMain.ts
```
resume session ${resumeSessionId} — its environment has expired.
```

### bridge/bridgeMain.ts
```
ResumeSessionId stays undefined → fresh session path below.
```

### bridge/bridgeMain.ts
```
ResumeId = toInfraSessionId(resumeSessionId)
      const reconnectCandidates =
        infraResumeId === resumeSessionId
          ? [resumeSessionId]
          : [resumeSessionId, infraResumeId]
      let reconnected = false
      let lastReconnectErr: unknown
      for (const candidateId of reconnectCandidates) {
        try {
          await api.
```

### bridge/bridgeMain.ts
```
ResumeSessionId = resumeSessionId
          reconnected = true
          break
        } catch (err) {
          lastReconnectErr = err
          logForDebugging(
            `
```

### bridge/bridgeMain.ts
```
resumePointerDir && isFatal) {
          const { clearBridgePointer } = await import('
```

### bridge/bridgeMain.ts
```
resumePointerDir)
        }
        // biome-ignore lint/suspicious/noConsole: intentional error output
        console.
```

### bridge/bridgeMain.ts
```
resumeSessionId}: ${errorMessage(err)}\nThe session may still be resumable — try running the same command again.
```

### bridge/bridgeMain.ts
```
resume()
    process.
```

### bridge/bridgeMain.ts
```
resume succeeded, skip creation entirely — the
  // session already exists and bridge/reconnect has re-queued it.
```

### bridge/bridgeMain.ts
```
resume was requested but failed on env mismatch, effectiveResumeSessionId
  // is undefined, so we fall through to fresh session creation (honoring the
  // "
```

### bridge/bridgeMain.ts
```
ResumeSessionId
      ? effectiveResumeSessionId
      : null
  if (preCreateSession && !(feature('
```

### bridge/bridgeMain.ts
```
ResumeSessionId)) {
    const { createBridgeSession } = await import('
```

### bridge/bridgeMain.ts
```
resumed ones (so a second crash after resume is still recoverable).
```

### bridge/bridgeMain.ts
```
continue forces single-session mode on resume,
  // so a pointer written in multi-session mode would contradict the user'
```

### bridge/bridgeMain.ts
```
pick up a token on next cycle.
```

### bridge/bridgePointer.ts
```
continue falls back to current-dir-only.
```

### bridge/bridgePointer.ts
```
resume via the --session-id flow from #20460.
```

### bridge/bridgePointer.ts
```
resume reconnects to the right env regardless of which worktree
  // --continue was invoked from.
```

### bridge/bridgeStatusUtil.ts
```
Continue coding in the Claude app or ${url}`
```

### bridge/bridgeUI.ts
```
continue
      }
      const width = stringWidth(logical)
      count += Math.
```

### bridge/bridgeUI.ts
```
pick up the new values.
```

### bridge/createSession.ts
```
resume) and title.
```

### bridge/initReplBridge.ts
```
continue
        const rawContent = getContentText(msg.
```

### bridge/initReplBridge.ts
```
continue
        const derived = deriveTitle(rawContent)
        if (!derived) continue
        title = derived
        hasTitle = true
        break
      }
    }
  }

  // Shared by both v1 and v2 — fires on every title-worthy user message until
  // it returns true.
```

### bridge/remoteBridgeCore.ts
```
resumes from the old
  // transport'
```

### bridge/replBridge.ts
```
resumes polling to get a fresh ingress token and reconnect.
```

### bridge/replBridge.ts
```
resume where this one left off, not replay from
    // the last transport-swap checkpoint.
```

### bridge/replBridge.ts
```
resumes the right session.
```

### bridge/replBridge.ts
```
resumes the stream instead of replaying from seq 0.
```

### bridge/replBridge.ts
```
continue during reconnection (they use getSessionIngressAuthToken()
        // independently of WS state).
```

### bridge/replBridge.ts
```
continues polling — the server will dispatch a new
 * work item if the ingress WebSocket drops, allowing automatic
 * reconnection without tearing down the bridge.
```

### bridge/replBridge.ts
```
continue
            }
          }
          // At-capacity sleep — reached by both the legacy path (heartbeat
          // disabled) and the heartbeat-backoff path (needsBackoff=true).
```

### bridge/replBridge.ts
```
continue
      }

      // Decode before type dispatch — need the JWT for the explicit ack.
```

### bridge/replBridge.ts
```
continue
      }

      // Explicitly acknowledge to prevent redelivery.
```

### bridge/replBridge.ts
```
continue
      }

      if (work.
```

### bridge/replBridge.ts
```
continue
        }

        onWorkReceived(
          workSessionId,
          secret.
```

### bridge/replBridge.ts
```
continue
        }

        environmentRecreations++
        logForDebugging(
          `
```

### bridge/replBridge.ts
```
continue
        }

        onStateChange?.
```

### bridge/replBridgeTransport.ts
```
resume from where the old one left off (otherwise the server replays
   * the entire session history from seq 0).
```

### bridge/replBridgeTransport.ts
```
resumes from where
   * the old stream left off.
```

### bridge/replBridgeTransport.ts
```
continues after the 409
      // branch, so callers see the logged warning and a false return.
```

### bridge/sessionRunner.ts
```
continue
        const b = block as Record<string, unknown>

        if (b.
```

### bridge/types.ts
```
resume a session after the original bridge died.
```

### buddy/prompt.ts
```
continue
    if (msg.
```

### buddy/prompt.ts
```
continue
    if (msg.
```

### cli/handlers/agents.ts
```
continue

    lines.
```

### cli/handlers/auth.ts
```
continue
      }
      hasAuthProperty = true
      if (prop.
```

### cli/handlers/plugins.ts
```
continue

      // Find loading errors for this plugin
      const pluginName = parsePluginIdentifier(pluginId).
```

### cli/handlers/plugins.ts
```
continue

    // Find loading errors for this plugin
    const pluginName = parsePluginIdentifier(pluginId).
```

### cli/print.ts
```
Resume,
  type TurnInterruptionState,
} from '
```

### cli/print.ts
```
continue: boolean | undefined
    resume: string | boolean | undefined
    resumeSessionAt: string | undefined
    verbose: boolean | undefined
    outputFormat: string | undefined
    jsonSchema: Record<string, unknown> | undefined
    permissionPromptToolName: string | undefined
    allowedTools: string[] | undefined
    thinkingConfig: ThinkingConfig | undefined
    maxTurns: number | undefined
```

### cli/print.ts
```
resumeSessionAt && !options.
```

### cli/print.ts
```
resume) {
    process.
```

### cli/print.ts
```
resume-session-at requires --resume\n`
```

### cli/print.ts
```
resume) {
    process.
```

### cli/print.ts
```
resumedAgentSetting,
  } = await loadInitialMessages(setAppState, {
    continue: options.
```

### cli/print.ts
```
continue,
    teleport: options.
```

### cli/print.ts
```
resume,
    resumeSessionAt: options.
```

### cli/print.ts
```
resumeSessionAt,
    forkSession: options.
```

### cli/print.ts
```
resumed session (if not overridden by current --agent flag
  // or settings-based agent, which would already have set mainThreadAgentType in main.
```

### cli/print.ts
```
resumedAgentSetting) {
    const { agentDefinition: restoredAgent } = restoreAgentFromSession(
      resumedAgentSetting,
      undefined,
      { activeAgents: agents, allAgents: agents },
    )
    if (restoredAgent) {
      setAppState(prev => ({ .
```

### cli/print.ts
```
resumes maintain the agent
      saveAgentSetting(restoredAgent.
```

### cli/print.ts
```
ResumeSessionId =
    typeof options.
```

### cli/print.ts
```
resume)) || options.
```

### cli/print.ts
```
ResumeSessionId && !isUsingSdkUrl) {
    process.
```

### cli/print.ts
```
resume interrupted turns on restart so CC continues from where it
  // left off without requiring the SDK to re-send the prompt.
```

### cli/print.ts
```
resumeInterruptedTurnEnv =
    process.
```

### cli/print.ts
```
RESUME_INTERRUPTED_TURN
  if (
    turnInterruptionState &&
    turnInterruptionState.
```

### cli/print.ts
```
resumeInterruptedTurnEnv
  ) {
    logForDebugging(
      `
```

### cli/print.ts
```
Continue from where you left off.
```

### cli/print.ts
```
continue
      }
      // Skip SDK MCP servers — elicitation flows through SdkControlClientTransport
      if (connection.
```

### cli/print.ts
```
continue
      }
      const serverName = connection.
```

### cli/print.ts
```
continue -- fall through to ask() so the model processes the result
          }

          const input = command.
```

### cli/print.ts
```
continue with shutdown anyway
      }
      suggestionState.
```

### cli/print.ts
```
resume logic pre-enqueued a command, drain it now
          // that initialize has set up systemPrompt, agents, hooks, etc.
```

### cli/print.ts
```
resume before first turn completes — no snapshot yet):
          // rebuild from scratch.
```

### cli/print.ts
```
continue
      } else if (message.
```

### cli/print.ts
```
continue
      } else if (message.
```

### cli/print.ts
```
continue
      } else if (message.
```

### cli/print.ts
```
continue
      } else if (message.
```

### cli/print.ts
```
continue
      }
      // After handling control, keep-alive, env-var, assistant, and system
      // messages above, only user messages should remain.
```

### cli/print.ts
```
continue
      }

      // First prompt message implicitly initializes if not already done.
```

### cli/print.ts
```
continue
        }

        // Track this UUID to prevent runtime duplicates
        trackReceivedMessageUuid(message.
```

### cli/print.ts
```
continue: boolean | undefined
    teleport: string | true | null | undefined
    resume: string | boolean | undefined
    resumeSessionAt: string | undefined
    forkSession: boolean | undefined
    outputFormat: string | undefined
    sessionStartHooksPromise?: ReturnType<typeof processSessionStartHooks>
    restoredWorkerState: Promise<SessionExternalMetadata | null>
  },
): Promise<LoadInitialM
```

### cli/print.ts
```
continue) {
    try {
      logEvent('
```

### cli/print.ts
```
Resume(
        undefined /* sessionId */,
        undefined /* file path */,
      )
      if (result) {
        // Match coordinator mode to the resumed session'
```

### cli/print.ts
```
resumed session
        if (feature('
```

### cli/print.ts
```
Resume,
        teleportResumeCodeSession,
        validateGitState,
      } = await import('
```

### cli/print.ts
```
ResumeCodeSession(options.
```

### cli/print.ts
```
Resume(
          teleportResult.
```

### cli/print.ts
```
resume in print mode (accepts session ID or URL)
  // URLs are [ANT-ONLY]
  if (options.
```

### cli/print.ts
```
resume) {
    try {
      logEvent('
```

### cli/print.ts
```
resume requires a valid session ID when used with --print.
```

### cli/print.ts
```
resume <session-id>'
```

### cli/print.ts
```
Resume(
        parsedSessionId.
```

### cli/print.ts
```
Resume returns {messages: []} not null.
```

### cli/print.ts
```
resume, start with empty session (it was hydrated but empty)
        if (
          parsedSessionId.
```

### cli/print.ts
```
resumeSessionAt feature
      if (options.
```

### cli/print.ts
```
resumeSessionAt) {
        const index = result.
```

### cli/print.ts
```
resumeSessionAt,
        )
        if (index < 0) {
          emitLoadError(
            `
```

### cli/print.ts
```
resumeSessionAt}`
```

### cli/print.ts
```
resumed session
      if (feature('
```

### cli/print.ts
```
resume session: ${error.
```

### cli/print.ts
```
resume session with --print mode'
```

### cli/print.ts
```
continue with no prior session falls through
  // here with sessionStartHooksPromise undefined because main.
```

### cli/print.ts
```
continue)
  return {
    messages: await (options.
```

### cli/print.ts
```
continue
    const scopedConfig = toScopedConfig(config)

    // SDK servers are managed by the SDK process, not the CLI.
```

### cli/print.ts
```
continue
    }

    try {
      const client = await connectToServer(name, scopedConfig)
      newClients.
```

### cli/transports/SSETransport.ts
```
continue

    const frame: SSEFrame = {}
    let isComment = false

    for (const line of rawFrame.
```

### cli/transports/SSETransport.ts
```
continue
      }

      const colonIdx = line.
```

### cli/transports/SSETransport.ts
```
continue

      const field = line.
```

### cli/transports/SSETransport.ts
```
resumes from the right point instead of replaying everything.
```

### cli/transports/SerialBatchEventUploader.ts
```
continue

        try {
          await this.
```

### cli/transports/SerialBatchEventUploader.ts
```
continue
          }
          // Re-queue the failed batch at the front.
```

### cli/transports/SerialBatchEventUploader.ts
```
continue
        }

        // Release backpressure waiters if space opened up
        this.
```

### cli/transports/SerialBatchEventUploader.ts
```
continue
      }
      if (count > 0 && bytes + itemBytes > maxBatchBytes) break
      bytes += itemBytes
      count++
    }
    return this.
```

### cli/transports/WebSocketTransport.ts
```
pick up a new session token).
```

### cli/transports/ccrClient.ts
```
continue to startHeartbeat(), leaking a
      // 20s timer against a dead epoch.
```

### cli/transports/ccrClient.ts
```
continue
      }

      if (response.
```

### commands/branch/branch.ts
```
resume
    const now = new Date()
    const firstPrompt = deriveFirstPrompt(
      serializedMessages.
```

### commands/branch/branch.ts
```
resume show the same session name
    // Always add "
```

### commands/branch/branch.ts
```
Resume into the fork
    const titleInfo = title ? `
```

### commands/branch/branch.ts
```
resume the original: claude -r ${originalSessionId}`
```

### commands/branch/branch.ts
```
resume) {
      await context.
```

### commands/branch/branch.ts
```
resume(sessionId, forkLog, '
```

### commands/branch/branch.ts
```
resume not available
      onDone(
        `
```

### commands/branch/branch.ts
```
Resume with: /resume ${sessionId}`
```

### commands/bridge/bridge.tsx
```
Continue() {
      onDone(undefined, {
        display: "
```

### commands/bridge/bridge.tsx
```
Continue = t5;
  let t6;
  let t7;
  if ($[10] === Symbol.
```

### commands/bridge/bridge.tsx
```
Continue || $[14] !== handleDisconnect) {
    t8 = {
      "
```

### commands/bridge/bridge.tsx
```
Continue();
          }
        }
      }
    };
    $[12] = focusIndex;
    $[13] = handleContinue;
    $[14] = handleDisconnect;
    $[15] = t8;
  } else {
    t8 = $[15];
  }
  let t9;
  if ($[16] === Symbol.
```

### commands/bridge/bridge.tsx
```
Continue || $[19] !== qrText || $[20] !== showQR) {
    const qrLines = qrText ? qrText.
```

### commands/bridge/bridge.tsx
```
Continue;
    t16 = true;
    T0 = Box;
    t10 = "
```

### commands/bridge/bridge.tsx
```
Continue;
    $[19] = qrText;
    $[20] = showQR;
    $[21] = T0;
    $[22] = T1;
    $[23] = t10;
    $[24] = t11;
    $[25] = t12;
    $[26] = t13;
    $[27] = t14;
    $[28] = t15;
    $[29] = t16;
  } else {
    T0 = $[21];
    T1 = $[22];
    t10 = $[23];
    t11 = $[24];
    t12 = $[25];
    t13 = $[26];
    t14 = $[27];
    t15 = $[28];
    t16 = $[29];
  }
  const t17 = focusIndex === 0;
 
```

### commands/bridge/bridge.tsx
```
Continue</Text>;
    $[40] = t25;
  } else {
    t25 = $[40];
  }
  let t26;
  if ($[41] !== t24) {
    t26 = <ListItem isFocused={t24}>{t25}</ListItem>;
    $[41] = t24;
    $[42] = t26;
  } else {
    t26 = $[42];
  }
  let t27;
  if ($[43] !== t19 || $[44] !== t23 || $[45] !== t26) {
    t27 = <Box flexDirection="
```

### commands/bridge/bridge.tsx
```
continue</Text>;
    $[47] = t28;
  } else {
    t28 = $[47];
  }
  let t29;
  if ($[48] !== T0 || $[49] !== t10 || $[50] !== t11 || $[51] !== t12 || $[52] !== t13 || $[53] !== t27) {
    t29 = <T0 flexDirection={t10} gap={t11}>{t12}{t13}{t27}{t28}</T0>;
    $[48] = T0;
    $[49] = t10;
    $[50] = t11;
    $[51] = t12;
    $[52] = t13;
    $[53] = t27;
    $[54] = t29;
  } else {
    t29 = $[54];
```

### commands/clear/caches.ts
```
resume/--continue, which are NOT compaction events.
```

### commands/clear/conversation.ts
```
continue
      if (isLocalAgentTask(task)) {
        preservedAgentIds.
```

### commands/clear/conversation.ts
```
resume after /clear
  if (feature('
```

### commands/clear/conversation.ts
```
continue
        }
        // Foreground task: kill it and drop from state
        try {
          if (task.
```

### commands/clear/conversation.ts
```
continue
    void initTaskOutputAsSymlink(
      task.
```

### commands/clear/conversation.ts
```
resume
  // knows what the new post-clear session was in.
```

### commands/copy/copy.tsx
```
continue;
    const content = (msg as AssistantMessage).
```

### commands/copy/copy.tsx
```
continue;
    const text = extractTextContent(content, '
```

### commands/desktop/index.ts
```
Continue the current session in Claude Desktop'
```

### commands/extra-usage/extra-usage-core.ts
```
continue — the create endpoint will enforce if necessary
    }

    try {
      const pendingOrDismissedRequests = await getMyAdminRequests(
        '
```

### commands/extra-usage/index.ts
```
keep working when limits are hit'
```

### commands/extra-usage/index.ts
```
keep working when limits are hit'
```

### commands/insights.ts
```
continue
        const meta = logToSessionMeta(log)
        allMetas.
```

### commands/install-github-app/ChooseRepoStep.tsx
```
continue</Text></Box>;
    $[41] = showEmptyError;
    $[42] = t18;
  } else {
    t18 = $[42];
  }
  const t19 = currentRepo ? "
```

### commands/install-github-app/OAuthFlowStep.tsx
```
continue after brief delay to show success
        const timer2 = setTimeout(onSuccess_0, 1000, accessToken);
        timersRef_0.
```

### commands/install-github-app/WarningsStep.tsx
```
Continue: () => void;
}
export function WarningsStep(t0) {
  const $ = _c(8);
  const {
    warnings,
    onContinue
  } = t0;
  let t1;
  if ($[0] === Symbol.
```

### commands/install-github-app/WarningsStep.tsx
```
Continue, t1);
  let t2;
  if ($[1] === Symbol.
```

### commands/install-github-app/WarningsStep.tsx
```
continue anyway</Text></Box>;
    $[1] = t2;
  } else {
    t2 = $[1];
  }
  let t3;
  if ($[2] !== warnings) {
    t3 = warnings.
```

### commands/install-github-app/WarningsStep.tsx
```
continue anyway, or Ctrl+C to exit and fix issues</Text></Box>;
    $[4] = t4;
  } else {
    t4 = $[4];
  }
  let t5;
  if ($[5] === Symbol.
```

### commands/install-github-app/install-github-app.tsx
```
Continue={handleSubmit} />;
    case '
```

### commands/install.tsx
```
Continue despite cleanup errors - native install already succeeded
        }

        // Clean up old shell aliases
        const aliasMessages = await cleanupShellAliases();
        if (aliasMessages.
```

### commands/plugin/BrowseMarketplace.tsx
```
continue;
          installablePlugins.
```

### commands/plugin/ManageMarketplaces.tsx
```
continue;
        }

        // Handle update
        if (state.
```

### commands/plugin/ManagePlugins.tsx
```
continue;
      }
      const existing_0 = orphanErrorsBySource.
```

### commands/plugin/ManagePlugins.tsx
```
continue;
      const parsed = parsePluginIdentifier(pluginId_0);
      const pluginName_0 = parsed.
```

### commands/plugin/ManagePlugins.tsx
```
continue;
      if (client_1.
```

### commands/plugin/ManagePlugins.tsx
```
continue;
      standaloneMcps.
```

### commands/plugin/PluginOptionsDialog.tsx
```
continue;
    }
    if (schema?.
```

### commands/plugin/PluginOptionsDialog.tsx
```
continue;
      const num = Number(value);
      finalValues[fieldKey] = Number.
```

### commands/plugin/PluginOptionsDialog.tsx
```
continue</Text>;
    $[54] = currentFieldIndex;
    $[55] = fields.
```

### commands/plugin/PluginSettings.tsx
```
continue;
    shownMarketplaceNames.
```

### commands/plugin/PluginSettings.tsx
```
continue;
    shownMarketplaceNames.
```

### commands/plugin/PluginSettings.tsx
```
continue;
    if (pluginName) shownPluginNames.
```

### commands/plugin/PluginSettings.tsx
```
continue;
    const updates: Record<string, unknown> = {};

    // Remove from extraKnownMarketplaces
    if (settings.
```

### commands/rate-limit-options/rate-limit-options.tsx
```
continue with extra usage"
```

### commands/resume/index.ts
```
resume: Command = {
  type: '
```

### commands/resume/index.ts
```
Resume a previous conversation'
```

### commands/resume/resume.tsx
```
ResumeEntrypoint } from '
```

### commands/resume/resume.tsx
```
ResumeResult = {
  resultType: '
```

### commands/resume/resume.tsx
```
resumeHelpMessage(result: ResumeResult): string {
  switch (result.
```

### commands/resume/resume.tsx
```
resume to pick a specific session.
```

### commands/resume/resume.tsx
```
ResumeError(t0) {
  const $ = _c(10);
  const {
    message,
    args,
    onDone
  } = t0;
  let t1;
  let t2;
  if ($[0] !== onDone) {
    t1 = () => {
      const timer = setTimeout(onDone, 0);
      return () => clearTimeout(timer);
    };
    t2 = [onDone];
    $[0] = onDone;
    $[1] = t1;
    $[2] = t2;
  } else {
    t1 = $[1];
    t2 = $[2];
  }
  React.
```

### commands/resume/resume.tsx
```
resume {args}</Text>;
    $[3] = args;
    $[4] = t3;
  } else {
    t3 = $[4];
  }
  let t4;
  if ($[5] !== message) {
    t4 = <MessageResponse><Text>{message}</Text></MessageResponse>;
    $[5] = message;
    $[6] = t4;
  } else {
    t4 = $[6];
  }
  let t5;
  if ($[7] !== t3 || $[8] !== t4) {
    t5 = <Box flexDirection="
```

### commands/resume/resume.tsx
```
ResumeCommand({
  onDone,
  onResume
}: {
  onDone: (result?: string, options?: {
    display?: CommandResultDisplay;
  }) => void;
  onResume: (sessionId: UUID, log: LogOption, entrypoint: ResumeEntrypoint) => Promise<void>;
}): React.
```

### commands/resume/resume.tsx
```
resume conversation'
```

### commands/resume/resume.tsx
```
Resume(fullLog, showAllProjects, worktreePaths);
    if (crossProjectCheck.
```

### commands/resume/resume.tsx
```
resume directly
        setResuming(true);
        void onResume(sessionId, fullLog, '
```

### commands/resume/resume.tsx
```
resume
    setResuming(true);
    void onResume(sessionId, fullLog, '
```

### commands/resume/resume.tsx
```
Resume cancelled'
```

### commands/resume/resume.tsx
```
Resume = async (sessionId: UUID, log: LogOption, entrypoint: ResumeEntrypoint) => {
    try {
      await context.
```

### commands/resume/resume.tsx
```
resume: ${(error as Error).
```

### commands/resume/resume.tsx
```
ResumeCommand key={Date.
```

### commands/resume/resume.tsx
```
Resume={onResume} />;
  }

  // Load logs to search (includes same-repo worktrees)
  const worktreePaths = await getWorktreePaths(getOriginalCwd());
  const logs = await loadSameRepoMessageLogs(worktreePaths);
  if (logs.
```

### commands/resume/resume.tsx
```
ResumeError message={message} args={arg} onDone={() => onDone(message)} />;
  }

  // First, check if arg is a valid UUID
  const maybeSessionId = validateUuid(arg);
  if (maybeSessionId) {
    const matchingLogs = logs.
```

### commands/resume/resume.tsx
```
Resume(maybeSessionId, fullLog, '
```

### commands/resume/resume.tsx
```
Resume(maybeSessionId, directLog, '
```

### commands/resume/resume.tsx
```
Resume(sessionId, fullLog, '
```

### commands/resume/resume.tsx
```
resumeHelpMessage({
        resultType: '
```

### commands/resume/resume.tsx
```
ResumeError message={message} args={arg} onDone={() => onDone(message)} />;
    }
  }

  // No match found - show error
  const message = resumeHelpMessage({
    resultType: '
```

### commands/tag/tag.tsx
```
resume and can be searched with /.
```

### commands/ultraplan.tsx
```
continue working — when the ${DIAMOND_OPEN} fills, press ↓ to view results`
```

### commands.ts
```
ResumeEntrypoint,
} from '
```

### commands.ts
```
resume,
  session,
  skills,
  stats,
  status,
  statusline,
  stickers,
  tag,
  theme,
  feedback,
  review,
  ultrareview,
  rewind,
  securityReview,
  terminalSetup,
  upgrade,
  extraUsage,
  extraUsageNonInteractive,
  rateLimitOptions,
  usage,
  usageReport,
  vim,
  .
```

### components/ClaudeInChromeOnboarding.tsx
```
continue
import { Box, Link, Newline, Text, useInput } from '
```

### components/ConsoleOAuthFlow.tsx
```
continue on success state
  useKeybinding('
```

### components/ConsoleOAuthFlow.tsx
```
continue from platform setup
  useKeybinding('
```

### components/ConsoleOAuthFlow.tsx
```
continue…</Text></>;
          $[39] = mode;
          $[40] = oauthStatus.
```

### components/DesktopHandoff.tsx
```
continue…</Text>;
      $[9] = t5;
    } else {
      t5 = $[9];
    }
    let t6;
    if ($[10] !== t4) {
      t6 = <Box flexDirection="
```

### components/FeedbackSurvey/useMemorySurvey.tsx
```
continue;
    }
    const content = message.
```

### components/FeedbackSurvey/useMemorySurvey.tsx
```
continue;
    }
    for (const block of content) {
      if (block.
```

### components/FeedbackSurvey/useMemorySurvey.tsx
```
continue;
      }
      const input = block.
```

### components/FullscreenLayout.tsx
```
resume (scroll back to bottom) so the "
```

### components/FullscreenLayout.tsx
```
continue;
    // Tool-use-only assistant entries aren'
```

### components/FullscreenLayout.tsx
```
continue;
    const isAssistant = m.
```

### components/GlobalSearchDialog.tsx
```
continue;
      }
      const rel = relativePath(cwd, m_1.
```

### components/IdeOnboardingDialog.tsx
```
continue</Text></Box>;
    $[20] = t15;
  } else {
    t15 = $[20];
  }
  let t16;
  if ($[21] !== t14) {
    t16 = <>{t14}{t15}</>;
    $[21] = t14;
    $[22] = t16;
  } else {
    t16 = $[22];
  }
  return t16;
}
export function hasIdeOnboardingDialogBeenShown(): boolean {
  const config = getGlobalConfig();
  const terminal = envDynamic.
```

### components/IdleReturnDialog.tsx
```
Continue this conversation"
```

### components/InvalidSettingsDialog.tsx
```
Continue: () => void;
  onExit: () => void;
};

/**
 * Dialog shown when settings files have validation errors.
```

### components/InvalidSettingsDialog.tsx
```
continue (skipping invalid files) or exit to fix them.
```

### components/InvalidSettingsDialog.tsx
```
Continue,
    onExit
  } = t0;
  let t1;
  if ($[0] !== onContinue || $[1] !== onExit) {
    t1 = function handleSelect(value) {
      if (value === "
```

### components/InvalidSettingsDialog.tsx
```
Continue();
      }
    };
    $[0] = onContinue;
    $[1] = onExit;
    $[2] = t1;
  } else {
    t1 = $[2];
  }
  const handleSelect = t1;
  let t2;
  if ($[3] !== settingsErrors) {
    t2 = <ValidationErrorsList errors={settingsErrors} />;
    $[3] = settingsErrors;
    $[4] = t2;
  } else {
    t2 = $[4];
  }
  let t3;
  if ($[5] === Symbol.
```

### components/InvalidSettingsDialog.tsx
```
Continue without these settings"
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled = t3;
  const isDeepSearchEnabled = false;
  const [themeName] = useTheme();
  let t4;
  if ($[1] !== themeName) {
    t4 = getTheme(themeName);
    $[1] = themeName;
    $[2] = t4;
  } else {
    t4 = $[2];
  }
  const theme = t4;
  let t5;
  if ($[3] !== theme.
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled) {
    let t22;
    if ($[24] !== logs) {
      t22 = logs.
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled) {
      let t30;
      if ($[65] === Symbol.
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled) {
      let t31;
      if ($[72] === Symbol.
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled || !focusedLog) {
        return "
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled && onLogsChanged) {
          onLogsChanged();
        }
      }
      setViewMode("
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled && treeNodes.
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled && displayedLogs.
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled, treeNodes, displayedLogs];
    $[112] = agenticSearchState.
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled) {
    let t57;
    if ($[160] === Symbol.
```

### components/LogSelector.tsx
```
Resume Session{viewMode === "
```

### components/LogSelector.tsx
```
ResumeWithRenameEnabled ? <TreeSelect nodes={treeNodes} onSelect={node_0 => {
      onSelect(node_0.
```

### components/LogoV2/ChannelsNotice.tsx
```
continue;
    }
    if (!installedPluginIds.
```

### components/MCPServerApprovalDialog.tsx
```
Continue without using this MCP server"
```

### components/MarkdownTable.tsx
```
continue;
          lines_2.
```

### components/MessageRow.tsx
```
continue;
      }
      if (content?.
```

### components/MessageRow.tsx
```
continue;
        }
        // Non-collapsible tool uses appear in syntheticStreamingToolUseMessages
        // before their ID is added to inProgressToolUseIDs.
```

### components/MessageRow.tsx
```
continue;
        }
      }
      return true;
    }
    if (msg?.
```

### components/MessageRow.tsx
```
continue;
    }
    // Tool results arrive while the collapsed group is still being built
    if (msg?.
```

### components/MessageRow.tsx
```
continue;
      }
    }
    // Collapsible grouped_tool_use messages arrive transiently before being
    // merged into the current collapsed group on the next render cycle
    if (msg?.
```

### components/MessageRow.tsx
```
continue;
      }
    }
    return true;
  }
  return false;
}
function MessageRowImpl(t0) {
  const $ = _c(64);
  const {
    message: msg,
    isUserContinuation,
    hasContentAfter,
    tools,
    commands,
    verbose,
    inProgressToolUseIDs,
    streamingToolUseIDs,
    screen,
    canAnimate,
    onOpenRateLimitOptions,
    lastThinkingBlockId,
    latestBashOutputUUID,
    columns,
    i
```

### components/MessageSelector.tsx
```
continue;
    }
    const result = msg.
```

### components/MessageSelector.tsx
```
continue;
    }
    if (!filesChanged.
```

### components/MessageSelector.tsx
```
continue;
    }
  }
  return {
    filesChanged,
    insertions,
    deletions
  };
}
export function selectableUserMessagesFilter(message: Message): message is UserMessage {
  if (message.
```

### components/MessageSelector.tsx
```
continue;

    // Skip known non-meaningful message types
    if (isSyntheticMessage(msg)) continue;
    if (isToolUseResultMessage(msg)) continue;
    if (msg.
```

### components/MessageSelector.tsx
```
continue;
    if (msg.
```

### components/MessageSelector.tsx
```
continue;
    if (msg.
```

### components/MessageSelector.tsx
```
continue;
    if (msg.
```

### components/MessageSelector.tsx
```
continue;

    // Assistant with actual content = meaningful
    if (msg.
```

### components/MessageSelector.tsx
```
continue;
    }

    // User messages that aren'
```

### components/Messages.tsx
```
continue;
    }
    if (msg.
```

### components/Messages.tsx
```
ContinueAnimation?: true;
  } | null;
  toolUseConfirmQueue: ToolUseConfirm[];
  inProgressToolUseIDs: Set<string>;
  isMessageSelectorVisible: boolean;
  conversationId: string;
  screen: Screen;
  streamingToolUses: StreamingToolUse[];
  showAllInTranscript?: boolean;
  agentDefinitions?: AgentDefinitionsResult;
  onOpenRateLimitOptions?: () => void;
  /** Hide the logo/header - used for subagen
```

### components/Messages.tsx
```
ContinueAnimation) && !toolUseConfirmQueue.
```

### components/Messages.tsx
```
continue;
    if (prev[key] !== next[key]) {
      if (key === '
```

### components/Messages.tsx
```
continue;
        }
      }
      if (key === '
```

### components/Messages.tsx
```
continue;
        }
      }
      if (key === '
```

### components/Messages.tsx
```
continue;
        }
      }
      if (key === '
```

### components/Messages.tsx
```
continue;
        }
      }
      // streamingThinking changes frequently - always re-render when it changes
      // (no special handling needed, default behavior is correct)
      return false;
    }
  }
  return true;
});
export function shouldRenderStatically(message: RenderableMessage, streamingToolUseIDs: Set<string>, inProgressToolUseIDs: Set<string>, siblingToolUseIDs: ReadonlySet<string>,
```

### components/Onboarding.tsx
```
Continue />
    </Box>;
  const preflightStep = <PreflightStep onSuccess={goToNextStep} />;
  // Create the steps array - determine which steps to include based on reAuth and oauthEnabled
  const apiKeyNeedingApproval = useMemo(() => {
    // Add API key step if needed
    // On homespace, ANTHROPIC_API_KEY is preserved in process.
```

### components/Onboarding.tsx
```
Continue = useCallback(() => {
    if (currentStepIndex === steps.
```

### components/Onboarding.tsx
```
Continue
  }, {
    context: '
```

### components/PressEnterToContinue.tsx
```
Continue() {
  const $ = _c(1);
  let t0;
  if ($[0] === Symbol.
```

### components/PromptInput/PromptInput.tsx
```
continue/--resume).
```

### components/PromptInput/PromptInput.tsx
```
continue/--resume scenarios where we need to avoid ID collisions.
```

### components/RemoteCallout.tsx
```
pick up where you
            left off on any device.
```

### components/ResumeTask.tsx
```
ResumeTask({
  onSelect,
  onCancel,
  isEmbedded = false
}: Props): React.
```

### components/ResumeTask.tsx
```
continue with regular teleport flow
      return;
    }
  });
  const handleErrorComplete = useCallback(() => {
    setHasCompletedTeleportErrorFlow(true);
    void loadSessions();
  }, [setHasCompletedTeleportErrorFlow, loadSessions]);

  // Show error dialog if needed
  if (!hasCompletedTeleportErrorFlow) {
    return <TeleportError onComplete={handleErrorComplete} />;
  }
  if (loading) {
    r
```

### components/ResumeTask.tsx
```
resume
        {showScrollPosition && <Text dimColor>
            {'
```

### components/ScrollKeybindingHandler.tsx
```
resume in the same direction decays to mult≈1 (1 row).
```

### components/ScrollKeybindingHandler.tsx
```
continue scrolling while stationary.
```

### components/ScrollKeybindingHandler.tsx
```
resume after a
      // brief mouse dip into the viewport.
```

### components/ShowInIDEPrompt.tsx
```
continue…</Text>;
    $[4] = t3;
  } else {
    t3 = $[4];
  }
  let t4;
  if ($[5] !== filePath) {
    t4 = basename(filePath);
    $[5] = filePath;
    $[6] = t4;
  } else {
    t4 = $[6];
  }
  let t5;
  if ($[7] !== t4) {
    t5 = <Text>Do you want to make this edit to{"
```

### components/Spinner/TeammateSpinnerLine.tsx
```
continue;
    }
    const content = msg.
```

### components/Spinner/TeammateSpinnerLine.tsx
```
continue;
      if ('
```

### components/Spinner/TeammateSpinnerLine.tsx
```
continue;
          allLines.
```

### components/Spinner.tsx
```
pick up updates on the parent'
```

### components/StructuredDiff/Fallback.tsx
```
continue;
    }

    // Find a sequence of remove followed by add (possible word-level diff candidates)
    if (current.
```

### components/TagTabs.tsx
```
resumeLabel = showAllProjects ? '
```

### components/TagTabs.tsx
```
Resume (All Projects)'
```

### components/TagTabs.tsx
```
resumeLabelWidth = resumeLabel.
```

### components/TagTabs.tsx
```
resumeLabelWidth - rightHintWidth - 2; // 2 for gaps

  // Clamp selectedIndex to valid range
  const safeSelectedIndex = Math.
```

### components/TagTabs.tsx
```
continue;
        }
      }
      if (canExpandRight) {
        const rightWidth = (tabWidths[endIndex] ?? 0) + 1; // +1 for gap
        if (windowWidth + rightWidth <= effectiveMaxWidth) {
          endIndex++;
          windowWidth += rightWidth;
          continue;
        }
      }
      break;
    }
  }
  const hiddenLeft = startIndex;
  const hiddenRight = tabs.
```

### components/TagTabs.tsx
```
resumeLabel}</Text>
      {hiddenLeft > 0 && <Text dimColor>
          {LEFT_ARROW_PREFIX}
          {hiddenLeft}
        </Text>}
      {visibleTabs.
```

### components/TeleportError.tsx
```
Continue={handleStashComplete} onCancel={onCancel} />;
          $[12] = handleStashComplete;
          $[13] = t9;
        } else {
          t9 = $[13];
        }
        return t9;
      }
    case "
```

### components/TeleportProgress.tsx
```
Resume, type TeleportProgressStep, type TeleportResult, teleportResumeCodeSession } from '
```

### components/TeleportProgress.tsx
```
ResumeCodeSession(sessionId, setStep);
  setStep('
```

### components/TeleportResumeWrapper.tsx
```
ResumeTask } from '
```

### components/TeleportResumeWrapper.tsx
```
ResumeWrapperProps {
  onComplete: (result: TeleportRemoteResponse) => void;
  onCancel: () => void;
  onError?: (error: string, formattedMessage?: string) => void;
  isEmbedded?: boolean;
  source: TeleportSource;
}

/**
 * Wrapper component that manages the full teleport resume flow,
 * including session selection, loading state, and error handling
 */
export function TeleportResumeWrapper(t0) {
```

### components/TeleportResumeWrapper.tsx
```
resumeSession) {
    t4 = async session => {
      const result = await resumeSession(session);
      if (result) {
        onComplete(result);
      } else {
        if (error) {
          if (onError) {
            onError(error.
```

### components/TeleportResumeWrapper.tsx
```
resumeSession;
    $[7] = t4;
  } else {
    t4 = $[7];
  }
  const handleSelect = t4;
  let t5;
  if ($[8] !== onCancel) {
    t5 = () => {
      logEvent("
```

### components/TeleportResumeWrapper.tsx
```
resume session</Text>;
      $[15] = t8;
    } else {
      t8 = $[15];
    }
    let t9;
    if ($[16] !== error.
```

### components/TeleportStash.tsx
```
Continue: () => void;
  onCancel: () => void;
};
export function TeleportStash({
  onStashAndContinue,
  onCancel
}: TeleportStashProps): React.
```

### components/TeleportStash.tsx
```
Continue();
      } else {
        setError('
```

### components/TeleportStash.tsx
```
continue with teleport?
      </Text>

      {stashing ? <Box>
          <Spinner />
          <Text> Stashing changes.
```

### components/ValidationErrorsList.tsx
```
continue;
        const numericPart = parseInt(part, 10);

        // If this is a numeric index and it'
```

### components/VirtualMessageList.tsx
```
resumes where memory-update reminders are dense.
```

### components/VirtualMessageList.tsx
```
continue;
      // The prompt'
```

### components/VirtualMessageList.tsx
```
continue;
      idx = i;
      text = t;
      break;
    }
  }
  const baseOffset = firstVisibleTop >= 0 ? firstVisibleTop - offsets[firstVisible]! : 0;
  const estimate = idx >= 0 ? Math.
```

### components/WorkflowMultiselectDialog.tsx
```
continue</Text></Box>;
    $[9] = showError;
    $[10] = t7;
  } else {
    t7 = $[10];
  }
  let t8;
  if ($[11] !== t6 || $[12] !== t7) {
    t8 = <Dialog title="
```

### components/WorktreeExitDialog.tsx
```
continue working there, or remove it to clean up.
```

### components/agents/ToolSelector.tsx
```
Continue: true
    });
    let t10;
    if ($[37] !== customAgentTools || $[38] !== isAllSelected) {
      t10 = () => {
        const allToolNames_0 = customAgentTools.
```

### components/agents/ToolSelector.tsx
```
Continue ]</Text>;
    $[52] = t13;
    $[53] = t14;
    $[54] = t15;
    $[55] = t16;
  } else {
    t16 = $[55];
  }
  let t17;
  if ($[56] === Symbol.
```

### components/grove/Grove.tsx
```
continue</Text><Text>Your choice takes effect immediately upon confirmation.
```

### components/mcp/ElicitationDialog.tsx
```
Continue without waiting'
```

### components/mcp/MCPRemoteServerMenu.tsx
```
continue managing other servers
      onCancel();
    } catch (err_0) {
      const action = wasEnabled ? '
```

### components/mcp/MCPStdioServerMenu.tsx
```
continue managing other servers
      onCancel();
    } catch (err) {
      const action = wasEnabled ? '
```

### components/messages/AssistantTextMessage.tsx
```
continue{upgradeHint ? `
```

### components/messages/AssistantTextMessage.tsx
```
continue immediately, use /model to switch to{"
```

### components/messages/CollapsedReadSearchContent.tsx
```
continue;
      const latest = lookups.
```

### components/messages/CollapsedReadSearchContent.tsx
```
continue;
      const data = lookups.
```

### components/messages/CollapsedReadSearchContent.tsx
```
continue;
      }
      if (elapsed === undefined || data.
```

### components/messages/UserToolResultMessage/UserToolSuccessMessage.tsx
```
Resumed transcripts deserialize toolUseResult via raw JSON.
```

### components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx
```
resume-auto-mode'
```

### components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx
```
resume-auto-mode'
```

### components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx
```
ResumeAutoOption = feature('
```

### components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx
```
resume-auto-mode'
```

### components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx
```
ResumeAutoOption;
    if (value !== '
```

### components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx
```
resume-auto-mode'
```

### components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx
```
resume-auto-mode falls through here when the auto mode gate is
    // disabled (e.
```

### components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx
```
resume-auto-mode'
```

### components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx
```
resume-auto-mode'
```

### components/permissions/rules/PermissionRuleList.tsx
```
continue;
          }
          options.
```

### components/tasks/RemoteSessionDetailDialog.tsx
```
ResumeCodeSession } from '
```

### components/tasks/RemoteSessionDetailDialog.tsx
```
continue;
    }
    for (const block of msg.
```

### components/tasks/RemoteSessionDetailDialog.tsx
```
continue;
      }
      calls++;
      lastBlock = block;
      if (block.
```

### components/tasks/RemoteSessionDetailDialog.tsx
```
ResumeCodeSession(session.
```

### components/tasks/taskStatusUtils.tsx
```
continue;
    }
    hasVisibleTask = true;
    if (t.
```

### components/teams/TeamsDialog.tsx
```
pick up mode changes from teammates
  useInterval(() => {
    setRefreshKey(k => k + 1);
  }, 1000);
  const currentTeammate = useMemo(() => {
    if (dialogLevel.
```

### components/ui/OrderedList.tsx
```
continue;
    }
    numberOfItems++;
  }
  const maxMarkerWidth = String(numberOfItems).
```

### constants/prompts.ts
```
resume the verifier with its findings plus your fix, repeat until PASS.
```

### constants/prompts.ts
```
resume the verifier with the specifics.
```

### constants/prompts.ts
```
Keep working until you approach the target \u2014 plan your work to fill it productively.
```

### context/stats.tsx
```
continue;
        }
        result[`
```

### context.ts
```
resume) or when git instructions are disabled
    const gitStatus =
      isEnvTruthy(process.
```

### coordinator/coordinatorMode.ts
```
Continue an existing worker (send a follow-up to its \`
```

### coordinator/coordinatorMode.ts
```
Continue workers whose work is complete via ${SEND_MESSAGE_TOOL_NAME} to take advantage of their loaded context
- After launching agents, briefly tell the user what you launched and end your response.
```

### coordinator/coordinatorMode.ts
```
continue that worker

### Example

Each "
```

### coordinator/coordinatorMode.ts
```
Continue the same worker with ${SEND_MESSAGE_TOOL_NAME} — it has the full error context
- If a correction attempt fails, try a different approach or report to the user

### Stopping Workers

Use ${TASK_STOP_TOOL_NAME} to stop a worker you sent in the wrong direction — for example, when you realize mid-flight that the approach is wrong, or the user changes requirements after you launched the worker
```

### coordinator/coordinatorMode.ts
```
continued with ${SEND_MESSAGE_TOOL_NAME}.
```

### coordinator/coordinatorMode.ts
```
Continue with corrected instructions
${SEND_MESSAGE_TOOL_NAME}({ to: "
```

### coordinator/coordinatorMode.ts
```
continue that worker via ${SEND_MESSAGE_TOOL_NAME} or spawn a fresh one.
```

### coordinator/coordinatorMode.ts
```
continue or spawn)
${AGENT_TOOL_NAME}({ prompt: "
```

### coordinator/coordinatorMode.ts
```
continued — the spec quality determines the outcome.
```

### coordinator/coordinatorMode.ts
```
Continue** (${SEND_MESSAGE_TOOL_NAME}) with synthesized spec | Worker already has the files in context AND now gets a clear plan |
| Research was broad but implementation is narrow | **Spawn fresh** (${AGENT_TOOL_NAME}) with synthesized spec | Avoid dragging along exploration noise; focused context is cleaner |
| Correcting a failure or extending recent work | **Continue** | Worker has the error c
```

### coordinator/coordinatorMode.ts
```
Continue mechanics

When continuing a worker with ${SEND_MESSAGE_TOOL_NAME}, it has full context from its previous run:
\`
```

### coordinator/coordinatorMode.ts
```
continued worker, short): "
```

### dialogLaunchers.tsx
```
ResumeConversation'
```

### dialogLaunchers.tsx
```
ResumeConversationProps = React.
```

### dialogLaunchers.tsx
```
ResumeConversation.
```

### dialogLaunchers.tsx
```
ResumeConversation>;

/**
 * Site ~3173: SnapshotUpdateDialog (agent memory snapshot update prompt).
```

### dialogLaunchers.tsx
```
Continue={done}, onExit passed through from caller.
```

### dialogLaunchers.tsx
```
Continue={done} onExit={props.
```

### dialogLaunchers.tsx
```
ResumeWrapper (interactive teleport session picker).
```

### dialogLaunchers.tsx
```
ResumeWrapper(root: Root): Promise<TeleportRemoteResponse | null> {
  const {
    TeleportResumeWrapper
  } = await import('
```

### dialogLaunchers.tsx
```
ResumeWrapper onComplete={done} onCancel={() => done(null)} source="
```

### dialogLaunchers.tsx
```
ResumeConversation mount (interactive session picker).
```

### dialogLaunchers.tsx
```
ResumeChooser(root: Root, appProps: {
  getFpsMetrics: () => FpsMetrics | undefined;
  stats: StatsStore;
  initialState: AppState;
}, worktreePathsPromise: Promise<string[]>, resumeProps: Omit<ResumeConversationProps, '
```

### dialogLaunchers.tsx
```
ResumeConversation
  }, {
    App
  }] = await Promise.
```

### dialogLaunchers.tsx
```
ResumeConversation.
```

### dialogLaunchers.tsx
```
ResumeConversation {.
```

### entrypoints/agentSdkTypes.ts
```
Resume an existing session by ID.
```

### entrypoints/agentSdkTypes.ts
```
resumeSession(
  _sessionId: string,
  _options: SDKSessionOptions,
): SDKSession {
  throw new Error('
```

### entrypoints/agentSdkTypes.ts
```
resumeSession is not implemented in the SDK'
```

### entrypoints/init.ts
```
pick up remote settings before initializing telemetry.
```

### history.ts
```
continue
    expanded =
      expanded.
```

### history.ts
```
continue
        }
        yield entry
      } catch (error) {
        // Not a critical error - just skip malformed lines
        logForDebugging(`
```

### history.ts
```
continue
    if (entry.
```

### history.ts
```
continue
    if (seen.
```

### history.ts
```
continue
    if (entry.
```

### history.ts
```
continue

    if (entry.
```

### history.ts
```
continue
      }

      // For small text content, store inline
      if (content.
```

### hooks/fileSuggestions.ts
```
pick up untracked files,
// which don'
```

### hooks/notifs/useTeammateShutdownNotification.ts
```
continue
      }

      if (task.
```

### hooks/toolPermission/handlers/interactiveHandler.ts
```
continue // refine for TS
        void client.
```

### hooks/toolPermission/handlers/interactiveHandler.ts
```
Continue: () => !isResolved() && !userInteracted,
        onComplete: () => {
          clearClassifierChecking(ctx.
```

### hooks/useArrowKeyHistory.tsx
```
continue;
        }
      }
      entries.
```

### hooks/useHistorySearch.ts
```
resume: boolean, signal?: AbortSignal): Promise<void> => {
      if (!isSearching) {
        return
      }

      if (historyQuery.
```

### hooks/useHistorySearch.ts
```
resume) {
        closeHistoryReader()
        historyReader.
```

### hooks/useInboxPoller.ts
```
continue

        if (setToolUseConfirmQueue) {
          // Route through the standard ToolUseConfirmQueue so tmux workers
          // get the same tool-specific UI (BashPermissionRequest, FileEditToolDiff, etc.
```

### hooks/useInboxPoller.ts
```
continue
          }

          const entry: ToolUseConfirm = {
            assistantMessage: createAssistantMessage({ content: '
```

### hooks/useInboxPoller.ts
```
continue

        if (hasPermissionCallback(parsed.
```

### hooks/useInboxPoller.ts
```
continue

        // Validate required nested fields to prevent crashes from malformed messages
        if (!parsed.
```

### hooks/useInboxPoller.ts
```
continue
        }

        newSandboxRequests.
```

### hooks/useInboxPoller.ts
```
continue

        // Check if we have a registered callback for this request
        if (hasSandboxPermissionCallback(parsed.
```

### hooks/useInboxPoller.ts
```
continue
        }

        // Validate required nested fields to prevent crashes from malformed messages
        if (
          !parsed.
```

### hooks/useInboxPoller.ts
```
continue
        }

        // Apply the permission update to the teammate'
```

### hooks/useInboxPoller.ts
```
continue
        }

        const parsed = isModeSetRequest(m.
```

### hooks/useInboxPoller.ts
```
continue
        }

        const targetMode = permissionModeFromString(parsed.
```

### hooks/useInboxPoller.ts
```
continue

        // Write approval response to teammate'
```

### hooks/useInboxPoller.ts
```
continue

        // Kill the pane if we have the info (pane-based teammates)
        if (parsed.
```

### hooks/useIssueFlagBanner.ts
```
continue
    }
    const content = msg.
```

### hooks/useIssueFlagBanner.ts
```
continue
    }
    for (const block of content) {
      if (block.
```

### hooks/useIssueFlagBanner.ts
```
continue
      }
      const toolName = block.
```

### hooks/useIssueFlagBanner.ts
```
continue
    }
    const text = getUserMessageText(msg)
    if (!text) {
      continue
    }
    return FRICTION_PATTERNS.
```

### hooks/useRemoteSession.ts
```
Resume loading indicator after approving
            setIsLoading(true)
          },
          onReject(feedback?: string) {
            const response: RemotePermissionResponse = {
              behavior: '
```

### hooks/useSSHSession.ts
```
continue reloads
        // history but there'
```

### hooks/useSkillsChange.ts
```
continue
      if (error instanceof Error) {
        logError(error)
      }
    }
  }, [cwd, onCommandsChange])

  useEffect(() => skillChangeDetector.
```

### hooks/useSwarmInitialization.ts
```
resumed teammate sessions.
```

### hooks/useSwarmInitialization.ts
```
Resumed teammate sessions (from --resume or /resume) where teamName/agentName
 *   are stored in transcript messages
 * - Fresh spawns where context is read from environment variables
 */
export function useSwarmInitialization(
  setAppState: SetAppState,
  initialMessages: Message[] | undefined,
  { enabled = true }: { enabled?: boolean } = {},
): void {
  useEffect(() => {
    if (!enabled) retu
```

### hooks/useSwarmInitialization.ts
```
Resumed agent session - set up team context from stored info
        initializeTeammateContextFromSession(setAppState, teamName, agentName)

        // Get agentId from team file for hook initialization
        const teamFile = readTeamFile(teamName)
        const member = teamFile?.
```

### hooks/useSwarmPermissionPoller.ts
```
continue execution.
```

### hooks/useTaskListWatcher.ts
```
pick up the next task.
```

### hooks/useTeleportResume.tsx
```
ResumeCodeSession } from '
```

### hooks/useTeleportResume.tsx
```
ResumeError = {
  message: string;
  formattedMessage?: string;
  isOperationError: boolean;
};
export type TeleportSource = '
```

### hooks/useTeleportResume.tsx
```
Resume(source) {
  const $ = _c(8);
  const [isResuming, setIsResuming] = useState(false);
  const [error, setError] = useState(null);
  const [selectedSession, setSelectedSession] = useState(null);
  let t0;
  if ($[0] !== source) {
    t0 = async session => {
      setIsResuming(true);
      setError(null);
      setSelectedSession(session);
      logEvent("
```

### hooks/useTeleportResume.tsx
```
ResumeCodeSession(session.
```

### hooks/useTeleportResume.tsx
```
resumeSession = t0;
  let t1;
  if ($[2] === Symbol.
```

### hooks/useTurnDiffs.ts
```
continue

      // Check if this is a user prompt (not a tool result)
      const isToolResult =
        message.
```

### hooks/useTypeahead.tsx
```
ResumeInputFromSuggestion(suggestion: SuggestionItem): string {
  const metadata = suggestion.
```

### hooks/useTypeahead.tsx
```
resume ${metadata.
```

### hooks/useTypeahead.tsx
```
resume ${suggestion.
```

### hooks/useTypeahead.tsx
```
continue;
          if (!t.
```

### hooks/useTypeahead.tsx
```
continue;
          seen.
```

### hooks/useTypeahead.tsx
```
continue;
        if (!name.
```

### hooks/useTypeahead.tsx
```
continue;
        const status = state.
```

### hooks/useTypeahead.tsx
```
resume command
      if (parsedCommand && parsedCommand.
```

### hooks/useTypeahead.tsx
```
resume-title-${sessionId}`
```

### hooks/useTypeahead.tsx
```
resume
      // we need to clear the suggestions.
```

### hooks/useTypeahead.tsx
```
resume command with sessionId
        if (suggestion) {
          const newInput = buildResumeInputFromSuggestion(suggestion);
          onInputChange(newInput);
          setCursorOffset(newInput.
```

### hooks/useTypeahead.tsx
```
continue typing or select a specific option
          // Instead, update for the new prefix
          void updateSuggestions(input.
```

### hooks/useTypeahead.tsx
```
resume command with sessionId
      if (suggestion) {
        const newInput = buildResumeInputFromSuggestion(suggestion);
        onInputChange(newInput);
        setCursorOffset(newInput.
```

### hooks/useTypeahead.tsx
```
continue with navigation if we have suggestions
    if (suggestions.
```

### hooks/useVirtualScroll.ts
```
resumed session this showed as +270MB RSS during PageUp spam
  // (yoga Node constructor + createWorkInProgress fiber alloc proportional
  // to scroll distance).
```

### hooks/useVirtualScroll.ts
```
continue
      const h = yoga.
```

### hooks/useVoiceIntegration.tsx
```
continue
    // appending subsequent transcripts after it.
```

### hooks/useVoiceIntegration.tsx
```
continue;
      if (binding.
```

### hooks/useVoiceIntegration.tsx
```
continue;
      const ks = binding.
```

### hooks/useVoiceIntegration.tsx
```
continue;
      if (binding.
```

### hooks/useVoiceIntegration.tsx
```
continued keypresses and forward
      // to voice for release detection.
```

### ink/Ansi.tsx
```
continue;
    }
    if (action.
```

### ink/Ansi.tsx
```
continue;
      const props = textStyleToSpanProps(action.
```

### ink/components/App.tsx
```
RESUME_GAP_MS = 5000;
type Props = {
  readonly children: ReactNode;
  readonly stdin: NodeJS.
```

### ink/components/App.tsx
```
RESUME_GAP_MS gap.
```

### ink/components/App.tsx
```
Resume?: () => void;
  // Receives the declared native-cursor position from useDeclaredCursor
  // so ink.
```

### ink/components/App.tsx
```
RESUME_GAP_MS) {
      this.
```

### ink/components/App.tsx
```
resume handler
    const resumeHandler = () => {
      // Restore raw mode to exact previous state
      for (let i = 0; i < rawModeCountBeforeSuspend; i++) {
        if (this.
```

### ink/components/App.tsx
```
resume event for Claude Code to handle
      this.
```

### ink/components/App.tsx
```
resumeHandler);
    };
    process.
```

### ink/components/App.tsx
```
resumeHandler);
    process.
```

### ink/components/App.tsx
```
continue;
    }

    // Mouse click/drag events update selection state (fullscreen only).
```

### ink/components/App.tsx
```
continue;
    }
    const sequence = item.
```

### ink/components/App.tsx
```
continue;
    }
    if (sequence === FOCUS_OUT) {
      app.
```

### ink/components/App.tsx
```
continue;
    }

    // Failsafe: if we receive input, the terminal must be focused
    if (!getTerminalFocused()) {
      setTerminalFocused(true);
    }

    // Handle Ctrl+Z (suspend) using parsed key to support both raw (\x1a) and
    // CSI u format (\x1b[122;5u) from Kitty keyboard protocol terminals
    if (item.
```

### ink/components/App.tsx
```
continue;
    }
    app.
```

### ink/hit-test.ts
```
continue
    const hit = hitTest(child, col, row)
    if (hit) return hit
  }
  return node
}

/**
 * Hit-test the root at (col, row) and bubble a ClickEvent from the deepest
 * containing node up through parentNode.
```

### ink/hooks/use-animation-frame.ts
```
resumes from the current clock time
 * when a number is passed again.
```

### ink/hooks/use-terminal-viewport.ts
```
pick up the latest value naturally.
```

### ink/ink.tsx
```
resume knows whether to
  // re-enable mouse tracking (not all <AlternateScreen> uses want it).
```

### ink/ink.tsx
```
Resume);
      this.
```

### ink/ink.tsx
```
Resume);
      };
    }
    this.
```

### ink/ink.tsx
```
Resume = () => {
    if (!this.
```

### ink/ink.tsx
```
Resume (SIGCONT) and the sleep-wake detector; resize itself
    // doesn'
```

### ink/ink.tsx
```
Resume Ink after an external TUI handoff with a full repaint.
```

### ink/ink.tsx
```
resumeStdin();
    if (this.
```

### ink/ink.tsx
```
resume();
    // Re-enable focus reporting and extended key reporting — terminal
    // editors (vim, nano, etc.
```

### ink/ink.tsx
```
resume(): void {
    this.
```

### ink/ink.tsx
```
resume() when the terminal content has been corrupted by
   * an external process (e.
```

### ink/ink.tsx
```
ResumeHint(), which tmux (at least) interprets
   * as restoring the saved cursor position — clobbering the resume hint.
```

### ink/ink.tsx
```
resumeStdin(): void {
    const stdin = this.
```

### ink/ink.tsx
```
resumeStdin: called with no stored listeners and wasRawMode=false (possible desync)'
```

### ink/ink.tsx
```
resumeStdin: re-attaching ${this.
```

### ink/log-update.ts
```
resumes from suspension (SIGCONT) to prevent clobbering terminal content
  reset(): void {
    this.
```

### ink/log-update.ts
```
continue
      }

      moveCursorTo(screen, x, y)

      // Handle hyperlink
      const targetHyperlink = cell.
```

### ink/optimizer.ts
```
continue
    } else if (type === '
```

### ink/optimizer.ts
```
continue
    } else if (type === '
```

### ink/optimizer.ts
```
continue
    }

    // Try to merge with previous patch
    if (len > 0) {
      const lastIdx = len - 1
      const last = result[lastIdx]!
      const lastType = last.
```

### ink/optimizer.ts
```
continue
      }

      // Collapse consecutive cursorTo (only the last one matters)
      if (type === '
```

### ink/optimizer.ts
```
continue
      }

      // Concat adjacent style patches.
```

### ink/optimizer.ts
```
continue
      }

      // Dedupe hyperlinks
      if (
        type === '
```

### ink/optimizer.ts
```
continue
      }

      // Cancel cursor hide/show pairs
      if (
        (type === '
```

### ink/optimizer.ts
```
continue
      }
    }

    result.
```

### ink/output.ts
```
continue
      const { x, y, width, height } = operation.
```

### ink/output.ts
```
continue
      const rect = {
        x: startX,
        y: startY,
        width: maxX - startX,
        height: maxY - startY,
      }
      screen.
```

### ink/output.ts
```
continue

        case '
```

### ink/output.ts
```
continue

        case '
```

### ink/output.ts
```
continue

        case '
```

### ink/output.ts
```
continue
          // Skip rows covered by an absolute-positioned node'
```

### ink/output.ts
```
continue
          }
          let rowStart = startY
          for (let row = startY; row <= maxY; row++) {
            const excluded =
              row < maxY &&
              absoluteClears.
```

### ink/output.ts
```
continue
        }

        case '
```

### ink/output.ts
```
continue
        }

        case '
```

### ink/output.ts
```
continue
              }
            }

            if (clipVertically) {
              const height = lines.
```

### ink/output.ts
```
continue
              }
            }

            if (clipHorizontally) {
              lines = lines.
```

### ink/output.ts
```
continue
        }
      }
    }

    // noSelect ops go LAST so they win over blits (which copy noSelect
    // from prevScreen) and writes (which don'
```

### ink/output.ts
```
continue
    }

    // Zero-width characters (combining marks, ZWNJ, ZWS, etc.
```

### ink/output.ts
```
continue
    }

    const isWideCharacter = charWidth >= 2

    // Wide char at last column can'
```

### ink/output.ts
```
continue
    }

    // styleId + hyperlink were precomputed during clustering (once per
    // style run, cached via charCache).
```

### ink/reconciler.ts
```
continue
        }

        if (key === '
```

### ink/reconciler.ts
```
continue
        }

        if (EVENT_HANDLER_PROPS.
```

### ink/reconciler.ts
```
continue
        }

        setAttribute(node, key, value as DOMNodeAttribute)
      }
    }

    if (style && node.
```

### ink/render-node-to-output.ts
```
pick up the
      // `
```

### ink/render-node-to-output.ts
```
continues even past the clamp — the render-clamp below
          // holds the VISUAL at the mounted edge regardless.
```

### ink/render-node-to-output.ts
```
resumes → edge again.
```

### ink/render-node-to-output.ts
```
continue
                  // Uncached = culled last frame, now re-entering.
```

### ink/render-node-to-output.ts
```
continue
                const childTop = cy.
```

### ink/render-node-to-output.ts
```
continue
                // Skip children entirely within edge rows (already rendered)
                if (childTop >= edgeTopLocal && childBottom <= edgeBottomLocal)
                  continue
                const screenY = Math.
```

### ink/render-node-to-output.ts
```
continue
                  }
                }
                // Wipe this child'
```

### ink/render-node-to-output.ts
```
continue
              const shiftedTop = Math.
```

### ink/render-node-to-output.ts
```
continue
              if (shiftedTop >= shiftedBottom) continue
              const fill = Array(shiftedBottom - shiftedTop)
                .
```

### ink/render-node-to-output.ts
```
continue
    return sib.
```

### ink/render-node-to-output.ts
```
continue
    return sib.
```

### ink/render-node-to-output.ts
```
continue
    const elem = child as DOMElement
    if (elem.
```

### ink/render-node-to-output.ts
```
continue
      }
    }
    const wasDirty = childElem.
```

### ink/render-to-screen.ts
```
continue
      }
      const lc = cell.
```

### ink/render-to-screen.ts
```
continue
    const cell = cellAtIndex(screen, rowOff + col)
    setCellStyleId(screen, col, row, transform(cell.
```

### ink/root.ts
```
resume) can find it.
```

### ink/screen.ts
```
continues on the next line.
```

### ink/screen.ts
```
continue
    const match = code.
```

### ink/screen.ts
```
continue
    cellAtCI(next, ci, nextCell)
    if (cb(x, y, undefined, nextCell)) return true
  }
  return false
}

/**
 * Diff two screens with identical width.
```

### ink/screen.ts
```
continue
      }
      cellAtCI(prev, prevCI, prevCell)
      cellAtCI(next, nextCI, nextCell)
      prevCI += 2
      nextCI += 2
      if (cb(x, y, prevCell, nextCell)) return true
    }

    if (prevEndX > bothEndX) {
      prevCI = prevRowCI + ((bothEndX - startX) << 1)
      for (let x = bothEndX; x < prevEndX; x++) {
        cellAtCI(prev, prevCI, prevCell)
        prevCI += 2
        if (cb
```

### ink/searchHighlight.ts
```
continue
      }
      const lc = cell.
```

### ink/selection.ts
```
continue
    }
    if (charClass(pc.
```

### ink/selection.ts
```
continue
    }
    if (charClass(nc.
```

### ink/selection.ts
```
continue
    }
    const opener = OPENER[last]
    if (!opener) break
    let opens = 0
    let closes = 0
    for (let i = 0; i < url.
```

### ink/selection.ts
```
continue
    const cell = cellAt(screen, col, row)
    if (!cell) continue
    // Skip spacer tails (second half of wide chars) — the head already
    // contains the full grapheme.
```

### ink/selection.ts
```
continue
    }
    line += cell.
```

### ink/selection.ts
```
continue
      const cell = cellAtIndex(screen, idx)
      setCellStyleId(screen, col, row, stylePool.
```

### ink/squash-text-nodes.ts
```
continue
    }

    if (childNode.
```

### ink/squash-text-nodes.ts
```
continue
    }

    if (childNode.
```

### ink/stringWidth.ts
```
continue
    }

    // Calculate width for non-emoji graphemes
    // For grapheme clusters (like Devanagari conjuncts with virama+ZWJ), only count
    // the first non-zero-width character'
```

### ink/styles.ts
```
pick up leading whitespace from middle rows.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 1) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 2) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 3) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 4) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 5 || code === 6) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 7) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 8) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 9) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 21) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 22) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 23) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 24) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 25) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 27) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 28) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 29) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 53) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 55) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }

    if (code >= 30 && code <= 37) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 39) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code >= 40 && code <= 47) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code === 49) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code >= 90 && code <= 97) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }
    if (code >= 100 && code <= 107) {
      s.
```

### ink/termio/sgr.ts
```
continue
    }

    if (code === 38) {
      const c = parseExtendedColor(params, i)
      if (c) {
        s.
```

### ink/termio/sgr.ts
```
continue
      }
    }
    if (code === 48) {
      const c = parseExtendedColor(params, i)
      if (c) {
        s.
```

### ink/termio/sgr.ts
```
continue
      }
    }
    if (code === 58) {
      const c = parseExtendedColor(params, i)
      if (c) {
        s.
```

### ink/termio/sgr.ts
```
continue
      }
    }
    if (code === 59) {
      s.
```

### ink/termio/tokenize.ts
```
continue buffering
          result.
```

### interactiveHelpers.tsx
```
continue;
          }
          const now = Date.
```

### keybindings/defaultBindings.ts
```
resume, permission prompts, etc.
```

### keybindings/resolver.ts
```
continue
    if (!ctxSet.
```

### keybindings/resolver.ts
```
continue

    if (matchesBinding(input, key, binding)) {
      match = binding
    }
  }

  if (!match) {
    return { type: '
```

### keybindings/validate.ts
```
continue

    // Find the context for this block by looking backwards
    const textBeforeBlock = jsonString.
```

### keybindings/validate.ts
```
continue

      const count = (keysByName.
```

### main.tsx
```
ResumeChooser, launchSnapshotUpdateDialog, launchTeleportRepoMismatchDialog, launchTeleportResumeWrapper } from '
```

### main.tsx
```
Resume, processResumedConversation } from '
```

### main.tsx
```
ResumeWrapper dynamically imported at call sites
import { migrateAutoUpdatesToSettings } from '
```

### main.tsx
```
Resume, teleportToRemoteWithErrorHandling, validateGitState, validateSessionRepository } from '
```

### main.tsx
```
resume + model flags to the remote CLI'
```

### main.tsx
```
continue/-c and --resume <uuid> operate on the REMOTE session history
      // (which persists under the remote'
```

### main.tsx
```
continues without remote settings
    // Settings are applied via hot-reload when they arrive
    // Must happen after init() to ensure config reading is allowed
    void loadRemoteManagedSettings();
    void loadPolicyLimits();
    profileCheckpoint('
```

### main.tsx
```
Continue the most recent conversation in the current directory'
```

### main.tsx
```
Resume a conversation by session ID, or open interactive picker with optional search term'
```

### main.tsx
```
resume or --continue)'
```

### main.tsx
```
Resume a session linked to a PR by PR number/URL, or open interactive picker with optional search term'
```

### main.tsx
```
resumed (only works with --print)'
```

### main.tsx
```
resume-session-at <message id>'
```

### main.tsx
```
resume in print mode)'
```

### main.tsx
```
resume and terminal title)'
```

### main.tsx
```
continue or --resume when --fork-session is also provided
      // (to specify a custom ID for the forked session)
      if ((options.
```

### main.tsx
```
continue || options.
```

### main.tsx
```
resume) && !options.
```

### main.tsx
```
continue or --resume if --fork-session is also specified.
```

### main.tsx
```
resume view display and restoration
    if (mainThreadAgentDefinition?.
```

### main.tsx
```
resume-picker — p99 was ~70s
      // dominated by dialog-wait time, not code-path startup.
```

### main.tsx
```
resume/continue (conversationRecovery.
```

### main.tsx
```
resume
    // and the second systemMessage clobbers the first.
```

### main.tsx
```
continue || options.
```

### main.tsx
```
resume ? null : processSessionStartHooks('
```

### main.tsx
```
continue/resume/teleport paths don'
```

### main.tsx
```
resume branch, where this promise is
      // undefined and the ?? fallback runs).
```

### main.tsx
```
continue || options.
```

### main.tsx
```
resume || teleport || setupTrigger ? undefined : processSessionStartHooks('
```

### main.tsx
```
continue;
            const sig = getMcpServerSignature(config);
            if (sig && claudeaiSigs.
```

### main.tsx
```
continue;
              c.
```

### main.tsx
```
continue,
        resume: options.
```

### main.tsx
```
resume,
        verbose: verbose,
        outputFormat: outputFormat,
        jsonSchema,
        permissionPromptToolName: options.
```

### main.tsx
```
resumeSessionAt: options.
```

### main.tsx
```
resumeSessionAt || undefined,
        rewindFiles: options.
```

### main.tsx
```
ResumedConversation calls
    const resumeContext = {
      modeApi: coordinatorModeModule,
      mainThreadAgentDefinition,
      agentDefinitions,
      currentCwd,
      cliAgents,
      initialState
    };
    if (options.
```

### main.tsx
```
continue) {
      // Continue the most recent conversation directly
      let resumeSucceeded = false;
      try {
        const resumeStart = performance.
```

### main.tsx
```
Resume(undefined /* sessionId */, undefined /* sourceFile */);
        if (!result) {
          logEvent('
```

### main.tsx
```
ResumedConversation(result, {
          forkSession: !!options.
```

### main.tsx
```
resumeContext);
        if (loaded.
```

### main.tsx
```
resume_duration_ms: Math.
```

### main.tsx
```
resumeStart)
        });
        resumeSucceeded = true;
        await launchRepl(root, {
          getFpsMetrics,
          stats,
          initialState: loaded.
```

### main.tsx
```
resumeSucceeded) {
          logEvent('
```

### main.tsx
```
resume || options.
```

### main.tsx
```
resume flow - from file (ant-only), session ID, or interactive selector

      // Clear stale caches before resuming to ensure fresh file/skill discovery
      const {
        clearSessionCaches
      } = await import('
```

### main.tsx
```
Resume: ProcessedResume | undefined = undefined;
      let maybeSessionId = validateUuid(options.
```

### main.tsx
```
resume);
      let searchTerm: string | undefined = undefined;
      // Store full LogOption when found by custom title (for cross-worktree resume)
      let matchedLog: LogOption | null = null;
      // PR filter for --from-pr flag
      let filterByPr: boolean | number | string | undefined = undefined;

      // Handle --from-pr flag
      if (options.
```

### main.tsx
```
resume value is not a UUID, try exact match by custom title first
      if (options.
```

### main.tsx
```
resume && typeof options.
```

### main.tsx
```
resume
            matchedLog = matches[0]!;
            maybeSessionId = getSessionIdFromLog(matchedLog) ?? null;
          } else {
            // No match or multiple matches - use as search term for picker
            searchTerm = trimmedValue;
          }
        }
      }

      // --remote and --teleport both create/resume Claude Code Web (CCR) sessions.
```

### main.tsx
```
Resume with: claude --teleport ${createdSession.
```

### main.tsx
```
resume
          logEvent('
```

### main.tsx
```
ResumeTeleportTask: Starting teleport flow.
```

### main.tsx
```
ResumeWrapper(root);
          if (!teleportResult) {
            // User cancelled or error occurred
            await gracefulShutdown(0);
            process.
```

### main.tsx
```
Resume(teleportResult.
```

### main.tsx
```
resume && typeof options.
```

### main.tsx
```
resume);
          if (ccshareId) {
            try {
              const resumeStart = performance.
```

### main.tsx
```
Resume(logOption, undefined);
              if (result) {
                processedResume = await processResumedConversation(result, {
                  forkSession: true,
                  transcriptPath: result.
```

### main.tsx
```
resumeContext);
                if (processedResume.
```

### main.tsx
```
resume_duration_ms: Math.
```

### main.tsx
```
resumeStart)
                });
              } else {
                logEvent('
```

### main.tsx
```
resume from ccshare: ${errorMessage(error)}`
```

### main.tsx
```
resume);
            try {
              const resumeStart = performance.
```

### main.tsx
```
Resume(logOption, undefined /* sourceFile */);
                if (result) {
                  processedResume = await processResumedConversation(result, {
                    forkSession: !!options.
```

### main.tsx
```
resumeContext);
                  if (processedResume.
```

### main.tsx
```
resume_duration_ms: Math.
```

### main.tsx
```
resumeStart)
                  });
                } else {
                  logEvent('
```

### main.tsx
```
Resume specific session by ID
        const sessionId = maybeSessionId;
        try {
          const resumeStart = performance.
```

### main.tsx
```
resume by custom title)
          // Otherwise fall back to sessionId string (for direct UUID resume)
          const result = await loadConversationForResume(matchedLog ?? sessionId, undefined);
          if (!result) {
            logEvent('
```

### main.tsx
```
Resume = await processResumedConversation(result, {
            forkSession: !!options.
```

### main.tsx
```
resumeContext);
          if (processedResume.
```

### main.tsx
```
resume_duration_ms: Math.
```

### main.tsx
```
resumeStart)
          });
        } catch (error) {
          logEvent('
```

### main.tsx
```
resume session ${sessionId}`
```

### main.tsx
```
resume or teleport messages, render the REPL
      const resumeData = processedResume ?? (Array.
```

### main.tsx
```
resumeData) {
        maybeActivateProactive(options);
        maybeActivateBrief(options);
        await launchRepl(root, {
          getFpsMetrics,
          stats,
          initialState: resumeData.
```

### main.tsx
```
ResumeConversation loads logs internally to ensure proper GC after selection
        await launchResumeChooser(root, {
          getFpsMetrics,
          stats,
          initialState
        }, getWorktreePaths(getOriginalCwd()), {
          .
```

### main.tsx
```
resumes know what mode was used
      if (feature('
```

### main.tsx
```
Resume a teleport session, optionally specify session ID'
```

### memdir/memdir.ts
```
continues either way; the model'
```

### memdir/memoryTypes.ts
```
keep working, files with unknown types degrade gracefully.
```

### migrations/migrateOpusToOpus1m.ts
```
continues to use plain Opus.
```

### native-ts/color-diff/index.ts
```
continue
          }
        }
        cur.
```

### native-ts/color-diff/index.ts
```
continue
      }

      let remaining = text
      let pos = textStart
      while (remaining.
```

### native-ts/file-index/index.ts
```
continue

      const haystack = caseSensitive ? paths[i]! : lowerPaths[i]!

      // Fused indexOf scan: find positions (SIMD-accelerated in JSC/V8) AND
      // accumulate gap/consecutive terms inline.
```

### native-ts/file-index/index.ts
```
continue
      posBuf[0] = pos
      let gapPenalty = 0
      let consecBonus = 0
      let prev = pos
      for (let j = 1; j < nLen; j++) {
        pos = haystack.
```

### native-ts/file-index/index.ts
```
continue outer
        posBuf[j] = pos
        const gap = pos - prev - 1
        if (gap === 0) consecBonus += BONUS_CONSECUTIVE
        else gapPenalty += PENALTY_GAP_START + gap * PENALTY_GAP_EXTENSION
        prev = pos
      }

      // Gap-bound reject: if the best-case score (all boundary bonuses) minus
      // known gap penalties can'
```

### native-ts/file-index/index.ts
```
continue
      }

      // Boundary/camelCase scoring: check the char before each match position.
```

### native-ts/yoga-layout/index.ts
```
continue
        const mTop = resolveEdge(c.
```

### native-ts/yoga-layout/index.ts
```
continue
      if (isMarginAuto(c.
```

### native-ts/yoga-layout/index.ts
```
continue
      const c = children[i]!
      let t = c.
```

### native-ts/yoga-layout/index.ts
```
continue
      const v = children[i]!.
```

### native-ts/yoga-layout/index.ts
```
continue
    if (c.
```

### native-ts/yoga-layout/index.ts
```
continue
    if (
      resolveChildAlign(node, c) === Align.
```

### plugins/builtinPlugins.ts
```
continue
    }

    const pluginId = `
```

### plugins/builtinPlugins.ts
```
continue
    for (const skill of definition.
```

### query/tokenBudget.ts
```
ContinueDecision = {
  action: '
```

### query/tokenBudget.ts
```
ContinueDecision | StopDecision

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  if (agentId || budget === null || budget <= 0) {
    return { action: '
```

### query.ts
```
Continue | undefined
}

export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent
  | RequestStartEvent
  | Message
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal
> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // Only reached if queryLoop returned normally.
```

### query.ts
```
Continue sites write `
```

### query.ts
```
continue
  // sites.
```

### query.ts
```
resume: agentId
    // routes to sidechain file (AgentTool resume) or session file (/resume).
```

### query.ts
```
continue site (query.
```

### query.ts
```
Continue on with the current query call using the post compact messages
      messagesForQuery = postCompactMessages
    } else if (consecutiveFailures !== undefined) {
      // Autocompact failed — propagate failure count so the circuit breaker
      // can stop retrying on the next iteration.
```

### query.ts
```
resume,
                    // while adding nothing the SDK stream needs — hooks get
                    // the expanded path via toolExecution.
```

### query.ts
```
continue
          }
          throw innerError
        }
      }
    } catch (error) {
      logError(error)
      const errorMessage =
        error instanceof Error ? error.
```

### query.ts
```
continue
          }
        }
      }
      if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
        const compacted = await reactiveCompact.
```

### query.ts
```
continue
        }

        // No recovery — surface the withheld error and exit.
```

### query.ts
```
continue
        }

        if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
          const recoveryMessage = createUserMessage({
            content:
              `
```

### query.ts
```
Resume directly — no apology, no recap of what you were doing.
```

### query.ts
```
Pick up mid-thought if that is where the cut happened.
```

### query.ts
```
continue
        }

        // Recovery exhausted — surface the withheld error now.
```

### query.ts
```
continue
      }

      if (feature('
```

### query.ts
```
continue
        }

        if (decision.
```

### screens/Doctor.tsx
```
Continue /></Box>;
    $[75] = t40;
  } else {
    t40 = $[75];
  }
  let t41;
  if ($[76] !== t23 || $[77] !== t30 || $[78] !== t35 || $[79] !== t36 || $[80] !== t37 || $[81] !== t38 || $[82] !== t39) {
    t41 = <Pane>{t23}{t30}{t31}{t32}{t33}{t34}{t35}{t36}{t37}{t38}{t39}{t40}</Pane>;
    $[76] = t23;
    $[77] = t30;
    $[78] = t35;
    $[79] = t36;
    $[80] = t37;
    $[81] = t38;
    $[82]
```

### screens/REPL.tsx
```
ResumeEntrypoint, getCommandName, isCommandEnabled } from '
```

### screens/REPL.tsx
```
resumeAgentBackground } from '
```

### screens/REPL.tsx
```
Resume, getPlanSlug, setPlanSlug } from '
```

### screens/REPL.tsx
```
ResumedSessionFile, removeTranscriptMessage, restoreSessionMetadata, getCurrentSessionTitle, isEphemeralToolProgress, isLoggableMessage, saveWorktreeState, getAgentTranscript } from '
```

### screens/REPL.tsx
```
Resume, fileHistoryEnabled, fileHistoryHasAnyChanges } from '
```

### screens/REPL.tsx
```
Resume, exitRestoredWorktree } from '
```

### screens/REPL.tsx
```
resume (name/color set via /rename or /color)
  initialAgentName?: string;
  initialAgentColor?: AgentColorName;
  mcpClients?: MCPServerConnection[];
  dynamicMcpConfig?: Record<string, ScopedMcpServerConfig>;
  autoConnectIdeFlag?: boolean;
  strictMcpConfig?: boolean;
  systemPrompt?: string;
  appendSystemPrompt?: string;
  // Optional callback invoked before query execution
  // Called after 
```

### screens/REPL.tsx
```
resume can update it mid-session
  const [mainThreadAgentDefinition, setMainThreadAgentDefinition] = useState(initialMainThreadAgentDefinition);
  const toolPermissionContext = useAppState(s => s.
```

### screens/REPL.tsx
```
ResumeConversation.
```

### screens/REPL.tsx
```
resumed teammate sessions
  useSwarmInitialization(setAppState, initialMessages, {
    enabled: !isRemoteSession
  });
  const mergedTools = useMergedTools(combinedInitialTools, mcp.
```

### screens/REPL.tsx
```
ContinueAnimation?: true;
    showSpinner?: boolean;
    isLocalJSXCommand?: boolean;
    isImmediate?: boolean;
  } | null>(null);

  // Track local JSX commands separately so tools can'
```

### screens/REPL.tsx
```
ContinueAnimation?: true;
    showSpinner?: boolean;
    isLocalJSXCommand: true;
  } | null>(null);

  // Wrapper for setToolJSX that preserves local JSX commands (like /btw).
```

### screens/REPL.tsx
```
ContinueAnimation?: true;
    showSpinner?: boolean;
    isLocalJSXCommand?: boolean;
    clearLocalJSX?: boolean;
  } | null) => {
    // If setting a local JSX command, store it in the ref
    if (args?.
```

### screens/REPL.tsx
```
resume) wins over
  // the agent name, which wins over the Haiku-extracted topic;
  // all fall back to the product name.
```

### screens/REPL.tsx
```
resume (initialMessages present) so we don'
```

### screens/REPL.tsx
```
resumed
  // session from mid-conversation context.
```

### screens/REPL.tsx
```
resumed sessions, reconstruction does O(messages × blocks)
  // work; we only want that once.
```

### screens/REPL.tsx
```
ResumeConsistency report false delta<0 for
      // every turn that ran a progress-emitting tool.
```

### screens/REPL.tsx
```
resume = useCallback(async (sessionId: UUID, log: LogOption, entrypoint: ResumeEntrypoint) => {
    const resumeStart = performance.
```

### screens/REPL.tsx
```
resumed session
      if (feature('
```

### screens/REPL.tsx
```
resumed one, mirroring the /clear flow in conversation.
```

### screens/REPL.tsx
```
resume
      const hookMessages = await processSessionStartHooks('
```

### screens/REPL.tsx
```
resumes, reuse the original session'
```

### screens/REPL.tsx
```
Resume(log, asSessionId(sessionId));
      }

      // Restore file history and attribution state from the resumed conversation
      restoreSessionStateFromLog(log, setAppState);
      if (log.
```

### screens/REPL.tsx
```
Resume(log);
      }

      // Restore agent setting from the resumed conversation
      // Always reset to the new session'
```

### screens/REPL.tsx
```
resumed conversation
      // Always reset to the new session'
```

### screens/REPL.tsx
```
resumed session ID
      const {
        renameRecordingForSession
      } = await import('
```

### screens/REPL.tsx
```
Resumed sessions shouldn'
```

### screens/REPL.tsx
```
resume entered, then cd into the one
      // this session was in.
```

### screens/REPL.tsx
```
ResumedConversation for the adopt —
      // fork materializes its own file via recordTranscript on REPL mount.
```

### screens/REPL.tsx
```
ResumedSessionFile();
        void restoreRemoteAgentTasks({
          abortController: new AbortController(),
          getAppState: () => store.
```

### screens/REPL.tsx
```
resumes know what mode this session was in
      if (feature('
```

### screens/REPL.tsx
```
resume write to the
      // resumed session'
```

### screens/REPL.tsx
```
resume_duration_ms: Math.
```

### screens/REPL.tsx
```
resumeStart)
      });
    } catch (error) {
      logEvent('
```

### screens/REPL.tsx
```
resume flows)
  // This allows Claude to edit files that were read in previous sessions
  const restoreReadFileState = useCallback((messages: MessageType[], cwd: string) => {
    const extracted = extractReadFilesFromMessages(messages, cwd, READ_FILE_STATE_CACHE_SIZE);
    readFileState.
```

### screens/REPL.tsx
```
resume (--resume-session) and ResumeConversation screen
  // where messages are passed as props rather than through the resume callback
  useEffect(() => {
    if (initialMessages && initialMessages.
```

### screens/REPL.tsx
```
ContinueAnimation is true.
```

### screens/REPL.tsx
```
ContinueAnimation;
    if (allowDialogsWithAnimation && toolUseConfirmQueue[0]) return '
```

### screens/REPL.tsx
```
resume when focusedInputDialog changes
  // This ensures accurate timing even under high system load, rather than
  // relying on the 100ms polling interval to detect state changes
  useEffect(() => {
    if (!isLoading) return;
    const isPaused = focusedInputDialog === '
```

### screens/REPL.tsx
```
resume when they submit their next input (see onSubmit).
```

### screens/REPL.tsx
```
resume,
      setConversationId,
      requestPrompt: feature('
```

### screens/REPL.tsx
```
resume, requestPrompt, disabled, customSystemPrompt, appendSystemPrompt, setConversationId]);

  // Session backgrounding (Ctrl+B to background/foreground)
  const handleBackgroundQuery = useCallback(() => {
    // Stop the foreground query so the background one takes over
    abortController?.
```

### screens/REPL.tsx
```
resume
        if (feature('
```

### screens/REPL.tsx
```
resume after compaction.
```

### screens/REPL.tsx
```
Resume loop mode if paused
    if (feature('
```

### screens/REPL.tsx
```
resumeProactive();
    }

    // Handle immediate commands - these bypass the queue and execute right away
    // even while Claude is processing.
```

### screens/REPL.tsx
```
resumeAgentBackground({
          agentId: task.
```

### screens/REPL.tsx
```
resumeAgentBackground failed: ${errorMessage(err)}`
```

### screens/REPL.tsx
```
resume-agent-failed-${task.
```

### screens/REPL.tsx
```
resume agent: {errorMessage(err)}
                </Text>,
            priority: '
```

### screens/REPL.tsx
```
resumes a conversation then quites before doing
  // anything else
  useLogMessages(messages, messages.
```

### screens/REPL.tsx
```
resume events
  const {
    internal_eventEmitter
  } = useStdin();
  const [remountKey, setRemountKey] = useState(0);
  useEffect(() => {
    const handleSuspend = () => {
      // Print suspension instructions
      process.
```

### screens/REPL.tsx
```
Resume = () => {
      // Force complete component tree replacement instead of terminal clear
      // Ink now handles line count reset internally on SIGCONT
      setRemountKey(prev => prev + 1);
    };
    internal_eventEmitter?.
```

### screens/REPL.tsx
```
Resume);
    return () => {
      internal_eventEmitter?.
```

### screens/REPL.tsx
```
Resume);
    };
  }, [internal_eventEmitter]);

  // Derive stop hook spinner suffix from messages state
  const stopHookSpinnerSuffix = useMemo(() => {
    if (!isLoading) return null;

    // Find stop hook progress messages
    const progressMsgs = messages.
```

### screens/REPL.tsx
```
resume for
      // terminal editors; GUI editors spawn detached.
```

### screens/ResumeConversation.tsx
```
ResumedSessionFile, enrichLogs, isCustomTitleEnabled, loadAllProjectsMessageLogsProgressive, loadSameRepoMessageLogsProgressive, recordContentReplacement, resetSessionFilePointer, restoreSessionMetadata, type SessionLogResult } from '
```

### screens/ResumeConversation.tsx
```
ResumeConversation({
  commands,
  worktreePaths,
  initialTools,
  mcpClients,
  dynamicMcpConfig,
  debug,
  mainThreadAgentDefinition,
  autoConnectIdeFlag,
  strictMcpConfig = false,
  systemPrompt,
  appendSystemPrompt,
  initialSearchQuery,
  disableSlashCommands = false,
  forkSession,
  taskListId,
  filterByPr,
  thinkingConfig,
  onTurnComplete
}: Props): React.
```

### screens/ResumeConversation.tsx
```
resumeData, setResumeData] = React.
```

### screens/ResumeConversation.tsx
```
ResumeWithRenameEnabled = isCustomTitleEnabled();
  React.
```

### screens/ResumeConversation.tsx
```
resumeStart = performance.
```

### screens/ResumeConversation.tsx
```
Resume(log_0, showAllProjects, worktreePaths);
    if (crossProjectCheck.
```

### screens/ResumeConversation.tsx
```
Resume(log_0, undefined);
      if (!result_3) {
        throw new Error('
```

### screens/ResumeConversation.tsx
```
ResumedSessionFile();
        }
      }
      if (feature('
```

### screens/ResumeConversation.tsx
```
resume_duration_ms: Math.
```

### screens/ResumeConversation.tsx
```
resumeStart)
      });
      setLogs([]);
      setResumeData({
        messages: result_3.
```

### screens/ResumeConversation.tsx
```
resumeData) {
    return <REPL debug={debug} commands={commands} initialTools={initialTools} initialMessages={resumeData.
```

### screens/ResumeConversation.tsx
```
ResumeWithRenameEnabled ? () => loadLogs(showAllProjects) : undefined} onLoadMore={loadMoreLogs} initialSearchQuery={initialSearchQuery} showAllProjects={showAllProjects} onToggleAllProjects={handleToggleAllProjects} onAgenticSearch={agenticSessionSearch} />;
}
function NoConversationsMessage() {
  const $ = _c(2);
  let t0;
  if ($[0] === Symbol.
```

### screens/ResumeConversation.tsx
```
resume, run:</Text>;
    $[2] = t3;
  } else {
    t3 = $[2];
  }
  let t4;
  if ($[3] !== command) {
    t4 = <Box flexDirection="
```

### server/directConnectManager.ts
```
continue
        }

        if (!isStdoutMessage(raw)) {
          continue
        }
        const parsed = raw

        // Handle control requests (permission requests)
        if (parsed.
```

### server/directConnectManager.ts
```
continue
        }

        // Forward SDK messages (assistant, result, system, etc.
```

### server/types.ts
```
resumed across server restarts.
```

### services/AgentSummary/agentSummary.ts
```
continue
        // Skip API error messages
        if (msg.
```

### services/AgentSummary/agentSummary.ts
```
continue
        }
        const textBlock = msg.
```

### services/PromptSuggestion/promptSuggestion.ts
```
continue
    const textBlock = msg.
```

### services/SessionMemory/prompts.ts
```
continue after the edits.
```

### services/SessionMemory/sessionMemory.ts
```
continue
    }

    if (message.
```

### services/SessionMemory/sessionMemoryUtils.ts
```
continue anyway
      return
    }

    await sleep(1000)
  }
}

/**
 * Get the current session memory content
 */
export async function getSessionMemoryContent(): Promise<string | null> {
  const fs = getFsImplementation()
  const memoryPath = getSessionMemoryPath()

  try {
    const content = await fs.
```

### services/analytics/firstPartyEventLogger.ts
```
pick up
 * changes to batch size, delay, endpoint, etc.
```

### services/analytics/firstPartyEventLoggingExporter.ts
```
continue loop to drain any newly queued events
      this.
```

### services/analytics/firstPartyEventLoggingExporter.ts
```
resume POSTs as soon as the GrowthBook
      // cache picks up the cleared flag.
```

### services/analytics/firstPartyEventLoggingExporter.ts
```
continue
      }

      // Extract event name
      const eventName =
        (attributes.
```

### services/analytics/firstPartyEventLoggingExporter.ts
```
continue
      }

      // Transform to 1P format
      const formatted = to1PEventFormat(
        coreMetadata,
        userMetadata,
        eventMetadata,
      )

      // _PROTO_* keys are PII-tagged values meant only for privileged BQ
      // columns.
```

### services/analytics/growthbook.ts
```
continues running
          resetGrowthBook()
          clientWrapper = getGrowthBookClient()
          if (!clientWrapper) {
            return null
          }
        }
      }
    }

    await clientWrapper.
```

### services/analytics/metadata.ts
```
continue
    const tokens = subcmd.
```

### services/analytics/metadata.ts
```
continue

    const firstToken = tokens[0]!
    const slashIdx = firstToken.
```

### services/analytics/metadata.ts
```
continue

    for (let i = 1; i < tokens.
```

### services/analytics/metadata.ts
```
continue
      const ext = getFileExtensionForAnalytics(arg)
      if (ext && !seen.
```

### services/api/claude.ts
```
continue feature — this one is sent to the API
  // so the model can pace itself.
```

### services/api/claude.ts
```
continue consuming the generator to ensure
  // logAPISuccessAndDuration gets called (which happens after all yields)
  let assistantMessage: AssistantMessage | undefined
  for await (const message of withStreamingVCR(messages, async function* () {
    yield* queryModel(
      messages,
      systemPrompt,
      thinkingConfig,
      tools,
      signal,
      options,
    )
  })) {
    if (messag
```

### services/api/claude.ts
```
continue
    for (const block of msg.
```

### services/api/claude.ts
```
continue from
              // where you left off.
```

### services/api/claude.ts
```
Continue to success logging below
      } catch (fallbackError) {
        // Propagate model-fallback signal to query.
```

### services/api/claude.ts
```
resume from there — with one marker they'
```

### services/api/claude.ts
```
continue
        }
        let cloned = false
        for (let j = 0; j < msg.
```

### services/api/client.ts
```
continue

    // Parse header in format "
```

### services/api/client.ts
```
continue
    const name = headerString.
```

### services/api/errorUtils.ts
```
resume), the error object may
  // be a plain object without a `
```

### services/api/errors.ts
```
continue
      const content = msg.
```

### services/api/errors.ts
```
continue
      if (msg.
```

### services/api/errors.ts
```
continue
      const content = msg.
```

### services/api/errors.ts
```
continue

      switch (msg.
```

### services/api/filesApi.ts
```
continue retrying
 */
type RetryResult<T> = { done: true; value: T } | { done: false; error?: string }

/**
 * Executes an operation with exponential backoff retry logic
 *
 * @param operation - Operation name for logging
 * @param attemptFn - Function to execute on each attempt, returns RetryResult
 * @returns The successful result value
 * @throws Error if all retries exhausted
 */
async functio
```

### services/api/filesApi.ts
```
continue
    }

    const fileId = spec.
```

### services/api/filesApi.ts
```
continue
    }

    files.
```

### services/api/grove.ts
```
continue
      writeToStderr(
        '
```

### services/api/promptCacheBreakDetection.ts
```
continue
          if (newHashes[name] !== prev.
```

### services/api/sessionIngress.ts
```
continue // retry with updated lastUuid
      }

      if (response.
```

### services/api/withRetry.ts
```
continue
        }

        const retryAfterMs = getRetryAfterMs(error)
        if (retryAfterMs !== null && retryAfterMs < SHORT_RETRY_THRESHOLD_MS) {
          // Short retry-after: wait and retry with fast mode still active
          // to preserve prompt cache (same model name on retry).
```

### services/api/withRetry.ts
```
continue
        }
        // Long or unknown retry-after: enter cooldown (switches to standard
        // speed model), with a minimum floor to avoid flip-flopping.
```

### services/api/withRetry.ts
```
continue
      }

      // Fast mode fallback: if the API rejects the fast mode parameter
      // (e.
```

### services/api/withRetry.ts
```
continue
      }

      // Non-foreground sources bail immediately on 529 — no retry amplification
      // during capacity cascades.
```

### services/api/withRetry.ts
```
continue
        }
      }

      // For other errors, proceed with normal retry logic
      // Get retry-after header if available
      const retryAfter = getRetryAfter(error)
      let delayMs: number
      if (persistent && error instanceof APIError && error.
```

### services/compact/compact.ts
```
continues operating in plan mode after compaction
    const planModeAttachment = await createPlanModeAttachmentIfNeeded(context)
    if (planModeAttachment) {
      postCompactFileAttachments.
```

### services/compact/compact.ts
```
resume to show the auto-generated title
    // instead of the user-set session name.
```

### services/compact/compact.ts
```
continue
      }

      logForDebugging(
        `
```

### services/compact/compact.ts
```
continues to operate in plan mode after compaction
 * (otherwise it would lose the plan mode instructions since those are
 * normally only injected on tool-use turns via getAttachmentMessages).
```

### services/compact/compact.ts
```
continue
    }
    for (const block of message.
```

### services/compact/compact.ts
```
continue
    }
    for (const block of message.
```

### services/compact/compact.ts
```
continue
      }
      const input = block.
```

### services/compact/compact.ts
```
continue with other checks
  }

  // Exclude all types of claude.
```

### services/compact/grouping.ts
```
resume/truncation) the fork'
```

### services/compact/grouping.ts
```
resume-from-partial-batch or max_tokens truncation) — and in that
  // case it pins the gate shut forever, merging all subsequent rounds into
  // one group.
```

### services/compact/microCompact.ts
```
continue
    }

    if (!Array.
```

### services/compact/microCompact.ts
```
continue
    }

    for (const block of message.
```

### services/compact/prompt.ts
```
continue the work in subsequent messages.
```

### services/compact/prompt.ts
```
continue the work]

</summary>
</example>

Please provide your summary following this structure, ensuring precision and thoroughness in your response.
```

### services/compact/prompt.ts
```
continued from a previous conversation that ran out of context.
```

### services/compact/prompt.ts
```
Continue the conversation from where it left off without asking the user any further questions.
```

### services/compact/prompt.ts
```
Resume directly — do not acknowledge the summary, do not recap what was happening, do not preface with "
```

### services/compact/prompt.ts
```
Pick up the last task as if the break never happened.
```

### services/compact/prompt.ts
```
Continue your work loop: pick up where you left off based on the summary above.
```

### services/compact/sessionMemoryCompact.ts
```
Resumed session: lastSummarizedMessageId is not set but session memory has content,
 *    keep all messages but use session memory as the summary
 */
export async function trySessionMemoryCompaction(
  messages: Message[],
  agentId?: AgentId,
  autoCompactThreshold?: number,
): Promise<CompactionResult | null> {
  if (!shouldUseSessionMemoryCompaction()) {
    return null
  }

  // Initialize con
```

### services/compact/sessionMemoryCompact.ts
```
Resumed session case: session memory has content but we don'
```

### services/extractMemories/extractMemories.ts
```
continue
    }
    if (isModelVisibleMessage(message)) {
      n++
    }
  }
  // If sinceUuid was not found (e.
```

### services/extractMemories/extractMemories.ts
```
continue
    }
    if (message.
```

### services/extractMemories/extractMemories.ts
```
continue
    }
    const content = (message as AssistantMessage).
```

### services/extractMemories/extractMemories.ts
```
continue
    }
    for (const block of content) {
      const filePath = getWrittenFilePath(block)
      if (filePath !== undefined && isAutoMemPath(filePath)) {
        return true
      }
    }
  }
  return false
}

// ============================================================================
// Tool Permissions
// ============================================================================

f
```

### services/extractMemories/extractMemories.ts
```
continue
    }
    const content = (message as AssistantMessage).
```

### services/extractMemories/extractMemories.ts
```
continue
    }
    for (const block of content) {
      const filePath = getWrittenFilePath(block)
      if (filePath !== undefined) {
        paths.
```

### services/lsp/LSPClient.ts
```
Continue to cleanup despite shutdown failure
      } finally {
        // Always cleanup resources, even if shutdown/exit failed
        if (connection) {
          try {
            connection.
```

### services/lsp/LSPDiagnosticRegistry.ts
```
continue
        }

        seenDiagnostics.
```

### services/lsp/LSPDiagnosticRegistry.ts
```
continue - failure to track shouldn'
```

### services/lsp/LSPServerInstance.ts
```
continue
        }

        // Non-retryable error or max retries exceeded
        break
      }
    }

    // All retries failed or non-retryable error
    const requestError = new Error(
      `
```

### services/lsp/LSPServerManager.ts
```
Continue with other servers - don'
```

### services/lsp/passiveFeedback.ts
```
continue // Skip this server but track the failure
      }

      // Errors are isolated to avoid breaking other servers
      serverInstance.
```

### services/mcp/auth.ts
```
continue
              logMCPDebug(
                serverName,
                `
```

### services/mcp/auth.ts
```
continue
        }
        logMCPDebug(
          this.
```

### services/mcp/client.ts
```
continue
                  }

                  // Emit progress when tool fails
                  if (onProgress && toolUseId) {
                    onProgress({
                      toolUseID: toolUseId,
                      data: {
                        type: '
```

### services/mcp/client.ts
```
continue
        }

        // Resolve the URL elicitation via callback (print/SDK mode) or queue (REPL mode).
```

### services/mcp/config.ts
```
continue
    }
    const manualDup = manualSigs.
```

### services/mcp/config.ts
```
continue
    }
    const pluginDup = seenPluginSigs.
```

### services/mcp/config.ts
```
continue
    }
    seenPluginSigs.
```

### services/mcp/config.ts
```
continue
    const sig = getMcpServerSignature(config)
    if (sig && !manualSigs.
```

### services/mcp/config.ts
```
continue
    }
    servers[name] = config
  }
  return { servers, suppressed }
}

/**
 * Convert a URL pattern with wildcards to a RegExp
 * Supports * as wildcard matching any characters
 * Examples:
 *   "
```

### services/mcp/config.ts
```
continue
        }

        if (config.
```

### services/mcp/config.ts
```
continue
      }
      filtered[name] = serverConfig
    }

    return { servers: filtered, errors: [] }
  }

  // Load other scopes — unless the managed policy locks MCP to plugin-only.
```

### services/mcp/config.ts
```
continue
    mcpErrors.
```

### services/mcp/config.ts
```
continue
    }
    filtered[name] = serverConfig as ScopedMcpServerConfig
  }

  return { servers: filtered, errors: mcpErrors }
}

/**
 * Get all MCP configurations across all scopes, including claude.
```

### services/mcp/oauthPort.ts
```
continue
    }
  }

  // If random selection failed, try the fallback port
  try {
    await new Promise<void>((resolve, reject) => {
      const testServer = createServer()
      testServer.
```

### services/mcp/useManageMCPConnections.ts
```
pick up newly
  // enabled plugin MCP servers.
```

### services/mcp/utils.ts
```
continue

    for (const spec of agent.
```

### services/mcp/utils.ts
```
continue

      // Inline definition as { [name]: config }
      const entries = Object.
```

### services/mcp/utils.ts
```
continue

      const [serverName, serverConfig] = entries[0]!
      const existing = serverMap.
```

### services/mcp/xaa.ts
```
continue
    }
    if (
      candidate.
```

### services/mcp/xaa.ts
```
continue
    }
    asMeta = candidate
    break
  }
  if (!asMeta) {
    throw new Error(
      `
```

### services/plugins/pluginOperations.ts
```
continue

    for (const key of Object.
```

### services/plugins/pluginOperations.ts
```
continue
      }
    }
  }

  if (!foundPlugin || !foundMarketplace) {
    const location = marketplaceName
      ? `
```

### services/policyLimits/index.ts
```
continues without restrictions
 * - API returns empty restrictions for users without policy limits
 */

import axios from '
```

### services/policyLimits/index.ts
```
continue to check OAuth
  }

  // For OAuth users, check if they have Claude.
```

### services/policyLimits/index.ts
```
continue to check OAuth
  }

  // Fall back to OAuth tokens (for Claude.
```

### services/policyLimits/index.ts
```
continues without restrictions
 * Also starts background polling to pick up changes mid-session
 */
export async function loadPolicyLimits(): Promise<void> {
  if (isPolicyLimitsEligible() && !loadingCompletePromise) {
    loadingCompletePromise = new Promise(resolve => {
      loadingCompleteResolve = resolve
    })
  }

  try {
    await fetchAndLoadPolicyLimits()

    if (isPolicyLimitsEligible
```

### services/rateLimitMessages.ts
```
resume
    let overageResetMessage = '
```

### services/remoteManagedSettings/index.ts
```
continues without remote settings
 * - API returns empty settings for users without managed settings
 */

import axios from '
```

### services/remoteManagedSettings/index.ts
```
continue to check OAuth
  }

  // Fall back to OAuth tokens (for Claude.
```

### services/remoteManagedSettings/index.ts
```
continue without remote settings
      return null
    }

    // Handle 304 Not Modified - cached settings are still valid
    if (result.
```

### services/remoteManagedSettings/index.ts
```
continue without remote settings
    return null
  }
}

/**
 * Load remote settings during CLI initialization
 * Fails open - if fetch fails, continues without remote settings
 * Also starts background polling to pick up settings changes mid-session
 *
 * This function sets up a promise that other systems can await via
 * waitForRemoteManagedSettingsToLoad() to ensure they don'
```

### services/remoteManagedSettings/index.ts
```
pick up settings changes mid-session
    if (isRemoteManagedSettingsEligible()) {
      startBackgroundPolling()
    }

    // Trigger hot-reload if settings were loaded (new or from cache).
```

### services/remoteManagedSettings/index.ts
```
continues without remote settings
 */
export async function refreshRemoteManagedSettings(): Promise<void> {
  // Clear caches first
  await clearRemoteManagedSettingsCache()

  // If not enabled, notify that policy settings changed (to empty)
  if (!isRemoteManagedSettingsEligible()) {
    settingsChangeDetector.
```

### services/remoteManagedSettings/index.ts
```
continue
  }
}

/**
 * Start background polling for remote settings
 * Polls every hour to pick up settings changes mid-session
 */
export function startBackgroundPolling(): void {
  if (pollingIntervalId !== null) {
    return
  }

  if (!isRemoteManagedSettingsEligible()) {
    return
  }

  pollingIntervalId = setInterval(() => {
    void pollRemoteSettings()
  }, POLLING_INTERVAL_MS)
  polling
```

### services/remoteManagedSettings/securityCheck.tsx
```
continue, false if we should stop
 */
export function handleSecurityCheckResult(result: SecurityCheckResult): boolean {
  if (result === '
```

### services/settingsSync/index.ts
```
pick up new content
  if (settingsWritten) {
    resetSettingsCache()
  }
  if (memoryWritten) {
    clearMemoryFileCaches()
  }

  logForDiagnosticsNoPII('
```

### services/teamMemorySync/index.ts
```
resumes from the uncommitted tail (those keys still differ).
```

### services/teamMemorySync/secretScanner.ts
```
continue
    }
    if (rule.
```

### services/tips/tipRegistry.ts
```
continue or claude --resume to resume a conversation'
```

### services/tips/tipRegistry.ts
```
Continue your session in Claude Code Desktop with ${blue('
```

### services/tools/StreamingToolExecutor.ts
```
continue

      if (this.
```

### services/tools/StreamingToolExecutor.ts
```
continue
      }

      if (tool.
```

### services/tools/toolHooks.ts
```
continue
        }

        // For JSON {decision:"
```

### services/tools/toolHooks.ts
```
continue
        }

        // Skip hook_blocking_error in result.
```

### services/tools/toolHooks.ts
```
continue
        if (result.
```

### services/tools/toolOrchestration.ts
```
continue
        }
        for (const modifier of modifiers) {
          currentContext = modifier(currentContext)
        }
      }
      yield { newContext: currentContext }
    } else {
      // Run non-read-only batch serially
      for await (const update of runToolsSerially(
        blocks,
        assistantMessages,
        canUseTool,
        currentContext,
      )) {
        if (update.
```

### services/vcr.ts
```
resumed sessions would treat different responses as duplicates.
```

### services/voice.ts
```
continues until the caller explicitly calls
        // stopRecording() (e.
```

### services/voiceStreamSTT.ts
```
resume()
    req.
```

### skills/bundled/claudeApi.ts
```
continue
    for (const indicator of indicators) {
      if (indicator.
```

### skills/bundled/claudeApi.ts
```
continue
    sections.
```

### skills/bundled/scheduleRemoteAgents.ts
```
continue
    }
    if (client.
```

### skills/bundled/scheduleRemoteAgents.ts
```
continue
    }
    const uuid = taggedIdToUUID(client.
```

### skills/bundled/scheduleRemoteAgents.ts
```
continue
    }
    connectors.
```

### skills/bundled/updateConfig.ts
```
resume, compact) |
| PreCompact | "
```

### skills/bundledSkills.ts
```
continues to work, just without the base-directory prefix).
```

### skills/loadSkillsDir.ts
```
continue
      const { skill } = entry

      const fileId = fileIds[i]
      if (fileId === null || fileId === undefined) {
        deduplicatedSkills.
```

### skills/loadSkillsDir.ts
```
continue
      }

      const existingSource = seenFileIds.
```

### skills/loadSkillsDir.ts
```
continue
      }

      seenFileIds.
```

### skills/loadSkillsDir.ts
```
continue
          }
          newDirs.
```

### skills/loadSkillsDir.ts
```
continue
        }
      }

      // Move to parent
      const parent = dirname(currentDir)
      if (parent === currentDir) break // Reached root
      currentDir = parent
    }
  }

  // Sort by path depth (deepest first) so skills closer to the file take precedence
  return newDirs.
```

### skills/loadSkillsDir.ts
```
continue
    }

    const skillIgnore = ignore().
```

### skills/loadSkillsDir.ts
```
continue
      }

      if (skillIgnore.
```

### state/AppStateStore.ts
```
pick up newly-enabled plugin MCP servers.
```

### state/AppStateStore.ts
```
resume so clicks stay on the display the model last saw.
```

### tasks/LocalAgentTask/LocalAgentTask.tsx
```
resumeAgentBackground route the prompt to the
 * agent'
```

### tasks/LocalMainSessionTask.ts
```
continues running in the background
 * - The UI clears to a fresh prompt
 * - A notification is sent when the query completes
 *
 * This reuses the LocalAgentTask state structure since the behavior is similar.
```

### tasks/LocalMainSessionTask.ts
```
continues running normally.
```

### tasks/LocalMainSessionTask.ts
```
continue
        }

        bgMessages.
```

### tasks/LocalShellTask/LocalShellTask.tsx
```
Continue\?/i, /Overwrite\?/i];
export function looksLikePrompt(tail: string): boolean {
  const lastLine = tail.
```

### tasks/LocalShellTask/LocalShellTask.tsx
```
continues receiving data automatically
  if (!shellCommand.
```

### tasks/RemoteAgentTask/RemoteAgentTask.tsx
```
resume via the sidecar'
```

### tasks/RemoteAgentTask/RemoteAgentTask.tsx
```
continue;
    const fullText = extractTextContent(msg.
```

### tasks/RemoteAgentTask/RemoteAgentTask.tsx
```
continue;
    const fullText = extractTextContent(msg.
```

### tasks/RemoteAgentTask/RemoteAgentTask.tsx
```
continue;
    const fullText = extractTextContent(msg.
```

### tasks/RemoteAgentTask/RemoteAgentTask.tsx
```
resume can reconnect to
  // still-running remote sessions.
```

### tasks/RemoteAgentTask/RemoteAgentTask.tsx
```
continue;
    }
    if (remoteStatus === '
```

### tasks/RemoteAgentTask/RemoteAgentTask.tsx
```
continue;
    }
    const taskState: RemoteAgentTaskState = {
      .
```

### tasks/RemoteAgentTask/RemoteAgentTask.tsx
```
continue polling
      }
    }

    // Continue polling
    if (isRunning) {
      setTimeout(poll, POLL_INTERVAL_MS);
    }
  };

  // Start polling
  void poll();

  // Return cleanup function
  return () => {
    isRunning = false;
  };
}

/**
 * RemoteAgentTask - Handles remote Claude.
```

### tools/AgentTool/AgentTool.tsx
```
resume: false,
      is_async: (run_in_background === true || selectedAgent.
```

### tools/AgentTool/AgentTool.tsx
```
ContinueAnimation: true,
                showSpinner: true
              });
            }

            // Race between next message and background signal
            // If background tasks are disabled, just await the next message directly
            const nextMessagePromise = agentIterator.
```

### tools/AgentTool/AgentTool.tsx
```
Continue agent in background and return async result
                void runWithAgentContext(syncAgentContext, async () => {
                  let stopBackgroundedSummarization: (() => void) | undefined;
                  try {
                    // Clean up the foreground iterator so its finally block runs
                    // (releases MCP connections, session hooks, prompt cache tracking, e
```

### tools/AgentTool/AgentTool.tsx
```
continue;
            }
            const {
              result
            } = raceResult;
            if (result.
```

### tools/AgentTool/AgentTool.tsx
```
continue;
            }

            // Increment token count in spinner for assistant messages
            // Subagent streaming events are filtered out in runAgent.
```

### tools/AgentTool/AgentTool.tsx
```
continue;
                }

                // Forward progress updates
                if (onProgress) {
                  onProgress({
                    toolUseID: `
```

### tools/AgentTool/AgentTool.tsx
```
continue this agent.
```

### tools/AgentTool/AgentTool.tsx
```
continued via SendMessage
      // — the agentId hint and <usage> block are dead weight (~135 chars ×
      // 34M Explore runs/week ≈ 1-2 Gtok/week).
```

### tools/AgentTool/AgentTool.tsx
```
resume compat — missing means show trailer.
```

### tools/AgentTool/AgentTool.tsx
```
continue this agent)${worktreeInfoText}
<usage>total_tokens: ${data.
```

### tools/AgentTool/UI.tsx
```
continue;
    }
    if (pm.
```

### tools/AgentTool/UI.tsx
```
continue;
    }
    const info = getSearchOrReadInfo(msg, tools, toolUseByID);
    if (info && (info.
```

### tools/AgentTool/agentMemorySnapshot.ts
```
continue
      const content = await readFile(join(snapshotMemDir, dirent.
```

### tools/AgentTool/agentToolUtils.ts
```
continue
      }
      // For main thread, filtering was skipped so Agent is in availableToolMap —
      // fall through to normal resolution below.
```

### tools/AgentTool/agentToolUtils.ts
```
resume replays
    // results verbatim without re-validation).
```

### tools/AgentTool/agentToolUtils.ts
```
continue
      const textBlocks = m.
```

### tools/AgentTool/agentToolUtils.ts
```
continue
    const text = extractTextContent(m.
```

### tools/AgentTool/agentToolUtils.ts
```
resumeAgentBackground.
```

### tools/AgentTool/built-in/claudeCodeGuideAgent.ts
```
continue via ${SEND_MESSAGE_TOOL_NAME}.
```

### tools/AgentTool/built-in/statuslineSetup.ts
```
continue to make changes to the status line.
```

### tools/AgentTool/constants.ts
```
resumed sessions)
export const LEGACY_AGENT_TOOL_NAME = '
```

### tools/AgentTool/prompt.ts
```
Continue with other work or respond to the user instead.
```

### tools/AgentTool/prompt.ts
```
continue a previously spawned agent, use ${SEND_MESSAGE_TOOL_NAME} with the agent'
```

### tools/AgentTool/prompt.ts
```
resumes with its full context preserved.
```

### tools/AgentTool/resumeAgent.ts
```
ResumeAgentResult = {
  agentId: string
  description: string
  outputFile: string
}
export async function resumeAgentBackground({
  agentId,
  prompt,
  toolUseContext,
  canUseTool,
  invokingRequestId,
}: {
  agentId: string
  prompt: string
  toolUseContext: ToolUseContext
  canUseTool: CanUseToolFn
  invokingRequestId?: string
}): Promise<ResumeAgentResult> {
  const startTime = Date.
```

### tools/AgentTool/resumeAgent.ts
```
resumedMessages = filterWhitespaceOnlyAssistantMessages(
    filterOrphanedThinkingOnlyMessages(
      filterUnresolvedToolUses(transcript.
```

### tools/AgentTool/resumeAgent.ts
```
resumedReplacementState = reconstructForSubagentResume(
    toolUseContext.
```

### tools/AgentTool/resumeAgent.ts
```
resumedMessages,
    transcript.
```

### tools/AgentTool/resumeAgent.ts
```
resumedWorktreePath = meta?.
```

### tools/AgentTool/resumeAgent.ts
```
Resumed worktree ${meta.
```

### tools/AgentTool/resumeAgent.ts
```
resumedWorktreePath) {
    // Bump mtime so stale-worktree cleanup doesn'
```

### tools/AgentTool/resumeAgent.ts
```
resumed worktree (#22355)
    const now = new Date()
    await fsp.
```

### tools/AgentTool/resumeAgent.ts
```
resumedWorktreePath, now, now)
  }

  // Skip filterDeniedAgents re-gating — original spawn already passed permission checks
  let selectedAgent: AgentDefinition
  let isResumedFork = false
  if (meta?.
```

### tools/AgentTool/resumeAgent.ts
```
ResumedFork = true
  } else if (meta?.
```

### tools/AgentTool/resumeAgent.ts
```
ResumedFork) {
    if (toolUseContext.
```

### tools/AgentTool/resumeAgent.ts
```
resume fork agent: unable to reconstruct parent system prompt'
```

### tools/AgentTool/resumeAgent.ts
```
ResumedFork
    ? toolUseContext.
```

### tools/AgentTool/resumeAgent.ts
```
resumedMessages,
      createUserMessage({ content: prompt }),
    ],
    toolUseContext,
    canUseTool,
    isAsync: true,
    querySource: getQuerySourceForAgent(
      selectedAgent.
```

### tools/AgentTool/resumeAgent.ts
```
resume: pass parent'
```

### tools/AgentTool/resumeAgent.ts
```
resumedWorktreePath.
```

### tools/AgentTool/resumeAgent.ts
```
ResumedFork
      ? { systemPrompt: forkParentSystemPrompt }
      : undefined,
    availableTools: workerTools,
    // Transcript already contains the parent context slice from the
    // original fork.
```

### tools/AgentTool/resumeAgent.ts
```
ResumedFork && { useExactTools: true }),
    // Re-persist so metadata survives runAgent'
```

### tools/AgentTool/resumeAgent.ts
```
resumedWorktreePath,
    description: meta?.
```

### tools/AgentTool/resumeAgent.ts
```
resumedReplacementState,
  }

  // Skip name-registry write — original entry persists from the initial spawn
  const agentBackgroundTask = registerAsyncAgent({
    agentId,
    description: uiDescription,
    prompt,
    selectedAgent,
    setAppState: rootSetAppState,
    toolUseId: toolUseContext.
```

### tools/AgentTool/resumeAgent.ts
```
resumedWorktreePath ? runWithCwdOverride(resumedWorktreePath, fn) : fn()

  void runWithAgentContext(asyncAgentContext, () =>
    wrapWithCwd(() =>
      runAsyncAgentLifecycle({
        taskId: agentBackgroundTask.
```

### tools/AgentTool/runAgent.ts
```
continue
      }
    } else {
      // Inline definition as { [name]: config }
      // These are agent-specific servers that should be cleaned up
      const entries = Object.
```

### tools/AgentTool/runAgent.ts
```
continue
      }
      const [serverName, serverConfig] = entries[0]!
      name = serverName
      config = {
        .
```

### tools/AgentTool/runAgent.ts
```
resumed sidechain transcript so
   * the same tool results are re-replaced (prompt cache stability).
```

### tools/AgentTool/runAgent.ts
```
resume can restore the correct cwd.
```

### tools/AgentTool/runAgent.ts
```
continue
      }

      const skill = getCommand(resolvedName, allSkills)
      if (skill.
```

### tools/AgentTool/runAgent.ts
```
continue
      }
      validSkills.
```

### tools/AgentTool/runAgent.ts
```
resume can route correctly when subagent_type is omitted.
```

### tools/AgentTool/runAgent.ts
```
continue
      }

      // Yield attachment messages (e.
```

### tools/AgentTool/runAgent.ts
```
continue
      }

      if (isRecordableMessage(message)) {
        // Record only the new message with correct parent (O(1) per message)
        await recordSidechainTranscript(
          [message],
          agentId,
          lastRecordedUuid,
        ).
```

### tools/AskUserQuestionTool/AskUserQuestionTool.tsx
```
continue with the user'
```

### tools/BashTool/BashTool.tsx
```
continue;
    }
    if (part === '
```

### tools/BashTool/BashTool.tsx
```
continue;
    }
    if (part === '
```

### tools/BashTool/BashTool.tsx
```
continue;
    }
    const baseCommand = part.
```

### tools/BashTool/BashTool.tsx
```
continue;
    }
    if (BASH_SEMANTIC_NEUTRAL_COMMANDS.
```

### tools/BashTool/BashTool.tsx
```
continue;
    }
    hasNonNeutralCommand = true;
    const isPartSearch = BASH_SEARCH_COMMANDS.
```

### tools/BashTool/BashTool.tsx
```
continue;
    }
    if (part === '
```

### tools/BashTool/BashTool.tsx
```
continue;
    }
    if (part === '
```

### tools/BashTool/BashTool.tsx
```
continue;
    }
    const baseCommand = part.
```

### tools/BashTool/BashTool.tsx
```
continue;
    }
    if (lastOperator === '
```

### tools/BashTool/BashTool.tsx
```
continue;
    }
    hasNonFallbackCommand = true;
    if (!BASH_SILENT_COMMANDS.
```

### tools/BashTool/BashTool.tsx
```
ContinueAnimation: true,
          showSpinner: true
        });
      }
      yield {
        type: '
```

### tools/BashTool/bashCommandHelpers.ts
```
continue // Skip empty segments

    const segmentResult = await bashToolHasPermissionFn({
      .
```

### tools/BashTool/bashPermissions.ts
```
continue
    if (blocklist?.
```

### tools/BashTool/bashPermissions.ts
```
continue
        }
        // Try stripping env vars
        const envStripped = stripAllLeadingEnvVars(cmd)
        if (!seen.
```

### tools/BashTool/bashPermissions.ts
```
continue
    subcommands.
```

### tools/BashTool/bashPermissions.ts
```
Continue: () => boolean
  onAllow: (decisionReason: PermissionDecisionReason) => void
  onComplete?: () => void
}

/**
 * Execute the bash allow classifier check asynchronously.
```

### tools/BashTool/bashPermissions.ts
```
continue and handle approval
 */
export async function executeAsyncClassifierCheck(
  pendingCheck: { command: string; cwd: string; descriptions: string[] },
  signal: AbortSignal,
  isNonInteractiveSession: boolean,
  callbacks: AsyncClassifierCheckCallbacks,
): Promise<void> {
  const { command, cwd, descriptions } = pendingCheck
  const speculativeResult = consumeSpeculativeClassifierCheck(comm
```

### tools/BashTool/bashPermissions.ts
```
Continue()) return

  if (
    feature('
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === "
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (!inSingleQuote) withDoubleQuotes += char
    if (!inSingleQuote && !inDoubleQuote) fullyUnquoted += char
    if (!inSingleQuote && !inDoubleQuote) unquotedKeepQuoteChars += char
  }

  return { withDoubleQuotes, fullyUnquoted, unquotedKeepQuoteChars }
}

function stripSafeRedirections(content: string): string {
  // SECURITY: All three patterns MUST have a trailing boundary
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    // Check if current character matches
    if (content[i] === char) {
      return true // Found unescaped occurrence
    }

    i++
  }

  return false // No unescaped occurrences found
}

function validateEmpty(context: ValidationContext): PermissionResult {
  if (!context.
```

### tools/BashTool/bashSecurity.ts
```
continue
      if (inner.
```

### tools/BashTool/bashSecurity.ts
```
continue
    const delimiter = match[2] || match[3]
    if (!delimiter) continue
    const isDash = match[1] === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    if (!/^[ \t]*$/.
```

### tools/BashTool/bashSecurity.ts
```
continue

    const bodyStart = operatorEnd + openLineEnd + 1
    const bodyLines = command.
```

### tools/BashTool/bashSecurity.ts
```
continue
        }
        if (c === '
```

### tools/BashTool/bashSecurity.ts
```
continue
        }
        if (!inSQ && !inDQ) unquoted += c
      }
      if (/[<>]/.
```

### tools/BashTool/bashSecurity.ts
```
continue
    }
    if (c === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }
    if (c === "
```

### tools/BashTool/bashSecurity.ts
```
continue
    }
    if (c === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }
    if (c === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    // SECURITY: Only treat backslash as escape OUTSIDE single quotes.
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (currentChar === "
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (currentChar === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    // Only look for flags when not inside quoted strings
    // This prevents false positives like: make test TEST="
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    // Look for whitespace followed by quote that contains a dash (potential flag obfuscation)
    // SECURITY: Block ANY quoted content starting with dash - err on side of safety
    // Catches: "
```

### tools/BashTool/bashSecurity.ts
```
continue after quote)
      //   3.
```

### tools/BashTool/bashSecurity.ts
```
continue a flag after a closing quote.
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === "
```

### tools/BashTool/bashSecurity.ts
```
continue
    }
  }

  return false
}

function validateBackslashEscapedWhitespace(
  context: ValidationContext,
): PermissionResult {
  if (hasBackslashEscapedWhitespace(context.
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    // Quote toggles come AFTER backslash handling (backslash already skipped
    // any escaped quote char, so these toggles only fire on unescaped quotes).
```

### tools/BashTool/bashSecurity.ts
```
continue
    }
    if (char === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }
  }

  return false
}

function validateBackslashEscapedOperators(
  context: ValidationContext,
): PermissionResult {
  // Tree-sitter path: if tree-sitter confirms no actual operator nodes exist
  // in the AST, then any \; is just an escaped character in a word argument
  // (e.
```

### tools/BashTool/bashSecurity.ts
```
continue
    if (isEscapedAtPosition(content, i)) continue

    // Find matching unescaped `
```

### tools/BashTool/bashSecurity.ts
```
continue

    // Check for `
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (inSingleQuote) {
      if (char === "
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (inDoubleQuote) {
      if (char === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === "
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    // Unquoted `
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === "
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    if (char === '
```

### tools/BashTool/bashSecurity.ts
```
continue
    }

    // A newline inside quotes: the NEXT line (from bash'
```

### tools/BashTool/bashSecurity.ts
```
continue
    // Skip Zsh precommand modifiers (they don'
```

### tools/BashTool/bashSecurity.ts
```
continue
    baseCmd = token
    break
  }

  if (ZSH_DANGEROUS_COMMANDS.
```

### tools/BashTool/bashSecurity.ts
```
Continue running validators; if any
  // misparsing validator fires, return THAT (with the flag).
```

### tools/BashTool/bashSecurity.ts
```
continue
      }
      return { .
```

### tools/BashTool/bashSecurity.ts
```
continue using bashCommandIsSafe().
```

### tools/BashTool/bashSecurity.ts
```
continue
      }
      return { .
```

### tools/BashTool/pathValidation.ts
```
continue

    if (!afterDoubleDash && arg === '
```

### tools/BashTool/pathValidation.ts
```
continue
    }

    if (!afterDoubleDash && arg.
```

### tools/BashTool/pathValidation.ts
```
continue
    }

    // First non-flag is pattern, rest are paths
    if (!patternFound) {
      patternFound = true
      continue
    }
    paths.
```

### tools/BashTool/pathValidation.ts
```
continue

      if (afterDoubleDash) {
        paths.
```

### tools/BashTool/pathValidation.ts
```
continue
      }

      if (arg === '
```

### tools/BashTool/pathValidation.ts
```
continue
      }

      // Handle flags
      if (arg.
```

### tools/BashTool/pathValidation.ts
```
continue

        // Mark that we'
```

### tools/BashTool/pathValidation.ts
```
continue
      }

      // Only collect non-flag arguments before first non-global flag
      if (!foundNonGlobalFlag) {
        paths.
```

### tools/BashTool/pathValidation.ts
```
continue
      }

      const arg = args[i]
      if (!arg) continue

      if (!afterDoubleDash && arg === '
```

### tools/BashTool/pathValidation.ts
```
continue
      }

      // Handle flags (only before `
```

### tools/BashTool/pathValidation.ts
```
continue
      }

      // First non-flag is the script (if not already found via -e/-f)
      if (!scriptFound) {
        scriptFound = true
        continue
      }

      // Rest are file paths
      paths.
```

### tools/BashTool/pathValidation.ts
```
continue

      if (!afterDoubleDash && arg === '
```

### tools/BashTool/pathValidation.ts
```
continue
      }

      if (!afterDoubleDash && arg.
```

### tools/BashTool/pathValidation.ts
```
continue
      }

      // First non-flag is filter, rest are file paths
      if (!filterFound) {
        filterFound = true
        continue
      }
      paths.
```

### tools/BashTool/pathValidation.ts
```
continue
    }
    const { allowed, resolvedPath, decisionReason } = validatePath(
      target,
      cwd,
      toolPermissionContext,
      '
```

### tools/BashTool/readOnlyValidation.ts
```
continue checking flags after `
```

### tools/BashTool/readOnlyValidation.ts
```
continue
    // Reject any token containing $ (variable expansion)
    if (token.
```

### tools/BashTool/readOnlyValidation.ts
```
continue
    }

    // SECURITY: Only treat backslash as escape OUTSIDE single quotes.
```

### tools/BashTool/readOnlyValidation.ts
```
continue
    }

    // Update quote state
    if (currentChar === "
```

### tools/BashTool/readOnlyValidation.ts
```
continue
    }

    if (currentChar === '
```

### tools/BashTool/readOnlyValidation.ts
```
continue
    }

    // Inside single quotes: everything is literal.
```

### tools/BashTool/readOnlyValidation.ts
```
continue
    }

    // Check `
```

### tools/BashTool/readOnlyValidation.ts
```
continue
    }

    // Check for glob characters outside all quotes.
```

### tools/BashTool/sedEditParser.ts
```
continue
    }
    if (arg.
```

### tools/BashTool/sedEditParser.ts
```
continue
    }

    // Handle extended regex flags
    if (arg === '
```

### tools/BashTool/sedEditParser.ts
```
continue
    }

    // Handle -e flag with expression
    if (arg === '
```

### tools/BashTool/sedEditParser.ts
```
continue
      }
      return null
    }
    if (arg.
```

### tools/BashTool/sedEditParser.ts
```
continue
    }

    // Skip other flags we don'
```

### tools/BashTool/sedEditParser.ts
```
continue
    }

    if (char === '
```

### tools/BashTool/sedEditParser.ts
```
continue
    }

    if (state === '
```

### tools/BashTool/sedValidation.ts
```
continue
    }
    if (rest[i] === '
```

### tools/BashTool/sedValidation.ts
```
continue

      // If it'
```

### tools/BashTool/sedValidation.ts
```
continue

      // Handle -e flag followed by expression
      if ((arg === '
```

### tools/BashTool/sedValidation.ts
```
continue
      }

      // Handle --expression=value format
      if (arg.
```

### tools/BashTool/sedValidation.ts
```
continue
      }

      // Handle -e=value format (non-standard but defense in depth)
      if (arg.
```

### tools/BashTool/sedValidation.ts
```
continue
      }

      // Skip other flags
      if (arg.
```

### tools/BashTool/sedValidation.ts
```
continue

      argCount++

      // If we used -e flags, ALL non-flag arguments are file arguments
      if (hasEFlag) {
        return true
      }

      // If we didn'
```

### tools/BashTool/sedValidation.ts
```
continue

      // Handle -e flag followed by expression
      if ((arg === '
```

### tools/BashTool/sedValidation.ts
```
continue
      }

      // Handle --expression=value format
      if (arg.
```

### tools/BashTool/sedValidation.ts
```
continue
      }

      // Handle -e=value format (non-standard but defense in depth)
      if (arg.
```

### tools/BashTool/sedValidation.ts
```
continue
      }

      // Skip other flags
      if (arg.
```

### tools/BashTool/sedValidation.ts
```
continue

      // If we haven'
```

### tools/BashTool/sedValidation.ts
```
continue
      }

      // If we'
```

### tools/BashTool/sedValidation.ts
```
continue
    }

    // In acceptEdits mode, allow file writes (-i flag) but still block dangerous operations
    const allowFileWrites = toolPermissionContext.
```

### tools/BriefTool/BriefTool.ts
```
resumed sessions replay pre-attachment
// outputs verbatim and a required field would crash the UI renderer on resume.
```

### tools/BriefTool/BriefTool.ts
```
resumed sessions replay pre-sentAt outputs verbatim.
```

### tools/ConfigTool/prompt.ts
```
continue
    // Voice settings are registered at build-time but gated by GrowthBook
    // at runtime.
```

### tools/ConfigTool/prompt.ts
```
continue

    const options = getOptionsForSetting(key)
    let line = `
```

### tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts
```
continue with implementation.
```

### tools/LSPTool/LSPTool.ts
```
Continue to formatter which will handle empty/null gracefully
        }
      }

      // Filter out gitignored files from location-based results
      if (
        result &&
        Array.
```

### tools/LSPTool/formatters.ts
```
continue with un-decoded path
    const errorMsg = errorMessage(error)
    logForDebugging(
      `
```

### tools/LSPTool/formatters.ts
```
continue
    }
    const filePath = formatUri(call.
```

### tools/LSPTool/formatters.ts
```
continue // Already logged above
      }
      const kind = symbolKindToString(call.
```

### tools/LSPTool/formatters.ts
```
continue
    }
    const filePath = formatUri(call.
```

### tools/LSPTool/formatters.ts
```
continue // Already logged above
      }
      const kind = symbolKindToString(call.
```

### tools/LSPTool/symbolContext.ts
```
continues past our window,
    // so the last split element may be truncated mid-line.
```

### tools/MCPTool/UI.tsx
```
continue;
      }
      if (t.
```

### tools/PowerShellTool/PowerShellTool.tsx
```
continue;
    }
    const canonical = resolveToCanonical(baseCommand);
    if (PS_SEMANTIC_NEUTRAL_COMMANDS.
```

### tools/PowerShellTool/PowerShellTool.tsx
```
continue;
    }
    hasNonNeutralCommand = true;
    const isPartSearch = PS_SEARCH_COMMANDS.
```

### tools/PowerShellTool/PowerShellTool.tsx
```
continue;
        }
        shellCommand.
```

### tools/PowerShellTool/PowerShellTool.tsx
```
ContinueAnimation: true,
          showSpinner: true
        });
      }
      yield {
        type: '
```

### tools/PowerShellTool/gitSafety.ts
```
continue
    if (n === p || n.
```

### tools/PowerShellTool/modeValidation.ts
```
continue
    // Normalize unicode dash prefixes (–, —, ―) and forward-slash (PS 5.
```

### tools/PowerShellTool/modeValidation.ts
```
continue
    const rawVal =
      colonIdx > 0
        ? lower.
```

### tools/PowerShellTool/modeValidation.ts
```
continue
        if (isCwdChangingCmdlet(cmd.
```

### tools/PowerShellTool/modeValidation.ts
```
continue
      }
      if (!isAcceptEditsAllowedCmdlet(cmd.
```

### tools/PowerShellTool/modeValidation.ts
```
continue
        }
        if (!isAcceptEditsAllowedCmdlet(cmd.
```

### tools/PowerShellTool/pathValidation.ts
```
continue

    // Check if this arg is a parameter name.
```

### tools/PowerShellTool/pathValidation.ts
```
Continue the loop so we still extract any recognizable paths
        // (useful for the ask message), but the flag ensures overall '
```

### tools/PowerShellTool/pathValidation.ts
```
continue
    }

    // Positional arguments: extract as paths (e.
```

### tools/PowerShellTool/pathValidation.ts
```
continue
    }
    positionalsSeen++
    checkArgElementType(i)
    paths.
```

### tools/PowerShellTool/pathValidation.ts
```
continue
    }

    const { paths, operationType, hasUnvalidatablePathArg, optionalWrite } =
      extractPathsFromCommand(cmd)

    // SECURITY: Cmdlet receiving piped path from expression source.
```

### tools/PowerShellTool/pathValidation.ts
```
continue — fall through to path loop so deny rules on
      // extracted paths are still checked.
```

### tools/PowerShellTool/pathValidation.ts
```
continue — fall through to path loop so deny rules on
      // extracted paths are still checked.
```

### tools/PowerShellTool/pathValidation.ts
```
continue
    }

    // SECURITY: bash-parity hard-deny for removal cmdlets on
    // system-critical paths.
```

### tools/PowerShellTool/pathValidation.ts
```
continue — fall through to path loop for deny checks.
```

### tools/PowerShellTool/pathValidation.ts
```
continue
      }

      // SECURITY: bash-parity hard-deny for removal on system-critical
      // paths — mirror the main-loop check above.
```

### tools/PowerShellTool/pathValidation.ts
```
continue
          if (!redir.
```

### tools/PowerShellTool/pathValidation.ts
```
continue
          if (isNullRedirectionTarget(redir.
```

### tools/PowerShellTool/pathValidation.ts
```
continue

          const { allowed, resolvedPath, decisionReason } = validatePath(
            redir.
```

### tools/PowerShellTool/pathValidation.ts
```
continue
      if (!redir.
```

### tools/PowerShellTool/pathValidation.ts
```
continue
      if (isNullRedirectionTarget(redir.
```

### tools/PowerShellTool/pathValidation.ts
```
continue

      const { allowed, resolvedPath, decisionReason } = validatePath(
        redir.
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
      }
      subCommands.
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue // skip empty fragments
      // Skip the full command ONLY if it starts with a cmdlet name (no
      // assignment prefix).
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
      }
      // SECURITY: Normalize invocation-operator and assignment prefixes before
      // rule matching (findings #5/#22).
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
          if (isDangerousRemovalRawPath(arg)) {
            return dangerousRemovalDeny(arg)
          }
        }
      }
      const { matchingDenyRules: fragDenyRules } = matchingRulesForInput(
        { command: normalizedFrag },
        toolPermissionContext,
        '
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
      for (const arg of cmd.
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
            if (isGitInternalPathPS(c.
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue, no push), $env:SECRET is VariableExpressionAst
  // (not a sub-command), statement marked seen → gate skips → auto-allow →
  // secret leaks.
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
    }

    // Explicitly allowed by a user rule — BUT NOT for applications/scripts.
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
    // skips all checks and the local script runs.
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
      }
      continue
    }
    if (subResult.
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue; fall through to approval (NOT deny — the user may
      // actually want to run `
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
    }

    // SECURITY: fail-closed gate.
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
    }

    // Check per-sub-command acceptEdits mode (BashTool parity).
```

### tools/PowerShellTool/powershellPermissions.ts
```
continue
      }
    }

    // Not allowlisted, no mode auto-allow, and no explicit rule — needs approval
    if (statement !== null) {
      statementsSeenInLoop.
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
    }
    const nameElementType = cmd.
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
    }
    const hasDownloader = cmds.
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
    }
    // -ComObject min abbrev is -com (New-Object params: -TypeName, -ComObject,
    // -ArgumentList, -Property, -Strict; -co is ambiguous in PS5.
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
          }
          // Colon-bound form: -Param:Value (single token, no skip needed)
          if (lower.
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
          if (SWITCH_PARAMS.
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
          if (VALUE_PARAMS.
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
          }
          // Unknown param — skip conservatively
          continue
        }
        // First non-dash arg is the positional TypeName
        typeName = a
        break
      }
    }
    if (typeName !== undefined && !isClmAllowedType(typeName)) {
      return {
        behavior: '
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
    }
    if (
      psExeHasParamAbbreviation(cmd, '
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
    }
    // ForEach-Object params starting with -m: only -MemberName.
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
    }
    // Vector 1: -Verb RunAs (space or colon syntax).
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
        const kids = cmd.
```

### tools/PowerShellTool/powershellSecurity.ts
```
continue
        for (const child of kids) {
          if (child.
```

### tools/PowerShellTool/readOnlyValidation.ts
```
continue
      }
      return true
    }
    if (argTypes[i] === '
```

### tools/PowerShellTool/readOnlyValidation.ts
```
continue
      }
      if (!isAllowlistedCommand(cmd, command)) {
        return false
      }
    }

    // SECURITY: Reject statements with nested commands.
```

### tools/PowerShellTool/readOnlyValidation.ts
```
continue
        }
        return false
      }
      // Colon-bound parameter (`
```

### tools/PowerShellTool/readOnlyValidation.ts
```
continue
      }
      const isSafe = config.
```

### tools/ScheduleCronTool/prompt.ts
```
resume automatically.
```

### tools/SendMessageTool/SendMessageTool.ts
```
resumeAgentBackground } from '
```

### tools/SendMessageTool/SendMessageTool.ts
```
continue
    }
    recipients.
```

### tools/SendMessageTool/SendMessageTool.ts
```
resume
            try {
              const result = await resumeAgentBackground({
                agentId,
                prompt: input.
```

### tools/SendMessageTool/SendMessageTool.ts
```
resumed it in the background with your message.
```

### tools/SendMessageTool/SendMessageTool.ts
```
resumed: ${errorMessage(e)}`
```

### tools/SendMessageTool/SendMessageTool.ts
```
resume from disk transcript.
```

### tools/SendMessageTool/SendMessageTool.ts
```
resumeAgentBackground({
                agentId,
                prompt: input.
```

### tools/SendMessageTool/SendMessageTool.ts
```
resumed from transcript in the background with your message.
```

### tools/SkillTool/SkillTool.ts
```
continue
    }
    // Property not in safe allowlist - check if it has a meaningful value
    const value = (command as Record<string, unknown>)[key]
    if (value === undefined || value === null) {
      continue
    }
    if (Array.
```

### tools/SkillTool/SkillTool.ts
```
continue
    }
    if (
      typeof value === '
```

### tools/SkillTool/SkillTool.ts
```
continue
    }
    return false
  }
  return true
}

function isOfficialMarketplaceSkill(command: PromptCommand): boolean {
  if (command.
```

### tools/TaskStopTool/TaskStopTool.ts
```
resume
    // without re-validation, so sessions from before this field was added lack it.
```

### tools/TodoWriteTool/TodoWriteTool.ts
```
continue to use the todo list to track your progress.
```

### tools/WebFetchTool/WebFetchTool.ts
```
continue with normal permission checks
    }

    // Check for a rule specific to the tool input (matching hostname)
    const ruleContent = webFetchToolInputToPermissionRuleContent(input)

    const denyRule = getRuleByContentsForTool(
      permissionContext,
      WebFetchTool,
      '
```

### tools/WebFetchTool/utils.ts
```
Continue with the fetch
          break
        case '
```

### tools/WebSearchTool/WebSearchTool.ts
```
continue
    }

    if (block.
```

### tools/WebSearchTool/WebSearchTool.ts
```
continue
      }
      // Success case - add results to our collection
      const hits = block.
```

### tools/WebSearchTool/WebSearchTool.ts
```
continue
      }

      // Track tool use ID when server_tool_use starts
      if (
        event.
```

### tools/WebSearchTool/WebSearchTool.ts
```
continue
        }
      }

      // Accumulate JSON for current tool use
      if (
        currentToolUseId &&
        event.
```

### tools/shared/gitOperationTracking.ts
```
continue
    return t
  }
  return undefined
}

/**
 * Scan bash command + output for git operations worth surfacing in the
 * collapsed tool-use summary ("
```

### tools/shared/spawnMultiAgent.ts
```
pick up the ITermBackend (it2 is now available)
    // - '
```

### types/command.ts
```
resume?: (
    sessionId: UUID,
    log: LogOption,
    entrypoint: ResumeEntrypoint,
  ) => Promise<void>
}

export type ResumeEntrypoint =
  | '
```

### types/hooks.ts
```
continue: z
      .
```

### types/hooks.ts
```
continue after hook (default: true)'
```

### types/logs.ts
```
resume)
}

export type LogOption = {
  date: string
  messages: SerializedMessage[]
  fullPath?: string
  value: number
  created: Date
  modified: Date
  firstPrompt: string
  messageCount: number
  fileSize?: number // File size in bytes (for display)
  isSidechain: boolean
  isLite?: boolean // True for lite logs (messages not loaded)
  sessionId?: string // Session ID for lite logs
  teamName?
```

### types/logs.ts
```
resume)
  fileHistorySnapshots?: FileHistorySnapshot[] // Optional file history snapshots
  attributionSnapshots?: AttributionSnapshotMessage[] // Optional attribution snapshots
  contextCollapseCommits?: ContextCollapseCommitEntry[] // Ordered — commit B may reference commit A'
```

### types/logs.ts
```
resume reconstruction
}

export type SummaryMessage = {
  type: '
```

### types/logs.ts
```
resume)
 * - VS Code'
```

### types/logs.ts
```
resume, restored only if the worktreePath
 * still exists on disk (the /exit dialog may have removed it).
```

### types/logs.ts
```
resume
 * for prompt cache stability.
```

### types/logs.ts
```
resume reads these); when absent, it'
```

### types/logs.ts
```
resume reads these).
```

### types/logs.ts
```
resumed Message[].
```

### utils/Cursor.ts
```
continue
          }
          const nextWidth = currentWidth + stringWidth(segment)
          if (nextWidth > column) {
            atCursor = segment
            cursorFound = true
          } else {
            currentWidth = nextWidth
            beforeCursor += segment
          }
        }

        // Only invert the cursor if we have a cursor character to show
        // When ghost text is 
```

### utils/Cursor.ts
```
continue

      // If we'
```

### utils/Cursor.ts
```
continue

      // If we'
```

### utils/Cursor.ts
```
continue
      const segment = this.
```

### utils/agentContext.ts
```
resumed this agent.
```

### utils/agentContext.ts
```
resume
   *  via SendMessage.
```

### utils/agentContext.ts
```
resume; flipped true by
   *  consumeInvokingRequestId() on the first terminal API event.
```

### utils/agentContext.ts
```
resumed this
   *  teammate.
```

### utils/agentContext.ts
```
resume, then
 * undefined until the next boundary.
```

### utils/ansiToPng.ts
```
continue // zero-width (combining marks, etc.
```

### utils/ansiToPng.ts
```
continue
      if (bold) a = Math.
```

### utils/ansiToPng.ts
```
continue
      // Top-left, top-right, bottom-left, bottom-right.
```

### utils/ansiToSvg.ts
```
continue
      }

      // Regular character - find extent of same-styled text
      const textStart = i
      while (i < line.
```

### utils/ansiToSvg.ts
```
continue

      const colorStr = `
```

### utils/api.ts
```
continue
      if (prompt === SYSTEM_PROMPT_DYNAMIC_BOUNDARY) continue // Skip boundary
      if (prompt.
```

### utils/api.ts
```
continue

        if (block.
```

### utils/api.ts
```
continue

    if (block.
```

### utils/api.ts
```
resumed from transcripts written before PR #20357, where
      // normalizeToolInput used to synthesize these.
```

### utils/argumentSubstitution.ts
```
continue

    // Match $name but not $name[.
```

### utils/asciicast.ts
```
resume)
const recordingState: { filePath: string | null; timestamp: number } = {
  filePath: null,
  timestamp: 0,
}

/**
 * Get the asciicast recording file path.
```

### utils/asciicast.ts
```
continue produces multiple recordings.
```

### utils/asciicast.ts
```
resume/--continue changes the session ID via switchSession().
```

### utils/asciicast.ts
```
resumed session ID.
```

### utils/asciicast.ts
```
resume
      const currentPath = recordingState.
```

### utils/attachments.ts
```
resumed sessions; render
         * path falls back to recomputing if missing.
```

### utils/attachments.ts
```
continue
    if (msg.
```

### utils/attachments.ts
```
continue
    for (const t of msg.
```

### utils/attachments.ts
```
continue
    }
    if (!toolUseContext.
```

### utils/attachments.ts
```
continue with file logic
        }

        return await generateFileAttachment(
          absoluteFilename,
          toolUseContext,
          '
```

### utils/attachments.ts
```
resumed sessions where the stored header is missing.
```

### utils/attachments.ts
```
continue
    if (isHumanTurn(m) && m !== lastUserMessage) break
    if (m.
```

### utils/attachments.ts
```
continue
    if (errored) {
      failed.
```

### utils/attachments.ts
```
resume when a skill_listing attachment already exists in the
 * transcript.
```

### utils/attachments.ts
```
resume re-injects the
 * full ~600-token listing even though it'
```

### utils/attachments.ts
```
resume; particularly loud for
 * daemons that respawn frequently.
```

### utils/attachments.ts
```
Resume path: prior process already injected a listing; it'
```

### utils/attachments.ts
```
resume deltas
  // (skills loaded later via /reload-plugins etc) get announced.
```

### utils/attachments.ts
```
continue
      }

      // Check for TodoWrite usage BEFORE incrementing counter
      // (we don'
```

### utils/attachments.ts
```
continue
      }

      // Check for TaskCreate or TaskUpdate usage BEFORE incrementing counter
      if (
        lastTaskManagementIndex === -1 &&
        '
```

### utils/attribution.ts
```
continue
    }

    const content = message.
```

### utils/attribution.ts
```
continue
    }

    let hasUserText = false

    if (typeof content === '
```

### utils/attribution.ts
```
continue
      }
      hasUserText = content.
```

### utils/attribution.ts
```
continue
    const content = entry.
```

### utils/attribution.ts
```
continue
    for (const block of content) {
      if (
        block.
```

### utils/attribution.ts
```
continue
      if (isMemoryFileAccess(block.
```

### utils/autoUpdater.ts
```
continue

      const { filtered, hadAlias } = filterClaudeAliases(lines)

      if (hadAlias) {
        await writeFileLines(configFile, filtered)
        logForDebugging(`
```

### utils/bash/ast.ts
```
continue
      if (SEPARATOR_TYPES.
```

### utils/bash/ast.ts
```
continue
      }
      const err = collectCommands(child, commands, scope)
      if (err) return err
    }
    return null
  }

  if (node.
```

### utils/bash/ast.ts
```
continue
      if (child.
```

### utils/bash/ast.ts
```
continue
      return collectCommands(child, commands, varScope)
    }
    return null
  }

  if (node.
```

### utils/bash/ast.ts
```
continue
      switch (child.
```

### utils/bash/ast.ts
```
continue
      if (child.
```

### utils/bash/ast.ts
```
continue // structural tokens
      } else if (child.
```

### utils/bash/ast.ts
```
continue
      if (c.
```

### utils/bash/ast.ts
```
continue
      const err = collectCommands(c, commands, bodyScope)
      if (err) return err
    }
    return null
  }

  if (node.
```

### utils/bash/ast.ts
```
continue
      if (
        child.
```

### utils/bash/ast.ts
```
continue
      }
      if (child.
```

### utils/bash/ast.ts
```
continue
      }
      if (child.
```

### utils/bash/ast.ts
```
continue
          if (c.
```

### utils/bash/ast.ts
```
continue
          const err = collectCommands(c, commands, bodyScope)
          if (err) return err
        }
        continue
      }
      if (child.
```

### utils/bash/ast.ts
```
continue
          if (
            c.
```

### utils/bash/ast.ts
```
continue
          }
          const err = collectCommands(c, commands, branchScope)
          if (err) return err
        }
        continue
      }
      // Condition (seenThen=false) or then-body (seenThen=true).
```

### utils/bash/ast.ts
```
continue
      if (child.
```

### utils/bash/ast.ts
```
continue
      const err = collectCommands(child, commands, innerScope)
      if (err) return err
    }
    return null
  }

  if (node.
```

### utils/bash/ast.ts
```
continue
      if (child.
```

### utils/bash/ast.ts
```
continue
      if (child.
```

### utils/bash/ast.ts
```
continue
      // Recurse into test expression structure: unary_expression,
      // binary_expression, parenthesized_expression, negated_expression.
```

### utils/bash/ast.ts
```
continue
      switch (child.
```

### utils/bash/ast.ts
```
continue
        const err = walkTestExpr(c, argv, innerCommands, varScope)
        if (err) return err
      }
      return null
    }
    case '
```

### utils/bash/ast.ts
```
continue
    if (child.
```

### utils/bash/ast.ts
```
continue
    if (child.
```

### utils/bash/ast.ts
```
continue
    if (child.
```

### utils/bash/ast.ts
```
continue
      if (child.
```

### utils/bash/ast.ts
```
continue
    if (child.
```

### utils/bash/ast.ts
```
continue
    // Content node: reuse walkArgument.
```

### utils/bash/ast.ts
```
continue

    switch (child.
```

### utils/bash/ast.ts
```
continue
    if (child.
```

### utils/bash/ast.ts
```
continue
    }
    const err = collectCommands(child, innerCommands, innerScope)
    if (err) return err
  }
  return null
}

/**
 * Convert an argument node to its literal string value.
```

### utils/bash/ast.ts
```
continue
        const part = walkArgument(child, innerCommands, varScope)
        if (typeof part !== '
```

### utils/bash/ast.ts
```
continue
    // Index gap between this child and the previous one = dropped newline(s).
```

### utils/bash/ast.ts
```
continue
    if (child.
```

### utils/bash/ast.ts
```
continue
    }
    switch (child.
```

### utils/bash/ast.ts
```
continue
    if (child.
```

### utils/bash/ast.ts
```
continue
    if (child.
```

### utils/bash/ast.ts
```
continue
    if (child.
```

### utils/bash/ast.ts
```
continue
    if (child.
```

### utils/bash/ast.ts
```
continue
    } else if (child.
```

### utils/bash/ast.ts
```
continue

    // SECURITY: Empty command name.
```

### utils/bash/ast.ts
```
continue
        if (a[i - 1]?.
```

### utils/bash/ast.ts
```
continue
        }
        if (arg[0] === '
```

### utils/bash/ast.ts
```
continue
        }
        if (arg.
```

### utils/bash/bashParser.ts
```
continue
        }
        advance(L)
        advance(L)
        continue
      }
      if (!isWordChar(ch) && ch !== '
```

### utils/bash/bashParser.ts
```
continue
    }
    restoreLex(P.
```

### utils/bash/bashParser.ts
```
continue
    if (t.
```

### utils/bash/bashParser.ts
```
continue
    }
    restoreLex(P.
```

### utils/bash/bashParser.ts
```
continue
      }
      children.
```

### utils/bash/bashParser.ts
```
continue
    }
    if (t.
```

### utils/bash/bashParser.ts
```
continue
    }
    if (terminator && t.
```

### utils/bash/bashParser.ts
```
continue
      }
    } else if (sep.
```

### utils/bash/bashParser.ts
```
continue
    } else {
      restoreLex(P.
```

### utils/bash/bashParser.ts
```
continue
      }
      parts.
```

### utils/bash/bashParser.ts
```
continue
    }
    const r = tryParseRedirect(P)
    if (r) {
      preRedirects.
```

### utils/bash/bashParser.ts
```
continue
    }
    break
  }

  skipBlanks(P.
```

### utils/bash/bashParser.ts
```
continue
    }
    // Once a file_redirect has been seen, command args are done — grammar'
```

### utils/bash/bashParser.ts
```
continue
      }
      break
    }
    // Lone `
```

### utils/bash/bashParser.ts
```
continue
    }
    // Word immediately followed by `
```

### utils/bash/bashParser.ts
```
continue
    }
    args.
```

### utils/bash/bashParser.ts
```
continue
        }
        restoreLex(P.
```

### utils/bash/bashParser.ts
```
continue
          }
          break
        }
        if (pipeCmds.
```

### utils/bash/bashParser.ts
```
continue
      }
      // && / || after heredoc_start: `
```

### utils/bash/bashParser.ts
```
continue
      }
      // Terminator / unhandled metachar — consume rest of line as ERROR so
      // ast.
```

### utils/bash/bashParser.ts
```
continue
      }
      // Unrecognized — consume rest of line as ERROR
      const eStart = P.
```

### utils/bash/bashParser.ts
```
continue
      }
      advance(P.
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    advance(P.
```

### utils/bash/bashParser.ts
```
continue
      }
      break
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === "
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
      }
      if (c1 === '
```

### utils/bash/bashParser.ts
```
continue
      }
      if (c1 === '
```

### utils/bash/bashParser.ts
```
continue
      }
      const exp = parseDollarLike(P)
      if (exp) parts.
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    // Brace expression {1.
```

### utils/bash/bashParser.ts
```
continue
      }
      // SECURITY: if `
```

### utils/bash/bashParser.ts
```
continue
      }
      // Otherwise treat { and } as word fragments
      const cat = tryParseBraceLikeCat(P)
      if (cat) {
        for (const p of cat) parts.
```

### utils/bash/bashParser.ts
```
continue
      }
    }
    // Standalone `
```

### utils/bash/bashParser.ts
```
continue
    }
    // `
```

### utils/bash/bashParser.ts
```
continue
    }
    // Bare word fragment
    const frag = parseBareWord(P)
    if (!frag) break
    // `
```

### utils/bash/bashParser.ts
```
continue
      }
    }
    parts.
```

### utils/bash/bashParser.ts
```
continue
    }
    if (
      c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    const midStart = P.
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
      }
      // Bare $ not at end-of-string: tree-sitter emits it as an anonymous
      // '
```

### utils/bash/bashParser.ts
```
continue
      }
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    advance(P.
```

### utils/bash/bashParser.ts
```
continue
      }
      if (c === '
```

### utils/bash/bashParser.ts
```
continue
      }
      // Skip past nested ${.
```

### utils/bash/bashParser.ts
```
continue
        }
        if (c1 === '
```

### utils/bash/bashParser.ts
```
continue
        }
      }
      if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    const c1 = peek(P.
```

### utils/bash/bashParser.ts
```
continue
      }
      if (c1 === "
```

### utils/bash/bashParser.ts
```
continue
      }
      if (isIdentStart(c1) || isDigit(c1) || SPECIAL_VARS.
```

### utils/bash/bashParser.ts
```
continue
      }
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === "
```

### utils/bash/bashParser.ts
```
continue
    }
    if ((c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    // Brace tracking so nested {a,b} brace-expansion chars don'
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === "
```

### utils/bash/bashParser.ts
```
continue
    }
    // Nested ${.
```

### utils/bash/bashParser.ts
```
continue
      }
      if (c1 === '
```

### utils/bash/bashParser.ts
```
continue
      }
    }
    advance(P.
```

### utils/bash/bashParser.ts
```
continue
    restoreLex(P.
```

### utils/bash/bashParser.ts
```
continue
      const text = sliceBytes(P, k.
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    // Paren counting: any ( inside pattern opens a scope; don'
```

### utils/bash/bashParser.ts
```
continue
    }
    if (parenDepth > 0) {
      if (c === '
```

### utils/bash/bashParser.ts
```
continue
      }
      if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === "
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    // Quoted string or concatenation: `
```

### utils/bash/bashParser.ts
```
continue
      }
      break
    }
    // Flag like -a or bare variable name
    const save = saveLex(P.
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
      }
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
      }
    }
    // $ "
```

### utils/bash/bashParser.ts
```
continue
      }
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === "
```

### utils/bash/bashParser.ts
```
continue
    }
    if (c === '
```

### utils/bash/bashParser.ts
```
continue
    }
    break
  }
  return out
}

function parseArithTernary(
  P: ParseState,
  stop: string,
  mode: ArithMode,
): TsNode | null {
  const cond = parseArithBinary(P, stop, 0, mode)
  if (!cond) return null
  skipBlanks(P.
```

### utils/bash/bashPipeCommand.ts
```
continue
      }

      // Handle 2>/dev/null style redirections
      if (op.
```

### utils/bash/bashPipeCommand.ts
```
continue
      }

      // Handle 2> &1 style (space between > and &1)
      if (
        op.
```

### utils/bash/bashPipeCommand.ts
```
continue
        }
      }
    }

    // Handle regular entries
    if (typeof entry === '
```

### utils/bash/commands.ts
```
continue
        }
      } else if ('
```

### utils/bash/commands.ts
```
continue
        }
      }
      parts.
```

### utils/bash/commands.ts
```
continue
    }

    // Strip redirections so they don'
```

### utils/bash/commands.ts
```
continue
      }

      // Determine if this redirection should be stripped
      let shouldStrip = false
      let stripThirdToken = false

      // SPECIAL CASE: The adjacent-string collapse merges `
```

### utils/bash/commands.ts
```
continue
    }

    if (typeof part === '
```

### utils/bash/commands.ts
```
continue
    }
    if ('
```

### utils/bash/commands.ts
```
continue
      } else if (COMMAND_LIST_SEPARATORS.
```

### utils/bash/commands.ts
```
continue
      } else if (part.
```

### utils/bash/commands.ts
```
continue
        }
      } else if (part.
```

### utils/bash/commands.ts
```
continue
      } else if (part.
```

### utils/bash/commands.ts
```
continue
      }
      // Other operators are unsafe
      return false
    }
  }
  // No unsafe operators found in entire command
  return true
}

/**
 * @deprecated Legacy regex/shell-quote path.
```

### utils/bash/commands.ts
```
continue

    const [prev, next] = [parsed[i - 1], parsed[i + 1]]

    // Skip redirected subshell parens
    if (
      (isOperator(part, '
```

### utils/bash/commands.ts
```
continue
    }

    // Track command substitution depth
    if (
      isOperator(part, '
```

### utils/bash/commands.ts
```
continue
      }
    }

    kept.
```

### utils/bash/commands.ts
```
continue
    }

    // Handle operators
    if (typeof part !== '
```

### utils/bash/commands.ts
```
continue
    const op = part.
```

### utils/bash/commands.ts
```
continue
    }

    // Handle file descriptor redirects (2>&1)
    if (
      op === '
```

### utils/bash/commands.ts
```
continue
    }

    // Handle heredocs
    if (op === '
```

### utils/bash/commands.ts
```
continue
      }
    }

    // Handle here-strings (always preserve the operator)
    if (op === '
```

### utils/bash/commands.ts
```
continue
    }

    // Handle parentheses
    if (op === '
```

### utils/bash/commands.ts
```
continue
    }

    if (op === '
```

### utils/bash/commands.ts
```
continue
      }

      if (cmdSubDepth > 0) cmdSubDepth--
      result += '
```

### utils/bash/commands.ts
```
continue
    }

    // Handle process substitution
    if (op === '
```

### utils/bash/commands.ts
```
continue
    }

    // All other operators
    if (['
```

### utils/bash/heredoc.ts
```
continue
      }

      if (scanInDoubleQuote) {
        if (scanDqEscapeNext) {
          scanDqEscapeNext = false
          continue
        }
        if (ch === '
```

### utils/bash/heredoc.ts
```
continue
        }
        if (ch === '
```

### utils/bash/heredoc.ts
```
continue
      }

      // Unquoted context.
```

### utils/bash/heredoc.ts
```
continue
      }
      const escaped = scanPendingBackslashes % 2 === 1
      scanPendingBackslashes = 0
      if (escaped) continue

      if (ch === "
```

### utils/bash/heredoc.ts
```
continue
    }

    // Security: Skip if this << is inside a comment (after unquoted #).
```

### utils/bash/heredoc.ts
```
continue
    }

    // Security: Skip if this << is preceded by an odd number of backslashes.
```

### utils/bash/heredoc.ts
```
continue
    }

    // Security: Bail if this `
```

### utils/bash/heredoc.ts
```
continue
    }

    const fullMatch = match[0]
    const isDash = match[1] === '
```

### utils/bash/heredoc.ts
```
continue
    }

    // Security: Determine if the delimiter is quoted ('
```

### utils/bash/heredoc.ts
```
continue
      }
    }

    // In bash, heredoc content starts on the NEXT LINE after the operator.
```

### utils/bash/heredoc.ts
```
continue
        }
        if (inDoubleQuote) {
          if (ch === '
```

### utils/bash/heredoc.ts
```
continue
          }
          if (ch === '
```

### utils/bash/heredoc.ts
```
continue
        }
        // Unquoted context
        if (ch === '
```

### utils/bash/heredoc.ts
```
continue // escaped char
        if (ch === "
```

### utils/bash/heredoc.ts
```
continue
    }

    // Security: Check for backslash-newline continuation at the end of the
    // same-line content (text between the operator and the newline).
```

### utils/bash/heredoc.ts
```
continue
    }

    const contentStartIndex = operatorEndIndex + firstNewlineOffset
    const afterNewline = command.
```

### utils/bash/heredoc.ts
```
continue
    }

    // If no closing delimiter found, this is malformed - skip it
    if (closingLineIndex === -1) {
      continue
    }

    // Calculate end position: contentStartIndex + 1 (newline) + length of lines up to and including closing delimiter
    const linesUpToClosing = contentLines.
```

### utils/bash/heredoc.ts
```
continue
    }

    // Build fullText: operator + newline + content (normalized form for restoration)
    // This creates a clean heredoc that can be restored correctly
    const operatorText = command.
```

### utils/bash/heredoc.ts
```
continue
      // Check if candidate'
```

### utils/bash/parser.ts
```
continue

    // Command name
    if (
      child.
```

### utils/bash/parser.ts
```
continue
    }

    // Arguments
    if (ARGUMENT_TYPES.
```

### utils/bash/prefix.ts
```
continue
    const result = await getCommandPrefixStatic(trimmed)
    if (result?.
```

### utils/bash/shellQuote.ts
```
continue
    }
    if (c === '
```

### utils/bash/shellQuote.ts
```
continue

    // Check for unbalanced curly braces
    const openBraces = (entry.
```

### utils/bash/shellQuote.ts
```
continue
    }

    if (char === '
```

### utils/bash/shellQuote.ts
```
continue
    }

    if (char === "
```

### utils/bash/shellQuote.ts
```
continues reading
      //   until it finds another '
```

### utils/bash/shellQuote.ts
```
continue
    }
  }

  return false
}

export function quote(args: ReadonlyArray<unknown>): string {
  // First try the strict validation
  const result = tryQuoteShellArgs([.
```

### utils/bash/specs/pyright.ts
```
Continue to run and watch for changes'
```

### utils/bash/treeSitterAnalysis.ts
```
pick up inner raw_string/ansi_c_string.
```

### utils/bash/treeSitterAnalysis.ts
```
continue
    if (doubleQuoteDelimSet.
```

### utils/bash/treeSitterAnalysis.ts
```
continue
    withDoubleQuotes += command[i]
  }

  // fullyUnquoted: remove all quoted content
  const fullyUnquoted = removeSpans(command, allQuoteSpans)

  // unquotedKeepQuoteChars: remove content but keep delimiter chars
  const spansWithQuoteChars: Array<[number, number, string, string]> = []
  for (const [start, end] of singleQuoteSpans) {
    spansWithQuoteChars.
```

### utils/bash/treeSitterAnalysis.ts
```
continue

      if (child.
```

### utils/bash/treeSitterAnalysis.ts
```
continue
          if (listChild.
```

### utils/bash/treeSitterAnalysis.ts
```
continue
          foundInner = true
          walkTopLevel({ .
```

### utils/claudeDesktop.ts
```
continue
    }
  }

  // Alternative approach - try to construct path based on typical Windows user location
  try {
    // List the /mnt/c/Users directory to find potential user directories
    const usersDir = '
```

### utils/claudeDesktop.ts
```
continue // Skip system directories
        }

        const potentialConfigPath = join(
          usersDir,
          user.
```

### utils/claudeDesktop.ts
```
continue
        }
      }
    } catch {
      // usersDir doesn'
```

### utils/claudeDesktop.ts
```
continue
      }

      const result = McpStdioServerConfigSchema().
```

### utils/claudeInChrome/chromeNativeHost.ts
```
continue
          }
          const pid = parseInt(file.
```

### utils/claudeInChrome/chromeNativeHost.ts
```
continue
          }
          try {
            process.
```

### utils/claudeInChrome/common.ts
```
continue
      }
    }

    if (dataPath && dataPath.
```

### utils/claudeInChrome/common.ts
```
continue checking
        }
        break
      }
      case '
```

### utils/claudeInChrome/common.ts
```
continue checking
          }
        }
        break
      }
    }
  }

  return null
}

export function isClaudeInChromeMCPServer(name: string): boolean {
  return normalizeNameForMCP(name) === CLAUDE_IN_CHROME_MCP_SERVER_NAME
}

const MAX_TRACKED_TABS = 200
const trackedTabIds = new Set<number>()

export function trackClaudeInChromeTabId(tabId: number): void {
  if (trackedTabIds.
```

### utils/claudeInChrome/mcpServer.ts
```
continue to experience issues, please report a bug: ${BUG_REPORT_URL}`
```

### utils/claudeInChrome/setup.ts
```
continue
    }

    try {
      await mkdir(manifestDir, { recursive: true })
      await writeFile(manifestPath, manifestContent)
      logForDebugging(
        `
```

### utils/claudeInChrome/setupPortable.ts
```
continue
      }
    }

    if (dataPath && dataPath.
```

### utils/claudeInChrome/setupPortable.ts
```
continue to next browser
      if (isFsInaccessible(e)) continue
      throw e
    }

    const profileDirs = browserProfileEntries
      .
```

### utils/claudeInChrome/setupPortable.ts
```
continue checking
        }
      }
    }
  }

  log?.
```

### utils/claudemd.ts
```
continue
      }
    }
    result += token.
```

### utils/claudemd.ts
```
continue

      // Strip fragment identifiers (#heading, #section-name, etc.
```

### utils/claudemd.ts
```
continue

      // Unescape the spaces in the path
      path = path.
```

### utils/claudemd.ts
```
continue
      }

      // For html tokens that contain comments, strip the comment spans and
      // check the residual for @paths (e.
```

### utils/claudemd.ts
```
continue
      }

      // Process text nodes
      if (element.
```

### utils/claudemd.ts
```
continue
    }

    // Find the static prefix before any glob characters
    const globStart = normalized.
```

### utils/claudemd.ts
```
continue
    }

    // Recursively process included files with this file as parent
    const includedFiles = await processMemoryFile(
      resolvedIncludePath,
      type,
      processedPaths,
      includeExternal,
      depth + 1,
      filePath, // Pass current file as parent
    )
    result.
```

### utils/claudemd.ts
```
continue
          const loadReason = file.
```

### utils/claudemd.ts
```
continue
    if (skipProjectLevel && (file.
```

### utils/claudemd.ts
```
continue
    if (file.
```

### utils/cleanup.ts
```
continue processing other files
        logError(error as Error)
      }
    }
  } catch (error: unknown) {
    // Ignore if directory doesn'
```

### utils/cleanup.ts
```
continue
    const projectDir = join(projectsDir, projectDirent.
```

### utils/cleanup.ts
```
continue
    }

    for (const entry of entries) {
      if (entry.
```

### utils/cleanup.ts
```
continue
        }
        try {
          if (
            await unlinkIfOld(join(projectDir, entry.
```

### utils/cleanup.ts
```
continue
        }
        for (const toolEntry of toolDirs) {
          if (toolEntry.
```

### utils/cleanup.ts
```
continue
            }
            for (const tf of toolFiles) {
              if (!tf.
```

### utils/cleanup.ts
```
continue
              try {
                if (
                  await unlinkIfOld(
                    join(toolDirPath, tf.
```

### utils/cleanup.ts
```
continue
    try {
      if (await unlinkIfOld(join(dirPath, dirent.
```

### utils/cleanup.ts
```
continue
    }
    try {
      if (await unlinkIfOld(join(debugDir, dirent.
```

### utils/codeIndexing.ts
```
continue$/i, tool: '
```

### utils/collapseReadSearch.ts
```
continue
    const command = group.
```

### utils/collapseReadSearch.ts
```
continue
    const { commit, push, branch, pr } = detectGitOperation(command, combined)
    if (commit) group.
```

### utils/commitAttribution.ts
```
continue

    if (result.
```

### utils/commitAttribution.ts
```
continue
    }

    files[result.
```

### utils/computerUse/cleanup.ts
```
continues in the
// background regardless; we just stop blocking on it.
```

### utils/computerUse/executor.ts
```
Continue with action execution even if switching fails"
```

### utils/computerUse/executor.ts
```
continue pushing to `
```

### utils/computerUse/gates.ts
```
continues
// regardless of subscription tier — not all ants are max/pro, and per
// CLAUDE.
```

### utils/computerUse/mcpServer.ts
```
continues in the background — swallow late rejections.
```

### utils/concurrentSessions.ts
```
resume / /resume mutates getSessionId() via switchSession.
```

### utils/concurrentSessions.ts
```
continue
    const pid = parseInt(file.
```

### utils/concurrentSessions.ts
```
continue
    }
    if (isProcessRunning(pid)) {
      count++
    } else if (getPlatform() !== '
```

### utils/config.ts
```
continue with write
    }

    // Write config file with secure permissions - mode only applies to new files
    writeFileSyncAndFlush_DEPRECATED(
      file,
      jsonStringify(filteredConfig, null, 2),
      {
        encoding: '
```

### utils/contextSuggestions.ts
```
continue
    }

    const suggestion = getLargeToolSuggestion(
      tool.
```

### utils/conversationRecovery.ts
```
Resume,
  type FileHistorySnapshot,
} from '
```

### utils/conversationRecovery.ts
```
ResumeConsistency,
  getLastSessionLog,
  getSessionIdFromLog,
  isLiteLog,
  loadFullLog,
  loadMessageLogs,
  loadTranscriptFile,
  removeExtraFields,
} from '
```

### utils/conversationRecovery.ts
```
Resume instead
 */
export function deserializeMessages(serializedMessages: Message[]): Message[] {
  return deserializeMessagesWithInterruptDetection(serializedMessages).
```

### utils/conversationRecovery.ts
```
resume path to auto-continue
 * interrupted turns after a gateway-triggered restart.
```

### utils/conversationRecovery.ts
```
Continue from where you left off.
```

### utils/conversationRecovery.ts
```
resume action is taken.
```

### utils/conversationRecovery.ts
```
resume fire after retry exhaustion instead of reading the error as
  // a completed turn.
```

### utils/conversationRecovery.ts
```
resume misclassifies every
      // brief-mode session as interrupted mid-turn and injects a phantom
      // "
```

### utils/conversationRecovery.ts
```
Continue from where you left off.
```

### utils/conversationRecovery.ts
```
continue
    for (const b of msg.
```

### utils/conversationRecovery.ts
```
resume after compaction.
```

### utils/conversationRecovery.ts
```
resume, the skills would be lost
 * because STATE.
```

### utils/conversationRecovery.ts
```
Resume instead
 */
export function restoreSkillStateFromMessages(messages: Message[]): void {
  for (const message of messages) {
    if (message.
```

### utils/conversationRecovery.ts
```
continue
    }
    if (message.
```

### utils/conversationRecovery.ts
```
Resume only happens for the main session, so agentId is null
          addInvokedSkill(skill.
```

### utils/conversationRecovery.ts
```
resume re-announces the same
    // ~600 tokens.
```

### utils/conversationRecovery.ts
```
continue
    const ts = new Date(m.
```

### utils/conversationRecovery.ts
```
resume from various sources.
```

### utils/conversationRecovery.ts
```
resume receives a .
```

### utils/conversationRecovery.ts
```
resume where the
 *   transcript lives outside the current project dir.
```

### utils/conversationRecovery.ts
```
Resume(
  source: string | LogOption | undefined,
  sourceJsonlFile: string | undefined,
): Promise<{
  messages: Message[]
  turnInterruptionState: TurnInterruptionState
  fileHistorySnapshots?: FileHistorySnapshot[]
  attributionSnapshots?: AttributionSnapshotMessage[]
  contentReplacements?: ContentReplacementRecord[]
  contextCollapseCommits?: ContextCollapseCommitEntry[]
  contextCollapseSnap
```

### utils/conversationRecovery.ts
```
resume)
  fullPath?: string
} | null> {
  try {
    let log: LogOption | null = null
    let messages: Message[] | null = null
    let sessionId: UUID | undefined

    if (source === undefined) {
      // --continue: most recent session, skipping live --bg/daemon sessions
      // that are actively writing their own transcript.
```

### utils/conversationRecovery.ts
```
resume
      if (sessionId) {
        await copyPlanForResume(log, asSessionId(sessionId))
      }

      // Copy file history for resume
      void copyFileHistoryForResume(log)

      messages = log.
```

### utils/conversationRecovery.ts
```
ResumeConsistency(messages)
    }

    // Restore skill state from invoked_skills attachments before deserialization.
```

### utils/conversationRecovery.ts
```
resume
    const hookMessages = await processSessionStartHooks('
```

### utils/conversationRecovery.ts
```
resume
      agentName: log?.
```

### utils/conversationRecovery.ts
```
resume
      fullPath: log?.
```

### utils/cron.ts
```
continue
    }

    // N-M or N-M/S
    const rangeMatch = part.
```

### utils/cron.ts
```
continue
    }

    // plain N
    const singleMatch = part.
```

### utils/cron.ts
```
continue
    }

    return null
  }

  if (out.
```

### utils/cron.ts
```
continue
    }

    const dom = t.
```

### utils/cron.ts
```
continue
    }

    if (!hourSet.
```

### utils/cron.ts
```
continue
    }

    if (!minuteSet.
```

### utils/cron.ts
```
continue
    }

    return t
  }

  return null
}

// --- cronToHuman ------------------------------------------------------------
// Intentionally narrow: covers common patterns; falls through to the raw cron
// string for anything else.
```

### utils/cronTasks.ts
```
continue
    }
    if (!parseCronExpression(t.
```

### utils/cronTasks.ts
```
continue
    }
    out.
```

### utils/cronTasksLock.ts
```
resume the session ID is restored
  // but the process has a new PID — update the lock file so other sessions
  // see a live PID and don'
```

### utils/crossProjectResume.ts
```
ResumeResult =
  | {
      isCrossProject: false
    }
  | {
      isCrossProject: true
      isSameRepoWorktree: true
      projectPath: string
    }
  | {
      isCrossProject: true
      isSameRepoWorktree: false
      command: string
      projectPath: string
    }

/**
 * Check if a log is from a different project directory and determine
 * whether it'
```

### utils/crossProjectResume.ts
```
resume directly without requiring cd.
```

### utils/crossProjectResume.ts
```
Resume(
  log: LogOption,
  showAllProjects: boolean,
  worktreePaths: string[],
): CrossProjectResumeResult {
  const currentCwd = getOriginalCwd()

  if (!showAllProjects || !log.
```

### utils/crossProjectResume.ts
```
resume ${sessionId}`
```

### utils/crossProjectResume.ts
```
resume ${sessionId}`
```

### utils/desktopDeepLink.ts
```
resume a CLI session.
```

### utils/desktopDeepLink.ts
```
resume?session={sessionId}&cwd={cwd}
 * In dev mode: claude-dev://resume?session={sessionId}&cwd={cwd}
 */
function buildDesktopDeepLink(sessionId: string): string {
  const protocol = isDevMode() ? '
```

### utils/desktopDeepLink.ts
```
resume the current session in Claude Desktop.
```

### utils/displayTags.ts
```
resume,
 * bridge session titles).
```

### utils/earlyInput.ts
```
continue without early capture
    isCapturing = false
  }
}

/**
 * Process a chunk of input data
 */
function processChunk(str: string): void {
  let i = 0
  while (i < str.
```

### utils/earlyInput.ts
```
continue
    }

    // Skip escape sequences (arrow keys, function keys, focus events, etc.
```

### utils/earlyInput.ts
```
continue
    }

    // Skip other control characters (except tab and newline)
    if (code < 32 && code !== 9 && code !== 10 && code !== 13) {
      i++
      continue
    }

    // Convert carriage return to newline
    if (code === 13) {
      earlyInputBuffer += '
```

### utils/earlyInput.ts
```
continue
    }

    // Add printable characters and allowed control chars to buffer
    earlyInputBuffer += char
    i++
  }
}

/**
 * Stop capturing early input.
```

### utils/exampleCommands.ts
```
continue
      const lastSep = Math.
```

### utils/exampleCommands.ts
```
continue
      const dir = lastSep >= 0 ? p.
```

### utils/exampleCommands.ts
```
continue
      picked.
```

### utils/file.ts
```
continue
        }

        const potentialDesktopPath = join(usersDir, user.
```

### utils/fileHistory.ts
```
continue
          const inherited = lastSnapshot.
```

### utils/fileHistory.ts
```
resume support
      void recordFileHistorySnapshot(
        messageId,
        newSnapshot,
        false, // isSnapshotUpdate
      ).
```

### utils/fileHistory.ts
```
continue
    filesChanged.
```

### utils/fileHistory.ts
```
continue
      }
      if (backupFileName === null) {
        // Backup says file did not exist; probe via stat (operate-then-catch).
```

### utils/fileHistory.ts
```
continue
      }
      if (await checkOriginFileChanged(filePath, backupFileName)) return true
    } catch (error) {
      logError(error)
    }
  }
  return false
}

/**
 * Applies the given file snapshot state to the tracked files (writes/deletes
 * on disk), returning the list of changed file paths.
```

### utils/fileHistory.ts
```
continue
      }

      if (backupFileName === null) {
        // File did not exist at the target version; delete it if present.
```

### utils/fileHistory.ts
```
continue
      }

      // File should exist at a specific version.
```

### utils/fileHistory.ts
```
Resume(log: LogOption): Promise<void> {
  if (!fileHistoryEnabled()) {
    return
  }

  const fileHistorySnapshots = log.
```

### utils/fileHistory.ts
```
resume_copy_failed'
```

### utils/fileHistory.ts
```
continue
    }

    // Get old content from the previous backup
    let oldContent: string | null = null
    if (oldBackup?.
```

### utils/filePersistence/outputsScanner.ts
```
continue
    }
    if (entry.
```

### utils/forkedAgent.ts
```
resumeAgentBackground to thread
   * state reconstructed from the resumed sidechain so the same results
   * are re-replaced (prompt cache stability).
```

### utils/forkedAgent.ts
```
resume (reconstructed from sidechain records)
    // and inProcessRunner (per-teammate persistent loop state).
```

### utils/forkedAgent.ts
```
continue
      }
      if (message.
```

### utils/forkedAgent.ts
```
continue
      }

      logForDebugging(
        `
```

### utils/frontmatterParser.ts
```
continue
      }

      // Skip if already quoted
      if (
        (value.
```

### utils/frontmatterParser.ts
```
continue
      }

      // Quote if contains special YAML characters
      if (YAML_SPECIAL_CHARS.
```

### utils/frontmatterParser.ts
```
continue
      }
    }

    result.
```

### utils/fsOperations.ts
```
continue with the original path rather than failing
 *
 * @param fs The filesystem implementation to use
 * @param filePath The path to resolve
 * @returns Object containing the resolved path and whether it was a symlink
 */
export function safeResolvePath(
  fs: FsOperations,
  filePath: string,
): { resolvedPath: string; isSymlink: boolean; isCanonical: boolean } {
  // Block UNC paths before an
```

### utils/fsOperations.ts
```
continue
    }
    if (st.
```

### utils/fsOperations.ts
```
continue with what we have
  }

  // Also add the final resolved path using realpathSync for completeness
  // This handles any remaining symlinks in directory components
  const { resolvedPath, isSymlink } = safeResolvePath(fsImpl, path)
  if (isSymlink && resolvedPath !== path) {
    pathSet.
```

### utils/fsOperations.ts
```
continue
      }

      remainder = Buffer.
```

### utils/genericProcessUtils.ts
```
Continue
        if (-not $proc -or -not $proc.
```

### utils/genericProcessUtils.ts
```
Continue
        if (-not $proc) { break }
        if ($proc.
```

### utils/git/gitConfigParser.ts
```
continue
    }

    // Section header
    if (trimmed[0] === '
```

### utils/git/gitConfigParser.ts
```
continue
    }

    if (!inSection) {
      continue
    }

    // Key-value line: find the key name
    const parsed = parseKeyValue(trimmed)
    if (parsed && parsed.
```

### utils/git/gitConfigParser.ts
```
continue
    }

    if (ch === '
```

### utils/git/gitConfigParser.ts
```
continue
      }
      // Outside quotes: backslash at end of line = continuation (we don'
```

### utils/git/gitConfigParser.ts
```
continue
      }
      // Fallthrough — treat backslash literally outside quotes
    }

    result += ch
    i++
  }

  // Trim trailing whitespace from unquoted portions.
```

### utils/git/gitConfigParser.ts
```
continue
      }
      // Git drops the backslash for other escapes in subsections
      foundSubsection += next
      i += 2
      continue
    }
    foundSubsection += line[i]
    i++
  }

  // Must have closing quote followed by '
```

### utils/git/gitFilesystem.ts
```
continue
      }
      const spaceIdx = line.
```

### utils/git/gitFilesystem.ts
```
continue
      }
      if (line.
```

### utils/git.ts
```
continue up
      }
      const parent = dirname(current)
      if (parent === current) {
        break
      }
      current = parent
    }

    // Check root directory as well
    try {
      const gitPath = join(root, '
```

### utils/git.ts
```
continue
    }

    try {
      const stats = await stat(filePath)
      const fileSize = stats.
```

### utils/git.ts
```
continue
      }

      // Check total size limit
      if (totalSize + fileSize > MAX_TOTAL_SIZE_BYTES) {
        logForDebugging(
          `
```

### utils/git.ts
```
continue
      }

      // Binary sniff on up to SNIFF_BUFFER_SIZE bytes.
```

### utils/git.ts
```
continue
        }

        let content: string
        if (fileSize <= sniffSize) {
          // Sniff already covers the whole file
          content = sniff.
```

### utils/gitDiff.ts
```
continue

    validFileCount++
    const addStr = parts[0]
    const remStr = parts[1]
    const filePath = parts.
```

### utils/gitDiff.ts
```
continue
    }

    const lines = fileDiff.
```

### utils/gitDiff.ts
```
continue
    const filePath = headerMatch[2] ?? headerMatch[1] ?? '
```

### utils/gitDiff.ts
```
continue
      }

      // Skip binary file markers and other metadata
      if (
        line.
```

### utils/gitDiff.ts
```
continue
      }

      // Add diff lines to current hunk (with line limit)
      if (
        currentHunk &&
        (line.
```

### utils/gitDiff.ts
```
continue
        }
        // Force a flat string copy to break V8 sliced string references.
```

### utils/gracefulShutdown.ts
```
ResumeHint() (and all sequences below)
    // land on the main buffer.
```

### utils/gracefulShutdown.ts
```
resume hint and the shell prompt lands on the wrong line.
```

### utils/gracefulShutdown.ts
```
ResumeHint still hits the main buffer.
```

### utils/gracefulShutdown.ts
```
ResumeHint() and clobber the
    // resume hint on tmux (and possibly other terminals) by restoring the
    // saved cursor position.
```

### utils/gracefulShutdown.ts
```
resumeHintPrinted = false

/**
 * Print a hint about how to resume the session.
```

### utils/gracefulShutdown.ts
```
ResumeHint(): void {
  // Only print once (failsafe timer may call this again after normal shutdown)
  if (resumeHintPrinted) {
    return
  }
  // Only show with TTY, interactive sessions, and persistence
  if (
    process.
```

### utils/gracefulShutdown.ts
```
resume hint if no session file exists (e.
```

### utils/gracefulShutdown.ts
```
resumeArg: string
      if (customTitle) {
        // Wrap in double quotes, escape backslashes first then quotes
        const escaped = customTitle.
```

### utils/gracefulShutdown.ts
```
resumeArg = sessionId
      }

      writeSync(
        1,
        chalk.
```

### utils/gracefulShutdown.ts
```
Resume this session with:\nclaude --resume ${resumeArg}\n`
```

### utils/gracefulShutdown.ts
```
resumeHintPrinted = true
    } catch {
      // Ignore write errors
    }
  }
}
/* eslint-enable custom-rules/no-sync-fs */

/**
 * Force process exit, handling the case where the terminal is gone.
```

### utils/gracefulShutdown.ts
```
ResumeHint()
      forceExit(exitCode)
    })
    // Prevent unhandled rejection: forceExit re-throws in test mode,
    // which would escape the .
```

### utils/gracefulShutdown.ts
```
resumeHintPrinted = false
  if (failsafeTimer !== undefined) {
    clearTimeout(failsafeTimer)
    failsafeTimer = undefined
  }
  pendingShutdown = undefined
}

/**
 * Returns the in-flight shutdown promise, if any.
```

### utils/gracefulShutdown.ts
```
ResumeHint()
      forceExit(code)
    },
    Math.
```

### utils/gracefulShutdown.ts
```
resume hint FIRST, before any async operations.
```

### utils/gracefulShutdown.ts
```
resume
  // hint would only appear after cleanup functions, hooks, and analytics
  // flush — which can take several seconds.
```

### utils/gracefulShutdown.ts
```
ResumeHint()

  // Flush session data first — this is the most critical cleanup.
```

### utils/groupToolUses.ts
```
continue
      }
    }

    // Skip user messages whose tool_results are all grouped
    if (msg.
```

### utils/groupToolUses.ts
```
continue
        }
      }
    }

    result.
```

### utils/handlePromptSubmit.ts
```
resumes in that
    // context — isolated from the parent'
```

### utils/heatmap.ts
```
continue
      }

      const dateStr = toDateString(currentDate)
      const activity = activityMap.
```

### utils/hooks/AsyncHookRegistry.ts
```
continue
    }
    const r = s.
```

### utils/hooks/execAgentHook.ts
```
continue
        }

        // Count assistant turns
        if (message.
```

### utils/hooks/fileChangedWatcher.ts
```
continue
    for (const name of m.
```

### utils/hooks/fileChangedWatcher.ts
```
continue
      staticPaths.
```

### utils/hooks/hooksConfigManager.ts
```
continue with tool call'
```

### utils/hooks/hooksConfigManager.ts
```
continue conversation\nOther exit codes - show stderr to user only'
```

### utils/hooks/hooksConfigManager.ts
```
continue having it run\nOther exit codes - show stderr to user only'
```

### utils/hooks/hooksConfigManager.ts
```
continue with compaction'
```

### utils/hooks/hooksConfigManager.ts
```
continues working)\nOther exit codes - show stderr to user only'
```

### utils/hooks/hooksConfigManager.ts
```
continue

      for (const matcher of matchers) {
        const matcherKey = matcher.
```

### utils/hooks/hooksSettings.ts
```
continue
        }
        seenFiles.
```

### utils/hooks/hooksSettings.ts
```
continue
      }

      for (const [event, matchers] of Object.
```

### utils/hooks/registerFrontmatterHooks.ts
```
continue
    }

    // For agents, convert Stop hooks to SubagentStop since that'
```

### utils/hooks/registerFrontmatterHooks.ts
```
continue
      }

      for (const hook of hooksArray) {
        addSessionHook(setAppState, sessionId, targetEvent, matcher, hook)
        hookCount++
      }
    }
  }

  if (hookCount > 0) {
    logForDebugging(
      `
```

### utils/hooks/registerSkillHooks.ts
```
continue

    for (const matcher of matchers) {
      for (const hook of matcher.
```

### utils/hooks.ts
```
continue === false) {
    result.
```

### utils/hooks.ts
```
continue

        try {
          const parsed = jsonParse(trimmed)
          const validation = promptRequestSchema().
```

### utils/hooks.ts
```
continue
          }
        } catch {
          // Not JSON, just a normal line
        }
      }
    }

    // Check for async response on first line of output.
```

### utils/hooks.ts
```
continue
      }
      hooks.
```

### utils/hooks.ts
```
continue working instead of going idle.
```

### utils/hooks.ts
```
resume, clear)
 * @param sessionId Optional The session id to use as hook input
 * @param agentType Optional The agent type (from --agent flag) running this session
 * @param model Optional The model being used for this session
 * @param signal Optional AbortSignal to cancel hook execution
 * @param timeoutMs Optional timeout in milliseconds for hook execution
 * @returns Async generator that yiel
```

### utils/http.ts
```
pick up the refreshed token.
```

### utils/ide.ts
```
continue
      }
      if (
        user.
```

### utils/ide.ts
```
continue // Skip system directories
      }
      paths.
```

### utils/ide.ts
```
continue
      }

      const host = await detectHostIP(
        lockfileInfo.
```

### utils/ide.ts
```
resumes the search.
```

### utils/ide.ts
```
continue
    }
    const ides = await detectIDEs(false)
    if (signal.
```

### utils/ide.ts
```
continue

      let isValid = false
      if (isEnvTruthy(process.
```

### utils/ide.ts
```
continue
      }

      // PID ancestry check: when running in a supported IDE'
```

### utils/ide.ts
```
continue
          }
          if (process.
```

### utils/ide.ts
```
continue
            }
          }
        }
      }

      const ideName =
        lockfileInfo.
```

### utils/imageStore.ts
```
continue
      }

      const sessionPath = join(baseDir, sessionDir.
```

### utils/imageValidation.ts
```
continue

    const m = msg as Record<string, unknown>

    // Handle wrapped message format { type: '
```

### utils/imageValidation.ts
```
continue

    const innerMessage = m.
```

### utils/imageValidation.ts
```
continue

    const content = innerMessage.
```

### utils/imageValidation.ts
```
continue

    for (const block of content) {
      if (isBase64ImageBlock(block)) {
        imageIndex++
        // Check the base64-encoded string length directly (not decoded bytes)
        // The API limit applies to the base64 payload size
        const base64Size = block.
```

### utils/jetbrains.ts
```
continue
          // Accept symlinks too — dirent.
```

### utils/jetbrains.ts
```
continue
          const dir = join(baseDir, entry.
```

### utils/jetbrains.ts
```
continue
          }
          const pluginDir = join(dir, '
```

### utils/jetbrains.ts
```
continue
    }
  }

  return foundDirectories.
```

### utils/jetbrains.ts
```
continue
    }
  }
  return false
}

const pluginInstalledCache = new Map<IdeType, boolean>()
const pluginInstalledPromiseCache = new Map<IdeType, Promise<boolean>>()

async function isJetBrainsPluginInstalledMemoized(
  ideType: IdeType,
  forceRefresh = false,
): Promise<boolean> {
  if (!forceRefresh) {
    const existing = pluginInstalledPromiseCache.
```

### utils/json.ts
```
continue
    try {
      results.
```

### utils/json.ts
```
continue
    try {
      results.
```

### utils/listSessionsImpl.ts
```
continue
      if (seen.
```

### utils/listSessionsImpl.ts
```
continue
      seen.
```

### utils/listSessionsImpl.ts
```
continue
      }
      sessions.
```

### utils/listSessionsImpl.ts
```
continue
    const existing = byId.
```

### utils/listSessionsImpl.ts
```
continue
    const dirName = caseInsensitive ? dirent.
```

### utils/listSessionsImpl.ts
```
continue

    for (const { path: wtPath, prefix } of indexed) {
      // Only use startsWith for truncated paths (>MAX_SANITIZED_LENGTH) where
      // a hash suffix follows.
```

### utils/managedEnv.ts
```
continue
    if (!isSettingSourceEnabled(source)) continue
    Object.
```

### utils/markdownConfigLoader.ts
```
continue
      }
      const existingSource = seenFileIds.
```

### utils/markdownConfigLoader.ts
```
continue
      }
      seenFileIds.
```

### utils/markdownConfigLoader.ts
```
continue
      const errorMessage =
        error instanceof Error ? error.
```

### utils/mcpInstructionsDelta.ts
```
continue
    attachmentCount++
    if (msg.
```

### utils/mcpInstructionsDelta.ts
```
continue
    midCount++
    for (const n of msg.
```

### utils/mcpInstructionsDelta.ts
```
continue
    const existing = blocks.
```

### utils/messageQueueManager.ts
```
continue
    const priority = PRIORITY_ORDER[cmd.
```

### utils/messageQueueManager.ts
```
continue
    const priority = PRIORITY_ORDER[cmd.
```

### utils/messages.ts
```
continue working on those.
```

### utils/messages.ts
```
continue with other tasks that don'
```

### utils/messages.ts
```
continue
    }

    // Handle pre-tool-use hooks
    if (
      isHookAttachmentMessage(message) &&
      message.
```

### utils/messages.ts
```
continue
    }

    // Handle tool results
    if (
      message.
```

### utils/messages.ts
```
continue
    }

    // Handle post-tool-use hooks
    if (
      isHookAttachmentMessage(message) &&
      message.
```

### utils/messages.ts
```
continue
    }
  }

  // Second pass: reconstruct the message list in the correct order
  const result: (
    | NormalizedUserMessage
    | NormalizedAssistantMessage
    | AttachmentMessage
    | SystemMessage
  )[] = []
  const processedToolUses = new Set<string>()

  for (const message of messages) {
    // Check if this is a tool use
    if (isToolUseRequestMessage(message)) {
      const tool
```

### utils/messages.ts
```
continue
    }

    // Check if this message is part of a tool use group
    if (
      isHookAttachmentMessage(message) &&
      (message.
```

### utils/messages.ts
```
continue
    }

    if (
      message.
```

### utils/messages.ts
```
continue
    }

    // Handle api error messages (only keep the last one)
    if (message.
```

### utils/messages.ts
```
continue
    }

    // Add standalone messages
    result.
```

### utils/messages.ts
```
continue
    // Skip blocks from the last original message if it'
```

### utils/messages.ts
```
continue
    for (const content of msg.
```

### utils/messages.ts
```
resumed session with one
 * of these 400s on every call and can'
```

### utils/messages.ts
```
continue
    const content = msg.
```

### utils/messages.ts
```
continue
    if (!contentHasToolReference(content)) continue

    const textSiblings = content.
```

### utils/messages.ts
```
continue

    // Find the next user message with tool_result but no tool_reference.
```

### utils/messages.ts
```
continue
      const cc = cand.
```

### utils/messages.ts
```
continue
      if (!cc.
```

### utils/messages.ts
```
continue
      if (contentHasToolReference(cc)) continue
      targetIdx = j
      break
    }

    if (targetIdx === -1) continue // No valid target; leave in place.
```

### utils/messages.ts
```
continue
    }
    // Determine which error this is
    const errorText =
      Array.
```

### utils/messages.ts
```
continue
    }
    const blockTypesToStrip = errorToBlockTypes[errorText]
    if (!blockTypesToStrip) {
      continue
    }
    // Walk backward to find the nearest preceding isMeta user message
    for (let j = i - 1; j >= 0; j--) {
      const candidate = reorderedMessages[j]!
      if (candidate.
```

### utils/messages.ts
```
continue
      }
      // Stop if we hit an assistant message or non-meta user message
      break
    }
  }

  const result: (UserMessage | AssistantMessage)[] = []
  reorderedMessages
    .
```

### utils/messages.ts
```
continue
            }
          }

          result.
```

### utils/messages.ts
```
resumed session with an
  // image-in-error tool_result 400s forever.
```

### utils/messages.ts
```
continue
      const prev = merged.
```

### utils/messages.ts
```
continue
    const content = msg.
```

### utils/messages.ts
```
continue
    for (const block of content) {
      if (block.
```

### utils/messages.ts
```
continue working on it.
```

### utils/messages.ts
```
Continue to follow these guidelines:\n\n${skillsContent}`
```

### utils/messages.ts
```
resumed sessions that predate
          // the stored-header field.
```

### utils/messages.ts
```
Continue on with the plan process and most importantly you should always edit the plan file one way or the other before calling ${ExitPlanModeV2Tool.
```

### utils/messages.ts
```
continue working seamlessly.
```

### utils/messages.ts
```
continue
    if (msg.
```

### utils/messages.ts
```
continue
    if (msg.
```

### utils/messages.ts
```
continue
    if (msg.
```

### utils/messages.ts
```
resume, interleaved user messages or attachments
 * can prevent proper merging by message.
```

### utils/messages.ts
```
continue

    const content = msg.
```

### utils/messages.ts
```
continue

    const hasNonThinking = content.
```

### utils/messages.ts
```
resume when the transcript starts mid-turn
      // (e.
```

### utils/messages.ts
```
continue
        }
      }
      result.
```

### utils/messages.ts
```
continue
    }

    // Collect server-side tool result IDs (*_tool_result blocks have tool_use_id).
```

### utils/messages.ts
```
continue
    }

    repaired = true

    // Build synthetic error tool_result blocks for missing IDs
    const syntheticBlocks: ToolResultBlockParam[] = missingIds.
```

### utils/model/deprecation.ts
```
continue
    }
    return {
      isDeprecated: true,
      modelName: value.
```

### utils/model/modelAllowlist.ts
```
continue
    }
    // Check if entry is a version-qualified variant of this family
    // e.
```

### utils/model/modelAllowlist.ts
```
continue
    }
    const afterFamily = idx + family.
```

### utils/model/modelSupportOverrides.ts
```
continue
      if (m !== pinned.
```

### utils/model/modelSupportOverrides.ts
```
continue
      return capabilities
        .
```

### utils/nativeInstaller/download.ts
```
continue
      }

      // Don'
```

### utils/nativeInstaller/installer.ts
```
Continue with copy if we can'
```

### utils/nativeInstaller/installer.ts
```
continue
        try {
          await unlink(join(executableDir, file))
          cleanedCount++
        } catch {
          // File might still be in use by another process
        }
      }
      if (cleanedCount > 0) {
        logForDebugging(
          `
```

### utils/nativeInstaller/installer.ts
```
continue
    }
    // Candidate version binary — stat once, reuse for isFile/size/mtime/mode
    try {
      const stats = await stat(entryPath)
      if (!stats.
```

### utils/nativeInstaller/installer.ts
```
continue
      if (
        process.
```

### utils/nativeInstaller/installer.ts
```
continue
      }
      versionFiles.
```

### utils/nativeInstaller/installer.ts
```
continue

      const lockFilePath = getLockFilePathFromVersionPath(dirs, v.
```

### utils/nativeInstaller/installer.ts
```
continue

      const { filtered, hadAlias } = filterClaudeAliases(lines)

      if (hadAlias) {
        await writeFileLines(configFile, filtered)
        messages.
```

### utils/notebook.ts
```
continue
    size += (o.
```

### utils/pasteStore.ts
```
continue
    }

    const filePath = join(pasteDir, file)
    try {
      const stats = await stat(filePath)
      if (stats.
```

### utils/permissions/filesystem.ts
```
continue
      }

      // Special case: .
```

### utils/permissions/filesystem.ts
```
continue checking other segments
        }
      }

      return true
    }
  }

  // Check for dangerous configuration files (case-insensitive)
  if (fileName) {
    const normalizedFileName = normalizeCaseForComparison(fileName)
    if (
      (DANGEROUS_FILES as readonly string[]).
```

### utils/permissions/filesystem.ts
```
continue
    }

    // Check each pattern to see if the full path starts with our reference root
    for (const pattern of patterns) {
      const normalizedPattern = normalizePatternToPath({
        patternRoot,
        pattern,
        rootPath: root,
      })
      if (normalizedPattern) {
        result.
```

### utils/permissions/filesystem.ts
```
continue
    }

    // Important: ig.
```

### utils/permissions/filesystem.ts
```
continue
    }

    const igResult = ig.
```

### utils/permissions/permissionSetup.ts
```
continue
    }
    const destination = perm.
```

### utils/permissions/permissionSetup.ts
```
continue
    }
    for (const ruleString of ruleStrings) {
      const ruleValue = permissionRuleValueFromString(ruleString)
      rules.
```

### utils/permissions/permissionSetup.ts
```
continue
    ;(stripped[perm.
```

### utils/permissions/permissionSetup.ts
```
continue
    result = applyPermissionUpdate(result, {
      type: '
```

### utils/permissions/permissionSetup.ts
```
continue // Skip this mode if it'
```

### utils/permissions/permissionSetup.ts
```
continue

    let current = '
```

### utils/permissions/permissions.ts
```
continue
      }
      const decision = hookResult.
```

### utils/permissions/shellRuleMatching.ts
```
continue
      } else if (nextChar === '
```

### utils/permissions/shellRuleMatching.ts
```
continue
      }
    }

    processed += char
    i++
  }

  // Escape regex special characters except *
  const escaped = processed.
```

### utils/permissions/yoloClassifier.ts
```
continue
      switch (entry.
```

### utils/plans.ts
```
resumed,
 *                        not the temporary session ID from before resume.
```

### utils/plans.ts
```
Resume(
  log: LogOption,
  targetSessionId?: SessionId,
): Promise<boolean> {
  const slug = getSlugFromLog(log)
  if (!slug) {
    return false
  }

  // Set the slug for the target session ID (or current if not provided)
  const sessionId = targetSessionId ?? getSessionId()
  setPlanSlug(sessionId, slug)

  // Attempt to read the plan file directly — recovery triggers on ENOENT.
```

### utils/plans.ts
```
resume: ${planPath}.
```

### utils/plans.ts
```
Resume (which reuses
 * the original slug), this generates a NEW slug for the forked session and
 * writes the original plan content to the new file.
```

### utils/plans.ts
```
continue
    }

    if (msg.
```

### utils/plugins/addDirPluginSettings.ts
```
continue
      }
      Object.
```

### utils/plugins/addDirPluginSettings.ts
```
continue
      }
      Object.
```

### utils/plugins/cacheUtils.ts
```
continue
          await processOrphanedPluginVersion(versionPath, now)
        }

        await removeIfEmpty(pluginPath)
      }

      await removeIfEmpty(marketplacePath)
    }
  } catch (error) {
    logForDebugging(`
```

### utils/plugins/dependencyResolver.ts
```
continue
      for (const rawDep of p.
```

### utils/plugins/installedPluginsManager.ts
```
continue
      }

      const entry = dirent.
```

### utils/plugins/installedPluginsManager.ts
```
continue
      }

      // This is a legacy flat cache directory
      // Check if it'
```

### utils/plugins/installedPluginsManager.ts
```
continues using the old version.
```

### utils/plugins/installedPluginsManager.ts
```
continue

    for (const diskEntry of diskInstallations) {
      const memoryEntry = memoryInstallations.
```

### utils/plugins/installedPluginsManager.ts
```
continue

    for (const diskEntry of diskInstallations) {
      const memoryEntry = memoryInstallations.
```

### utils/plugins/installedPluginsManager.ts
```
continue

    for (const diskEntry of diskInstallations) {
      const memoryEntry = memoryInstallations.
```

### utils/plugins/installedPluginsManager.ts
```
continue
    }

    for (const entry of data.
```

### utils/plugins/installedPluginsManager.ts
```
continue

      // Settings.
```

### utils/plugins/installedPluginsManager.ts
```
continue
      }

      try {
        logForDebugging(
          `
```

### utils/plugins/installedPluginsManager.ts
```
continue
        }

        const { entry, marketplaceInstallLocation } = pluginInfo

        let installPath: string
        let version = '
```

### utils/plugins/installedPluginsManager.ts
```
continue
          }

          installPath = pluginCachePath

          // Only read manifest if the .
```

### utils/plugins/loadPluginHooks.ts
```
continue
    }

    for (const matcher of matchers) {
      if (matcher.
```

### utils/plugins/loadPluginHooks.ts
```
continue
    }

    logForDebugging(`
```

### utils/plugins/lspPluginIntegration.ts
```
continue
      }

      // Load from file
      try {
        const content = await readFile(validatedPath, '
```

### utils/plugins/lspPluginIntegration.ts
```
continue

    const servers = await loadPluginLspServers(plugin, errors)
    if (servers) {
      const scopedServers = addPluginScopeToLspServers(servers, plugin.
```

### utils/plugins/lspRecommendation.ts
```
continue
      }
      // Try to extract from inline config object
      const info = extractFromServerConfigRecord(item)
      if (info) {
        return info
      }
    }
    return null
  }

  // It'
```

### utils/plugins/lspRecommendation.ts
```
continue
    }

    // Get command from first valid server config
    if (!command && typeof config.
```

### utils/plugins/lspRecommendation.ts
```
continue
          }

          const lspInfo = extractLspInfoFromManifest(entry.
```

### utils/plugins/lspRecommendation.ts
```
continue
          }

          const pluginId = `
```

### utils/plugins/lspRecommendation.ts
```
continue
    }

    // Filter: not in "
```

### utils/plugins/lspRecommendation.ts
```
continue
    }

    // Filter: not already installed
    if (isPluginInstalled(pluginId)) {
      logForDebugging(
        `
```

### utils/plugins/lspRecommendation.ts
```
continue
    }

    matchingPlugins.
```

### utils/plugins/managedPlugins.ts
```
continue
    }
    const name = pluginId.
```

### utils/plugins/marketplaceHelpers.ts
```
continue
    }

    let data = null
    try {
      data = await getMarketplace(name)
    } catch (err) {
      // Track individual marketplace failures but continue loading others
      const errorMessage = err instanceof Error ? err.
```

### utils/plugins/marketplaceManager.ts
```
continue

    for (const [name, seedEntry] of Object.
```

### utils/plugins/marketplaceManager.ts
```
continue

      // Compute installLocation relative to THIS seedDir, not the build-time
      // path baked into the seed'
```

### utils/plugins/marketplaceManager.ts
```
continue
      }
      claimed.
```

### utils/plugins/marketplaceManager.ts
```
continue

      // Seed wins — admin-managed.
```

### utils/plugins/marketplaceManager.ts
```
continue

    let needsUpdate = false
    const updates: {
      extraKnownMarketplaces?: typeof settings.
```

### utils/plugins/marketplaceManager.ts
```
Continues refreshing even if some marketplaces fail.
```

### utils/plugins/marketplaceManager.ts
```
continue
    }
    // settings-sourced marketplaces have no upstream — see refreshMarketplace.
```

### utils/plugins/marketplaceManager.ts
```
continue
    }
    // inc-5046: same GCS intercept as refreshMarketplace() — bulk update
    // hits this path on `
```

### utils/plugins/marketplaceManager.ts
```
continue
      }
      if (
        !getFeatureValue_CACHED_MAY_BE_STALE(
          '
```

### utils/plugins/marketplaceManager.ts
```
continue
      }
      // fall through to git
    }
    try {
      const { cachePath } = await loadAndCacheMarketplace(entry.
```

### utils/plugins/mcpPluginIntegration.ts
```
continue
    }
    const saved = loadMcpServerUserConfig(pluginId, channel.
```

### utils/plugins/mcpbHandler.ts
```
continue
    }

    // Skip validation for optional fields that aren'
```

### utils/plugins/mcpbHandler.ts
```
continue
    }

    // Type validation
    if (fieldSchema.
```

### utils/plugins/officialMarketplaceGcs.ts
```
continue
      const rel = arcPath.
```

### utils/plugins/officialMarketplaceGcs.ts
```
continue // prefix dir entry or subdir entry
      const dest = join(staging, rel)
      await mkdir(dirname(dest), { recursive: true })
      await writeFile(dest, data)
      const mode = modes[arcPath]
      if (mode && mode & 0o111) {
        // Only chmod when an exec bit is set — skip plain files to save syscalls.
```

### utils/plugins/pluginBlocklist.ts
```
continue

    const pluginName = pluginId.
```

### utils/plugins/pluginBlocklist.ts
```
continue

      const delisted = detectDelistedPlugins(
        installedPlugins,
        marketplace,
        marketplaceName,
      )

      for (const pluginId of delisted) {
        if (pluginId in alreadyFlagged) continue

        // Skip managed-only plugins — enterprise admin should handle those
        const installations = installedPlugins.
```

### utils/plugins/pluginBlocklist.ts
```
continue

        // Auto-uninstall the delisted plugin from all user-controllable scopes
        for (const installation of installations) {
          const { scope } = installation
          if (scope !== '
```

### utils/plugins/pluginBlocklist.ts
```
continue
          }
          try {
            await uninstallPluginOp(pluginId, scope)
          } catch (error) {
            logForDebugging(
              `
```

### utils/plugins/pluginBlocklist.ts
```
continue
      logForDebugging(
        `
```

### utils/plugins/pluginInstallationHelpers.ts
```
continue

    let localSourcePath: string | undefined
    const { source } = info.
```

### utils/plugins/pluginLoader.ts
```
continue
      const versionDir = join(pluginDir, versions[0]!)
      const entries = await readdir(versionDir)
      if (entries.
```

### utils/plugins/pluginLoader.ts
```
continue
      }

      // Resolve the source directory to handle symlinked source dirs
      let resolvedSrc: string
      try {
        resolvedSrc = await realpath(src)
      } catch {
        resolvedSrc = src
      }

      // Check if target is within the source tree (using proper path prefix matching)
      const srcPrefix = resolvedSrc.
```

### utils/plugins/pluginLoader.ts
```
continue
        if (check.
```

### utils/plugins/pluginLoader.ts
```
continue
        }
        // kind === '
```

### utils/plugins/pluginLoader.ts
```
continue
        }
        if (check.
```

### utils/plugins/pluginLoader.ts
```
continue
        }

        // Check if this path resolves to an already-loaded hooks file
        let normalizedPath: string
        try {
          normalizedPath = await realpath(hookFilePath)
        } catch {
          // If realpathSync fails, use original path
          normalizedPath = hookFilePath
        }

        if (loadedHookPaths.
```

### utils/plugins/pluginLoader.ts
```
continue
        }

        try {
          const additionalHooks = await loadPluginHooks(
            hookFilePath,
            manifest.
```

### utils/plugins/pluginLoader.ts
```
continue
          if (check.
```

### utils/plugins/pluginLoader.ts
```
continue
          }
          if (check.
```

### utils/plugins/pluginLoader.ts
```
continue
          if (check.
```

### utils/plugins/pluginLoader.ts
```
continue
          }
          if (check.
```

### utils/plugins/pluginLoader.ts
```
continue
      }

      const dirName = basename(resolvedPath)
      const { plugin, errors: pluginErrors } = await createPluginFromPath(
        resolvedPath,
        `
```

### utils/plugins/pluginLoader.ts
```
continues loading other plugins on errors
 * - Errors include source information for debugging
 *
 * @returns Promise resolving to categorized plugin results:
 *   - enabled: Array of enabled LoadedPlugin objects
 *   - disabled: Array of disabled LoadedPlugin objects
 *   - errors: Array of loading errors with source information
 */
export const loadAllPlugins = memoize(async (): Promise<PluginLo
```

### utils/plugins/pluginLoader.ts
```
continue
    }

    if (!merged) {
      merged = {}
    }

    for (const [key, value] of Object.
```

### utils/plugins/pluginStartupCheck.ts
```
continue
      }
      const idx = enabledPlugins.
```

### utils/plugins/pluginStartupCheck.ts
```
continue
    }
    if (value === true) {
      result.
```

### utils/plugins/pluginStartupCheck.ts
```
continue
    }

    for (const [pluginId, value] of Object.
```

### utils/plugins/pluginStartupCheck.ts
```
continue
      }

      // Log when a standard source overrides an --add-dir plugin
      if (pluginId in addDirPlugins && addDirPlugins[pluginId] !== value) {
        logForDebugging(
          `
```

### utils/plugins/pluginStartupCheck.ts
```
continue

    if (onProgress) {
      onProgress(pluginId, i + 1, pluginsToInstall.
```

### utils/plugins/pluginStartupCheck.ts
```
continue
      }

      // Cache the plugin if it'
```

### utils/plugins/reconciler.ts
```
continue
    }
    // For sourceChanged local-path entries, skip if the declared path doesn'
```

### utils/plugins/reconciler.ts
```
continue
    }
    toProcess.
```

### utils/plugins/refresh.ts
```
pick up new plugin MCP servers.
```

### utils/plugins/validatePlugin.ts
```
continue
        }
        const pluginJsonPath = path.
```

### utils/plugins/validatePlugin.ts
```
continue
        }
        if (manifestVersion && manifestVersion !== entry.
```

### utils/plugins/validatePlugin.ts
```
continue
        results.
```

### utils/plugins/validatePlugin.ts
```
continue
      }
      const r = validateComponentFile(filePath, content, fileType)
      if (r.
```

### utils/plugins/zipCache.ts
```
continue
    }

    const fullPath = join(currentDir, entry)
    const relPath = relativePath ? `
```

### utils/plugins/zipCache.ts
```
continue
    }

    // Skip symlinked directories (follow symlinked files)
    if (fileStat.
```

### utils/plugins/zipCache.ts
```
continue
        }
        // Symlinked file — read its contents below
        fileStat = targetStat
      } catch {
        continue // broken symlink
      }
    }

    if (fileStat.
```

### utils/plugins/zipCache.ts
```
continue
    }

    const fullPath = join(targetDir, relPath)
    await getFsImplementation().
```

### utils/plugins/zipCacheAdapters.ts
```
continue
    try {
      await saveMarketplaceJsonToZipCache(name, entry.
```

### utils/powershell/parser.ts
```
continue }
                $nested = @{
                    type = $cmd.
```

### utils/powershell/staticPrefix.ts
```
continue
      }
      // Positional arg that isn'
```

### utils/powershell/staticPrefix.ts
```
continue
    }
    const prefix = await extractPrefixFromElement(cmd)
    if (prefix) {
      prefixes.
```

### utils/powershell/staticPrefix.ts
```
continue
      }
    }
    collapsed.
```

### utils/processUserInput/processSlashCommand.tsx
```
ContinueAnimation: true,
      showSpinner: true
    });
  };

  // Show initial "
```

### utils/processUserInput/processSlashCommand.tsx
```
resume looks at latest timestamp message to determine which message to resume from
                // This is a perf optimization to avoid having to recaculcate the leaf node every time
                // Since we'
```

### utils/processUserInput/processUserInput.ts
```
continue
    }

    // Return only a system-level error message, erasing the original user input
    if (hookResult.
```

### utils/promptEditor.ts
```
resumeStdin()
      inkInstance.
```

### utils/promptEditor.ts
```
resume()
    }
  }
}

/**
 * Re-collapse expanded pasted text by finding content that matches
 * pastedContents and replacing it with references.
```

### utils/queryContext.ts
```
resume before a turn
 * completes — there'
```

### utils/queryHelpers.ts
```
continue
        }
        yield {
          type: '
```

### utils/queryHelpers.ts
```
resume, mutableMessages is seeded from the transcript and may already
  // contain this tool_use.
```

### utils/queryHelpers.ts
```
resume,
          // Cowork cold-restart per turn), so disk content at extraction time
          // IS the post-edit state.
```

### utils/queryHelpers.ts
```
continue
          const cmd = extractCliName(
            typeof input.
```

### utils/queryHelpers.ts
```
continue
    if (STRIPPED_COMMANDS.
```

### utils/readFileInRange.ts
```
continues (to count totalLines).
```

### utils/releaseNotes.ts
```
continue

      // Extract version from the first line
      // Handle both "
```

### utils/releaseNotes.ts
```
continue

      // First part before any dash is the version
      const version = versionLine.
```

### utils/releaseNotes.ts
```
continue

      // Extract bullet points
      const notes = lines
        .
```

### utils/sessionRestore.ts
```
ResumedSessionFile,
  recordContentReplacement,
  resetSessionFilePointer,
  restoreSessionMetadata,
  saveMode,
  saveWorktreeState,
} from '
```

### utils/sessionRestore.ts
```
ResumeResult = {
  messages?: Message[]
  fileHistorySnapshots?: FileHistorySnapshot[]
  attributionSnapshots?: AttributionSnapshotMessage[]
  contextCollapseCommits?: ContextCollapseCommitEntry[]
  contextCollapseSnapshot?: ContextCollapseSnapshotEntry
}

/**
 * Scan the transcript for the last TodoWrite tool_use block and return its todos.
```

### utils/sessionRestore.ts
```
resume so the model'
```

### utils/sessionRestore.ts
```
continue
    const toolUse = msg.
```

### utils/sessionRestore.ts
```
continue
    const input = toolUse.
```

### utils/sessionRestore.ts
```
ResumeResult,
  setAppState: (f: (prev: AppState) => AppState) => void,
): void {
  // Restore file history state
  if (result.
```

### utils/sessionRestore.ts
```
resumed Message[].
```

### utils/sessionRestore.ts
```
resume into a session with no
  // commits would leave the prior session'
```

### utils/sessionRestore.ts
```
ResumeResult,
): AttributionState | undefined {
  if (
    feature('
```

### utils/sessionRestore.ts
```
resumedAgent = agentDefinitions.
```

### utils/sessionRestore.ts
```
resumedAgent) {
    logForDebugging(
      `
```

### utils/sessionRestore.ts
```
Resumed session had agent "
```

### utils/sessionRestore.ts
```
resumedAgent, agentType: resumedAgent.
```

### utils/sessionRestore.ts
```
resumed/continued conversation for rendering.
```

### utils/sessionRestore.ts
```
Resume = {
  messages: Message[]
  fileHistorySnapshots?: FileHistorySnapshot[]
  contentReplacements?: ContentReplacementRecord[]
  agentName: string | undefined
  agentColor: AgentColorName | undefined
  restoredAgentDef: AgentDefinition | undefined
  initialState: AppState
}

/**
 * Subset of the coordinator mode module API needed for session resume.
```

### utils/sessionRestore.ts
```
ResumeLoadResult = {
  messages: Message[]
  fileHistorySnapshots?: FileHistorySnapshot[]
  attributionSnapshots?: AttributionSnapshotMessage[]
  contentReplacements?: ContentReplacementRecord[]
  contextCollapseCommits?: ContextCollapseCommitEntry[]
  contextCollapseSnapshot?: ContextCollapseSnapshotEntry
  sessionId: UUID | undefined
  agentName?: string
  agentColor?: string
  agentSetting?: st
```

### utils/sessionRestore.ts
```
ResumedSessionFile writes
 * it back to disk.
```

### utils/sessionRestore.ts
```
Resume(
  worktreeSession: PersistedWorktreeSession | null | undefined,
): void {
  const fresh = getCurrentWorktreeSession()
  if (fresh) {
    saveWorktreeState(fresh)
    return
  }
  if (!worktreeSession) return

  try {
    process.
```

### utils/sessionRestore.ts
```
resume slash command calls this mid-session after caches have been
  // populated against the old cwd.
```

### utils/sessionRestore.ts
```
Resume before a mid-session /resume switches to
 * another session.
```

### utils/sessionRestore.ts
```
resume from a worktree session to a
 * non-worktree session leaves the user in the old worktree directory with
 * currentWorktreeSession still pointing at the prior session.
```

### utils/sessionRestore.ts
```
resume to a
 * *different* worktree fails entirely — the getCurrentWorktreeSession()
 * guard above blocks the switch.
```

### utils/sessionRestore.ts
```
resume/--continue: those run once at startup where
 * getCurrentWorktreeSession() is only truthy if --worktree was used (fresh
 * worktree that should take precedence, handled by the re-assert above).
```

### utils/sessionRestore.ts
```
Resume
    // will cd into the target worktree next if there is one.
```

### utils/sessionRestore.ts
```
continue
 * and --resume paths in main.
```

### utils/sessionRestore.ts
```
ResumedConversation(
  result: ResumeLoadResult,
  opts: {
    forkSession: boolean
    sessionIdOverride?: string
    transcriptPath?: string
    includeAttribution?: boolean
  },
  context: {
    modeApi: CoordinatorModeApi | null
    mainThreadAgentDefinition: AgentDefinition | undefined
    agentDefinitions: AgentDefinitionsResult
    currentCwd: string
    cliAgents: AgentDefinition[]
    ini
```

### utils/sessionRestore.ts
```
resumed session ID so
      // getSessionRecordingPaths() can discover it during /share
      await renameRecordingForSession()
      await resetSessionFilePointer()
      restoreCostStateForSession(sid)
    }
  } else if (result.
```

### utils/sessionRestore.ts
```
ResumedSessionFile writes it.
```

### utils/sessionRestore.ts
```
resumed transcript and re-append metadata
    // now.
```

### utils/sessionRestore.ts
```
ResumedSessionFile()
  }

  // Restore context-collapse commit log + staged snapshot.
```

### utils/sessionRestore.ts
```
resume path goes through restoreSessionStateFromLog (REPL.
```

### utils/sessionRestore.ts
```
continue/--resume goes through here instead.
```

### utils/sessionRestore.ts
```
resumed session
  const { agentDefinition: restoredAgent, agentType: resumedAgentType } =
    restoreAgentFromSession(
      result.
```

### utils/sessionRestore.ts
```
resumes know what mode this session was in
  if (feature('
```

### utils/sessionRestore.ts
```
resumedAgentType && { agent: resumedAgentType }),
      .
```

### utils/sessionStart.ts
```
continue with session start without plugin hooks
      /* eslint-disable no-restricted-syntax -- both branches wrap with context, not a toError case */
      const enhancedError =
        error instanceof Error
          ? new Error(
              `
```

### utils/sessionStart.ts
```
Continue execution - plugin hooks won'
```

### utils/sessionStorage.ts
```
resume (see #14373, #23537).
```

### utils/sessionStorage.ts
```
resume/branch)
  // — different directories, so the hook sees MISSING (gh-30217).
```

### utils/sessionStorage.ts
```
resume to
 * route correctly when subagent_type is omitted — without this, resuming
 * a fork silently degrades to general-purpose (4KB system prompt, no
 * inherited history).
```

### utils/sessionStorage.ts
```
resume to restore the correct cwd.
```

### utils/sessionStorage.ts
```
continue
    try {
      const raw = await readFile(join(dir, entry.
```

### utils/sessionStorage.ts
```
resume
        // shows the auto-generated firstPrompt instead.
```

### utils/sessionStorage.ts
```
resume knows the session exited (vs.
```

### utils/sessionStorage.ts
```
continue
      }
      const batch = queue.
```

### utils/sessionStorage.ts
```
resume, messages arrive as SerializedMessage (carries source
          // sessionId/cwd/etc.
```

### utils/sessionStorage.ts
```
resume picker shows what the user was last doing.
```

### utils/sessionStorage.ts
```
resume); main-thread
      // records go to the session file (for /resume).
```

### utils/sessionStorage.ts
```
resume-of-fork loads a 10KB file instead of the full 85KB inherited
        // context).
```

### utils/sessionStorage.ts
```
continue chain at compact boundary).
```

### utils/sessionStorage.ts
```
resume scenarios where every message is
  // already in messageSet).
```

### utils/sessionStorage.ts
```
continue/--resume (non-fork).
```

### utils/sessionStorage.ts
```
resumed
 * mode/tag/agent).
```

### utils/sessionStorage.ts
```
resumed file
 * already exists on disk (we loaded from it), so this can'
```

### utils/sessionStorage.ts
```
ResumedSessionFile(): void {
  const project = getProject()
  project.
```

### utils/sessionStorage.ts
```
resume these are collected into an ordered
 * array and handed to restoreFromEntries() which rebuilds the commit log.
```

### utils/sessionStorage.ts
```
continue
          let list = byAgent.
```

### utils/sessionStorage.ts
```
continue
    // Skip compact summary messages - they should not be treated as the first prompt
    if ('
```

### utils/sessionStorage.ts
```
continue

    const content = msg.
```

### utils/sessionStorage.ts
```
continue

    // Collect all text values.
```

### utils/sessionStorage.ts
```
continue

      const commandNameTag = extractTag(textContent, COMMAND_NAME_TAG)
      if (commandNameTag) {
        const commandName = commandNameTag.
```

### utils/sessionStorage.ts
```
continue
        } else {
          // Otherwise, for custom commands, then keep it only if it has
          // arguments (e.
```

### utils/sessionStorage.ts
```
continue
          }
          // Return clean formatted command instead of raw XML
          return `
```

### utils/sessionStorage.ts
```
continue
      }

      return textContent
    }
  }
  return undefined
}

export function removeExtraFields(
  transcript: TranscriptMessage[],
): SerializedMessage[] {
  return transcript.
```

### utils/sessionStorage.ts
```
resume loads
      // the full pre-compact history.
```

### utils/sessionStorage.ts
```
resume → immediate autocompact spiral.
```

### utils/sessionStorage.ts
```
continue
      messages.
```

### utils/sessionStorage.ts
```
resume immediately PTLs (adamr-20260320-165831: 397K displayed → 1.
```

### utils/sessionStorage.ts
```
resume loads their pre-snip history (the pre-fix behavior).
```

### utils/sessionStorage.ts
```
continue
    for (const uuid of removedUuids) toDelete.
```

### utils/sessionStorage.ts
```
pick up null (chain-root
  // behavior — same as if compact truncated there, which it did).
```

### utils/sessionStorage.ts
```
continue
    deletedParent.
```

### utils/sessionStorage.ts
```
continue
    messages.
```

### utils/sessionStorage.ts
```
continue
    const t = Date.
```

### utils/sessionStorage.ts
```
continued the write chain) AND a TR child.
```

### utils/sessionStorage.ts
```
continue
    processedGroups.
```

### utils/sessionStorage.ts
```
continue
      for (const tr of trs) {
        if (!seen.
```

### utils/sessionStorage.ts
```
continue

    // Timestamp sort keeps content-block / completion order; stable-sort
    // preserves JSONL write order on ties.
```

### utils/sessionStorage.ts
```
resume_consistency_delta for BigQuery monitoring of
 * write→load round-trip drift — the class of bugs where snip/compact/
 * parallel-TR operations mutate in-memory but the parentUuid walk on disk
 * reconstructs a different set (adamr-20260320-165831: 397K displayed →
 * 1.
```

### utils/sessionStorage.ts
```
resume loaded MORE than in-session (the usual failure mode)
 * delta < 0: resume loaded FEWER (chain truncation — #22453 class)
 * delta = 0: round-trip consistent
 *
 * Called from loadConversationForResume — fires once per resume, not on
 * /share or log-listing chain rebuilds.
```

### utils/sessionStorage.ts
```
ResumeConsistency(chain: Message[]): void {
  for (let i = chain.
```

### utils/sessionStorage.ts
```
continue
    const expected = m.
```

### utils/sessionStorage.ts
```
resume_consistency_delta'
```

### utils/sessionStorage.ts
```
continue
    }
    const { snapshot, isSnapshotUpdate } = snapshotMessage
    const existingIndex = isSnapshotUpdate
      ? indexByMessageId.
```

### utils/sessionStorage.ts
```
resume bug where a stale AI title
 *   overwrites a mid-session user rename.
```

### utils/sessionStorage.ts
```
resumed
  // session'
```

### utils/sessionStorage.ts
```
resume is unaffected.
```

### utils/sessionStorage.ts
```
resume knows not to cd back into it.
```

### utils/sessionStorage.ts
```
continue
    }

    // Fast path: most chunks contain zero metadata markers.
```

### utils/sessionStorage.ts
```
continue
      }
      if (isTranscriptMessage(entry)) {
        if (entry.
```

### utils/sessionStorage.ts
```
resume); main-thread
        // decisions key by sessionId (/resume).
```

### utils/sessionStorage.ts
```
continues the conversation).
```

### utils/sessionStorage.ts
```
resume) skips a second full file load.
```

### utils/sessionStorage.ts
```
continue from */
  nextIndex: number
}

export async function loadSameRepoMessageLogs(
  worktreePaths: string[],
  limit?: number,
  initialEnrichCount: number = INITIAL_ENRICH_COUNT,
): Promise<LogOption[]> {
  const result = await loadSameRepoMessageLogsProgressive(
    worktreePaths,
    limit,
    initialEnrichCount,
  )
  return result.
```

### utils/sessionStorage.ts
```
resume: loading sessions for cwd=${getOriginalCwd()}, worktrees=[${worktreePaths.
```

### utils/sessionStorage.ts
```
resume: found ${allStatLogs.
```

### utils/sessionStorage.ts
```
continue
    const dirName = caseInsensitive ? dirent.
```

### utils/sessionStorage.ts
```
continue

    for (const { path: wtPath, prefix } of indexed) {
      if (dirName === prefix || dirName.
```

### utils/sessionStorage.ts
```
resume the model then sees a coherent native-tool-call history (assistant
 * called Bash, got result, called Read, got result) without the REPL wrapper.
```

### utils/sessionStorage.ts
```
continue
    const sessionId = validateUuid(basename(dirent.
```

### utils/sessionStorage.ts
```
continue
    candidates.
```

### utils/sessionStorage.ts
```
continue

    // Append trailing messages that are children of the leaf
    const trailingMessages = childrenByParent.
```

### utils/sessionStorage.ts
```
continue
    }
    if (line.
```

### utils/sessionStorage.ts
```
continue
    if (line.
```

### utils/sessionStorage.ts
```
continue

    try {
      const entry = jsonParse(line) as Record<string, unknown>
      if (entry.
```

### utils/sessionStorage.ts
```
continue

      const message = entry.
```

### utils/sessionStorage.ts
```
continue

      const content = message.
```

### utils/sessionStorage.ts
```
continue

        let result = text.
```

### utils/sessionStorage.ts
```
continue
          }
          // Custom command with meaningful args — use clean display
          return commandArgs
            ? `
```

### utils/sessionStorage.ts
```
continue
        }
        if (result.
```

### utils/sessionStorage.ts
```
continue
    }
  }
  // Session started with a slash command but had no subsequent real message —
  // use the clean command name so the session still appears in the resume picker
  if (firstCommandFallback) return firstCommandFallback
  // Proactive sessions have only tick messages — give them a synthetic prompt
  // so they'
```

### utils/sessionStorage.ts
```
continue

    const valueStart = idx + pattern.
```

### utils/sessionStorage.ts
```
continue
      }
      if (text[i] === '
```

### utils/sessionStorage.ts
```
continue
    const existing = deduped.
```

### utils/sessionStorage.ts
```
resume after crashes or large-context sessions.
```

### utils/sessionStorage.ts
```
resume: isSidechain=true`
```

### utils/sessionStorage.ts
```
resume: teamName=${enriched.
```

### utils/sessionStorage.ts
```
resume: enriched ${scanned} sessions, ${filtered} filtered out, ${result.
```

### utils/sessionStoragePortable.ts
```
continue

    const valueStart = idx + pattern.
```

### utils/sessionStoragePortable.ts
```
continue
      }
      if (text[i] === '
```

### utils/sessionStoragePortable.ts
```
continue
        }
        if (text[i] === '
```

### utils/sessionStoragePortable.ts
```
continue
    if (line.
```

### utils/sessionStoragePortable.ts
```
continue
    if (line.
```

### utils/sessionStoragePortable.ts
```
continue
    if (
      line.
```

### utils/sessionStoragePortable.ts
```
continue

    try {
      const entry = JSON.
```

### utils/sessionStoragePortable.ts
```
continue

      const message = entry.
```

### utils/sessionStoragePortable.ts
```
continue

      const content = message.
```

### utils/sessionStoragePortable.ts
```
continue

        // Skip slash-command messages but remember first as fallback
        const cmdMatch = COMMAND_NAME_RE.
```

### utils/sessionStoragePortable.ts
```
continue
        }

        // Format bash input with ! prefix before the generic XML skip
        const bashMatch = /<bash-input>([\s\S]*?)<\/bash-input>/.
```

### utils/sessionStoragePortable.ts
```
continue

        if (result.
```

### utils/sessionStoragePortable.ts
```
continue
    }
  }
  if (commandFallback) return commandFallback
  return '
```

### utils/sessionStoragePortable.ts
```
continue searching past
 * a truncated copy to find a valid one in a sibling directory.
```

### utils/sessionStoragePortable.ts
```
continue
      const wtProjectDir = await findProjectDir(wt)
      if (!wtProjectDir) continue
      const filePath = join(wtProjectDir, fileName)
      try {
        const s = await stat(filePath)
        if (s.
```

### utils/sessionStoragePortable.ts
```
resume load path.
```

### utils/sessionTitle.ts
```
continue
    const content = msg.
```

### utils/sessionUrl.ts
```
resume identifier which can be either:
 * - A URL containing session ID (e.
```

### utils/sessionUrl.ts
```
resumeIdentifier - The URL or session ID to parse
 * @returns Parsed session information or null if invalid
 */
export function parseSessionIdentifier(
  resumeIdentifier: string,
): ParsedSessionUrl | null {
  // Check for JSONL file path before URL parsing, since Windows absolute
  // paths (e.
```

### utils/sessionUrl.ts
```
resumeIdentifier.
```

### utils/sessionUrl.ts
```
resumeIdentifier,
      isJsonlFile: true,
    }
  }

  // Check if it'
```

### utils/sessionUrl.ts
```
resumeIdentifier)) {
    return {
      sessionId: resumeIdentifier as UUID,
      ingressUrl: null,
      isUrl: false,
      jsonlFile: null,
      isJsonlFile: false,
    }
  }

  // Check if it'
```

### utils/sessionUrl.ts
```
resumeIdentifier)

    // Use the entire URL as the ingress URL
    // Always generate a random session ID
    return {
      sessionId: randomUUID() as UUID,
      ingressUrl: url.
```

### utils/settings/changeDetector.ts
```
continue
    }
    const path = getSettingsFilePathForSource(source)
    if (!path) {
      continue
    }

    const dir = platformPath.
```

### utils/settings/changeDetector.ts
```
pick up new values
          setMdmSettingsCache(current, currentHkcu)
          logForDebugging('
```

### utils/settings/mdm/settings.ts
```
continue
      }
      try {
        const content = readFileSync(join(dropInDir, d.
```

### utils/settings/permissionValidation.ts
```
continues to work for backwards compatibility:
    // - "
```

### utils/settings/settings.ts
```
continue
      }

      const filePath = getSettingsFilePathForSource(source)
      if (filePath) {
        const resolvedPath = resolve(filePath)

        // Skip if we'
```

### utils/settings/settings.ts
```
continue
      const result = schema.
```

### utils/settings/settings.ts
```
continue
    }

    const filePath = getSettingsFilePathForSource(source)
    if (!filePath) {
      continue
    }

    try {
      const { resolvedPath } = safeResolvePath(getFsImplementation(), filePath)
      const content = readFileSync(resolvedPath)
      if (!content.
```

### utils/settings/settings.ts
```
continue
      }

      const rawData = safeParseJSON(content, false)
      if (rawData && typeof rawData === '
```

### utils/settings/validation.ts
```
continue

    perms[key] = rules.
```

### utils/shell/readOnlyCommandValidation.ts
```
continue checking flags after `
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
        // First non-flag positional: check if it'
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
        }
        // `
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
        }
        if (!seenDashDash && token.
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
        }
        // `
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
        }
        if (!seenDashDash && token.
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
    // For flag tokens, extract the VALUE after `
```

### utils/shell/readOnlyCommandValidation.ts
```
continue // flag without inline value, nothing to inspect
      value = token.
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
    }
    // Skip values that are clearly not repo specs (no `
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
    }
    // URL schemes: https://, http://, git://, ssh://
    if (value.
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
    }

    // Special handling for xargs: once we find the target command, stop validating flags
    if (
      options?.
```

### utils/shell/readOnlyCommandValidation.ts
```
continue processing subsequent tokens as flags.
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
    }

    if (token.
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
        }

        // Handle flags with directly attached numeric arguments (e.
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
              } else {
                return false // Invalid attached value
              }
            }
          }
        }

        // Handle combined single-letter flags like -nr
        // SECURITY: We must NOT allow any bundled flag that takes an argument.
```

### utils/shell/readOnlyCommandValidation.ts
```
continue
        } else {
          return false // Unknown flag
        }
      }

      // Validate flag arguments
      if (flagArgType === '
```

### utils/shell/specPrefix.ts
```
continue
    if (arg.
```

### utils/shell/specPrefix.ts
```
continue
    }
    if (!spec?.
```

### utils/shell/specPrefix.ts
```
continue
        }
      }

      // For commands with subcommands, skip global flags to find the subcommand
      if (hasSubcommands && !foundSubcommand) {
        if (flagTakesArg(arg, args[i + 1], spec)) i++
        continue
      }
      break // Stop at flags (original behavior)
    }

    if (await shouldStopAtArg(arg, args.
```

### utils/shell/specPrefix.ts
```
continue
      const option = spec.
```

### utils/shellConfig.ts
```
continue

    for (const line of lines) {
      if (CLAUDE_ALIAS_REGEX.
```

### utils/sideQuestion.ts
```
continues working independently in the background
- You share the conversation context but are a completely separate instance
- Do NOT reference being interrupted or what you were "
```

### utils/sliceAnsi.ts
```
continue
        include = true
        // Reduce and filter to only active start codes
        activeCodes = filterStartCodes(reduceAnsiCodes(activeCodes))
        result = ansiCodesToString(activeCodes)
      }

      if (include) {
        result += token.
```

### utils/slowOperations.ts
```
continue
    const m = line.
```

### utils/stats.ts
```
resumed today gets a new mtime write but old start date).
```

### utils/stats.ts
```
continue
      if (error || !entries) {
        logForDebugging(
          `
```

### utils/stats.ts
```
continue
      }

      const sessionId = basename(sessionFile, '
```

### utils/stats.ts
```
continue

      // Subagent transcripts mark all messages as sidechain.
```

### utils/stats.ts
```
continue

      const firstMessage = mainMessages[0]!
      const lastMessage = mainMessages.
```

### utils/stats.ts
```
continue
      }

      const dateKey = toDateString(firstTimestamp)

      // Apply date filters
      if (fromDate && isDateBefore(dateKey, fromDate)) continue
      if (toDate && isDateBefore(toDate, dateKey)) continue

      // Track daily activity (use first message date as session date)
      const existing = dailyActivityMap.
```

### utils/stats.ts
```
continue
            }

            if (!modelUsageAgg[model]) {
              modelUsageAgg[model] = {
                inputTokens: 0,
                outputTokens: 0,
                cacheReadInputTokens: 0,
                cacheCreationInputTokens: 0,
                webSearchRequests: 0,
                costUSD: 0,
                contextWindow: 0,
                maxOutputTokens: 0,
         
```

### utils/stats.ts
```
continue
    const content = m.
```

### utils/stats.ts
```
continue
    for (const block of content) {
      if (
        block.
```

### utils/stats.ts
```
continue
      }
      const match = SHOT_COUNT_REGEX.
```

### utils/stats.ts
```
Resume), which would cause resumed sessions to
 * be miscategorised as old and silently dropped from stats.
```

### utils/stats.ts
```
continue
        let entry: {
          type?: unknown
          timestamp?: unknown
          isSidechain?: unknown
        }
        try {
          entry = jsonParse(line)
        } catch {
          continue
        }
        if (typeof entry.
```

### utils/stats.ts
```
continue
        if (!TRANSCRIPT_MESSAGE_TYPES.
```

### utils/stats.ts
```
continue
        if (entry.
```

### utils/stats.ts
```
continue
        if (typeof entry.
```

### utils/suggestions/commandSuggestions.ts
```
continue
    }
    const name = getCommandName(suggestion.
```

### utils/suggestions/slackChannelSuggestions.ts
```
continue
    const start = m.
```

### utils/swarm/backends/ITermBackend.ts
```
continue shrinks teammateSessionIds by 1;
      // when empty → firstPaneUsed resets → next iteration has no target → throws.
```

### utils/swarm/backends/ITermBackend.ts
```
continue
            }
            // Target is alive or we can'
```

### utils/swarm/backends/InProcessBackend.ts
```
continues working).
```

### utils/swarm/inProcessRunner.ts
```
Continue polling even if one read fails
    }

    // Check the team'
```

### utils/swarm/inProcessRunner.ts
```
continue to idle state
      if (workWasAborted) {
        logForDebugging(
          `
```

### utils/swarm/permissionSync.ts
```
continues execution
 */

import { mkdir, readdir, readFile, unlink, writeFile } from '
```

### utils/swarm/reconnection.ts
```
Resumed sessions: Initialize from teamName/agentName stored in the transcript
 */

import type { AppState } from '
```

### utils/task/TaskOutput.ts
```
continue
      }
      void tailFile(entry.
```

### utils/task/diskOutput.ts
```
resumes after preload'
```

### utils/task/diskOutput.ts
```
continue
      }

      break
    }
  }

  #writeAllChunks(): Promise<void> {
    // This code is extremely precise.
```

### utils/task/framework.ts
```
resumeAgentBackground
    // replaces the task; user'
```

### utils/task/framework.ts
```
resume) — not a new start.
```

### utils/task/framework.ts
```
continue
        case '
```

### utils/task/framework.ts
```
continue
        case '
```

### utils/task/framework.ts
```
resume may have
      // replaced the task during the generateTaskAttachments await)
      if (!fresh || !isTerminalTaskStatus(fresh.
```

### utils/task/framework.ts
```
continue
      }
      if ('
```

### utils/task/framework.ts
```
continue
      }
      delete newTasks[id]
      changed = true
    }
    return changed ? { .
```

### utils/tasks.ts
```
continue
    }
    const taskId = parseInt(file.
```

### utils/teamDiscovery.ts
```
continue
    }

    // Read isActive from config, defaulting to true (active) if undefined
    const isActive = member.
```

### utils/teammateContext.ts
```
continue running when the leader'
```

### utils/teammateMailbox.ts
```
continue

    // Stop at wake-up boundary: a user prompt (string content), not tool results (array content)
    if (msg.
```

### utils/teammateMailbox.ts
```
continue
    for (const block of msg.
```

### utils/telemetry/instrumentation.ts
```
continue even if flush fails
  }
}

function parseOtelHeadersEnvVar(): Record<string, string> {
  const headers: Record<string, string> = {}
  const envHeaders = process.
```

### utils/telemetry/skillLoadedEvent.ts
```
continue

    logEvent('
```

### utils/teleport/environmentSelection.ts
```
continue
        }
        const sourceSettings = getSettingsForSource(source)
        if (
          sourceSettings?.
```

### utils/teleport.tsx
```
resume
 * @returns SystemMessage indicating session was resumed from another machine
 */
function createTeleportResumeSystemMessage(branchError: Error | null): SystemMessage {
  if (branchError === null) {
    return createSystemMessage('
```

### utils/teleport.tsx
```
resumed without branch: ${formattedError}`
```

### utils/teleport.tsx
```
resume
 * @returns User message indicating session was resumed from another machine
 */
function createTeleportResumeUserMessage() {
  return createUserMessage({
    content: `
```

### utils/teleport.tsx
```
continued from another machine.
```

### utils/teleport.tsx
```
resume, removing incomplete tool_use blocks
 * and adding teleport notice messages
 * @param messages The conversation messages
 * @param error Optional error from branch checkout
 * @returns Processed messages ready for resume
 */
export function processMessagesForTeleportResume(messages: Message[], error: Error | null): Message[] {
  // Shared logic with resume for handling interruped session tr
```

### utils/teleport.tsx
```
ResumeUserMessage(), createTeleportResumeSystemMessage(error)];
  return messagesWithTeleportNotice;
}

/**
 * Checks out the specified branch for a teleported session
 * @param branch Optional branch to checkout
 * @returns The current branch name and any error that occurred
 */
export async function checkOutTeleportedSessionBranch(branch?: string): Promise<{
  branchName: string;
  branchError: 
```

### utils/teleport.tsx
```
resume
 * @param onProgress Optional callback for progress updates
 * @returns The raw session log and branch name
 */
export async function teleportResumeCodeSession(sessionId: string, onProgress?: TeleportProgressCallback): Promise<TeleportRemoteResponse> {
  if (!isPolicyAllowed('
```

### utils/teleport.tsx
```
resume_session_id_catch'
```

### utils/teleport.tsx
```
continue;
        }
        if ('
```

### utils/textHighlighting.ts
```
continue

    const overlaps = usedRanges.
```

### utils/tmuxSocket.ts
```
resume session list.
```

### utils/tokenBudget.ts
```
Keep working \u2014 do not summarize.
```

### utils/toolResultStorage.ts
```
resume, or compact are never looked up (tool_use_ids are
 * UUIDs) so they'
```

### utils/toolResultStorage.ts
```
resumeAgentBackground threads one reconstructed
 * from sidechain records.
```

### utils/toolResultStorage.ts
```
resume so code changes to the preview template, size formatting,
 * or path layout can'
```

### utils/toolResultStorage.ts
```
continue
    const content = message.
```

### utils/toolResultStorage.ts
```
continue
    for (const block of content) {
      if (block.
```

### utils/toolResultStorage.ts
```
resume reconstruction.
```

### utils/toolResultStorage.ts
```
continue
    }

    // Tools with maxResultSizeChars: Infinity (Read) — never persist.
```

### utils/toolResultStorage.ts
```
continue
    messagesOverBudget++
    toPersist.
```

### utils/toolResultStorage.ts
```
continue
    replacedSize += candidate.
```

### utils/toolResultStorage.ts
```
resume (repl_main_thread*, agent:*); ephemeral runForkedAgent callers
 * (agentSummary, sessionMemory, /btw, compact) pass undefined.
```

### utils/toolResultStorage.ts
```
resume so the budget makes the same choices it
 * made in the original session (prompt cache stability).
```

### utils/toolResultStorage.ts
```
resume the sidechain has
 *     the original content but no record, so records alone would classify
 *     it as frozen.
```

### utils/toolResultStorage.ts
```
resumes (parent IDs aren'
```

### utils/toolResultStorage.ts
```
resume variant: encapsulates the feature-flag gate + parent
 * gap-fill so both AgentTool.
```

### utils/toolResultStorage.ts
```
resumeAgentBackground share one
 * implementation.
```

### utils/toolResultStorage.ts
```
Resume(
  parentState: ContentReplacementState | undefined,
  resumedMessages: Message[],
  sidechainRecords: ContentReplacementRecord[],
): ContentReplacementState | undefined {
  if (!parentState) return undefined
  return reconstructContentReplacementState(
    resumedMessages,
    sidechainRecords,
    parentState.
```

### utils/toolSearch.ts
```
continue
    }

    // Only user messages contain tool_result blocks (responses to tool_use)
    if (msg.
```

### utils/toolSearch.ts
```
continue

    const content = msg.
```

### utils/toolSearch.ts
```
continue

    for (const block of content) {
      // tool_reference blocks only appear inside tool_result content, specifically
      // in results from ToolSearchTool.
```

### utils/toolSearch.ts
```
continue
    attachmentCount++
    attachmentTypesSeen.
```

### utils/toolSearch.ts
```
continue
    dtdCount++
    for (const n of msg.
```

### utils/toolSearch.ts
```
continue
    if (!poolNames.
```

### utils/transcriptSearch.ts
```
resumes (memory reminders between prompt lines).
```

### utils/ultraplan/ccrSession.ts
```
continue
          const tu = block as ToolUseBlock
          if (tu.
```

### utils/ultraplan/ccrSession.ts
```
continue
        for (const block of content) {
          if (block.
```

### utils/ultraplan/ccrSession.ts
```
continue
        const tr = this.
```

### utils/ultraplan/ccrSession.ts
```
continue
    }

    let result: ScanResult
    try {
      result = scanner.
```

### utils/ultraplan/keyword.ts
```
continue
      }
      if (ch !== OPEN_TO_CLOSE[openQuote]) continue
      if (openQuote === "
```

### utils/ultraplan/keyword.ts
```
continue
      quotedRanges.
```

### utils/ultraplan/keyword.ts
```
continue
    const start = match.
```

### utils/ultraplan/keyword.ts
```
continue
    const before = text[start - 1]
    const after = text[end]
    if (before === '
```

### utils/ultraplan/keyword.ts
```
continue
    if (after === '
```

### utils/ultraplan/keyword.ts
```
continue
    if (after === '
```

### utils/ultraplan/keyword.ts
```
continue
    positions.
```

### utils/windowsPaths.ts
```
continue
      }

      // Return the first valid path that'
```

### utils/worktree.ts
```
continue
    }

    const sourcePath = join(repoRootPath, dir)
    const destPath = join(worktreePath, dir)

    try {
      await symlink(sourcePath, destPath, '
```

### utils/worktree.ts
```
resumes it if it already exists.
```

### utils/worktree.ts
```
resume path: if the worktree already exists skip fetch and creation.
```

### utils/worktree.ts
```
resume (rev-parse HEAD) would succeed and present a broken worktree
    // as "
```

### utils/worktree.ts
```
continue working there by running: cd ${worktreePath}`
```

### utils/worktree.ts
```
resume path is read-only and leaves the original
    // creation-time mtime intact, which can be past the 30-day cutoff.
```

### utils/worktree.ts
```
continue
    }

    const worktreePath = join(dir, slug)
    if (currentPath === worktreePath) {
      continue
    }

    let mtimeMs: number
    try {
      mtimeMs = (await stat(worktreePath)).
```

### utils/worktree.ts
```
continue
    }
    if (mtimeMs >= cutoffMs) {
      continue
    }

    // Both checks must succeed with empty output.
```

### utils/worktree.ts
```
continue
    }
    if (unpushed.
```

### utils/worktree.ts
```
continue
    }

    if (
      await removeAgentWorktree(worktreePath, worktreeBranchName(slug), gitRoot)
    ) {
      removed++
    }
  }

  if (removed > 0) {
    await execFileNoThrowWithCwd(gitExe(), ['
```

### utils/worktree.ts
```
continue
    if (arg === '
```

### utils/worktree.ts
```
resume worktree
    try {
      const result = await getOrCreateWorktree(
        repoRoot,
        worktreeName,
        prNumber !== null ? { prNumber } : undefined,
      )
      if (!result.
```

### utils/worktree.ts
```
continue
    if (arg === '
```

### utils/worktree.ts
```
continue
    if (arg === '
```

### utils/worktree.ts
```
continue
    }
    if (arg.
```

### utils/worktree.ts
```
continue
    newArgs.
```


## 6. Service Directory Structure

### services/AgentSummary/ (1 files)
  - agentSummary.ts (6KB)

### services/MagicDocs/ (2 files)
  - magicDocs.ts (8KB)
  - prompts.ts (5KB)

### services/PromptSuggestion/ (2 files)
  - promptSuggestion.ts (17KB)
  - speculation.ts (30KB)

### services/SessionMemory/ (3 files)
  - prompts.ts (12KB)
  - sessionMemory.ts (16KB)
  - sessionMemoryUtils.ts (6KB)

### services/analytics/ (9 files)
  - config.ts (1KB)
  - datadog.ts (9KB)
  - firstPartyEventLogger.ts (14KB)
  - firstPartyEventLoggingExporter.ts (26KB)
  - growthbook.ts (40KB)
  - index.ts (5KB)
  - metadata.ts (32KB)
  - sink.ts (3KB)
  - sinkKillswitch.ts (1KB)

### services/api/ (20 files)
  - adminRequests.ts (3KB)
  - bootstrap.ts (5KB)
  - claude.ts (123KB)
  - client.ts (16KB)
  - dumpPrompts.ts (7KB)
  - emptyUsage.ts (1KB)
  - errorUtils.ts (8KB)
  - errors.ts (41KB)
  - filesApi.ts (21KB)
  - firstTokenDate.ts (2KB)
  - grove.ts (11KB)
  - logging.ts (24KB)
  - metricsOptOut.ts (5KB)
  - overageCreditGrant.ts (5KB)
  - promptCacheBreakDetection.ts (26KB)
  - referral.ts (8KB)
  - sessionIngress.ts (17KB)
  - ultrareviewQuota.ts (1KB)
  - usage.ts (2KB)
  - withRetry.ts (28KB)

### services/autoDream/ (4 files)
  - autoDream.ts (11KB)
  - config.ts (1KB)
  - consolidationLock.ts (4KB)
  - consolidationPrompt.ts (3KB)

### services/compact/ (11 files)
  - apiMicrocompact.ts (5KB)
  - autoCompact.ts (13KB)
  - compact.ts (59KB)
  - compactWarningHook.ts (1KB)
  - compactWarningState.ts (1KB)
  - grouping.ts (3KB)
  - microCompact.ts (19KB)
  - postCompactCleanup.ts (4KB)
  - prompt.ts (16KB)
  - sessionMemoryCompact.ts (21KB)
  - timeBasedMCConfig.ts (2KB)

### services/extractMemories/ (2 files)
  - extractMemories.ts (21KB)
  - prompts.ts (7KB)

### services/lsp/ (7 files)
  - LSPClient.ts (14KB)
  - LSPDiagnosticRegistry.ts (12KB)
  - LSPServerInstance.ts (16KB)
  - LSPServerManager.ts (13KB)
  - config.ts (3KB)
  - manager.ts (10KB)
  - passiveFeedback.ts (11KB)

### services/mcp/ (23 files)
  - InProcessTransport.ts (2KB)
  - MCPConnectionManager.tsx (8KB)
  - SdkControlTransport.ts (4KB)
  - auth.ts (87KB)
  - channelAllowlist.ts (3KB)
  - channelNotification.ts (12KB)
  - channelPermissions.ts (9KB)
  - claudeai.ts (6KB)
  - client.ts (116KB)
  - config.ts (50KB)
  - elicitationHandler.ts (10KB)
  - envExpansion.ts (1KB)
  - headersHelper.ts (5KB)
  - mcpStringUtils.ts (4KB)
  - normalization.ts (1KB)
  - oauthPort.ts (2KB)
  - officialRegistry.ts (2KB)
  - types.ts (7KB)
  - useManageMCPConnections.ts (44KB)
  - utils.ts (18KB)
  - vscodeSdkMcp.ts (4KB)
  - xaa.ts (18KB)
  - xaaIdpLogin.ts (16KB)

### services/oauth/ (5 files)
  - auth-code-listener.ts (6KB)
  - client.ts (18KB)
  - crypto.ts (1KB)
  - getOauthProfile.ts (2KB)
  - index.ts (6KB)

### services/plugins/ (3 files)
  - PluginInstallationManager.ts (6KB)
  - pluginCliCommands.ts (11KB)
  - pluginOperations.ts (35KB)

### services/policyLimits/ (2 files)
  - index.ts (18KB)
  - types.ts (1KB)

### services/remoteManagedSettings/ (5 files)
  - index.ts (20KB)
  - securityCheck.tsx (10KB)
  - syncCache.ts (4KB)
  - syncCacheState.ts (4KB)
  - types.ts (1KB)

### services/settingsSync/ (2 files)
  - index.ts (18KB)
  - types.ts (2KB)

### services/teamMemorySync/ (5 files)
  - index.ts (43KB)
  - secretScanner.ts (9KB)
  - teamMemSecretGuard.ts (2KB)
  - types.ts (5KB)
  - watcher.ts (13KB)

### services/tips/ (3 files)
  - tipHistory.ts (1KB)
  - tipRegistry.ts (23KB)
  - tipScheduler.ts (2KB)

### services/toolUseSummary/ (1 files)
  - toolUseSummaryGenerator.ts (3KB)

### services/tools/ (4 files)
  - StreamingToolExecutor.ts (17KB)
  - toolExecution.ts (59KB)
  - toolHooks.ts (22KB)
  - toolOrchestration.ts (5KB)


## 7. Cross-Module Coupling (files importing from 5+ different service dirs)

- **cli/print.ts** imports from 9 services: settingsSync, remoteManagedSettings, analytics, api, mcp, policyLimits, PromptSuggestion, oauth, claudeAiLimits.js
- **main.tsx** imports from 11 services: analytics, api, mcp, policyLimits, remoteManagedSettings, claudeAiLimits.js, plugins, internalLogging.js, tips, lsp, PromptSuggestion
- **query.ts** imports from 5 services: api, compact, analytics, toolUseSummary, tools
- **screens/REPL.tsx** imports from 8 services: notifier.js, preventSleep.js, analytics, mcp, compact, diagnosticTracking.js, PromptSuggestion, tips
- **tools/FileEditTool/FileEditTool.ts** imports from 5 services: analytics, diagnosticTracking.js, lsp, mcp, teamMemorySync
- **tools/FileWriteTool/FileWriteTool.ts** imports from 5 services: analytics, diagnosticTracking.js, lsp, mcp, teamMemorySync
- **utils/attachments.ts** imports from 6 services: analytics, diagnosticTracking.js, skillSearch, mcp, lsp, compact
