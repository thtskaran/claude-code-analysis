# Community OSINT: What the Internet Found

**Added by:**  blind-spot analysis (April 2, 2026)
**Technique:** Web search + scraping across 15+ sources (VentureBeat, HN, The Register, Fortune, DEV, blogs)

---

## The Incident

On March 31, 2026, version 2.1.88 of `@anthropic-ai/claude-code` shipped with a 59.8 MB JavaScript source map file. Discovery was broadcast by Chaofan Shou (@Fried_rice) on X at 4:23am ET. Within hours, the ~512,000-line TypeScript codebase was mirrored across GitHub and analyzed by thousands of developers.

### Root Cause
Bun's bundler generates source maps by default. Bun issue #28001 (filed March 11, 2026) documented this. A missing `.npmignore` entry or `files` field in `package.json` would have prevented the entire leak.

### Concurrent Supply Chain Attack
On the same day, a malicious version of `axios` (1.14.1 and 0.30.4) containing a Remote Access Trojan was published to npm. Anyone who installed Claude Code between 00:21 and 03:29 UTC may have pulled in the compromised dependency.

### Anthropic's Response
Used `npm deprecate` instead of `npm unpublish`. With 231+ dependent packages and exceeding the 72-hour unpublish window, the source maps remained accessible.

---

## Model Codenames (Community Attribution)

| Codename | Maps To | Source |
|----------|---------|--------|
| Capybara | Claude 4.6 variant (Sonnet family) | Multiple community sources |
| Fennec | Opus 4.6 | Migration script: migrateFennecToOpus.ts |
| Numbat | Unreleased model in testing | Community reports |
| Tengu | Namespace prefix for GrowthBook gates (not a model) | Source analysis |

### Migration Chain (from source):
```
Haiku → Sonnet 1M → Sonnet 4.5 (20250929) → Sonnet 4.6 (via 'sonnet' alias)
Fennec → Opus → Opus 1M
claude-opus-4-1 (internal) → claude-opus-4-6 (current)
```

---

## Net-New Community Findings (Not in Our Prior Analysis)

| Discovery | Detail | Source |
|-----------|--------|--------|
| Anti-Distillation | Decoy tool injection to corrupt competitor training data | Alex Kim |
| 108 Gated Modules | Modules not in public package but present in source | WaveSpeed AI |
| print.ts Monolith | 3,167-line single function, 486 cyclomatic complexity, 12 nesting levels | HN commenters |
| Double LLM Calls per Bash | Each execution triggers prefix detection + filepath extraction | Kir Shatrov |
| Sentiment Regex (not ML) | Regex-based frustration detection — bypassed by non-English input | HN commenters |
| 250K Wasted API Calls/Day | Autocompact failures: 1,279 sessions with 50+ consecutive failures (up to 3,272/session) | Alex Kim |
| Fastest GitHub Repo | Leaked source became fastest growing repository in GitHub history | CyberNews |
| Undercover Mode | Stealth OSS contributions by Anthropic employees | Alex Kim, The Register |
| Clipboard Detection Bug | Nested async promises without proper chaining for wl-copy/xclip/xsel | HN commenters |

---

## Community Analysis Projects

| Project | Author | Focus |
|---------|--------|-------|
| [claude-code-reverse](https://github.com/Yuyz0112/claude-code-reverse) | Yuyz0112 | Visualization tool for LLM API interactions |
| [claude-code-analysis](https://github.com/ComeOnOliver/claude-code-analysis) | ComeOnOliver | Comprehensive architecture analysis |
| [claude-code (Rust)](https://github.com/Kuberwastaken/claude-code) | Kuberwastaken | Rust reimplementation + breakdown |
| [claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts) | Piebald-AI | Extracted system prompts |
| [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) | shareAI-lab | "Bash is all you need" nano agent harness |
| [claw-cli](https://github.com/ringmast4r/claw-cli-claude-code-source-code-v2.1.88) | Ringmast4r | Working toward Bun build of source |

---

## Blog Posts & Articles

- [Kir Shatrov: Reverse engineering Claude Code](https://kirshatrov.com/posts/claude-code-internals) — Double LLM call discovery, cost analysis
- [Alex Kim: Fake tools, frustration regexes, undercover mode](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/) — Anti-distillation, autocompact waste
- [Sathwick: Deep Dive](https://sathwick.xyz/blog/claude-code.html) — Comprehensive architecture walkthrough
- [Vrungta: Architecture (Reverse Engineered)](https://vrungta.substack.com/p/claude-code-architecture-reverse) — TAOR loop, 6-layer memory, co-evolution
- [WaveSpeed AI: Hidden Features](https://wavespeed.ai/blog/posts/claude-code-leaked-source-hidden-features/) — BUDDY, KAIROS, 108 gated modules
- [DEV Community: Full breakdown](https://dev.to/gabrielanhaia/claude-codes-entire-source-code-was-just-leaked-via-npm-source-maps-heres-whats-inside-cjo) — Architecture overview

---

## Media Coverage

- [VentureBeat](https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know)
- [Fortune](https://fortune.com/2026/03/31/anthropic-source-code-claude-code-data-leak-second-security-lapse-days-after-accidentally-revealing-mythos/)
- [The Register](https://www.theregister.com/2026/04/01/claude_code_source_leak_privacy_nightmare/)
- [The Hacker News](https://thehackernews.com/2026/04/claude-code-tleaked-via-npm-packaging.html)
- [CyberNews](https://cybernews.com/tech/claude-code-leak-spawns-fastest-github-repo/)
- [Hacker News Discussion #47584540](https://news.ycombinator.com/item?id=47584540)

---

## Key Architectural Insights from Community

1. **TAOR Loop:** Think-Act-Observe-Repeat with orchestrator being ~50 lines; all intelligence in model + prompts
2. **Primitive Tools Over Integrations:** Read/Write/Execute/Connect rather than 100+ specialized tools
3. **Co-Evolution Architecture:** Harness designed to shrink as models improve; scaffolding deleted with each release
4. **Context Window as Scarce Resource:** ~200K token budget treated as primary constraint; auto-compaction at ~50%
5. **Steerability via Emphasis:** Personality engineering via prompt structure, XML tags, intentional repetition — not fine-tuning
