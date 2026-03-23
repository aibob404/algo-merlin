# Agent Design — Algo-Merlin

## 1. Agent Philosophy

The AI agent in Algo-Merlin is **not a function** — it's an autonomous reasoner.
Instead of receiving pre-packaged indicators and returning a decision, the agent:

1. Receives a task: *"Should I trade BTC-USDT on 1h timeframe right now?"*
2. Decides what data it needs
3. Calls tools to gather that data
4. Reasons across all gathered context
5. Returns a structured trading decision

This approach means the agent can adapt its analysis depth per situation — sometimes one tool call is enough, sometimes it checks 4-5 different signals.

---

## 2. Research Validation

Industry research confirms the agentic approach is the right direction:

- **[TradingAgents (arXiv 2412.20138)](https://arxiv.org/abs/2412.20138)** — multi-agent LLM framework with 7 specialized agents (Technical Analyst, Sentiment Analyst, Risk Manager, Trader, etc.) shows improvements in cumulative return, Sharpe ratio, and max drawdown vs single-agent and rule-based baselines.
- **[LLM_trader](https://github.com/qrak/LLM_trader)** — uses OpenRouter (same as us) with a "Council of Models" and memory-augmented reasoning.
- **2025 competition results** — specialized agents with custom logic beat GPT-5, Gemini Pro, and DeepSeek in live trading competitions ([CoinDesk](https://www.coindesk.com/business/2025/12/13/crypto-s-machine-learning-iphone-moment-comes-closer-as-ai-agents-trade-the-market)).
- **Hybrid LLM+RL** outperforms pure LLM in 4 of 6 tested assets — considered for Phase 3.

---

## 3. Agent Loop

```
SYSTEM PROMPT (strategy context, risk rules, output format)
      +
USER MESSAGE ("Analyze BTC-USDT 1h and decide")
      ↓
┌─────────────────────────────────────┐
│           LLM Reasoning             │
│  "I need indicators first"          │
│  → tool_call: get_indicators(...)   │
└─────────────────┬───────────────────┘
                  │ tool result
┌─────────────────▼───────────────────┐
│           LLM Reasoning             │
│  "RSI is 72, overbought.            │
│   Let me check open positions"      │
│  → tool_call: get_open_positions()  │
└─────────────────┬───────────────────┘
                  │ tool result
┌─────────────────▼───────────────────┐
│           LLM Reasoning             │
│  "No open position. RSI high,       │
│   MACD bearish cross forming.       │
│   Decision: HOLD"                   │
│  → final_decision (structured JSON) │
└─────────────────────────────────────┘
```

Max tool call iterations: **10** (safety limit to prevent infinite loops).
- At iteration 9: inject *"Make your final decision now."* into conversation
- Hard timeout: 30s on the entire agent loop (separate from per-tool 10s timeout)
- If max iterations or timeout reached without decision → return **HOLD**

---

## 4. Memory-Augmented Context (MVP)

Every agent decision request includes the **last 5 decisions + outcomes** for this bot. This allows the agent to learn from its own session history and avoid repeating mistakes.

Passed as part of the `/agent/decide` request:
```json
{
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

System prompt addition:
> *"Here are your last 5 decisions and their outcomes. Use this to calibrate your confidence and avoid repeating mistakes in similar market conditions."*

Source: Validated by [LLM_trader](https://github.com/qrak/LLM_trader) and [TradingAgents](https://arxiv.org/abs/2412.20138) research.

---

## 5. Tools — MVP

### `get_candles`
Fetch raw OHLCV candles from BingX.
```json
{
  "name": "get_candles",
  "description": "Get recent OHLCV candlestick data for a trading pair",
  "parameters": {
    "symbol": "string (e.g. BTC-USDT)",
    "timeframe": "string (1m, 5m, 15m, 1h, 4h, 1d)",
    "limit": "integer (default 50, max 200)"
  }
}
```

### `get_indicators`
Calculate technical indicators from recent candles.
```json
{
  "name": "get_indicators",
  "description": "Get technical indicators: RSI, MACD, EMA20, EMA50, Bollinger Bands, ATR",
  "parameters": {
    "symbol": "string",
    "timeframe": "string"
  },
  "returns": {
    "rsi": "float (0-100)",
    "macd": { "macd": "float", "signal": "float", "histogram": "float" },
    "ema20": "float",
    "ema50": "float",
    "bb": { "upper": "float", "middle": "float", "lower": "float" },
    "atr": "float",
    "current_price": "float"
  }
}
```

### `get_balance`
Get current demo account balance.
```json
{
  "name": "get_balance",
  "description": "Get current account balance in USDT",
  "parameters": {},
  "returns": {
    "total": "float",
    "available": "float",
    "in_use": "float"
  }
}
```

### `get_open_positions`
Get all currently open trades.
```json
{
  "name": "get_open_positions",
  "description": "Get list of currently open trading positions",
  "parameters": {
    "symbol": "string (optional, filter by symbol)"
  },
  "returns": [
    {
      "id": "string",
      "symbol": "string",
      "side": "BUY|SELL",
      "entry_price": "float",
      "current_price": "float",
      "quantity": "float",
      "unrealized_pnl": "float",
      "stop_loss": "float",
      "take_profit": "float"
    }
  ]
}
```

### `place_order`
Execute a trade. Only called when agent decides BUY or SELL.
```json
{
  "name": "place_order",
  "description": "Place a market order with stop-loss and take-profit",
  "parameters": {
    "symbol": "string",
    "side": "BUY|SELL",
    "stop_loss_pct": "float (0.3 to 5.0)",
    "take_profit_pct": "float (0.5 to 15.0)",
    "reasoning": "string (explain why you are placing this order)"
  }
}
```
Note: Position size is calculated by the Platform's Risk Manager, not the agent.

### `close_position`
Close an existing open position.
```json
{
  "name": "close_position",
  "description": "Close an open position at market price",
  "parameters": {
    "position_id": "string",
    "reasoning": "string"
  }
}
```

---

## 6. Tools — Phase 2

| Tool | Description |
|------|-------------|
| `get_news_sentiment` | Analyze recent crypto news for sentiment score |
| `get_fear_greed_index` | Current crypto fear & greed index (0-100) |
| `get_funding_rate` | Perpetual futures funding rate (indicates market bias) |
| `get_volume_profile` | Volume distribution across price levels |

---

## 7. System Prompt

The system prompt sets the agent's personality, constraints, and output expectations.
It is configurable per bot via `strategy_hint`.

### Base System Prompt
```
You are Algo-Merlin, an expert crypto trading agent for BingX perpetual futures.
You operate on a demo account with a $1,000 balance.

Your job is to analyze market conditions and decide whether to BUY, SELL, or HOLD
for the given trading pair and timeframe.

RULES:
- Use your tools to gather the data you need before deciding
- Never place a trade without checking open positions first
- Never place more than one trade per symbol at a time
- Stop-loss must be between 0.3% and 5%
- Take-profit must be between 0.5% and 15%
- Risk/reward ratio should be at least 1:1.5
- Always explain your reasoning clearly

STRATEGY HINT: {strategy_hint}
SYMBOL: {symbol}
TIMEFRAME: {timeframe}
```

### Strategy Hints

| Hint | Behavior |
|------|----------|
| `momentum` | Follow trend, buy breakouts, use wider TP |
| `mean_reversion` | Buy dips, sell spikes, tighter SL/TP |
| `breakout` | Wait for key level breaks, high confidence only |
| `conservative` | HOLD bias, only trade very clear signals |
| `aggressive` | More trades, accept lower confidence signals |

---

## 8. Decision Output Format

The agent must end its reasoning with a structured JSON decision:

```json
{
  "action": "BUY | SELL | HOLD",
  "confidence": 0.85,
  "reasoning": "RSI at 28 indicates oversold conditions. MACD histogram turning positive suggests momentum shift. EMA20 > EMA50 confirms uptrend. Entering long with tight stop below recent low.",
  "stop_loss_pct": 1.5,
  "take_profit_pct": 3.0
}
```

---

## 9. LLM Models (via OpenRouter)

| Model | Use case |
|-------|----------|
| `anthropic/claude-3-5-haiku` | Default — fast, cheap, good reasoning |
| `anthropic/claude-3-5-sonnet` | Higher quality decisions, more expensive |
| `openai/gpt-4o-mini` | Alternative, fast |
| `openai/gpt-4o` | High quality alternative |
| `google/gemini-flash-1.5` | Fast, low cost |

Model is configurable per bot.

---

## 10. Safety & Guardrails

| Guardrail | Where enforced |
|-----------|---------------|
| Max 1% risk per trade | Platform (Risk Manager) |
| Max 1 position per symbol per bot | Platform (before order execution) |
| Stop-loss range 0.3-5% | Platform + DB CHECK constraint |
| Take-profit range 0.5-15% | Platform + DB CHECK constraint |
| `strategy_hint` whitelist | Platform + DB CHECK constraint (prevents prompt injection) |
| Max 10 tool calls per decision | AI Service (agent loop limit) |
| Agent loop hard timeout | AI Service (30s total) |
| AI Service down → HOLD | Platform (circuit breaker fallback) |
| Order rejected if balance insufficient | Platform |
| All decisions logged with full reasoning | DB (agent_decisions — immutable) |
| Tool execution requires auth token | Platform (shared secret on /tools/execute) |

---

## 11. Phase 2+ Roadmap (Research-Validated)

Based on [TradingAgents paper](https://arxiv.org/abs/2412.20138) and industry research:

| Phase | Enhancement | Expected Impact |
|-------|------------|----------------|
| **Phase 2** | Bull/Bear researcher debate before trader agent decides | Better decision quality, fewer false signals |
| **Phase 2** | News sentiment tool (FinGPT-style) | Catches macro events indicators miss |
| **Phase 2** | Model council (2-3 LLMs vote, take consensus) | Higher confidence decisions |
| **Phase 3** | Vision tool — screenshot chart → vision model | Multi-modal signal |
| **Phase 3** | RL fine-tuning on own trade history | Continuous improvement |
