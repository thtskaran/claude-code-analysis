# Claude Code Keybindings System - Comprehensive Technical Analysis

**Analyzed:** 14 files, ~3,159 lines of code
**Date:** 2026-04-02
**Scope:** Complete keybinding architecture, bindings inventory, security, platform differences

---

## ARCHITECTURE OVERVIEW

### System Design Pattern
The keybindings system follows a modular, configuration-driven architecture:

1. **Default Bindings Layer** (`defaultBindings.ts`)
   - Hard-coded platform-specific defaults
   - Feature-gated bindings (enabled via feature flags)
   - Acts as base layer before user overrides

2. **User Config Layer** (`loadUserBindings.ts`)
   - Loads from `~/.claude/keybindings.json` (Anthropic employees only)
   - Watched for hot-reload with chokidar
   - Merged on top of defaults (last wins)
   - Gated by `tengu_keybinding_customization_release` feature flag

3. **Resolution Pipeline** (`resolver.ts`, `match.ts`)
   - Single-key matching (Phase 1)
   - Multi-key chord sequences (Phase 2) with timeout
   - Context-based priority (specific contexts > Global)
   - Returns actions for dispatch or null for unbound keys

4. **React Integration** (`KeybindingContext.tsx`, `KeybindingProviderSetup.tsx`)
   - Context-based handler registration
   - Dual mode: ref-based (sync) + state-based (UI updates)
   - ChordInterceptor pre-processes chord keystrokes
   - useKeybinding hooks for components

5. **Validation & Safety** (`validate.ts`, `reservedShortcuts.ts`)
   - Pre-binding validation (syntax, context, action)
   - Reserved shortcut detection (OS-level, terminal)
   - Non-rebindable keys (ctrl+c, ctrl+d, ctrl+m)

---

## COMPLETE KEYBINDING INVENTORY

### Global Context (App-wide, 21 bindings)

#### Control Flow
- `ctrl+c` → `app:interrupt` ⚠️ HARDCODED, non-rebindable
  - Uses double-press time-based detection (see comments)
  - Cannot be overridden by user config
- `ctrl+d` → `app:exit` ⚠️ HARDCODED, non-rebindable
  - Cannot be overridden by user config
- `ctrl+l` → `app:redraw` - Full terminal redraw

#### Workspace/UI Toggle
- `ctrl+t` → `app:toggleTodos` - Show/hide todo sidebar
- `ctrl+o` → `app:toggleTranscript` - View message transcript
- `ctrl+shift+o` → `app:toggleTeammatePreview` - Team collaboration view
- `ctrl+shift+b` → `app:toggleBrief` - **FEATURE-GATED** (KAIROS | KAIROS_BRIEF)
- `meta+j` → `app:toggleTerminal` - **FEATURE-GATED** (TERMINAL_PANEL)

#### Search & Navigation (FEATURE-GATED: QUICK_SEARCH)
- `ctrl+shift+f` → `app:globalSearch` - Search entire codebase
- `cmd+shift+f` → `app:globalSearch` - (kitty protocol terminals only)
- `ctrl+shift+p` → `app:quickOpen` - Quick file/command opener
- `cmd+shift+p` → `app:quickOpen` - (kitty protocol)

#### History Navigation
- `ctrl+r` → `history:search` - Open command history search

---

### Chat Context (Input-focused, 26 bindings)

#### Submission & Cancellation
- `enter` → `chat:submit` - Send message
- `escape` → `chat:cancel` - Clear/cancel input

#### Navigation (History)
- `up` → `history:previous` - Previous command in history
- `down` → `history:next` - Next command in history

#### Mode Cycling & Model Selection
- `shift+tab` → `chat:cycleMode` (most platforms) - Cycle input modes
- `meta+m` → `chat:cycleMode` - **Windows fallback** (no VT mode support)
- `meta+p` → `chat:modelPicker` - Open model selection menu
- `meta+o` → `chat:fastMode` - Toggle fast/full reasoning mode
- `meta+t` → `chat:thinkingToggle` - Toggle extended thinking

#### Editing
- `ctrl+_` → `chat:undo` - Undo last input (legacy terminals)
- `ctrl+shift+-` → `chat:undo` - Undo (Kitty protocol)
- `ctrl+x ctrl+e` → `chat:externalEditor` - Open external editor
- `ctrl+g` → `chat:externalEditor` - Shorter binding for same action
- `ctrl+s` → `chat:stash` - Save input draft

#### Special Input
- `alt+v` → `chat:imagePaste` (Windows) - Paste image from clipboard
- `ctrl+v` → `chat:imagePaste` (Mac/Linux) - Paste image
- `space` → `voice:pushToTalk` - **FEATURE-GATED** (VOICE_MODE)
  - Hold-to-talk voice activation

#### Agent Control
- `ctrl+x ctrl+k` → `chat:killAgents` - Stop running agents

#### Advanced
- `shift+up` → `chat:messageActions` - **FEATURE-GATED** (MESSAGE_ACTIONS)
  - Access message action menu

---

### Autocomplete Context (4 bindings)
- `tab` → `autocomplete:accept` - Accept suggestion
- `escape` → `autocomplete:dismiss` - Close autocomplete menu
- `up` → `autocomplete:previous` - Previous suggestion
- `down` → `autocomplete:next` - Next suggestion

---

### Confirmation/Dialog Context (8 bindings)
- `y` → `confirm:yes` - Confirm yes
- `n` → `confirm:no` - Confirm no
- `enter` → `confirm:yes` - Alternative confirmation
- `escape` → `confirm:no` - Cancel dialog
- `up` → `confirm:previous` - Navigate dialog options up
- `down` → `confirm:next` - Navigate dialog options down
- `tab` → `confirm:nextField` - Move to next form field
- `space` → `confirm:toggle` - Toggle option/checkbox
- `shift+tab` → `confirm:cycleMode` - Cycle mode in dialogs
- `ctrl+e` → `confirm:toggleExplanation` - Show/hide permission explanation
- `ctrl+d` → `permission:toggleDebug` - **DEBUG** - Toggle debug info in permission dialogs

---

### Settings Context (6 bindings)
- `escape` → `confirm:no` - Close settings
- `up` → `select:previous` - Navigate up
- `down` → `select:next` - Navigate down
- `k` → `select:previous` - vim-style up
- `j` → `select:next` - vim-style down
- `ctrl+p` → `select:previous` - Emacs-style up
- `ctrl+n` → `select:next` - Emacs-style down
- `space` → `select:accept` - Toggle setting
- `enter` → `settings:close` - Save and close
- `/` → `settings:search` - Enter search mode
- `r` → `settings:retry` - Retry loading (on error only)

---

### Tabs Context (4 bindings)
- `tab` → `tabs:next` - Next tab
- `shift+tab` → `tabs:previous` - Previous tab
- `right` → `tabs:next` - Arrow key navigation
- `left` → `tabs:previous` - Arrow key navigation

---

### Transcript (Reader) Context (4 bindings)
- `escape` → `transcript:exit` - Exit transcript view
- `ctrl+c` → `transcript:exit` - Alternative exit (interrupt-style)
- `q` → `transcript:exit` - Standard pager convention (less, tmux)
- `ctrl+e` → `transcript:toggleShowAll` - Show/hide all messages

---

### History Search Context (5 bindings)
- `ctrl+r` → `historySearch:next` - Find next match
- `escape` → `historySearch:accept` - Accept selected
- `tab` → `historySearch:accept` - Alternative accept
- `ctrl+c` → `historySearch:cancel` - Cancel search
- `enter` → `historySearch:execute` - Execute selected command

---

### Task (Background Execution) Context (1 binding)
- `ctrl+b` → `task:background` - Send running task to background
  - Note: In tmux, users must press ctrl+b twice (tmux prefix escapes)

---

### Theme Picker Context (1 binding)
- `ctrl+t` → `theme:toggleSyntaxHighlighting` - Toggle syntax highlighting
  - Conflicts with Global `ctrl+t` → `app:toggleTodos` (Context Priority!)

---

### Scroll Context (8 bindings)
- `pageup` → `scroll:pageUp` - Scroll up one page
- `pagedown` → `scroll:pageDown` - Scroll down one page
- `wheelup` → `scroll:lineUp` - Scroll up (mouse wheel)
- `wheeldown` → `scroll:lineDown` - Scroll down (mouse wheel)
- `ctrl+home` → `scroll:top` - Jump to top
- `ctrl+end` → `scroll:bottom` - Jump to bottom
- `ctrl+shift+c` → `selection:copy` - Copy selected text
- `cmd+c` → `selection:copy` - (kitty protocol terminals)

---

### Help Overlay Context (1 binding)
- `escape` → `help:dismiss` - Close help overlay

---

### Attachments Context (Image Navigation) (4 bindings)
- `right` → `attachments:next` - Next attachment
- `left` → `attachments:previous` - Previous attachment
- `backspace` → `attachments:remove` - Remove selected
- `delete` → `attachments:remove` - Alternative remove
- `down` → `attachments:exit` - Exit attachment view
- `escape` → `attachments:exit` - Alternative exit

---

### Footer Context (Status Bar) (7 bindings)
- `up` → `footer:up` - Navigate footer items up
- `ctrl+p` → `footer:up` - Emacs-style up
- `down` → `footer:down` - Navigate footer items down
- `ctrl+n` → `footer:down` - Emacs-style down
- `right` → `footer:next` - Next indicator
- `left` → `footer:previous` - Previous indicator
- `enter` → `footer:openSelected` - Open/expand selected
- `escape` → `footer:clearSelection` - Clear selection

---

### Message Selector Context (Rewind Dialog) (9 bindings)
- `up` → `messageSelector:up` - Previous message
- `down` → `messageSelector:down` - Next message
- `k` → `messageSelector:up` - vim-style up
- `j` → `messageSelector:down` - vim-style down
- `ctrl+p` → `messageSelector:up` - Emacs-style up
- `ctrl+n` → `messageSelector:down` - Emacs-style down
- `ctrl+up` → `messageSelector:top` - Jump to first
- `shift+up` → `messageSelector:top` - Alternative first
- `meta+up` → `messageSelector:top` - Alternative first (cmd on macOS)
- `shift+k` → `messageSelector:top` - vim-style first
- `ctrl+down` → `messageSelector:bottom` - Jump to last
- `shift+down` → `messageSelector:bottom` - Alternative last
- `meta+down` → `messageSelector:bottom` - Alternative last
- `shift+j` → `messageSelector:bottom` - vim-style last
- `enter` → `messageSelector:select` - Select and apply

---

### Message Actions Context (13 bindings, FEATURE-GATED: MESSAGE_ACTIONS)
- `up` → `messageActions:prev` - Previous message
- `down` → `messageActions:next` - Next message
- `k` → `messageActions:prev` - vim-style up
- `j` → `messageActions:next` - vim-style down
- `meta+up` → `messageActions:top` - Jump to first (cmd/super)
- `meta+down` → `messageActions:bottom` - Jump to last
- `super+up` → `messageActions:top` - (kitty protocol)
- `super+down` → `messageActions:bottom` - (kitty protocol)
- `shift+up` → `messageActions:prevUser` - Previous user message
- `shift+down` → `messageActions:nextUser` - Next user message
- `escape` → `messageActions:escape` - Cancel action
- `ctrl+c` → `messageActions:ctrlc` - Interrupt (mirrors ctrl+c behavior)
- `enter` → `messageActions:enter` - Confirm action
- `c` → `messageActions:c` - Shortcut key
- `p` → `messageActions:p` - Shortcut key

---

### Diff Dialog Context (5 bindings)
- `escape` → `diff:dismiss` - Close diff viewer
- `left` → `diff:previousSource` - Previous diff source
- `right` → `diff:nextSource` - Next diff source
- `up` → `diff:previousFile` - Previous file in diff
- `down` → `diff:nextFile` - Next file in diff
- `enter` → `diff:viewDetails` - View details/expand

---

### Model Picker Context (2 bindings, effort cycling)
- `left` → `modelPicker:decreaseEffort` - Reduce model effort (ant-only)
- `right` → `modelPicker:increaseEffort` - Increase model effort

---

### Select/List Component Context (4 bindings, generic)
- `up` → `select:previous` - Navigate up
- `down` → `select:next` - Navigate down
- `j` → `select:next` - vim-style down
- `k` → `select:previous` - vim-style up
- `ctrl+n` → `select:next` - Emacs-style next
- `ctrl+p` → `select:previous` - Emacs-style previous
- `enter` → `select:accept` - Select/confirm
- `escape` → `select:cancel` - Cancel

---

### Plugin Dialog Context (2 bindings)
- `space` → `plugin:toggle` - Enable/disable plugin
- `i` → `plugin:install` - Install plugin

---

## HIDDEN & DEBUG KEYBINDINGS

### Explicitly Hidden (Not in Help)
1. **`permission:toggleDebug`** (`ctrl+d` in Confirmation context)
   - Purpose: Debug permission dialog information
   - Exposure: Only available in permission dialogs
   - Security: Non-sensitive (shows internal permission state)

### Fallback Bindings (Safety Net)
- System has fallback display text for actions that don't resolve from config
  - Logs `tengu_keybinding_fallback_used` event (for migration monitoring)
  - Only happens if bindings fail to load or action not found

### Implicit Behaviors (Not Listed)
- Space key handling:
  - Default: `voice:pushToTalk` when VOICE_MODE enabled
  - Validation catches bare letter bindings (like 'a') for voice (prints during warmup)
  - System warns if voice:pushToTalk bound to bare letter key

---

## PLATFORM-SPECIFIC DIFFERENCES

### Windows vs Mac/Linux

#### Image Paste Shortcut
- Windows: `alt+v` (ctrl+v is system paste on Windows terminal)
- Mac/Linux: `ctrl+v` (standard in *nix terminals)

#### Mode Cycling
- Windows (VT mode enabled): `shift+tab` (Node ≥24.2.0 / 22.17.0 or Bun ≥1.2.23)
- Windows (no VT mode): `meta+m` (**Fallback** - Modifier-only chords fail without VT)
- Mac/Linux: `shift+tab` (Always supported)

#### Display Names for Keys
- Alt vs Opt: opt on macOS, alt elsewhere
- cmd vs super: cmd on macOS, super elsewhere

#### Super/Cmd Modifier
- Only arrives via kitty keyboard protocol (kitty, WezTerm, ghostty, iTerm2)
- Bindings with cmd/super simply never fire on non-kitty terminals
- Not a silently-ignored modifier—the keystroke literally won't match

---

## VIM MODE ANALYSIS

The system includes extensive vim keybindings across multiple contexts:

### Navigation (vim-style)
- j → Next/down
- k → Previous/up
- ctrl+n → Next
- ctrl+p → Previous

### Jump to Bounds (vim-style)
- shift+k → Jump to top
- shift+j → Jump to bottom
- ctrl+up / shift+up → Jump to top
- ctrl+down / shift+down → Jump to bottom

### Design Pattern
Multiple bindings for same action reduce muscle memory switching:
- Arrow keys for unknown users
- vim (j/k) for vim users
- Emacs (ctrl+n/p) for Emacs users

---

## ACTION DISPATCH MECHANISM

### Phase 1: Single-Key Resolution
Input (string, Key) → Match ParsedKeystroke against active contexts → Last match wins → Return result

### Phase 2: Chord Resolution
1. Check if chord could START (is prefix of longer chords)
2. Check for EXACT MATCH of full chord
3. If pending chord: cancel on escape or invalid key
4. Default: {type: 'none'} (let other handlers process)

Timeout: 1000ms - if user doesn't complete chord in time, cancel

### Context Priority Resolution
Contexts: [RegisteredContexts, ComponentContext, Global]
Deduplicated, preserving order (first occurrence wins)
Later contexts override earlier

### Handler Invocation
1. Get all handlers registered for action
2. Find first handler whose context is ACTIVE
3. Invoke handler synchronously

---

## KEY CONFLICT RESOLUTION

### Chord Masking Strategy
- Multi-keystroke bindings shadow single-key bindings
- Define `ctrl+x ctrl+k` → `chat:killAgents`
- `ctrl+x` becomes "wait for next key" (chord pending)
- User can null-unbind `ctrl+x ctrl+k` to make `ctrl+x` single-key

### Context Override Strategy
When context "ThemePicker" is active:
- ThemePicker: { ctrl+t → toggle:syntax }
- Global: { ctrl+t → app:toggleTodos }
- **ThemePicker wins** (more specific)

### Last-Wins Override
User bindings always come AFTER defaults in merged list

---

## SECURITY ASSESSMENT

### Threat Model 1: Keybinding Bypass of Permissions
**Finding:** NO BYPASS RISK
- Keybindings only trigger CONFIGURED actions
- Actions do NOT include: file access, API calls, data exfiltration

### Threat Model 2: Hidden Commands via Keybinding
**Finding:** LIMITED RISK (mitigated)
- Command bindings are Chat-context only
- Restricted by application logic
- All command bindings go through normal validation

### Threat Model 3: Non-Rebindable Keys (Integrity)
**Finding:** CRITICAL KEYS PROTECTED
- ctrl+c → app:interrupt (Cannot override)
- ctrl+d → app:exit (Cannot override)
- ctrl+m → Cannot override (terminal limitation)

### Threat Model 4: Voice Mode Security
**Finding:** CONFIGURABLE WITH WARNINGS
- voice:pushToTalk on bare letters warns
- System validates bare letter keys (a-z)
- Recommendation: use space or modifier combos

### Threat Model 5: External Keybinding Injection
**Finding:** INJECTION IMPOSSIBLE
- Keybindings loaded from ~/.claude/keybindings.json only
- File requires valid JSON
- Validation runs before resolution

### Threat Model 6: DOS via Malformed Keybindings
**Finding:** HANDLED GRACEFULLY
- Parse errors fall back to defaults
- Hot-reload catches errors and notifies user

---

## FEATURE-GATED KEYBINDINGS

### GrowthBook Feature Flags
- KAIROS | KAIROS_BRIEF: ctrl+shift+b → app:toggleBrief
- TERMINAL_PANEL: meta+j → app:toggleTerminal
- QUICK_SEARCH: ctrl+shift+f/p, cmd+shift+f/p
- VOICE_MODE: space → voice:pushToTalk
- MESSAGE_ACTIONS: Entire MessageActions context (13 bindings)

### User Customization Gate
- tengu_keybinding_customization_release (loadUserBindings.ts)
- Currently: Anthropic employees only (USER_TYPE === 'ant')
- External users: Always use defaults (no customization)

---

## VALIDATION SYSTEM

### Pre-Binding Validation
1. Syntax Validation - Check for empty parts, ensure parsing
2. Context Validation - Must be one of 18 defined contexts
3. Action Validation - Must be known action OR command:* OR null
4. Voice Mode Validation - Warns for bare letter keys
5. Duplicate Detection - Checks raw JSON for duplicates
6. Reserved Shortcut Detection - Non-rebindable, terminal control, macOS system keys

### Validation Result Types
- parse_error - Syntax error
- duplicate - Key appears multiple times
- reserved - Key unlikely to work (OS/terminal intercepts)
- invalid_context - Unknown context name
- invalid_action - Unknown action or wrong context

---

## TECHNICAL DEBT & TODOS

### Active TODOs
1. Remove fallback parameter after migration (keybindings-migration)
   - Fallback display text exists as safety net
   - Plan: Once stable, remove defensive pattern
   - Telemetry: 'tengu_keybinding_fallback_used' events tracking

2. Chord Timeout Hardcoded (1000ms)
   - No user configuration for timeout
   - Potential TODO: Make configurable in keybindings.json

### Key Quirks & Comments
1. Escape Key: Ink sets key.meta=true when escape pressed (legacy behavior)
2. Alt/Meta Collapse: Can't distinguish at TTY level
3. Super/Cmd Limitation: Only arrives via kitty keyboard protocol
4. VT Mode Windows Terminal: Modifier-only chords fail without VT mode
5. readline Editing Keys: ctrl+x prefix used to avoid shadowing
6. Chord Prefix Masking: If "ctrl+x ctrl+k" is bound, "ctrl+x" enters chord-wait
7. Fallback to Defaults: Invalid keybindings.json falls back to defaults

---

## ACTION INVENTORY

### Complete Action List (97 actions)
- App-level: app:* (10 actions)
- Chat input: chat:* (13 actions)
- History: history:* (2 actions)
- Autocomplete: autocomplete:* (4 actions)
- Confirmation: confirm:* (8 actions)
- And 20+ more categories (tabs, transcript, search, task, theme, help, etc.)

---

## CUSTOM KEYBINDING SUPPORT

### User Config Format
Location: ~/.claude/keybindings.json

```json
{
  "$schema": "https://www.schemastore.org/claude-code-keybindings.json",
  "$docs": "https://code.claude.com/docs/en/keybindings",
  "bindings": [
    {
      "context": "Chat",
      "bindings": {
        "ctrl+alt+a": "chat:submit",
        "ctrl+k": null
      }
    }
  ]
}
```

### Rules
1. Bindings override defaults (last wins)
2. Null value unbinds a default binding
3. Command bindings format: "command:help", "command:compact"
4. Only available if tengu_keybinding_customization_release feature enabled

---

## REACT HOOK INTEGRATION

### useKeybinding(action, handler, options)
Single action binding with context and active state options

### useKeybindings(handlers, options)
Multiple actions in one hook call

### useShortcutDisplay(action, context, fallback)
Display configured shortcut or fallback text

### useRegisterKeybindingContext(context, isActive)
Register context priority while component mounted

---

## TELEMETRY EVENTS

### Logged Events
1. tengu_custom_keybindings_loaded
   - Fired once per day when user custom bindings loaded
   - Metric: user_binding_count

2. tengu_keybinding_fallback_used
   - Fired when display text lookup fails
   - Metrics: action, context, fallback, reason

---

## SUMMARY STATISTICS

- Total Contexts: 18
- Total Actions: 97+
- Total Default Bindings: ~220 (varies by features enabled)
- Platform-specific Paths: 2 (Windows vs Mac/Linux)
- Feature-gated Bindings: ~30
- Non-rebindable Keys: 3 (ctrl+c, ctrl+d, ctrl+m)
- Reserved (Warning) Keys: 7 additional OS/terminal keys
- vim Keybindings: 12+ across multiple contexts
- Files Analyzed: 14 (3,159 lines)

