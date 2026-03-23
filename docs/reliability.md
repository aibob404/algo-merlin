# Reliability Design — Algo-Merlin

## Overview

This document covers failure modes, circuit breakers, timeouts, reconciliation patterns, and observability requirements. Since this system trades financial positions, reliability is treated as a first-class concern.

---

## 1. Timeouts (All HTTP Calls)

| Call | Timeout | Reason |
|------|---------|--------|
| Platform → AI Service (`/agent/decide`) | 30s | Accounts for 10 tool calls × ~2s each |
| AI Service → Platform (`/tools/execute`) | 10s | Tool execution should be fast |
| AI Service → OpenRouter (LLM) | 20s | LLM inference + queue time |
| Platform → BingX REST | 5s | Exchange APIs are typically fast |
| Database queries | 10s | Alert if exceeded |
| Database transactions | 30s | Rollback if exceeded |

---

## 2. Circuit Breakers

All external calls use a circuit breaker. Implementation: **Resilience4j** (Kotlin), **tenacity** (Python).

### AI Service Circuit Breaker (in Platform)
```
CLOSED  → OPEN:      3 consecutive failures within 60s
OPEN    → HALF_OPEN: after 60s
HALF_OPEN → CLOSED:  1 successful request
Fallback: return HOLD decision, log incident, send Telegram alert
```

### BingX Circuit Breaker (in Platform)
```
CLOSED  → OPEN:      5 failures within 60s
OPEN    → HALF_OPEN: after 120s
HALF_OPEN → CLOSED:  1 successful request
Fallback: use candle cache, pause position monitoring, alert operator
```

### OpenRouter Circuit Breaker (in AI Service)
```
CLOSED  → OPEN:      2 consecutive timeouts or 3 errors within 60s
OPEN    → HALF_OPEN: after 120s
HALF_OPEN → CLOSED:  1 successful request
Fallback: try model fallback chain (see below)
```

### LLM Model Fallback Chain
```
1. Primary:    configured per bot (e.g., claude-3-5-haiku)
2. Fallback 1: gpt-4o-mini (fast, cheap)
3. Fallback 2: gemini-flash-1.5 (fast, cheap)
4. Final:      return HOLD decision
```

---

## 3. Retry Strategy

| Call | Max Retries | Backoff | Retry On |
|------|------------|---------|----------|
| BingX REST | 3 | Exponential: 2s, 4s, 8s + jitter | 5xx, timeout, 429 |
| OpenRouter | 2 | Exponential: 1s, 2s + jitter | 5xx, timeout |
| AI Service → Platform tools | 1 | Immediate | 5xx, timeout |
| DB write | 2 | 500ms, 1s | Connection error |

**Never retry:** 4xx errors (bad request, auth failure) — fix the bug, not the retry.

---

## 4. Trade Write-Ahead Pattern (Critical)

**Problem:** Order placed on BingX, then DB write fails → ghost position.

**Solution: Write PENDING before calling BingX.**

```
1. Generate trade UUID locally
2. INSERT trade (status=PENDING, bingxOrderId=NULL) → DB ✓
3. Call BingX with clientOrderId = trade.id (idempotency key)
4. BingX confirms → UPDATE trade SET status=OPEN, bingxOrderId=xxx
5. If step 2 fails → cancel before calling BingX
6. If step 4 fails → reconciliation job handles recovery
```

**Idempotency:** If Platform retries the BingX call, BingX sees the same `clientOrderId` and rejects the duplicate. No double orders.

---

## 5. Startup Position Reconciliation

Run on every Platform startup before the trading loop begins:

```
1. GET open positions from BingX
2. SELECT * FROM trades WHERE status = 'OPEN'
3. Compare:
   - In BingX but not DB → INSERT as RECOVERED, alert operator
   - In DB but not BingX → UPDATE status=CLOSED, exitPrice=last_known, alert operator
   - In both → verify prices match, update position_monitor
4. Find stuck PENDING (> 5 min old):
   - Query BingX by clientOrderId
   - If filled → mark OPEN
   - If not found → mark CANCELLED
5. Log reconciliation result, send Telegram summary
6. Only start trading loops after reconciliation completes
```

---

## 6. Reconciliation Job (Every 5 Minutes)

Background job, separate from trading loop:

```kotlin
fun reconcileOrders() {
    val stale = db.query(
        "SELECT * FROM trades WHERE status = 'PENDING' AND created_at < NOW() - INTERVAL '5 minutes'"
    )
    stale.forEach { trade ->
        val bingxOrder = bingx.getOrderByClientId(trade.id)
        when (bingxOrder?.status) {
            "FILLED"    → db.update(trade.id, status=OPEN, bingxOrderId=bingxOrder.id)
            "CANCELLED" → db.update(trade.id, status=CANCELLED)
            null        → db.update(trade.id, status=CANCELLED) // not found on BingX
            else        → db.insert(TradeFailure(trade, "unknown_bingx_status"))
        }
    }
}
```

---

## 7. Candle Timing — Prevent Duplicate Analysis

Trading engine tracks the last analyzed candle per bot:

```kotlin
val lastAnalyzedClose = mutableMapOf<BotId, Instant>()

suspend fun tradingLoop(bot: Bot) {
    while (isActive) {
        val candles = bingxClient.getCandles(bot.symbol, bot.timeframe, limit = 2)
        val latestClose = candles.last().closeTime

        if (latestClose > (lastAnalyzedClose[bot.id] ?: Instant.EPOCH)) {
            lastAnalyzedClose[bot.id] = latestClose
            triggerDecision(bot)
        }
        delay(10.seconds)
    }
}
```

This prevents duplicate decisions if the polling interval overlaps with the candle close boundary.

---

## 8. Graceful Shutdown

Platform handles SIGTERM (sent by Kubernetes on pod termination):

```
On SIGTERM:
1. Stop accepting new bot start commands
2. Pause all trading loops (no new /agent/decide calls)
3. Wait up to 30s for in-flight decisions to complete
4. Write all open position states to DB
5. Send Telegram: "⚠️ Platform shutting down. Open positions monitored by BingX SL/TP."
6. Close DB connections
7. Exit 0
```

Kubernetes config:
```yaml
terminationGracePeriodSeconds: 60
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

---

## 9. Agent Loop Safety

- **Max iterations:** 10 tool calls per decision
- **Early exit:** If LLM returns a final decision before iteration 10, exit immediately
- **Forced exit:** At iteration 9, inject message: *"Make your final decision now. No more tool calls."*
- **Decision timeout:** 30s hard limit on entire agent loop (separate from per-tool timeout)
- **Fallback:** If agent times out or hits max iterations without deciding → HOLD

---

## 10. Inter-Service Authentication

### Platform ↔ AI Service
Shared secret token for the `/tools/execute` callback endpoint:
```
Header: Authorization: Bearer {TOOL_EXECUTION_TOKEN}
Token stored in K8s Secret, injected as env var in both services
Platform validates token on every /api/tools/execute request
```

### Platform REST API ↔ UI
JWT authentication:
```
POST /auth/login → returns signed JWT (exp: 24h)
All /api/* endpoints require: Authorization: Bearer {JWT}
JWT validated on every request (signature + expiry)
```

---

## 11. Observability Requirements

### Structured Logging
Both services emit JSON logs to stdout:
```json
{
  "timestamp": "2025-03-23T10:00:00Z",
  "level": "INFO",
  "service": "platform",
  "request_id": "abc-123",
  "bot_id": "uuid",
  "message": "Agent decision received: BUY, confidence=0.82"
}
```

### Metrics (Prometheus endpoints on `/metrics`)

**Platform:**
- `trading_decisions_total{action, bot_id}` — decisions made
- `ai_service_latency_ms` — latency to AI service
- `bingx_api_errors_total{endpoint}` — BingX error rate
- `open_positions_count` — current open positions
- `circuit_breaker_state{service}` — 0=closed, 1=half-open, 2=open

**AI Service:**
- `agent_tool_calls_total{tool_name}` — tool call frequency
- `agent_decision_duration_ms` — per-decision latency
- `openrouter_latency_ms` — LLM API latency
- `agent_iterations_used` — tool call count per decision

### Alert Rules (Prometheus AlertManager)

| Alert | Condition | Severity |
|-------|-----------|----------|
| AI Service down | Unavailable > 2 min | CRITICAL |
| Platform down | Unavailable > 2 min | CRITICAL |
| Position mismatch | DB vs BingX > 0 | CRITICAL |
| BingX error rate | > 20% in 5m window | HIGH |
| Circuit breaker open | Any service | HIGH |
| Stuck PENDING trades | Any trade > 10 min | HIGH |
| Agent max iterations | Hit rate > 10% | MEDIUM |
| Decision latency | p99 > 15s | MEDIUM |

---

## 12. Production K8s Reliability Config

```yaml
# Platform: minimum 2 replicas
replicas: 2

# Pod disruption budget (keep at least 1 running during node drain)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: platform-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: platform

# AI Service: minimum 2 replicas in prod
minReplicas: 2
maxReplicas: 5
```

### Network Policies
```
Platform     ← UI (port 8080)
Platform     ← Ingress (port 8080)
AI Service   ← Platform only (port 8001)
PostgreSQL   ← Platform only (port 5432)
```
