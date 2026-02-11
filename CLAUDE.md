# JobHunter

## What
Automated job search pipeline that scrapes niche job boards, scores matches with LLM, auto-tailors resumes, and notifies via Telegram/email.

## Why
Job hunting across multiple countries (Netherlands, Germany, UAE) is tedious manual work — this automates the grunt work so you only see high-quality matches with tailored resumes ready to go.

## Stack
Python 3.11+, httpx, BeautifulSoup4, structlog, WeasyPrint, python-telegram-bot
Package manager: uv

### Key Library Choices
- **Config format**: YAML — human-readable, great for site scraper configs
- **HTTP client**: httpx — async support, modern API, HTTP/2
- **HTML parser**: BeautifulSoup4 + lxml — forgiving parser, most docs/examples
- **Logging**: structlog — structured JSON output, fits pipeline logging needs
- **Storage**: SQLite — zero setup, single file, sufficient for personal use
- **Telegram**: python-telegram-bot — most popular, async, feature-rich
- **Email**: smtplib — built-in, zero deps
- **HTML → PDF**: WeasyPrint — pure Python, good CSS support
- **LLM**: claude-max-api-proxy (primary), OpenRouter (fallback) — raw httpx calls, no SDK needed
- **Scheduler**: system cron

## Structure
- `src/jobhunter/` — source code (scraper, dedup, matcher, tailor, notifier, tracker, feedback, llm, checkpoint, db)
- `config/` — global settings, master resume YAML
- `site_configs/` — one YAML config per job board
- `templates/` — HTML resume template
- `tests/` — test files
- `docs/` — design doc, plans, research, handoffs, logs
- `data/` — raw scraped data, checkpoints, SQLite DB (gitignored)
- `output/` — generated resume PDFs (gitignored)
- `logs/` — structured pipeline logs (gitignored)

## Commands
- Test: `uv run pytest`
- Lint: `uv run ruff check .`
- Lint fix: `uv run ruff check --fix .`
- Format: `uv run ruff format .`
- Format check: `uv run ruff format --check .`
- Run pipeline: `uv run python -m jobhunter.main`

## Conventions
- Tests mirror source: `src/jobhunter/scraper.py` → `tests/test_scraper.py`
- Site configs are YAML files in `site_configs/` — no code changes needed to add a new site
- Each pipeline module is independent and testable in isolation
- All LLM calls go through `src/jobhunter/llm/` abstraction (handles proxy/fallback chain)
- Checkpoint after every critical pipeline stage — never lose scraped data
- Errors are isolated: one site/job failing never kills the whole run
- Structured logging everywhere (structlog with JSON output)

## Subagents
Always launch opus subagents unless specified otherwise.

## Workflow
- /draft for complex features → /specify → /clarify → /clear → /create_plan → /clear → /implement_plan
- /specify for clear features → /clarify → /clear → /create_plan → /clear → /implement_plan
- /validate_plan to check for drift
- /create_handoff before stopping, /resume_handoff to continue
- /log_error when things go wrong, /log_success when things go right

## Rules
- When running workflow commands (/specify, /create_plan, /implement_plan, etc.), follow ONLY the command instructions. Do not invoke additional skills or override the command's process.
- Always /clear between spec→plan and plan→implement phases. The files on disk are the handoffs.
- Always use Context7 MCP to look up library documentation for up-to-date information before implementing features.
