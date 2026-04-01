# Migration And Product-History Timeline

Generated: 2026-04-01
Extraction basis:
- shipped startup migrations

## Scope

This is a source-derived timeline of what the product has changed recently, based on shipped migration code.

Migrations are especially valuable because they encode:

- deprecated names
- old defaults
- new defaults
- rollout safety conditions
- what user state the team was willing to rewrite automatically

This document is meant to stand on its own. The migration list below is the shipped product-change history visible in the product startup path.

## 0. Complete Migration Inventory

The product startup path contains these 11 migrations:

- `migrateAutoUpdatesToSettings`
- `migrateBypassPermissionsAcceptedToSettings`
- `migrateEnableAllProjectMcpServersToSettings`
- `migrateFennecToOpus`
- `migrateLegacyOpusToCurrent`
- `migrateOpusToOpus1m`
- `migrateReplBridgeEnabledToRemoteControlAtStartup`
- `migrateSonnet1mToSonnet45`
- `migrateSonnet45ToSonnet46`
- `resetAutoModeOptInForDefaultOffer`
- `resetProToOpusDefault`

## 1. Settings-Store Migration Wave

### Auto-updates setting moved into settings.json

Behavior:

- if the user explicitly disabled auto-updates in old global config
- and that disablement was not due to native-protection logic
- migrate the preference into `userSettings.env.DISABLE_AUTOUPDATER = '1'`
- remove the old global-config keys afterwards

Product signal:

- updater policy moved from legacy config into the main settings system
- the migration also writes `DISABLE_AUTOUPDATER=1` into settings `env`, so updater policy now piggybacks on the broader env-backed configuration layer

### Bypass-permissions opt-in moved into settings.json

Behavior:

- old global config key `bypassPermissionsModeAccepted`
- becomes `skipDangerousModePermissionPrompt` in user settings

Product signal:

- permission-mode acknowledgements were normalized into the settings layer

### MCP approval state moved from project config to local settings

Behavior:

- migrate `enableAllProjectMcpServers`
- migrate `enabledMcpjsonServers`
- migrate `disabledMcpjsonServers`
- remove old project-config fields after successful migration

Product signal:

- MCP approval state was consolidated into the settings system
- enabled and disabled server lists are merged rather than blindly replaced, preserving prior local choices where possible

### `replBridgeEnabled` renamed to `remoteControlAtStartup`

Behavior:

- copy old key to new key if the new one is unset
- delete old key

Product signal:

- the user-facing product concept changed from “REPL bridge” to “Remote Control”

## 2. Model-Alias And Default Migration Wave

### Pro default shifted to Opus

Behavior:

- first-party Pro users on the default model are marked migrated
- timestamp is recorded so the UI can show a notification
- users with custom model settings are not rewritten, but migration is still marked complete

Product signal:

- “Opus default for Pro” was important enough to warrant explicit migration bookkeeping

### Legacy explicit Opus 4.0 / 4.1 strings collapsed to `opus`

Behavior:

- first-party users only
- only when legacy model remap is enabled
- explicit old Opus IDs become `opus`
- a migration timestamp is written to global config

Product signal:

- old explicit model IDs were considered obsolete, but the team still wanted a one-time UX notification

### `opus` promoted to `opus[1m]` for eligible users

Behavior:

- if Opus 1M merge is enabled
- and userSettings pinned exactly `opus`
- migrate to `opus[1m]`, unless that would now match the default and can be omitted

Product signal:

- “Opus 1M” became the merged/default experience for a subset of eligible users

### Removed `fennec` aliases remapped to Opus family

Behavior:

- `fennec-latest[1m] -> opus[1m]`
- `fennec-latest -> opus`
- `fennec-fast-latest -> opus[1m] + fastMode`
- `opus-4-5-fast -> opus[1m] + fastMode`

Product signal:

- `fennec` was a real alias family that has now been retired
- “fast” variants were folded into model + fast-mode state
- only `userSettings` are rewritten; project or policy pins are left untouched

### `sonnet[1m]` pinned to explicit Sonnet 4.5 1M

Behavior:

- `sonnet[1m]` became `sonnet-4-5-20250929[1m]`
- in-memory override is migrated too
- guarded by a global completion flag

Reason given in comments:

- after `sonnet` moved on to 4.6, old users who intended Sonnet 4.5 1M needed to stay pinned there
- the in-memory main-loop override is migrated too, so the running session matches the persisted setting

### Explicit Sonnet 4.5 strings remapped back to `sonnet` / `sonnet[1m]`

Behavior:

- first-party Pro / Max / Team Premium only
- explicit 4.5 strings become `sonnet` or `sonnet[1m]`
- migration timestamp is written for established users

Product signal:

- Sonnet 4.6 superseded 4.5 for eligible first-party subscribers

### Inventory of model-history changes implied by migrations

Taken together, the model migrations encode this alias history:

- `fennec-latest` became `opus`
- `fennec-latest[1m]` became `opus[1m]`
- `fennec-fast-latest` became `opus[1m]` plus `fastMode`
- `opus-4-5-fast` became `opus[1m]` plus `fastMode`
- explicit legacy Opus 4.0 and 4.1 strings collapsed to `opus`
- `opus` can be promoted to `opus[1m]` for eligible users
- `sonnet[1m]` was temporarily pinned to explicit Sonnet 4.5 1M
- explicit Sonnet 4.5 strings later collapsed back to `sonnet` or `sonnet[1m]` when 4.6 took over

## 3. Permission-Mode Migration Wave

### Auto-mode opt-in was reset to show a new “make default” offer

Behavior:

- classifier feature must be enabled
- auto mode must be in `enabled` state, not `opt-in`
- if the user had accepted the old prompt but did not make auto their default
- clear `skipAutoPermissionPrompt` to show the new dialog again

Product signal:

- the auto-mode product flow changed from a simpler opt-in to a richer dialog with default-mode semantics

## 4. Migration Design Patterns

These migrations follow a few repeatable patterns:

- rewrite only `userSettings` when the team wants to preserve project or policy overrides
- use global-config completion flags when a migration must run once even if the target value no longer exists
- write timestamps when the UI needs to show a one-time notification about a changed default
- skip or narrow migrations by cohort:
  first-party only, Pro only, Max only, Team Premium only, feature-gated only
- prefer silent compatibility cleanup over breaking old config

## 5. Product Narrative Implied By The Migrations

Reading the migrations together suggests this evolution path:

1. legacy config keys were consolidated into the main settings system
2. MCP approvals and remote-control naming were normalized
3. model aliases were aggressively simplified around `opus` and `sonnet`
4. long-context and fast-mode offerings were being folded into cleaner default aliases
5. permission UX around dangerous mode and auto mode was still actively evolving

## 6. Why These Files Matter

Migrations are one of the best product-history artifacts in the repo because they reveal:

- which names are legacy
- which defaults changed
- which user cohorts were targeted
- which distinctions matter to the product team

For reverse-engineering product evolution, startup migrations are a higher-signal artifact than most ordinary feature code because they encode deliberate, user-visible compatibility decisions.
