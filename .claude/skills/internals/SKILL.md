---
description: "Deep system analysis skill. Audits the user's codebase against production-grade patterns extracted from Anthropic's own Claude Code (512K-line TypeScript agent). Covers: context management, permission systems, security hardening, agent orchestration, tool execution pipelines, API client design, prompt engineering, state management, plugin architectures, rendering engines, build systems, telemetry, memory systems, error handling, resilience patterns. TRIGGER when: reviewing architecture, auditing security, designing agent systems, building CLI/TUI tools, implementing streaming, designing permission models, building plugin systems, optimizing context windows, designing multi-agent coordination, writing system prompts, building API clients, implementing retry/backoff, designing build pipelines, or whenever the user wants to understand how a production AI system handles something and whether their approach measures up."
---

# Deep System Audit — Anthropic's Production Patterns vs Your Code

You have access to a complete reverse-engineering of how Anthropic built Claude Code — a 512,000-line production AI agent. 82 analysis documents. 15 architectural diagrams. Every subsystem dissected down to thresholds, constants, and failure modes.

This is NOT a one-pass skill. You will loop multiple times. You will build scratchpads. You will fetch many documents. You will re-read user code after each new thing you learn. The goal is depth that a human architect would need days to produce.

## The Loop

This skill runs as a multi-pass ReAct loop. Every pass deepens your understanding. You do NOT do this in one shot.

```
PASS 1: Recon         → Read user code + fetch tree + create scratchpad
PASS 2: First fetch   → Pull primary Anthropic docs, extract patterns to scratchpad
PASS 3: User re-read  → Re-read user code with new knowledge, find gaps you missed
PASS 4: Deep fetch    → Pull secondary/chained docs for gaps found in pass 3
PASS 5: Synthesis     → Cross-reference everything in scratchpad, write findings
PASS N: Keep going    → If scratchpad reveals more threads to pull, pull them
```

You stop when your scratchpad has no open questions left.

---

## PASS 1: Recon

### 1A. Fetch the Knowledge Tree

Always start here. The tree is the live index. Never rely on cached/memorized paths.

```
https://raw.githubusercontent.com/thtskaran/claude-code-analysis/master/TREE.md
```

WebFetch this. It has every document's path, topic tags, and description. Paths change, docs get added — the tree is always current.

### 1B. Read the User's System

Read the actual source files. Not a summary. Not just the file the user pointed at. Read around it:

- The file itself
- Its imports — what does it depend on?
- Its callers — what calls it?
- Adjacent files in the same directory
- Config files, env files, package.json, build config
- Test files if they exist
- Any error handling, retry logic, validation

You are building a mental model of their system. You need enough context that you could explain their architecture to a new engineer.

### 1C. Create the Scratchpad

Create a markdown file at `.claude/internals-scratchpad.md` in the user's project. This is your working memory across passes. Structure it like this:

```markdown
# Internals Audit Scratchpad

## Subsystems Identified
- [ ] (list each subsystem you see in the user's code)

## Anthropic Docs to Fetch
- [ ] (paths from TREE.md that match, with reason)

## Patterns Extracted (filled in pass 2+)
<!-- For each Anthropic doc you read, extract the key patterns here -->

## User Code vs Anthropic Patterns (filled in pass 3+)
<!-- Line-by-line comparison notes -->

## Gaps Found
<!-- Things the user's system lacks that Anthropic's handles -->

## Open Questions
<!-- Things you need to fetch more docs to answer -->

## Threads to Pull
<!-- Chains: "doc X mentioned Y, need to fetch doc Z to understand Y" -->
```

Write this file now. You will update it on every pass. This is how you think across passes without losing context.

---

## PASS 2: First Fetch

### 2A. Fetch Primary Docs

From the tree, identify the 2-4 most relevant documents. Fetch them ALL — don't stop at one.

```
https://raw.githubusercontent.com/thtskaran/claude-code-analysis/master/{path}
```

For each document you fetch, you MUST read the entire thing, not just the first section. These docs are long on purpose — the critical patterns are often buried deep.

### 2B. Extract to Scratchpad

For every document you read, write into your scratchpad under "Patterns Extracted":

```markdown
### From: {document path}

**Architecture:**
(how they structured it — layers, components, data flow)

**Thresholds & Constants:**
(every hardcoded number — what, value, why)

**Failure Modes & Fallbacks:**
(what breaks, circuit breakers, escalation paths, degradation strategy)

**Edge Cases Handled:**
(scenarios most devs miss that Anthropic explicitly handles)

**Security Boundaries:**
(where they don't trust input, what's sanitized/validated/sandboxed)

**Performance Decisions:**
(what's optimized, what tradeoffs, caching strategy)
```

Don't summarize. Extract specifics. Numbers. Mechanisms. Exact patterns.

---

## PASS 3: Re-read User Code

Now go back and read the user's code AGAIN. You now know what Anthropic does. On this pass you'll see things you missed the first time:

- Missing error paths that Anthropic handles
- Thresholds the user hardcoded differently (or didn't set at all)
- Security surfaces the user left open
- Resilience mechanisms that don't exist in the user's code
- Architectural layers the user collapsed or skipped

Update the scratchpad under "User Code vs Anthropic Patterns" and "Gaps Found".

Also: check "Open Questions". Did reading the Anthropic docs raise new questions about the user's system? Go read more of their code to answer them.

---

## PASS 4: Deep Fetch

### 4A. Follow the Chains

By now your scratchpad should have entries under "Threads to Pull" — references from one doc to another subsystem. Fetch those secondary docs.

Common chains:
- API client → compaction (if they stream, do they manage context?)
- Tool execution → permissions → bash parser (security layers)
- Agent orchestration → IPC → memory sync (coordination primitives)
- Plugin system → settings/policy → security (scope and isolation)
- Prompt engineering → model behavior contract (what the model expects)

### 4B. Cross-System Patterns

Some of Anthropic's most important patterns span multiple subsystems:

- **3-tier fallback pattern** — appears in compaction (micro → full → session), model selection (primary → fallback → degraded), permissions (auto → prompt → deny)
- **Circuit breaker pattern** — appears in compaction (3 failures), permissions (3 consecutive / 20 total denials), API retry (max attempts)
- **Cache-aware ordering** — appears in prompt construction (static before dynamic), API requests (prefix reuse), tool schemas (stable first)
- **Escalation pattern** — appears in token limits (8K→64K), permission denials (auto→manual), error recovery (retry→fallback→abort)

Look for whether the user's system has ANY of these cross-cutting patterns. Write findings to scratchpad.

### 4C. Fetch Even More If Needed

If your scratchpad still has open questions or threads to pull after this pass, keep going. Fetch the relevant docs. There's no limit on how many docs you should read — read as many as it takes to fully cover the user's system.

---

## PASS 5+: Synthesis

### Update the Scratchpad One Final Time

Go through every section. Make sure:
- Every identified subsystem has been compared
- Every gap has a specific Anthropic pattern it's missing
- Every open question is either answered or explicitly marked as out-of-scope
- Threads to pull is empty (you followed them all)

### Deliver the Audit

Now — and only now — present findings to the user. Structure as:

**For each subsystem:**

1. **What you built** — Brief description of the user's implementation
2. **What Anthropic built** — The specific production pattern, with exact thresholds/constants/mechanisms
3. **Where you diverge** — Concrete differences, not hand-wavy. Line numbers. Specific missing mechanisms.
4. **What breaks without this** — The failure mode. Not theoretical — what actually happens under load, at scale, or under attack.
5. **What to change** — Specific. Not "add retry logic." Instead: "Add exponential backoff with jitter. Base delay 1s, max 5 retries, factor 2x, jitter range 0-500ms. Without jitter, 1000 concurrent failures retry simultaneously and DDoS your own API."

**After all subsystems:**

6. **Cross-cutting gaps** — Patterns that span subsystems (circuit breakers, fallback tiers, cache-aware ordering, escalation chains) that the user's system lacks
7. **What you got right** — Where the user's approach matches or exceeds production patterns. Reinforce these.
8. **Priority order** — Rank the gaps by blast radius. What kills you first?

### Clean Up

Leave the scratchpad at `.claude/internals-scratchpad.md`. The user might want to reference it. Add a timestamp and summary at the top:

```markdown
# Internals Audit Scratchpad
> Audit completed: {date}
> Subsystems audited: {list}
> Anthropic docs consulted: {count}
> Gaps found: {count}
```

---

## Rules

1. **Never one-pass.** Minimum 3 passes through the loop. If you're done in one pass you didn't go deep enough.
2. **Always create the scratchpad.** It's your working memory. Without it you'll lose context between passes and your analysis will be shallow.
3. **Always re-read user code after fetching Anthropic docs.** You WILL see things you missed. This is not optional.
4. **Fetch at minimum 3 documents from the knowledge base.** Usually more. One doc is never enough — subsystems interconnect.
5. **Never summarize Anthropic docs.** Extract specific patterns, numbers, mechanisms. The value is in the details.
6. **Follow every chain.** If doc A mentions a pattern covered in doc B, fetch doc B. If the user's code has a gap that touches three subsystems, fetch all three.
7. **The scratchpad must have zero open questions before you deliver findings.** If questions remain, you need more passes.
8. **Prescriptions must be specific.** Thresholds, constants, exact mechanisms. "Add error handling" is worthless. "Add a circuit breaker that trips after 3 consecutive failures, with a 30-second cooldown, and falls back to X" is useful.
