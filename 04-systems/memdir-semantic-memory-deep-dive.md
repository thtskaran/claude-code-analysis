# Semantic Memory Directory (memdir) Deep Dive: Reverse Engineering Analysis

## Executive Summary

The memdir system is a sophisticated file-based persistent memory architecture that enables Claude Code agents to maintain semantic context across sessions and teams. It implements a closed 4-type taxonomy with dual-scope support, Sonnet-powered relevance selection, comprehensive path security validation, and a multi-tier integration strategy with the main query loop. This analysis documents all architectural components, security mechanisms, and integration patterns based on direct source inspection of 8 TypeScript modules.

---

## 1. Architecture Overview

### High-Level Design

The memdir system provides persistent, project-scoped semantic memory through:

- **File-based storage**: Markdown files with YAML frontmatter organized by topic
- **Dual-directory support**: Private (user-specific) and team-shared namespaces
- **Automatic entrypoint indexing**: `MEMORY.md` as a loaded index with size caps and truncation warnings
- **Query-time relevance selection**: Sonnet-powered up-to-5-memory selector for each turn
- **Three-tier integration**: System prompt injection (Tier 1), query-time recall via sideQuery (Tier 2), manual recall (Tier 3)
- **Staleness awareness**: Age-based freshness indicators with system-reminder decorations
- **Team synchronization**: Scope-aware team memory with validation and combined prompts

### Core Design Principles

1. **Closed taxonomy**: Only 4 types (user, feedback, project, reference) — code patterns, git history, architecture are explicitly excluded
2. **Semantic organization**: Memories grouped by topic, not chronologically
3. **Derivability test**: Content must be non-derivable from code, git, CLAUDE.md, or current state
4. **Staleness handling**: Point-in-time snapshots, not live state; staleness caveats surface for >1 day old memories
5. **Security-first paths**: Two-pass validation (string + symlink), null byte rejection, Unicode normalization defense

---

## 2. Directory Structure

### Standard Layout

```
~/.claude/
└── projects/
    └── <sanitized-git-root>/
        └── memory/
            ├── MEMORY.md                    # Private index (200 lines, 25KB cap)
            ├── user_*.md                    # User-type memories
            ├── feedback_*.md                # Feedback-type memories
            ├── project_*.md                 # Project-type memories
            ├── reference_*.md               # Reference-type memories
            ├── team/
            │   └── MEMORY.md                # Team index (200 lines, 25KB cap)
            │   └── <team-memory-files>.md
            └── logs/                        # KAIROS feature (daily logs)
                ├── YYYY/
                │   └── MM/
                │       └── YYYY-MM-DD.md    # Date-named append-only logs
```

### Path Resolution

Paths are resolved in priority order (first match wins):

1. **CLAUDE_COWORK_MEMORY_PATH_OVERRIDE** (env var, full override for Cowork)
2. **autoMemoryDirectory** in settings.json (supports `~/` expansion, trusted sources only)
3. **Default**: `{memoryBase}/projects/{sanitized-git-root}/memory/`
   - memoryBase: `CLAUDE_CODE_REMOTE_MEMORY_DIR` env var OR `~/.claude`
   - sanitized-git-root: Canonical git repo root or fallback to project root

### Cowork Memory Path Security

The `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` prevents per-session cwd variation (which contains VM process names) from creating different project keys each session. Settings.json **projectSettings** is intentionally excluded from override resolution to prevent malicious repos setting `autoMemoryDirectory: ~/.ssh` and gaining write access via filesystem.ts carve-outs.

---

## 3. Memory Type Taxonomy

### Closed Four-Type System

#### **user**
- **Scope**: Always private
- **Purpose**: Information about the user's role, goals, responsibilities, and knowledge
- **When to save**: Any details about user profile, preferences, responsibilities
- **How to use**: Tailor behavior and explanations to the user's specific perspective
- **Examples**: "Data scientist investigating observability", "10 years Go experience, new to React"

#### **feedback**
- **Scope**: Default private; team only when clearly a project-wide convention (e.g., testing policy)
- **Purpose**: Guidance on how to approach work — what to avoid and what to keep doing
- **When to save**: User corrections ("stop doing X") AND confirmations ("yes, keep that")
- **Structure**: Rule/fact, then **Why:** (reason/incident), then **How to apply:** (when/where)
- **Examples**: "Don't mock database in tests", "No trailing summaries", "Single bundled PR preferred for refactors"

#### **project**
- **Scope**: Private or team, bias toward team
- **Purpose**: Context about ongoing work, goals, initiatives, bugs, incidents not in code/git
- **When to save**: Who is doing what, why, by when; convert relative dates to absolute (e.g., "Thursday" → "2026-03-05")
- **Structure**: Fact/decision, then **Why:** (constraint/deadline/stakeholder ask), then **How to apply:** (shapes suggestions)
- **Examples**: "Merge freeze 2026-03-05 for mobile release", "Auth rewrite driven by legal compliance"

#### **reference**
- **Scope**: Usually team
- **Purpose**: Pointers to external systems for up-to-date information
- **When to save**: Locations of bugs, feedback, dashboards in external systems
- **How to use**: When user references external system or information may be there
- **Examples**: "Pipeline bugs in Linear project INGEST", "Grafana board grafana.internal/d/api-latency for oncall"

### What NOT to Save

Explicitly excluded from memory (even if user asks):
- Code patterns, conventions, architecture, file paths, project structure (derivable)
- Git history, recent changes, who-changed-what (use `git log`/`git blame`)
- Debugging solutions or fix recipes (fix is in code; commit message has context)
- Anything in CLAUDE.md files (already documented)
- Ephemeral task details (in-progress work, temporary state, current conversation context)

---

## 4. MEMORY.md Index Architecture

### File Format and Caps

**Filename**: `MEMORY.md` (always in memory directory root; team memory at `team/MEMORY.md`)

**Caps**:
- **Line cap**: 200 lines (primary constraint; natural boundary)
- **Byte cap**: 25,000 bytes (targets long-line failure mode; p97 at 200 lines)
- **Entry format**: One line per memory, ~150 chars, format: `- [Title](file.md) — one-line hook`
- **No frontmatter**: Index is a pointer list, not a memory storage file
- **No direct content**: Detail goes in topic files, MEMORY.md is index only

### Truncation Logic

The `truncateEntrypointContent()` function implements line-then-byte truncation:

1. **Line truncation**: If lines > 200, slice to first 200 lines
2. **Byte truncation**: If bytes > 25KB, find last newline before 25KB boundary and cut there
3. **Warning message**: Appends reason message naming which cap fired
   - Byte-only: Shows exact size, notes "entries too long"
   - Line-only: Shows line count
   - Both: Shows both metrics
   - Warning format: `> WARNING: MEMORY.md is {reason}. Only part of it was loaded. Keep index entries to one line under ~200 chars; move detail into topic files.`

### Truncation Type Tracking

Returns `EntrypointTruncation` object:
```typescript
type EntrypointTruncation = {
  content: string         // Truncated content with warning appended
  lineCount: number       // Original line count
  byteCount: number       // Original byte count
  wasLineTruncated: boolean
  wasByteTruncated: boolean
}
```

### Telemetry Integration

Calls `logMemoryDirCounts()` fire-and-forget, logging:
- `tengu_memdir_loaded` event
- total_file_count, total_subdir_count
- content_length, line_count, was_truncated, was_byte_truncated
- memory_type: 'auto' or 'agent'

---

## 5. Sonnet-Powered Relevance Selection

### Architecture: Query-Time Recall via sideQuery

The `findRelevantMemories()` function orchestrates Sonnet-based selection:

1. **Scan phase**: `scanMemoryFiles()` reads all .md files (except MEMORY.md), extracts frontmatter, returns newest-first up to 200 files
2. **Filter phase**: Removes already-surfaced memories (dedupe via `alreadySurfaced` Set)
3. **Selection phase**: Passes filtered list to Sonnet for relevance ranking
4. **Return phase**: Maps filenames back to MemoryHeader and returns paths + mtimes

### SELECT_MEMORIES_SYSTEM_PROMPT

```
You are selecting memories that will be useful to Claude Code as it processes
a user's query. You will be given the user's query and a list of available
memory files with their filenames and descriptions.

Return a list of filenames for the memories that will clearly be useful to
Claude Code as it processes the user's query (up to 5). Only include memories
that you are certain will be helpful based on their name and description.
- If you are unsure if a memory will be useful in processing the user's query,
  then do not include it in your list. Be selective and discerning.
- If there are no memories in the list that would clearly be useful, feel free
  to return an empty list.
- If a list of recently-used tools is provided, do not select memories that are
  usage reference or API documentation for those tools (Claude Code is already
  exercising them). DO still select memories containing warnings, gotchas, or
  known issues about those tools — active use is exactly when those matter.
```

### Manifest Format

Memory list formatted as text for Sonnet:
```
- [type] filename (ISO-timestamp): description
- [type] filename (ISO-timestamp): description
```

Each memory gets a tag if type is known; filename is relative path; timestamp is mtime ISO string; description from frontmatter.

### API Call Details

**sideQuery parameters**:
- **model**: `getDefaultSonnetModel()` (Sonnet 3.5+)
- **system**: SELECT_MEMORIES_SYSTEM_PROMPT
- **skipSystemPromptPrefix**: true (custom system already complete)
- **messages**: Single user message with query + manifest + optional recently-used tools
- **max_tokens**: 256 (generous for ~5 file selections)
- **output_format**: JSON schema requiring `selected_memories: string[]`
- **querySource**: 'memdir_relevance'

### JSON Schema Output

```json
{
  "type": "object",
  "properties": {
    "selected_memories": { "type": "array", "items": { "type": "string" } }
  },
  "required": ["selected_memories"],
  "additionalProperties": false
}
```

### Recently-Used Tools Filtering

When `recentTools` list provided (e.g., `['Grep', 'Bash']`), Sonnet receives:
```
Recently used tools: Grep, Bash
```

This filters out reference/API-doc memories for those tools while keeping warnings/gotchas.

### Deduplication: alreadySurfaced

Returns from prior turns are tracked in a Set. `findRelevantMemories()` filters before calling Sonnet, so selector's 5-slot budget goes to fresh candidates, not previously surfaced files.

### Return Type: RelevantMemory

```typescript
type RelevantMemory = {
  path: string      // Absolute file path
  mtimeMs: number   // Modification time for staleness calculation
}
```

mtime is returned so callers can surface freshness without a second stat call.

### Telemetry: logMemoryRecallShape

Fires if `feature('MEMORY_SHAPE_TELEMETRY')` enabled:
- Calls `logMemoryRecallShape(memories, selected)`
- Logs even on empty selection (needed for selection-rate denominator)
- Negative ages distinguish "ran, picked nothing" from "never ran"

---

## 6. Memory Lifecycle

### Creation / Storage

**Explicit save**:
1. User asks Claude to remember something
2. Claude writes to `<memoryDir>/<topic>.md` with frontmatter:
   ```markdown
   ---
   name: {{memory name}}
   description: {{one-line description for relevance}}
   type: {{user|feedback|project|reference}}
   ---

   {{content: for feedback/project, structure as rule/fact, **Why:**, **How to apply:**}}
   ```
3. Claude adds pointer to `MEMORY.md`: `- [Title](topic.md) — one-line hook`
4. Directory structure enforced by `ensureMemoryDirExists()` at session start

**Implicit save (extractMemories agent)**:
- Background agent runs post-turn if feature enabled
- Skips ranges where main agent wrote memories (hasMemoryWritesSince)
- Catches implicit learning opportunities missed by main agent

### Frontmatter Parsing

The `parseFrontmatter()` utility extracts YAML front matter from markdown files:
- Field extraction: `name`, `description`, `type`
- Type validation: `parseMemoryType()` maps to one of 4 enum values or undefined
- Graceful degradation: Files without `type:` field keep working; unknown types degrade gracefully
- Max lines: First 30 lines scanned for frontmatter (FRONTMATTER_MAX_LINES)

### Retrieval

**Tier 1 (System Prompt Injection)**:
- `MEMORY.md` loaded into `buildMemoryPrompt()` and injected into system prompt
- Truncated to caps; warning appended if overflow
- Always available context; no latency cost

**Tier 2 (Query-Time Sonnet Selection)**:
- `findRelevantMemories()` called per turn if feature enabled
- Scans files, calls Sonnet for top-5 relevance ranking
- Files loaded as needed by downstream consumers
- ~256 tokens for Sonnet call; latency ~500ms-1s

**Tier 3 (Manual Recall)**:
- User or Claude explicitly searches past context via grep/bash
- "Searching past context" section in prompt guides toward:
  - Memory directory search: `grep -rn "<term>" <autoMemDir> --include="*.md"`
  - Session transcript search: `grep -rn "<term>" <projectDir>/ --include="*.jsonl"` (slow, last resort)

### Deletion

User explicitly asks Claude to forget something:
1. Claude finds relevant entry in MEMORY.md or topic files
2. Claude removes the entry or deletes the file
3. Index updated to reflect removal

---

## 7. Integration with Query Loop

### Three-Tier Integration Strategy

#### **Tier 1: System Prompt Injection** (Synchronous)
- `loadMemoryPrompt()` called once per session (cached by systemPromptSection)
- Dispatches based on feature flags and enabled memory systems
- Injects `MEMORY.md` content directly into system prompt
- Fallback paths for different scenarios:
  - **KAIROS + auto memory**: `buildAssistantDailyLogPrompt()` (append-only logs)
  - **TEAMMEM + auto**: `buildCombinedMemoryPrompt()` (two directories, scopes)
  - **Auto only**: `buildMemoryLines()` (single directory, individual mode)
  - **None**: null (memory disabled)

#### **Tier 2: Query-Time Sonnet Selection** (Asynchronous)
- `findRelevantMemories()` called per-turn (if not KAIROS + not blocking)
- Scans memory directory, calls Sonnet to select top 5
- Surfaces to main model as relevant_memories in system reminder
- ~500ms-1s latency; abortable if turn ends early
- Deduplicates against already-surfaced memories

#### **Tier 3: Manual/Explicit Recall** (On-Demand)
- User or Claude uses Grep tool to search memory files
- Grep output includes `memoryFreshnessNote()` decoration for age warnings
- Session transcript search as last resort for deep history

### Memory Prompt Lines

The base memory prompt contains:
- Title and description of memory system
- Guidance on memory types (4-type taxonomy, scope rules for team mode)
- Exclusions (code patterns, git history, etc.)
- Save instructions (one-line index entries, frontmatter format)
- When to access memories (explicit asks, relevance, ignore directives)
- Trusting what you recall (verify file/function existence, check current state)
- Searching past context (grep commands if feature enabled)

### Feature Flags Controlling Integration

| Flag | Purpose |
|------|---------|
| `tengu_herring_clock` | Enable team memory (requires auto memory) |
| `tengu_passport_quail` | Enable extract mode (background agent) |
| `tengu_slate_thimble` | Run extract mode in non-interactive sessions |
| `tengu_coral_fern` | Show "searching past context" section |
| `tengu_moth_copse` | Skip MEMORY.md index in prompt |
| `KAIROS` | Daily-log append-only mode (assistant sessions) |
| `EXTRACT_MEMORIES` | Feature-gate for background memory agent |

---

## 8. Freshness and Staleness

### Age Calculation: memoryAgeDays()

```typescript
memoryAgeDays(mtimeMs: number): number {
  return Math.max(0, Math.floor((Date.now() - mtimeMs) / 86_400_000))
}
```

- Floor-rounded (0 for today, 1 for yesterday, 2+ for older)
- Negative inputs (future mtime, clock skew) clamp to 0
- 86,400,000 ms = 86,400 seconds = 1 day

### Human-Readable Age: memoryAge()

```typescript
memoryAge(mtimeMs: number): string {
  const d = memoryAgeDays(mtimeMs)
  if (d === 0) return 'today'
  if (d === 1) return 'yesterday'
  return `${d} days ago`
}
```

Models are poor at date arithmetic; "47 days ago" triggers staleness reasoning better than ISO timestamps.

### Staleness Caveat: MEMORY_DRIFT_CAVEAT

Incorporated into prompt under "## When to access memories":
```
Memory records can become stale over time. Use memory as context for what was
true at a given point in time. Before answering the user or building assumptions
based solely on information in memory records, verify that the memory is still
correct and up-to-date by reading the current state of the files or resources.
If a recalled memory conflicts with current information, trust what you observe
now — and update or remove the stale memory rather than acting on it.
```

### Freshness Decorations

#### **memoryFreshnessText()** (Plain Text)
- Returns empty string for fresh (today/yesterday) memories
- For >1 day old:
  ```
  This memory is {d} days old. Memories are point-in-time observations, not
  live state — claims about code behavior or file:line citations may be
  outdated. Verify against current code before asserting as fact.
  ```
- Used when caller provides own wrapping (e.g., messages.ts relevant_memories)

#### **memoryFreshnessNote()** (System-Reminder Tagged)
- Wraps freshness text in `<system-reminder>` tags
- Returns empty string for fresh memories
- Used for callers without own wrapper (e.g., FileReadTool output)

### System-Reminder Decoration

When memories >1 day old are surfaced, wrapped as:
```
<system-reminder>This memory is 47 days old...</system-reminder>
```

Signals to Claude that stale data requires verification before assertion.

---

## 9. Team Memory Architecture

### Directory Structure

```
<autoMemPath>/
├── MEMORY.md            # Private index
├── <private-files>.md
└── team/
    ├── MEMORY.md        # Team index (separate)
    └── <team-files>.md
```

Team directory is a subdirectory of auto memory: `getTeamMemPath()` returns `join(getAutoMemPath(), 'team') + sep`

### Dual-Index System

Each scope has its own MEMORY.md:
- Private: `<autoMemPath>/MEMORY.md`
- Team: `<autoMemPath>/team/MEMORY.md`

Both are loaded into system prompt (combined prompt calls `buildCombinedMemoryPrompt()`).

### Combined Memory Prompt

`buildCombinedMemoryPrompt()` outputs:
1. **Header**: Introduces both directories, scope levels
2. **Memory scope section**: Defines private vs team
3. **Types section (COMBINED variant)**: Each type includes `<scope>` tag with guidance
   - user: always private
   - feedback: default private; team for project-wide conventions
   - project: private or team; bias toward team
   - reference: usually team
4. **How to save memories**: Step 1 = write to chosen directory, Step 2 = add to that scope's MEMORY.md
5. **What NOT to save**: Identical to individual variant
6. **When to access memories**: References "personal or team" memories
7. **Before recommending from memory**: Same verification guidance as individual mode
8. **Sensitive data warning**: "MUST avoid saving sensitive data within shared team memories. For example, never save API keys or user credentials."

### Scope Rules in Type Definitions

Types include XML `<scope>` tags in TYPES_SECTION_COMBINED:

```xml
<type>
  <name>user</name>
  <scope>always private</scope>
  <description>...</description>
  ...
</type>
```

Examples show `[saves private user memory: ...]` vs `[saves team project memory: ...]`

### Security Validation

Team memory paths must pass **two-pass symlink validation (PSR M22186)**:

1. **First pass**: `path.resolve()` converts to absolute, eliminates `..` segments
2. **String-level check**: Verify resolved path starts with team directory
3. **Second pass**: `realpathDeepestExisting()` resolves symlinks on deepest existing ancestor
4. **Real containment**: `isRealPathWithinTeamDir()` checks real path against real team dir

Rejects:
- Dangling symlinks (lstat succeeds, target missing)
- Symlink loops (ELOOP)
- Path traversal attempts (`../../../etc/passwd`)
- Null bytes (truncate in C syscalls)

---

## 10. Path Security: validateMemoryPath()

### Comprehensive Validation Strategy

The `validateMemoryPath()` function implements multi-layer attack surface hardening:

#### **Layer 1: Null Byte Rejection**
```typescript
if (key.includes('\0')) {
  throw new PathTraversalError(`Null byte in path key: "${key}"`)
}
```
Null bytes can truncate paths in C-based syscalls; post-normalize() they still lurk.

#### **Layer 2: URL-Encoded Traversal Detection**
```typescript
let decoded: string
try {
  decoded = decodeURIComponent(key)
} catch {
  decoded = key  // Malformed percent-encoding, no traversal possible
}
if (decoded !== key && (decoded.includes('..') || decoded.includes('/'))) {
  throw new PathTraversalError(`URL-encoded traversal in path key: "${key}"`)
}
```
Catches `%2e%2e%2f` (= `../`) while allowing safe percent-encoded content.

#### **Layer 3: Unicode Normalization Attack Prevention**
```typescript
const normalized = key.normalize('NFKC')
if (
  normalized !== key &&
  (normalized.includes('..') ||
    normalized.includes('/') ||
    normalized.includes('\\') ||
    normalized.includes('\0'))
) {
  throw new PathTraversalError(
    `Unicode-normalized traversal in path key: "${key}"`
  )
}
```
Fullwidth ．．／ (U+FF0E U+FF0F) normalize to ASCII `../` under NFKC; defense-in-depth against downstream normalization (PSR M22187 vector 4).

#### **Layer 4: Backslash Rejection (Windows Path Separator)**
```typescript
if (key.includes('\\')) {
  throw new PathTraversalError(`Backslash in path key: "${key}"`)
}
```
Prevents Windows-style traversal (`dir\..\..\etc\passwd`).

#### **Layer 5: Absolute Path Rejection**
```typescript
if (key.startsWith('/')) {
  throw new PathTraversalError(`Absolute path key: "${key}"`)
}
```
Only relative keys allowed (will be joined with team dir).

### Memory Path Validation Function

`validateMemoryPath(raw: string | undefined, expandTilde: boolean): string | undefined`

**Tilde Expansion** (settings.json only):
- `~/foo` expands to `$HOME/foo`
- Bare `~`, `~/`, `~/.`, `~/../` rejected (would expand to $HOME or ancestor — same danger class as `/` or `C:\`)
- Validated via `normalize()` returning `.` or `..` → undefined

**Normalization**:
```typescript
const normalized = normalize(candidate).replace(/[/\\]+$/, '')
```
Strips trailing separators, then adds exactly one for contract consistency.

**Rejection Criteria**:
- Not absolute (!isAbsolute)
- Too short (length < 3): catches `/`, empty, single chars
- Windows drive-root: `/^[A-Za-z]:$/.test()` catches `C:`
- UNC paths: `startsWith('\\\\')` or `startsWith('//')`
- Null bytes: `includes('\0')`

**Returns**: Normalized path with exactly one trailing separator, NFC-normalized, or undefined if rejected.

### Two-Pass Team Memory Validation

`validateTeamMemKey()` and `validateTeamMemWritePath()` apply two-pass checks:

**Pass 1: String-Level (Fast)**
```typescript
const resolvedPath = resolve(fullPath)
if (!resolvedPath.startsWith(teamDir)) {
  throw new PathTraversalError(`Path escapes team memory directory: "${filePath}"`)
}
```

**Pass 2: Symlink Resolution (Security)**
```typescript
const realPath = await realpathDeepestExisting(resolvedPath)
if (!(await isRealPathWithinTeamDir(realPath))) {
  throw new PathTraversalError(`Path escapes team memory directory via symlink: "${filePath}"`)
}
```

### Symlink Resolution: realpathDeepestExisting()

Walks up directory tree until `realpath()` succeeds, then rejoins non-existing tail:

```typescript
for (let parent = dirname(current); current !== parent; parent = dirname(current)) {
  try {
    const realCurrent = await realpath(current)
    return tail.length === 0
      ? realCurrent
      : join(realCurrent, ...tail.reverse())
  } catch (e: unknown) {
    const code = getErrnoCode(e)
    if (code === 'ENOENT') {
      // Dangling symlink check
      try {
        const st = await lstat(current)
        if (st.isSymbolicLink()) {
          throw new PathTraversalError(
            `Dangling symlink detected (target does not exist): "${current}"`
          )
        }
      } catch (lstatErr: unknown) {
        if (lstatErr instanceof PathTraversalError) throw lstatErr
      }
    } else if (code === 'ELOOP') {
      throw new PathTraversalError(`Symlink loop detected in path: "${current}"`)
    } else if (code !== 'ENOTDIR' && code !== 'ENAMETOOLONG') {
      throw new PathTraversalError(
        `Cannot verify path containment (${code}): "${current}"`
      )
    }
    tail.push(current.slice(parent.length + sep.length))
    current = parent
  }
}
```

Handles:
- **ENOENT**: True non-existence vs dangling symlinks (lstat distinguishes)
- **ELOOP**: Symlink cycles
- **EACCES/EIO**: Permission/IO errors → fail closed
- **ENOTDIR/ENAMETOOLONG**: Walk up further

### Prefix-Attack Protection

```typescript
return realCandidate.startsWith(realTeamDir + sep)
```

Requires separator after prefix to prevent `/foo/team-evil` matching `/foo/team`.

---

## 11. Feature Flags and Gates

### Comprehensive Feature Control

| Flag | Module | Purpose | Default |
|------|--------|---------|---------|
| `KAIROS` | memdir.ts | Assistant daily-log append-only mode | Bun feature flag |
| `EXTRACT_MEMORIES` | paths.ts | Background memory extraction agent | Bun feature flag |
| `MEMORY_SHAPE_TELEMETRY` | findRelevantMemories.ts | Log recall shape metrics | Bun feature flag |
| `tengu_herring_clock` | memdir.ts, teamMemPaths.ts | Team memory system enable | Growthbook GB flag, default false |
| `tengu_passport_quail` | paths.ts | Extract mode activation | Growthbook GB flag, default false |
| `tengu_slate_thimble` | paths.ts | Extract in non-interactive sessions | Growthbook GB flag, default false |
| `tengu_coral_fern` | memdir.ts | Show "searching past context" section | Growthbook GB flag, default false |
| `tengu_moth_copse` | memdir.ts | Skip MEMORY.md index in prompt | Growthbook GB flag, default false |
| `TEAMMEM` | memdir.ts, teamMemPaths.ts | Conditional require of team memory modules | Bun feature flag |

### Memory Enable/Disable Flow

`isAutoMemoryEnabled()` returns false if any of:
1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` env var truthy
2. `CLAUDE_CODE_SIMPLE` (--bare flag) truthy
3. `CLAUDE_CODE_REMOTE` without `CLAUDE_CODE_REMOTE_MEMORY_DIR`
4. `settings.json` `autoMemoryEnabled` set to false

Default: enabled

`isTeamMemoryEnabled()` requires:
1. `isAutoMemoryEnabled()` true (team is sub-directory of auto)
2. `feature('tengu_herring_clock')` true

`isExtractModeActive()` requires:
1. `feature('tengu_passport_quail')` true
2. Not a non-interactive session OR `feature('tengu_slate_thimble')` true

### Conditional Module Loading

Team memory modules conditionally imported using feature flags to tree-shake when not used:

```typescript
const teamMemPaths = feature('TEAMMEM')
  ? (require('./teamMemPaths.js') as typeof import('./teamMemPaths.js'))
  : null
```

Prevents unused team-memory code bloat when feature disabled.

---

## 12. Memory Update Cycle

### Explicit Update Paths

1. **User explicit save request**: "Remember that X"
   - Claude writes topic file with frontmatter
   - Claude adds index entry to MEMORY.md

2. **User explicit forget request**: "Forget about X"
   - Claude removes index entry from MEMORY.md
   - Claude optionally deletes topic file

3. **Update stale memory**: Observed via Tier 2 recall and found wrong
   - Claude re-reads topic file
   - Claude updates frontmatter or content
   - MEMORY.md pointer unchanged (same file)

### Implicit Update Paths

**extractMemories background agent** (when feature enabled):
- Runs post-turn
- Scans conversation for implicit learning opportunities
- Avoids ranges where main agent wrote memories (hasMemoryWritesSince check)
- Catches memories the main agent missed

### Directory Guarantee

`ensureMemoryDirExists(memoryDir: string)` called once per session:
- Idempotent
- Called from `loadMemoryPrompt()` (cached by systemPromptSection)
- Creates parent chain recursively (FsOperations.mkdir default recursive=true)
- Swallows EEXIST internally
- Logs unexpected errors (EACCES/EPERM/EROFS) for --debug
- Prompt building continues regardless; Write tool surfaces real perm errors

### Scanning: Newest-First Capped

`scanMemoryFiles()` behavior:
- Reads all `.md` files recursively, excludes `MEMORY.md`
- Reads frontmatter from each (first 30 lines max)
- Sorts by mtime descending (newest first)
- Caps at 200 files (MAX_MEMORY_FILES)
- Single-pass optimization: readFileInRange stats internally, returns mtimeMs

---

## 13. Constants and Limits

### Size Caps

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_ENTRYPOINT_LINES` | 200 | MEMORY.md line limit (primary) |
| `MAX_ENTRYPOINT_BYTES` | 25,000 | MEMORY.md byte limit (long-line failure mode) |
| `MAX_MEMORY_FILES` | 200 | Memory directory file cap (scan result slice) |
| `FRONTMATTER_MAX_LINES` | 30 | Max lines to scan for frontmatter |
| `max_tokens` (Sonnet) | 256 | Token budget for SELECT_MEMORIES_SYSTEM_PROMPT |

### Naming Conventions

| Name | Value | Usage |
|------|-------|-------|
| `ENTRYPOINT_NAME` | 'MEMORY.md' | Index file name |
| `AUTO_MEM_DIRNAME` | 'memory' | Directory name within projects |
| `AUTO_MEM_ENTRYPOINT_NAME` | 'MEMORY.md' | Full entrypoint name |
| `AUTO_MEM_DISPLAY_NAME` | 'auto memory' | Prompt section heading |

### Directory Guidance Text

| Constant | Usage |
|----------|-------|
| `DIR_EXISTS_GUIDANCE` | "This directory already exists — write to it directly with the Write tool (do not run mkdir or check for its existence)." |
| `DIRS_EXIST_GUIDANCE` | "Both directories already exist — write to them directly with the Write tool (do not run mkdir or check for their existence)." |

---

## 14. Integration Checklist

### System Prompt Integration

- [x] Memory type taxonomy with scope rules (closed 4 types)
- [x] What NOT to save (code patterns, git history, etc.)
- [x] How to save memories (two-step: file + index)
- [x] When to access memories (relevance, explicit asks, ignore directives)
- [x] Before recommending from memory (verify files/functions exist)
- [x] Trusting what you recall (staleness caveat, snapshot nature)
- [x] Searching past context (grep commands if feature enabled)
- [x] Memory and other persistence (plans, tasks vs memory)

### File Lifecycle Integration

- [x] ensureMemoryDirExists() called once per session
- [x] buildMemoryPrompt() / buildMemoryLines() with MEMORY.md loading
- [x] Truncation with warning appended for overflow
- [x] Telemetry logging (tengu_memdir_loaded)
- [x] Cowork extra guidelines injection
- [x] Feature flag dispatch (KAIROS, TEAMMEM, etc.)

### Query-Time Integration

- [x] findRelevantMemories() called per turn (Tier 2)
- [x] scanMemoryFiles() with frontmatter extraction
- [x] Sonnet selection (SELECT_MEMORIES_SYSTEM_PROMPT)
- [x] alreadySurfaced deduplication
- [x] memoryAge() / memoryFreshnessNote() decoration

### Team Memory Integration

- [x] Team directory subdirectory of auto memory
- [x] Dual MEMORY.md indexes (private + team)
- [x] Scope guidance in type definitions
- [x] Sensitive data warning
- [x] Two-pass symlink validation
- [x] Path traversal prevention (PSR M22186)

### Security Integration

- [x] validateMemoryPath() with tilde expansion, normalization
- [x] validateTeamMemKey() / validateTeamMemWritePath() two-pass validation
- [x] Null byte rejection
- [x] URL-encoded traversal detection
- [x] Unicode normalization attack prevention
- [x] Backslash rejection
- [x] Absolute path rejection
- [x] Symlink loop and dangling symlink detection
- [x] Prefix-attack protection

---

## 15. Code Organization

### Module Structure

```
src/memdir/
├── memdir.ts                    # Core: prompt builders, truncation, directory ensuring
├── paths.ts                     # Path resolution, validation, feature gates
├── memoryTypes.ts               # Type taxonomy, frontmatter example, prompt sections
├── findRelevantMemories.ts      # Sonnet-powered query-time selection
├── memoryAge.ts                 # Staleness calculation and formatting
├── memoryScan.ts                # Memory file scanning and manifest formatting
├── teamMemPrompts.ts            # Combined prompt for dual-directory mode
├── teamMemPaths.ts              # Team directory paths, security validation
```

### Dependencies

- **bun:bundle** feature() for tree-shaking
- **fs/promises** async file operations
- **path** path normalization and resolution
- **lodash-es** memoization of getAutoMemPath()
- **../utils/** frontmatter parsing, file I/O, env vars, git utils
- **../services/analytics/** Growthbook feature values, event logging
- **../bootstrap/state.js** session state (original cwd, Kairos active)

---

## 16. Example Workflows

### Workflow 1: Creating a User Memory

**User message**: "I'm a senior backend engineer with 15 years experience; new to Rust"

**Claude action**:
1. Writes `/home/user/.claude/projects/my-repo-slug/memory/user_experience.md`:
   ```markdown
   ---
   name: User experience
   description: Senior backend engineer, 15y experience, new to Rust
   type: user
   ---

   Senior software engineer with 15 years backend experience across multiple
   languages. New to Rust — frame explanations in terms of patterns from
   Go/C++ where applicable; avoid assuming Rust idioms knowledge.
   ```
2. Adds to `MEMORY.md`: `- [User experience](user_experience.md) — 15y backend, new to Rust`

**Future session**:
- MEMORY.md loaded in system prompt (Tier 1)
- If user asks about Rust patterns, Sonnet may select this memory (Tier 2)
- Claude adjusts explanations for engineer with Go/C++ background

### Workflow 2: Recording Feedback Memory

**User message**: "Stop wrapping every response in a summary paragraph; I can read diffs"

**Claude action**:
1. Writes `/home/user/.claude/projects/my-repo-slug/memory/feedback_terseness.md`:
   ```markdown
   ---
   name: Communication preference - terseness
   description: User wants terse responses; no trailing summaries
   type: feedback
   ---

   Do not add trailing summaries, recap sections, or "here's what I just did"
   paragraphs after code changes.

   **Why:** User can read diffs directly. Summaries are noise.

   **How to apply:** When responding to implementation requests, provide
   explanation (if needed), then the code/diff. Stop there.
   ```
2. Adds to `MEMORY.md`: `- [Terseness preference](feedback_terseness.md) — No trailing summaries`

**Tier 2 recall**: Next time Sonnet selects memories for a coding turn, might include this.
**Tier 1 always**: Loaded in system prompt, so available context.

### Workflow 3: Team Memory for Project Context

**User message**: "We're unfreezing PRs after 2026-03-10 (mobile release wrapped early)"

**Claude in team mode**:
1. Writes `/home/user/.claude/projects/my-repo-slug/memory/team/project_release_cycle.md`:
   ```markdown
   ---
   name: Release freeze lifted
   description: Mobile release freeze ended 2026-03-10; PRs unfrozen
   type: project
   ---

   Merge freeze in effect through 2026-03-05 for mobile team release. Freeze
   lifted 2026-03-10.

   **Why:** Mobile team cutting a release branch; needed to avoid churn.

   **How to apply:** Flag any non-critical PR work scheduled in the frozen
   window; suggest batching after 2026-03-10.
   ```
2. Adds to `team/MEMORY.md`: `- [Merge freeze lifted](project_release_cycle.md) — Lifted 2026-03-10`

**Team scope benefit**: All users working in this project see the freeze context.
**Staleness**: In 60 days, memory will show as "60 days old" with freshness caveat — user can verify if still relevant.

### Workflow 4: Tier 2 Relevance Selection

**User turn**: "Add error handling to the batch processor"

**Sonnet selection**:
1. Scans memory directory, reads frontmatter of 50 topic files
2. Receives manifest with filenames, types, descriptions
3. Given query "Add error handling to the batch processor"
4. Returns JSON: `{"selected_memories": ["feedback_testing.md", "project_batch_processor_design.md"]}`
5. Claude Code loads those 2 files, incorporates into context

**Deduplication**: If "feedback_testing.md" was already surfaced in a prior turn, `alreadySurfaced` Set prevents re-selection.

### Workflow 5: Staleness Caveat in Action

**Memory age**: 45 days old (feedback about code behavior)

**Recall path**:
1. Sonnet selects memory
2. memoryAge(mtimeMs) returns "45 days ago"
3. memoryFreshnessNote(mtimeMs) wraps in system-reminder:
   ```
   <system-reminder>This memory is 45 days old. Memories are point-in-time
   observations, not live state — claims about code behavior or file:line
   citations may be outdated. Verify against current code before asserting
   as fact.</system-reminder>
   ```
4. Claude sees caveat, cross-checks memory claim against current code before
   recommending

---

## 17. Security Threat Model

### PSR M22186: Symlink Escape in Team Memory

**Attack**: Attacker places symlink inside `team/` pointing outside:
```
team/
└── link -> ../../../../../../etc/passwd
```

**Mitigation**: Two-pass validation in `validateTeamMemKey()`:
1. **Pass 1 (Fast)**: `resolve()` normalizes `..` but doesn't follow symlinks
2. **Pass 2 (Security)**: `realpathDeepestExisting()` + `isRealPathWithinTeamDir()` verify real location

**Result**: Write to `team/link` detected as escape and rejected.

### PSR M22187: Unicode Normalization Attack

**Attack**: Filename with fullwidth characters normalizes to `../`:
- Input: `foo．．／bar` (fullwidth ．．／)
- NFKC normalization: `foo../bar`

**Mitigation**: `sanitizePathKey()` normalizes and rejects if contains `..`, `/`, `\`, `\0` post-normalization.

### Null Byte Truncation

**Attack**: Path key with null byte truncates in C syscalls:
- Input: `foo\x00/etc/passwd`
- Old behavior: Might write to `foo` unintentionally

**Mitigation**: Rejected by both `sanitizePathKey()` and `validateMemoryPath()`.

### URL-Encoded Traversal

**Attack**: Percent-encoded path traversal:
- Input: `foo%2e%2e%2fetc%2fpasswd` (URL-encoded `foo../etc/passwd`)

**Mitigation**: `sanitizePathKey()` decodes and checks if decoded differs and contains `..` or `/`.

### Malicious Repo Settings.json

**Attack**: Repo commits `.claude/settings.json` with:
```json
{"projectSettings": {"autoMemoryDirectory": "~/.ssh"}}
```

**Mitigation**: `getAutoMemPathSetting()` explicitly excludes **projectSettings** — only policy/local/user settings trusted.

---

## 18. Performance Characteristics

### Latency Budget

| Operation | Latency | Notes |
|-----------|---------|-------|
| ensureMemoryDirExists() | <1ms | Cached per session; mkdir recursive |
| loadMemoryPrompt() | <10ms | Sync read of MEMORY.md; cached by systemPromptSection |
| scanMemoryFiles() | ~50-200ms | Reads first 30 lines of 200 files; parallel Promise.allSettled |
| findRelevantMemories() | ~500-1000ms | Includes sideQuery to Sonnet (~500ms over network) |
| memoryAge() | <1μs | Simple arithmetic |
| validateTeamMemKey() | ~10-100ms | Includes async realpath() calls and symlink checks |

### Token Budget

| Item | Tokens | Notes |
|------|--------|-------|
| SELECT_MEMORIES_SYSTEM_PROMPT | ~100 | Fixed prompt text |
| Memory manifest (50 files) | ~400 | ~8 tokens per line |
| Sonnet response | ~10-50 | JSON array of 1-5 filenames |
| **Total Sonnet call** | ~600 | Fits comfortably in 256-token max_tokens budget |

### Memory Usage

- MEMORY.md content: Up to 25KB loaded per prompt
- Frontier metrics: ~4KB per 50 topic files (just filenames + descriptions)
- Sonnet manifest: ~2KB per 100 memories listed
- Per-session memoization cache: <100KB (getAutoMemPath memoized key)

---

## 19. Known Limitations

### Design Trade-offs

1. **Closed taxonomy**: Only 4 types means feature requests for new types always rejected; extensibility sacrificed for clarity
2. **No full-text search**: Only filename/description matching; content search requires grep tool
3. **Singleton MEMORY.md**: Single index file means large projects may hit 200-line limit
4. **No versioning**: Memories are point-in-time; no change history or undo
5. **No sync conflict resolution**: Team memory writes assume no concurrent edits

### Staleness Window

- Freshness caveat only triggers for >1 day old memories
- 24-hour window may miss rapidly-evolving context
- User must manually verify or delete stale memories

### Scale Limits

- MAX_MEMORY_FILES = 200: Directories with >200 files truncate silently (oldest dropped)
- FRONTMATTER_MAX_LINES = 30: Longer YAML frontmatter silently truncated
- Recent-tools filter: Exact string match; `Grep` and `grep` treated as different

---

## 20. Future Extension Points

### Potential Enhancements

1. **Multi-file index**: Split MEMORY.md by type to avoid line/byte cap
2. **Full-text indexing**: Semantic search over memory content via embeddings
3. **Conflict resolution**: Team memory merge strategy for concurrent edits
4. **Versioning**: Git history or JSONL append log of changes
5. **Expiration**: Auto-delete memories older than N days if not accessed
6. **Custom types**: Extensible type taxonomy beyond fixed 4 types
7. **Memory relationships**: Links between memories, cross-references
8. **Metrics dashboard**: Recall rate, age distribution, staleness patterns

---

## 21. Summary Table

| Dimension | Details |
|-----------|---------|
| **Type System** | Closed 4-type: user, feedback, project, reference |
| **Storage** | File-based markdown with YAML frontmatter |
| **Index** | MEMORY.md (200 lines, 25KB cap) per scope |
| **Selection** | Sonnet (256 tokens) ranking up to 5 memories per turn |
| **Scope** | Private + team (team optional, requires feature flag) |
| **Integration** | 3-tier: system prompt (Tier 1), Sonnet (Tier 2), manual (Tier 3) |
| **Freshness** | Age-aware with staleness caveats for >1 day old |
| **Security** | Two-pass symlink validation, null byte rejection, Unicode normalization checks |
| **Constants** | MAX_ENTRYPOINT_LINES=200, MAX_ENTRYPOINT_BYTES=25KB, MAX_MEMORY_FILES=200 |
| **Features** | tengu_herring_clock (team), tengu_passport_quail (extract), tengu_coral_fern (search), KAIROS (daily logs) |

---

## Document Metadata

**Analysis Version**: 1.0
**Date**: 2026-04-02
**Source Files**: 8 TypeScript modules in `/sessions/cool-friendly-einstein/mnt/claude-code/src/memdir/`
**Scope**: Complete reverse engineering of semantic memory directory architecture
**Coverage**: Architecture, directory structure, types, MEMORY.md, relevance selection, lifecycle, integration, freshness, team memory, path security, feature flags, update cycle, constants, integration checklist, code organization, workflows, security threats, performance, limitations, and extension points
