# NotebookLM → Discord Q&A Bot — Design

## Summary

A Discord bot that relays user questions to a fixed Google NotebookLM notebook and
replies with the generated answer and citations. Users ask via a `/ask` slash command;
the bot forwards the question to NotebookLM (via the unofficial `notebooklm-py`
CLI/SDK), formats the response, and posts it back in Discord.

This is a **new, standalone repository** — separate from the `py-fastapi-project`
codebase it was designed alongside. It does not use this project's FastAPI backend,
database, or frontend.

## Goals

- Let people in a Discord server ask questions and get answers sourced from a
  specific NotebookLM notebook, without opening the NotebookLM web UI.
- Keep the system simple: one notebook, one bot, stateless per-question interactions.
- Run entirely on a local machine (no VPS / cloud hosting cost).

## Non-goals

- No conversation memory / follow-up context across questions.
- No multi-notebook routing or per-channel notebook selection.
- No Line OA integration (may be added later as a second adapter, out of scope now).
- No 24/7 uptime guarantee — bot is only online while the local machine is running.

## Architecture

Single Python process, containerized:

```
Discord user
   │  /ask "question"
   ▼
discord.py bot (bot.py)
   │  defer() → call wrapper
   ▼
NotebookLM client wrapper (nlm_client.py)
   │  notebooklm ask "<question>" --json -n <NOTEBOOK_ID>
   ▼
notebooklm-py CLI (unofficial, cookie-based Google auth)
   │
   ▼
Google NotebookLM (fixed notebook, sources pre-loaded)
```

## Components

1. **`bot.py`** — discord.py client. Registers a single slash command:
   `/ask <question>`. On invocation:
   - `interaction.response.defer()` immediately (NotebookLM calls take several
     seconds, longer than Discord's 3s ack window).
   - Calls the NotebookLM client wrapper.
   - Edits the deferred response with the formatted answer.

2. **`nlm_client.py`** — thin wrapper around `notebooklm-py`. Responsible for:
   - Invoking `notebooklm ask "<question>" --json -n <NOTEBOOK_ID>` (subprocess or
     the package's async Python client, whichever proves more reliable in testing).
   - Parsing `answer` and `references` (citations) from the JSON response.
   - Raising typed errors (`AuthExpiredError`, `RateLimitedError`, `TimeoutError`) so
     `bot.py` can map them to user-facing messages.

3. **Config (`.env`)**:
   - `DISCORD_BOT_TOKEN`
   - `NOTEBOOKLM_NOTEBOOK_ID` — the single fixed notebook all questions query.
   - `NOTEBOOKLM_AUTH_JSON` — inline `storage_state.json` contents (per
     notebooklm-py's CI/CD auth pattern), so the container doesn't need an
     interactive browser login.

4. **Deploy** — Docker container run locally via `docker compose up -d`, with
   `restart: unless-stopped`. A local cron/launchd job refreshes the NotebookLM
   auth session periodically (see Auth Maintenance below).

## Data Flow

1. User runs `/ask <question>` in Discord.
2. Bot defers the interaction response.
3. Bot calls `nlm_client.ask(question)`.
4. Wrapper shells out to `notebooklm ask ... --json -n <NOTEBOOK_ID>`.
5. Wrapper parses `{"answer": ..., "references": [...]}`.
6. Bot builds a Discord embed: answer text (chunked if needed) + a "Sources" field
   listing cited passages.
7. Bot edits the original deferred response with the embed.
8. On any failure, bot edits the response with a short Thai-language error message
   and logs the underlying exception.

## Error Handling

| Condition | Behavior |
|---|---|
| Answer text > Discord's 2000-char limit | Split across multiple embed fields, or truncate with a "(ตัดข้อความ)" note |
| Google rate limit / RPC error | Retry once with backoff; if still failing, reply "ลองใหม่ใน 5 นาที" |
| Auth cookie expired | Catch, log to a local log file / admin-visible channel, reply "บอทขัดข้องชั่วคราว" |
| Call exceeds ~30s | Abort, reply "คิดนานผิดปกติ ลองใหม่" |
| Notebook has no ready sources yet | Reply pointing user to check back later (not expected to occur often; notebook is pre-populated before bot goes live) |

## Auth Maintenance

`notebooklm-py` authenticates via the operator's personal Google account cookies —
there is no official NotebookLM API. This has two consequences that must be handled
operationally, not just in code:

- **Session refresh**: cookies rotate/expire. A local scheduled job
  (`launchd` on macOS, or cron) runs `notebooklm auth refresh --quiet` every
  15–20 minutes to keep the session warm.
- **Machine sleep risk**: if the local machine sleeps, the scheduled refresh job
  won't fire, increasing the chance the session goes stale between fires. Mitigate
  by disabling sleep while the bot is expected to be available, or by accepting
  that stale sessions require a manual `notebooklm login` re-run.

⚠️ **ToS caveat**: automating a personal Google account this way sits outside
Google's NotebookLM terms of service. Acceptable for personal / small-group use;
do not scale this to high query volume or many concurrent users — that increases
risk of the account being flagged.

## Testing

- **Unit tests** for `nlm_client.py`: mock the `notebooklm` CLI/subprocess call,
  verify correct parsing of `answer`/`references` and correct error typing for
  malformed/error JSON.
- **Manual integration test**: before wiring up Discord, run
  `notebooklm ask "..." -n <NOTEBOOK_ID> --json` directly against the real notebook
  (with sources already loaded and status `ready`) to confirm answers come back
  as expected.
- **Manual end-to-end test**: run the bot locally against a test Discord server,
  invoke `/ask`, confirm embed formatting, error paths (temporarily revoke auth to
  test the expired-cookie path), and the 2000-char split/truncate path with a
  long question.

## Open Questions / Future Work (explicitly out of scope for this spec)

- Line OA adapter, if needed later, would be a second bot process reusing
  `nlm_client.py` — not designed here.
- Multi-notebook or per-channel routing, if usage grows beyond one notebook.
