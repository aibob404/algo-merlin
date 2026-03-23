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

## 2. Agent Loop

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

---

## 3. Tools — MVP

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

## 4. Tools — Phase 2

| Tool | Description |
|------|-------------|
| `get_news_sentiment` | Analyze recent crypto news for sentiment score |
| `get_fear_greed_index` | Current crypto fear & greed index (0-100) |
| `get_funding_rate` | Perpetual futures funding rate (indicates market bias) |
| `get_volume_profile` | Volume distribution across price levels |

---

## 5. System Prompt

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

## 6. Decision Output Format

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

## 7. LLM Models (via OpenRouter)

| Model | Use case |
|-------|----------|
| `anthropic/claude-3-5-haiku` | Default — fast, cheap, good reasoning |
| `anthropic/claude-3-5-sonnet` | Higher quality decisions, more expensive |
| `openai/gpt-4o-mini` | Alternative, fast |
| `openai/gpt-4o` | High quality alternative |
| `google/gemini-flash-1.5` | Fast, low cost |

Model is configurable per bot.

---

## 8. Safety & Guardrails

| Guardrail | Where enforced |
|-----------|---------------|
| Max 1% risk per trade | Platform (Risk Manager) |
| Max 1 position per symbol per bot | Platform (before order execution) |
| Stop-loss range 0.3-5% | Platform (rejects out-of-range values) |
| Max 10 tool calls per decision | AI Service (agent loop limit) |
| Order rejected if balance insufficient | Platform |
| All decisions logged with full reasoning | Platform (DB) |
