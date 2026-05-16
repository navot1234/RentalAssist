# FBScrapeIdeas — How it works

Cloned from https://github.com/MasuRii/FBScrapeIdeas into `FBScrapeIdeas_src/`. This is the repo we picked as a starter for RentalAssist because it does authenticated scraping of private Facebook groups plus LLM categorization — almost exactly your use case, just aimed at "thesis ideas" instead of rental offers.

## The big picture

It's a two-phase pipeline, and that separation is the most important design choice in the repo:

1. **Scrape phase** — a real Chrome browser, driven by Selenium, logs into Facebook with credentials you supply, navigates to a group URL, scrolls the feed, and parses every post + its comments out of the live HTML. Everything is written into a local SQLite database as it comes in (incremental save, so a crash doesn't lose progress).
2. **AI phase** — runs separately against the SQLite DB. Reads the "unprocessed" posts/comments, sends them in batches to the configured LLM, writes the structured output back as new columns on the same rows.

This separation is exactly the right pattern for you. Once you've scraped a group, you can iterate on your rental-extraction prompt as many times as you want without re-scraping (and without burning your FB account on repeated logins).

## How the scraper actually works

The interesting code is in `scraper/facebook_scraper.py`. The shape:

**Login** (`login_to_facebook`): drives the real Facebook login form by element IDs (`#email`, `#pass`, `name=login`), dismisses cookie banners, then waits for the feed element to appear as the "you're logged in" signal. Wrapped in `tenacity` retry with exponential backoff.

**Group scrape loop** (`scrape_authenticated_group`): a generator that yields posts as it finds them. The loop is roughly:

- Navigate to the group URL, wait for the feed container.
- Scroll to the bottom with a random 1.5–3.5s jitter delay (that's the only meaningful anti-detection trick in here — no proxy use, no stealth plugins).
- Wait for new post elements to appear; bail if three consecutive scrolls produce no new posts.
- Dismiss any modal overlays Facebook throws up ("Save your login info?", "Turn on notifications?", etc.) by walking a list of XPath patterns.
- For each post element on the page:
    - Pull its permalink and post-id from the link's `href` (tries `/posts/`, `/videos/`, `/photos/`, query params like `story_fbid`, regex fallback). If nothing works, generates a UUID as the local key.
    - Click "See more" if present to expand truncated text.
    - Grab the post's `outerHTML` and hand it off to a worker thread in a `ThreadPoolExecutor` (5 workers) for BeautifulSoup parsing.
- Dedup by URL and ID via two `set`s so the same post isn't yielded twice when scrolling re-renders it.

**HTML extraction** (`_extract_data_from_post_html`): the part most likely to break. It's BeautifulSoup selectors for each field — author name, profile pic, post text, post image, timestamp, comments. Every selector has multiple fallbacks chained together with commas:

```python
AUTHOR_NAME_BS = f"{AUTHOR_NAME_PRIMARY_BS}, {ANON_AUTHOR_NAME_BS}, {GENERAL_AUTHOR_NAME_BS}"
```

That's what the README means by "self-healing selectors" — there isn't anything clever happening, just a stack of selectors so when Facebook breaks one, the next one in line probably still works. When *all* of them break (Facebook ships a redesign), you'll need to update them by hand.

Timestamps are extracted by trying `<abbr title="...">`, then specific link text, then any link with a `title`/`aria-label`/text that `dateparser.parse()` accepts. Parsed timestamps then go through a custom `parse_fb_timestamp` (in `scraper/timestamp_parser.py`) which handles Facebook's relative formats ("2h", "Yesterday at 4:30pm").

For each post, comments are extracted the same way and embedded inside the post dict as a `comments` list.

## How the AI phase works

`ai/provider_factory.py` is the entry point. Based on `AI_PROVIDER` in `.env` it instantiates either `GeminiProvider` or `OpenAIProvider`. Both implement the same `AIProvider` base class with `analyze_posts_batch(posts)` and `analyze_comments_batch(comments)`.

In `main.py`'s `handle_process_ai_command`:

1. Get the provider.
2. Query `get_unprocessed_posts()` — posts with no `ai_category` yet.
3. Batch them (`create_post_batches`) so each LLM call processes multiple posts at once.
4. For each batch: `await ai_provider.analyze_posts_batch(batch)` → returns a JSON array where each object has `internal_post_id` plus `category`, `subCategory`, `keywords`, `summary`, `isPotentialIdea`, `reasoning`.
5. Loop the results and call `update_post_with_ai_results(conn, id, result)`.
6. Same again for comments (with sentiment + category).

The prompts live in `ai/prompts.py`. There are two defaults — `post_categorization` and `comment_analysis` — and **the customization hook is `custom_prompts.json`** in the project root. If that file exists, its keys override the defaults. So to repurpose this repo for rentals, you don't fork the code — you drop a `custom_prompts.json` with your rental-extraction prompt and adjust the JSON schema constants. Example structure is in `custom_prompts.example.json`.

The JSON schema is enforced two ways depending on provider:
- **Gemini**: native structured output via `ai/gemini_schema.json` (Gemini's `responseSchema` feature).
- **OpenAI-compatible**: the schema is appended to the prompt as text (`POST_SCHEMA_FOR_PROMPT` in `prompts.py`), and the model is asked to return JSON matching it.

For your rental-offer schema, the OpenAI path is what you'd modify, and `gemini_schema.json` if you want to keep Gemini's native enforcement.

## Configuration model

`config.py` is the credentials and paths layer. Worth knowing:

- **Cross-platform app data dir**: `~/Library/Application Support/FBScrapeIdeas` on macOS, `%APPDATA%\FBScrapeIdeas` on Windows, `~/.local/share/fbscrapeideas` on Linux. The `.env` and `insights.db` live there for the frozen (binary) build; in dev mode the cwd takes precedence.
- **Credentials**: prompted via `getpass` if not in `.env`; offered to save. Keys: `FB_USER`, `FB_PASS`, `GOOGLE_API_KEY`, `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `OPENAI_MODEL`, `GEMINI_MODEL`, `AI_PROVIDER`.
- **First-run wizard** (`run_setup_wizard`): interactive setup that walks new users through API key + FB creds.
- **OpenAI base URL trick**: if base URL contains `localhost` or `127.0.0.1` the API key check is skipped — that's how Ollama and LM Studio work without keys.

To point this at Claude, you'd set `AI_PROVIDER=openai` and `OPENAI_BASE_URL=https://openrouter.ai/api/v1`, `OPENAI_MODEL=anthropic/claude-3.5-sonnet` (or newer), `OPENAI_API_KEY=<your OpenRouter key>`. The README literally has this as an example.

## Database

SQLite. Schema is set up in `database/db_setup.py` (didn't read it in detail but the CRUD module in `database/crud.py` reveals the shape):

- `Groups` (group_id, group_name, group_url)
- `Posts` (internal_post_id, group_id, facebook_post_id, post_url, post_author_name, post_author_profile_pic_url, post_image_url, post_content_raw, posted_at, scraped_at, plus AI columns: `ai_category`, `ai_sub_category`, `ai_keywords`, `ai_summary`, `ai_is_potential_idea`, `ai_reasoning`)
- `Comments` (one row per comment, linked to a post, with similar AI columns including `ai_sentiment`)

`get_unprocessed_posts()` is just `WHERE ai_category IS NULL`. That's how the AI phase knows what's new.

## CLI surface

All commands run via `python main.py <cmd>`:

- `scrape --group-url URL [--num-posts 50] [--headless]` — login + scrape into DB
- `process-ai` — run the LLM phase on whatever's unprocessed
- `view [--category X] [--author Y] [--limit N]` — interactive browser with filter menus
- `export --format csv|json [--output-path PATH] [--category ...]` — dump to file
- `stats` — totals per category, top authors, comment counts
- `add_group`/`list_groups`/`remove_group` — manage tracked groups
- `setup` — re-run the wizard

The interactive menu (`cli/menu_handler.py`) wraps these as numbered choices when you run `python main.py` with no args.

## What's good for our use case

- **Authenticated scraping is exactly what you need.** Most private rental groups won't render posts without login; this handles that.
- **Provider abstraction.** Swapping in Claude (or a local model) is a config change, not a code change.
- **`custom_prompts.json` hook.** Your rental extraction schema is a JSON file, not a code patch.
- **SQLite-first design.** Means you can run `process-ai` repeatedly with different prompts to refine extraction without re-paying the scrape cost.
- **Incremental save during scraping.** A network blip mid-scrape doesn't lose what was already collected.

## What you'll have to deal with

- **Selectors will break eventually.** Facebook redesigns its DOM every few months. The "self-healing" is just stacked fallbacks; when the whole stack misses, the scraper returns empty fields. You'll need to update the BS4 selectors at the top of `facebook_scraper.py` when that happens.
- **No proxy / stealth layer.** Random scroll-delay jitter is the only anti-detection. Run from a residential IP, not a datacenter. Use a throwaway FB account — not your main — because eventually it'll get challenged or restricted.
- **Single maintainer, ~40 stars.** Treat it as a working scaffold you'll maintain, not turnkey infrastructure.
- **Selenium speed.** A real browser scrolling Facebook is slow — figure ~20-40 posts/minute best case. Fine for personal-scale (one group, a few times a day); painful at scale.
- **The README is wrong about Playwright.** Tagline says "Playwright & Gemini AI" but the code is Selenium + webdriver-manager + BS4. Not a bug, just heads-up.

## Suggested next steps for RentalAssist

1. **Sanity-run it** — create a throwaway FB account, join one rental group, install requirements (`pip install -r FBScrapeIdeas_src/requirements.txt`), run `python main.py scrape --group-url <your_group> --num-posts 5` and confirm posts land in SQLite.
2. **Write the rental prompt + schema.** Create `custom_prompts.json` with a rental-extraction prompt and update `POST_SCHEMA_FOR_PROMPT` in `ai/prompts.py` (or `gemini_schema.json` if going Gemini-native) to your rental schema: `{price_monthly, currency, city, neighborhood, rooms, sqm, furnished, available_from, contact, post_language, is_rental_offer (bool)}`.
3. **Wire Claude** via OpenRouter as the provider — `.env` change only.
4. **Add a ranker on top.** The repo stops at "categorized posts." You'll want a step that filters `is_rental_offer == true` and scores by your criteria (max price, min rooms, neighborhoods, recency) — that's a small Python script on top of the SQLite DB.
5. **Schedule it.** Once stable, cron the `scrape` then `process-ai` pair, and have the ranker post matches to wherever you want them (email, Slack, push notification).

Source files now in `FBScrapeIdeas_src/` if you want to dig in further. The repo's README and `CHANGELOG.md` are in there too.

---
Sources:
- [FBScrapeIdeas on GitHub](https://github.com/MasuRii/FBScrapeIdeas)
