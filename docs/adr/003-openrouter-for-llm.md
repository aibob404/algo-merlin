# ADR-003: OpenRouter as LLM Gateway

**Date:** 2025-03
**Status:** Accepted

## Context

We want to use AI models for trading decisions but don't want to be locked into a single provider.
Different bots may benefit from different models (speed vs quality trade-off).

## Decision

Use **OpenRouter** as a unified LLM API gateway. All LLM calls go through OpenRouter,
which provides access to Claude, GPT, Gemini, Mistral and others via a single OpenAI-compatible API.

## Consequences

**Positive:**
- Single API key, multiple models
- Easy to switch model per bot without code changes
- OpenAI-compatible API — standard tool use format
- Can compare model performance across bots

**Negative:**
- Extra hop (OpenRouter → provider) adds slight latency
- Dependent on OpenRouter uptime
- Pricing varies by model

## Mitigation

- Model is configurable per bot — can fall back to cheaper model if needed
- OpenRouter has high availability SLA
- Future: add direct provider fallback if OpenRouter is down
