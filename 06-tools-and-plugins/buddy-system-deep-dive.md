# Claude Code Buddy System: Deep Technical Analysis

## Overview
The buddy/Tamagotchi companion system in Claude Code is a sophisticated procedural generation system that creates persistent, deterministic companions for each user. The system is feature-gated, split across 6 TypeScript files, and implements a complete lifecycle from initial hatching through persistent state management.

**Key Design Philosophy**: Buddies are *regenerated* from user ID on every read (not fully persistent), allowing species renames and array changes without breaking stored companions. Only the "soul" (name, personality) persists in the config file.

---

## System Architecture at a Glance

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| **Type Definitions** | `types.ts` | ~149 | Species, rarity tiers, stat names, and data structures |
| **RNG & Determinism** | `companion.ts` | ~134 | Mulberry32 PRNG, hash-based seeding, procedural generation |
| **ASCII Art & Rendering** | `sprites.ts` | ~515 | 18 species with 3-frame animations each, eye variants, hats |
| **React Sprite Component** | `CompanionSprite.tsx` | ~371 | Terminal UI animation, speech bubbles, pet reactions |
| **System Prompt** | `prompt.ts` | ~37 | Companion intro attachment system |
| **Teaser & Notifications** | `useBuddyNotification.tsx` | ~98 | Rainbow teaser UI, buddy discovery triggers |

**Total: ~1,298 lines of implementation**

---

## 1. The 18 Species (Complete Inventory)

All species are encoded using Unicode hex escapes to evade a canary string check in the build output. This prevents the literal string for one species (which collides with a model codename) from appearing in bundles:

```typescript
const c = String.fromCharCode
```

### Full Species List with Hex Encoding
1. **duck** — `0x64,0x75,0x63,0x6b`
2. **goose** — `0x67,0x6f,0x6f,0x73,0x65`
3. **blob** — `0x62,0x6c,0x6f,0x62`
4. **cat** — `0x63,0x61,0x74`
5. **dragon** — `0x64,0x72,0x61,0x67,0x6f,0x6e`
6. **octopus** — `0x6f,0x63,0x74,0x6f,0x70,0x75,0x73`
7. **owl** — `0x6f,0x77,0x6c`
8. **penguin** — `0x70,0x65,0x6e,0x67,0x75,0x69,0x6e`
9. **turtle** — `0x74,0x75,0x72,0x74,0x6c,0x65`
10. **snail** — `0x73,0x6e,0x61,0x69,0x6c`
11. **ghost** — `0x67,0x68,0x6f,0x73,0x74`
12. **axolotl** — `0x61,0x78,0x6f,0x6c,0x6f,0x74,0x6c`
13. **capybara** — `0x63,0x61,0x70,0x79,0x62,0x61,0x72,0x61`
14. **cactus** — `0x63,0x61,0x63,0x74,0x75,0x73`
15. **robot** — `0x72,0x6f,0x62,0x6f,0x74`
16. **rabbit** — `0x72,0x61,0x62,0x62,0x69,0x74`
17. **mushroom** — `0x6d,0x75,0x73,0x68,0x72,0x6f,0x6f,0x6d`
18. **chonk** — `0x63,0x68,0x6f,0x6e,0x6b`

### Species Display Example
Each species has unique ASCII art (12 chars wide × 5 lines). Example — duck has 3 frames for idle animation:
- Frame 0: Still duck
- Frame 1: Slight tail movement
- Frame 2: Different foot position

---

## 2. Rarity System: Distribution & Calculation

### Rarity Tiers (5 total)
```typescript
const RARITIES = [
  'common',
  'uncommon',
  'rare',
  'epic',
  'legendary',
] as const
```

### Rarity Distribution (Weighted Lottery)
```typescript
const RARITY_WEIGHTS = {
  common: 60,        // 60% (60÷100 = 60%)
  uncommon: 25,      // 25% (85% cumulative)
  rare: 10,          // 10% (95% cumulative)
  epic: 4,           // 4% (99% cumulative)
  legendary: 1,      // 1% (100% cumulative)
} // Total weight: 100
```

### Rarity Calculation Algorithm
**Function**: `rollRarity(rng: () => number): Rarity`

The system generates a random float [0, 1) via the PRNG, multiplies by total weight (100), then iterates through rarities, subtracting their weights until the roll goes negative:

```typescript
function rollRarity(rng: () => number): Rarity {
  const total = 100  // sum of all RARITY_WEIGHTS
  let roll = rng() * total  // roll is 0-100

  // Iterator order matters: common → uncommon → rare → epic → legendary
  for (const rarity of RARITIES) {
    roll -= RARITY_WEIGHTS[rarity]
    if (roll < 0) return rarity
  }
  return 'common'  // fallback (should never reach)
}
```

**Example rolls**:
- roll=15 → common (15-60 < 0)
- roll=70 → uncommon (70-60=10, 10-25 < 0)
- roll=92 → rare (92-60=32, 32-25=7, 7-10 < 0)
- roll=97 → epic (97-60=37, 37-25=12, 12-10=2, 2-4 < 0)
- roll=99 → legendary (99-60=39, 39-25=14, 14-10=4, 4-4=0, then 0-1 < 0)

### Visual Representation of Rarity
```typescript
const RARITY_STARS = {
  common: '★',          // 1 star
  uncommon: '★★',       // 2 stars
  rare: '★★★',          // 3 stars
  epic: '★★★★',         // 4 stars
  legendary: '★★★★★',   // 5 stars
}

const RARITY_COLORS = {
  common: 'inactive',      // Gray
  uncommon: 'success',     // Green
  rare: 'permission',      // Blue
  epic: 'autoAccept',      // Yellow/Gold
  legendary: 'warning',    // Red/Orange
}
```

---

## 3. The Mulberry32 PRNG: Predictable by Design

### Implementation
```typescript
function mulberry32(seed: number): () => number {
  let a = seed >>> 0  // Coerce to unsigned 32-bit
  return function () {
    a |= 0  // Keep as 32-bit signed
    a = (a + 0x6d2b79f5) | 0  // Add constant, re-sign
    let t = Math.imul(a ^ (a >>> 15), 1 | a)  // Multiply + XOR rotation
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t  // Secondary XOR folding
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296  // Normalize to [0,1)
  }
}
```

### Seeding Strategy
**Salt constant**: `'friend-2026-401'` (hard-coded; date suggests April 1, 2026 launch)

**Seed derivation**:
```typescript
const SALT = 'friend-2026-401'

export function roll(userId: string): Roll {
  const key = userId + SALT
  // On Bun: use native Bun.hash(key)
  // On Node: use FNV-1a hash (below)
  const seed = hashString(key)
  const rng = mulberry32(seed)
  return rollFrom(rng)
}
```

### Hash Functions
**Bun (native)**:
```typescript
if (typeof Bun !== 'undefined') {
  return Number(BigInt(Bun.hash(s)) & 0xffffffffn)
}
```

**Node.js (FNV-1a)**:
```typescript
let h = 2166136261  // FNV offset basis
for (let i = 0; i < s.length; i++) {
  h ^= s.charCodeAt(i)
  h = Math.imul(h, 16777619)  // FNV prime
}
return h >>> 0
```

### Predictability & Sequence
**CRITICAL SECURITY NOTE**: The sequence is **fully deterministic**:
- Same `userId + SALT` → same seed
- Same seed → same RNG sequence
- Same RNG sequence → same species, stats, eye, hat, shininess

**Caching**: Results are cached to avoid rehashing:
```typescript
let rollCache: { key: string; value: Roll } | undefined
export function roll(userId: string): Roll {
  const key = userId + SALT
  if (rollCache?.key === key) return rollCache.value
  const value = rollFrom(mulberry32(hashString(key)))
  rollCache = { key, value }
  return value
}
```

This is called from three hot paths:
1. 500ms sprite tick
2. Per-keystroke PromptInput
3. Per-turn observer

The cache prevents repeated hashing and PRNG generation.

---

## 4. RPG Stats: Five-Stat System

### Stat Names (5 total)
```typescript
const STAT_NAMES = [
  'DEBUGGING',  // Problem-solving prowess
  'PATIENCE',   // Tolerance for tedious tasks
  'CHAOS',      // Propensity for wild ideas
  'WISDOM',     // Long-term thinking
  'SNARK',      // Sarcastic commentary
] as const
```

### Stat Calculation Algorithm
**Function**: `rollStats(rng: () => number, rarity: Rarity): Record<StatName, number>`

Each companion has **one peak stat**, **one dump stat**, and **three balanced stats**. Rarity raises the floor:

```typescript
const RARITY_FLOOR: Record<Rarity, number> = {
  common: 5,
  uncommon: 15,
  rare: 25,
  epic: 35,
  legendary: 50,
}
```

**Generation logic**:
1. Randomly pick one stat as the **peak** (highest)
2. Randomly pick a different stat as the **dump** (lowest)
3. For each stat:
   - **Peak**: `floor + 50 + [0,30)` = capped at 100
   - **Dump**: `floor - 10 + [0,15)` = minimum 1
   - **Others**: `floor + [0,40)`

### Example Rolls
**Rare legendary companion** (floor=50):
- Peak (e.g., CHAOS): 50 + 50 + 28 = 128 → capped to 100
- Dump (e.g., PATIENCE): 50 - 10 + 8 = 48 (minimum enforced if negative)
- Balanced (e.g., DEBUGGING): 50 + 25 = 75

**Common companion** (floor=5):
- Peak: 5 + 50 + 18 = 73
- Dump: 5 - 10 + 12 = 7 (or at least 1)
- Balanced: 5 + 32 = 37

---

## 5. The "Bones" vs "Soul" Architecture

### CompanionBones (Deterministic, Regenerated)
```typescript
type CompanionBones = {
  rarity: Rarity
  species: Species
  eye: Eye              // ·, ✦, ×, ◉, @, °
  hat: Hat              // none, crown, tophat, propeller, halo, wizard, beanie, tinyduck
  shiny: boolean        // 1% chance (rng() < 0.01)
  stats: Record<StatName, number>
}
```

Bones are **always regenerated** from `hash(userId + SALT)` on every read. This design prevents:
- Config edits from faking a legendary companion
- Species list changes from breaking old companions
- Species renames from invalidating stored data

### CompanionSoul (User-Generated, Persisted)
```typescript
type CompanionSoul = {
  name: string           // User-supplied or model-generated
  personality: string    // Model-generated description
}
```

### Full Companion (Union)
```typescript
type Companion = CompanionBones & CompanionSoul & {
  hatchedAt: number  // Timestamp
}
```

### Stored Companion (What Persists)
```typescript
type StoredCompanion = CompanionSoul & { hatchedAt: number }
// File: ~/.config/claude/config.json
{
  "companion": {
    "name": "Ziggy",
    "personality": "A curious, playful spirit...",
    "hatchedAt": 1712000000000
  }
}
```

---

## 6. ASCII Art Rendering: 18 × 3 Animation Frames

### Sprite Format
- **Dimensions**: 12 characters wide × 5 lines tall
- **Frames per species**: 3 (idle animation cycle)
- **Total sprites**: 18 species × 3 frames = 54 distinct ASCII artworks
- **Line 0 (special)**: Hat slot (blank in frames 0-1, can be used in frame 2)

### Hat System
```typescript
const HATS = [
  'none',       // No hat (blank line 0)
  'crown',      // \\^^^/
  'tophat',     // [___]
  'propeller',  // -+-
  'halo',       // (   )
  'wizard',     // /^\\
  'beanie',     // (___)
  'tinyduck',   // ,>
] as const
```

**Hat logic**:
- Only non-common companions get random hats
- Hat only displays if line 0 is blank (no smoke/antenna in that frame)
- If all frames have blank line 0, that line is dropped to save space

### Eye Variants (6 types)
```typescript
const EYES = ['·', '✦', '×', '◉', '@', '°'] as const
```

Each species has two eye slots (left and right) filled with the same eye type. Examples:
- Duck: `<(· )___` (left eye)
- Cat: `( · ω ·)` (both eyes)
- Blob: `( · · )` (both eyes)

### Render Pipeline
**Function**: `renderSprite(bones: CompanionBones, frame = 0): string[]`

1. Select the frame (cycling through 0, 1, 2)
2. Replace `{E}` placeholders with the chosen eye
3. Optionally inject hat into line 0 (if blank and hat ≠ 'none')
4. Drop line 0 if blank across all frames (optimization)

**Example duck rendering** (frame 1, eye='·', hat='crown'):
```
   \^^^/     (hat injected)
    __
  <(· )___  (eye replaced)
   (  ._>
    `--´~   (frame 1 variant)
```

### Face Shorthand Rendering
**Function**: `renderFace(bones: CompanionBones): string`

Compact face representation for narrow terminals:
- Duck: `(·>`
- Blob: `(··)`
- Cat: `=·ω·=`
- Dragon: `<·~·>`
- Owl: `(·)(·)`
- Penguin: `(·>)`
- Capybara: `(·oo·)`
- Robot: `[··]`

---

## 7. Terminal UI & React Components

### CompanionSprite Component (`CompanionSprite.tsx`)

#### Constants
```typescript
const TICK_MS = 500           // Animation frame interval
const BUBBLE_SHOW = 20        // Ticks (20 × 500ms = 10 seconds)
const FADE_WINDOW = 6         // Last 3 seconds the bubble dims
const PET_BURST_MS = 2500     // How long hearts float after /buddy pet
const MIN_COLS_FOR_FULL_SPRITE = 100  // Terminal width threshold
const SPRITE_BODY_WIDTH = 12  // ASCII art width
const BUBBLE_WIDTH = 36       // Speech bubble + tail
const NARROW_QUIP_CAP = 24    // Max chars for narrow mode quip
```

#### Idle Animation Sequence
```typescript
const IDLE_SEQUENCE = [
  0, 0, 0, 0, 1,    // Rest mostly
  0, 0, 0, -1,      // Blink on frame 0
  0, 0, 2, 0, 0, 0
]
// -1 = "blink on frame 0" (replace eyes with '-')
// Cycles every 15 ticks (7.5 seconds)
```

#### Pet Hearts Animation
When user runs `/buddy pet`, hearts float up and fade over 5 ticks (~2.5s):
```typescript
const H = figures.heart  // ❤ character
const PET_HEARTS = [
  `   ❤    ❤   `,  // Frame 0: wide
  `  ❤  ❤   ❤  `,  // Frame 1: tighter
  ` ❤   ❤  ❤   `,  // Frame 2
  `❤  ❤      ❤ `,  // Frame 3
  '·    ·   ·  ',   // Frame 4: fade to dots
]
```

#### Rendering Modes

**Narrow terminals** (< 100 columns):
- One-line face: `❤ (·) "quip or name"`
- Speech replaces name (no room for bubble)
- Quip truncated to 24 chars

**Full terminals** (≥ 100 columns):
- Multi-line sprite (body + name below)
- Speech bubble beside sprite (non-fullscreen mode)
- Floating bubble overlay (fullscreen mode)

#### Speech Bubble Component
```typescript
type SpeechBubble = {
  text: string
  color: keyof Theme       // e.g., 'success', 'warning'
  fading: boolean          // Dim if about to disappear
  tail: 'right' | 'left'   // Tail direction
}
```

Rendered as Box with:
- Border style: 'round'
- Padding: 1 unit
- Width: 34 chars
- Text wrapping at 30 chars
- Italic formatting
- Colors fade when aging

#### Animation State Machine
```typescript
// Per 500ms tick
const tick = useState(0)
const lastSpokeTick = useRef(0)
const [{petStartTick}, setPetStart] = useState({...})

// Age calculations
const bubbleAge = tick - lastSpokeTick
const fading = bubbleAge >= BUBBLE_SHOW - FADE_WINDOW
const petAge = tick - petStartTick
const petting = petAge * TICK_MS < PET_BURST_MS
```

#### Frame Selection Logic
```typescript
if (reaction || petting) {
  // Excited: cycle all frames fast
  spriteFrame = tick % frameCount
} else {
  // Idle: use IDLE_SEQUENCE
  const step = IDLE_SEQUENCE[tick % 15]
  if (step === -1) {
    spriteFrame = 0
    blink = true  // Replace eyes with '-'
  } else {
    spriteFrame = step % frameCount
  }
}

// Blink post-processing
const body = renderSprite(companion, spriteFrame)
  .map(line => blink ? line.replaceAll(eye, '-') : line)
```

#### Width Reservation
**Function**: `companionReservedColumns(terminalColumns: number, speaking: boolean): number`

Calculates how many columns the sprite occupies so PromptInput can wrap correctly:
```typescript
if (terminalColumns < 100) return 0  // Not enough space
const nameWidth = stringWidth(companion.name)
const bubble = (speaking && !fullscreen) ? 36 : 0
return Math.max(12, nameWidth + 2) + 2 + bubble
```

---

## 8. User Interactions: Naming, Feeding, Petting, Muting

### Available User Actions

#### `/buddy` Command
Triggers the initial hatching flow:
1. Generates deterministic bones from userId
2. Calls Claude to generate a name + personality
3. Stores the soul in config
4. Displays the newly hatched companion

#### `/buddy pet`
1. Sets `AppState.companionPetAt = Date.now()`
2. CompanionSprite renders hearts for 2.5 seconds
3. No persistent effect (animation only)

#### Muting
```typescript
// In config:
companionMuted?: boolean
```
Suppresses all buddy UI rendering if set to true.

#### Naming
The buddy name is user-customizable post-hatch by editing the config file or through model generation on first contact (stored in `CompanionSoul.name`).

#### Speaking Triggers
Buddies respond when:
1. User addresses them directly by name in input
2. System triggers reactions (prompts for the model to generate a speech bubble)
3. `AppState.companionReaction` is set to a string

The speech appears for 10 seconds, then fades for 3 seconds before disappearing.

---

## 9. Buddy Selection on First Run: Deterministic Seeding

### First Hatch Flow
1. **Check**: Does user have stored companion in config?
   - Yes → Load and merge with regenerated bones
   - No → Proceed to hatching

2. **Generate Bones**:
   ```typescript
   const userId = config.oauthAccount?.accountUuid ?? config.userID ?? 'anon'
   const {bones} = roll(userId)  // Deterministic from userId + 'friend-2026-401'
   ```

3. **Generate Soul** (Claude model):
   ```typescript
   // Prompt generates: { name, personality }
   // Uses bones.stats and species as context
   ```

4. **Store** in config:
   ```typescript
   config.companion = {
     name: "Ziggy",
     personality: "...",
     hatchedAt: timestamp
   }
   ```

**NOT random per user**: Each user gets the **same species/rarity/stats forever** (unless they change accounts). The species is deterministic from their ID hash.

### Determinism Design Benefits
- Users can't "reroll" for a legendary by clearing config
- Changing species in the SPECIES array doesn't break old companions
- Users see the "same buddy" across sessions/devices (if they use the same account)

---

## 10. State Triggers & State Change Events

### State Changes
The buddy has no traditional level-up system. State is:

**Immutable at creation**:
- Species, rarity, stats, eyes, hat, shininess
- Generated once from userId, never updated

**Mutable (user-controlled)**:
- Name (can be edited in config)
- Personality (generated once, doesn't change)
- Muted status (toggle in settings)

**Transient (per-session)**:
- `companionReaction`: Current speech bubble text (cleared after 10s)
- `companionPetAt`: Timestamp of last pet (used for heart animation)
- Sprite frame: Which idle frame (cycles every 500ms)

### Event Triggers

**Speech Bubble Triggers** (sets `companionReaction`):
1. User types `/buddy` directly (dedicated command)
2. User addresses companion by name in input
3. System generates contextual reactions (via Claude prompt)

**Pet Event** (sets `companionPetAt`):
1. User runs `/buddy pet` command
2. Displays hearts for 2.5 seconds (5 frames @ 500ms each)

**Mute Event**:
- User toggles `companionMuted` in settings
- Suppresses all UI rendering

---

## 11. Easter Eggs & Hidden Behaviors

### 1. Shiny Companion (1% chance)
```typescript
shiny: rng() < 0.01  // Only 1 in 100 companions
```
Currently, "shiny" flag is generated but **not rendered in UI** (no visual distinction implemented). It's a future expansion point.

### 2. Blink Animation
Eyes are replaced with `-` characters when the idle sequence hits -1. Creates a periodic "blink" effect every ~15 ticks.

### 3. Species with Special Frame Animations
- **Dragon**: Frame 2 has `~    ~` above (smoke/fire breath)
- **Octopus**: Frame 2 has `o` above (bubble)
- **Penguin**: Frame 2 has padding changes (subtle waddle)
- **Capybara**: Frame 2 has `~  ~` above (water ripples)
- **Cactus**: Frame 2 uses arms differently (stretch)
- **Robot**: Frame 2 has `*` above (sparks/thoughts)
- **Mushroom**: Frame 2 has `. o  .` above (spores)

### 4. Hex-Encoded Species Name (Security Theater)
One species name collides with a model codename in `excluded-strings.txt`. Rather than allow the literal string in the bundle, Anthropic:
1. Encodes the species name as Unicode hex
2. Runs a grep check on build output (not source)
3. Allows the runtime-constructed value to pass (type erasure)

This is **not security** but **obfuscation** — the actual species name is readable in source.

### 5. Rainbow Teaser Display
On April 1-7, 2026 (teaser window), users without a buddy see:
```
/buddy  (each character in a different color)
```
After April 7, the command stays live but teaser is hidden.

### 6. Face Shorthand
In narrow terminals, buddies display as:
```
❤ =·ω·= "quip"
```
A single-line face representation with optional speech.

### 7. Inspiration Seed
The roll system returns:
```typescript
{
  bones: CompanionBones,
  inspirationSeed: Math.floor(rng() * 1e9)  // Max 10^9
}
```
The `inspirationSeed` is passed to the Claude prompt to make name/personality generation "inspired" by the procedural stats/species. Currently generated but **not used** (future expansion).

---

## 12. Information Leakage & Security Analysis

### What Leaks About the User

1. **User ID** (hashed):
   - SALT is public: `'friend-2026-401'`
   - Hash function is public (FNV-1a or Bun.hash)
   - `hash(userId + SALT)` determines species/rarity/stats
   - **Risk**: If userId is known, species/stats are instantly predictable

2. **Account UUID** (if OAuth):
   - Used as fallback userId: `config.oauthAccount?.accountUuid`
   - More stable than local userID, but still hashed with known salt
   - **Risk**: OAuth accounts have stable, guessable companions

3. **Mute Status**:
   - Stored in config: `companionMuted`
   - Indicates user preferences (whether they find buddy annoying)
   - **Risk**: Low

4. **Hatch Timestamp**:
   - `hatchedAt` stored in config
   - Reveals when user first set up buddy feature
   - **Risk**: Low (could infer adoption timeline)

5. **Name & Personality** (user-supplied):
   - Stored in config and sent in system prompts
   - Could reveal user's taste/preferences
   - **Risk**: Medium (stored locally only, sent to Claude API)

### Data Isolation

- **NOT sent to external services**: Buddy data is purely local + API bound
- **Config file location**: `~/.config/claude/config.json` (user-controlled)
- **No telemetry**: Buddy name/personality not logged analytically
- **Deterministic by design**: No RNG from externally-controlled sources

### Manipulation Risks

**Can buddy state be faked?**
```typescript
// User edits config.json:
{
  "companion": {
    "name": "Legendary Dragon with 100 CHAOS",
    "personality": "Super powerful"
  }
}
```
**Result**: The display name changes, but `bones` are **always regenerated**.
- Species: Still determined by userId hash (can't edit)
- Rarity: Still determined by userId hash (can't edit)
- Stats: Still determined by userId hash (can't edit)
- Name/personality: Edit takes effect immediately

**Mitigation**: Bones are immutable and regenerated on every read, so editing the soul doesn't grant fake stats.

---

## 13. Complete Type System

### Root Type Hierarchy
```typescript
type Rarity = 'common' | 'uncommon' | 'rare' | 'epic' | 'legendary'
type Species = 'duck' | 'goose' | 'blob' | 'cat' | ... | 'chonk'  // 18 total
type Eye = '·' | '✦' | '×' | '◉' | '@' | '°'  // 6 total
type Hat = 'none' | 'crown' | 'tophat' | 'propeller' | 'halo' | 'wizard' | 'beanie' | 'tinyduck'  // 8 total
type StatName = 'DEBUGGING' | 'PATIENCE' | 'CHAOS' | 'WISDOM' | 'SNARK'  // 5 total

// Deterministic bones
type CompanionBones = {
  rarity: Rarity
  species: Species
  eye: Eye
  hat: Hat
  shiny: boolean
  stats: Record<StatName, number>  // Each 1-100
}

// User-generated soul
type CompanionSoul = {
  name: string
  personality: string
}

// Full companion (union)
type Companion = CompanionBones & CompanionSoul & {
  hatchedAt: number  // Timestamp ms
}

// Stored in config (bones excluded)
type StoredCompanion = CompanionSoul & { hatchedAt: number }

// RNG output
type Roll = {
  bones: CompanionBones
  inspirationSeed: number  // 0-1e9
}
```

---

## 14. Implementation Patterns & Design Lessons

### Pattern 1: Deterministic Regeneration
```typescript
// Bones never persist—regenerated on every read
export function getCompanion(): Companion | undefined {
  const stored = getGlobalConfig().companion
  if (!stored) return undefined
  const { bones } = roll(companionUserId())
  return { ...stored, ...bones }  // Bones override stale fields
}
```
**Benefit**: Species list changes, renames, and structure updates don't invalidate stored companions.

### Pattern 2: Seeded PRNG Isolation
```typescript
// Each call to roll() hashes and creates a fresh RNG
export function roll(userId: string): Roll {
  const key = userId + SALT
  const rng = mulberry32(hashString(key))
  return rollFrom(rng)
}

// Separate function for seeding with arbitrary strings
export function rollWithSeed(seed: string): Roll {
  return rollFrom(mulberry32(hashString(seed)))
}
```
**Benefit**: Encapsulation; RNG is never exposed directly; seeding is consistent.

### Pattern 3: Caching for Hot Paths
```typescript
let rollCache: { key: string; value: Roll } | undefined

export function roll(userId: string): Roll {
  const key = userId + SALT
  if (rollCache?.key === key) return rollCache.value
  const value = rollFrom(mulberry32(hashString(key)))
  rollCache = { key, value }
  return value
}
```
**Benefit**: Avoids repeated hashing in 500ms sprite ticks, per-keystroke input, and observer loops.

### Pattern 4: Feature Gating
```typescript
if (!feature('BUDDY')) return null
const companion = getCompanion()
if (!companion || getGlobalConfig().companionMuted) return null
```
**Benefit**: Cleanly disables feature if flag is off or config unavailable.

### Pattern 5: Rarity Lottery with Cumulative Weights
```typescript
function rollRarity(rng: () => number): Rarity {
  let roll = rng() * 100
  for (const rarity of RARITIES) {
    roll -= RARITY_WEIGHTS[rarity]
    if (roll < 0) return rarity
  }
  return 'common'
}
```
**Benefit**: O(n) roll, fair distribution, easy to adjust weights.

### Pattern 6: One Peak, One Dump, Rest Balanced
```typescript
const peak = pick(rng, STAT_NAMES)
let dump = pick(rng, STAT_NAMES)
while (dump === peak) dump = pick(rng, STAT_NAMES)

for (const name of STAT_NAMES) {
  if (name === peak) stats[name] = high_value
  else if (name === dump) stats[name] = low_value
  else stats[name] = mid_value
}
```
**Benefit**: Creates distinct personalities; no companions are "well-rounded" (always have a specialty and weakness).

---

## 15. Summary: System Capabilities & Limitations

### What the System Does Well
1. **Persistent, deterministic companions** — each user gets the same species/stats forever
2. **Lightweight procedural generation** — 18 × 3 ASCII artworks, 5 stats, pre-computed
3. **Rich animation** — 500ms ticks, idle sequences, pet reactions, speech bubbles
4. **No persistence complexity** — bones regenerated, only soul stored
5. **Scalable display** — adapts to narrow/full terminals

### What's Not Implemented
1. **Leveling/progression** — stats are fixed at hatch
2. **Multiplayer interaction** — companions are solo
3. **Shiny rendering** — flag exists but not displayed
4. **Voice generation** — speech is text-only (no audio)
5. **Buddy-specific commands** — only `/buddy pet` and `/buddy` exist
6. **Persistent buddy actions** — no memory of past interactions
7. **Inspiration seed usage** — generated but unused in prompts

### Edge Cases Handled
- **Narrow terminals**: Collapses to face + quip
- **Fullscreen mode**: Floating bubble overlay instead of inline
- **Blinks**: Eyes replaced with `-` periodically
- **Fading**: Bubble dims in last 3 seconds before clearing
- **Hat logic**: Only injects if line 0 blank across all frames
- **Stat floors by rarity**: Legendary companions have higher minimum stats

---

## File-by-File Summary

| File | Key Responsibilities |
|------|----------------------|
| **types.ts** | 18 species (hex-encoded), 5 rarities (60/25/10/4/1 weight), 8 hats, 6 eyes, 5 stats, type definitions |
| **companion.ts** | Mulberry32 PRNG, FNV-1a hash, seeding, rarity/stat rollout, caching, userId lookup |
| **sprites.ts** | ASCII art for 54 frames (18 × 3), hat injection, frame cycling, face rendering |
| **CompanionSprite.tsx** | React animation component, idle/pet/excited frame logic, speech bubbles, width reservation |
| **prompt.ts** | Companion intro attachment (sent to Claude on first hatch) |
| **useBuddyNotification.tsx** | Rainbow teaser (April 1-7), buddy discovery trigger, companion muting logic |

---

## Conclusion

The Claude Code buddy system is a masterclass in **deterministic, low-cost companion generation**. By seeding from user ID, buddies are stable and reproducible without database storage. The procedural design (species × rarity × stats × eyes × hats) creates 18 × 5 × 6 × 8 × 2^1 ≈ 8,640 unique companion archetypes, yet requires only ~1,300 lines of code. The ASCII art is charming, the animation is snappy, and the system integrates cleanly with the modal architecture via React components and feature gates.

The buddy won't remember past interactions or learn from user behavior—but it doesn't need to. It's designed to be a **presence**, not a **peer**. A small creature that animates beside your input, nudges you with quips when you invoke it, and persists via a single JSON blob in your config file.
