# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**RentalAssist** is being built on top of [FBScrapeIdeas](https://github.com/MasuRii/FBScrapeIdeas), a CLI tool that scrapes Facebook groups with Selenium and categorizes posts using an LLM. The goal is to repurpose it for rental listing extraction rather than thesis-idea mining.

All source code lives in `FBScrapeIdeas_src/`. Run all commands from that directory.

## Setup

```bash
cd FBScrapeIdeas_src
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
pip install -r requirements-dev.txt   # for linting/testing
```

Create `.env` in `FBScrapeIdeas_src/` (or let the interactive wizard create it on first run):

```dotenv
AI_PROVIDER=gemini          # or openai
GOOGLE_API_KEY=...          # if using Gemini
OPENAI_BASE_URL=...         # if using OpenAI-compatible (Ollama, OpenRouter, etc.)
OPENAI_API_KEY=...
OPENAI_MODEL=...
GEMINI_MODEL=models/gemini-2.0-flash
FB_USER=...
FB_PASS=...
```

For Claude via OpenRouter: `AI_PROVIDER=openai`, `OPENAI_BASE_URL=https://openrouter.ai/api/v1`, `OPENAI_MODEL=anthropic/claude-3-5-sonnet`, `OPENAI_API_KEY=<openrouter key>`.

## Commands

All run from `FBScrapeIdeas_src/`:

```bash
python main.py                                                  # interactive menu
python main.py scrape --group-url "URL" [--num-posts 50] [--headless]
python main.py process-ai                                       # run LLM on unprocessed rows
python main.py view [--category X] [--author Y] [--limit N]
python main.py export --format csv|json [--output-path PATH]
python main.py stats
python main.py add_group --name "Name" --url "URL"
python main.py setup                                            # re-run credentials wizard
```

## Linting and tests

```bash
ruff check .                        # lint
ruff format --check .               # format check
ruff format .                       # auto-format

python -m unittest discover -s tests -p "test_*.py" -v   # all tests
python -m unittest tests.test_facebook_scraper -v         # single test file
```

Ruff is configured in `pyproject.toml`: line length 100, double quotes, E/F/I/W/B/UP rules enabled, E501/E402/F401/B008 ignored.

## Architecture

The pipeline has two independent phases intentionally separated so you can iterate on prompts without re-scraping:

**Phase 1 — Scrape** (`main.py` → `scraper/facebook_scraper.py`):
- Selenium drives a real Chrome browser, logs in via FB's `#email`/`#pass` form, and scrolls the group feed
- Posts are parsed in parallel via a `ThreadPoolExecutor` (5 workers) using BeautifulSoup4 selectors
- Each post is written to SQLite immediately as it arrives (incremental save — crash-safe)
- Scroll loop continues until 3 consecutive scrolls produce no new posts, or `--num-posts` is reached
- Dedup is done in-memory via two sets (by URL and by FB post ID)

**Phase 2 — AI** (`main.py` → `ai/provider_factory.py`):
- Reads rows where `ai_category IS NULL` from the DB
- Batches them and calls `ai_provider.analyze_posts_batch()` / `analyze_comments_batch()`
- Writes structured JSON results back to the same rows as `ai_*` columns

**AI provider abstraction** (`ai/base_provider.py`, `ai/provider_factory.py`):
- `GeminiProvider` uses native structured output with `ai/gemini_schema.json`
- `OpenAIProvider` appends `POST_SCHEMA_FOR_PROMPT` text to the prompt and parses the returned JSON
- Switching providers is a `.env` change (`AI_PROVIDER=gemini|openai`)

**Database** (`database/db_setup.py`, `database/crud.py`) — SQLite, three tables:
- `Groups` — tracked group URLs
- `Posts` — one row per post; raw fields + `ai_category`, `ai_sub_category`, `ai_keywords`, `ai_summary`, `ai_is_potential_idea`, `ai_reasoning`
- `Comments` — one row per comment, linked to a post, with `ai_sentiment`

**Config** (`config.py`):
- DB and `.env` live in cwd in dev mode; in `~/Library/Application Support/FBScrapeIdeas/` for frozen binaries
- Credentials are prompted interactively if not in `.env` and offered to save

## Adapting for rental extraction

The cleanest path requires only two files to change — no code patches needed:

1. **`custom_prompts.json`** (create in `FBScrapeIdeas_src/`) — overrides default prompts. Keys: `post_categorization`, `comment_analysis`. Copy `custom_prompts.example.json` as a starting point.

2. **`ai/prompts.py`** — update `POST_SCHEMA_FOR_PROMPT` (for OpenAI path) to your rental schema (e.g., `price_monthly`, `currency`, `city`, `neighborhood`, `rooms`, `furnished`, `available_from`, `is_rental_offer`).

3. **`ai/gemini_schema.json`** — update if using Gemini native structured output.

After extraction, filter and rank directly against SQLite: `SELECT * FROM Posts WHERE ai_is_potential_idea = 1` (or whatever field maps to `is_rental_offer` in your schema).

## Fragility notes

- **BS4 selectors in `scraper/facebook_scraper.py`** break when Facebook redesigns its DOM. The "self-healing" is stacked CSS selector fallbacks — when all fail, fields come back empty. Update selectors by hand when this happens.
- **No proxy/stealth layer** — only random 1.5–3.5s scroll jitter. Use a throwaway FB account and a residential IP.
- Scrape rate is roughly 20–40 posts/minute (real browser limitation).
