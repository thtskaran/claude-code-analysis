# Bundled Skills Catalog

Generated: 2026-04-01
Extraction basis:
- bundled skill registry
- bundled skill definitions

## Scope

This document catalogs the skills that ship in the product binary.

It does not cover:

- user-installed skills
- marketplace plugin skills
- built-in plugin skills

It is meant to be self-contained. The descriptions below explain what each skill is for, what kind of workflow it automates, and what makes it materially different from the others.

## 1. What A Bundled Skill Is

A bundled skill is a shipped workflow prompt that the product already knows how to invoke. It is not:

- a low-level execution tool
- a user-installed extension
- a marketplace plugin

Instead, it is a reusable operating procedure packaged with the product. A skill usually does one of these things:

- teaches the model a built-in workflow
- narrows the allowed tools for that workflow
- changes how the model should interact with the user for that workflow
- packages domain-specific guidance or documentation for a recurring task

## 2. Always-Available Bundled Skills

### `update-config`

- purpose: configure Claude Code via `settings.json`
- focuses on hooks, permissions, env vars, and settings troubleshooting
- allowed tools: `Read`
- user-invocable: yes
- practical meaning:
  this is the built-in settings expert. It exists to explain how Claude Code is configured and how to change that configuration safely.

### `keybindings-help`

- purpose: help customize keyboard shortcuts and `~/.claude/keybindings.json`
- allowed tools: `Read`
- user-invocable: no
- enabled only when keybinding customization is enabled
- practical meaning:
  this is a hidden helper skill used when the product needs to explain or generate keyboard shortcut configuration.

### `verify`

- purpose: verify that a change actually works by running or exercising the thing that changed
- user-invocable: yes
- bundled reference files:
  `examples/cli.md`, `examples/server.md`
- practical meaning:
  this is the product’s built-in “prove it works” workflow. It pushes the model away from static reasoning and toward real validation through app runs, CLI checks, or other concrete verification steps.

### `debug`

- purpose:
  - ant build: debug the current Claude Code session by reading session debug logs
  - external build: enable debug logging and diagnose issues
- allowed tools: `Read`, `Grep`, `Glob`
- disables automatic model invocation
- user-invocable: yes
- practical meaning:
  this is the built-in incident triage workflow for Claude Code itself, not for arbitrary app debugging in general.

### `lorem-ipsum`

- purpose: generate filler text for long-context testing
- argument: token count
- user-invocable: yes
- ant-only in practice
- practical meaning:
  this is a utility skill used to manufacture large amounts of text for harness, prompt-window, or context-behavior testing.

### `skillify`

- purpose: turn a successful session workflow into a reusable custom skill
- allowed tools:
  - `Read`
  - `Write`
  - `Edit`
  - `Glob`
  - `Grep`
  - `AskUserQuestion`
  - `Bash(mkdir:*)`
- disables automatic model invocation
- user-invocable: yes
- practical meaning:
  `skillify` is a workflow-capture and packaging assistant. It studies the current session, identifies the repeatable process the user and model just went through, interviews the user about the desired reusable version, drafts a `SKILL.md`, asks for confirmation, and saves the new skill either to the repo or the user’s personal skills directory.
- what a reader should understand without source access:
  `skillify` is how the product teaches itself new workflows from real usage. It is a built-in skill authoring assistant.

### `remember`

- purpose: review auto-memory entries and promote or reconcile them across memory layers
- user-invocable: yes
- enabled only when auto-memory is enabled
- practical meaning:
  this is the memory curation workflow. It lets the model inspect what has been auto-learned and decide what should be kept, merged, or promoted into a more durable memory layer.

### `simplify`

- purpose: review changed code for reuse, quality, and efficiency, then fix issues found
- user-invocable: yes
- practical meaning:
  this is a built-in self-review and cleanup workflow. It asks the model to critique and tighten its own code changes after implementation.

### `batch`

- purpose: plan a large-scale change and execute it in parallel across many isolated worktree agents
- `whenToUse`: sweeping, decomposable multi-file migrations or refactors
- disables automatic model invocation
- user-invocable: yes
- strong behavioral contract:
  research first, enter plan mode, split work into 5-30 units, determine an end-to-end verification recipe, then launch isolated background worktree agents that each open a PR
- practical meaning:
  this is the product’s built-in bulk-change orchestrator. It is not just “do many edits”; it is “research, decompose, distribute, verify, and track a large migration.”

### `stuck`

- purpose: investigate frozen or slow Claude Code sessions and post a diagnostic report
- marked ant-only in description
- user-invocable: yes
- operational contract:
  inspect other Claude Code processes, detect high CPU / hung / zombie / stopped sessions, inspect child processes and debug logs, and post a diagnostic report if a stuck session is found
- practical meaning:
  this is an operational support skill for diagnosing unhealthy Claude Code processes on the local machine.

## 3. Feature-Gated Bundled Skills

### `dream`

Availability: only when dream/background-memory features are enabled.

Interpretation:

- this is tied to the “dream” or background-memory product surface
- available evidence indicates it belongs to the background-memory / deferred-work family

### `hunter`

Availability: only when review-artifact features are enabled.

Interpretation:

- review-artifact / hunter workflow is not always present
- available evidence indicates it belongs to the deep-review / artifact-review family

### `loop`

Availability: only when recurring-trigger features are enabled.

- purpose: run a prompt or slash command on a recurring interval
- `whenToUse`: recurring polling or repeated tasks, not one-offs
- user-invocable: yes
- final visibility also depends on `isKairosCronEnabled()`
- practical meaning:
  `loop` is the local recurring-work scheduler for prompts and commands.

### `schedule`

Availability: only when remote recurring-agent features are enabled.

- purpose: create, update, list, or run scheduled remote agents on a cron schedule
- `whenToUse`: recurring remote-agent automation
- allowed tools: `REMOTE_TRIGGER_TOOL_NAME`, `ASK_USER_QUESTION_TOOL_NAME`
- also requires:
  - feature value `tengu_surreal_dali`
  - policy allowance for remote sessions
- practical meaning:
  this is the remote counterpart to `loop`: instead of repeating a local prompt, it manages recurring remote agents.

### `claude-api`

Availability: only when app-building / Claude API features are enabled.

- purpose: build apps with the Claude API or Anthropic SDK
- trigger guidance is embedded in the description
- allowed tools: `Read`, `Grep`, `Glob`, `WebFetch`
- user-invocable: yes
- packaging detail:
  this skill bundles a large inline documentation corpus for Python, TypeScript, Java, Go, Ruby, C#, PHP, curl, prompt caching, tool use, batches, files API, streaming, and agent SDK patterns
- practical meaning:
  this is a built-in developer advocate skill. It specializes the model for projects that are explicitly using the Claude API or Anthropic SDKs.

### `claude-in-chrome`

Availability: only when Chrome integration is present and auto-enabled.

- purpose: automate Chrome to interact with pages, forms, screenshots, console logs, and navigation
- should be invoked before direct `mcp__claude-in-chrome__*` use
- allowed tools: Chrome MCP tool set
- user-invocable: yes
- visibility depends on `shouldAutoEnableClaudeInChrome()`
- activation behavior:
  the prompt explicitly tells the model to begin with the browser tab context MCP call before using lower-level browser tools
- practical meaning:
  this is the browser-automation entry skill. It converts raw browser MCP capability into a guided workflow.

### `run-skill-generator`

Availability: only when internal or experimental skill-generation features are enabled.

Interpretation:

- there is an internal or experimental skill-generation workflow not always exposed
- available evidence indicates it is related to automated or semi-automated skill generation

## 4. Product Signals From The Skill Set

The bundled skills reveal several product bets:

- settings and harness customization are important enough to ship dedicated guidance (`update-config`)
- repeatable workflow capture is a first-class UX (`skillify`)
- background / scheduled / remote automation is a real feature family (`loop`, `schedule`, `dream`)
- browser automation is part of the built-in vision (`claude-in-chrome`)
- large-scale parallel change execution is explicitly encouraged (`batch`)
- memory curation is becoming productized (`remember`)

## 5. Bundled Skill Capability Matrix

The bundled skills divide into a few clear families:

- configuration and harness help:
  `update-config`, `keybindings-help`, `debug`
- verification and improvement:
  `verify`, `simplify`
- workflow capture and memory:
  `skillify`, `remember`
- orchestration and automation:
  `batch`, `loop`, `schedule`, `dream`
- browser and API building:
  `claude-in-chrome`, `claude-api`
- diagnostics and support:
  `stuck`
- utility or testing:
  `lorem-ipsum`, `hunter`, `run-skill-generator`

This matters because bundled skills are not a random grab bag. They encode the product team’s opinionated workflows.

## 6. Registration Gates At A Glance

Always registered:

- `update-config`
- `keybindings-help`
- `verify`
- `debug`
- `lorem-ipsum`
- `skillify`
- `remember`
- `simplify`
- `batch`
- `stuck`

Conditionally registered:

- `dream` when `KAIROS` or `KAIROS_DREAM`
- `hunter` when `REVIEW_ARTIFACT`
- `loop` when `AGENT_TRIGGERS`
- `schedule` when `AGENT_TRIGGERS_REMOTE`
- `claude-api` when `BUILDING_CLAUDE_APPS`
- `claude-in-chrome` when Chrome integration auto-enables
- `run-skill-generator` when `RUN_SKILL_GENERATOR`

This gate inventory is useful because it distinguishes shipped knowledge that is always present from shipped knowledge that is rollout- or feature-dependent.

## 7. Relationship To Other Extension Surfaces

Bundled skills are only one extension layer.

The repo also supports:

- built-in plugins
- marketplace plugins
- user skills
- dynamic skills

But the bundled skills catalog is still valuable because it shows the opinionated workflows the binary ships by default.
