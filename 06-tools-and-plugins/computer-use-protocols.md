# Claude Code Computer Use - Coordinate System & Detailed Protocols

## 1. COORDINATE SYSTEM ARCHITECTURE

### Logical vs Physical Pixels

**Definitions**:
- **Logical pixels**: What the display advertises (user-facing resolution)
  - Example: 1920x1080 on a 27" 4K display (but rendered at 3840x2160 physical)
- **Physical pixels**: Actual device pixels
  - Example: 3840x2160 on the same 27" 4K display
- **Scale factor**: physical / logical (typically 1.0 or 2.0 on macOS, up to 3.0 on new Retinas)

### Coordinate Mode

**Two modes** (feature gate in gates.ts):

1. **pixels** (default, 'pixels'):
   - Coordinates in logical display pixels
   - Click at (500, 300) → clicks logical pixel 500,300
   - Model sees: logical dimensions, logical click coordinates
   - Executor transforms: logical → physical internally

2. **normalized** (alternative, 'normalized'):
   - Coordinates in [0.0, 1.0] normalized to display bounds
   - Click at (0.5, 0.5) → clicks center of display
   - Model sees: normalized coordinates
   - Executor denormalizes to logical before physical conversion

**Frozen at session start**:
```typescript
let frozenCoordinateMode: CoordinateMode | undefined
export function getChicagoCoordinateMode(): CoordinateMode {
  frozenCoordinateMode ??= readConfig().coordinateMode
  return frozenCoordinateMode
}
```

Why frozen? Prevents mid-session GB flip from causing model/executor coordinate mismatch.

### Screenshot Size Computation

**Pipeline**:

```typescript
function computeTargetDims(logicalW: number, logicalH: number, scaleFactor: number): [number, number] {
  const physW = Math.round(logicalW * scaleFactor)
  const physH = Math.round(logicalH * scaleFactor)
  return targetImageSize(physW, physH, API_RESIZE_PARAMS)
}
```

**Steps**:

1. Get display geometry:
   ```
   display.getSize(displayId) → {width, height, scaleFactor}
   ```

2. Convert to physical:
   ```
   physW = Math.round(logicalW * scaleFactor)
   physH = Math.round(logicalH * scaleFactor)
   ```

3. Compute API target dims (from @ant/computer-use-mcp):
   ```
   targetImageSize(physW, physH, API_RESIZE_PARAMS)
   ```
   - Applies maximum size constraints
   - Returns [targetW, targetH] for JPEG encoding
   - Prevents server-side resize, keeps scaleCoord coherent

4. Pass to screenshot capture:
   ```
   cu.screenshot.captureExcluding(allowedBundleIds, quality=0.75, targetW, targetH, displayId)
   ```

**Why pre-sizing?**
- Server-side resize causes coordinate mismatch
- Pre-sized to target prevents server-side scaling
- Keeps the coordinate transformation predictable: physical ← logical

### Zoom Region Mapping

**zoom()** function (executor.ts line 420):

```typescript
async zoom(
  regionLogical: { x: number; y: number; w: number; h: number },
  allowedBundleIds: string[],
  displayId?: number,
): Promise<{ base64: string; width: number; height: number }>
```

**Transformation**:

1. regionLogical: User provides logical pixel bounds (x, y, width, height)
2. Compute output dims:
   ```
   const d = cu.display.getSize(displayId)
   const [outW, outH] = computeTargetDims(
     regionLogical.w,
     regionLogical.h,
     d.scaleFactor,
   )
   ```
3. Call Swift:
   ```
   cu.screenshot.captureRegion(
     allowedBundleIds,
     regionLogical.x,      // logical x
     regionLogical.y,      // logical y
     regionLogical.w,      // logical width
     regionLogical.h,      // logical height
     outW, outH,           // target output dims
     SCREENSHOT_JPEG_QUALITY,
     displayId,
   )
   ```

**Note**: regionLogical is passed as-is (integers, coordinates are logical); Swift handles physical conversion internally.

---

## 2. INPUT DEVICE SIMULATION PROTOCOLS

### Mouse Movement & Settling

**moveAndSettle()**:

```typescript
async function moveAndSettle(input: Input, x: number, y: number): Promise<void> {
  await input.moveMouse(x, y, false)
  await sleep(MOVE_SETTLE_MS)  // 50ms
}
```

**Why settle?**

- enigo posts CGEvent.moveMouse immediately
- HID tap delay: ~10-20ms for event to reach system
- Input queue drain: ~20-30ms for AppKit to process
- Total: ~50ms for NSEvent.mouseLocation to reflect the move
- Without settling: click arrives before mouse position updates

**Used in**:
- Before every click
- Before every scroll
- Before drag (from) position
- Before drag (to) with animatedMove (which has different timing)

### Animated Drag Movement

**animatedMove()** (executor.ts line 217):

```typescript
async function animatedMove(
  input: Input,
  targetX: number,
  targetY: number,
  mouseAnimationEnabled: boolean,
): Promise<void> {
  if (!mouseAnimationEnabled) {
    await moveAndSettle(input, targetX, targetY)
    return
  }

  const start = await input.mouseLocation()
  const deltaX = targetX - start.x
  const deltaY = targetY - start.y
  const distance = Math.hypot(deltaX, deltaY)

  if (distance < 1) return

  const durationSec = Math.min(distance / 2000, 0.5)
  if (durationSec < 0.03) {
    await moveAndSettle(input, targetX, targetY)
    return
  }

  const frameRate = 60
  const frameIntervalMs = 1000 / frameRate
  const totalFrames = Math.floor(durationSec * frameRate)

  for (let frame = 1; frame <= totalFrames; frame++) {
    const t = frame / totalFrames
    const eased = 1 - Math.pow(1 - t, 3)

    await input.moveMouse(
      Math.round(start.x + deltaX * eased),
      Math.round(start.y + deltaY * eased),
      false,
    )

    if (frame < totalFrames) {
      await sleep(frameIntervalMs)
    }
  }

  await sleep(MOVE_SETTLE_MS)
}
```

**Algorithm**:

1. **Distance calculation**: Euclidean distance in logical pixels
2. **Duration**: distance / 2000px per second, capped at 0.5s
   - 100px → 50ms
   - 500px → 250ms
   - 1000px → 500ms
3. **Frame count**: totalFrames = duration * 60fps
4. **Easing**: ease-out-cubic: `1 - (1 - t)^3`
   - t=0: eased=0 (at start)
   - t=0.5: eased=0.875 (87.5% toward target)
   - t=1: eased=1.0 (at target)
5. **Interpolation**: Move to eased position each frame
6. **Final settle**: 50ms sleep after reaching target

**Why animated?**

Some target apps (scrollbars, window resizers, drag-to-reorder lists) specifically watch for `.leftMouseDragged` events with intermediate positions. Slow motion gives them time to react to each intermediate position, preventing "stutter" or missed interactions.

**Per-frame timing**:
- 60fps = 16.67ms per frame
- Sleep between frames (except last)
- Final frame immediately precedes settle sleep

### Click Modifiers

**click()** with modifiers:

```typescript
async click(
  x: number,
  y: number,
  button: 'left' | 'right' | 'middle',
  count: 1 | 2 | 3,
  modifiers?: string[],
): Promise<void> {
  const input = requireComputerUseInput()
  await moveAndSettle(input, x, y)

  if (modifiers && modifiers.length > 0) {
    await drainRunLoop(() =>
      withModifiers(input, modifiers, () =>
        input.mouseButton(button, 'click', count),
      ),
    )
  } else {
    await input.mouseButton(button, 'click', count)
  }
}
```

**withModifiers() helper**:

```typescript
async function withModifiers<T>(
  input: Input,
  mods: string[],
  fn: () => Promise<T>,
): Promise<T> {
  const pressed: string[] = []
  try {
    for (const m of mods) {
      await input.key(m, 'press')
      pressed.push(m)
    }
    return await fn()
  } finally {
    await releasePressed(input, pressed)
  }
}

async function releasePressed(input: Input, pressed: string[]): Promise<void> {
  let k: string | undefined
  while ((k = pressed.pop()) !== undefined) {
    try {
      await input.key(k, 'release')
    } catch {
      // Swallow — best-effort release.
    }
  }
}
```

**Key points**:

- Press modifiers in order (cmd, shift, ctrl, etc.)
- Track each successful press in array
- Execute click (or other function)
- Release in **reverse order** (LIFO)
- Finally block ensures release even if fn() throws
- Release errors swallowed (best-effort)

**Why reverse order?**

Some OS APIs care about modifier release sequence. Reverse order is the safe default.

### Drag Sequence

**drag()** (executor.ts line 579):

```typescript
async drag(
  from: { x: number; y: number } | undefined,
  to: { x: number; y: number },
): Promise<void> {
  const input = requireComputerUseInput()

  if (from !== undefined) {
    await moveAndSettle(input, from.x, from.y)
  }

  await input.mouseButton('left', 'press')
  await sleep(MOVE_SETTLE_MS)  // 50ms

  try {
    await animatedMove(input, to.x, to.y, getMouseAnimationEnabled())
  } finally {
    await input.mouseButton('left', 'release')
  }
}
```

**Sequence**:

1. [Optional] Move to start position and settle (50ms)
2. Press left mouse button
3. Sleep 50ms (let HID tap register pressedMouseButtons)
4. Animated move to target (with intermediate frames if enabled)
5. Finally: release left mouse button (even if move throws)

**Why the post-press sleep?**

enigo's moveMouse() reads `NSEvent.pressedMouseButtons` to decide whether to emit `.leftMouseDragged` (button down + move) vs `.mouseMoved` (just move). The synthetic leftMouseDown needs ~50ms HID round-trip for the flag to update.

### Scroll

**scroll()** (executor.ts line 600):

```typescript
async scroll(x: number, y: number, dx: number, dy: number): Promise<void> {
  const input = requireComputerUseInput()
  await moveAndSettle(input, x, y)

  if (dy !== 0) {
    await input.mouseScroll(dy, 'vertical')
  }

  if (dx !== 0) {
    await input.mouseScroll(dx, 'horizontal')
  }
}
```

**Behavior**:

- Move to (x, y) and settle
- Vertical scroll first (common axis)
- Horizontal scroll second
- Vertical failure doesn't prevent horizontal (independent calls)

**Why vertical first?**

Most apps only need vertical scrolling. If horizontal fails, vertical still succeeded.

### Keyboard Key Press

**key()** (executor.ts line 455):

```typescript
async key(keySequence: string, repeat?: number): Promise<void> {
  const input = requireComputerUseInput()
  const parts = keySequence.split('+').filter(p => p.length > 0)
  const isEsc = isBareEscape(parts)
  const n = repeat ?? 1

  await drainRunLoop(async () => {
    for (let i = 0; i < n; i++) {
      if (i > 0) {
        await sleep(8)  // 8ms between repeats
      }
      if (isEsc) {
        notifyExpectedEscape()
      }
      await input.keys(parts)
    }
  })
}

function isBareEscape(parts: readonly string[]): boolean {
  if (parts.length !== 1) return false
  const lower = parts[0]!.toLowerCase()
  return lower === 'escape' || lower === 'esc'
}
```

**Parsing**:

- Input: "ctrl+shift+a"
- Split on '+': ["ctrl", "shift", "a"]
- Filter empty: (none in this example)
- Check for bare escape: false (length > 1)

**Escape special case**:

- Bare "escape" or "esc" (single key, case-insensitive)
- Calls `notifyExpectedEscape()` before key() to punch hole in CGEventTap
- CGEventTap checks `event.flags.isEmpty`, so ctrl+escape passes through (flags not empty, different code path)

**Repeats**:

- 8ms sleep between iterations (125Hz USB polling rate)
- re=0: no sleep before first press
- repeat=3: keys → sleep(8) → keys → sleep(8) → keys

### Hold Key

**holdKey()** (executor.ts line 475):

```typescript
async holdKey(keyNames: string[], durationMs: number): Promise<void> {
  const input = requireComputerUseInput()
  const pressed: string[] = []
  let orphaned = false

  try {
    await drainRunLoop(async () => {
      for (const k of keyNames) {
        if (orphaned) return
        if (isBareEscape([k])) {
          notifyExpectedEscape()
        }
        await input.key(k, 'press')
        pressed.push(k)
      }
    })

    await sleep(durationMs)
  } finally {
    orphaned = true
    await drainRunLoop(() => releasePressed(input, pressed))
  }
}
```

**Sequence**:

1. Press all keys (in order)
2. Sleep for durationMs
3. Release all keys (in reverse order)

**Orphaned flag**:

If press-phase drainRunLoop times out while CFRunLoop pump stays running (esc-hotkey retain), the orphaned lambda might continue pushing to `pressed` after finally's releasePressed. The flag stops at next iteration.

### Type Text

**type()** (executor.ts line 509):

```typescript
async type(text: string, opts: { viaClipboard: boolean }): Promise<void> {
  const input = requireComputerUseInput()

  if (opts.viaClipboard) {
    await drainRunLoop(() => typeViaClipboard(input, text))
    return
  }

  await input.typeText(text)
}
```

**Two paths**:

1. **typeText()**: Direct enigo typeText
   - Per-grapheme loop handled by caller
   - 8ms sleep between graphemes (toolCalls.ts)
   - No clipboard involved

2. **viaClipboard**: Clipboard-based paste
   - Used for special characters, non-ASCII
   - Safer for multiline text
   - Requires drainRunLoop (Cmd+V dispatches to main queue)

**Clipboard paste sequence** (typeViaClipboard):

```typescript
async function typeViaClipboard(input: Input, text: string): Promise<void> {
  let saved: string | undefined
  try {
    saved = await readClipboardViaPbpaste()
  } catch {
    logForDebugging('[computer-use] pbpaste before paste failed; proceeding without restore')
  }

  try {
    await writeClipboardViaPbcopy(text)
    if ((await readClipboardViaPbpaste()) !== text) {
      throw new Error('Clipboard write did not round-trip.')
    }
    await input.keys(['command', 'v'])
    await sleep(100)
  } finally {
    if (typeof saved === 'string') {
      try {
        await writeClipboardViaPbcopy(saved)
      } catch {
        logForDebugging('[computer-use] clipboard restore after paste failed')
      }
    }
  }
}
```

**Steps**:

1. Read current clipboard (pbpaste)
   - Failure: log, proceed without restore
2. Write text via pbcopy
3. **Read-back verify**: pbpaste again
   - Failure: throw (never paste junk)
4. Cmd+V to paste
5. Sleep 100ms (paste effect vs restore race)
6. Restore saved in finally
   - Failures swallowed (user's clipboard not clobbered)

**Why read-back verify?**

pbcopy can fail silently (permissions, system state). Verify ensures the clipboard actually has our text before pasting. Prevents pasting stale content.

---

## 3. APP MANAGEMENT PROTOCOLS

### prepareForAction (Hide & Defocus)

**Call sequence**:

```typescript
async prepareForAction(
  allowlistBundleIds: string[],
  displayId?: number,
): Promise<string[]>
```

**Steps**:

1. Check if hideBeforeAction gate enabled
2. Wrap in drainRunLoop (prepareDisplay triggers window-manager events on CFRunLoop)
3. Call `cu.apps.prepareDisplay(allowlistBundleIds, surrogateHost, displayId)`
   - allowlistBundleIds: User-granted apps (safe to show)
   - surrogateHost: Terminal (if detected), or sentinel CLI_HOST_BUNDLE_ID
   - displayId: Display to hide on (optional)
4. Swift side:
   - Hides all apps NOT in allowlist and NOT surrogateHost
   - Defocus hidden apps
   - Move them off-screen
   - Fire window-manager events
5. Return hidden array for cleanup tracking

**Return value stored**:

```typescript
ctx.setAppState(prev => ({
  ...prev,
  computerUseMcpState: {
    ...prev.computerUseMcpState,
    hiddenDuringTurn: new Set(result.hidden),
  },
}))
```

**Error handling**:

If prepareForAction throws:
```typescript
catch (err) {
  logForDebugging(
    `[computer-use] prepareForAction failed; continuing to action: ${errorMessage(err)}`,
    { level: 'warn' },
  )
  return []
}
```

Logs warning, continues (frontmost gate enforces safety at package level).

### listInstalledApps (Enumeration)

**Flow**:

```typescript
async listInstalledApps(): Promise<InstalledApp[]> {
  return drainRunLoop(() => cu.apps.listInstalled())
}
```

**Wrapping in drainRunLoop**:

Swift's `listInstalled()` is `@MainActor`, dispatches to DispatchQueue.main, needs CFRunLoop pump.

**Filtering for tool description**:

```typescript
async function tryGetInstalledAppNames(): Promise<string[] | undefined> {
  const adapter = getComputerUseHostAdapter()
  const enumP = adapter.executor.listInstalledApps()

  let timer: ReturnType<typeof setTimeout> | undefined
  const timeoutP = new Promise<undefined>(resolve => {
    timer = setTimeout(resolve, APP_ENUM_TIMEOUT_MS, undefined)  // 1000ms
  })

  const installed = await Promise.race([enumP, timeoutP])
    .catch(() => undefined)
    .finally(() => clearTimeout(timer))

  if (!installed) {
    void enumP.catch(() => {})  // Swallow late rejection
    logForDebugging(
      `[Computer Use MCP] app enumeration exceeded ${APP_ENUM_TIMEOUT_MS}ms or failed; tool description omits list`,
    )
    return undefined
  }

  return filterAppsForDescription(installed, homedir())
}
```

**Timeout handling**:

- 1s soft timeout (Spotlight may be slow)
- If timeout wins race: late enumeration rejection swallowed
- If enumeration wins: filtered via appNames.ts (whitelist, dedup, cap at 50)

### appUnderPoint (Point Query)

**Flow**:

```typescript
async appUnderPoint(x: number, y: number): Promise<{ bundleId: string; displayName: string } | null> {
  return cu.apps.appUnderPoint(x, y)
}
```

**Usage**: Detect which app is at coordinate (x, y). Used to verify which app is being targeted.

---

## 4. TCC (Transparency, Consent, Control) Permissions

**Check at startup**:

```typescript
ensureOsPermissions: async () => {
  const cu = requireComputerUseSwift()
  const accessibility = cu.tcc.checkAccessibility()
  const screenRecording = cu.tcc.checkScreenRecording()
  return accessibility && screenRecording
    ? { granted: true }
    : { granted: false, accessibility, screenRecording }
}
```

**Two permissions**:

1. **Accessibility**: Required for mouse/keyboard simulation + CGEventTap
   - Without: mouse/keyboard fail, Escape hotkey fails
   - Without: CU proceeds but degraded

2. **Screen Recording**: Required for SCContentFilter (screenshot)
   - Without: screenshot fails
   - Tool call fails before action execution

---

## 5. Display Management

### Display Geometry

**Query**:

```typescript
async getDisplaySize(displayId?: number): Promise<DisplayGeometry> {
  return cu.display.getSize(displayId)
}
```

**Returns**:

```typescript
{
  width: number,      // Logical width
  height: number,     // Logical height
  scaleFactor: number // Physical / logical
}
```

### List All Displays

```typescript
async listDisplays(): Promise<DisplayGeometry[]> {
  return cu.display.listAll()
}
```

### Find Window Displays

```typescript
async findWindowDisplays(
  bundleIds: string[],
): Promise<Array<{ bundleId: string; displayIds: number[] }>>
```

Map bundle IDs to displays their windows are on.

---

## 6. Lock Acquisition Lifecycle

**Flow during first CU tool call**:

```
request_access or screenshot or click, etc.
  ↓
MCP tool dispatch
  ↓
tryAcquireComputerUseLock()
  ├─ O_EXCL create at ~/.claude/computer-use.lock
  ├─ Read existing if create fails
  ├─ Check PID liveness (process.kill(pid, 0))
  ├─ Stale recovery: unlink + retry create
  └─ Return {kind: 'acquired', fresh: true} or {kind: 'blocked', by: sessionId}
  ↓
registerEscHotkey()
  ├─ CGEventTap on DispatchQueue.main
  ├─ Retains CFRunLoop pump
  └─ Returns true/false on permission/tapCreate status
  ↓
[Tool executes]
  ↓
cleanupComputerUseAfterTurn() [at turn end]
  ├─ unhideComputerUseApps([...hidden]) [fire-and-forget, 5s timeout]
  ├─ unregisterEscHotkey() [swallows errors]
  ├─ releaseComputerUseLock()
  └─ Send OS notification
```

---

## Summary

The coordinate system is well-layered: logical pixels (model-facing) → physical pixels (internal) → API target (pre-sized). Input primitives are carefully timed (50ms settles, 100ms paste races, 8ms key repeats). App management uses atomic operations (O_EXCL lock) and graceful degradation (soft timeouts, swallowed errors). TCC permissions are checked but don't block execution (accessibility failure only degrades CU, not blocks it).

