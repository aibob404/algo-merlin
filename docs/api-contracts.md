# API Contracts — Algo-Merlin

## 1. Platform REST API (Kotlin/Ktor → UI)

Base URL: `http://platform:8080/api`

---

### Bots

#### `GET /api/bots`
List all bots.

**Response:**
```json
[
  {
    "id": "uuid",
    "name": "BTC Scalper",
    "symbol": "BTC-USDT",
    "timeframe": "15m",
    "aiModel": "anthropic/claude-3-5-haiku",
    "strategyHint": "momentum",
    "maxRiskPct": 1.0,
    "isActive": true,
    "createdAt": "2025-01-01T00:00:00Z"
  }
]
```

#### `POST /api/bots`
Create a new bot.

**Request:**
```json
{
  "name": "BTC Scalper",
  "symbol": "BTC-USDT",
  "timeframe": "15m",
  "aiModel": "anthropic/claude-3-5-haiku",
  "strategyHint": "momentum",
  "maxRiskPct": 1.0
}
```

#### `PUT /api/bots/{id}`
Update bot configuration.

#### `DELETE /api/bots/{id}`
Delete a bot (only if not active).

#### `POST /api/bots/{id}/start`
Start a bot's trading loop.

#### `POST /api/bots/{id}/stop`
Stop a bot's trading loop.

---

### Trades

#### `GET /api/trades`
List all trades with optional filters.

**Query params:** `botId`, `symbol`, `status` (OPEN/CLOSED), `limit`, `offset`

**Response:**
```json
[
  {
    "id": "uuid",
    "botId": "uuid",
    "symbol": "BTC-USDT",
    "side": "BUY",
    "entryPrice": 45000.0,
    "exitPrice": 46350.0,
    "quantity": 0.011,
    "stopLoss": 44325.0,
    "takeProfit": 46350.0,
    "pnl": 14.85,
    "status": "CLOSED",
    "aiReasoning": "RSI oversold, MACD bullish cross...",
    "aiConfidence": 0.82,
    "openedAt": "2025-01-01T10:00:00Z",
    "closedAt": "2025-01-01T14:30:00Z"
  }
]
```

#### `GET /api/trades/{id}`
Get single trade with full agent log.

---

### Dashboard

#### `GET /api/dashboard`
Summary stats.

**Response:**
```json
{
  "totalBots": 3,
  "activeBots": 2,
  "openPositions": 1,
  "totalTrades": 142,
  "winRate": 0.58,
  "totalPnl": 87.5,
  "accountBalance": 1087.5
}
```

---

### Tools (Platform → AI Service callbacks)

#### `POST /api/tools/execute`
Called by AI Service to execute a tool on behalf of the agent.

**Request:**
```json
{
  "botId": "uuid",
  "tool": "get_indicators",
  "args": {
    "symbol": "BTC-USDT",
    "timeframe": "1h"
  }
}
```

**Response:** tool-specific result (see agents.md)

---

## 2. AI Service API (Python/FastAPI)

Base URL: `http://ai-service:8001`

---

#### `POST /agent/decide`
Trigger the agent to make a trading decision.

**Request:**
```json
{
  "botId": "uuid",
  "symbol": "BTC-USDT",
  "timeframe": "1h",
  "aiModel": "anthropic/claude-3-5-haiku",
  "strategyHint": "momentum",
  "platformUrl": "http://platform:8080"
}
```

**Response:**
```json
{
  "action": "BUY",
  "confidence": 0.82,
  "reasoning": "RSI at 28 (oversold), MACD histogram turning positive...",
  "stopLossPct": 1.5,
  "takeProfitPct": 3.0,
  "toolCallsUsed": 3,
  "agentLog": [...]
}
```

#### `GET /health`
Health check.

---

## 3. Data Models

### Bot
```typescript
interface Bot {
  id: string
  name: string
  symbol: string           // "BTC-USDT"
  timeframe: string        // "1m" | "5m" | "15m" | "1h" | "4h" | "1d"
  aiModel: string          // OpenRouter model ID
  strategyHint: string     // "momentum" | "mean_reversion" | "breakout" | "conservative" | "aggressive"
  maxRiskPct: number       // 0.1 to 3.0
  isActive: boolean
  createdAt: string
  updatedAt: string
}
```

### Trade
```typescript
interface Trade {
  id: string
  botId: string
  symbol: string
  side: "BUY" | "SELL"
  entryPrice: number
  exitPrice: number | null
  quantity: number
  stopLoss: number
  takeProfit: number
  pnl: number | null
  status: "OPEN" | "CLOSED" | "CANCELLED"
  aiReasoning: string
  aiConfidence: number
  bingxOrderId: string
  openedAt: string
  closedAt: string | null
}
```

### AgentDecision
```typescript
interface AgentDecision {
  action: "BUY" | "SELL" | "HOLD"
  confidence: number         // 0.0 to 1.0
  reasoning: string
  stopLossPct: number        // 0.3 to 5.0
  takeProfitPct: number      // 0.5 to 15.0
}
```
