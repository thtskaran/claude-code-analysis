# Plugin And Marketplace Contract

Generated: 2026-04-01
Extraction basis:
- plugin and marketplace schemas
- plugin markdown loader
- built-in plugin registry
- plugin install/update/enable/disable logic

## Bottom Line

The plugin system is already more than “load markdown commands from disk”.

It includes:

- marketplace trust and impersonation defenses
- plugin manifest schemas
- built-in plugins as a first-class source
- plugin-provided commands and skills
- scoped install / enable / disable / update operations
- per-plugin user configuration

This document is self-contained. It describes the trust model, manifest shape, naming rules, and lifecycle rules directly so the reader can understand the plugin subsystem without needing the source tree.

## 1. Marketplace Trust Model

### Reserved official marketplace names

Reserved names:

- `claude-code-marketplace`
- `claude-code-plugins`
- `claude-plugins-official`
- `anthropic-marketplace`
- `anthropic-plugins`
- `agent-skills`
- `life-sciences`
- `knowledge-work-plugins`

### Auto-update default

Marketplace auto-update defaults:

- `true` for official marketplaces
- `false` for non-official marketplaces
- except `knowledge-work-plugins`, which is reserved but does not auto-update by default

### Anti-impersonation checks

Anti-impersonation defenses:

- blocked-name pattern for “official anthropic/claude” impersonation
- non-ASCII marketplace names are rejected to prevent homograph attacks
- reserved names must come from the official `anthropics` GitHub org

### Reserved marketplace IDs

Marketplace names also cannot be:

- spaces
- path separators
- `..`
- `.`
- `inline`
- `builtin`

This prevents both filesystem abuse and namespace squatting.

## 2. Plugin Manifest Contract

The manifest metadata layer includes:

- `name`
- `version`
- `description`
- `author`
- `homepage`
- `repository`
- `license`
- `keywords`
- `dependencies`

Important dependency rule:

- bare dependency names are resolved against the declaring plugin’s own marketplace

### Effective top-level manifest shape

A plugin manifest can carry these top-level sections:

- metadata:
  `name`, `version`, `description`, `author`, `homepage`, `repository`, `license`, `keywords`, `dependencies`
- hooks:
  inline hooks or references to hook JSON files
- commands:
  one path, an array of paths, or an object mapping command names to metadata
- agents:
  extra markdown agent files
- skills:
  extra skill directories
- output styles:
  extra output-style files or directories
- channels:
  assistant-mode channel declarations tied to MCP servers
- MCP servers:
  inline config, JSON files, or MCP bundle references
- LSP servers:
  command, args, extension mapping, transport, env, initialization options, and settings
- settings:
  allowlisted plugin-provided settings that merge when enabled
- user configuration:
  prompted values saved in settings or secure storage and exposed as `${user_config.KEY}`

Important parsing behavior:

- unknown top-level manifest fields are stripped rather than treated as fatal
- nested configuration objects remain strict where typos are likely to be author mistakes
- this makes the system forward-compatible at the top level without making nested config loose

## 3. Hooks And MCP In Plugins

The plugin contract is not limited to commands.

The broader plugin schema also supports:

- plugin hook bundles via `PluginHooksSchema` (`328-339`)
- extra manifest hooks inline or via JSON file(s) (`348-373`)
- MCP server configuration via plugin manifest types elsewhere in the file

Built-in plugins can provide:

- skills
- hooks
- MCP servers

## 4. Plugin Commands And Skills Are Markdown-Native

### File discovery rules

- markdown files are collected recursively (`102-130`)
- directories containing `SKILL.md` become skill roots (`50-55`, `132-167`)
- when `SKILL.md` exists, sibling markdown files in that directory are ignored (`149-160`)

### Command naming

Plugin markdown names are built as:

- `plugin:command`
- `plugin:namespace:command`
- `plugin:skillDirectoryName` for `SKILL.md`

### Frontmatter-driven command behavior

Each plugin markdown command or skill can parse:

- `description`
- `allowed-tools`
- `argument-hint`
- `arguments`
- `when_to_use`
- `version`
- display `name`
- `model`
- `effort`
- `disable-model-invocation`
- `user-invocable`
- `shell`

### Variable substitution

The loader substitutes:

- `${CLAUDE_PLUGIN_ROOT}`
- `${CLAUDE_PLUGIN_DATA}`
- `${CLAUDE_SKILL_DIR}` for skill-mode content
- `${user_config.*}` for saved plugin options

This is what turns plugin markdown into a true runtime prompt surface, not just static text.

### What plugin markdown can express

A plugin markdown command or skill can carry:

- a user-facing description
- an allowed-tool allowlist
- argument names and an argument hint
- trigger guidance via `when_to_use`
- a default model and effort level
- a flag to disable automatic model invocation
- a visibility flag controlling whether users can invoke it directly
- optional shell-command execution inside the prompt template

Runtime substitutions include:

- plugin root path
- plugin data path
- skill directory path
- current session ID
- plugin user-config values

In other words, plugin markdown is not just documentation. It is a parametrized prompt program.

## 5. `CommandMetadata` Object Contract

The object-form command metadata contract supports rich command definitions.

Each command can define either:

- `source`: path to markdown
- `content`: inline markdown

but not both.

Optional metadata includes:

- `description`
- `argumentHint`
- `model`
- `allowedTools`

## 6. Built-In Plugins

Differences from bundled skills:

- they appear in `/plugin`
- users can enable or disable them
- they can provide skills, hooks, and MCP servers

Key mechanics:

- built-in plugin IDs are `{name}@builtin`
- enabled state comes from user settings or plugin defaults (`57-101`)
- enabled built-in plugins can inject skill commands (`104-120`)

So there are really three different “shipped by the binary” extension surfaces:

- hardcoded built-in commands
- bundled skills
- built-in plugins

## 7. Scoped Plugin Operations

Installable scopes (`67-72`):

- `user`
- `project`
- `local`

Update scopes (`73-78`) also include:

- `managed`

Important helper behavior:

- scope-specific project path is only defined for `project` and `local` (`102-107`)
- plugin enablement can exist at project scope even if the plugin was installed at user scope (`110-123`)
- settings search precedence for finding a plugin is `local > project > user` (`164-193`)

This means install scope and enablement scope are related but not identical concepts.

### Practical scope model

There are four distinct scope concepts visible in the system:

- `user`: personal install or enablement
- `project`: shared project-level declaration
- `local`: gitignored project-local override
- `managed`: admin-controlled install or update scope

Key behavior:

- install scope answers where the plugin is declared or materialized
- enablement scope answers which settings layer currently turns it on or off
- a plugin may be installed at user scope and still be enabled or overridden at project scope
- uninstalling from one scope does not necessarily stop the plugin if another scope still enables it

## 8. CLI Plugin Operations Are Telemetry-Aware

CLI plugin operations wrap plugin changes with:

- console output
- process exit behavior
- telemetry
- classified error handling

Notable behavior:

- failures emit `tengu_plugin_command_failed`
- success paths route plugin names through privileged `_PROTO_*` fields rather than general metadata

That is consistent with the repo’s broader privacy posture around plugin and MCP naming.

## 9. Settings Integration

The plugin contract also appears in `SettingsSchema`:

- `enabledPlugins`
- `extraKnownMarketplaces`
- `strictKnownMarketplaces`
- `blockedMarketplaces`
- `pluginConfigs`
- `strictPluginOnlyCustomization`

Taken together, this means plugin behavior is jointly defined by:

- marketplace manifests
- local plugin manifests
- plugin markdown/frontmatter
- settings and managed policy
- runtime loaders

## 10. Marketplace And Plugin Source Types

Marketplace source types supported by the system:

- direct URL
- GitHub repository
- generic git repository
- npm package
- local file
- local directory
- host pattern
- path pattern
- inline settings-defined marketplace

Plugin source types supported in marketplace entries:

- relative path inside the marketplace
- npm package
- Python package
- git URL
- GitHub repository
- git subdirectory inside a larger repository

This is a broad distribution model. The system is designed to ingest plugins from curated marketplaces, inline repo policy, package managers, and source repositories.

## 11. Built-In Plugins Are A Separate Product Surface

Built-in plugins differ from bundled skills in three important ways:

- they appear in `/plugin` UI under a built-in section
- they can be enabled or disabled by the user
- they can package multiple surfaces at once: skills, hooks, and MCP servers

Built-in plugin IDs use this format:

- `{name}@builtin`

Enabled state resolves in this order:

- explicit user setting
- plugin default
- otherwise enabled

That makes built-in plugins a true extension surface, not just another hardcoded command list.

## 12. Practical Reading Of The System

At a product level, the plugin system can be summarized as:

- a marketplace distribution layer
- a manifest-validated extension layer
- a markdown command/skill authoring layer
- a policy-controlled trust layer
- a settings-backed per-plugin config layer

That is enough structure to treat plugins as a major first-class subsystem, not a minor add-on.
