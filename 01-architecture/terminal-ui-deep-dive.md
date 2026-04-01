# Claude Code Terminal UI Layer (Ink) - Deep Reverse Engineering Analysis

## Overview
The Ink UI layer (96 files, ~19,842 lines) is a custom React-based terminal rendering engine using Yoga layout engine. It handles all terminal I/O, rendering, event dispatch, and interactive UI.

---

## COMPONENT TREE HIERARCHY

### Root Structure
```
ink-root (DOMElement)
├── App (React Component, PureComponent)
│   ├── TerminalSizeContext.Provider
│   ├── AppContext.Provider
│   ├── StdinContext.Provider (raw mode control)
│   ├── TerminalFocusProvider (terminal focus state)
│   ├── ClockProvider (animation/interval timing)
│   ├── CursorDeclarationContext.Provider (IME/A11y)
│   └── ErrorOverview (error boundary fallback)
│
└── DOM Elements (host components)
    ├── ink-box (layout container, Yoga flex)
    ├── ink-text (text content, text styles)
    ├── ink-virtual-text (text in text context)
    ├── ink-link (OSC 8 hyperlink)
    ├── ink-progress (progress indicator)
    ├── ink-raw-ansi (pre-rendered ANSI, bypass Yoga)
    └── ink-root (window root)
```

### Key Components

**App.tsx** (850+ lines)
- Props: children, stdin, stdout, stderr, selection, mouse handlers, cursor declaration
- State: error (error boundary), rawModeEnabledCount, keyParseState, incomplete escape timer
- Input processing: handleReadable → parseMultipleKeypresses → reconciler.discreteUpdates
- Terminal control: EBP (bracketed paste), EFE (focus events), ENABLE_KITTY_KEYBOARD, ENABLE_MODIFY_OTHER_KEYS
- Multi-click tracking: lastClickTime, lastClickCol, lastClickRow, clickCount (500ms threshold, 1-cell distance)
- Suspension handling: handleSuspend → SIGSTOP, resumeHandler → restore raw mode
- Key features:
  - Keyboard input parser state machine (handles incomplete sequences)
  - Paste mode detection (PASTE_START/PASTE_END markers)
  - Terminal focus events (focus-in/focus-out via DECSET 1004)
  - Mouse event handling (SGR mouse, x10 mouse)
  - Ctrl+Z suspension support (both raw \x1a and CSI u format)

**ErrorOverview.tsx** (error boundary component)
- Renders error stack trace in red text
- Likely displays to stderr
- Confirms error handling UI

**Text.tsx** (Text component)
- Props: color, backgroundColor, bold, dim, italic, underline, strikethrough, inverse, wrap, children
- Converts properties to text styles
- Validates mutual exclusivity: bold XOR dim (terminal constraint)
- wrap modes: wrap, wrap-trim, end, middle, truncate, truncate-end, truncate-middle, truncate-start

**RawAnsi.tsx** (pre-rendered ANSI bypass)
- Props: lines (ANSI-escaped strings), width
- Skips Ansi → React tree → Yoga → re-serialize roundtrip
- Direct output.write() for pre-wrapped content (e.g., ColorDiff diffs)
- One Yoga leaf measure func: O(1) instead of O(span count)

**Box.tsx** (core layout container)
- Props: children, flexDirection, flexGrow, flexShrink, flexWrap, tabIndex, autoFocus
- Event handlers: onClick, onFocus, onFocusCapture, onBlur, onBlurCapture, onKeyDown, onKeyDownCapture, onMouseEnter, onMouseLeave
- Maps to Yoga flex container
- Focus management: tabIndex >= 0 participates in Tab/Shift+Tab cycling, -1 = programmatic only

**Link.tsx** (OSC 8 hyperlinks)
- Props: url, children, fallback
- Conditional: supportsHyperlinks() ? <ink-link href={url}> : fallback text
- Wraps URL in OSC 8 markers for terminal opening

---

## INPUT PIPELINE: KEYSTROKE FLOW

### Phase 1: Raw Terminal Input (stdin)
1. **App.handleReadable()** - stdin 'readable' event handler
   - Reads from stdin in loop: `stdin.read() → Buffer | string | null`
   - Gap detection: `now - lastStdinTime > 5000ms` → STDIN_RESUME_GAP_MS → onStdinResume() callback
   - Passes chunks to `processInput(chunk)`

### Phase 2: Input Parsing (parse-keypress.ts)
2. **parseMultipleKeypresses(prevState, input)**
   - State machine maintains: mode (NORMAL | IN_PASTE), incomplete, pasteBuffer, _tokenizer
   - Tokenizes input → creates Tokenizer instance (termio/tokenize.ts)
   - Token types:
     - `sequence`: escape sequences (CSI, OSC, DCS, etc.)
     - `text`: literal characters
   - **Paste handling**: PASTE_START (\x1b[200~) → mode = IN_PASTE, PASTE_END (\x1b[201~) → emit paste key
   - **Terminal responses**: parseTerminalResponse() recognizes:
     - DECRPM: CSI ? Ps ; Pm $ y (mode status)
     - DA1: CSI ? Ps ; ... c (device attributes)
     - DA2: CSI > Ps ; ... c (secondary DA)
     - KITTY_FLAGS: CSI ? flags u (Kitty keyboard response)
     - CURSOR_POSITION: CSI ? row ; col R (DSR reply)
     - OSC_RESPONSE: OSC code ; data (BEL | ST)
     - XTVERSION: DCS > | name ST (terminal identity)
   - **Mouse events**: parseMouseEvent() recognizes:
     - SGR mouse: CSI < button ; col ; row M|m
     - X10 mouse: ESC [ M cb cx cy (legacy, limited button info)
   - **Incomplete sequence handling**: if tokenizer.incomplete is truthy, set timer (NORMAL_TIMEOUT=50ms, PASTE_TIMEOUT=500ms)
   - Returns: [ParsedInput[], KeyParseState] where ParsedInput = {kind: 'key'|'response'|'mouse', ...}

3. **Escape Sequence Recognition** (parse-keypress.ts regexes)
   - CSI u (Kitty keyboard): `ESC [ codepoint [; modifier] u` → ParsedKey with ctrl/shift/alt/meta flags
   - ModifyOtherKeys (xterm): `ESC [ 27 ; modifier ; keycode ~` → ParsedKey
   - Function keys: `ESC (O|N|[|[[) ...` → F1-F24, Home, End, etc.
   - Meta key: `ESC + [a-zA-Z0-9]` → key with meta=true
   - Single chars → ParsedKey { name: '', sequence: char, raw: char }

### Phase 3: React Reconciler Update Batching
4. **App.processInput(input)**
   - Parses input → keys array
   - **CRITICAL**: Uses `reconciler.discreteUpdates(processKeysInBatch, app, keys)` to batch ALL keys in ONE React update
   - Prevents "Maximum update depth exceeded" on fast key repeat or paste
   - All listeners (useInput hooks) fire within same high-priority update context

5. **processKeysInBatch(app, items)** (App.tsx, ~100 lines)
   - Updates interaction time (only for 'key' and 'mouse' with button held, NOT mode-1003 passive motion)
   - Routes items:
     - `response`: app.querier.onResponse(item.response) → resolves pending promises
     - `mouse`: handleMouseEvent(app, item) → selection, clicks, hovers
     - `FOCUS_IN`/`FOCUS_OUT` → app.handleTerminalFocus(boolean)
     - `Ctrl+Z`: app.handleSuspend() → kills process with SIGSTOP
     - `Ctrl+C` (exitOnCtrlC): app.handleExit()
   - **Other keys**: routed through:
     - useInput listeners (component hooks)
     - dispatchKeyboardEvent() → DOM event dispatch (capture + bubble)

### Phase 4: Event Dispatch (events/dispatcher.ts)
6. **Dispatcher.dispatch(target, event)**
   - Collects listeners: walk from target to root, accumulate capture handlers (unshift/prepend) + bubble handlers (push)
   - Dispatch order: [root-capture, ..., parent-capture, target-capture, target-bubble, parent-bubble, ..., root-bubble]
   - Before each handler: event._prepareForTarget(node) (allows per-node setup)
   - Stoppage control:
     - event.stopPropagation() → stops bubble/capture at sibling level
     - event.stopImmediatePropagation() → stops everything
   - Error handling: try-catch around each handler, logError()

7. **Event Types** (events/event-handlers.ts)
   - keydown, focus, blur, paste, resize, click
   - onMouseEnter/onMouseLeave (not in HANDLER_FOR_EVENT mapping, probably hover-specific)
   - Capture variants: onKeyDownCapture, onFocusCapture, onBlurCapture, onPasteCapture

---

## PERMISSION/CONSENT UI LAYER

### No Explicit Permission Dialogs Found
The analyzed files do NOT contain typical permission dialogs (modals asking user consent).

### Implicit Permission via Terminal Capabilities
Instead, permissions are determined by terminal state:

1. **Mouse Tracking** (ink.tsx + App.tsx)
   - Controlled via ENABLE_MOUSE_TRACKING / DISABLE_MOUSE_TRACKING (termio/dec.ts)
   - Emitted when entering alt-screen or on stdin resume
   - Env gate: CLAUDE_CODE_DISABLE_MOUSE skips enable
   - No user dialog; terminal feature gates it

2. **Alt-Screen Mode** (ink.tsx: enterAlternateScreen / exitAlternateScreen)
   - enterAlternateScreen():
     - Calls pause() + suspendStdin()
     - Writes: DISABLE_KITTY_KEYBOARD, DISABLE_MODIFY_OTHER_KEYS, DISABLE_MOUSE_TRACKING, ENTER_ALT_SCREEN, DFE, reset attrs, show cursor, clear, home
     - Returns control to external editor (git commit, etc.)
     - Called from <AlternateScreen> component (not shown, likely in app code)
   - exitAlternateScreen():
     - Writes: re-enter alt if fullscreen, clear, home, re-enable mouse, hide cursor
     - Calls resumeStdin() + repaint()
     - Re-enables ENABLE_MODIFY_OTHER_KEYS + ENABLE_KITTY_KEYBOARD

3. **Terminal Mode Assertions** (App.tsx: handleSetRawMode)
   - Raw mode enable sequence: EBP (bracketed paste), EFE (focus events), ENABLE_KITTY_KEYBOARD, ENABLE_MODIFY_OTHER_KEYS
   - No user prompt; gates on stdin.isTTY and raw mode support check
   - Disable sequence: DISABLE_MODIFY_OTHER_KEYS, DISABLE_KITTY_KEYBOARD, DFE, DBP

4. **Clipboard via OSC 52** (termio/osc.ts: setClipboard)
   - getClipboardPath(): native | tmux-buffer | osc52
   - Native (pbcopy/wl-copy): SSH_CONNECTION check; only local
   - tmux: load-buffer with -w flag (requires allow-passthrough on)
   - OSC 52: base64-encoded clipboard, BEL or ST terminator
   - Fire-and-forget; no user interaction required

### Feature Gates (Environment Variables)
- CLAUDE_CODE_DISABLE_MOUSE: skips ENABLE_MOUSE_TRACKING write
- CLAUDE_CODE_DEBUG_REPAINTS: enables owner chain capture + debug logging
- CLAUDE_CODE_COMMIT_LOG: performance instrumentation file path
- CLAUDE_CODE_ACCESSIBILITY: skips HIDE_CURSOR on mount
- DEV (process.env.NODE_ENV === 'development'): attempts devtools connection
- TERM_PROGRAM: used for terminal detection (iTerm.app, ghostty, etc.)

---

## RENDERING SECURITY ANALYSIS

### ANSI Escape Sequence Handling (Critical)

**Input Sanitization on Rendering Path:**

1. **termio/parser.ts** - Semantic ANSI Parser
   - Tokenizes raw ANSI sequences → semantic actions
   - Parses: cursor moves, erase, scroll, SGR (colors/bold/italic), DECSET/DECRESET, etc.
   - **Color handling**: applySGR() validates color codes (0-255, rgb:r/g/b, hex #rrggbb)
   - **Cursor movement**: bounds-checked (CUU, CUD, CUF, CUB, CHA, CUP, VPA, etc.)
   - **No direct passthrough**: all sequences reinterpreted, unknown → ignored

2. **output.ts - Content Tokenization**
   - Uses @alcalzone/ansi-tokenize to parse styled text
   - tokenize(text) → [tokens] = {type: 'text'|'ansi', value: string}
   - styledCharsFromTokens(tokens) → Grapheme[] with style IDs
   - **Style interning** via StylePool: maps AnsiCode[] → unique ID
   - Per-frame caching: charCache[text] = ClusteredChar[] persists across renders

3. **screen.ts - Screen Buffer**
   - Screen is a 2D grid: Cell[] where Cell = {char: string, styleId: number, hyperlink?: string, width: CellWidth}
   - setCellAt(screen, x, y, text, styleId, hyperlink) validates bounds
   - CellWidth enum: Normal (1), Wide (2 for CJK/emoji), SpacerTail (continuation), SpacerHead (wrap overflow)
   - **Hyperlink interning**: HyperlinkPool.intern(url) → validates unique URLs, returns ID
   - All writes filtered through setCellAt → bounds clamp + wide-char handling

4. **render-node-to-output.ts - DOM to Screen**
   - renderNodeToOutput(node, output) walks DOM tree, emits write operations
   - Text nodes: wrapped in Ansi parser, segments pushed to output
   - Box nodes: clipped to bbox, children rendered recursively
   - RawAnsi nodes: lines.join('\n') written directly (assumes pre-sanitized)
   - Clipping: maintains clip stack [Clip, Clip, ...], intersects with each node's bounds
   - **Selection overlay**: applySelectionOverlay(screen, selection, stylePool) inverts style IDs post-render
   - **Search highlight**: applySearchHighlight(screen, query, stylePool) inverses matching cells

### ANSI Injection Attack Vectors

**Vulnerable Patterns Identified:**

1. **RawAnsi component** (RawAnsi.tsx)
   - Lines must be pre-escaped and width-wrapped by producer
   - No re-parsing; ANSI codes passed through directly
   - **Risk**: if user-controlled content rendered as RawAnsi, injection possible
   - Mitigation: producer (e.g., ColorDiff NAPI) responsible for sanitization

2. **Link component** (Link.tsx)
   - URL prop wrapped in OSC 8: `ESC ] 8 ;; {url} BEL {text} ESC ] 8 ;; BEL`
   - **Risk**: URL not escaped; if user input, could inject OSC sequences
   - Example attack: url = `bad.com BEL ESC ] 0 ; hacked ST` would change terminal title
   - Mitigation: AppContext or component consumer must validate URLs (not enforced here)

3. **Text content** (Text.tsx → output.ts)
   - All text parsed by Ansi parser → sequences re-interpreted
   - User input "hello \x1b[31m world" would be parsed, color applied, re-serialized
   - **NOT a vulnerability**: content styles are controlled, not attacker-chosen
   - Exception: if Text.color prop is user-controlled, stylePool.intern([]) could be gamed
     - But stylePool.intern only accepts AnsiCode[], no direct color injection

4. **Selection and Search Highlight** (selection.ts, searchHighlight.ts)
   - Both work post-render on screen buffer (already parsed)
   - No new ANSI parsing; only inverses existing style IDs
   - Safe: they operate on the semantic screen model, not raw text

### Prompt Injection via Content Rendering

**Scenario**: Claude renders a message containing `ESC ] 0 ; hacked! ST` (set terminal title)

**Current Path**:
1. Message text passed to Text component
2. Text → ink-text node → output.ts tokenize()
3. @alcalzone/ansi-tokenize parses sequence, recognizes OSC 0
4. applySGR/parseOSC validates code, emits action
5. **Semantic renderer** interprets action (title change):
   - OSC 0 is a **FORMATTING action, not a state change**
   - Emitted but NOT WRITTEN to terminal unless explicitly rendered
   - render-node-to-output and screen.ts don't write OSC back
6. Result: **Content OSC sequences are parsed but discarded**

**Exception - OSC 8 (Hyperlinks)**:
- Link component explicitly wraps URL in OSC 8
- If URL is user-controlled (e.g., markdown link from message), injection possible
- Example: `[click](data:///evil\x1bP\x1bSTY)` embeds commands in URL param
- **Mitigation needed**: validate URL scheme (http/https only), escape backslashes

### Terminal Title / Window Name Injection
- Possible if a component explicitly renders OSC 0 or OSC 1/2
- Not visible in analyzed code (no explicit OSC 0 write on render path)
- Could occur if custom component calls output.write() or emits OSC through RawAnsi
- **Control point**: output.write() takes Operation (write/blit/clear/clip), not raw text
- Escaping: none; relies on semantic layer

### ANSI Color/Attribute Injection
- **Safe**: SGR (colors, bold, italic, etc.) are semantic and fully validated
- stylePool.intern([AnsiCode, ...]) rejects invalid codes
- Unknown SGR codes silently ignored by parser

---

## STATE FLOW: UI STATE ↔ APPLICATION STATE

### Central State Holders

1. **Ink class** (ink.tsx, ~600 lines)
   - Owns FiberRoot, RootNode, Renderer
   - Manages: currentNode, frontFrame, backFrame, terminalColumns/Rows
   - Owns: selection (text selection state), searchHighlightQuery, searchPositions
   - Mouse state: hoveredNodes (Set), displayCursor (native cursor position)
   - Alt-screen state: altScreenActive, altScreenMouseTracking
   - Scroll state: drainTimer, (ScrollBox uses pendingScrollDelta)
   - Methods:
     - render(node): reconciler.root.render(node) → React commit
     - onRender(): renderer(frontFrame, backFrame) → Frame, writeDiffToTerminal()
     - onComputeLayout(): rootNode.yogaNode.calculateLayout()
     - Handlers: handleResize, handleResume, handleResize, handleSuspend
   - Subscriptions: stdout.on('resize'), process.on('SIGCONT')

2. **DOMElement** (dom.ts)
   - yogaNode: LayoutNode (from yoga-layout WASM binding)
   - dirty: boolean (marks for re-render)
   - scrollTop, pendingScrollDelta, scrollHeight, etc. (ScrollBox state)
   - style, textStyles, attributes (CSS-like styling)
   - _eventHandlers: {[key]: handler} (event dispatch cache)
   - focusManager: FocusManager (on root only)

3. **SelectionState** (selection.ts)
   - anchor, focus: Point | null (selection endpoints)
   - isDragging: boolean
   - anchorSpan: {lo, hi, kind: 'word'|'line'} | null (multi-click)
   - scrolledOffAbove/Below: string[] (rows scrolled during drag)
   - virtualAnchorRow, virtualFocusRow (pre-clamp positions)

4. **FocusManager** (focus.ts)
   - activeElement: DOMElement | null
   - focusStack: DOMElement[] (last 32 focused nodes)
   - enabled: boolean
   - Methods: focus(node), blur(), moveFocus(direction), handleAutoFocus, handleNodeRemoved

5. **React Context** (components/\*.tsx)
   - TerminalSizeContext: { columns, rows }
   - AppContext: { exit(error?) }
   - StdinContext: { stdin, setRawMode, isRawModeSupported, internal_eventEmitter, internal_querier }
   - TerminalFocusContext: { focused } (wraps terminal-focus-state.ts getter/setter)
   - ClockProvider: animation frame timer (not shown, likely in component code)
   - CursorDeclarationContext: setter function for native cursor position

### State Flow Diagram

```
stdin bytes
    ↓
App.handleReadable() → processInput() → parseMultipleKeypresses()
    ↓ (parsed keys)
reconciler.discreteUpdates(processKeysInBatch)
    ↓ (routes response/mouse/keyboard)
    ├─→ querier.onResponse() [terminal response]
    ├─→ handleMouseEvent() → selection state updates → onSelectionChange()
    ├─→ dispatchKeyboardEvent() → Dispatcher.dispatch() → DOM handlers
    └─→ useInput listeners → component state changes
         ↓
    reconciler.root.render(node) [React commit]
         ↓
    reconciler.resetAfterCommit()
         ↓
    rootNode.onComputeLayout() → Yoga.calculateLayout()
         ↓
    rootNode.onRender() → scheduleRender → microtask: onRender()
         ↓
    onRender():
         ├─→ renderer(frontFrame, backFrame) → Frame
         ├─→ applySelectionOverlay(screen, selection)
         ├─→ applySearchHighlight(screen, query)
         ├─→ writeDiffToTerminal(diff)
         └─→ swap: frontFrame ←→ backFrame
```

### Render Optimization: Double Buffering & Blitting

- **frontFrame**: last-written screen (in terminal)
- **backFrame**: current-computed screen
- **swap**: frontFrame = backFrame, backFrame = new empty screen
- **blit optimization**: if unchanged subtree, copy prevScreen pixels instead of recomputing
- **prevFrameContaminated**: selection overlay or alt-screen reset mutated screen post-render → disable blit next frame

---

## FEATURE-GATED UI ELEMENTS

### Conditional Rendering

1. **supportsHyperlinks()** (supports-hyperlinks.ts, not read but referenced in Link.tsx)
   - Link component: if true, render OSC 8; else fallback text
   - Checks: likely TERM_PROGRAM env var, terminal detection

2. **isProgressReportingAvailable()** (terminal.ts)
   - OSC 9;4 support: Ghostty 1.2.0+, iTerm2 3.6.6+, ConEmu (all versions)
   - Explicitly excludes: Windows Terminal (interprets OSC 9;4 as notifications)
   - Checks: TERM_PROGRAM + version, process.env.ConEmu*

3. **isSynchronizedOutputSupported()** (terminal.ts)
   - DEC 2026 (synchronized output) support: modern terminals
   - Excludes: tmux (breaks atomicity)
   - Returns: boolean for BSU/ESU wrapping decision
   - Used in log-update.ts: if true, wrap diffs in BSU/ESU for atomic updates

4. **supportsExtendedKeys()** (terminal.ts)
   - Kitty keyboard protocol (CSI >1u) + xterm modifyOtherKeys (CSI >4;2m)
   - Allowlist: iTerm.app, kitty, WezTerm, ghostty, tmux, windows-terminal
   - Used in App.tsx: if true, emit ENABLE_KITTY_KEYBOARD + ENABLE_MODIFY_OTHER_KEYS

5. **isXtermJs()** (terminal.ts)
   - xterm.js-based terminals (VS Code, Cursor, Windsurf integrated terminals)
   - Fast check: TERM_PROGRAM === 'vscode'
   - Async probe: XTVERSION query (CSI > 0 q) → DCS > | name ST
   - Used in render-node-to-output.ts: detect wheel scroll behavior

6. **hasCursorUpViewportYankBug()** (terminal.ts)
   - Windows (platform === 'win32' or WT_SESSION) cursor-up yanks viewport
   - Affects scroll behavior

### Hidden/Undocumented UI Features

1. **PASTE Mode** (parse-keypress.ts)
   - PASTE_START (\x1b[200~) → IN_PASTE mode
   - Content buffered until PASTE_END (\x1b[201~)
   - Paste key emitted with isPasted=true flag
   - Incomplete sequence handling: 500ms timeout (vs 50ms normal)
   - Allows bulk text handling without line-by-line processing

2. **Terminal Query/Response Loop** (App.tsx, parse-keypress.ts)
   - Querier: send(sequence) → returns Promise<response>
   - Queries: XTVERSION, DA1, DA2, DECRQM (mode status), DSR (cursor position)
   - Responses parsed and routed to querier.onResponse()
   - Timeout: flush() resolves after 100ms, letting queries fail gracefully
   - **Feature**: terminal identity detection survives SSH

3. **Multi-Click Modes** (App.tsx, App.handleMouseEvent not shown but referenced)
   - Single-click: startSelection()
   - Double-click (count=2): selectWordAt() → select word, set isDragging=true
   - Triple-click (count=3): selectLineAt() → select line
   - 500ms window, 1-cell distance tolerance
   - Hyperlink deferral: pending timer cancelled if second click within timeout

4. **Scroll Animation** (render-node-to-output.ts)
   - Terminal-specific drain strategies:
     - Native (iTerm2, Ghostty): proportional drain ~3/4 per frame
     - xterm.js (VS Code): adaptive drain (instant ≤5, step 2-3 for faster, cap 30)
   - DECSTBM optimization: if scrollTop changed only, emit CSI top;bot r + CSI n S/T (hardware scroll)
   - Reduces full-screen rewrites to 2-3 escape sequences

5. **Text Selection Copy** (selection.ts: getSelectedText)
   - Extracts text from screen buffer (cells)
   - Handles word/line mode boundaries
   - Joins scrolledOffAbove + viewport + scrolledOffBelow
   - Respects softWrap bits (continuations) to reconstruct logical lines
   - Copies to clipboard via setClipboard() with one of: native, tmux-buffer, osc52

6. **Focus Stack** (focus.ts)
   - Shift+Tab cycles backward through focused nodes
   - Stack keeps last 32 focused elements
   - Dedup: if already in stack, remove then push (prevents unbounded growth)
   - Auto-focus on mount (autoFocus prop)

7. **Search Highlight Current Match** (screen.ts: withCurrentMatch)
   - Base inverse + bold + yellow-bg via fg-swap (unambiguous on any theme)
   - Filters fg + bg codes to avoid ambiguity (inverse semantics)
   - OTHER matches: plain inverse only
   - **Undocumented**: positioned highlight via VML (not in analyzed code)

---

## ERROR HANDLING IN UI LAYER

### Error Boundaries

1. **App (PureComponent)**
   - getDerivedStateFromError(error) → return { error }
   - componentDidCatch(error) → this.handleExit(error)
   - Renders: ErrorOverview (red error stack) if state.error set

### Error Recovery

1. **handleReadable() exception handling** (App.tsx)
   - Try-catch around stdin.read() loop
   - Bun quirk: uncaught throw in 'readable' handler wedges stream
   - Re-attaches listener if lost: `stdin.listeners('readable').includes(this.handleReadable)`
   - logError() for observability

2. **Dispatcher exception handling** (events/dispatcher.ts)
   - Try-catch around each handler invocation
   - logError() on exception
   - Continues dispatch (doesn't stop on handler error)

3. **Yoga exception handling** (reconciler.ts, renderer.ts)
   - onComputeLayout() wrapped with isUnmounted guard
   - Invalid dimensions (NaN, negative, Infinity) → return empty frame
   - logForDebugging() for unrecognized patterns

### Debug Instrumentation

- CLAUDE_CODE_COMMIT_LOG file: logs slow reconcile/yoga/paint phases
- CLAUDE_CODE_DEBUG_REPAINTS: captures owner chain for each node, debugOwnerChain on DOMElement
- isDebugRepaintsEnabled(): gated by getOwnerChain() call in createInstance
- logForDebugging(): writes to stdout if --debug flag (not analyzed here)

---

## OUTPUT FORMATTING PIPELINE

### Path from Component to Terminal

```
Component text prop
    ↓
Text.tsx → ink-text DOM node
    ↓
dom.ts createNode('ink-text')
    ↓
reconciler: createTextInstance(text) → TextNode
    ↓
render-node-to-output.ts: renderNodeToOutput()
    ├─→ squashTextNodesToSegments(): coalesce adjacent text nodes
    ├─→ output.ts: write(x, y, text, softWrap)
    ├─→ tokenize(text): @alcalzone/ansi-tokenize
    ├─→ styledCharsFromTokens(tokens): Grapheme[]{value, width, styleId}
    ├─→ setCellAt(screen, x, y, char, styleId, hyperlink)
    └─→ screen.ts: writes to 2D Cell grid
         ↓
renderer.ts: creates Frame with screen
    ↓
log-update.ts: render(prevFrame, nextFrame)
    ├─→ diffEach(prevScreen, nextScreen): O(changed cells)
    └─→ writeDiffToTerminal(diff)
         ├─→ patches: [Diff, ...] where Diff = {type: 'stdout'|'cursorShow'|'cursorMove'|'carriageReturn', ...}
         ├─→ BSU/ESU wrapping (if SYNC_OUTPUT_SUPPORTED)
         ├─→ stdout.write(patch.content)
         └─→ terminal renders
```

### Markdown/Code Rendering

- Not in analyzed Ink layer; handled by application code (likely in main app tree)
- Ink provides: Text, Box, Link, RawAnsi components
- RawAnsi used for pre-rendered content (ColorDiff, syntax highlighting, diffs)

### Streaming Output

- No explicit streaming in Ink; React drives updates on reconciliation
- High-frequency renders: scheduled via reconciler.discreteUpdates() → onRender throttle
- Throttle: FRAME_INTERVAL_MS (~16ms for 60fps) via lodash throttle
- Trailing: true to render after user stops interacting

---

## UNDOCUMENTED UI FEATURES & EASTER EGGS

1. **Terminal Identity Detection** (terminal.ts, App.tsx)
   - XTVERSION query: identifies terminal via DCS reply
   - Survives SSH (unlike TERM_PROGRAM env)
   - Async, fire-and-forget (Promise.all with 100ms timeout)
   - Used for wheel scroll heuristics in xterm.js detection

2. **Bracketed Paste Mode** (App.tsx: EBP)
   - When raw mode enabled, emit EBP (CSI ? 2004 h)
   - Terminal wraps pasted text in \x1b[200~ ... \x1b[201~
   - parse-keypress.ts recognizes markers, buffers content, emits one paste key
   - Prevents line-by-line interpretation of bulk input

3. **Terminal Focus Events** (App.tsx: EFE, FOCUS_IN/FOCUS_OUT)
   - DECSET 1004 enables focus reporting
   - Terminal emits \x1b I (focus-in) and \x1b O (focus-out) on focus changes
   - App.handleTerminalFocus(boolean) sets global state
   - Used by Clock (slow down on unfocus), maybe other components

4. **Clipboard Integration** (termio/osc.ts: setClipboard)
   - Tries: native (pbcopy) → tmux load-buffer -w → OSC 52
   - pbcopy fires async, fire-and-forget (race-safe: starts before await tmux)
   - Fallback chain: if any method fails, next is tried
   - No user dialog; all silent

5. **Lazy XTVERSION Probe** (App.tsx)
   - Deferred to setImmediate (next macrotask) so raw mode init sequence completes first
   - Avoids interleaving with alt-screen/mouse tracking enables
   - Fire-and-forget; Promise.all resolves after 100ms or reply

6. **Mode-1003 Motion** (App.tsx, selection.ts)
   - Terminal sends every mouse motion (no button held)
   - App dedupes: lastHoverCol, lastHoverRow
   - Triggers dispatchHover() → hit-test → onMouseEnter/onMouseLeave
   - Used for link underlines, button highlights (not verified in analyzed code)
   - Not counted as user interaction (doesn't update lastStdinTime)

---

## CROSS-SYSTEM DEPENDENCIES

### External Libraries

- **react-reconciler**: React rendering engine (host config at reconciler.ts)
- **@alcalzone/ansi-tokenize**: ANSI escape sequence tokenization + GraphemeSegmenter
- **yoga-layout**: Flexbox layout engine (WASM binding, yoga.ts wraps native module)
- **lodash-es**: throttle, noop, auto-bind
- **signal-exit**: onExit hook for cleanup

### Terminal I/O

- **stdout**: write(string) for terminal updates
- **stdin**: read() loop for keyboard input (raw mode via setRawMode)
- **stderr**: write(string) for error messages (patchStderr redirects console.error)
- **TTY checks**: stdout.isTTY, stdin.isTTY, supportsHyperlinks(), etc.

### Process

- **process.on('SIGCONT')**: resume from suspension
- **process.on('resize')**: stdout 'resize' event
- **process.kill(pid, 'SIGSTOP')**: suspend on Ctrl+Z
- **process.env**: TERM_PROGRAM, TMUX, SSH_CONNECTION, etc.

### File System

- **fs**: openSync/readSync/writeSync for CLIPBOARD_WRITE_FD (not shown)
- **execFile**: tmux load-buffer, native clipboard tools (osutils)

---

## TECHNICAL DEBT & TODOs

### Code Comments Indicating Technical Debt

1. **renderer.ts line 97** - Alt-screen bug: if sibling renders outside <AlternateScreen>, yogaHeight > terminalRows
   - Clamping enforces invariant, but root cause should be fixed
   - Comment: "something is rendering outside <AlternateScreen>. Overflow clipped."

2. **ink.tsx line 364** - Mouse tracking disabled post-exit: `DISABLE_MOUSE_TRACKING: @ts-expect-error`
   - Indicates type mismatch or intentional bypass
   - Sequence emission order matters for cursor/attribute reset

3. **reconciler.ts line 261** - React reconciler version mismatch
   - @types/react-reconciler@0.32.3 expects 11 args, actual 0.33.0 only 10
   - Workaround: @ts-expect-error, noop callbacks

4. **App.tsx line 294** - Backframe unused after handleResume
   - Comment: "start fresh to prevent clobbering terminal content"
   - Suggests historical issue where stale state corrupted terminal

### Missing Features

1. **No explicit permission dialogs** for:
   - Clipboard access (OSC 52)
   - Mouse tracking
   - Terminal mode changes (raw mode, extended keys)
   - Alt-screen mode
   - **Design**: implicit via terminal capabilities, not user interaction

2. **No rate limiting** on:
   - Render throttle is FRAME_INTERVAL_MS (~16ms), not per-event
   - Input parser has 50ms incomplete timeout, but no per-second limit on key events
   - Mouse events: mode-1003 can fire every motion pixel

3. **No undo/redo** in text selection

4. **No search navigation UI** for "prev/next match" (positions managed, but no buttons/hotkeys visible)

---

## SECURITY SUMMARY

### Attack Surface

1. **ANSI Injection through Text Content**: LOW RISK
   - All text parsed by @alcalzone/ansi-tokenize → semantic interpreter
   - Unknown sequences ignored; OSC/CSI/DCS codes re-interpreted, not blind-passed through
   - Exception: RawAnsi component (producer-dependent)

2. **Prompt Injection via URLs**: MEDIUM RISK
   - Link component wraps URL in OSC 8 without escaping
   - If URL user-controlled (markdown link), attacker can inject OSC sequences
   - Mitigation: validate scheme (http/https), escape special chars

3. **Selection/Copy Information Disclosure**: LOW RISK
   - Selection state owned by Ink, not exposed to application
   - getSelectedText() reads screen buffer only
   - Copy goes to system clipboard (OSC 52, tmux, native) — transparent

4. **Focus Hijacking**: LOW RISK
   - FocusManager.focus() gates on enabled flag
   - No programmatic route to steal focus from user input
   - tabIndex >= 0 tabbable nodes publicly declared

5. **Mouse Event Spoofing**: LOW RISK
   - Mouse events from terminal stdin only (mode-1003 SGR)
   - Can't be synthesized by application; terminal emulator sends them
   - Selection/click handlers validate bounds

### Recommendations

1. **Validate Link URLs**: scheme whitelist (http/https), reject data:/javascript:
2. **RawAnsi Producer Audit**: ensure producer sanitizes ANSI codes before passing to component
3. **OSC 8 ID Parameter**: generate random UUID for hyperlink ID, not user-controllable
4. **Terminal Feature Detection**: cache XTVERSION result, don't re-query every frame
5. **Clipboard Fallback Order**: native → tmux → OSC 52 (current) is good; document silent failures

---

## FILES ANALYZED

### Core Rendering (15 files)
- ink.tsx (630 lines) - Main Ink class, frame management, event routing
- renderer.ts (180 lines) - Renderer function, yoga layout
- reconciler.ts (510 lines) - React reconciler host config
- output.ts (400 lines) - Content tokenization, screen buffer operations
- screen.ts (300 lines) - 2D cell grid, style/hyperlink pools
- render-node-to-output.ts (600 lines) - DOM tree to screen rendering
- log-update.ts (300 lines) - Diff-based terminal updates
- render-to-screen.ts (200 lines) - Search/positioned highlight overlay
- selection.ts (200 lines) - Text selection state machine

### Input & Events (10 files)
- parse-keypress.ts (300 lines) - Keyboard input parsing
- components/App.tsx (600 lines) - Root component, input handling
- events/dispatcher.ts (230 lines) - Event dispatch (capture/bubble)
- events/event-handlers.ts (70 lines) - Event handler registry
- terminal-focus-state.ts (50 lines) - Focus tracking
- terminal-querier.ts (not analyzed, referenced)
- focus.ts (150 lines) - Focus manager

### UI Components (10 files)
- components/Text.tsx (200 lines) - Text styling
- components/Box.tsx (300 lines) - Layout container
- components/Link.tsx (50 lines) - OSC 8 hyperlinks
- components/RawAnsi.tsx (50 lines) - Pre-rendered ANSI bypass
- components/Button.tsx (500+ lines, not fully read)
- components/ErrorOverview.tsx (300+ lines, partially read)
- components/Ansi.tsx (not found, likely compiled)
- components/AlternateScreen.tsx (not analyzed)
- components/ScrollBox.tsx (not analyzed)

### Supporting Infrastructure (20+ files)
- dom.ts (250 lines) - DOM node creation, tree manipulation
- termio/parser.ts (200 lines) - Semantic ANSI parser
- termio/osc.ts (300+ lines) - OSC sequences, clipboard
- termio/tokenize.ts (not analyzed)
- terminal.ts (200 lines) - Terminal capability detection
- focus.ts (150 lines) - Focus manager
- stringWidth.ts, widest-line.ts, wrap-text.ts - Text measurement
- styles.ts, colorize.ts - Style application
- searchHighlight.ts (100 lines) - Search overlay

### Terminal Control (5+ files)
- termio/csi.ts (not analyzed) - CSI sequences
- termio/dec.ts (not analyzed) - DEC private mode
- termio/sgr.ts (not analyzed) - SGR color codes
- termio/types.ts (not analyzed) - Type definitions
- terminal.ts - Terminal detection

---

## TOTAL FILES & LOC
- **Files**: 96
- **Lines**: ~19,842
- **Core/Critical**: 25 files (60% of analysis)
- **Supporting**: 71 files (40%, many utility/helper functions)

---

## CONCLUSION

Claude Code's Ink UI layer is a **production-grade terminal UI framework** with:
- **Robust input handling**: state machine parsing, paste mode, incomplete sequence recovery
- **Safe content rendering**: semantic ANSI interpretation, style pooling, bounds checking
- **Advanced layout**: Yoga flex engine, virtual scrolling, sticky scroll
- **Interactive features**: text selection, search highlight, focus management, mouse tracking
- **Terminal adaptation**: capability detection, fallback chains, SSH-resilient probing
- **Performance optimization**: double-buffering, blitting, throttling, caching

**Security posture**: Generally good, with few injection vectors. Main risk is URL validation in Link component. RawAnsi component trusts producer.

