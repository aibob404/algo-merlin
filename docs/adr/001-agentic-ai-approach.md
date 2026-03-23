# ADR-001: Agentic AI Approach for Trading Decisions

**Date:** 2025-03
**Status:** Accepted

## Context

We need an AI system to make trading decisions. The simplest approach would be to pre-calculate a fixed set of indicators and feed them to an LLM, which returns a decision. However, this limits what data the AI can access and requires us to decide upfront what's relevant.

## Decision

Use an **agentic approach** where the LLM has access to tools it can call autonomously to gather its own context before deciding.

## Consequences

**Positive:**
- AI can decide what data it needs — more adaptive
- New capabilities added by adding tools, not changing core logic
- Decisions are more explainable (tool call trace is logged)
- Works with any LLM that supports tool use (via OpenRouter)

**Negative:**
- More complex than a simple function call
- Slower (multiple round-trips to LLM)
- More expensive (more tokens per decision)
- Harder to debug if agent behaves unexpectedly

## Mitigation

- Max tool call limit (10) prevents infinite loops
- All tool calls logged for debugging
- Platform enforces risk rules regardless of AI output
- Start with fast/cheap models (Claude Haiku, GPT-4o-mini)
