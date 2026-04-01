# Claude Code v2.1.88 Model Selection & Management System - Deep Dive Analysis

**Date**: 2026-04-02
**Source**: `/sessions/cool-friendly-einstein/mnt/claude-code/src/utils/model/` (16 files, 2,710 LOC)
**Analysis Focus**: Model selection algorithm, provider routing, context window management, token pricing, and design decisions.

---

## Executive Summary

Claude Code's model selection system is a sophisticated, multi-layered architecture designed to serve diverse user segments (Anthropic employees, Max subscribers, Pro/Team users, PAYG/API, 3P providers) with intelligent fallback strategies, subscription-aware pricing, and provider-agnostic model management.

The system supports **11 distinct Claude models** across 4 API providers (first-party, AWS Bedrock, Google Vertex AI, Azure Foundry). Model selection follows a strict priority hierarchy with runtime overrides, subscription-aware defaults, context window negotiation, and graceful degradation when models are unavailable.

Key architectural insight: **The system prioritizes correctness over simplicity through explicit provider routing**, separate model ID mappings per provider, capability overrides for 3P deployments, and codename masking to prevent internal model names from leaking to public releases.

---

## 1. Model Registry: Complete Supported Models

### 1.1 Model Catalog by Family

The model registry is defined in `/sessions/cool-friendly-einstein/mnt/claude-code/src/utils/model/configs.ts` with provider-specific strings for each model:

#### **Haiku Family** (Fastest/Cheapest)
| Key | First-Party ID | Bedrock | Vertex | Foundry | Type |
|-----|---|---|---|---|---|
| `haiku35` | `claude-3-5-haiku-20241022` | `us.anthropic.claude-3-5-haiku-20241022-v1:0` | `claude-3-5-haiku@20241022` | `claude-3-5-haiku` | Legacy |
| `haiku45` | `claude-haiku-4-5-20251001` | `us.anthropic.claude-haiku-4-5-20251001-v1:0` | `claude-haiku-4-5@20251001` | `claude-haiku-4-5` | Current |

#### **Sonnet Family** (Balanced)
| Key | First-Party ID | Bedrock | Vertex | Foundry | Type |
|-----|---|---|---|---|---|
| `sonnet35` | `claude-3-5-sonnet-20241022` | `anthropic.claude-3-5-sonnet-20241022-v2:0` | `claude-3-5-sonnet-v2@20241022` | `claude-3-5-sonnet` | Legacy |
| `sonnet37` | `claude-3-7-sonnet-20250219` | `us.anthropic.claude-3-7-sonnet-20250219-v1:0` | `claude-3-7-sonnet@20250219` | `claude-3-7-sonnet` | Legacy |
| `sonnet40` | `claude-sonnet-4-20250514` | `us.anthropic.claude-sonnet-4-20250514-v1:0` | `claude-sonnet-4@20250514` | `claude-sonnet-4` | Legacy |
| `sonnet45` | `claude-sonnet-4-5-20250929` | `us.anthropic.claude-sonnet-4-5-20250929-v1:0` | `claude-sonnet-4-5@20250929` | `claude-sonnet-4-5` | Legacy |
| `sonnet46` | `claude-sonnet-4-6` | `us.anthropic.claude-sonnet-4-6` | `claude-sonnet-4-6` | `claude-sonnet-4-6` | **Current** |

#### **Opus Family** (Most Capable)
| Key | First-Party ID | Bedrock | Vertex | Foundry | Type |
|-----|---|---|---|---|---|
| `opus40` | `claude-opus-4-20250514` | `us.anthropic.claude-opus-4-20250514-v1:0` | `claude-opus-4@20250514` | `claude-opus-4` | Legacy |
| `opus41` | `claude-opus-4-1-20250805` | `us.anthropic.claude-opus-4-1-20250805-v1:0` | `claude-opus-4-1@20250805` | `claude-opus-4-1` | Legacy |
| `opus45` | `claude-opus-4-5-20251101` | `us.anthropic.claude-opus-4-5-20251101-v1:0` | `claude-opus-4-5@20251101` | `claude-opus-4-5` | Legacy |
| `opus46` | `claude-opus-4-6` | `us.anthropic.claude-opus-4-6-v1` | `claude-opus-4-6` | `claude-opus-4-6` | **Current** |

### 1.2 Model ID Conventions Across Providers

**First-Party (Anthropic 1P API):**
- Format: `claude-{family}-{version}-{date}` or `claude-{family}-{version}` for latest
- Example: `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5-20251001`
- Behavior: Dates are typically `YYYYMMDD` format; non-dated versions auto-update

**AWS Bedrock:**
- Format: `us|eu|apac|global.anthropic.{model}-v{n}:0` (cross-region inference)
- Example: `us.anthropic.claude-opus-4-6-v1:0`
- Special: Foundation models use `anthropic.{model}` prefix (no region); inference profiles discovered at runtime via API

**Google Vertex AI:**
- Format: `{model}@{date}` (always dated for reproducibility)
- Example: `claude-sonnet-4-6@20250514`
- Behavior: Uses versioned snapshots rather than floating versions

**Azure Foundry:**
- Format: Plain model name (deployment ID user-configurable)
- Example: `claude-opus-4-6`, `claude-sonnet-4-6`
- Behavior: Foundry acts as pass-through; real ID is user-defined deployment name

### 1.3 Context Window & Token Limits

Context windows are determined dynamically via two mechanisms:

1. **Built-in Defaults** (hardcoded, fast path):
   - Opus/Sonnet/Haiku: 200K context by default
   - With `[1m]` suffix: 1M context window

2. **Ant-Only Model Capabilities Cache** (from API):
   - Location: `~/.claude/cache/model-capabilities.json`
   - Fetched from Anthropic API by ant users only
   - Updates periodically via `refreshModelCapabilities()`
   - Fields: `id`, `max_input_tokens`, `max_tokens`

The context window affects:
- **Autocompact threshold**: Triggers at 80% of effective context
- **Skill model resolution**: Carries `[1m]` suffix across model family when appropriate
- **Model-to-context mapping**: `resolveSkillModelOverride()` ensures downgrades only when necessary

---

## 2. Model Selection Algorithm

### 2.1 Priority Hierarchy (getMainLoopModel)

The model selection follows strict priority order in `model.ts:getMainLoopModel()`:

```
1. Session override via /model command (getMainLoopModelOverride)
   └─ Highest priority; can change any time during session

2. Startup override via --model flag (ANTHROPIC_MODEL env var)
   └─ Command-line override for entire process

3. Environment variable ANTHROPIC_MODEL
   └─ User can set globally per shell

4. Settings.model (from ~/.claude/settings.json)
   └─ Persistent user preference

5. getDefaultMainLoopModel() [Fallback]
   └─ Subscription & provider-dependent default
```

### 2.2 Default Model Selection by User Segment

Logic in `model.ts:getDefaultMainLoopModelSetting()` & `modelOptions.ts`:

#### **Anthropic Employees (USER_TYPE='ant')**
```javascript
Default: getAntModelOverrideConfig()?.defaultModel ?? getDefaultOpusModel() + '[1m]'
```
- Uses feature-flagged config from GrowthBook (`tengu_ant_model_override`)
- Falls back to Opus 1M if not configured
- Can specify custom models, default effort level, and system prompt suffix
- Model codenames masked during display (first 3 chars + asterisks)

#### **Max Subscribers & Team Premium Users**
```javascript
Default: getDefaultOpusModel() + (isOpus1mMergeEnabled() ? '[1m]' : '')
// Currently: claude-opus-4-6 or claude-opus-4-6[1m]
```
- Opus 4.6 is baseline
- 1M context enabled by default if `isOpus1mMergeEnabled()` returns true
- Can access Sonnet and Haiku as alternatives

#### **Pro/Team Standard/Enterprise Users**
```javascript
Default: getDefaultSonnetModel()
// Currently: claude-sonnet-4-6
```
- Sonnet 4.6 is baseline (best balance)
- Opus available as paid extra usage (if enabled)
- Can access Haiku for quick answers

#### **PAYG/API (Pay-As-You-Go) Users**
- **1P (First-Party)**: Sonnet 4.6 default, Opus/Haiku as alternatives
- **3P (Bedrock/Vertex/Foundry)**: May be on older default (Sonnet 4.5) if provider lags

### 2.3 Model Aliases

Aliases are defined in `aliases.ts` and enable user-friendly shortcuts:

```typescript
MODEL_ALIASES = [
  'sonnet',        // → getDefaultSonnetModel()
  'opus',          // → getDefaultOpusModel()
  'haiku',         // → getDefaultHaikuModel()
  'best',          // → getBestModel() [always Opus 4.6]
  'sonnet[1m]',    // → Sonnet with 1M context
  'opus[1m]',      // → Opus with 1M context
  'opusplan',      // → Opus in plan mode, Sonnet otherwise
]
```

**Special Behavior**:
- `opusplan`: Uses Opus only in plan/research mode; reverts to Sonnet in default mode
- `[1m]` suffix can be applied to any alias (haiku, sonnet, opus)
- Bare family aliases can be used in availability allowlist as wildcards

### 2.4 Model Alias Resolution (parseUserSpecifiedModel)

When user provides `sonnet`, the system:

1. Strips `[1m]` suffix if present
2. Resolves to current default for provider:
   - 1P Sonnet → `claude-sonnet-4-6`
   - 3P Sonnet → `claude-sonnet-4-5` (may lag)
3. Re-applies `[1m]` if original had it
4. For Ant users: resolves from `tengu_ant_model_override` config

This enables **floating-version semantics**: When user specifies `opus`, they always get the current recommended Opus, even if that changes on server side.

### 2.5 Runtime Model Selection (getRuntimeMainLoopModel)

Special behavior for specific modes:

```javascript
// opusplan in plan mode without exceeding 200K tokens → use Opus
if (getUserSpecifiedModelSetting() === 'opusplan' &&
    permissionMode === 'plan' &&
    !exceeds200kTokens)
  return getDefaultOpusModel()

// haiku in plan mode → downgrade to Sonnet (plan reasoning needs more capability)
if (getUserSpecifiedModelSetting() === 'haiku' && permissionMode === 'plan')
  return getDefaultSonnetModel()

// Otherwise → use mainLoopModel as-is
```

---

## 3. Provider Routing Architecture

### 3.1 Provider Detection (getAPIProvider)

Determined at startup via environment variables in priority order:

```javascript
if (CLAUDE_CODE_USE_BEDROCK) return 'bedrock'
if (CLAUDE_CODE_USE_VERTEX) return 'vertex'
if (CLAUDE_CODE_USE_FOUNDRY) return 'foundry'
return 'firstParty'  // default
```

Once set, provider is **immutable** for the session.

### 3.2 Provider-Specific Model String Resolution

**Flow: model.ts → modelStrings.ts → configs.ts**

1. **Get base model strings** (from `ALL_MODEL_CONFIGS`):
   ```typescript
   const sonnet46 = ALL_MODEL_CONFIGS['sonnet46'][provider]
   // Returns: claude-sonnet-4-6 (1P),
   //          us.anthropic.claude-sonnet-4-6 (Bedrock),
   //          claude-sonnet-4-6@20250514 (Vertex),
   //          claude-sonnet-4-6 (Foundry)
   ```

2. **For Bedrock**: Fetch inference profile list at runtime
   - Queries AWS Bedrock API with `ListInferenceProfilesCommand`
   - Looks for profiles matching the canonical first-party ID
   - Falls back to hardcoded Bedrock ID if profile not found
   - Example: "eu.anthropic.claude-sonnet-4-6" discovered → used instead of hardcoded

3. **Apply user overrides** (from settings.json `modelOverrides`):
   ```json
   {
     "modelOverrides": {
       "claude-opus-4-6": "arn:aws:bedrock:us-east-1:...:inference-profile/custom-opus"
     }
   }
   ```

4. **Result**: Provider-specific, user-customizable model ID

### 3.3 Bedrock Cross-Region Inference Routing

Bedrock supports cross-region inference prefixes. When user specifies a region:

```typescript
function applyBedrockRegionPrefix(modelId: string, prefix: 'us'|'eu'|'apac'|'global'): string
```

Example workflow:
1. User is on `eu.anthropic.claude-sonnet-4-6` (EU region)
2. Spawns subagent with `model: haiku` (alias)
3. `getAgentModel()` → inherits parent region prefix
4. Result: `eu.anthropic.claude-haiku-4-5` (not us.anthropic.claude-haiku-4-5)
5. Ensures IAM permissions scoped to specific region still work

### 3.4 First-Party vs 3P Divergence Points

The system treats 1P and 3P as distinct:

**1P Behavior** (`getAPIProvider() === 'firstParty'`):
- Uses latest model versions: Opus 4.6, Sonnet 4.6, Haiku 4.5
- Supports 1M context on Opus/Sonnet without gating
- No custom model configuration needed
- Bedrock profile discovery skipped

**3P Behavior** (bedrock/vertex/foundry):
- May use older defaults (Sonnet 4.5 still default on Bedrock)
- Requires explicit env vars for custom models:
  - `ANTHROPIC_DEFAULT_OPUS_MODEL` → deployed model ID
  - `ANTHROPIC_DEFAULT_SONNET_MODEL` → deployed model ID
  - `ANTHROPIC_DEFAULT_HAIKU_MODEL` → deployed model ID
  - `ANTHROPIC_DEFAULT_*_MODEL_SUPPORTED_CAPABILITIES` → comma-separated capabilities
- Bedrock requires AWS credentials and region configuration
- Vertex requires GCP credentials and project

---

## 4. Context Window Management

### 4.1 Context Window Determination

**Priority 1: Explicit suffix**
```javascript
const model = 'sonnet[1m]'  // User specified → 1M context
const model = 'sonnet'      // No suffix → 200K context
```

**Priority 2: Model capability cache** (Ant users only)
```javascript
modelCapability = getModelCapability('claude-opus-4-6')
// Returns: { id: 'claude-opus-4-6', max_input_tokens: 1048576, max_tokens: 16000 }
// Used for accurate autocompact calculation
```

**Priority 3: Built-in defaults**
```javascript
// All models: 200K by default
// With [1m]: 1M override
```

### 4.2 Autocompact Threshold Calculation

The system uses context window to determine when to trigger compression:

```javascript
// Pseudocode
effectiveContextWindow = contextWindow(model)  // 200K or 1M
autocompactThreshold = effectiveContextWindow * 0.8  // 80% trigger

if (currentTokens > autocompactThreshold) {
  triggerAutoCompact()
}
```

**Critical Design Point**: If a user on Opus[1M] (1M context) is mistakenly shown the 200K context UI, they would see 80% * 200K = 160K as warning threshold. At 230K tokens, they'd incorrectly see "context limit reached" even though 1M was available. This is why context window MUST be determined correctly before setting any UI warnings.

### 4.3 Skill Model Override Handling

When a skill specifies `model: opus`, the system prevents unintended downgrades:

```typescript
function resolveSkillModelOverride(skillModel: string, currentModel: string): string
```

**Scenario**: User is on `opus[1m]` with 230K tokens. Skill specifies `model: opus`.

**Incorrect behavior**: Would resolve to `opus` → lose 1M context → autocompact triggers incorrectly

**Correct behavior** (implemented):
1. Parse skill model: `opus`
2. Parse current model: `opus[1m]`
3. Check if current has `[1m]`: YES
4. Check if skill model family supports 1M: `opus` → YES (Opus always supports 1M)
5. Return: `opus[1m]` (carry suffix forward)

**Result**: Skill downgrades context only when necessary (e.g., `model: haiku` → haiku doesn't support 1M, so return bare haiku).

---

## 5. Token Counting & Pricing

### 5.1 Pricing Tiers

Pricing constants are in `modelCost.ts`:

```typescript
COST_TIER_3_15 = { input: 3, output: 15 }      // Sonnet 4.6
COST_HAIKU_45 = { input: 0.8, output: 4 }     // Haiku 4.5
COST_HAIKU_35 = { input: 0.8, output: 4 }     // Haiku 3.5
// More tiers defined for Opus, older Sonnet/Haiku versions
```

Displayed as formatted strings in UI:
```
"Sonnet 4.6 · Best for everyday tasks · $0.003/$0.015 per 1K tokens"
```

### 5.2 Model Cost Lookup

`getOpus46CostTier(fastMode: boolean)` returns pricing:

```typescript
if (fastMode) {
  return dynamicCostTier(opus46, fastMode)  // May vary
} else {
  return standardCostTier(opus46)  // $3 input / $15 output
}
```

Fast mode may use batching or lower-cost computation paths.

### 5.3 Token Counting

Token counting is delegated to:
1. **Anthropic SDK**: `tokenCounter.countTokens(messages)` for user-facing estimates
2. **Backend**: Actual token count returned in API responses
3. **Context window management**: Used for autocompact calculations

The system normalizes model IDs before counting:
```typescript
function normalizeModelStringForAPI(model: string): string {
  return model.replace(/\[(1|2)m\]/gi, '')  // Strip 1m/2m suffix for API
}
```

---

## 6. Model Codenames & Ant-Only Features

### 6.1 Codename Guard System

The comment at top of `model.ts` reveals:

```javascript
/**
 * Ensure that any model codenames introduced here are also added to
 * scripts/excluded-strings.txt to avoid leaking them. Wrap any codename string
 * literals with process.env.USER_TYPE === 'ant' for Bun to remove the codenames
 * during dead code elimination
 */
```

**Purpose**: Prevent internal model codenames (capybara, panda, etc.) from leaking to public/external builds.

**Mechanism**:
1. Ant user specifies `model: capybara-v2` (codename)
2. Resolved via `getAntModelOverrideConfig()` → maps to actual model ID
3. Displayed masked: `cap*****-v2` (first 3 chars + asterisks)
4. During production build: `if (process.env.USER_TYPE === 'ant')` wraps codename branches → Bun dead-code eliminates → no trace in public bundle

### 6.2 Ant Model Configuration

Ant models come from feature flag `tengu_ant_model_override` in GrowthBook:

```typescript
interface AntModelOverrideConfig {
  defaultModel?: string              // e.g., "capybara-v2[1m]"
  defaultModelEffortLevel?: 'light'|'normal'|'extended'|'max'
  defaultSystemPromptSuffix?: string
  antModels?: AntModel[]
  switchCallout?: { modelAlias?: string, description: string, version: string }
}

interface AntModel {
  alias: string                // "cb2", "panda"
  model: string               // Full model ID
  label: string               // UI display: "Capybara v2"
  description?: string        // UI description
  contextWindow?: number      // Optional override
  defaultMaxTokens?: number   // Optional override
  defaultEffortValue?: number // Effort level
  alwaysOnThinking?: boolean  // Force adaptive thinking
}
```

**Example**:
```json
{
  "defaultModel": "capybara-v2[1m]",
  "antModels": [
    {
      "alias": "cb2",
      "model": "anthropic.capybara-v2-experimental",
      "label": "Capybara v2",
      "contextWindow": 2000000,
      "alwaysOnThinking": true
    }
  ]
}
```

Model picker displays these as options; users can switch with `/model cb2`.

---

## 7. Fallback Strategies & Error Handling

### 7.1 Legacy Opus Remap

When user has pinned to old Opus versions:

```typescript
const LEGACY_OPUS_FIRSTPARTY = [
  'claude-opus-4-20250514',
  'claude-opus-4-1-20250805',
  'claude-opus-4-0',
  'claude-opus-4-1',
]

if (isLegacyOpusFirstParty(model) && isLegacyModelRemapEnabled()) {
  return getDefaultOpusModel() + (has1mTag ? '[1m]' : '')
  // Remaps to claude-opus-4-6[1m] (current)
}
```

**Rationale**: Opus 4.0/4.1 retired from 1P API; silently upgrade users to 4.6.

**Opt-out**: `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP=true` env var.

**3P Handling**: 3P users get the old ID passed through (provider may or may not support it).

### 7.2 3P Model Fallback Chain

If a 3P user specifies an unavailable model, `validateModel()` suggests a fallback:

```typescript
function get3PFallbackSuggestion(model: string): string | undefined {
  if (lowerModel.includes('opus-4-6')) return getModelStrings().opus41
  if (lowerModel.includes('sonnet-4-6')) return getModelStrings().sonnet45
  if (lowerModel.includes('sonnet-4-5')) return getModelStrings().sonnet40
}
```

**Flow**:
1. User tries `opus-4-6` on Bedrock
2. API returns 404 NotFoundError
3. System suggests: "Try 'claude-opus-4-1-20250805' instead"

### 7.3 Model Allowlist Filtering

Organizations can restrict available models via `settings.json`:

```json
{
  "availableModels": ["opus", "sonnet", "opus-4-5"]
}
```

**Matching rules** (in `modelAllowlist.ts`):
1. **Family wildcard**: `opus` allows all Opus versions (unless narrowed)
2. **Version prefix**: `opus-4-5` matches `claude-opus-4-5-20251101`
3. **Exact match**: `claude-opus-4-6` matches only that exact ID
4. **Narrowing**: If both `opus` and `opus-4-5` present, only 4.5 is allowed

**Enforcement**:
```typescript
if (!isModelAllowed(userSpecifiedModel)) {
  return undefined  // Model filtered out, use default
}
```

### 7.4 Model Validation

Before using a model, the system validates:

```typescript
async function validateModel(model: string): {
  // 1. Check allowlist
  if (!isModelAllowed(model)) return { valid: false, error: '...' }

  // 2. Check aliases
  if (MODEL_ALIASES.includes(model)) return { valid: true }

  // 3. Check cache
  if (validModelCache.has(model)) return { valid: true }

  // 4. Make test API call
  try {
    await sideQuery({
      model,
      max_tokens: 1,
      messages: [{ role: 'user', content: [{ type: 'text', text: 'Hi' }] }]
    })
    return { valid: true }
  } catch (error) {
    // Handle NotFoundError, AuthenticationError, etc.
  }
}
```

---

## 8. Fast Mode Model Selection

Fast mode uses smaller/faster models for quick responses:

```typescript
function getSmallFastModel(): ModelName {
  return process.env.ANTHROPIC_SMALL_FAST_MODEL || getDefaultHaikuModel()
}
```

**Default**: Haiku 4.5 (fastest)

**Override**: Set `ANTHROPIC_SMALL_FAST_MODEL=claude-haiku-4-5` to pin to specific version.

**Region override**: For small fast model on Bedrock, can set:
- `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION=eu-west-1`

**Use cases**:
- Code suggestions/completion
- Syntax validation
- Quick fact checks
- Batch processing

**Pricing display**: Fast mode adds lightning bolt emoji and pricing to model picker.

---

## 9. Access Control & Subscription Gating

### 9.1 1M Context Access

`check1mAccess.ts` determines who can use models with `[1m]` suffix:

```typescript
function checkOpus1mAccess(): boolean {
  if (is1mContextDisabled()) return false  // Globally disabled

  if (isClaudeAISubscriber()) {
    // Only if extra usage is enabled
    return isExtraUsageEnabled()
  }

  // Non-subscribers (API/PAYG) have access
  return true
}

function checkSonnet1mAccess(): boolean {
  // Same logic
}
```

**Extra usage enabled when**:
- `cachedExtraUsageDisabledReason === null` (explicitly enabled)
- `cachedExtraUsageDisabledReason === 'out_of_credits'` (provisioned but depleted)

**Extra usage disabled when**:
- `overage_not_provisioned`
- `org_level_disabled`
- `seat_tier_level_disabled`
- etc.

### 9.2 Model Merge Decision

For Max/Team Premium users, Opus 1M can be merged into the default:

```typescript
function isOpus1mMergeEnabled(): boolean {
  // Pro users can't merge (don't get 1M)
  if (isProSubscriber()) return false

  // 3P users can't merge (provider may not support)
  if (getAPIProvider() !== 'firstParty') return false

  // CloudAI subscribers need valid subscription type (prevent stale tokens)
  if (isClaudeAISubscriber() && getSubscriptionType() === null) return false

  return true
}
```

**When merged**: Max users see "Opus 4.6 (1M context)" as default, not separate option.

**When not merged**: Separate "Opus (1M context)" option in picker.

---

## 10. Agent/Subagent Model Routing

### 10.1 Agent Model Options

Agents (subagents) have simpler model picker: `inherit`, `sonnet`, `opus`, `haiku`.

```typescript
const AGENT_MODEL_OPTIONS = [...MODEL_ALIASES, 'inherit']
// ['sonnet', 'opus', 'haiku', 'best', 'sonnet[1m]', 'opus[1m]', 'opusplan', 'inherit']
```

**Default**: `inherit` (use parent model)

### 10.2 Smart Tier Matching

When subagent specifies `model: opus` but parent is already Opus:

```typescript
function aliasMatchesParentTier(alias: string, parentModel: string): boolean {
  const canonical = getCanonicalName(parentModel)
  if (alias === 'opus' && canonical.includes('opus')) {
    return true  // Inherit parent's exact Opus variant
  }
}
```

**Prevents surprises**: Vertex user on Opus 4.6 spawns subagent with `model: opus` → gets Opus 4.6, not Bedrock's Opus 4.1 (if older).

### 10.3 Region Prefix Inheritance

For Bedrock cross-region inference:

```typescript
const parentRegionPrefix = getBedrockRegionPrefix(parentModel)
// "eu" from "eu.anthropic.claude-opus-4-6"

if (parentRegionPrefix && getAPIProvider() === 'bedrock') {
  resolvedModel = applyBedrockRegionPrefix(resolvedModel, parentRegionPrefix)
  // sonnet → eu.anthropic.claude-sonnet-4-6
}
```

**Ensures**: Subagent uses same region as parent (required for IAM scoping).

---

## 11. Deprecation Tracking

Models retire on provider-specific dates:

```typescript
const DEPRECATED_MODELS = {
  'claude-3-opus': {
    retirementDates: {
      firstParty: 'January 5, 2026',
      bedrock: 'January 15, 2026',
      vertex: 'January 5, 2026',
      foundry: 'January 5, 2026',
    }
  },
  'claude-3-7-sonnet': { /* dates */ },
  'claude-3-5-haiku': { /* dates */ },
}
```

**User-facing warning**:
```
⚠ Claude 3.7 Sonnet will be retired on February 19, 2026.
Consider switching to a newer model.
```

Warnings suppress when user has no choice (e.g., 3P provider only offers deprecated model).

---

## 12. Model Capabilities & Thinking Support

### 12.1 Capability Overrides for 3P

3P deployments may support different features than 1P:

```typescript
export type ModelCapabilityOverride =
  | 'effort'
  | 'max_effort'
  | 'thinking'
  | 'adaptive_thinking'
  | 'interleaved_thinking'

const TIERS = [
  { modelEnvVar: 'ANTHROPIC_DEFAULT_OPUS_MODEL',
    capabilitiesEnvVar: 'ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES' }
]

// Example:
// ANTHROPIC_DEFAULT_OPUS_MODEL=my-custom-opus
// ANTHROPIC_DEFAULT_OPUS_MODEL_SUPPORTED_CAPABILITIES=effort,thinking
// → Custom Opus supports effort levels and basic thinking
```

**Use case**: Azure Foundry may not support full Extended Thinking; set `effort,thinking` to restrict UI.

### 12.2 Ant Model Thinking Override

Certain internal models force adaptive thinking:

```typescript
interface AntModel {
  alwaysOnThinking?: boolean  // true = reject `thinking: { type: 'disabled' }`
}
```

When `true`: User can't disable thinking; requests always include thinking block.

---

## 13. Model String Management & Caching

### 13.1 Model Strings Lifecycle

`modelStrings.ts` manages provider-specific ID resolution:

1. **Initial state**: `null` (uninitialized)
2. **Init** → `setModelStringsState(strings)` from provider defaults
3. **For Bedrock**: Async fetch of inference profiles in background
4. **Fallback**: Use hardcoded if fetch fails

**Bedrock profile discovery**:
```typescript
async getBedrockInferenceProfiles(): Promise<string[]> {
  // Query AWS Bedrock API ListInferenceProfilesCommand
  // Filter for SYSTEM_DEFINED profiles with "anthropic" in ID
  // Returns: ["us.anthropic.claude-opus-4-6-v1", "eu.anthropic.claude-sonnet-4-6", ...]
}

function findFirstMatch(profiles: string[], substring: string): string | null {
  // Match "claude-opus-4-6" against profiles → first match
  // "eu.anthropic.claude-opus-4-6-v1" matches → return it
}
```

### 13.2 Model Overrides in Settings

User can remap any model to custom ID:

```typescript
// ~/.claude/settings.json
{
  "modelOverrides": {
    "claude-opus-4-6": "arn:aws:bedrock:us-east-1:123:inference-profile/my-opus",
    "claude-sonnet-4-6": "arn:aws:bedrock:eu-west-1:456:inference-profile/my-sonnet"
  }
}
```

Applied at resolution time:
```typescript
function applyModelOverrides(ms: ModelStrings): ModelStrings {
  const overrides = getInitialSettings().modelOverrides
  for (const [canonicalId, override] of Object.entries(overrides)) {
    out[key] = override  // Replace hardcoded with custom
  }
  return out
}
```

---

## 14. Display & UI Rendering

### 14.1 Public Model Display Names

```typescript
function getPublicModelDisplayName(model: ModelName): string | null {
  switch (model) {
    case getModelStrings().opus46: return 'Opus 4.6'
    case getModelStrings().opus46 + '[1m]': return 'Opus 4.6 (1M context)'
    case getModelStrings().sonnet46: return 'Sonnet 4.6'
    case getModelStrings().haiku45: return 'Haiku 4.5'
    // ... all public models
    default: return null
  }
}
```

Returns `null` for unknown/internal models.

### 14.2 Internal Model Rendering (Ant Only)

For internal models, show masked codename:

```typescript
function renderModelName(model: ModelName): string {
  const publicName = getPublicModelDisplayName(model)
  if (publicName) return publicName

  if (process.env.USER_TYPE === 'ant') {
    const antModel = resolveAntModel(model)
    if (antModel) {
      const masked = maskModelCodename(antModel.model)
      // capybara-v2-fast → cap*****-v2-fast
      return masked + (has1mContext(model) ? '[1m]' : '')
    }
  }
  return model  // Fallback
}
```

### 14.3 Model Picker Population

`getModelOptions(fastMode)` builds the picker list based on user segment:

**For Ants**:
```
├─ Default (from tengu_ant_model_override)
├─ [Custom models from antModels config]
├─ Merged Opus[1m]
├─ Sonnet 4.6
├─ Sonnet[1m]
└─ Haiku 4.5
```

**For Max/Team Premium**:
```
├─ Default (Opus or Opus[1m])
├─ Sonnet 4.6
├─ [Sonnet[1m] if access]
├─ Haiku 4.5
└─ [Opus[1m] if not merged & access]
```

**For PAYG 1P**:
```
├─ Default (Sonnet)
├─ [Sonnet[1m] if access]
├─ Opus (or Merged Opus[1m])
├─ [Opus[1m] if separate & access]
└─ Haiku 4.5
```

**For PAYG 3P**:
```
├─ Default (Sonnet 4.5 on Bedrock)
├─ [Custom Sonnet if ANTHROPIC_DEFAULT_SONNET_MODEL]
├─ Sonnet 4.6 (if different from default)
├─ [Custom Opus if set]
├─ Opus 4.1 + Opus 4.6 (if not custom)
├─ [Custom Haiku if set]
└─ Haiku (auto-selects 4.5 or 3.5 based on provider)
```

**Additional options**:
- Environment custom model: `ANTHROPIC_CUSTOM_MODEL_OPTION`
- Bootstrap additional options: `getGlobalConfig().additionalModelOptionsCache`
- Current/initial model if not in list (with deprecation hint)

---

## 15. Security & Attack Vectors

### 15.1 Model Override Injection

**Vector**: User-controlled settings could force expensive model:

```json
{ "model": "opus" }  // Upgrade 1P PAYG user to Opus at every session
```

**Mitigation**:
- Model allowlist enforced: `isModelAllowed()` blocks unauthorized models
- Settings loaded only from trusted sources (local ~/.claude/)
- /model command sanitized: goes through `parseUserSpecifiedModel()` → validates

### 15.2 Codename Leakage

**Vector**: Internal model codenames (capybara-v2, panda-next) visible in:
- Error messages
- Settings files
- Git commits
- Logs

**Mitigations**:
1. Dead code elimination: Codenames wrapped in `if (process.env.USER_TYPE === 'ant')`
2. Excluded strings: `scripts/excluded-strings.txt` prevents accidental inclusion
3. Display masking: `maskModelCodename()` shows `cap*****-v2` not full name
4. Settings isolation: Codenames never persisted to user-visible settings

### 15.3 Provider Credential Escalation

**Vector**: User gains access to higher-tier model on 3P via Bedrock ARN forgery:

```json
{
  "modelOverrides": {
    "claude-sonnet-4-6": "arn:aws:bedrock:us-east-1:admin-account:inference-profile/premium-opus"
  }
}
```

**Mitigation**:
- ARN must resolve to actual backing model via AWS API
- Model validation: `validateModel()` makes test API call
- AWS IAM enforced: User's credentials must have permission

### 15.4 Context Window Disclosure

**Vector**: User tricks system into showing 200K threshold when 1M available:

```javascript
// Attacker crafts custom model string
settings.model = "claude-sonnet-4-6" // No [1m] suffix
// But actually supports 1M on this provider
```

**Mitigation**:
- Context window always determined from model ID format (presence of `[1m]`)
- Capability cache used to verify (Ant users)
- Autocompact based on true window, not display

### 15.5 Allowlist Bypass

**Vector**: User edits settings to add unauthorized model:

```json
{
  "availableModels": ["unauthorized-model"]
}
```

**Mitigation**:
- Settings validated on load
- Only specific fields honored (availableModels must be string array)
- Allowlist enforced at model selection time
- /model command subject to same allowlist

---

## 16. Configuration & Environment Variables

### 16.1 Model Override Environment Variables

| Variable | Purpose | Format | Provider |
|----------|---------|--------|----------|
| `ANTHROPIC_MODEL` | Default model | alias or ID | Any |
| `CLAUDE_CODE_USE_BEDROCK` | Switch to Bedrock | `true` | Bedrock |
| `CLAUDE_CODE_USE_VERTEX` | Switch to Vertex | `true` | Vertex |
| `CLAUDE_CODE_USE_FOUNDRY` | Switch to Foundry | `true` | Foundry |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Opus model ID | Model string | 3P |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Sonnet model ID | Model string | 3P |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Haiku model ID | Model string | 3P |
| `ANTHROPIC_DEFAULT_*_MODEL_SUPPORTED_CAPABILITIES` | Feature gates | CSV | 3P |
| `ANTHROPIC_CUSTOM_MODEL_OPTION` | Custom option | Model string | Any |
| `ANTHROPIC_CUSTOM_MODEL_OPTION_NAME` | Display name | String | Any |
| `ANTHROPIC_SMALL_FAST_MODEL` | Fast model | Model ID | Any |
| `ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION` | Fast model region | `us-east-1` | Bedrock |
| `CLAUDE_CODE_DISABLE_LEGACY_MODEL_REMAP` | Disable auto-upgrade | `true` | 1P |
| `ANTHROPIC_BEDROCK_BASE_URL` | Bedrock endpoint | URL | Bedrock |
| `ANTHROPIC_BASE_URL` | Custom API endpoint | URL | Any |
| `AWS_REGION` / `AWS_DEFAULT_REGION` | AWS region | `us-east-1` | Bedrock |
| `AWS_BEARER_TOKEN_BEDROCK` | Bedrock auth | Token | Bedrock |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | Skip AWS auth | `true` | Bedrock |
| `USER_TYPE` | User segment | `ant` or undefined | Any |

### 16.2 Feature Flags (Ant Only)

Fetched from GrowthBook:

| Flag | Type | Purpose |
|------|------|---------|
| `tengu_ant_model_override` | AntModelOverrideConfig | Define available internal models |

### 16.3 Settings File

`~/.claude/settings.json`:

```json
{
  "model": "opus[1m]",
  "availableModels": ["opus", "sonnet"],
  "modelOverrides": {
    "claude-opus-4-6": "arn:aws:bedrock:us-east-1:123:inference-profile/custom"
  }
}
```

---

## 17. Model Launch Markers

Throughout the codebase, look for `@[MODEL LAUNCH]` comments to find all locations requiring updates when adding/updating models:

```typescript
// @[MODEL LAUNCH]: Update the default Opus model (3P providers may lag so keep defaults unchanged).
export function getDefaultOpusModel(): ModelName {
  if (process.env.ANTHROPIC_DEFAULT_OPUS_MODEL) {
    return process.env.ANTHROPIC_DEFAULT_OPUS_MODEL
  }
  if (getAPIProvider() !== 'firstParty') {
    return getModelStrings().opus46
  }
  return getModelStrings().opus46
}
```

**Checklist for new model**:
1. Add `CLAUDE_*_CONFIG` in `configs.ts`
2. Register in `ALL_MODEL_CONFIGS`
3. Update `getDefaultOpusModel()` / `getDefaultSonnetModel()` / `getDefaultHaikuModel()`
4. Add display name in `getPublicModelDisplayName()`
5. Add canonical name case in `firstPartyNameToCanonical()`
6. Update `getModelOptions()` helper functions
7. Add marketing name in `getMarketingNameForModel()`
8. Update deprecation dates in `deprecation.ts`
9. Update fallback chain in `validateModel()`
10. Add to `excluded-strings.txt` if codename
11. Update skill model override logic if needed

---

## 18. Conclusion & Architecture Insights

**Key Design Principles**:

1. **Subscriber-First Defaults**: Different user tiers get different defaults (Opus for Max, Sonnet for PAYG)
2. **Provider Abstraction**: Single model config; provider-specific strings resolved at runtime
3. **Explicit over Implicit**: Model IDs never guessed; always derived from config or user input
4. **Fail Closed**: Unknown/unauthorized models blocked; fallback only to known safe options
5. **Deprecation Path**: Old model usage triggers upgrade suggestions, not errors
6. **Flexibility for Enterprise**: 3P deployments can customize any model; system respects Bedrock profiles
7. **Correctness over Performance**: Context window determined early; multiple safeguards prevent context-based errors
8. **Segmentation**: Ant, Max, Pro, PAYG 1P, PAYG 3P treated distinctly where necessary

**Complexity Drivers**:

1. Multi-provider support (1P, Bedrock, Vertex, Foundry)
2. Subscription-aware pricing & capabilities
3. Dynamic model versions (aliases vs. pinned versions)
4. Internal development models (codenamesmasking)
5. Cross-region inference (Bedrock)
6. Legacy model remapping
7. Capability overrides per deployment

This is a production-grade, highly configurable system designed to serve internal and external users on multiple cloud providers while maintaining security, performance, and user experience.

---

**Analysis completed**: 2026-04-02 | **Source LOC**: 2,710 across 16 files
