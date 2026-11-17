# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Uses `uv` (Python >= 3.13). There is no test suite or linter configured yet.

```
uv sync                                   # install deps (incl. dev group)
uv run fastapi dev app/main.py            # API server (only /health exists so far)
uv run python -m app.poller               # Gmail polling loop (the actual bot)
uv run python -m scripts.gmail_check      # smoke test: connects to Gmail, REPLIES to the first unprocessed inbox mail
uv run python -m scripts.get_refresh_token  # one-off OAuth flow (needs credentials.json) to get GOOGLE_REFRESH_TOKEN
supabase db push --remote                 # apply supabase/migrations to the hosted project
```

- Run scripts with `python -m <pkg>.<module>` from the repo root so `from app...` imports resolve.
- `supabase db push` targets the local Docker stack by default; the hosted project is linked, so pass `--remote`.
- Config comes from `.env` (see `.env.example`). `app/config.py` calls `load_dotenv()` to populate `os.environ`, then `Settings()` (pydantic-settings) reads from it. `google_*` and `database_url` are required; the app fails at import without them. LLMs are Gemini via pydantic-ai: `classifier_model`/`agent_model` use the `google:<model>` form (Gemini API; `google-cloud:` is Vertex) and `GEMINI_API_KEY` is read from `os.environ` by pydantic-ai directly (it is not a `Settings` field). Values in `.env` override the defaults in `config.py`.

## Architecture

A Gmail-polling support bot. `app/poller.py` loops every `poll_interval_seconds`, builds a `Gmail` client, and passes each new message to `app/pipeline.py:handle_message`. Errors are caught per message and per poll so one bad email never stops the loop.

Processing flow (`pipeline.py`):
1. `db.start_event` claims the message (see idempotency below).
2. `process()` runs `guards.skip_reason` (own mail, `Auto-Submitted`, bulk/list precedence) and returns "skipped" if it matches. Otherwise it upserts a ticket by Gmail thread id, saves the inbound message, sends a reply, saves the outbound message, and marks the ticket `answered`. The reply is currently a fixed `ACKNOWLEDGEMENT`; the classifier/agent step is not implemented.
3. The Gmail label `ai-processed` is added to the **message** (not the thread).

Idempotency and retries rely on two layers that must stay consistent:
- **Gmail label**: `list_new_message_ids` queries `in:inbox -label:ai-processed`, so labelled messages are never fetched again. A customer follow-up in the same thread is a new unlabelled message and gets processed.
- **`events` table**: `gmail_message_id` is unique. `start_event` inserts, or, on conflict, re-claims the row only if its status is `failed` (incrementing `attempts`). Otherwise it returns `None` and the message is just labelled. After `MAX_ATTEMPTS` (3) failures the message is labelled anyway so it stops retrying.

Data model (`supabase/migrations`): `events` (one per Gmail message, processing state), `tickets` (one per Gmail thread), `messages` (in/out, linked to ticket), `knowledge_base`. RLS is enabled on all tables with no policies, so access works only through the direct Postgres connection in `DATABASE_URL` (`app/db.py`, psycopg with autocommit, a new connection per call), not via Supabase's anon/public API.

`app/gmail.py` holds the `Gmail` wrapper and message helpers (`get_body`, `strip_quoted`, `format_thread`). On init it creates the `ai-processed` and `needs-human` labels if missing. `send_reply` sets `In-Reply-To`/`References` and `threadId` so replies stay threaded.
