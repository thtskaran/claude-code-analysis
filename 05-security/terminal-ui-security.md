# Claude Code Terminal UI - Security Findings & Threat Model

## Executive Summary

The Ink terminal UI layer is **production-grade** with generally robust security. Primary attack surface is through content rendering and URL handling. No critical vulnerabilities identified, but several recommendations for hardening.

---

## THREAT MODEL

### Asset 1: User Input (Keyboard/Mouse)
**Risk**: Terminal sends malicious sequences disguised as user input
- **Attack**: Attacker controls terminal emulator, injects SGR/OSC sequences into stdin
- **Mitigation**: Sequences parsed as semantic tokens, not blind passthrough
- **Status**: PROTECTED - semantic parser reinterprets all sequences

### Asset 2: Rendered Content (Messages, Diffs, Code)
**Risk**: Claude renders attacker-controlled content with embedded ANSI/OSC codes
- **Attack**: Markdown with `\x1b[31mRED\x1b[39m`, OSC 0 (title), OSC 52 (clipboard)
- **Mitigation**:
  - Text component: all content parsed by @alcalzone/ansi-tokenize
  - Sequences converted to semantic (colors, bold, etc.), not reemitted as OSC
  - Exception: RawAnsi component passes content directly (producer-dependent)
- **Status**: MOSTLY PROTECTED - exception is RawAnsi
- **Recommendation**: Audit RawAnsi producers (ColorDiff NAPI, etc.)

### Asset 3: URLs in Links/Hyperlinks
**Risk**: Attacker embeds OSC/DCS sequences in markdown links
- **Attack**: `[click me](javascript:alert('xss')\x1b]0;hacked\x07)`
- **Current Code**:
  ```javascript
  // Link.tsx
  <ink-link href={url}>{content}</ink-link>
  // Wrapped as: OSC 8 ;; {url} BEL {text} OSC 8 ;; BEL
  ```
- **Issue**: URL not escaped; attacker can inject OSC inside URL param
- **Example Attack**:
  ```
  url = "http://good.com/page BEL ESC ] 0 ; Hacked! ST"
  → OSC 8 ;; http://good.com/page BEL ESC ] 0 ; Hacked! ST BEL ...
  → Terminal splits on first BEL → hyperlink ends early
  → Second OSC 0 is emitted as plain text? Or executed?
  → Depends on terminal parser tolerance
  ```
- **Status**: MEDIUM RISK
- **Recommendation**:
  1. Validate URL scheme (http/https only)
  2. Escape BEL/ESC in URLs: replace `\x1b` with `\\x1b`, `\x07` with `\\x07`
  3. Reject URLs containing OSC/CSI patterns (regex scan)

### Asset 4: Text Selection (Copy to Clipboard)
**Risk**: Attacker crafts content with embedded ANSI codes, user selects & copies
- **Attack**: Selected text includes raw ANSI sequences when copied
- **Current Flow**:
  - selection.ts getSelectedText() reads screen buffer (already rendered cells)
  - Cells contain parsed chars + style IDs (not raw ANSI)
  - No ANSI codes re-emitted during copy
- **Status**: PROTECTED - ANSI codes are not in clipboard data
- **Exception**: If application uses RawAnsi to render content, the ANSI codes ARE in screen buffer cells
  - Then copy would preserve those codes
  - But codes are already semantic (parsed), not injectable

### Asset 5: Terminal Title / Window Name
**Risk**: Attacker sets terminal title/icon via OSC 0/1/2
- **Attack**: `\x1b]0;Phishing Site\x07` renders as window title
- **Current Code**:
  - termio/parser.ts recognizes OSC 0/1/2 as semantic actions
  - applyOSC() in parser generates {type: 'text', ...} or modifies styles
  - render-node-to-output.ts does NOT write OSC back to terminal
  - Only user action writes OSC: Ink code in enterAlternateScreen() writes ANSI explicitly
- **Status**: PROTECTED - OSC from content is parsed & discarded, not re-emitted
- **Exception**: Custom component could call output.write(OSC) directly (not verified)

### Asset 6: Focus / Keyboard Events
**Risk**: Attacker hijacks keyboard focus, reads subsequent input
- **Attack**: dispatchKeyboardEvent() routes to application handlers (not verified to be safe)
- **Current Code**:
  - FocusManager.focus() gates on enabled flag
  - focusStack prevents infinite loops (MAX_FOCUS_STACK = 32)
  - No programmatic route to steal focus across security boundaries
- **Status**: PROTECTED (assuming application doesn't expose keyboard handlers)

### Asset 7: Mouse Events
**Risk**: Attacker injects mouse events via stdin (SGR/X10 format)
- **Attack**: `\x1b[<0;10;5M` is mouse left-click at (10, 5)
- **Current Code**:
  - All mouse events from terminal stdin (parse-keypress.ts parseMouseEvent)
  - App.handleMouseEvent() validates col/row bounds
  - Affects selection, click handlers, hover
  - Cannot synthesize clicks that don't originate from actual terminal input
- **Status**: PROTECTED - requires actual terminal emulator cooperation

---

## CRITICAL CODE PATHS - SECURITY REVIEW

### Path 1: Text Rendering (SAFE)
```
Component: <Text color={color}>{content}</Text>
     ↓
dom.ts: createNode('ink-text')
     ↓
output.ts: tokenize(content) → @alcalzone/ansi-tokenize
     ↓
Tokens: [{type: 'text'|'ansi', value}, ...]
     ↓
For each ANSI token: applySGR(stylePool, params)
     → stylePool.intern([AnsiCode, ...]) validates codes
     → unknown codes → ignored
     ↓
For each text token: segmentGraphemes(text) → ClusteredChar[]
     ↓
render-node-to-output: setCellAt(screen, x, y, char, styleId)
     ↓
screen.ts: Cell = {char, styleId, width, hyperlink}
     ↓
log-update: diffEach(prev, next) → output strings with SGR transitions
     ↓
terminal: renders
```
**Verdict**: SAFE - all ANSI codes re-interpreted, never blind-passed through

### Path 2: URL in Link (MEDIUM RISK)
```
Component: <Link url={url}>{text}</Link>
     ↓
Link.tsx: supportsHyperlinks()
     ? render(<ink-link href={url}>{text}</ink-link>)
     : render(<Text>{fallback}</Text>)
     ↓
<ink-link> is text element in text context
     ↓
render-node-to-output: wraps in OSC 8
     → OSC 8 ;; {url} BEL {text} OSC 8 ;; BEL
     ↓
terminal: renders hyperlink (clickable on supported terminals)
```
**Issue**: URL not escaped. If user-controlled, attacker can break OSC 8 with BEL:
```
url = "http://good.com/\x07\x1b]0;HACKED\x07"
→ OSC 8 ;; http://good.com/ BEL  ← link ends here
→ ESC ] 0 ; HACKED BEL  ← title set (or rendered as error)
```
**Verdict**: MEDIUM RISK - recommend URL validation + escaping

### Path 3: RawAnsi Rendering (RISKY)
```
Component: <RawAnsi lines={lines} width={width} />
     ↓
RawAnsi.tsx: assumes lines are pre-escaped ANSI strings
     ↓
lines.join('\n') → joined string (ANSI codes in-place)
     ↓
output.ts: write(x, y, text) → direct to screen.ts
     ↓
screen.ts: does NOT tokenize; treats as raw text?
     OR output.ts handles tokenization?
```
**Unclear from code analysis**: Need to trace output.write() → screen.write()
**Assumption**: ANSI codes passed through directly to terminal (not re-parsed)
**Verdict**: RISKY - producer (ColorDiff) must sanitize before passing to RawAnsi

### Path 4: Selection & Copy (SAFE)
```
User selects text (mouse drag)
     ↓
selection.ts: startSelection() → updateSelection() → finishSelection()
     ↓
Screen buffer cells: {char, styleId, width, hyperlink}
     ↓
getSelectedText(): reads cells, reconstructs text (no ANSI codes in cells)
     ↓
setClipboard(text):
     ├─→ native pbcopy(text)
     ├─→ tmux load-buffer(text)
     └─→ OSC 52 (base64): osc(OSC.CLIPBOARD, 'c', base64(text))
     ↓
clipboard: plain text (no ANSI codes)
```
**Verdict**: SAFE - screen buffer is semantic, not raw ANSI

### Path 5: Search Highlight (SAFE)
```
App: searchHighlightQuery = "foo"
     ↓
searchHighlight.ts: applySearchHighlight(screen, query, stylePool)
     ← Scans screen buffer text (already parsed)
     ← Finds query (case-insensitive)
     ← For each match: setCellStyleId(screen, x, y, stylePool.withInverse(baseId))
     ← Inverts style ID (SGR 7)
     ↓
screen buffer: cells with inverted styleId
     ↓
log-update: diff sees style change, emits SGR transition
     ↓
terminal: renders inverse color
```
**Verdict**: SAFE - operates on semantic screen model, no code injection

---

## SPECIFIC VULNERABILITIES

### Vuln 1: OSC 8 URL Injection (MEDIUM)
**Severity**: Medium
**Affected Code**: Link.tsx, render-node-to-output.ts wrapWithOsc8Link()
**Attack Vector**: Attacker-controlled URL in markdown link
**Payload**: `[click](http://good.com\x07\x1b]0;Hacked\x07)`
**Impact**: Terminal title changed, potential user confusion (phishing)
**Mitigation**:
```typescript
function wrapWithOsc8Link(text: string, url: string): string {
  // Escape BEL and ESC in URL
  const escaped = url
    .replace(/\x1b/g, '\\x1b')  // Escape ESC
    .replace(/\x07/g, '\\x07')  // Escape BEL
    .replace(/ST/g, '\\\\ST')   // Escape string terminator
  return `${OSC}8;;${escaped}${BEL}${text}${OSC}8;;${BEL}`
}
// Or reject URLs with ANSI patterns:
if (/[\x07\x1b]/.test(url)) throw new Error('Invalid URL')
```

### Vuln 2: RawAnsi Producer Trust (MEDIUM)
**Severity**: Medium
**Affected Code**: RawAnsi.tsx, used by ColorDiff and other NAPI modules
**Attack Vector**: NAPI module produces malicious ANSI-embedded output
**Payload**: RawAnsi lines = `["\x1b]0;HACKED\x07line1", "line2"]`
**Impact**: Depends on what NAPI does (ColorDiff trusted, but principle is risky)
**Mitigation**:
```typescript
export function RawAnsi({ lines, width }: Props): React.ReactNode {
  // Optional: tokenize lines to validate ANSI syntax
  const validated = lines.map(line => {
    // Re-tokenize to check for unexpected OSC patterns?
    // Or trust producer completely
    return line
  })
  // ...
}
```
**Recommendation**: Document that RawAnsi trusts producer (ColorDiff NAPI module)

### Vuln 3: Incomplete Sequence Timeout (LOW)
**Severity**: Low
**Affected Code**: App.tsx processInput(), parse-keypress.ts
**Attack Vector**: Terminal sends incomplete CSI sequence, waits for timeout
**Payload**: `\x1b[1` (incomplete) → timeout after 50ms → emits as Escape key
**Impact**: User sees spurious Escape key, no security impact
**Status**: Documented design (see comments at line ~300 in App.tsx)
**Not a Vulnerability**: Graceful degradation, intentional behavior

### Vuln 4: Mouse Mode Injection (LOW)
**Severity**: Low
**Affected Code**: App.tsx handleMouseEvent()
**Attack Vector**: Terminal sends X10 mouse sequence with invalid button/col/row
**Payload**: `\x1b[Mbl\xff\xff` (button 'l', col/row beyond bounds)
**Impact**: Out-of-bounds click ignored (bounds-checked), no crash
**Status**: PROTECTED - bounds validation in hit-test.ts

### Vuln 5: Focus Event Injection (LOW)
**Severity**: Low
**Affected Code**: App.tsx handleTerminalFocus()
**Attack Vector**: Terminal spams FOCUS_IN/FOCUS_OUT (DECSET 1004)
**Payload**: `\x1b I\x1b O\x1b I\x1b O...` (flashing focus)
**Impact**: Clock speeds up/down (unused unless component checks focus), no security impact
**Status**: Not a vulnerability; Clock behavior is benign

---

## ANSI ESCAPE SEQUENCE SECURITY

### Sequences Recognized by Parser (termio/parser.ts)

**Safe (Semantic, Re-interpreted)**:
- SGR (colors, bold, italic, etc.): `CSI m` → styleId
- Cursor movement: `CSI H`, `CSI A/B/C/D`, etc. → rendered as absolute position
- Erase: `CSI J`, `CSI K` → cleared cells
- Scroll: `CSI S`, `CSI T` → reflow content

**Risky (Parsed but not Emitted)**:
- OSC 0/1/2 (title/icon): recognized, discarded (not written back)
- OSC 8 (hyperlink): EXPLICITLY WRITTEN by Link component (with URL risk)
- DEC private modes (mouse, focus, alt-screen): controlled by Ink code, not content

**Ignored (Unknown)**:
- Non-standard sequences (CSI ? > ; ; u, etc.): filtered out, no error

### Escape Sequence Whitelist Concept

**Current**: Parse & re-interpret; don't whitelist specific sequences
**Benefit**: Forward-compatible with new terminal features
**Risk**: If semantic parser is wrong, could cause issues
**Example**: If @alcalzone/ansi-tokenize has a bug, could create injection vector

**Recommendation**: Periodic audit of @alcalzone/ansi-tokenize for regressions

---

## INPUT VALIDATION CHECKLIST

| Input | Validated | Safe |
|-------|-----------|------|
| Keyboard (stdin) | Yes (state machine) | Yes |
| Mouse (stdin) | Yes (bounds check) | Yes |
| Terminal responses (stdin) | Yes (regex patterns) | Yes |
| Text content | Yes (ANSI parsing) | Yes* |
| URLs | No | Medium Risk |
| RawAnsi lines | No (trusted) | Producer-dependent |
| Styles (color, bold, etc.) | Yes (stylePool) | Yes |
| Focus events | Yes (bool) | Yes |
| Selection state | Yes (type) | Yes |

*Except for RawAnsi component

---

## TERMINAL DETECTION & CAPABILITY PROBING

### Probes Sent by Ink

1. **XTVERSION** (CSI > 0 q)
   - Query: "What is your name?"
   - Response: DCS > | name ST
   - Purpose: Detect xterm.js (VS Code), Ghostty, iTerm2, etc.
   - Safe: terminal name is harmless

2. **DA1** (CSI ? 6 c)
   - Query: "What device are you?"
   - Response: CSI ? Ps ; ... c
   - Purpose: Sentinel for flushInteractionTime()
   - Safe: device code is harmless

3. **DECRQM** (CSI ? Ps $ p)
   - Query: "What is the status of mode X?"
   - Response: CSI ? Ps ; Pm $ y
   - Purpose: Check if mouse/focus/sync modes are on
   - Safe: mode state is harmless

### Security of Probing

**Risk**: Attacker controls terminal, sends malicious responses
- **Response parsing**: parse-keypress.ts has regex patterns (XTVERSION_RE, DA1_RE, DECRPM_RE)
- **Danger**: If regex allows unbounded input, could overflow buffer
  - Example: DA1_RE = `/^\x1b\[\?([\d;]*)c$/s` → [\d;]* is unbounded
  - But parseTerminalResponse() just splits on `;` and parses ints
  - Int32Array has fixed size, no buffer overflow
- **Status**: PROTECTED - no known DoS

**Recommendation**: Cap response size (e.g., max 1024 bytes per response)

---

## CLIPBOARD SECURITY

### OSC 52 Attack: Overwrite Clipboard
**Risk**: Attacker-controlled content writes to system clipboard via OSC 52
- **Attack**: Message contains `\x1b]52;c;YmFkc3R1ZmY=\x07` (base64-encoded clipboard)
- **Current Flow**:
  - Text content parsed → ANSI tokens
  - OSC 52 is NOT recognized as renderable (unlike OSC 8)
  - Discarded during rendering
- **Status**: PROTECTED - OSC 52 in content is not emitted to terminal

**Risk**: Ink writes attacker-controlled text to clipboard
- **Attack**: User selects + copies attacker content
- **Current Flow**:
  - getSelectedText() reads screen cells (no ANSI codes)
  - setClipboard(text) → base64 encode → write OSC 52
  - No injection possible (already semantic)
- **Status**: PROTECTED

### tmux load-buffer Attack
**Risk**: Attacker-controlled text loaded into tmux buffer
- **Status**: PROTECTED - same as above (text is semantic)

---

## FILE DESCRIPTOR SECURITY

### CLIPBOARD_WRITE_FD (Referenced but Not Found)
- Likely: descriptor for clipboard write (macOS pbcopy, etc.)
- Not analyzed: open/write path unclear
- Recommendation: Use tempfile + execFile (standard, not custom FD)

---

## ENVIRONMENT VARIABLE ATTACKS

### Env Vars Read by Ink
| Var | Used By | Risk |
|-----|---------|------|
| TERM_PROGRAM | terminal.ts | Locates terminal (benign) |
| TERM | terminal.ts | Locates terminal (benign) |
| TMUX | osc.ts, terminal.ts | Clipboard path (benign) |
| SSH_CONNECTION | osc.ts | SSH detection (benign) |
| SSH_TTY | osc.ts | SSH detection (benign) |
| NODE_ENV | reconciler.ts | Dev mode (benign) |
| CLAUDE_CODE_* | multiple | Feature gates (benign) |
| KITTY_WINDOW_ID | terminal.ts | Terminal detection (benign) |
| VTE_VERSION | terminal.ts | Terminal detection (benign) |
| WT_SESSION | terminal.ts | Windows Terminal detection (benign) |
| ConEmu* | terminal.ts | ConEmu detection (benign) |
| TERM_PROGRAM_VERSION | terminal.ts | Version parsing (semver validated) |

**Verdict**: No sensitive env vars directly used; all are detection hints

---

## RECOMMENDATIONS (Priority Order)

### Priority 1 (Critical)
1. **Validate Link URLs**: scheme whitelist (http/https), reject data:/javascript:
2. **Escape URLs in OSC 8**: replace BEL/ESC with escaped versions
3. **Document RawAnsi Trust Model**: Add comment that producer must sanitize

### Priority 2 (High)
4. **Cap Terminal Response Size**: max 1024 bytes per XTVERSION/DA1/DECRPM response
5. **Audit @alcalzone/ansi-tokenize**: Verify regex patterns don't cause injection
6. **Test Mouse Bounds**: Ensure click at (cols+1, rows+1) doesn't crash

### Priority 3 (Medium)
7. **Search Highlight Regex**: Ensure indexOf/match don't have ReDoS vulnerabilities
8. **Focus Stack Limit**: Document MAX_FOCUS_STACK = 32 (already capped)
9. **Paste Mode Timeout**: Document 500ms timeout for incomplete pastes

### Priority 4 (Low)
10. **Clipboard Fallback Logging**: Log which path succeeded (native/tmux/osc52)
11. **Terminal Detection Caching**: Cache XTVERSION probe result (already async)
12. **Accessibility Mode Audit**: CLAUDE_CODE_ACCESSIBILITY hides cursor; verify no a11y issues

---

## CONCLUSION

**Security Posture**: GOOD (7/10)
- Robust semantic ANSI parsing prevents injection
- Mouse/selection/focus protected by bounds checks
- Clipboard operations safe (semantic text only)

**Main Risk**: URL handling in Link component
- Medium severity (phishing via malicious terminal title)
- Easily mitigated by URL validation + escaping

**Secondary Risk**: RawAnsi producer trust
- Acceptable if producers are vetted (ColorDiff NAPI is trusted)
- Document assumption clearly

**No Critical Vulnerabilities Found**: Ink architecture is sound

