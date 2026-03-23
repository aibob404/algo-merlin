# Algo-Merlin — Claude Code Context

This file is automatically loaded by Claude Code at the start of every session.
It contains project rules, architecture decisions, and workflow preferences.

---

## Project Overview

**Algo-Merlin** is an AI-powered automated crypto trading system running on BingX demo account.
The AI acts as an autonomous agent that uses tools to gather market data, analyze indicators,
and make trading decisions (BUY/SELL/HOLD) with dynamic stop-loss and take-profit levels.

---

## Repositories

| Repo | Tech | Purpose |
|------|------|---------|
| `algo-merlin` | Docs | System design, architecture, docker-compose, this file |
| `algo-merlin-platform` | Kotlin/Ktor | Backend trading engine, BingX client, REST API, Telegram bot |
| `algo-merlin-ai` | Python/FastAPI | AI agent microservice, tool execution, LLM via OpenRouter |
| `algo-merlin-ui` | React/TypeScript | Web dashboard — bot config, trade history, monitoring |

---

## Git Workflow Rules

- **NEVER commit directly to `main`**
- Always create a feature branch first: `feat/`, `fix/`, `docs/`, `chore/`
- Always open a PR with a clear description
- PRs are reviewed by Claude via `claude-code-action` (mention `@claude` for review)
- Branch naming: `feat/trading-engine`, `docs/system-design`, `fix/bingx-auth`

---

## Tech Stack

| Layer | Technology | Reason |
|-------|-----------|--------|
| Backend | Kotlin + Ktor | Type-safe, coroutines for async trading loop |
| AI Service | Python + FastAPI | Best ecosystem for ML/AI, pandas-ta for indicators |
| Frontend | React + TypeScript | Fast MVP UI development |
| Database | PostgreSQL | Reliable, good JSON support for trade metadata |
| LLM Gateway | OpenRouter | Single API for Claude, GPT, Gemini, Mistral |
| Exchange | BingX Demo (VST) | Safe demo trading environment |
| Infra | Docker + Docker Compose | Consistent deployment across environments |
| Notifications | Telegram Bot | Simple one-way trade notifications (MVP) |

---

## Architecture Decisions

### AI Agent Approach (Option B — Agentic)
The AI is not just called once per candle. It's an autonomous agent with tools it can call
to gather its own context before making a decision. This makes it extensible and more powerful.

**MVP Agent Tools:**
- `get_candles(symbol, timeframe, limit)` — fetch OHLCV data from BingX
- `get_indicators(symbol, timeframe)` — RSI, MACD, EMA20/50, Bollinger Bands
- `get_balance()` — current demo account balance
- `get_open_positions()` — currently open trades
- `place_order(symbol, side, quantity, stop_loss, take_profit)` — execute trade
- `close_position(position_id)` — close an open position

**Future Agent Tools (v2+):**
- `get_news_sentiment(symbol)` — crypto news analysis
- `get_fear_greed_index()` — market sentiment
- `get_funding_rate(symbol)` — perpetual futures funding rate
- `scan_pairs(symbols[])` — analyze multiple pairs

### Risk Management Rules (enforced by platform, not AI)
- Max risk per trade: **1% of account balance**
- Demo account balance: **$1,000 USD**
- Max position size calculated as: `balance * risk_pct / stop_loss_pct`
- One open position per bot at a time

### LLM Strategy
- Primary: Claude (via OpenRouter)
- Configurable per bot — can switch to GPT-4, Gemini, Mistral per bot
- Agent loop runs in Python AI service
- Platform (Kotlin) triggers the agent, agent calls back platform tools via HTTP

---

## Service Communication

```
UI (React) ←→ Platform (Kotlin) ←→ AI Service (Python) ←→ OpenRouter (LLM)
                    ↕                       ↕
               PostgreSQL              BingX Demo API
                    ↕
              Telegram Bot
```

---

## Infrastructure

### Production: K3s on VPS
- Two environments: `prod` (algo-merlin-prod namespace) and `test` (algo-merlin-test namespace)
- Managed via **Kustomize** overlays in `k8s/overlays/prod` and `k8s/overlays/test`
- Deploy: `kubectl apply -k k8s/overlays/prod` or `kubectl apply -k k8s/overlays/test`
- `prod` uses image tag `stable`, `test` uses `latest`
- Secrets created manually via `kubectl create secret` — never in Git (see `k8s/secrets/README.md`)
- Ingress via Traefik (built into K3s) with Let's Encrypt SSL

### Local Development: Docker Compose
- `docker-compose.yml` in repo root — mirrors production topology
- Uses `.env` file (copy from `.env.example`)

### Image Registry
- GitHub Container Registry: `ghcr.io/aibob404/algo-merlin-{service}:tag`
- Tags: `latest` (test), `stable` (prod)

---

## Environment Variables

| Variable | Service | Description |
|----------|---------|-------------|
| `BINGX_API_KEY` | platform | BingX API key (demo) |
| `BINGX_API_SECRET` | platform | BingX API secret (demo) |
| `OPENROUTER_API_KEY` | ai | OpenRouter API key |
| `TELEGRAM_BOT_TOKEN` | platform | Telegram bot token |
| `TELEGRAM_CHAT_ID` | platform | Telegram chat ID for notifications |
| `DB_URL` | platform | PostgreSQL connection URL |
| `AI_SERVICE_URL` | platform | Python AI service URL |
| `PLATFORM_URL` | ai | Kotlin platform URL (for tool callbacks) |

---

## PR Description Format

When opening PRs, use this structure:
```
## What
Brief description of what changed.

## Why
Reason for the change.

## How
Key implementation details.

## Testing
How to verify the change works.
```
