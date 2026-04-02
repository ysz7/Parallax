# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**Parallax** — AI Research & Intelligence Agent combining a local knowledge base (ChromaDB + LangChain) with web intelligence collection (HN, Reddit, GitHub, Product Hunt) and market data. The project is mid-refactor: moving from a FastAPI+React+Gradio stack to a pure CLI + Telegram bot stack. See `refactor-plan.txt` for the full plan.

## Running the project

```bash
# Activate venv (Python 3.14, Windows)
.venv\Scripts\activate

# CLI (current)
python cli.py

# FastAPI backend (current, being removed)
python backend/main.py          # → http://localhost:8000

# Frontend (current, being removed)
cd frontend && npm run dev      # → http://localhost:5173
cd frontend && npm run build
cd frontend && npm run lint

# After refactor — primary modes:
python bot.py    # Telegram bot
python cli.py    # Local CLI
```

Ollama must be running locally with models pulled:
```bash
ollama pull gemma3
ollama pull nomic-embed-text
```

## Architecture

### Core modules (stable, keep as-is)
All business logic lives in `core/`. Entry points (`cli.py`, `bot.py`) must only call core functions — no logic duplication.

| Module | Responsibility |
|---|---|
| `core/rag_engine.py` | ChromaDB vectorstore, document loading/chunking, RAG query chain. `ask()`, `index_file()`, `index_inbox()`, `stats()`, `delete_file()` |
| `core/intel.py` | Collects HN, Reddit, GitHub Trending, Product Hunt. Deduplicates via MD5 hashes. `collect_and_index()` indexes new items into RAG. Background scheduler every 6h |
| `core/trends.py` | LLM-based trend analysis, daily briefing, opportunity scanner, quick post-collection summary |
| `core/agent.py` | Smart Alerts (signal scoring → LLM alert if score ≥ 7), Idea Generator (5 business ideas/day). Currently emails results — **replace with Telegram notify during refactor** |
| `core/reminders.py` | JSON-backed reminders, natural language date parsing (ru+en via `dateparser`), background watcher thread firing system notifications |
| `core/market_pulse.py` | CoinGecko (crypto, no key), open.er-api (forex, no key), Alpha Vantage + Finnhub (stocks, optional keys) |

### Being removed
- `core/gmail_client.py` — email replaced by Telegram
- `webui.py` — Gradio UI
- `backend/` — entire FastAPI layer
- `frontend/` — entire React/Vite UI

### Data paths (current → planned)

Current: data scattered across `~/my-datacenter/` (separate from code).

Target after refactor — everything co-located:
```
datacenter/
├── core/
├── data/
│   ├── inbox/        ← files for indexing
│   ├── processed/    ← after indexing
│   ├── vector_db/    ← ChromaDB
│   ├── exports/      ← reports
│   ├── intel/        ← HN/Reddit/GitHub cache + market_pulse.json
│   ├── reminders.json
│   └── config.json
├── cli.py
├── bot.py
├── .env
└── requirements.txt
```

Key change in `rag_engine.py`: `BASE_DIR = Path(__file__).parent.parent / "data"`

### LLM provider switch (to implement)

`.env` controls the provider:
```
LLM_PROVIDER=ollama        # or anthropic
OLLAMA_MODEL=gemma3
ANTHROPIC_API_KEY=sk-...
ANTHROPIC_MODEL=claude-sonnet-4-20250514
```

`get_llm()` factory to add in `rag_engine.py`:
```python
if provider == "anthropic":
    from langchain_anthropic import ChatAnthropic
    return ChatAnthropic(model=model)
else:
    from langchain_ollama import ChatOllama
    return ChatOllama(model=model)
```

## Refactor priorities (from refactor-plan.txt)

1. **Delete**: `webui.py`, `core/gmail_client.py`, `backend/`, `frontend/`
2. **Remove email calls** from `agent.py` and `trends.py`; replace with a `notify(text)` function that calls the Telegram bot
3. **Rewrite `cli.py`**: drag-and-drop file detection, multiline input (Shift+Enter), command history via `pyreadline3`, progress indicator during indexing, full command set: `/research /stats /ideas /market /briefing /remind /reminders /report /help /exit`
4. **Create `bot.py`**: `python-telegram-bot>=20.0`, async handlers, same command set as CLI, file upload → index pipeline, `TELEGRAM_ALLOWED_USER_ID` whitelist
5. **Update paths** in all `core/` modules to use `data/` subdir relative to project root
6. **Update `requirements.txt`**: remove `gradio`, add `python-telegram-bot>=20.0 python-dotenv langchain-anthropic pyreadline3`

## Code conventions

- All code, comments, and logs in **English**
- Single config source: `.env` (use `python-dotenv`)
- `market_pulse.py` currently reads config via `gmail_client.load_config()` — after removing gmail, give it its own `.env`-based config
