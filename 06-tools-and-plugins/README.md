# Command System Deep-Dive Analysis

## Overview

This directory contains a comprehensive analysis of Claude Code v2.1.88's command system architecture.

## Main Document

**File:** `command-system-deep-dive.md` (1,850 lines)

A complete reverse-engineering analysis covering:
- Registry architecture and import organization
- Command type system (prompt, local, local-jsx)
- Dynamic loading pipeline (skills, plugins, MCP, workflows)
- Three execution models with detailed comparisons
- Feature-gated commands and availability filtering
- The 3,200-line /insights command (session analysis)
- Plugin marketplace system (17 files, 7,575 LOC)
- GitHub Actions multi-step wizard (14 components, 2,352 LOC)
- Session management (/resume, /session, /branch, /rewind)
- All built-in commands organized by category
- Security boundaries and attack mitigations
- Performance optimizations (memoization, lazy loading)
- Design decisions and trade-offs

## Quick Navigation

### Core Architecture (Sections 1-5)
- Registry memoization & import organization
- CommandBase + three type union
- Provider/auth gating
- Dynamic loading pipeline
- Command discovery utilities

### Execution Models (Sections 6-15)
- Remote-safe vs bridge-safe commands
- Command filtering (skill listings, MCP commands)
- Cache invalidation strategies
- /insights deep-dive
- Plugin marketplace
- GitHub Actions wizard
- Code review (/review vs /ultrareview)

### Commands & Features (Sections 16-22)
- /mcp add CLI subcommand
- /init codebase onboarding (8 phases)
- Complete built-in command registry
- Command interaction patterns (onDone, context)
- Feature-gated commands
- All three command type examples

### Security & Performance (Sections 23-25)
- Attack vectors and mitigations
- Performance analysis (discovery, lazy loading, insights)
- Extensibility: how to add new commands
- Future design directions

## Key Insights

1. **Layered Memoization:** COMMANDS() memoized globally; loadAllCommands() memoized per cwd
2. **Three Execution Models:** Prompt (model-invocable), Local (Node.js), Local-JSX (TUI)
3. **Dynamic Registration:** Skills, plugins, MCP, workflows inject commands at runtime
4. **Priority System:** Bundled > built-in plugins > user skills > workflows > installed plugins > built-ins
5. **Auth Gating:** Commands hidden based on provider (Claude.ai vs Console API vs Bedrock)
6. **Feature Flags:** Bun bundled features gate conditional imports & command availability
7. **Lazy Loading:** Large commands (insights: 113KB) defer import until invocation
8. **Bridge Safety:** Prompt commands always safe, local-jsx always blocked, local needs allowlist

## Codebase Statistics

- **Total commands:** ~150 built-in + dynamic skills/plugins
- **Internal-only commands:** 14 (Anthropic-only, eliminated from external builds)
- **Feature-gated commands:** 15+ (behind Bun feature flags)
- **Command files:** 189 total across src/commands/
- **Central registry:** /src/commands.ts (755 lines)
- **Type definitions:** /src/types/command.ts (217 lines)
- **Largest command:** insights.ts (3,200 lines, lazy-loaded)

## Design Principles

✓ **Composition over monolithism** — Commands authored by Anthropic, users, plugins, MCP, workflows
✓ **Type safety** — Union types for three execution models
✓ **Performance** — Memoization, lazy loading, parallel I/O
✓ **Security** — Allowlisting, permission boundaries, trust warnings
✓ **Extensibility** — Clear registration patterns for new commands
✓ **UX** — Immediate interactive commands, queued async operations

## Related Analyses

- (Future) Security model & permission framework
- (Future) Plugin architecture & marketplace integration
- (Future) MCP server integration & tool reflection
- (Future) Skill discovery & model invocation system
