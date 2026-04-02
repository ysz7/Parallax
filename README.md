# Parallax

Personal intelligence agent: RAG knowledge base + web research + market data + Telegram bot.

Collects signals from 8 sources, scores them with AI, combines news + market data into investment-grade briefings, and surfaces concrete opportunities — all from a CLI or Telegram.

---

## Features

### RAG Knowledge Base
- Index local files (`.pdf`, `.docx`, `.csv`, `.txt`, `.md`) by dropping them in `data/inbox/` or passing a path in the CLI
- ChromaDB vectorstore with `nomic-embed-text` embeddings
- Ask any question — answer is generated from your indexed documents + collected intel
- Multi-turn conversation history in both CLI and Telegram
- Three search modes: `all` (docs + intel), `docs` (local files only), `intel` (web research only)
- `/stats` shows chunk counts per file; delete indexed files by number

### Web Intelligence Collection (`/research`)
Collects fresh items from 8 sources in parallel, deduplicates via MD5, scores each item with AI (CCW 1–10), and generates an instant summary of what's new:

| Source | Auth |
|---|---|
| Hacker News | None |
| Reddit | None — customize subreddits via `REDDIT_SUBS` |
| GitHub Trending | None |
| Product Hunt | None |
| Mastodon | None — configure instance via `MASTODON_INSTANCE` |
| Dev.to | Optional `DEVTO_API_KEY` for higher rate limits |
| Google Trends | None — set region via `GOOGLE_TRENDS_GEO` |
| Custom RSS | Add any feed via `/rss add <url>` |

Auto-research runs every N minutes in the background (set `AUTO_RESEARCH_INTERVAL`). After each research run, a market snapshot is also fetched automatically so briefings always have fresh prices.

### CCW Impact Scoring
Every new item is scored 1–10 by the LLM for real-world impact:

| Score | Meaning |
|---|---|
| 9–10 🌍 | Major breakthrough / civilisation-level event |
| 7–8 ⚡ | Significant shift worth acting on |
| 5–6 💡 | Interesting, worth following |
| 1–4 | Routine noise |

CCW scores rank `/news`, prioritize `/briefing` context, and trigger smart alerts (score ≥ 7).

### Market Data (`/market`)
Fetches a live snapshot of crypto, stocks, and forex using free public APIs (no mandatory API keys):

- **Crypto** — CoinGecko: price, 24h change, market cap, volume (configurable via `CRYPTO_COINS`)
- **Stocks** — Alpha Vantage + Finnhub: price, 24h change (configurable via `STOCK_SYMBOLS`)
- **Forex** — open.er-api: USD rates for configured currency pairs (`FOREX_PAIRS`)
- All snapshots are stored in SQLite (`data/intel/market_history.db`) for trend tracking
- Alert signals fire automatically when crypto moves ≥ 8% or stocks ≥ 5% in 24h

### Daily Briefing (`/briefing`)
Combines top news from the last 30 days (ranked by CCW) with up to 10 market snapshots into a structured report:

- **Signal themes** — recurring topics across 2+ sources (confirmed signals, not noise)
- **Market analysis** — price movements with interpretation of risk appetite
- **Dominant trend** — strongest combined news + market signal
- **Capital flow** — sectors attracting builders and investors right now
- **Investment direction** — explicit BUY / HOLD / AVOID verdict for crypto, stocks, and niches to enter or avoid
- **3 things to research today** — specific items from the data with one-line rationale

Saved to `data/exports/briefing_YYYYMMDD_HHMM.md`.

### Trend & Opportunity Analysis
- **`/ideas`** — 5 business ideas generated from current trends, each with entry strategy and competition estimate. Saved to `data/exports/`
- **`/report <topic>`** — deep research report on any topic combining RAG + live LLM analysis

### Smart Alerts (background)
Runs on a configurable interval, scores recent items, and sends a Telegram notification when a signal score ≥ 7 is detected. Alert history saved to `data/intel/alerts_log.json`.

### Morning Briefing (Telegram)
Set `MORNING_BRIEFING_HOUR=8` to receive an automatic daily briefing at the configured hour.

### Reminders (`/remind`)
Natural language date parsing in Russian and English via `dateparser`:
```
/remind Call Alex tomorrow at 3pm
/remind Quarterly review in 2 weeks
```
Fires a system notification (desktop) or Telegram message when due. Active reminders listed via `/reminders`.

### Custom RSS Feeds
```
/rss list            — show all feeds
/rss add <url>       — add a feed
/rss remove <N>      — remove feed by number
```

### Token Tracking (`/tokens`)
Tracks Anthropic API usage and cost across all operations. Reset the counter with `/tokens reset`.

### Error Log (`/logs`)
Actionable log of recent errors — source, message, context — for debugging without log file diving.

### Response Language (`/set lang`)
Switch the agent's response language at runtime:
```
/set lang RU
/set lang EN
/set lang DE
```

### Concurrent Safety
Heavy commands (`/research`, `/briefing`, `/ideas`, `/market`) use two-level locking:
- **In-process gate** — if the same command is already running, a second caller waits and receives the same result (no duplicate LLM calls)
- **File lock** (`filelock`) — safe across CLI + Telegram bot running simultaneously, and Docker containers sharing `data/`

---

## Quick Start

**Requirements:** Python 3.11+, [Ollama](https://ollama.com) (or Anthropic API key)

```bash
ollama pull gemma3
ollama pull nomic-embed-text
```

```bash
git clone <repo>
cd parallax
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # Linux/macOS
pip install -r requirements.txt
```

### Configure `.env`

```env
# LLM — choose one
LLM_PROVIDER=ollama
OLLAMA_MODEL=gemma3

# or Anthropic (recommended for server — no GPU needed)
# LLM_PROVIDER=anthropic
# ANTHROPIC_API_KEY=sk-ant-...
# ANTHROPIC_MODEL=claude-sonnet-4-20250514

# Telegram
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_USER_ID=123456789   # your Telegram user ID (comma-separated for multiple)

# Auto-research interval in minutes (0 = off)
AUTO_RESEARCH_INTERVAL=360

# Morning briefing hour in 24h format (0 = off), Telegram only
MORNING_BRIEFING_HOUR=8
```

### Run

```bash
python cli.py    # local CLI
python bot.py    # Telegram bot

# Windows shortcuts (auto-activate venv + install deps on first run):
start-cli.bat
start-bot.bat
```

---

## Commands

| Command | Description |
|---|---|
| `/research` | Collect all sources + CCW scoring + auto-summary + market snapshot |
| `/news [N]` | Last N news items sorted by CCW score (default 10) |
| `/briefing` | Full briefing: signal themes + market analysis + investment direction |
| `/ideas` | 5 business ideas based on current trends |
| `/market` | Crypto + stocks + forex live snapshot |
| `/report <topic>` | Deep report on any topic |
| `/reports` | List / view / delete saved reports |
| `/rss list` | List configured RSS feeds |
| `/rss add <url>` | Add an RSS feed |
| `/rss remove <N>` | Remove an RSS feed |
| `/tokens` | Anthropic token usage and cost |
| `/tokens reset` | Reset token counter |
| `/logs` | Recent error log |
| `/set lang <code>` | Response language: EN, RU, DE, etc. |
| `/remind <text>` | Add reminder (natural language date, ru+en) |
| `/reminders` | List active reminders |
| `/stats` | Knowledge base stats + delete indexed files |

**Any text** → RAG search across the knowledge base.

**File path or drag & drop** → indexes the file automatically.

---

## Architecture

```
cli.py / bot.py              ← entry points, no business logic
    │
    └── core/
        ├── rag_engine.py    ← ChromaDB vectorstore, document loading, RAG chain
        ├── intel.py         ← web collectors (8 sources), deduplication, feed storage
        ├── rss.py           ← custom RSS feed manager
        ├── ccw.py           ← AI impact scoring (1-10) via LLM batch
        ├── trends.py        ← trend analysis, briefing (news+market), opportunity scanner
        ├── agent.py         ← smart alerts (score ≥ 7), idea generator
        ├── market_pulse.py  ← CoinGecko + Alpha Vantage + Finnhub, SQLite history
        ├── reminders.py     ← JSON reminders, natural language date parsing
        ├── config.py        ← persistent settings (language, search mode, etc.)
        ├── token_tracker.py ← Anthropic usage and cost tracking
        ├── error_logger.py  ← actionable error log
        └── formatter.py     ← markdown → ANSI (CLI) / HTML (Telegram)
```

### Data layout

```
data/
├── inbox/              ← drop files here for automatic indexing
├── processed/          ← files after indexing
├── vector_db/          ← ChromaDB embeddings
├── exports/            ← generated reports and briefings (.md)
├── config.json         ← persistent settings (lang, mode)
├── tokens.json         ← Anthropic token usage log
├── logs/errors.json    ← error log
└── intel/
    ├── latest_feed.json     ← cached feed (last 3000 items)
    ├── seen_hashes.json     ← dedup hashes (last 10k)
    ├── rss_feeds.json       ← custom RSS feed list
    ├── market_history.db    ← SQLite: market snapshots over time
    ├── alerts_log.json      ← smart alerts history
    └── ideas_log.json       ← generated ideas cache
```

---

## Deployment

### Docker

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "bot.py"]
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

### VPS without Docker

```bash
# Linux/macOS
source .venv/bin/activate
python bot.py
```

Use `screen`, `tmux`, or systemd to keep it running. Set `LLM_PROVIDER=anthropic` — no GPU or Ollama needed on the server.

---

## Environment Variables

```env
# ── LLM ───────────────────────────────────────────
LLM_PROVIDER=ollama              # ollama | anthropic
OLLAMA_MODEL=gemma3
ANTHROPIC_API_KEY=
ANTHROPIC_MODEL=claude-sonnet-4-20250514

# ── Telegram ───────────────────────────────────────
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_USER_ID=        # comma-separated for multiple users
MORNING_BRIEFING_HOUR=8          # 24h format, 0 = off

# ── Research ───────────────────────────────────────
AUTO_RESEARCH_INTERVAL=360       # minutes, 0 = off
REDDIT_SUBS=                     # comma-separated subreddits, empty = defaults
MASTODON_INSTANCE=mastodon.social
DEVTO_API_KEY=                   # optional, for higher rate limits
GOOGLE_TRENDS_GEO=US             # 2-letter country code

# ── Market data ────────────────────────────────────
ALPHAVANTAGE_KEY=                # free at alphavantage.co
FINNHUB_KEY=                     # free at finnhub.io
CRYPTO_COINS=bitcoin,ethereum,solana
STOCK_SYMBOLS=AAPL,NVDA,MSFT
FOREX_PAIRS=EUR,GBP,JPY
```
