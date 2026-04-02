```
  ██████╗  █████╗ ██████╗  █████╗ ██╗     ██╗      █████╗ ██╗  ██╗
  ██╔══██╗██╔══██╗██╔══██╗██╔══██╗██║     ██║     ██╔══██╗╚██╗██╔╝
  ██████╔╝███████║██████╔╝███████║██║     ██║     ███████║ ╚███╔╝
  ██╔═══╝ ██╔══██║██╔══██╗██╔══██║██║     ██║     ██╔══██║ ██╔██╗
  ██║     ██║  ██║██║  ██╗██║  ██║███████╗███████╗██║  ██║██╔╝ ██╗
  ╚═╝     ╚═╝  ╚═╝╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝╚══════╝╚═╝  ╚═╝╚═╝  ╚═╝
                              AI Research & Intelligence Agent
```

[![Python](https://img.shields.io/badge/Python-3.11+-blue?logo=python&logoColor=white)](https://python.org)
[![LangChain](https://img.shields.io/badge/LangChain-RAG-green?logo=chainlink&logoColor=white)](https://langchain.com)
[![ChromaDB](https://img.shields.io/badge/ChromaDB-vectorstore-orange)](https://trychroma.com)
[![Telegram](https://img.shields.io/badge/Telegram-bot-2CA5E0?logo=telegram&logoColor=white)](https://core.telegram.org/bots)
[![License](https://img.shields.io/github/license/ysz7/Parallax)](LICENSE)

Personal AI research agent. Collects signals from 8 web sources, scores them with LLM, combines news + market data into investment-grade briefings, and answers questions from your local knowledge base — all from a CLI or Telegram bot.

---

## Features

**RAG Knowledge Base**
- Drop `.pdf`, `.docx`, `.csv`, `.txt`, `.md` into `data/inbox/` — indexed automatically
- ChromaDB + `nomic-embed-text` embeddings
- Three search modes: `all` (docs + intel), `docs` (local only), `intel` (web only)
- Multi-turn conversation history

**Web Intelligence (`/research`)**  
Collects in parallel, deduplicates via MD5, scores every item 1–10 with LLM (CCW):

| Source | Auth |
|---|---|
| Hacker News | None |
| Reddit | None — configure via `REDDIT_SUBS` |
| GitHub Trending | None |
| Product Hunt | None |
| Mastodon | None — configure via `MASTODON_INSTANCE` |
| Dev.to | Optional `DEVTO_API_KEY` |
| Google Trends | None — configure via `GOOGLE_TRENDS_GEO` |
| Custom RSS | `/rss add <url>` |

Auto-research runs every N minutes in the background (`AUTO_RESEARCH_INTERVAL`).

**CCW Impact Scoring**

| Score | Meaning |
|---|---|
| 9–10 | Major breakthrough |
| 7–8 | Significant shift worth acting on |
| 5–6 | Interesting, worth following |
| 1–4 | Routine noise |

Scores rank `/news`, prioritize `/briefing` context, and trigger smart alerts (≥ 7).

**Market Data (`/market`)**
- Crypto via CoinGecko, stocks via Alpha Vantage + Finnhub, forex via open.er-api
- All free, no mandatory API keys
- SQLite history for trend tracking
- Auto-alerts on crypto ≥ 8% or stocks ≥ 5% move in 24h

**Daily Briefing (`/briefing`)**  
Combines top news (ranked by CCW) + market snapshots into a structured report:
signal themes, market analysis, dominant trend, capital flow, BUY/HOLD/AVOID verdict, 3 things to research today. Saved to `data/exports/`.

**Other**
- `/ideas` — 5 business ideas from current trends with entry strategy
- `/report <topic>` — deep research report combining RAG + LLM
- `/predict`, `/flows` — cross-niche predictions and money flow analysis
- `/remind` — natural language reminders (ru + en via `dateparser`)
- `/tokens` — Anthropic API usage and cost tracking
- Smart alerts via Telegram when signal score ≥ 7
- Morning briefing at configured hour (`MORNING_BRIEFING_HOUR`)
- Concurrent safety: in-process gate + file lock for CLI + bot running simultaneously

---

## Install

**Requirements:** Python 3.11+, [Ollama](https://ollama.com) (or Anthropic API key)

```bash
git clone https://github.com/ysz7/Parallax.git
cd Parallax
python -m venv .venv
```

```bash
# Windows
.venv\Scripts\activate
# Linux / macOS
source .venv/bin/activate
```

```bash
pip install -r requirements.txt
```

Pull local models (skip if using Anthropic):
```bash
ollama pull gemma3
ollama pull nomic-embed-text
```

---

## Configure

Create `.env` in the project root:

```env
# LLM — choose one
LLM_PROVIDER=ollama
OLLAMA_MODEL=gemma3

# or Anthropic (recommended for server deployments — no GPU needed)
# LLM_PROVIDER=anthropic
# ANTHROPIC_API_KEY=sk-ant-...
# ANTHROPIC_MODEL=claude-sonnet-4-20250514

# Telegram (required for bot.py)
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_USER_ID=         # your Telegram user ID; comma-separated for multiple

# Auto-research interval in minutes (0 = off)
AUTO_RESEARCH_INTERVAL=360

# Morning briefing hour in 24h format (0 = off), Telegram only
MORNING_BRIEFING_HOUR=8

# Optional market data keys (free tiers at alphavantage.co and finnhub.io)
ALPHAVANTAGE_KEY=
FINNHUB_KEY=

# Optional customization
REDDIT_SUBS=
MASTODON_INSTANCE=mastodon.social
GOOGLE_TRENDS_GEO=US
CRYPTO_COINS=bitcoin,ethereum,solana
STOCK_SYMBOLS=AAPL,NVDA,MSFT
FOREX_PAIRS=EUR,GBP,JPY
```

---

## CLI

```bash
python cli.py
# or on Windows:
start-cli.bat
```

Type any text to query the knowledge base. Use commands:

| Command | Description |
|---|---|
| `/research` | Collect all sources + CCW scoring + market snapshot |
| `/news [N]` | Last N items sorted by CCW score (default 10) |
| `/briefing [days]` | Full briefing: signals + market + investment direction |
| `/predict [days]` | Cross-niche predictions with probabilities |
| `/flows [days]` | Sectors where money is moving now |
| `/ideas` | 5 business ideas based on current trends |
| `/market` | Crypto + stocks + forex live snapshot |
| `/report <topic>` | Deep report on any topic |
| `/reports` | List / view / export / delete saved reports |
| `/rss list/add/remove` | Manage custom RSS feeds |
| `/remind <text>` | Add reminder (natural language, ru + en) |
| `/reminders` | List active reminders |
| `/stats` | Knowledge base stats + delete indexed files |
| `/tokens` | Anthropic token usage and cost |
| `/logs` | Recent error log |
| `/set lang <code>` | Response language: EN, RU, DE, etc. |

Drop a file path or drag & drop a file → indexed automatically.

---

## Telegram Bot

```bash
python bot.py
# or on Windows:
start-bot.bat
```

Same command set as CLI. Additional Telegram features:
- Send a file → indexed directly from the chat
- Smart alerts pushed automatically when signal score ≥ 7
- Morning briefing at `MORNING_BRIEFING_HOUR`
- Access restricted to `TELEGRAM_ALLOWED_USER_ID`

---

## Docker

```bash
docker compose up -d
```

```yaml
# docker-compose.yml
services:
  bot:
    build: .
    env_file: .env
    volumes:
      - ./data:/app/data
    restart: unless-stopped
```

Data persists in `./data/` on the host.

---

## Architecture

```
cli.py / bot.py              ← entry points, no business logic
    └── core/
        ├── rag_engine.py    ← ChromaDB vectorstore, document loading, RAG chain
        ├── intel.py         ← web collectors (8 sources), deduplication, feed storage
        ├── rss.py           ← custom RSS feed manager
        ├── ccw.py           ← AI impact scoring (1–10)
        ├── trends.py        ← trend analysis, briefing, opportunity scanner
        ├── agent.py         ← smart alerts (score ≥ 7), idea generator
        ├── market_pulse.py  ← CoinGecko + Alpha Vantage + Finnhub, SQLite history
        ├── reminders.py     ← JSON reminders, natural language date parsing
        ├── config.py        ← persistent settings
        ├── token_tracker.py ← Anthropic usage and cost tracking
        ├── error_logger.py  ← actionable error log
        └── formatter.py     ← markdown → ANSI (CLI) / HTML (Telegram)
```
