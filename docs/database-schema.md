# Database Schema — Algo-Merlin

## Overview

PostgreSQL is the single source of truth for all system state. The schema is designed around 4 concerns:
- **Operational** — trades, positions, bots (transactional, real-time)
- **Decision audit** — agent reasoning logs (immutable, insert-only)
- **Market cache** — candles, indicators (ephemeral, high-volume)
- **Observability** — failures, account snapshots (reconciliation & analytics)

---

## Trade State Machine

```
                    ┌─────────────────────────────┐
                    │     PENDING                  │
                    │  (PENDING row written BEFORE │
                    │   BingX call — write-ahead)  │
                    └──────────────┬───────────────┘
                                   │ BingX confirms
                    ┌──────────────▼───────────────┐
                    │     OPEN                     │
                    │  (position active, monitored) │
                    └──┬──────────┬────────────┬───┘
                       │          │            │
                    SL hit    TP hit       manual close
                       │          │            │
                  ┌────▼──┐  ┌───▼──┐  ┌──────▼──┐  ┌───────┐
                  │SL_HIT │  │TP_HIT│  │ CLOSED  │  │ ERROR │
                  └───────┘  └──────┘  └─────────┘  └───────┘

PENDING  → OPEN       (BingX confirms fill)
PENDING  → CANCELLED  (rejected by risk manager or BingX)
PENDING  → ERROR      (DB write ok but BingX unconfirmed — reconcile)
OPEN     → SL_HIT     (stop-loss triggered)
OPEN     → TP_HIT     (take-profit triggered)
OPEN     → CLOSED     (agent called close_position)
OPEN     → ERROR      (monitoring lost, crash — needs reconciliation)
ERROR    → CLOSED     (manual reconciliation)
```

---

## Full Schema

```sql
-- ============================================================
-- BOTS
-- ============================================================

CREATE TABLE bots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    symbol          VARCHAR(20)  NOT NULL,   -- "BTC-USDT"
    timeframe       VARCHAR(10)  NOT NULL,   -- "1m","5m","15m","1h","4h","1d"
    ai_model        VARCHAR(100) NOT NULL,   -- OpenRouter model ID
    strategy_hint   VARCHAR(50)  NOT NULL
                    CHECK (strategy_hint IN (
                        'momentum','mean_reversion','breakout',
                        'conservative','aggressive'
                    )),
    max_risk_pct    DECIMAL(5,2) NOT NULL
                    CHECK (max_risk_pct > 0 AND max_risk_pct <= 3.0),
    is_active       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bots_is_active ON bots(is_active);
CREATE INDEX idx_bots_symbol    ON bots(symbol);

-- ============================================================
-- TRADES
-- ============================================================

CREATE TYPE trade_side AS ENUM ('BUY', 'SELL');
CREATE TYPE trade_status AS ENUM (
    'PENDING',    -- Row written before BingX call (write-ahead)
    'OPEN',       -- Position active on BingX
    'CLOSED',     -- Closed at market by agent
    'SL_HIT',     -- Stop-loss triggered
    'TP_HIT',     -- Take-profit triggered
    'CANCELLED',  -- Rejected before execution
    'ERROR'       -- Needs manual reconciliation
);

CREATE TABLE trades (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bot_id              UUID NOT NULL REFERENCES bots(id) ON DELETE RESTRICT,
    bingx_order_id      VARCHAR(100),              -- NULL until BingX confirms
    symbol              VARCHAR(20)  NOT NULL,
    side                trade_side   NOT NULL,

    -- Entry
    entry_price         DECIMAL(18,8) NOT NULL,
    entry_quantity      DECIMAL(18,8) NOT NULL CHECK (entry_quantity > 0),
    entry_time          TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    -- Risk params (DB-enforced)
    stop_loss_pct       DECIMAL(5,2) NOT NULL
                        CHECK (stop_loss_pct  >= 0.3 AND stop_loss_pct  <= 5.0),
    take_profit_pct     DECIMAL(5,2) NOT NULL
                        CHECK (take_profit_pct >= 0.5 AND take_profit_pct <= 15.0),
    stop_loss_price     DECIMAL(18,8) NOT NULL,
    take_profit_price   DECIMAL(18,8) NOT NULL,

    -- Exit
    exit_price          DECIMAL(18,8),
    exit_time           TIMESTAMPTZ,
    exit_reason         VARCHAR(50),               -- "SL_HIT","TP_HIT","AGENT_CLOSE"

    -- P&L (null while open)
    pnl_usd             DECIMAL(18,8),
    pnl_pct             DECIMAL(5,2),

    -- Status
    status              trade_status NOT NULL DEFAULT 'PENDING',

    -- AI decision snapshot
    ai_reasoning        TEXT         NOT NULL,
    ai_confidence       DECIMAL(3,2) NOT NULL CHECK (ai_confidence BETWEEN 0 AND 1),

    -- SL/TP direction validation
    CONSTRAINT sl_tp_direction CHECK (
        (side = 'BUY'  AND stop_loss_price < entry_price AND take_profit_price > entry_price) OR
        (side = 'SELL' AND stop_loss_price > entry_price AND take_profit_price < entry_price)
    ),

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_trades_bot_id        ON trades(bot_id);
CREATE INDEX idx_trades_status        ON trades(status);
CREATE INDEX idx_trades_status_symbol ON trades(status, symbol);
CREATE INDEX idx_trades_entry_time    ON trades(entry_time DESC);
CREATE INDEX idx_trades_bot_entry     ON trades(bot_id, entry_time DESC);

-- ============================================================
-- AGENT DECISION LOG (immutable — insert only, never update)
-- ============================================================

CREATE TABLE agent_decisions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bot_id                  UUID NOT NULL REFERENCES bots(id),
    trade_id                UUID REFERENCES trades(id),  -- NULL if HOLD
    symbol                  VARCHAR(20)  NOT NULL,
    timeframe               VARCHAR(10)  NOT NULL,

    action                  VARCHAR(10)  NOT NULL CHECK (action IN ('BUY','SELL','HOLD')),
    confidence              DECIMAL(3,2) NOT NULL CHECK (confidence BETWEEN 0 AND 1),
    reasoning               TEXT         NOT NULL,

    suggested_sl_pct        DECIMAL(5,2),
    suggested_tp_pct        DECIMAL(5,2),

    tool_calls_count        INT  NOT NULL DEFAULT 0 CHECK (tool_calls_count BETWEEN 0 AND 10),
    total_duration_ms       INT,
    was_executed            BOOLEAN NOT NULL DEFAULT false,

    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_agent_decisions_bot_id     ON agent_decisions(bot_id);
CREATE INDEX idx_agent_decisions_trade_id   ON agent_decisions(trade_id);
CREATE INDEX idx_agent_decisions_created_at ON agent_decisions(created_at DESC);
CREATE INDEX idx_agent_decisions_bot_created ON agent_decisions(bot_id, created_at DESC);

-- ============================================================
-- AGENT TOOL CALLS (per-call log under each decision)
-- ============================================================

CREATE TABLE agent_tool_calls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    decision_id     UUID NOT NULL REFERENCES agent_decisions(id) ON DELETE CASCADE,

    tool_name       VARCHAR(50)  NOT NULL,   -- "get_indicators", "place_order", etc.
    call_sequence   INT          NOT NULL CHECK (call_sequence > 0),

    request_args    JSONB        NOT NULL,   -- Tool input parameters
    response_result JSONB        NOT NULL,   -- Tool output
    error_message   TEXT,                   -- Non-null if tool call failed

    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    finished_at     TIMESTAMPTZ,
    duration_ms     INT
);

CREATE INDEX idx_tool_calls_decision_id ON agent_tool_calls(decision_id);
CREATE INDEX idx_tool_calls_tool_name   ON agent_tool_calls(tool_name);

-- ============================================================
-- CANDLE CACHE (ephemeral — 30-day TTL, nightly cleanup)
-- ============================================================

CREATE TABLE candle_cache (
    id                  BIGSERIAL PRIMARY KEY,
    symbol              VARCHAR(20)  NOT NULL,
    timeframe           VARCHAR(10)  NOT NULL,
    open_time           TIMESTAMPTZ  NOT NULL,
    close_time          TIMESTAMPTZ  NOT NULL,
    open_price          DECIMAL(18,8) NOT NULL,
    high_price          DECIMAL(18,8) NOT NULL,
    low_price           DECIMAL(18,8) NOT NULL,
    close_price         DECIMAL(18,8) NOT NULL,
    volume              DECIMAL(18,8) NOT NULL,
    cached_at           TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    CONSTRAINT candle_unique UNIQUE (symbol, timeframe, open_time)
);

CREATE INDEX idx_candle_symbol_timeframe_time ON candle_cache(symbol, timeframe, open_time DESC);
CREATE INDEX idx_candle_cached_at             ON candle_cache(cached_at DESC);

-- Nightly cleanup: DELETE FROM candle_cache WHERE cached_at < NOW() - INTERVAL '30 days';

-- ============================================================
-- INDICATOR CACHE (recomputed on each candle close)
-- ============================================================

CREATE TABLE indicator_cache (
    id              BIGSERIAL PRIMARY KEY,
    symbol          VARCHAR(20)  NOT NULL,
    timeframe       VARCHAR(10)  NOT NULL,
    current_price   DECIMAL(18,8) NOT NULL,
    rsi             DECIMAL(5,2),
    macd            DECIMAL(18,8),
    macd_signal     DECIMAL(18,8),
    macd_histogram  DECIMAL(18,8),
    ema_20          DECIMAL(18,8),
    ema_50          DECIMAL(18,8),
    bb_upper        DECIMAL(18,8),
    bb_middle       DECIMAL(18,8),
    bb_lower        DECIMAL(18,8),
    atr             DECIMAL(18,8),
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT indicator_unique UNIQUE (symbol, timeframe)
);

CREATE INDEX idx_indicator_symbol_timeframe ON indicator_cache(symbol, timeframe);

-- ============================================================
-- POSITION MONITOR (tracks 10s monitoring loop state)
-- ============================================================

CREATE TABLE position_monitor (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trade_id                UUID NOT NULL REFERENCES trades(id) UNIQUE,
    last_check_time         TIMESTAMPTZ NOT NULL,
    last_price              DECIMAL(18,8) NOT NULL,
    unrealized_pnl_usd      DECIMAL(18,8),
    unrealized_pnl_pct      DECIMAL(5,2),
    times_checked           INT NOT NULL DEFAULT 0,
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_position_monitor_last_check ON position_monitor(last_check_time);

-- ============================================================
-- ACCOUNT SNAPSHOTS (for analytics and backtesting)
-- ============================================================

CREATE TABLE account_snapshots (
    id                  BIGSERIAL PRIMARY KEY,
    total_balance_usd   DECIMAL(18,8) NOT NULL,
    available_usd       DECIMAL(18,8) NOT NULL,
    in_use_usd          DECIMAL(18,8) NOT NULL,
    open_position_count INT          NOT NULL,
    unrealized_pnl_usd  DECIMAL(18,8),
    snapshot_time       TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_account_snapshots_time ON account_snapshots(snapshot_time DESC);

-- ============================================================
-- TRADE FAILURES (reconciliation log)
-- ============================================================

CREATE TABLE trade_failures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bot_id          UUID NOT NULL REFERENCES bots(id),
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    action          VARCHAR(50)  NOT NULL,   -- "place_order","close_position","fetch_balance"
    symbol          VARCHAR(20),
    error_type      VARCHAR(100),            -- "bingx_timeout","db_write_failed","reconcile_mismatch"
    error_message   TEXT NOT NULL,
    bingx_response  JSONB,
    resolved_at     TIMESTAMPTZ,
    resolution_notes TEXT
);

CREATE INDEX idx_trade_failures_bot_id      ON trade_failures(bot_id);
CREATE INDEX idx_trade_failures_resolved_at ON trade_failures(resolved_at);

-- ============================================================
-- USEFUL VIEWS
-- ============================================================

CREATE VIEW trade_performance AS
SELECT
    bot_id,
    COUNT(*)                                                        AS total_trades,
    SUM(CASE WHEN pnl_usd > 0 THEN 1 ELSE 0 END)                  AS wins,
    SUM(CASE WHEN pnl_usd <= 0 THEN 1 ELSE 0 END)                  AS losses,
    ROUND(SUM(CASE WHEN pnl_usd > 0 THEN 1 ELSE 0 END)::numeric
          / NULLIF(COUNT(*), 0) * 100, 1)                           AS win_rate_pct,
    ROUND(SUM(pnl_usd)::numeric, 2)                                 AS total_pnl_usd,
    ROUND(AVG(pnl_pct)::numeric, 2)                                 AS avg_pnl_pct,
    MAX(pnl_usd)                                                    AS best_trade_usd,
    MIN(pnl_usd)                                                    AS worst_trade_usd
FROM trades
WHERE status IN ('CLOSED','SL_HIT','TP_HIT')
GROUP BY bot_id;
```

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Write `PENDING` before BingX call | Prevents ghost trades if DB write fails after order placed |
| `clientOrderId = trade.id` on BingX | Idempotency — safe to retry without double-ordering |
| `agent_decisions` insert-only | Immutable audit trail for compliance and debugging |
| `agent_tool_calls` JSONB | Flexible schema as tools evolve; indexed for analytics |
| `strategy_hint` DB CHECK constraint | Enforces whitelist at DB level, prevents prompt injection |
| `stop_loss_pct` / `take_profit_pct` CHECK constraints | Risk rules enforced at DB level, not just application |
| Candle cache 30-day TTL | Reduces BingX API calls; enables backtesting |
| Separate `trade_failures` table | Clean separation of normal audit from error reconciliation |

---

## Reconciliation Queries

```sql
-- Find stuck PENDING trades (> 5 min old — need reconciliation)
SELECT * FROM trades
WHERE status = 'PENDING'
  AND created_at < NOW() - INTERVAL '5 minutes';

-- Position mismatch check (run on startup)
SELECT COUNT(*) FROM trades WHERE status = 'OPEN';
-- Compare with BingX open positions count

-- Full agent decision trace for a trade
SELECT ad.*, array_agg(atc.* ORDER BY atc.call_sequence) AS tool_calls
FROM agent_decisions ad
LEFT JOIN agent_tool_calls atc ON atc.decision_id = ad.id
WHERE ad.trade_id = $1
GROUP BY ad.id;

-- Slowest tool calls (performance debugging)
SELECT tool_name, AVG(duration_ms), MAX(duration_ms), COUNT(*)
FROM agent_tool_calls
GROUP BY tool_name
ORDER BY AVG(duration_ms) DESC;
```
