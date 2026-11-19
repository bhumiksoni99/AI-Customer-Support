# AI Customer Support

An email support agent for a small online store. It watches a Gmail inbox, answers routine questions from a knowledge base, and hands everything else to a human.

The example business is a fictional store, Harbor & Pine, that sells inflatable paddle boards and flat-pack shelving on Amazon. The knowledge base is plain markdown, so you can swap in your own business.

Built following the [Customer Support Automation course](https://learn.datalumina.com/docs/customer-support-automation) by Datalumina, with Gemini models in place of the course's Claude models.

## How it works

```
Gmail inbox ──▶ poller ──▶ guards ──▶ classifier ──▶ agent ──▶ rules ──▶ reply or escalate
                              │            │             │
                              │            │             └── reads the knowledge base (Postgres)
                              │            └── picks a section, flags emails that need a human
                              └── skips our own mail, auto-replies, bulk mail
```

1. **Poll.** `app/poller.py` checks the inbox on an interval for messages that don't have the `ai-processed` label.
2. **Guard.** Plain code skips mail we sent, auto-submitted mail and bulk or list mail.
3. **Classify.** A small, fast model picks the knowledge base section and flags emails that always need a person: angry customers, refund requests, injuries, legal threats, attachments, and so on.
4. **Answer.** A second model reads the thread and the relevant knowledge base sections through tools, then drafts a reply as structured output.
5. **Decide.** Plain Python makes the final call. It escalates if the model says the knowledge base doesn't cover the question, if the reply promises a refund, shipment or date, or if a reported problem needs a human.
6. **Send.** The reply goes out in the same Gmail thread. Escalated threads get a short acknowledgement and a `needs-human` label, and the bot stays out of them until a colleague removes it.

The models propose and the code decides. Business rules such as "new board within 30 days of delivery, patch kit after that" live in `app/ai/rules.py` as ordinary functions, so they are testable and can't be talked around.

### Reliability

- **No double replies.** Every inbound Gmail message gets one row in `events`, keyed by its Gmail id. The `ai-processed` label and that row work together to make processing idempotent.
- **Retries.** A failed message is retried on later polls, up to 3 attempts, and then labelled so it can't loop forever. One bad email never stops the poller.
- **History.** Tickets (one per Gmail thread) and every inbound and outbound message are stored in Postgres.

## Stack

- Python 3.13, managed with [uv](https://docs.astral.sh/uv/)
- [pydantic-ai](https://ai.pydantic.dev/) for the classifier and agent, with Gemini models
- Gmail API for reading and sending mail
- Postgres on [Supabase](https://supabase.com/), accessed with psycopg
- FastAPI (a health endpoint for now)
- [pydantic-evals](https://ai.pydantic.dev/evals/) for a golden-email test set

## Setup

You need Python 3.13, `uv`, the [Supabase CLI](https://supabase.com/docs/guides/cli), a Supabase project, a Google Cloud project with the Gmail API enabled, and a Gemini API key.

**1. Install dependencies**

```
uv sync
```

**2. Create the database tables**

```
supabase link --project-ref <your-project-ref>
supabase db push --remote
```

**3. Get Gmail credentials**

Create an OAuth client (type: Desktop app) in Google Cloud Console and save it as `credentials.json` in the repo root. Then uncomment the three `print` lines at the bottom of `scripts/get_refresh_token.py` and run:

```
uv run python -m scripts.get_refresh_token
```

Sign in as the Gmail account the bot will use. Copy the printed values into `.env` and comment the prints out again. Keep the refresh token secret.

**4. Configure the environment**

```
cp .env.example .env
```

Fill in the Google values, `DATABASE_URL` and `GEMINI_API_KEY`. `CLASSIFIER_MODEL` and `AGENT_MODEL` are optional and use the `google:<model>` form. Values in `.env` override the defaults in `app/config.py`.

**5. Load the knowledge base**

```
uv run python -m scripts.seed_knowledge_base
```

Run it again after editing the markdown. It upserts by section and title.

**6. Run the bot**

```
uv run python -m app.poller
```

Send an email to the connected inbox from a different address. To check the Gmail connection by itself first, `uv run python -m scripts.gmail_check` connects and **sends a real reply** to the first unprocessed message, so only run it against test mail.

## Knowledge base

`knowledge_base/` holds one markdown file per section. The file name is the section, each `##` heading is one entry, and each file starts with a `# Title` line. The agent reads sections on demand with `list_sections` and `get_section` tools, so it only answers from what is written there. `brand-and-tone.md` and `escalation-rules.md` control how replies sound and when to hand off.

## Evals

`evals/sample_emails.py` holds a set of golden emails with the expected section and action for each, including non-English messages. `evals/run_evals.py` runs the pipeline over them and reports whether the section and action match.

```
uv run python -m evals.run_evals
```

## Project layout

```
app/
  poller.py        polling loop
  pipeline.py      one message end to end: claim, guard, decide, send, record
  guards.py        mail the bot must never answer
  gmail.py         Gmail client and message helpers
  db.py            Postgres access
  config.py        settings from .env
  ai/
    classifier.py  triage model
    agent.py       answer model and knowledge base tools
    decide.py      combines model output with rules into a final decision
    rules.py       deterministic business rules
    schemas.py     structured output types
knowledge_base/    markdown content, one file per section
supabase/          database migrations
scripts/           one-off setup scripts
evals/             golden emails and eval runner
```

## Security notes

- `.env` and `credentials.json` are git-ignored. Never commit them.
- The Gmail scope is `gmail.modify`: read, label and send, but not permanent delete.
- Row level security is enabled on every table with no policies, so the data is reachable only through the direct database connection, not through Supabase's public API.
- Customer emails are sent to the model provider. Check that this fits your privacy requirements before pointing the bot at a real inbox.
