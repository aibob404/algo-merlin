# ADR-007: Memory-Augmented Agent Reasoning

**Date:** 2025-03
**Status:** Accepted

## Context

Research on AI trading systems (TradingAgents, LLM_trader, FinGPT papers) shows that agents with access to their own trade history make significantly better decisions. A stateless agent that sees only current market data has no way to learn from its own mistakes within a session.

## Decision

Include the **last 5 completed decisions + outcomes** in every agent decision request as part of the context passed to the LLM.

```json
{
  "botId": "uuid",
  "symbol": "BTC-USDT",
  "timeframe": "1h",
  "recentDecisions": [
    {
      "timestamp": "2025-03-23T09:00:00Z",
      "action": "BUY",
      "confidence": 0.82,
      "reasoning": "RSI oversold, MACD bullish cross",
      "outcome": "TP_HIT",
      "pnl_pct": 2.8
    },
    {
      "timestamp": "2025-03-23T08:00:00Z",
      "action": "HOLD",
      "confidence": 0.65,
      "reasoning": "Unclear trend, waiting for confirmation",
      "outcome": null
    }
  ]
}
```

Platform fetches this from `agent_decisions JOIN trades` and passes it with each `/agent/decide` call.

The system prompt instructs the agent to consider its recent performance:
*"Here are your last 5 decisions and their outcomes. Use this to avoid repeating mistakes and to calibrate your confidence."*

## Consequences

**Positive:**
- Agent learns from its own session history (momentum/mean-reversion awareness)
- Avoids entering same losing trade twice in a row
- Higher confidence when recent similar setups were profitable
- Validated by research: memory-augmented agents outperform stateless agents

**Negative:**
- Adds ~500-1000 tokens per request (small cost increase)
- Past outcomes may bias agent in fast-changing market regimes
- Need to be careful not to overload context with too much history

## Future: Phase 2 Options

- Bull/Bear researcher debate before final decision (TradingAgents architecture)
- Model council: 2-3 LLMs vote, take consensus
- Vision tool: screenshot chart, pass to vision-capable model
- RL fine-tuning on accumulated trade history
