# NotebookLM Discord Bot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone Discord bot that answers `/ask` questions by querying a fixed Google NotebookLM notebook and posting the answer + citations back to Discord.

**Architecture:** Single Python process (`discord.py` client) that shells out to the `notebooklm` CLI (from `notebooklm-py`) for each question, parses the JSON response, formats it into a Discord embed, and replies. No database, no web server, no conversation state.

**Tech Stack:** Python 3.11+, `discord.py`, `notebooklm-py` (CLI, invoked via subprocess), `pytest` + `pytest-asyncio`, Docker.

## Global Constraints

- This is a **new standalone repository**, separate from `py-fastapi-project`. Root path: `/Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot`. All file paths below are relative to that root unless stated otherwise.
- Single fixed notebook per spec — notebook ID comes from `NOTEBOOKLM_NOTEBOOK_ID` env var, never chosen per-request.
- Stateless per question — no conversation memory across `/ask` calls (per spec Non-goals).
- User-facing bot messages are in Thai (per spec's error-handling table); code, comments, commit messages are in English.
- Runs locally via Docker (`restart: unless-stopped`), not on a VPS.
- `notebooklm-py` authenticates via personal Google cookies — code must never assume an API key exists.

---

### Task 1: Repository scaffolding

**Files:**
- Create: `notebooklm-discord-bot/.gitignore`
- Create: `notebooklm-discord-bot/requirements.txt`
- Create: `notebooklm-discord-bot/.env.example`
- Create: `notebooklm-discord-bot/README.md`

**Interfaces:**
- Produces: a Python package root other tasks add modules to; `requirements.txt` other tasks append dependencies to.

- [ ] **Step 1: Create the repo directory and git init**

```bash
mkdir -p /Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot
cd /Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot
git init
```

- [ ] **Step 2: Write `.gitignore`**

```gitignore
__pycache__/
*.pyc
.env
.venv/
venv/
*.egg-info/
.pytest_cache/
storage_state.json
```

- [ ] **Step 3: Write `requirements.txt`**

```
discord.py>=2.4.0
python-dotenv>=1.0.0
"notebooklm-py[browser]"
pytest>=8.0.0
pytest-asyncio>=0.24.0
```

- [ ] **Step 4: Write `.env.example`**

```
DISCORD_BOT_TOKEN=
NOTEBOOKLM_NOTEBOOK_ID=
NOTEBOOKLM_AUTH_JSON=
```

- [ ] **Step 5: Write `README.md` skeleton**

```markdown
# NotebookLM Discord Bot

Discord bot that answers `/ask` questions from a fixed Google NotebookLM notebook.

## Setup

1. Copy `.env.example` to `.env` and fill in `DISCORD_BOT_TOKEN` and `NOTEBOOKLM_NOTEBOOK_ID`.
2. Authenticate NotebookLM: `notebooklm login`, then copy the contents of the resulting
   `storage_state.json` into `NOTEBOOKLM_AUTH_JSON` in `.env`.
3. `pip install -r requirements.txt`
4. `python bot.py`

See later sections of this README (added in later tasks) for Docker deployment and
auth-refresh scheduling.
```

- [ ] **Step 6: Verify the directory installs cleanly**

Run: `cd /Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot && python3 -m venv .venv && .venv/bin/pip install -r requirements.txt`
Expected: exits 0, no errors (network required; `notebooklm-py[browser]` pulls Playwright deps — this may take a minute).

- [ ] **Step 7: Commit**

```bash
git add .gitignore requirements.txt .env.example README.md
git commit -m "chore: scaffold notebooklm-discord-bot repo"
```

---

### Task 2: Config loader

**Files:**
- Create: `notebooklm-discord-bot/config.py`
- Test: `notebooklm-discord-bot/tests/test_config.py`

**Interfaces:**
- Consumes: environment variables `DISCORD_BOT_TOKEN`, `NOTEBOOKLM_NOTEBOOK_ID`.
- Produces: `config.load_config() -> Config` where `Config` is a dataclass with fields
  `discord_bot_token: str` and `notebooklm_notebook_id: str`. Raises `MissingConfigError`
  (subclass of `RuntimeError`) listing every missing variable name if any are unset/empty.

- [ ] **Step 1: Create `tests/__init__.py` (empty, makes tests a package)**

```bash
mkdir -p /Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot/tests
touch /Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot/tests/__init__.py
```

- [ ] **Step 2: Write the failing test**

`tests/test_config.py`:
```python
import pytest
from config import load_config, MissingConfigError


def test_load_config_reads_env(monkeypatch):
    monkeypatch.setenv("DISCORD_BOT_TOKEN", "token123")
    monkeypatch.setenv("NOTEBOOKLM_NOTEBOOK_ID", "nb-abc")

    config = load_config()

    assert config.discord_bot_token == "token123"
    assert config.notebooklm_notebook_id == "nb-abc"


def test_load_config_raises_on_missing_vars(monkeypatch):
    monkeypatch.delenv("DISCORD_BOT_TOKEN", raising=False)
    monkeypatch.delenv("NOTEBOOKLM_NOTEBOOK_ID", raising=False)

    with pytest.raises(MissingConfigError) as exc_info:
        load_config()

    assert "DISCORD_BOT_TOKEN" in str(exc_info.value)
    assert "NOTEBOOKLM_NOTEBOOK_ID" in str(exc_info.value)


def test_load_config_raises_on_empty_string(monkeypatch):
    monkeypatch.setenv("DISCORD_BOT_TOKEN", "")
    monkeypatch.setenv("NOTEBOOKLM_NOTEBOOK_ID", "nb-abc")

    with pytest.raises(MissingConfigError) as exc_info:
        load_config()

    assert "DISCORD_BOT_TOKEN" in str(exc_info.value)
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd /Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot && .venv/bin/pytest tests/test_config.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'config'`

- [ ] **Step 4: Write `config.py`**

```python
import os
from dataclasses import dataclass


class MissingConfigError(RuntimeError):
    pass


@dataclass(frozen=True)
class Config:
    discord_bot_token: str
    notebooklm_notebook_id: str


def load_config() -> Config:
    required = ["DISCORD_BOT_TOKEN", "NOTEBOOKLM_NOTEBOOK_ID"]
    missing = [name for name in required if not os.environ.get(name)]
    if missing:
        raise MissingConfigError(
            f"Missing required environment variable(s): {', '.join(missing)}"
        )
    return Config(
        discord_bot_token=os.environ["DISCORD_BOT_TOKEN"],
        notebooklm_notebook_id=os.environ["NOTEBOOKLM_NOTEBOOK_ID"],
    )
```

- [ ] **Step 5: Run test to verify it passes**

Run: `.venv/bin/pytest tests/test_config.py -v`
Expected: 3 passed

- [ ] **Step 6: Commit**

```bash
git add config.py tests/__init__.py tests/test_config.py
git commit -m "feat: add env-based config loader"
```

---

### Task 3: NotebookLM client wrapper

**Files:**
- Create: `notebooklm-discord-bot/errors.py`
- Create: `notebooklm-discord-bot/nlm_client.py`
- Test: `notebooklm-discord-bot/tests/test_nlm_client.py`

**Interfaces:**
- Consumes: nothing from earlier tasks (calls `subprocess.run` directly against the
  `notebooklm` CLI).
- Produces:
  - `errors.NotebookLMError(Exception)` — base class.
  - `errors.AuthExpiredError(NotebookLMError)`.
  - `errors.RateLimitedError(NotebookLMError)`.
  - `errors.AskTimeoutError(NotebookLMError)`.
  - `nlm_client.Reference` dataclass: `source_id: str`, `citation_number: int`, `cited_text: str`.
  - `nlm_client.AskResult` dataclass: `answer: str`, `references: list[Reference]`.
  - `nlm_client.ask(question: str, notebook_id: str, timeout: float = 30.0) -> AskResult`.
    Later tasks (bot.py) call this and catch the `errors` types above.

- [ ] **Step 1: Write `errors.py`**

```python
class NotebookLMError(Exception):
    """Base error for all NotebookLM CLI failures."""


class AuthExpiredError(NotebookLMError):
    """The NotebookLM CLI's Google session cookie has expired."""


class RateLimitedError(NotebookLMError):
    """Google rate-limited the request; caller should retry later."""


class AskTimeoutError(NotebookLMError):
    """The notebooklm CLI did not respond within the allotted timeout."""
```

- [ ] **Step 2: Write the failing tests**

`tests/test_nlm_client.py`:
```python
import json
import subprocess
from unittest.mock import patch, MagicMock

import pytest

from errors import AuthExpiredError, RateLimitedError, AskTimeoutError, NotebookLMError
from nlm_client import ask, AskResult, Reference


def _completed_process(stdout="", stderr="", returncode=0):
    proc = MagicMock(spec=subprocess.CompletedProcess)
    proc.stdout = stdout
    proc.stderr = stderr
    proc.returncode = returncode
    return proc


def test_ask_parses_successful_response():
    payload = {
        "answer": "The sky is blue because of Rayleigh scattering.",
        "references": [
            {"source_id": "src-1", "citation_number": 1, "cited_text": "Rayleigh scattering explanation"}
        ],
    }
    with patch("nlm_client.subprocess.run", return_value=_completed_process(stdout=json.dumps(payload))) as mock_run:
        result = ask("Why is the sky blue?", "nb-123")

    assert result == AskResult(
        answer="The sky is blue because of Rayleigh scattering.",
        references=[Reference(source_id="src-1", citation_number=1, cited_text="Rayleigh scattering explanation")],
    )
    args = mock_run.call_args.args[0]
    assert args == ["notebooklm", "ask", "Why is the sky blue?", "--json", "--notebook", "nb-123"]


def test_ask_parses_response_with_no_references():
    payload = {"answer": "No citations here.", "references": []}
    with patch("nlm_client.subprocess.run", return_value=_completed_process(stdout=json.dumps(payload))):
        result = ask("A question", "nb-123")

    assert result == AskResult(answer="No citations here.", references=[])


def test_ask_raises_auth_expired_on_login_stderr():
    with patch("nlm_client.subprocess.run", return_value=_completed_process(stderr="Auth error: please run notebooklm login", returncode=1)):
        with pytest.raises(AuthExpiredError):
            ask("A question", "nb-123")


def test_ask_raises_rate_limited_on_rpc_stderr():
    with patch("nlm_client.subprocess.run", return_value=_completed_process(stderr="No result found for RPC ID abc", returncode=1)):
        with pytest.raises(RateLimitedError):
            ask("A question", "nb-123")


def test_ask_raises_generic_error_on_unknown_stderr():
    with patch("nlm_client.subprocess.run", return_value=_completed_process(stderr="Invalid notebook ID", returncode=1)):
        with pytest.raises(NotebookLMError) as exc_info:
            ask("A question", "nb-123")
    assert "Invalid notebook ID" in str(exc_info.value)


def test_ask_raises_timeout_error_on_subprocess_timeout():
    with patch("nlm_client.subprocess.run", side_effect=subprocess.TimeoutExpired(cmd="notebooklm", timeout=30.0)):
        with pytest.raises(AskTimeoutError):
            ask("A question", "nb-123", timeout=30.0)
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `.venv/bin/pytest tests/test_nlm_client.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'nlm_client'`

- [ ] **Step 4: Write `nlm_client.py`**

```python
import json
import subprocess
from dataclasses import dataclass

from errors import AuthExpiredError, RateLimitedError, AskTimeoutError, NotebookLMError


@dataclass(frozen=True)
class Reference:
    source_id: str
    citation_number: int
    cited_text: str


@dataclass(frozen=True)
class AskResult:
    answer: str
    references: list[Reference]


def ask(question: str, notebook_id: str, timeout: float = 30.0) -> AskResult:
    command = ["notebooklm", "ask", question, "--json", "--notebook", notebook_id]
    try:
        proc = subprocess.run(command, capture_output=True, text=True, timeout=timeout)
    except subprocess.TimeoutExpired as exc:
        raise AskTimeoutError(f"notebooklm ask timed out after {timeout}s") from exc

    if proc.returncode != 0:
        stderr_lower = proc.stderr.lower()
        if "login" in stderr_lower or "auth" in stderr_lower or "cookie" in stderr_lower:
            raise AuthExpiredError(proc.stderr.strip())
        if "rate limit" in stderr_lower or "rpc id" in stderr_lower:
            raise RateLimitedError(proc.stderr.strip())
        raise NotebookLMError(proc.stderr.strip())

    payload = json.loads(proc.stdout)
    references = [
        Reference(
            source_id=ref["source_id"],
            citation_number=ref["citation_number"],
            cited_text=ref["cited_text"],
        )
        for ref in payload.get("references", [])
    ]
    return AskResult(answer=payload["answer"], references=references)
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `.venv/bin/pytest tests/test_nlm_client.py -v`
Expected: 6 passed

- [ ] **Step 6: Commit**

```bash
git add errors.py nlm_client.py tests/test_nlm_client.py
git commit -m "feat: add notebooklm CLI wrapper with typed errors"
```

---

### Task 4: Answer formatting

**Files:**
- Create: `notebooklm-discord-bot/formatting.py`
- Test: `notebooklm-discord-bot/tests/test_formatting.py`

**Interfaces:**
- Consumes: `nlm_client.Reference` (from Task 3).
- Produces:
  - `formatting.format_answer(answer: str, max_chars: int = 2000) -> list[str]` — splits
    `answer` into chunks no longer than `max_chars`, breaking on whitespace boundaries.
  - `formatting.format_sources(references: list[Reference], max_chars: int = 1024) -> str` —
    renders `"[n] cited_text"` lines joined by `\n`, truncated to `max_chars` (Discord
    embed field value limit) with a trailing `"…"` if truncated. Returns `""` for an empty list.
    Later tasks (bot.py) call both of these when building the Discord embed.

- [ ] **Step 1: Write the failing tests**

`tests/test_formatting.py`:
```python
from nlm_client import Reference
from formatting import format_answer, format_sources


def test_format_answer_returns_single_chunk_for_short_text():
    assert format_answer("Short answer.") == ["Short answer."]


def test_format_answer_splits_long_text_on_word_boundary():
    word = "word "
    long_answer = word * 500  # 2500 chars, well over the 2000 default
    chunks = format_answer(long_answer, max_chars=2000)

    assert len(chunks) == 2
    assert all(len(chunk) <= 2000 for chunk in chunks)
    # No word should be split across a chunk boundary
    assert not chunks[0].endswith("wor")
    # Chunks reconstruct the original words when rejoined with a space
    # (the space between the last word of one chunk and the first word of
    # the next is not stored in either chunk, so join with " " here rather
    # than "" -- unlike within a chunk, where spaces are preserved).
    rejoined = " ".join(chunks)
    assert rejoined.split() == long_answer.split()


def test_format_answer_respects_custom_max_chars():
    chunks = format_answer("one two three four five", max_chars=10)
    assert all(len(chunk) <= 10 for chunk in chunks)
    assert " ".join(chunks) == "one two three four five"


def test_format_sources_empty_list_returns_empty_string():
    assert format_sources([]) == ""


def test_format_sources_formats_each_reference_on_its_own_line():
    refs = [
        Reference(source_id="src-1", citation_number=1, cited_text="First citation"),
        Reference(source_id="src-2", citation_number=2, cited_text="Second citation"),
    ]
    assert format_sources(refs) == "[1] First citation\n[2] Second citation"


def test_format_sources_truncates_to_max_chars():
    refs = [Reference(source_id=f"src-{i}", citation_number=i, cited_text="x" * 50) for i in range(1, 40)]
    result = format_sources(refs, max_chars=100)

    assert len(result) == 100
    assert result.endswith("…")
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `.venv/bin/pytest tests/test_formatting.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'formatting'`

- [ ] **Step 3: Write `formatting.py`**

```python
from nlm_client import Reference


def format_answer(answer: str, max_chars: int = 2000) -> list[str]:
    if len(answer) <= max_chars:
        return [answer]

    words = answer.split(" ")
    chunks: list[str] = []
    current = ""
    for word in words:
        candidate = f"{current} {word}".strip() if current else word
        if len(candidate) > max_chars:
            if current:
                chunks.append(current)
            current = word
        else:
            current = candidate
    if current:
        chunks.append(current)
    return chunks


def format_sources(references: list[Reference], max_chars: int = 1024) -> str:
    if not references:
        return ""

    lines = [f"[{ref.citation_number}] {ref.cited_text}" for ref in references]
    full_text = "\n".join(lines)
    if len(full_text) <= max_chars:
        return full_text
    return full_text[: max_chars - 1] + "…"
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `.venv/bin/pytest tests/test_formatting.py -v`
Expected: 6 passed

- [ ] **Step 5: Commit**

```bash
git add formatting.py tests/test_formatting.py
git commit -m "feat: add answer/source chunking for discord embed limits"
```

---

### Task 5: Discord bot command wiring

**Files:**
- Create: `notebooklm-discord-bot/bot.py`
- Test: `notebooklm-discord-bot/tests/test_bot.py`

**Interfaces:**
- Consumes:
  - `config.load_config() -> Config` (Task 2)
  - `nlm_client.ask(question, notebook_id, timeout=30.0) -> AskResult`, and
    `errors.{AuthExpiredError, RateLimitedError, AskTimeoutError, NotebookLMError}` (Task 3)
  - `formatting.format_answer(answer) -> list[str]`, `formatting.format_sources(references) -> str` (Task 4)
- Produces: `bot.handle_ask(interaction, question, notebook_id) -> None` — the testable
  core of the `/ask` command, called by the `discord.py` slash-command handler registered
  in `bot.py`'s module-level `tree`. `bot.main()` is the process entrypoint (not unit
  tested; exercised in Task 8's manual end-to-end check).

- [ ] **Step 1: Write the failing tests**

`tests/test_bot.py`:
```python
from unittest.mock import AsyncMock, patch

import pytest

from errors import AuthExpiredError, RateLimitedError, AskTimeoutError, NotebookLMError
from nlm_client import AskResult, Reference
from bot import handle_ask


@pytest.mark.asyncio
async def test_handle_ask_sends_embed_on_success():
    interaction = AsyncMock()
    result = AskResult(
        answer="Paris is the capital of France.",
        references=[Reference(source_id="src-1", citation_number=1, cited_text="Capital city fact")],
    )
    with patch("bot.ask", return_value=result):
        await handle_ask(interaction, "What is the capital of France?", "nb-123")

    interaction.response.defer.assert_awaited_once()
    interaction.followup.send.assert_awaited_once()
    _, kwargs = interaction.followup.send.call_args
    embed = kwargs["embed"]
    assert "Paris is the capital of France." in embed.description
    assert any(field.name == "แหล่งอ้างอิง" for field in embed.fields)


@pytest.mark.asyncio
async def test_handle_ask_reports_auth_expired():
    interaction = AsyncMock()
    with patch("bot.ask", side_effect=AuthExpiredError("cookie expired")):
        await handle_ask(interaction, "A question", "nb-123")

    interaction.followup.send.assert_awaited_once_with("บอทขัดข้องชั่วคราว ลองใหม่ภายหลัง")


@pytest.mark.asyncio
async def test_handle_ask_reports_rate_limited():
    interaction = AsyncMock()
    with patch("bot.ask", side_effect=RateLimitedError("rpc id not found")):
        await handle_ask(interaction, "A question", "nb-123")

    interaction.followup.send.assert_awaited_once_with("ตอนนี้มีคนถามเยอะ ลองใหม่ใน 5 นาที")


@pytest.mark.asyncio
async def test_handle_ask_reports_timeout():
    interaction = AsyncMock()
    with patch("bot.ask", side_effect=AskTimeoutError("timed out")):
        await handle_ask(interaction, "A question", "nb-123")

    interaction.followup.send.assert_awaited_once_with("คิดนานผิดปกติ ลองใหม่")


@pytest.mark.asyncio
async def test_handle_ask_reports_generic_error():
    interaction = AsyncMock()
    with patch("bot.ask", side_effect=NotebookLMError("something broke")):
        await handle_ask(interaction, "A question", "nb-123")

    interaction.followup.send.assert_awaited_once_with("เกิดข้อผิดพลาด ลองใหม่ภายหลัง")
```

- [ ] **Step 2: Add `pytest-asyncio` config and run tests to verify they fail**

`pytest.ini`:
```ini
[pytest]
asyncio_mode = auto
```

Run: `.venv/bin/pytest tests/test_bot.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'bot'`

- [ ] **Step 3: Write `bot.py`**

```python
import logging

import discord
from discord import app_commands

from config import load_config
from errors import AuthExpiredError, RateLimitedError, AskTimeoutError, NotebookLMError
from formatting import format_answer, format_sources
from nlm_client import ask

logger = logging.getLogger("nlm_bot")

intents = discord.Intents.default()
client = discord.Client(intents=intents)
tree = app_commands.CommandTree(client)


async def handle_ask(interaction: discord.Interaction, question: str, notebook_id: str) -> None:
    await interaction.response.defer()
    try:
        result = ask(question, notebook_id)
    except AuthExpiredError:
        logger.exception("NotebookLM auth expired")
        await interaction.followup.send("บอทขัดข้องชั่วคราว ลองใหม่ภายหลัง")
        return
    except RateLimitedError:
        logger.warning("NotebookLM rate limited")
        await interaction.followup.send("ตอนนี้มีคนถามเยอะ ลองใหม่ใน 5 นาที")
        return
    except AskTimeoutError:
        logger.warning("NotebookLM call timed out")
        await interaction.followup.send("คิดนานผิดปกติ ลองใหม่")
        return
    except NotebookLMError:
        logger.exception("NotebookLM call failed")
        await interaction.followup.send("เกิดข้อผิดพลาด ลองใหม่ภายหลัง")
        return

    chunks = format_answer(result.answer)
    embed = discord.Embed(title=f"คำถาม: {question}", description=chunks[0], color=0x4285F4)
    for i, chunk in enumerate(chunks[1:], start=2):
        embed.add_field(name=f"(ต่อ {i})", value=chunk, inline=False)
    sources = format_sources(result.references)
    if sources:
        embed.add_field(name="แหล่งอ้างอิง", value=sources, inline=False)
    await interaction.followup.send(embed=embed)


@tree.command(name="ask", description="ถามคำถามจาก NotebookLM")
@app_commands.describe(question="คำถามของคุณ")
async def ask_command(interaction: discord.Interaction, question: str) -> None:
    config = load_config()
    await handle_ask(interaction, question, config.notebooklm_notebook_id)


@client.event
async def on_ready() -> None:
    await tree.sync()
    logger.info("Logged in as %s", client.user)


def main() -> None:
    logging.basicConfig(level=logging.INFO)
    config = load_config()
    client.run(config.discord_bot_token)


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `.venv/bin/pytest tests/test_bot.py -v`
Expected: 5 passed

- [ ] **Step 5: Run the full test suite**

Run: `.venv/bin/pytest -v`
Expected: all tests across `test_config.py`, `test_nlm_client.py`, `test_formatting.py`, `test_bot.py` pass

- [ ] **Step 6: Commit**

```bash
git add bot.py tests/test_bot.py pytest.ini
git commit -m "feat: wire discord /ask command to notebooklm client"
```

---

### Task 6: Docker packaging

**Files:**
- Create: `notebooklm-discord-bot/Dockerfile`
- Create: `notebooklm-discord-bot/docker-compose.yml`
- Modify: `notebooklm-discord-bot/README.md` (append Docker section)

**Interfaces:**
- Consumes: `requirements.txt` (Task 1), `bot.py` (Task 5), `.env` (user-provided, based on
  `.env.example` from Task 1).
- Produces: a runnable container image; no Python interfaces (this task has no unit tests —
  verification is manual per the steps below).

- [ ] **Step 1: Write `Dockerfile`**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt \
    && python -m playwright install --with-deps chromium

COPY config.py errors.py nlm_client.py formatting.py bot.py ./

CMD ["python", "bot.py"]
```

- [ ] **Step 2: Write `docker-compose.yml`**

```yaml
services:
  notebooklm-discord-bot:
    build: .
    env_file:
      - .env
    restart: unless-stopped
```

- [ ] **Step 3: Append Docker instructions to `README.md`**

Add this section to the end of `README.md`:
```markdown
## Docker deployment (local)

1. Ensure `.env` is filled in (see Setup above).
2. Build and run: `docker compose up -d --build`
3. Check logs: `docker compose logs -f`
4. Stop: `docker compose down`

The container restarts automatically (`restart: unless-stopped`) if it crashes or the
machine reboots, but only while the host machine itself is powered on and running Docker.
```

- [ ] **Step 4: Verify the image builds**

Run: `cd /Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot && docker compose build`
Expected: exits 0, image builds successfully (this step requires Docker Desktop running
and network access; Playwright's `--with-deps` step can take several minutes).

- [ ] **Step 5: Commit**

```bash
git add Dockerfile docker-compose.yml README.md
git commit -m "chore: add docker packaging for local deployment"
```

---

### Task 7: NotebookLM auth setup and refresh scheduling

**Files:**
- Create: `notebooklm-discord-bot/scripts/refresh_auth.sh`
- Create: `notebooklm-discord-bot/scripts/com.notebooklm.refresh.plist`
- Modify: `notebooklm-discord-bot/README.md` (append auth maintenance section)

**Interfaces:**
- Consumes: the `notebooklm` CLI installed on the host (via `pip install "notebooklm-py[browser]"`,
  already a dependency from Task 1).
- Produces: a scheduled job definition; no Python interfaces. This task documents and scripts
  an operational concern from the spec (`Auth Maintenance` section) — there is no automated
  test for a scheduled OS job, so verification is manual.

- [ ] **Step 1: Write `scripts/refresh_auth.sh`**

```bash
#!/usr/bin/env bash
set -euo pipefail

notebooklm auth refresh --quiet
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x /Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot/scripts/refresh_auth.sh
```

- [ ] **Step 3: Write the launchd template `scripts/com.notebooklm.refresh.plist`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.notebooklm.refresh</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot/scripts/refresh_auth.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>900</integer>
    <key>StandardOutPath</key>
    <string>/tmp/notebooklm-refresh.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/notebooklm-refresh.log</string>
</dict>
</plist>
```

- [ ] **Step 4: Append auth maintenance instructions to `README.md`**

Add this section to the end of `README.md`:
```markdown
## NotebookLM auth setup and maintenance

Initial login (run once, opens a browser):
```bash
notebooklm login
notebooklm auth check --test --json   # confirm status: ok, checks.token_fetch: true
```

Copy the resulting `~/.notebooklm/profiles/default/storage_state.json` contents into the
`NOTEBOOKLM_AUTH_JSON` variable in `.env` so the Docker container doesn't need an
interactive browser.

### Keeping the session alive (macOS)

Cookies rotate/expire, so a scheduled refresh keeps the session usable while the bot
runs unattended:

```bash
cp scripts/com.notebooklm.refresh.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.notebooklm.refresh.plist
```

This runs `notebooklm auth refresh --quiet` every 15 minutes. Check `/tmp/notebooklm-refresh.log`
if the bot starts reporting "บอทขัดข้องชั่วคราว" (auth-expired errors).

⚠️ If the machine sleeps, this scheduled job won't fire. Disable sleep while the bot needs
to be available, or expect to re-run `notebooklm login` manually after long idle periods.
```

- [ ] **Step 5: Verify auth check works locally (manual)**

Run: `notebooklm auth check --test --json`
Expected: JSON with `"status": "ok"` and `"checks": {..., "token_fetch": true}`. If this
fails, run `notebooklm login` first (see Troubleshooting in the notebooklm skill).

- [ ] **Step 6: Commit**

```bash
git add scripts/refresh_auth.sh scripts/com.notebooklm.refresh.plist README.md
git commit -m "chore: add notebooklm auth refresh scheduling for local deployment"
```

---

### Task 8: End-to-end verification

**Files:**
- Modify: `notebooklm-discord-bot/README.md` (append Discord app setup section)

**Interfaces:**
- Consumes: everything from Tasks 1–7 running together.
- Produces: nothing new — this is a manual checklist confirming the whole system works
  against a real Discord server and a real NotebookLM notebook with sources loaded.

- [ ] **Step 1: Append Discord application setup to `README.md`**

Add this section to the end of `README.md`:
```markdown
## Discord application setup

1. Create an application at https://discord.com/developers/applications
2. Bot tab → Reset Token → copy into `DISCORD_BOT_TOKEN` in `.env`
3. Bot tab → enable no privileged intents (this bot only uses slash commands)
4. OAuth2 → URL Generator → scopes: `bot`, `applications.commands` → permissions:
   `Send Messages`, `Embed Links` → open the generated URL to invite the bot to your
   test server
```

- [ ] **Step 2: Prepare a real NotebookLM notebook**

In the NotebookLM web UI (or via the `notebooklm` CLI), create or pick a notebook, add at
least one source, and confirm it's indexed:
```bash
notebooklm source list --notebook <NOTEBOOK_ID> --json
```
Expected: at least one source with `"status": "ready"`. Put this notebook's ID into
`NOTEBOOKLM_NOTEBOOK_ID` in `.env`.

- [ ] **Step 3: Run the bot locally (not in Docker, for faster iteration)**

Run: `cd /Users/mungmee/Desktop/Project/Python/notebooklm-discord-bot && set -a && source .env && set +a && .venv/bin/python bot.py`
Expected: log line `Logged in as <bot name>#<discriminator>`, no exceptions.

- [ ] **Step 4: Test the happy path in Discord**

In your test server, run `/ask` with a question answerable from the loaded source.
Expected: bot shows "thinking...", then replies with an embed containing the answer and
a "แหล่งอ้างอิง" (sources) field.

- [ ] **Step 5: Test the long-answer chunking path**

Ask a question likely to produce a long answer (e.g. "สรุปเนื้อหาทั้งหมดอย่างละเอียด").
Expected: embed has multiple `(ต่อ N)` fields, none exceeding Discord's field-value limit,
and no word is cut mid-way at a chunk boundary.

- [ ] **Step 6: Test the auth-expired path**

Temporarily corrupt `NOTEBOOKLM_AUTH_JSON` (or rename `~/.notebooklm/profiles/default/storage_state.json`)
and re-run `/ask`.
Expected: bot replies "บอทขัดข้องชั่วคราว ลองใหม่ภายหลัง"; restore the auth file/env var
afterward and confirm `notebooklm auth check --test --json` is healthy again.

- [ ] **Step 7: Run in Docker end-to-end**

Run: `docker compose up -d --build && docker compose logs -f`
Expected: same "Logged in as..." line in container logs; `/ask` in Discord works
identically to the local run in Step 4.

- [ ] **Step 8: Commit the final README updates**

```bash
git add README.md
git commit -m "docs: add discord app setup and end-to-end verification steps"
```
