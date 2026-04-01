# Prompt Engineering & Behavioral Contracts

Complete system prompt collection and model behavioral specifications. This section documents how Anthropic instructs Claude at the prompt level, including safety rules, capability boundaries, and dynamic prompt composition strategies. The prompt bible represents the most comprehensive extraction of Claude Code's LLM behavioral contracts.

## Document Inventory

| Document | Description | Lines |
|----------|-------------|-------|
| **prompt-bible.md** | The complete system prompt concatenation (2.1 MB uncompressed). Contains every prompt section, conditional injection logic, behavioral rule, safety constraint, and dynamic context loading. Extracted verbatim from the running codebase. | 32,294 |
| **model-behavioral-contract.md** | Model behavior specifications and constraint definitions. Formalizes what Claude is instructed to do, capability restrictions, safety boundaries, and evaluation criteria. | 193 |

## System Prompt Organization

![System Prompt Ordering](../infographics/system-prompt-ordering.svg)

## Prompt Architecture Overview

The system prompt is composed of multiple layers, assembled dynamically:

### Base Prompt Layer
- Core Claude identity and capabilities
- Role definition as a code analysis and autonomous agent
- Fundamental behavioral constraints and safety rules

### Conditional Injection Layer
- Feature flag-driven prompt sections
- User context conditional injections
- Session-specific behavioral modifications
- Permission model enforcement

### Dynamic Context Layer
- Recent chat history context windows
- Previous analysis findings and memory
- Tool output integration and constraint mapping
- Token budget awareness and planning

### Safety & Constraint Layer
- Immutable security rule enforcement
- Prompt injection defense mechanisms
- Social engineering resistance instructions
- Content isolation policies

### Capability Specification Layer
- Tool availability declarations
- Supported model families and versions
- Feature gate status (runtime gates)
- Behavioral instruction for each capability

## Key Findings

### Scale & Composition
- **32,000+ lines of prompt text** across all layers
- **20 novel prompt engineering techniques** identified in the implementation
- **Multi-turn context accumulation** with dynamic pruning
- **Conditional prompt assembly**: Different prompts for different feature gate states

### Safety Architecture
- **Immutable rule layering**: Safety rules that cannot be overridden by subsequent instructions
- **Injection defense mechanisms**: Explicit handling of instruction embedding in user content
- **Social engineering resistance**: Structured responses to authority claims and emotional manipulation
- **Content isolation policies**: Untrusted data marked and separated from system instructions

### Behavioral Contracts
- **Capability restriction zones**: Clear boundaries on what Claude can and cannot do
- **Tool availability gates**: Conditional tool exposure based on context and permissions
- **Output constraint mapping**: Structured responses with format requirements
- **Fallback behavior specification**: What Claude does when constraints cannot be satisfied

### Dynamic Prompt Composition
- **Feature flag conditioning**: Entire prompt sections included/excluded based on build flags
- **Runtime gate integration**: GrowthBook gates controlling prompt behavior at runtime
- **User context personalization**: Prompt adaptation based on authentication level
- **Token budget planning**: Prompt size adjusted dynamically based on context window availability

### Prompt Engineering Techniques
1. **Explicit instruction prioritization**: Clear ordering of rules prevents lower-priority rules from overriding safety constraints
2. **Nested permission scopes**: Capability boundaries nested within role definitions
3. **Adversarial example inclusion**: Explicit instruction on how to respond to known attack patterns
4. **Constraint reminder loops**: Key constraints repeated at decision points
5. **Fallback instruction chaining**: Multi-level fallbacks when primary constraints cannot be satisfied
6. **Metadata annotation**: Inline comments explaining purpose of each prompt section
7. **State machine specification**: Explicit state transitions for multi-step behaviors
8. **Audit trail requirement**: Logging when constraints are applied or bypassed
9. **Exception specification**: Explicit enumeration of rare cases when constraints don't apply
10. **Graduated restriction levels**: Multiple severity levels of constraint enforcement

### Model-Specific Adaptations
- **Anthropic Claude family instructions**: Specific guidance for Claude Opus, Sonnet, Haiku variants
- **Provider-specific constraints**: Different safety rules for different deployment contexts
- **Version compatibility notes**: Behavior changes across Claude model versions
- **Fallback model selection**: Specification for when primary model is unavailable

### Behavioral Specification Format
Each behavioral contract includes:
- **Pre-condition**: When the behavior applies
- **Action**: Specific Claude behavior required
- **Constraint**: What Claude cannot do
- **Output format**: Required response structure
- **Post-condition**: Guaranteed outcome

This systematic specification ensures consistent Claude behavior across all Claude Code sessions and contexts.

