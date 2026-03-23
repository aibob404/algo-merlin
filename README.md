# 🧙 Algo-Merlin

> AI-powered autonomous crypto trading system for BingX demo account.

Algo-Merlin uses an agentic AI approach — the AI doesn't just receive indicators, it actively uses tools to gather market context, reason across multiple data sources, and make informed trading decisions.

---

## Repositories

| Repo | Purpose |
|------|---------|
| [`algo-merlin`](https://github.com/aibob404/algo-merlin) | Docs, system design, docker-compose |
| [`algo-merlin-platform`](https://github.com/aibob404/algo-merlin-platform) | Kotlin/Ktor backend — trading engine, BingX client, REST API |
| [`algo-merlin-ai`](https://github.com/aibob404/algo-merlin-ai) | Python/FastAPI — AI agent, indicators, LLM via OpenRouter |
| [`algo-merlin-ui`](https://github.com/aibob404/algo-merlin-ui) | React — web dashboard, bot config, trade history |

---

## How It Works

```
┌─────────────────────────────────────────────────┐
│                  Web UI (React)                  │
│         Configure bots · Review trades           │
└───────────────────────┬─────────────────────────┘
                        │ REST API
┌───────────────────────▼─────────────────────────┐
│             Platform (Kotlin/Ktor)               │
│  Trading Engine · Risk Manager · Telegram Bot    │
└──────────┬────────────────────────┬─────────────┘
           │                        │
┌──────────▼──────────┐  ┌─────────▼────────────┐
│   BingX Demo API    │  │   AI Service (Python) │
│  Market data        │  │  Agent loop           │
│  Order execution    │  │  Tool execution       │
└─────────────────────┘  │  LLM via OpenRouter   │
                         └─────────────────────--┘
           │                        │
┌──────────▼────────────────────────▼─────────────┐
│                  PostgreSQL                       │
│        Bots · Trades · Candles · Logs            │
└─────────────────────────────────────────────────┘
```

### Agent Decision Flow

```
Platform triggers agent (new candle closed)
  └─> AI Agent starts reasoning
        ├─> tool: get_indicators("BTC-USDT", "1h")
        ├─> tool: get_open_positions()
        ├─> tool: get_balance()
        └─> LLM reasons over all data
              ├─> HOLD → do nothing
              └─> BUY/SELL
                    └─> Risk Manager validates
                          └─> tool: place_order(...)
                                └─> Telegram notification
```

---

## Key Features

- **Agentic AI** — LLM actively gathers context via tools before deciding
- **Configurable per bot** — symbol, timeframe, AI model (Claude/GPT/Gemini/Mistral)
- **Risk management** — enforced 1% max risk per trade, dynamic position sizing
- **Multi-model** — switch LLM per bot via OpenRouter
- **Web dashboard** — configure bots, review trade history, monitor performance
- **Telegram notifications** — real-time alerts on trade open/close

---

## Quick Start

```bash
cp .env.example .env
# Fill in your API keys
docker-compose up -d
```

See [docs/setup.md](docs/setup.md) for full setup guide.

---

## Documentation

| Doc | Description |
|-----|-------------|
| [System Design](docs/system-design.md) | Full architecture and component design |
| [Agent Design](docs/agents.md) | AI agent tools, flow, and prompts |
| [API Contracts](docs/api-contracts.md) | Service interfaces and data models |
| [ADR Index](docs/adr/) | Architecture Decision Records |
| [Setup Guide](docs/setup.md) | How to run locally and on server |

---

## Tech Stack

- **Backend:** Kotlin + Ktor + Exposed + PostgreSQL
- **AI Service:** Python + FastAPI + pandas-ta + OpenRouter
- **Frontend:** React + TypeScript
- **Exchange:** BingX Demo (VST API)
- **Infra:** Docker + Docker Compose
