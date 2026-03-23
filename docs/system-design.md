# System Design — Algo-Merlin

## 1. Overview

Algo-Merlin is a multi-service, AI-driven crypto trading system. It runs autonomously on a BingX demo account, using an agentic LLM to make trading decisions based on technical indicators and market data.

---

## 2. Goals

| Goal | Description |
|------|-------------|
| **Autonomous trading** | System runs 24/7, no human intervention required |
| **AI-driven decisions** | LLM agent reasons over market data, not hardcoded rules |
| **Configurable** | Each bot has its own symbol, timeframe, AI model, risk settings |
| **Safe** | Risk management enforced at platform level, not delegated to AI |
| **Observable** | Every decision logged with reasoning, all trades tracked |
| **Extensible** | New tools, strategies, and LLM models can be added without refactoring |

---

## 3. Services

### 3.1 Platform (Kotlin/Ktor)

The central orchestrator. Responsible for:

- **Trading Engine** — coroutine-based loop, one per active bot
- **BingX Client** — authenticated REST API calls (market data + orders)
- **Risk Manager** — validates AI decisions, calculates position size
- **Order Executor** — places, monitors, and closes orders on BingX
- **REST API** — CRUD for bots and trades, consumed by UI
- **Telegram Notifier** — sends trade notifications
- **AI Service Client** — triggers the AI agent, passes tool execution back

**Key design decision:** Platform owns risk rules. The AI agent can suggest a position size, but the platform enforces max 1% risk per trade.

### 3.2 AI Service (Python/FastAPI)

The intelligence layer. Responsible for:

- **Agent Loop** — runs tool calls until LLM is ready to decide
- **Tool Executor** — receives tool calls from LLM, calls Platform API, returns results
- **Indicator Calculator** — calculates RSI, MACD, EMA, Bollinger Bands via pandas-ta
- **LLM Client** — calls OpenRouter with tool definitions and conversation history
- **Prompt Manager** — system prompts per strategy type

**Key design decision:** The AI service is stateless per request. State (open positions, balance) comes from Platform via tools at decision time.

### 3.3 UI (React/TypeScript)

Simple MVP dashboard:

- **Bot Management** — create, edit, start/stop bots
- **Trade History** — table of all trades with AI reasoning visible
- **Performance** — basic PnL summary per bot

### 3.4 Database (PostgreSQL)

Shared storage owned by Platform:

- `bots` — bot configurations
- `trades` — all trades with full metadata
- `candle_cache` — recent candles to reduce API calls
- `agent_logs` — full agent reasoning logs per decision

---

## 4. Data Flow

### 4.1 Trading Cycle (per bot, per candle close)

```
1. Trading Engine detects new candle closed
2. Fetch latest candles from BingX → cache in DB
3. Call AI Service: POST /agent/decide {bot_config, context}
4. AI Agent starts tool loop:
   a. LLM receives system prompt + available tools
   b. LLM calls tools (get_indicators, get_balance, get_open_positions...)
   c. Platform executes tools, returns results to AI Service
   d. LLM reasons over results
   e. LLM returns final decision: {action, reasoning, stop_loss_pct, take_profit_pct}
5. AI Service returns decision to Platform
6. Risk Manager validates:
   - Calculate position size: balance * risk_pct / stop_loss_pct
   - Reject if position would exceed max risk
7. Order Executor places order on BingX
8. Trade saved to DB
9. Telegram notification sent
```

### 4.2 Position Monitoring

```
Every 30 seconds (per open position):
1. Platform fetches current price from BingX
2. Check if stop-loss or take-profit hit
3. If hit: close position, update trade in DB, send Telegram
4. Optionally: call AI agent to re-evaluate ("should I close early?")
```

---

## 5. Agent Architecture

See [agents.md](agents.md) for full agent design.

### Tool Call Flow

```
Platform → AI Service
  POST /agent/decide
  {
    bot_id, symbol, timeframe, ai_model,
    available_tools: [...]
  }

AI Service → LLM (OpenRouter)
  system_prompt + tool_definitions

LLM → AI Service
  tool_call: { name: "get_indicators", args: {...} }

AI Service → Platform
  POST /tools/execute
  { tool: "get_indicators", args: {...} }

Platform → AI Service
  { rsi: 65.3, macd: {...}, ... }

AI Service → LLM
  tool_result + continue

LLM → AI Service
  final_decision: { action: "BUY", reasoning: "...", stop_loss_pct: 1.5, take_profit_pct: 3.0 }

AI Service → Platform
  decision response
```

---

## 6. Risk Management

Risk is enforced by the Platform, regardless of what AI suggests.

| Rule | Value |
|------|-------|
| Max risk per trade | 1% of account balance |
| Demo account balance | ~$1,000 USD |
| Max loss per trade | ~$10 |
| Position size formula | `balance × risk_pct / stop_loss_pct` |
| Concurrent positions per bot | 1 |
| Min stop-loss | 0.3% |
| Max stop-loss | 5% |

**Example:**
- Balance: $1,000
- Risk: 1% → $10 max loss
- AI suggests stop-loss: 2%
- Position size: $10 / 2% = $500

---

## 7. Configuration Model

Each bot is independently configurable:

```json
{
  "id": "uuid",
  "name": "BTC Scalper",
  "symbol": "BTC-USDT",
  "timeframe": "15m",
  "ai_model": "anthropic/claude-3-5-haiku",
  "strategy_hint": "momentum",
  "max_risk_pct": 1.0,
  "is_active": true
}
```

`strategy_hint` influences the system prompt sent to the LLM — e.g., "momentum", "mean_reversion", "breakout".

---

## 8. BingX Demo API

- **Base URL:** `https://open-api-vst.bingx.com`
- **Auth:** HMAC-SHA256 signature on query string + timestamp
- **Key endpoints:**
  - `GET /openApi/swap/v2/quote/klines` — candles
  - `GET /openApi/swap/v2/user/balance` — account balance
  - `POST /openApi/swap/v2/trade/order` — place order
  - `GET /openApi/swap/v2/trade/openOrders` — open orders
  - `DELETE /openApi/swap/v2/trade/order` — cancel order

---

## 9. Deployment

```
Server (Linux/VPS)
└── Docker Compose
    ├── postgres        (port 5432, internal only)
    ├── platform        (port 8080)
    ├── ai-service      (port 8001, internal only)
    └── ui              (port 80)
```

All services run in Docker. Only `platform` (API) and `ui` are exposed. AI service is internal.

---

## 10. Security

| Concern | Solution |
|---------|----------|
| REST API unauthorized access | JWT authentication on all `/api/*` endpoints |
| Tool execution unauthenticated | Shared secret token (`TOOL_EXECUTION_TOKEN`) on `/api/tools/execute` |
| Prompt injection via strategy_hint | Whitelist enforced at Platform + DB CHECK constraint |
| Secrets in env vars | Volume-mounted K8s Secrets (files, not env vars) |
| Trading param out-of-range | DB CHECK constraints + Platform validation layer |

See [ADR-006](adr/006-jwt-authentication.md) for authentication decisions.

---

## 11. Reliability Highlights

| Pattern | Where |
|---------|-------|
| Write-ahead PENDING row | Before every BingX order call |
| Startup position reconciliation | Platform boot sequence |
| Circuit breakers | AI Service, BingX, OpenRouter |
| Graceful shutdown | SIGTERM handler — waits for in-flight decisions |
| Idempotent orders | `clientOrderId = trade.id` on BingX |
| Duplicate candle guard | `lastAnalyzedClose` per bot in trading loop |

See [docs/reliability.md](reliability.md) for full failure mode analysis.

---

## 12. Phased Roadmap

### Phase 1 — MVP
- [ ] Single bot, single pair trading
- [ ] Agent with 5 core tools
- [ ] Basic web UI (bot config + trade history)
- [ ] Telegram notifications
- [ ] BingX demo integration

### Phase 2 — Multi-bot
- [ ] Multiple concurrent bots
- [ ] News sentiment tool
- [ ] Fear & Greed index tool
- [ ] Performance charts in UI
- [ ] Backtesting endpoint

### Phase 3 — Advanced
- [ ] Multi-agent coordination (one supervisor agent)
- [ ] Portfolio-level risk management
- [ ] Live account support
- [ ] Strategy marketplace
