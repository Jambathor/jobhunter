# Plan: Job Search Pipeline V1
**Date:** 2026-02-12
**Status:** Draft
**Spec:** `docs/specs/0001-job-search-pipeline-v1.md`

## Overview

Build the complete JobHunter pipeline from scratch: scrape niche job boards via YAML-driven configs, deduplicate, filter by keywords, score with LLM, auto-tailor resumes to PDF, and notify via Telegram/email. The system is a sync batch pipeline run via cron, with crash recovery via checkpoints.

## Current State

- `pyproject.toml:1-31` — project config with deps (httpx, bs4, lxml, structlog, weasyprint, python-telegram-bot, pyyaml)
- `main.py:1-6` — stub "Hello from jobhunter!"
- No `src/`, `config/`, `site_configs/`, `templates/`, `tests/`, `data/`, `output/`, `logs/` directories exist
- No source code, no database, no configs

## What We're NOT Doing

- LinkedIn, Indeed, or any site requiring anti-bot evasion
- Multi-user support, authentication, or web UI
- Cover letter generation
- Salary parsing or currency conversion
- Auto-apply functionality
- Automatic retraining from feedback (V1 collects data only)
- Anti-bot evasion or CAPTCHA solving
- Multi-language resume formats
- Company intelligence layer (Glassdoor, visa sponsor lists)
- LLM proxy lifecycle management

## Architecture Decisions

- **Sync pipeline** — batch job, not a server. Uses `concurrent.futures.ThreadPoolExecutor` for parallel site scraping. `asyncio.run()` wraps the few async calls (Telegram bot API).
- **Raw SQL with sqlite3** — no ORM. DB is simple enough. WAL mode for safe reads.
- **Dataclasses for models** — no Pydantic dependency. Manual validation where needed.
- **httpx sync client** — for all HTTP (scraping + LLM calls).
- **Playwright sync API** — for browser scraping strategy.
- **python-telegram-bot** — async API wrapped in `asyncio.run()` for sending messages. Callbacks polled via `getUpdates` during pipeline runs.
- **pathlib.Path** everywhere for file paths.
- **`from __future__ import annotations`** in all files for clean type hints.

## Directory Structure

```
src/jobhunter/
├── __init__.py
├── __main__.py
├── models.py
├── config.py
├── db.py
├── log.py
├── dedup.py
├── checkpoint.py
├── pipeline.py
├── tracker.py
├── feedback.py
├── llm/
│   ├── __init__.py
│   └── client.py
├── scraper/
│   ├── __init__.py
│   ├── engine.py
│   ├── strategies.py
│   └── normalizer.py
├── matcher/
│   ├── __init__.py
│   ├── keyword_filter.py
│   └── scorer.py
├── tailor/
│   ├── __init__.py
│   ├── engine.py
│   ├── verifier.py
│   └── pdf.py
└── notifier/
    ├── __init__.py
    ├── telegram.py
    └── email.py
```

---

## Implementation

### Phase 1: Foundation

Sets up the project structure, data models, configuration loading, database, and structured logging. Everything else depends on this phase.

**Implements:** FR-001, FR-003, FR-004, FR-006 (config schema), FR-016 (DB schema), FR-023, FR-024, FR-029 (DB schema), NFR-003, NFR-004, NFR-005

#### Changes Required

**File:** `pyproject.toml`
**What:** Add build system, package discovery, playwright dep, dev deps

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/jobhunter"]

[project]
name = "jobhunter"
version = "0.1.0"
description = "Automated job search pipeline: scrape niche boards, score matches, tailor resumes, notify via Telegram"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "beautifulsoup4>=4.14.3",
    "httpx>=0.28.1",
    "lxml>=6.0.2",
    "playwright>=1.52.0",
    "python-telegram-bot>=22.6",
    "pyyaml>=6.0.3",
    "structlog>=25.5.0",
    "weasyprint>=68.1",
]

[dependency-groups]
dev = [
    "pytest>=9.0.2",
    "ruff>=0.15.0",
]

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP"]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

**File:** `src/jobhunter/__init__.py`
**What:** Package init, empty

```python
```

**File:** `src/jobhunter/__main__.py`
**What:** Entry point for `python -m jobhunter`

```python
from __future__ import annotations

from jobhunter.main import main

if __name__ == "__main__":
    main()
```

**File:** `src/jobhunter/models.py`
**What:** All data models as dataclasses. This is the core schema — every other module imports from here.

```python
from __future__ import annotations

import hashlib
import re
import uuid
from dataclasses import dataclass, field
from datetime import datetime


def _normalize_for_hash(text: str) -> str:
    """Lowercase, strip whitespace, remove punctuation."""
    text = text.lower().strip()
    text = re.sub(r"[^\w\s]", "", text)
    text = re.sub(r"\s+", " ", text)
    return text


def generate_job_id(title: str, company: str, location: str) -> str:
    """SHA-256 hash of normalized title + company + location."""
    normalized = _normalize_for_hash(title) + _normalize_for_hash(company) + _normalize_for_hash(location)
    return hashlib.sha256(normalized.encode()).hexdigest()


def generate_run_id() -> str:
    """Generate a unique pipeline run ID."""
    return uuid.uuid4().hex[:12]


@dataclass
class Job:
    id: str
    site_id: str
    title: str
    company: str
    location: str
    country: str
    url: str
    salary: str | None
    description: str | None
    requirements: str | None
    posted_date: str | None
    scraped_at: str
    run_id: str


@dataclass
class ScoredJob:
    job: Job
    score: int
    reasoning: str
    provider: str  # which LLM provider scored it
    scored_at: str


@dataclass
class TailoredResume:
    job_id: str
    html_path: str
    pdf_path: str
    verified: bool
    verification_issues: list[str]
    generated_at: str
    run_id: str


@dataclass
class Application:
    id: str
    job_id: str
    company: str
    role: str
    country: str
    applied_date: str | None
    resume_version: str | None
    status: str  # matched, applied, phone_screen, interview, offer, rejected, withdrawn, expired
    status_updated: str
    notes: str | None
    source_site: str


@dataclass
class Feedback:
    job_id: str
    score: int
    action: str  # applied, skipped, not_relevant
    reason: str | None
    timestamp: str


@dataclass
class SiteConfig:
    site_id: str
    name: str
    url: str
    country: str
    enabled: bool
    strategy: str  # "api", "html", "browser"
    max_pages: int

    # Strategy-specific config (only one populated based on strategy)
    api: dict | None = None
    html: dict | None = None
    browser: dict | None = None

    # Detail page config
    detail_page: dict | None = None

    # Per-site keyword overrides
    keywords: dict | None = None


@dataclass
class KeywordConfig:
    must_have_any: list[str] = field(default_factory=list)
    must_not_have: list[str] = field(default_factory=list)
    title_must_have_any: list[str] = field(default_factory=list)


@dataclass
class LLMProviderConfig:
    base_url: str
    model: str
    api_key: str | None = None


@dataclass
class NotificationThresholds:
    instant_telegram: int = 80
    digest_email: int = 60
    log_only: int = 40


@dataclass
class Settings:
    scoring_threshold: int
    scoring_criteria: dict
    keywords: KeywordConfig
    notifications: NotificationThresholds
    llm_primary: LLMProviderConfig
    llm_fallback: LLMProviderConfig
    llm_timeout: int
    llm_max_retries: int
    max_description_tokens: int
    telegram_bot_token: str | None
    telegram_chat_id: str | None
    email_smtp_host: str | None
    email_smtp_port: int
    email_from: str | None
    email_to: str | None
    email_password: str | None
    data_dir: str
    output_dir: str
    log_dir: str


@dataclass
class PipelineRun:
    run_id: str
    started_at: str
    completed_at: str | None = None
    status: str = "running"
    sites_attempted: list[str] = field(default_factory=list)
    sites_succeeded: list[str] = field(default_factory=list)
    sites_failed: list[dict] = field(default_factory=list)
    jobs_scraped: int = 0
    jobs_new: int = 0
    jobs_filtered_out: int = 0
    jobs_scored: int = 0
    jobs_above_threshold: int = 0
    resumes_generated: int = 0
    notifications_sent: int = 0
    errors: list[dict] = field(default_factory=list)
    llm_providers_used: dict = field(default_factory=dict)
```

**File:** `src/jobhunter/config.py`
**What:** Load settings.yaml, site configs, and master resume. Merges env vars for secrets.

```python
from __future__ import annotations

import os
from pathlib import Path

import yaml

from jobhunter.models import (
    KeywordConfig,
    LLMProviderConfig,
    NotificationThresholds,
    Settings,
    SiteConfig,
)

PROJECT_ROOT = Path(__file__).parent.parent.parent
CONFIG_DIR = PROJECT_ROOT / "config"
SITE_CONFIGS_DIR = PROJECT_ROOT / "site_configs"


def load_settings(config_path: Path | None = None) -> Settings:
    """Load settings from YAML + environment variables.

    Environment variables override YAML values for secrets:
    - JOBHUNTER_TELEGRAM_BOT_TOKEN
    - JOBHUNTER_TELEGRAM_CHAT_ID
    - JOBHUNTER_EMAIL_PASSWORD
    - JOBHUNTER_OPENROUTER_API_KEY
    - JOBHUNTER_LLM_PRIMARY_URL (optional override)
    """
    path = config_path or CONFIG_DIR / "settings.yaml"
    with open(path) as f:
        raw = yaml.safe_load(f)

    scoring = raw.get("scoring", {})
    kw = raw.get("keywords", {}).get("global", {})
    notif = raw.get("notifications", {})
    llm = raw.get("llm", {})
    telegram = notif.get("telegram", {})
    email_cfg = notif.get("email", {})
    paths = raw.get("paths", {})

    return Settings(
        scoring_threshold=scoring.get("threshold", 60),
        scoring_criteria=scoring.get("criteria", {}),
        keywords=KeywordConfig(
            must_have_any=kw.get("must_have_any", []),
            must_not_have=kw.get("must_not_have", []),
            title_must_have_any=kw.get("title_must_have_any", []),
        ),
        notifications=NotificationThresholds(
            instant_telegram=telegram.get("instant_threshold", 80),
            digest_email=email_cfg.get("digest_threshold", 60),
            log_only=notif.get("log_threshold", 40),
        ),
        llm_primary=LLMProviderConfig(
            base_url=os.environ.get(
                "JOBHUNTER_LLM_PRIMARY_URL",
                llm.get("primary", {}).get("base_url", "http://localhost:3456/v1"),
            ),
            model=llm.get("primary", {}).get("model", "claude-sonnet-4-5-20250929"),
        ),
        llm_fallback=LLMProviderConfig(
            base_url=llm.get("fallback", {}).get("base_url", "https://openrouter.ai/api/v1"),
            model=llm.get("fallback", {}).get("model", "anthropic/claude-3.5-sonnet"),
            api_key=os.environ.get("JOBHUNTER_OPENROUTER_API_KEY"),
        ),
        llm_timeout=llm.get("timeout", 60),
        llm_max_retries=llm.get("max_retries", 1),
        max_description_tokens=raw.get("pipeline", {}).get("max_description_tokens", 4000),
        telegram_bot_token=os.environ.get("JOBHUNTER_TELEGRAM_BOT_TOKEN"),
        telegram_chat_id=os.environ.get("JOBHUNTER_TELEGRAM_CHAT_ID"),
        email_smtp_host=email_cfg.get("smtp_host"),
        email_smtp_port=email_cfg.get("smtp_port", 587),
        email_from=email_cfg.get("from"),
        email_to=email_cfg.get("to"),
        email_password=os.environ.get("JOBHUNTER_EMAIL_PASSWORD"),
        data_dir=paths.get("data_dir", "data"),
        output_dir=paths.get("output_dir", "output"),
        log_dir=paths.get("log_dir", "logs"),
    )


def load_site_configs(configs_dir: Path | None = None) -> list[SiteConfig]:
    """Load all YAML site configs from directory. Only returns enabled configs."""
    directory = configs_dir or SITE_CONFIGS_DIR
    configs = []
    for path in sorted(directory.glob("*.yaml")):
        if path.name.startswith("_"):
            continue  # skip templates
        with open(path) as f:
            raw = yaml.safe_load(f)
        if not raw.get("enabled", False):
            continue
        config = SiteConfig(
            site_id=raw["site_id"],
            name=raw["name"],
            url=raw["url"],
            country=raw["country"],
            enabled=raw["enabled"],
            strategy=raw["strategy"],
            max_pages=raw.get("max_pages", 3),
            api=raw.get("api"),
            html=raw.get("html"),
            browser=raw.get("browser"),
            detail_page=raw.get("detail_page"),
            keywords=raw.get("keywords"),
        )
        configs.append(config)
    return configs


def load_master_resume(path: Path | None = None) -> dict:
    """Load master resume YAML. Returns raw dict — structure validated by consumers."""
    resume_path = path or CONFIG_DIR / "master_resume.yaml"
    if not resume_path.exists():
        raise FileNotFoundError(f"Master resume not found at {resume_path}")
    with open(resume_path) as f:
        return yaml.safe_load(f)


def merge_keywords(global_kw: KeywordConfig, site_kw: dict | None) -> KeywordConfig:
    """Merge global keyword config with per-site overrides.

    Per-site keywords extend global lists. Use 'override: true' in site config
    to replace instead of extend.
    """
    if not site_kw:
        return global_kw

    if site_kw.get("override", False):
        return KeywordConfig(
            must_have_any=site_kw.get("must_have_any", []),
            must_not_have=site_kw.get("must_not_have", []),
            title_must_have_any=site_kw.get("title_must_have_any", []),
        )

    return KeywordConfig(
        must_have_any=list(set(global_kw.must_have_any + site_kw.get("must_have_any", []))),
        must_not_have=list(set(global_kw.must_not_have + site_kw.get("must_not_have", []))),
        title_must_have_any=list(
            set(global_kw.title_must_have_any + site_kw.get("title_must_have_any", []))
        ),
    )
```

**File:** `src/jobhunter/db.py`
**What:** SQLite database with all tables. WAL mode. All CRUD operations as methods on a Database class.

```python
from __future__ import annotations

import json
import sqlite3
from pathlib import Path

from jobhunter.models import Application, Feedback, Job, PipelineRun, ScoredJob, TailoredResume

SCHEMA = """
CREATE TABLE IF NOT EXISTS jobs (
    id TEXT PRIMARY KEY,
    site_id TEXT NOT NULL,
    title TEXT NOT NULL,
    company TEXT NOT NULL,
    location TEXT NOT NULL,
    country TEXT NOT NULL,
    url TEXT NOT NULL,
    salary TEXT,
    description TEXT,
    requirements TEXT,
    posted_date TEXT,
    scraped_at TEXT NOT NULL,
    run_id TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS seen_jobs (
    hash TEXT PRIMARY KEY,
    first_seen_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS scores (
    job_id TEXT PRIMARY KEY REFERENCES jobs(id),
    score INTEGER NOT NULL,
    reasoning TEXT NOT NULL,
    provider TEXT NOT NULL,
    scored_at TEXT NOT NULL,
    run_id TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS resumes (
    job_id TEXT PRIMARY KEY REFERENCES jobs(id),
    html_path TEXT NOT NULL,
    pdf_path TEXT NOT NULL,
    verified INTEGER NOT NULL DEFAULT 0,
    verification_issues TEXT,
    generated_at TEXT NOT NULL,
    run_id TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS applications (
    id TEXT PRIMARY KEY,
    job_id TEXT NOT NULL REFERENCES jobs(id),
    company TEXT NOT NULL,
    role TEXT NOT NULL,
    country TEXT NOT NULL,
    applied_date TEXT,
    resume_version TEXT,
    status TEXT NOT NULL DEFAULT 'matched',
    status_updated TEXT NOT NULL,
    notes TEXT,
    source_site TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS feedback (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    job_id TEXT NOT NULL REFERENCES jobs(id),
    score INTEGER NOT NULL,
    action TEXT NOT NULL,
    reason TEXT,
    timestamp TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS pipeline_runs (
    run_id TEXT PRIMARY KEY,
    started_at TEXT NOT NULL,
    completed_at TEXT,
    status TEXT NOT NULL DEFAULT 'running',
    sites_attempted TEXT,
    sites_succeeded TEXT,
    sites_failed TEXT,
    jobs_scraped INTEGER DEFAULT 0,
    jobs_new INTEGER DEFAULT 0,
    jobs_filtered_out INTEGER DEFAULT 0,
    jobs_scored INTEGER DEFAULT 0,
    jobs_above_threshold INTEGER DEFAULT 0,
    resumes_generated INTEGER DEFAULT 0,
    notifications_sent INTEGER DEFAULT 0,
    errors TEXT,
    llm_providers_used TEXT
);

CREATE TABLE IF NOT EXISTS notifications (
    job_id TEXT PRIMARY KEY REFERENCES jobs(id),
    telegram_sent INTEGER DEFAULT 0,
    email_sent INTEGER DEFAULT 0,
    telegram_message_id TEXT,
    sent_at TEXT,
    run_id TEXT NOT NULL
);
"""


class Database:
    def __init__(self, db_path: Path):
        self.db_path = db_path
        db_path.parent.mkdir(parents=True, exist_ok=True)
        self.conn = sqlite3.connect(str(db_path))
        self.conn.execute("PRAGMA journal_mode=WAL")
        self.conn.execute("PRAGMA foreign_keys=ON")
        self.conn.row_factory = sqlite3.Row
        self._init_schema()

    def _init_schema(self):
        self.conn.executescript(SCHEMA)
        self.conn.commit()

    def close(self):
        self.conn.close()

    # --- Jobs ---
    def insert_job(self, job: Job) -> None: ...
    def get_job(self, job_id: str) -> Job | None: ...
    def get_jobs_by_run(self, run_id: str) -> list[Job]: ...

    # --- Dedup ---
    def has_seen_job(self, hash: str) -> bool: ...
    def mark_job_seen(self, hash: str, timestamp: str) -> None: ...

    # --- Scores ---
    def insert_score(self, job_id: str, score: int, reasoning: str, provider: str, scored_at: str, run_id: str) -> None: ...
    def get_score(self, job_id: str) -> dict | None: ...
    def get_unscored_jobs(self, run_id: str) -> list[Job]: ...

    # --- Resumes ---
    def insert_resume(self, resume: TailoredResume) -> None: ...
    def get_resume(self, job_id: str) -> TailoredResume | None: ...

    # --- Applications ---
    def insert_application(self, app: Application) -> None: ...
    def get_applications_by_company(self, company: str) -> list[Application]: ...
    def update_application_status(self, app_id: str, status: str, updated: str) -> None: ...

    # --- Feedback ---
    def insert_feedback(self, fb: Feedback) -> None: ...

    # --- Pipeline Runs ---
    def insert_run(self, run: PipelineRun) -> None: ...
    def update_run(self, run: PipelineRun) -> None: ...
    def get_last_run(self) -> PipelineRun | None: ...
    def get_incomplete_run(self) -> PipelineRun | None: ...

    # --- Notifications ---
    def mark_notified(self, job_id: str, telegram_sent: bool, email_sent: bool, message_id: str | None, sent_at: str, run_id: str) -> None: ...
    def is_notified(self, job_id: str) -> bool: ...
    def get_unnotified_scored_jobs(self, run_id: str, min_score: int) -> list[dict]: ...
```

The `...` method bodies should be implemented with standard sqlite3 INSERT/SELECT/UPDATE queries. Each method follows the same pattern:
- INSERT: `self.conn.execute("INSERT INTO ... VALUES (...)", (...))` + `self.conn.commit()`
- SELECT: `self.conn.execute("SELECT ... FROM ... WHERE ...", (...)).fetchone()` → convert Row to dataclass
- UPDATE: `self.conn.execute("UPDATE ... SET ... WHERE ...", (...))` + `self.conn.commit()`

For JSON columns (sites_attempted, errors, etc.), use `json.dumps()` on write and `json.loads()` on read.

**File:** `src/jobhunter/log.py`
**What:** structlog configuration for JSON structured logging

```python
from __future__ import annotations

import sys
from pathlib import Path

import structlog


def setup_logging(log_dir: Path | None = None, json_output: bool = True) -> None:
    """Configure structlog for the pipeline.

    Args:
        log_dir: Directory for log files. If None, logs to stdout only.
        json_output: If True, output JSON. If False, pretty console output (for dev).
    """
    processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
    ]

    if json_output:
        processors.append(structlog.processors.JSONRenderer())
    else:
        processors.append(structlog.dev.ConsoleRenderer())

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(0),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(file=sys.stderr),
        cache_logger_on_first_use=True,
    )

    if log_dir:
        log_dir.mkdir(parents=True, exist_ok=True)


def get_logger(name: str) -> structlog.stdlib.BoundLogger:
    """Get a named logger instance."""
    return structlog.get_logger(name)
```

**File:** `config/settings.yaml`
**What:** Default settings with Cloud/API/Platform Engineering keywords

```yaml
scoring:
  threshold: 60
  criteria:
    technical_match_weight: 30
    seniority_match_weight: 20
    role_relevance_weight: 20
    visa_feasibility_weight: 15
    growth_potential_weight: 15
  preferences:
    target_roles:
      - Cloud Engineer
      - API Engineer
      - Platform Engineer
      - Solutions Architect
      - Integration Engineer
    target_countries:
      - Netherlands
      - Germany
      - UAE
    visa_needs: "Requires work visa sponsorship"
    min_seniority: "Mid-Senior"

keywords:
  global:
    must_have_any:
      - cloud
      - api
      - apigee
      - gcp
      - aws
      - azure
      - kubernetes
      - platform
      - infrastructure
      - integration
      - microservices
    must_not_have:
      - intern
      - internship
      - junior
      - student
      - entry-level
      - traineeship
      - werkstudent
      - apprentice
    title_must_have_any:
      - engineer
      - architect
      - lead
      - senior
      - principal
      - manager
      - specialist
      - consultant
      - developer

notifications:
  telegram:
    instant_threshold: 80
  email:
    digest_threshold: 60
    smtp_host: smtp.gmail.com
    smtp_port: 587
    from: null
    to: null
  log_threshold: 40

llm:
  primary:
    base_url: "http://localhost:3456/v1"
    model: "claude-sonnet-4-5-20250929"
  fallback:
    base_url: "https://openrouter.ai/api/v1"
    model: "anthropic/claude-3.5-sonnet"
  timeout: 60
  max_retries: 1

pipeline:
  max_description_tokens: 4000

paths:
  data_dir: "data"
  output_dir: "output"
  log_dir: "logs"
```

**File:** `config/master_resume.yaml`
**What:** Template master resume (user fills in their own data)

```yaml
# Master Resume — Fill in your actual data
# This is the source of truth for all resume tailoring.
# Include EVERYTHING — the LLM will select the most relevant items per job.

personal:
  name: "Your Name"
  email: "you@example.com"
  phone: "+1234567890"
  linkedin: "https://linkedin.com/in/yourprofile"
  location: "City, Country"
  languages:
    - language: English
      level: Native
    - language: Dutch
      level: Basic

summary:
  default: "Experienced engineer with X+ years in cloud and API platforms..."
  variants:
    cloud: "Cloud-focused engineer specializing in..."
    api: "API platform specialist with deep expertise in..."
    platform: "Platform engineer building developer experiences and tooling..."

experience:
  - company: "Company A"
    role: "Senior Cloud Engineer"
    location: "Amsterdam, NL"
    start_date: "2022-01"
    end_date: null
    achievements:
      - text: "Describe an achievement with metrics"
        tags: [cloud, gcp, api]
      - text: "Another achievement"
        tags: [kubernetes, infrastructure]

certifications:
  - name: "Google Cloud Professional Cloud Architect"
    date: "2023-06"
    tags: [cloud, gcp]

education:
  - institution: "University"
    degree: "BSc Computer Science"
    date: "2015"

skills:
  cloud: [GCP, AWS, Azure, Terraform, Kubernetes]
  api: [Apigee X, Kong, REST, GraphQL, gRPC]
  devops: [Docker, CI/CD, GitHub Actions, Jenkins]
  languages: [Python, Go, Java, TypeScript]
  databases: [PostgreSQL, Redis, BigQuery, Firestore]
```

**File:** `site_configs/_template.yaml`
**What:** Blank template for adding new sites

```yaml
# Site Config Template
# Copy this file and fill in the values for your target site.
# File name: {site_id}.yaml

site_id: ""           # unique identifier (e.g., "bayt", "gulftalent")
name: ""              # display name (e.g., "Bayt.com")
url: ""               # base URL (e.g., "https://www.bayt.com")
country: ""           # "UAE", "Netherlands", or "Germany"
enabled: false
strategy: "html"      # "api", "html", or "browser"
max_pages: 3          # how many listing pages to scrape

# --- HTML Strategy Config ---
# html:
#   list_url: "https://example.com/jobs?page={page}"
#   job_card_selector: ".job-card"
#   fields:
#     title:
#       selector: "h2 a"
#       attribute: "text"    # "text", "href", or any HTML attribute
#     company:
#       selector: ".company-name"
#       attribute: "text"
#     location:
#       selector: ".location"
#       attribute: "text"
#     url:
#       selector: "h2 a"
#       attribute: "href"
#       prefix: ""           # prepend to relative URLs
#     salary:
#       selector: ".salary"
#       attribute: "text"
#       optional: true
#     date:
#       selector: ".date"
#       attribute: "text"
#       optional: true
#   pagination:
#     method: "url_param"    # "url_param" or "next_button"
#     param: "page"
#     start: 1

# --- API Strategy Config ---
# api:
#   url: "https://api.example.com/jobs"
#   method: "GET"
#   params:
#     q: "cloud engineer"
#     location: "UAE"
#     page: "{page}"
#   headers: {}
#   response_path: "data.jobs"     # dot-notation path to job array in response
#   fields:
#     title: "title"               # JSON key for each field
#     company: "company.name"
#     location: "location"
#     url: "url"
#     salary: "salary"
#     date: "posted_at"

# --- Browser Strategy Config ---
# browser:
#   url: "https://example.com/jobs"
#   wait_for: ".job-list"
#   scroll: true
#   # Then same field selectors as html strategy
#   job_card_selector: ".job-card"
#   fields: { ... }

# --- Detail Page (optional) ---
# detail_page:
#   enabled: true
#   description_selector: ".job-description"
#   requirements_selector: ".job-requirements"

# --- Per-site keyword overrides (merged with global) ---
# keywords:
#   must_have_any:
#     - extra_keyword
#   must_not_have:
#     - extra_exclusion
#   # Set override: true to REPLACE global keywords instead of extending
#   # override: true
```

**File:** `.env.example`
**What:** Document all required/optional env vars

```
# Required for Telegram notifications
JOBHUNTER_TELEGRAM_BOT_TOKEN=
JOBHUNTER_TELEGRAM_CHAT_ID=

# Required for email notifications
JOBHUNTER_EMAIL_PASSWORD=

# Required for OpenRouter fallback
JOBHUNTER_OPENROUTER_API_KEY=

# Optional: override LLM proxy URL
# JOBHUNTER_LLM_PRIMARY_URL=http://localhost:3456/v1
```

**File:** `tests/conftest.py`
**What:** Shared test fixtures

```python
from __future__ import annotations

import tempfile
from pathlib import Path

import pytest

from jobhunter.db import Database
from jobhunter.models import Job


@pytest.fixture
def tmp_dir(tmp_path):
    """Provide a temporary directory for tests."""
    return tmp_path


@pytest.fixture
def db(tmp_path):
    """Provide a fresh test database."""
    db = Database(tmp_path / "test.db")
    yield db
    db.close()


@pytest.fixture
def sample_job():
    """A sample Job for testing."""
    return Job(
        id="abc123hash",
        site_id="testsite",
        title="Senior Cloud Engineer",
        company="TestCorp",
        location="Amsterdam, Netherlands",
        country="Netherlands",
        url="https://example.com/job/123",
        salary="EUR 80,000 - 100,000",
        description="We are looking for a senior cloud engineer...",
        requirements="5+ years GCP experience...",
        posted_date="2026-02-10",
        scraped_at="2026-02-12T10:00:00",
        run_id="run123",
    )
```

**File:** `tests/test_models.py`
**What:** Test model creation and hash generation

```python
from jobhunter.models import generate_job_id, _normalize_for_hash

def test_normalize_for_hash():
    assert _normalize_for_hash("  Senior Cloud Engineer!! ") == "senior cloud engineer"
    assert _normalize_for_hash("TestCorp, Inc.") == "testcorp inc"

def test_generate_job_id_deterministic():
    id1 = generate_job_id("Senior Cloud Engineer", "TestCorp", "Amsterdam")
    id2 = generate_job_id("Senior Cloud Engineer", "TestCorp", "Amsterdam")
    assert id1 == id2

def test_generate_job_id_case_insensitive():
    id1 = generate_job_id("Senior Cloud Engineer", "TestCorp", "Amsterdam")
    id2 = generate_job_id("senior cloud engineer", "testcorp", "amsterdam")
    assert id1 == id2

def test_generate_job_id_different_inputs():
    id1 = generate_job_id("Senior Cloud Engineer", "TestCorp", "Amsterdam")
    id2 = generate_job_id("Junior Cloud Engineer", "TestCorp", "Amsterdam")
    assert id1 != id2
```

**File:** `tests/test_config.py`
**What:** Test config loading (write temp YAML files, load them, verify)

Test cases:
- Load settings.yaml with all fields
- Load settings.yaml with env var overrides
- Load site configs from directory (skip disabled, skip templates)
- Load master resume
- Merge keywords (extend mode)
- Merge keywords (override mode)
- Missing master resume raises FileNotFoundError

**File:** `tests/test_db.py`
**What:** Test all Database CRUD operations

Test cases:
- Insert and retrieve a job
- Dedup: mark seen, check seen
- Insert and retrieve scores
- Insert and retrieve applications
- Get applications by company
- Insert and retrieve feedback
- Pipeline run insert/update/get
- Notification tracking

#### Success Criteria

**Automated (run by agent):**
- [ ] `uv sync` installs all dependencies without error
- [ ] `uv run pytest tests/test_models.py tests/test_config.py tests/test_db.py -v` — all tests pass
- [ ] `uv run ruff check src/jobhunter/` — no lint errors
- [ ] `uv run ruff format --check src/jobhunter/` — no format errors
- [ ] `uv run python -c "from jobhunter.models import Job, Settings; print('OK')"` — imports work

**Manual (verified by human):**
- [ ] `config/settings.yaml` contains sensible defaults
- [ ] `config/master_resume.yaml` is a clear template
- [ ] `.env.example` documents all env vars

#### Implementation Note
Complete this phase and verify ALL automated criteria before proceeding.
The Database class `...` method stubs must be fully implemented with actual SQL queries.

---

### Phase 2: LLM Client

LLM abstraction layer with fallback chain (proxy → OpenRouter → skip). Handles retries, JSON parsing, and provider switching.

**Implements:** FR-021, FR-022, US-14

#### Changes Required

**File:** `src/jobhunter/llm/__init__.py`
**What:** Empty init

**File:** `src/jobhunter/llm/client.py`
**What:** LLMClient with fallback chain

```python
from __future__ import annotations

import json
import time

import httpx
import structlog

from jobhunter.models import LLMProviderConfig

logger = structlog.get_logger("llm")


class LLMError(Exception):
    """Raised when all LLM providers fail."""
    pass


class LLMClient:
    """LLM client with fallback chain.

    Tries primary provider first, falls back to secondary on failure.
    Handles JSON parsing, retries for malformed responses, and timeouts.
    """

    def __init__(
        self,
        primary: LLMProviderConfig,
        fallback: LLMProviderConfig,
        timeout: int = 60,
        max_retries: int = 1,
    ):
        self.providers = [primary, fallback]
        self.timeout = timeout
        self.max_retries = max_retries
        self.last_provider_used: str | None = None

    def chat(
        self,
        messages: list[dict],
        json_response: bool = False,
    ) -> dict:
        """Send a chat completion request. Tries providers in order.

        Args:
            messages: List of {"role": "...", "content": "..."} dicts.
            json_response: If True, parse response as JSON and retry on parse failure.

        Returns:
            {"content": str, "provider": str} or
            {"content": dict, "provider": str} if json_response=True

        Raises:
            LLMError: If all providers fail.
        """
        errors = []
        for provider in self.providers:
            try:
                result = self._call_provider(provider, messages, json_response)
                self.last_provider_used = provider.base_url
                return result
            except Exception as e:
                logger.warning(
                    "llm_provider_failed",
                    provider=provider.base_url,
                    error=str(e),
                )
                errors.append((provider.base_url, str(e)))

        raise LLMError(f"All LLM providers failed: {errors}")

    def _call_provider(
        self,
        provider: LLMProviderConfig,
        messages: list[dict],
        json_response: bool,
    ) -> dict:
        """Call a single provider with retry logic for malformed JSON."""
        headers = {"Content-Type": "application/json"}
        if provider.api_key:
            headers["Authorization"] = f"Bearer {provider.api_key}"

        payload = {
            "model": provider.model,
            "messages": messages,
        }
        if json_response:
            payload["response_format"] = {"type": "json_object"}

        for attempt in range(1 + self.max_retries):
            with httpx.Client(timeout=self.timeout) as client:
                response = client.post(
                    f"{provider.base_url}/chat/completions",
                    headers=headers,
                    json=payload,
                )
                response.raise_for_status()

            data = response.json()
            content = data["choices"][0]["message"]["content"]

            if not json_response:
                return {"content": content, "provider": provider.base_url}

            # Parse JSON response
            try:
                parsed = json.loads(content)
                return {"content": parsed, "provider": provider.base_url}
            except json.JSONDecodeError:
                if attempt < self.max_retries:
                    logger.warning(
                        "malformed_json_response",
                        provider=provider.base_url,
                        attempt=attempt + 1,
                        raw=content[:200],
                    )
                    # Add stricter instruction and retry
                    messages = messages + [
                        {"role": "assistant", "content": content},
                        {"role": "user", "content": "Your response was not valid JSON. Please respond ONLY with valid JSON, no other text."},
                    ]
                    continue
                raise ValueError(f"Malformed JSON after {self.max_retries + 1} attempts")
```

**File:** `tests/test_llm_client.py`
**What:** Test LLM client with mocked httpx

Test cases (all use `unittest.mock.patch` to mock `httpx.Client`):
- Successful call to primary provider returns content
- Successful JSON parsing
- Primary fails, fallback succeeds
- Malformed JSON retries once with stricter prompt
- All providers fail raises LLMError
- Timeout handling
- API key sent in Authorization header when present
- `last_provider_used` is set correctly

#### Success Criteria

**Automated (run by agent):**
- [ ] `uv run pytest tests/test_llm_client.py -v` — all tests pass
- [ ] `uv run ruff check src/jobhunter/llm/` — no lint errors

---

### Phase 3: Scraper Engine

YAML-driven scraper with three strategies (API, HTML, Browser). Fetches listings, saves raw data to disk, normalizes into Job schema.

**Implements:** FR-001, FR-002, FR-003, FR-020, FR-030, US-1, US-12, NFR-002

#### Changes Required

**File:** `src/jobhunter/scraper/__init__.py`
**What:** Empty init

**File:** `src/jobhunter/scraper/strategies.py`
**What:** Three scraping strategy implementations

```python
from __future__ import annotations

from dataclasses import dataclass

import httpx
import structlog
from bs4 import BeautifulSoup

from jobhunter.models import SiteConfig

logger = structlog.get_logger("scraper")


@dataclass
class RawJobData:
    """Raw extracted data from a single job listing, before normalization."""
    title: str | None
    company: str | None
    location: str | None
    url: str | None
    salary: str | None
    date: str | None
    description: str | None
    requirements: str | None
    raw_html: str | None = None  # raw detail page HTML if fetched


class ScrapingError(Exception):
    pass


def scrape_api(config: SiteConfig, page: int, client: httpx.Client) -> tuple[list[RawJobData], str]:
    """Fetch job listings via API strategy.

    Args:
        config: Site configuration with api section populated.
        page: Page number to fetch.
        client: httpx client for requests.

    Returns:
        Tuple of (list of raw job data, raw response text for checkpoint).
    """
    # Build request from config.api dict:
    # - url, method, params (with {page} substitution), headers
    # - Navigate response JSON using response_path (dot-notation)
    # - Extract fields using field mappings (dot-notation into each job object)
    ...


def scrape_html(config: SiteConfig, page: int, client: httpx.Client) -> tuple[list[RawJobData], str]:
    """Fetch job listings via HTML parsing strategy.

    Args:
        config: Site configuration with html section populated.
        page: Page number to fetch.
        client: httpx client for requests.

    Returns:
        Tuple of (list of raw job data, raw response text for checkpoint).
    """
    # Build URL from config.html.list_url with {page} substitution
    # Fetch page, parse with BeautifulSoup + lxml
    # Find all job cards using job_card_selector
    # For each card, extract fields using field selectors and attribute type
    # Handle prefix for relative URLs
    # Handle optional fields (salary, date)
    ...


def scrape_browser(config: SiteConfig, page: int) -> tuple[list[RawJobData], str]:
    """Fetch job listings via headless browser strategy.

    Uses Playwright sync API. Handles JS-rendered content.

    Args:
        config: Site configuration with browser section populated.
        page: Page number to fetch.

    Returns:
        Tuple of (list of raw job data, page HTML content for checkpoint).
    """
    # Import playwright inline (optional dependency check)
    # Launch headless Chromium
    # Navigate to URL, wait for selector
    # Optionally scroll for lazy-loaded content
    # Extract using same selector approach as HTML strategy
    # Return raw page HTML
    ...


def fetch_detail_page(url: str, config: SiteConfig, client: httpx.Client) -> tuple[str | None, str | None]:
    """Fetch detail page for a job listing.

    Args:
        url: Job listing URL.
        config: Site config with detail_page section.
        client: httpx client.

    Returns:
        Tuple of (description text, requirements text).
    """
    # Fetch the detail URL
    # Parse with BeautifulSoup
    # Extract description and requirements using configured selectors
    ...
```

**File:** `src/jobhunter/scraper/normalizer.py`
**What:** Convert raw scraped data into normalized Job objects

```python
from __future__ import annotations

from datetime import datetime

from jobhunter.models import Job, generate_job_id


def normalize_job(
    raw: "RawJobData",
    site_id: str,
    country: str,
    run_id: str,
) -> Job | None:
    """Convert raw scraped data into a normalized Job.

    Returns None if required fields (title, company, location) are missing.
    """
    if not raw.title or not raw.company or not raw.location:
        return None

    title = raw.title.strip()
    company = raw.company.strip()
    location = raw.location.strip()

    return Job(
        id=generate_job_id(title, company, location),
        site_id=site_id,
        title=title,
        company=company,
        location=location,
        country=country,
        url=raw.url or "",
        salary=raw.salary.strip() if raw.salary else None,
        description=raw.description,
        requirements=raw.requirements,
        posted_date=raw.date,
        scraped_at=datetime.now().isoformat(),
        run_id=run_id,
    )
```

**File:** `src/jobhunter/scraper/engine.py`
**What:** Main scraper engine — loads configs, runs strategies, saves raw data, normalizes

```python
from __future__ import annotations

import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import date
from pathlib import Path

import httpx
import structlog

from jobhunter.models import Job, SiteConfig
from jobhunter.scraper.normalizer import normalize_job
from jobhunter.scraper.strategies import (
    RawJobData,
    ScrapingError,
    fetch_detail_page,
    scrape_api,
    scrape_browser,
    scrape_html,
)

logger = structlog.get_logger("scraper")

MAX_RETRIES = 3
RETRY_BACKOFF = [1, 3, 10]  # seconds


class ScraperEngine:
    """Scrapes all enabled sites and returns normalized jobs.

    Features:
    - Parallel scraping of sites via ThreadPoolExecutor
    - Raw data saved to disk before parsing (checkpoint)
    - Retry with exponential backoff on network errors
    - Per-site error isolation
    - Detail page fetching (configurable)
    """

    def __init__(self, data_dir: Path):
        self.data_dir = data_dir
        self.raw_dir = data_dir / "raw" / date.today().isoformat()
        self.raw_dir.mkdir(parents=True, exist_ok=True)

    def scrape_all(
        self,
        configs: list[SiteConfig],
        run_id: str,
    ) -> tuple[list[Job], list[dict]]:
        """Scrape all configured sites in parallel.

        Returns:
            Tuple of (all jobs, list of error dicts for failed sites).
        """
        all_jobs = []
        errors = []

        with ThreadPoolExecutor(max_workers=min(len(configs), 5)) as executor:
            futures = {
                executor.submit(self._scrape_site, config, run_id): config
                for config in configs
            }
            for future in as_completed(futures):
                config = futures[future]
                try:
                    jobs = future.result()
                    all_jobs.extend(jobs)
                    logger.info("site_scraped", site=config.site_id, jobs=len(jobs))
                except Exception as e:
                    error = {
                        "site": config.site_id,
                        "error": str(e),
                        "stage": "scrape",
                    }
                    errors.append(error)
                    logger.error("site_scrape_failed", **error)

        return all_jobs, errors

    def _scrape_site(self, config: SiteConfig, run_id: str) -> list[Job]:
        """Scrape a single site across all pages."""
        all_raw: list[RawJobData] = []

        with httpx.Client(timeout=30, follow_redirects=True) as client:
            for page in range(1, config.max_pages + 1):
                raw_jobs, raw_response = self._fetch_page_with_retry(config, page, client)
                # Checkpoint: save raw response to disk immediately
                self._save_raw(config.site_id, page, raw_response)
                all_raw.extend(raw_jobs)

                if not raw_jobs:
                    break  # no more results

            # Fetch detail pages if configured
            if config.detail_page and config.detail_page.get("enabled"):
                for raw in all_raw:
                    if raw.url:
                        try:
                            desc, reqs = fetch_detail_page(raw.url, config, client)
                            raw.description = desc
                            raw.requirements = reqs
                        except Exception as e:
                            logger.warning("detail_page_failed", url=raw.url, error=str(e))

        # Normalize
        jobs = []
        for raw in all_raw:
            job = normalize_job(raw, config.site_id, config.country, run_id)
            if job:
                jobs.append(job)

        return jobs

    def _fetch_page_with_retry(
        self, config: SiteConfig, page: int, client: httpx.Client
    ) -> tuple[list[RawJobData], str]:
        """Fetch a page with retry and backoff."""
        for attempt in range(MAX_RETRIES):
            try:
                if config.strategy == "api":
                    return scrape_api(config, page, client)
                elif config.strategy == "html":
                    return scrape_html(config, page, client)
                elif config.strategy == "browser":
                    return scrape_browser(config, page)
                else:
                    raise ScrapingError(f"Unknown strategy: {config.strategy}")
            except Exception as e:
                if attempt < MAX_RETRIES - 1:
                    wait = RETRY_BACKOFF[attempt]
                    logger.warning("scrape_retry", site=config.site_id, page=page, attempt=attempt + 1, wait=wait)
                    time.sleep(wait)
                else:
                    raise

    def _save_raw(self, site_id: str, page: int, content: str) -> None:
        """Save raw response to disk (checkpoint)."""
        path = self.raw_dir / f"{site_id}_page{page}.html"
        path.write_text(content, encoding="utf-8")
```

**File:** `tests/test_strategies.py`
**What:** Test each scraping strategy with mocked HTTP responses

Test cases:
- `scrape_html`: mock HTML response with job cards, verify RawJobData extraction
- `scrape_api`: mock JSON API response, verify field mapping via dot-notation path
- `scrape_html` with optional fields missing
- `scrape_html` with relative URL + prefix
- `fetch_detail_page`: mock detail page HTML, verify description extraction
- Empty response returns empty list

**File:** `tests/test_normalizer.py`
**What:** Test normalization logic

Test cases:
- Valid raw data produces Job with correct fields
- Missing required field (title/company/location) returns None
- Job ID is deterministic
- Whitespace stripped from all fields

**File:** `tests/test_scraper_engine.py`
**What:** Test engine orchestration (mocked strategies)

Test cases:
- Parallel scraping of multiple sites
- Raw data saved to disk before parsing
- Failed site doesn't block others (error isolation)
- Retry logic on network error
- Empty page stops pagination
- Detail page fetching when configured

#### Success Criteria

**Automated (run by agent):**
- [ ] `uv run pytest tests/test_strategies.py tests/test_normalizer.py tests/test_scraper_engine.py -v` — all tests pass
- [ ] `uv run ruff check src/jobhunter/scraper/` — no lint errors
- [ ] `uv run pytest tests/ -v` — all existing tests still pass

---

### Phase 4: Dedup & Matching

Hash-based deduplication, keyword filtering (global + per-site), and LLM scoring.

**Implements:** FR-004, FR-005, FR-006, FR-007, FR-008, US-2, US-3, US-4

#### Changes Required

**File:** `src/jobhunter/dedup.py`
**What:** Hash-based deduplication engine

```python
from __future__ import annotations

from datetime import datetime

import structlog

from jobhunter.db import Database
from jobhunter.models import Job

logger = structlog.get_logger("dedup")


class DedupEngine:
    """Filters out previously seen jobs using hash-based deduplication."""

    def __init__(self, db: Database):
        self.db = db

    def filter_new(self, jobs: list[Job]) -> list[Job]:
        """Return only jobs not previously seen.

        Uses the job ID (SHA-256 hash of title+company+location) as the dedup key.
        Marks new jobs as seen in the database.
        """
        new_jobs = []
        for job in jobs:
            if not self.db.has_seen_job(job.id):
                self.db.mark_job_seen(job.id, datetime.now().isoformat())
                new_jobs.append(job)
                logger.debug("new_job", job_id=job.id, title=job.title)
            else:
                logger.debug("duplicate_job", job_id=job.id, title=job.title)
        return new_jobs
```

**File:** `src/jobhunter/matcher/__init__.py`
**What:** Empty init

**File:** `src/jobhunter/matcher/keyword_filter.py`
**What:** Keyword-based filtering with global + per-site keywords

```python
from __future__ import annotations

import structlog

from jobhunter.models import Job, KeywordConfig

logger = structlog.get_logger("matcher")


class KeywordFilter:
    """Filters jobs based on keyword rules.

    Three checks applied in order:
    1. Job text must contain at least one keyword from must_have_any
    2. Job text must NOT contain any keyword from must_not_have
    3. Job title must contain at least one keyword from title_must_have_any

    All checks are case-insensitive.
    """

    def __init__(self, keywords: KeywordConfig):
        self.keywords = keywords

    def filter(self, jobs: list[Job]) -> tuple[list[Job], list[Job]]:
        """Filter jobs by keywords.

        Returns:
            Tuple of (passed jobs, filtered-out jobs).
        """
        passed = []
        filtered = []
        for job in jobs:
            reason = self._check(job)
            if reason is None:
                passed.append(job)
            else:
                filtered.append(job)
                logger.debug("job_filtered", job_id=job.id, title=job.title, reason=reason)
        return passed, filtered

    def _check(self, job: Job) -> str | None:
        """Check a single job against keyword rules.

        Returns None if job passes, or a reason string if filtered out.
        """
        # Combine all text fields for checking
        text = " ".join(
            filter(None, [job.title, job.description, job.requirements])
        ).lower()
        title = job.title.lower()

        # Check 1: must_have_any
        if self.keywords.must_have_any:
            if not any(kw.lower() in text for kw in self.keywords.must_have_any):
                return "no_required_keyword"

        # Check 2: must_not_have
        for kw in self.keywords.must_not_have:
            if kw.lower() in text:
                return f"has_excluded_keyword:{kw}"

        # Check 3: title_must_have_any
        if self.keywords.title_must_have_any:
            if not any(kw.lower() in title for kw in self.keywords.title_must_have_any):
                return "title_missing_role_keyword"

        return None
```

**File:** `src/jobhunter/matcher/scorer.py`
**What:** LLM-based job scoring

```python
from __future__ import annotations

import structlog

from jobhunter.llm.client import LLMClient, LLMError
from jobhunter.models import Job, ScoredJob

logger = structlog.get_logger("scorer")

SCORING_PROMPT = """You are a job matching assistant. Score how well this job matches the candidate profile.

## Candidate Profile (Master Resume)
{master_resume}

## Scoring Criteria
{scoring_criteria}

## Job Listing
Title: {title}
Company: {company}
Location: {location}
Country: {country}
Salary: {salary}

Description:
{description}

Requirements:
{requirements}

## Instructions
Score this job 0-100 based on:
- Technical skill match ({tech_weight}% weight)
- Seniority level match ({seniority_weight}% weight)
- Role relevance ({role_weight}% weight)
- Visa/relocation feasibility ({visa_weight}% weight)
- Growth potential ({growth_weight}% weight)

Respond ONLY with valid JSON:
{{"score": <integer 0-100>, "reasoning": "<2-3 sentences explaining the score>", "concerns": "<any concerns or red flags, or null>"}}
"""


class LLMScorer:
    """Scores jobs against candidate profile using LLM."""

    def __init__(self, llm: LLMClient, master_resume: dict, scoring_criteria: dict):
        self.llm = llm
        self.master_resume = master_resume
        self.scoring_criteria = scoring_criteria

    def score(self, job: Job) -> ScoredJob | None:
        """Score a single job. Returns None if LLM call fails entirely."""
        from datetime import datetime
        import yaml

        prompt = SCORING_PROMPT.format(
            master_resume=yaml.dump(self.master_resume, default_flow_style=False),
            scoring_criteria=yaml.dump(self.scoring_criteria, default_flow_style=False),
            title=job.title,
            company=job.company,
            location=job.location,
            country=job.country,
            salary=job.salary or "Not specified",
            description=self._truncate(job.description or "Not available"),
            requirements=self._truncate(job.requirements or "Not available"),
            tech_weight=self.scoring_criteria.get("technical_match_weight", 30),
            seniority_weight=self.scoring_criteria.get("seniority_match_weight", 20),
            role_weight=self.scoring_criteria.get("role_relevance_weight", 20),
            visa_weight=self.scoring_criteria.get("visa_feasibility_weight", 15),
            growth_weight=self.scoring_criteria.get("growth_potential_weight", 15),
        )

        try:
            result = self.llm.chat(
                messages=[{"role": "user", "content": prompt}],
                json_response=True,
            )
            data = result["content"]
            score = max(0, min(100, int(data.get("score", 0))))  # Clamp 0-100
            reasoning = data.get("reasoning", "")
            concerns = data.get("concerns")
            if concerns:
                reasoning += f"\nConcerns: {concerns}"

            return ScoredJob(
                job=job,
                score=score,
                reasoning=reasoning,
                provider=result["provider"],
                scored_at=datetime.now().isoformat(),
            )
        except LLMError as e:
            logger.error("scoring_failed", job_id=job.id, error=str(e))
            return None

    def _truncate(self, text: str, max_chars: int = 8000) -> str:
        """Truncate text to avoid exceeding token limits."""
        if len(text) > max_chars:
            logger.warning("description_truncated", original_length=len(text))
            return text[:max_chars] + "\n[...truncated]"
        return text
```

**File:** `tests/test_dedup.py`
**What:** Test dedup engine

Test cases:
- New job passes through and is marked as seen
- Same job on second pass is filtered out
- Different jobs both pass through
- Multiple jobs with one duplicate

**File:** `tests/test_keyword_filter.py`
**What:** Test keyword filter

Test cases:
- Job with required keyword passes
- Job without any required keyword is filtered
- Job with excluded keyword is filtered
- Job title without role keyword is filtered
- Empty keyword lists pass everything
- Case insensitive matching
- Keywords checked across title + description + requirements

**File:** `tests/test_scorer.py`
**What:** Test LLM scorer with mocked LLMClient

Test cases:
- Successful scoring returns ScoredJob with valid score
- Score clamped to 0-100 range
- LLM failure returns None (doesn't crash)
- Long description is truncated
- Prompt includes all job fields

#### Success Criteria

**Automated (run by agent):**
- [ ] `uv run pytest tests/test_dedup.py tests/test_keyword_filter.py tests/test_scorer.py -v` — all tests pass
- [ ] `uv run ruff check src/jobhunter/dedup.py src/jobhunter/matcher/` — no lint errors
- [ ] `uv run pytest tests/ -v` — all existing tests still pass

---

### Phase 5: Resume Tailor

Tailors resumes via LLM, verifies for accuracy, converts HTML to PDF.

**Implements:** FR-009, FR-010, FR-011, FR-012, US-5, US-6

#### Changes Required

**File:** `src/jobhunter/tailor/__init__.py`
**What:** Empty init

**File:** `src/jobhunter/tailor/engine.py`
**What:** Resume tailoring via LLM

```python
from __future__ import annotations

import structlog
import yaml

from jobhunter.llm.client import LLMClient, LLMError
from jobhunter.models import Job

logger = structlog.get_logger("tailor")

TAILOR_PROMPT = """You are a professional resume writer. Given a master resume and a job description, produce a tailored one-page resume in HTML.

## Master Resume
{master_resume}

## Job Description
Title: {title}
Company: {company}
Location: {location}
Description: {description}

## Instructions
1. Select the most relevant experience, skills, and achievements for THIS specific job.
2. Choose the best summary variant or write a new one that targets this role.
3. Reorder experience to put the most relevant first.
4. Include only certifications and skills relevant to this role.
5. Keep it to one page worth of content.
6. NEVER fabricate skills, experience, or achievements not in the master resume.
7. Output ONLY the HTML content to be inserted into the resume template body. Use semantic HTML (h1, h2, h3, p, ul, li, etc.). Do not include <html>, <head>, or <body> tags.
"""


class ResumeTailor:
    """Generates tailored resumes using LLM."""

    def __init__(self, llm: LLMClient, master_resume: dict):
        self.llm = llm
        self.master_resume = master_resume

    def tailor(self, job: Job) -> str | None:
        """Generate tailored resume HTML for a job.

        Returns HTML content string, or None if LLM fails.
        """
        prompt = TAILOR_PROMPT.format(
            master_resume=yaml.dump(self.master_resume, default_flow_style=False),
            title=job.title,
            company=job.company,
            location=job.location,
            description=job.description or "Not available",
        )

        try:
            result = self.llm.chat(
                messages=[{"role": "user", "content": prompt}],
                json_response=False,
            )
            html_content = result["content"]
            # Strip markdown code fences if present
            if html_content.startswith("```"):
                lines = html_content.split("\n")
                html_content = "\n".join(lines[1:-1])
            return html_content
        except LLMError as e:
            logger.error("tailoring_failed", job_id=job.id, error=str(e))
            return None
```

**File:** `src/jobhunter/tailor/verifier.py`
**What:** Verify tailored resume against master resume

```python
from __future__ import annotations

from dataclasses import dataclass

import structlog
import yaml

from jobhunter.llm.client import LLMClient, LLMError

logger = structlog.get_logger("verifier")

VERIFY_PROMPT = """Compare this tailored resume against the master resume. Check for accuracy.

## Master Resume (Source of Truth)
{master_resume}

## Tailored Resume
{tailored_html}

## Check for:
1. Fabricated skills not in master resume
2. Inflated claims or exaggerated metrics
3. Companies, roles, or dates that don't match master resume
4. Missing critical information (name, contact)

Respond ONLY with valid JSON:
{{"pass": <true or false>, "issues": [<list of issue strings, empty if pass>]}}
"""


@dataclass
class VerificationResult:
    passed: bool
    issues: list[str]


class ResumeVerifier:
    """Verifies tailored resumes against master resume for accuracy."""

    def __init__(self, llm: LLMClient, master_resume: dict):
        self.llm = llm
        self.master_resume = master_resume

    def verify(self, tailored_html: str) -> VerificationResult:
        """Verify a tailored resume. Returns pass/fail with issues."""
        prompt = VERIFY_PROMPT.format(
            master_resume=yaml.dump(self.master_resume, default_flow_style=False),
            tailored_html=tailored_html,
        )

        try:
            result = self.llm.chat(
                messages=[{"role": "user", "content": prompt}],
                json_response=True,
            )
            data = result["content"]
            return VerificationResult(
                passed=bool(data.get("pass", False)),
                issues=data.get("issues", []),
            )
        except (LLMError, Exception) as e:
            logger.error("verification_failed", error=str(e))
            return VerificationResult(passed=False, issues=[f"Verification error: {e}"])
```

**File:** `src/jobhunter/tailor/pdf.py`
**What:** HTML to PDF conversion using WeasyPrint

```python
from __future__ import annotations

from pathlib import Path

import structlog

logger = structlog.get_logger("pdf")

RESUME_TEMPLATE_PATH = Path(__file__).parent.parent.parent.parent / "templates" / "resume.html"


def render_resume_html(body_content: str, template_path: Path | None = None) -> str:
    """Insert tailored body content into the HTML resume template.

    Args:
        body_content: The tailored HTML content from the LLM.
        template_path: Path to the HTML template. Uses default if not provided.

    Returns:
        Complete HTML string ready for PDF conversion.
    """
    path = template_path or RESUME_TEMPLATE_PATH
    template = path.read_text(encoding="utf-8")
    return template.replace("{{CONTENT}}", body_content)


def html_to_pdf(html_content: str, output_path: Path) -> Path:
    """Convert HTML string to PDF file using WeasyPrint.

    Args:
        html_content: Complete HTML string.
        output_path: Where to save the PDF.

    Returns:
        Path to the generated PDF file.
    """
    from weasyprint import HTML

    output_path.parent.mkdir(parents=True, exist_ok=True)
    HTML(string=html_content).write_pdf(str(output_path))
    logger.info("pdf_generated", path=str(output_path))
    return output_path
```

**File:** `templates/resume.html`
**What:** HTML resume template with CSS styling

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<style>
  @page {
    size: A4;
    margin: 1.5cm 2cm;
  }
  body {
    font-family: 'Helvetica Neue', Arial, sans-serif;
    font-size: 10pt;
    line-height: 1.4;
    color: #333;
    margin: 0;
    padding: 0;
  }
  h1 {
    font-size: 18pt;
    color: #1a1a1a;
    margin: 0 0 4pt 0;
    border-bottom: 2pt solid #2563eb;
    padding-bottom: 4pt;
  }
  h2 {
    font-size: 12pt;
    color: #2563eb;
    margin: 12pt 0 4pt 0;
    text-transform: uppercase;
    letter-spacing: 0.5pt;
  }
  h3 {
    font-size: 10.5pt;
    color: #1a1a1a;
    margin: 6pt 0 2pt 0;
  }
  p {
    margin: 2pt 0;
  }
  ul {
    margin: 2pt 0;
    padding-left: 16pt;
  }
  li {
    margin: 1pt 0;
  }
  .contact {
    font-size: 9pt;
    color: #666;
    margin-bottom: 8pt;
  }
  .skills-list {
    columns: 2;
    column-gap: 20pt;
  }
  .job-title {
    font-weight: bold;
  }
  .job-meta {
    font-size: 9pt;
    color: #666;
    font-style: italic;
  }
</style>
</head>
<body>
{{CONTENT}}
</body>
</html>
```

**File:** `tests/test_tailor.py`
**What:** Test tailoring, verification, and PDF generation

Test cases:
- `ResumeTailor.tailor()` with mocked LLM returns HTML content
- `ResumeTailor.tailor()` strips markdown code fences from response
- `ResumeVerifier.verify()` with passing resume returns passed=True
- `ResumeVerifier.verify()` with fabricated skill returns passed=False with issues
- `render_resume_html()` inserts content into template
- `html_to_pdf()` produces a valid PDF file (check file exists and is > 0 bytes)
- LLM failure in tailoring returns None
- LLM failure in verification returns passed=False

#### Success Criteria

**Automated (run by agent):**
- [ ] `uv run pytest tests/test_tailor.py -v` — all tests pass
- [ ] `uv run ruff check src/jobhunter/tailor/` — no lint errors
- [ ] `uv run pytest tests/ -v` — all existing tests still pass

---

### Phase 6: Notifications & Feedback

Telegram notifications with inline buttons, email digest, application tracking, and feedback collection.

**Implements:** FR-013, FR-014, FR-015, FR-023, FR-024, FR-025, FR-026, FR-027, FR-028, US-7, US-8, US-10, US-11, US-13

#### Changes Required

**File:** `src/jobhunter/notifier/__init__.py`
**What:** Empty init

**File:** `src/jobhunter/notifier/telegram.py`
**What:** Telegram bot client for notifications, health alerts, and feedback collection

```python
from __future__ import annotations

import asyncio
from pathlib import Path

import structlog

logger = structlog.get_logger("telegram")


class TelegramNotifier:
    """Sends Telegram notifications and collects feedback via inline buttons.

    Uses python-telegram-bot library. Async methods wrapped with asyncio.run()
    for use in the sync pipeline.
    """

    def __init__(self, bot_token: str, chat_id: str):
        self.bot_token = bot_token
        self.chat_id = chat_id

    def send_match(
        self,
        job_id: str,
        title: str,
        company: str,
        location: str,
        salary: str | None,
        score: int,
        reasoning: str,
        job_url: str,
        pdf_path: Path | None = None,
        prior_applications: list[dict] | None = None,
    ) -> str | None:
        """Send a match notification with inline reaction buttons.

        Returns the Telegram message ID, or None on failure.
        """
        return asyncio.run(self._send_match(
            job_id, title, company, location, salary,
            score, reasoning, job_url, pdf_path, prior_applications
        ))

    async def _send_match(self, job_id, title, company, location, salary,
                           score, reasoning, job_url, pdf_path, prior_applications):
        """Async implementation of send_match."""
        from telegram import Bot, InlineKeyboardButton, InlineKeyboardMarkup

        bot = Bot(self.bot_token)

        # Format message
        text = f"*Match Score: {score}/100*\n\n"
        text += f"*{title}* — {company}\n"
        text += f"{location}\n"
        if salary:
            text += f"{salary}\n"
        text += f"\n_{reasoning}_\n"
        if prior_applications:
            text += "\n⚠️ *Prior applications at this company:*\n"
            for app in prior_applications:
                text += f"  • {app['role']} ({app['status']})\n"
        text += f"\n[View Listing]({job_url})"

        keyboard = InlineKeyboardMarkup([
            [
                InlineKeyboardButton("Applied", callback_data=f"applied:{job_id}"),
                InlineKeyboardButton("Skip", callback_data=f"skip:{job_id}"),
                InlineKeyboardButton("Not Relevant", callback_data=f"not_relevant:{job_id}"),
            ]
        ])

        msg = await bot.send_message(
            chat_id=self.chat_id,
            text=text,
            parse_mode="Markdown",
            reply_markup=keyboard,
        )

        # Send PDF if available
        if pdf_path and pdf_path.exists():
            with open(pdf_path, "rb") as f:
                await bot.send_document(chat_id=self.chat_id, document=f)

        return str(msg.message_id)

    def send_health_alert(self, message: str) -> None:
        """Send a pipeline health alert."""
        asyncio.run(self._send_alert(message))

    async def _send_alert(self, message: str):
        from telegram import Bot
        bot = Bot(self.bot_token)
        await bot.send_message(chat_id=self.chat_id, text=f"🔧 *Pipeline Alert*\n\n{message}", parse_mode="Markdown")

    def poll_feedback(self) -> list[dict]:
        """Poll for pending callback queries (button taps).

        Returns list of {"job_id": str, "action": str} dicts.
        Acknowledges each callback to remove the "loading" indicator.
        """
        return asyncio.run(self._poll_feedback())

    async def _poll_feedback(self) -> list[dict]:
        from telegram import Bot
        bot = Bot(self.bot_token)
        updates = await bot.get_updates(allowed_updates=["callback_query"])
        feedback = []
        for update in updates:
            if update.callback_query:
                data = update.callback_query.data
                if ":" in data:
                    action, job_id = data.split(":", 1)
                    feedback.append({"job_id": job_id, "action": action})
                    await update.callback_query.answer(text=f"Marked as {action}")
                # Acknowledge the update
                await bot.get_updates(offset=update.update_id + 1)
        return feedback
```

**File:** `src/jobhunter/notifier/email.py`
**What:** Email digest sender using smtplib

```python
from __future__ import annotations

import smtplib
from email.mime.application import MIMEApplication
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from pathlib import Path

import structlog

logger = structlog.get_logger("email")


class EmailNotifier:
    """Sends daily digest emails with job matches and resume PDFs."""

    def __init__(self, smtp_host: str, smtp_port: int, from_addr: str, to_addr: str, password: str):
        self.smtp_host = smtp_host
        self.smtp_port = smtp_port
        self.from_addr = from_addr
        self.to_addr = to_addr
        self.password = password

    def send_digest(self, matches: list[dict]) -> bool:
        """Send a daily digest email with all matches.

        Args:
            matches: List of dicts with keys: title, company, location, score, reasoning, pdf_path, job_url

        Returns:
            True if email sent successfully.
        """
        if not matches:
            return True

        msg = MIMEMultipart()
        msg["From"] = self.from_addr
        msg["To"] = self.to_addr
        msg["Subject"] = f"JobHunter Daily Digest — {len(matches)} matches"

        # Build HTML body
        body = "<h1>JobHunter Daily Digest</h1>\n"
        for match in matches:
            body += f"<h2>{match['title']} — {match['company']}</h2>\n"
            body += f"<p>Score: {match['score']}/100 | {match['location']}</p>\n"
            body += f"<p><em>{match['reasoning']}</em></p>\n"
            body += f"<p><a href=\"{match['job_url']}\">View Listing</a></p>\n<hr>\n"

        msg.attach(MIMEText(body, "html"))

        # Attach PDFs
        for match in matches:
            pdf_path = Path(match.get("pdf_path", ""))
            if pdf_path.exists():
                with open(pdf_path, "rb") as f:
                    attachment = MIMEApplication(f.read(), _subtype="pdf")
                    attachment.add_header(
                        "Content-Disposition", "attachment",
                        filename=f"{match['company']}_{match['title']}.pdf"
                    )
                    msg.attach(attachment)

        try:
            with smtplib.SMTP(self.smtp_host, self.smtp_port) as server:
                server.starttls()
                server.login(self.from_addr, self.password)
                server.send_message(msg)
            logger.info("digest_sent", matches=len(matches))
            return True
        except Exception as e:
            logger.error("digest_failed", error=str(e))
            return False
```

**File:** `src/jobhunter/tracker.py`
**What:** Application tracker

```python
from __future__ import annotations

import uuid
from datetime import datetime

import structlog

from jobhunter.db import Database
from jobhunter.models import Application, Job

logger = structlog.get_logger("tracker")

VALID_STATUSES = {"matched", "applied", "phone_screen", "interview", "offer", "rejected", "withdrawn", "expired"}


class ApplicationTracker:
    """Tracks job applications and checks for duplicate company applications."""

    def __init__(self, db: Database):
        self.db = db

    def create_application(self, job: Job) -> Application:
        """Create a new application record with status 'matched'."""
        app = Application(
            id=uuid.uuid4().hex[:12],
            job_id=job.id,
            company=job.company,
            role=job.title,
            country=job.country,
            applied_date=None,
            resume_version=None,
            status="matched",
            status_updated=datetime.now().isoformat(),
            notes=None,
            source_site=job.site_id,
        )
        self.db.insert_application(app)
        return app

    def get_prior_applications(self, company: str) -> list[Application]:
        """Get all prior applications to the same company."""
        return self.db.get_applications_by_company(company)

    def update_status(self, app_id: str, status: str) -> None:
        """Update application status."""
        if status not in VALID_STATUSES:
            raise ValueError(f"Invalid status: {status}. Must be one of {VALID_STATUSES}")
        self.db.update_application_status(app_id, status, datetime.now().isoformat())
```

**File:** `src/jobhunter/feedback.py`
**What:** Feedback storage

```python
from __future__ import annotations

from datetime import datetime

import structlog

from jobhunter.db import Database
from jobhunter.models import Feedback

logger = structlog.get_logger("feedback")


class FeedbackStore:
    """Stores user feedback on job matches."""

    def __init__(self, db: Database):
        self.db = db

    def record(self, job_id: str, score: int, action: str, reason: str | None = None) -> None:
        """Record user feedback for a job match."""
        if action not in ("applied", "skipped", "not_relevant"):
            raise ValueError(f"Invalid action: {action}")
        fb = Feedback(
            job_id=job_id,
            score=score,
            action=action,
            reason=reason,
            timestamp=datetime.now().isoformat(),
        )
        self.db.insert_feedback(fb)
        logger.info("feedback_recorded", job_id=job_id, action=action)
```

**File:** `tests/test_telegram.py`, `tests/test_email_notifier.py`, `tests/test_tracker.py`, `tests/test_feedback.py`
**What:** Tests for all notification and tracking modules

Telegram test cases (mock `telegram.Bot`):
- `send_match` formats message correctly with score, title, company
- `send_match` attaches PDF when path exists
- `send_match` includes prior application warning
- `send_health_alert` sends formatted alert message
- `poll_feedback` parses callback data correctly

Email test cases (mock `smtplib.SMTP`):
- `send_digest` constructs correct email with all matches
- `send_digest` attaches PDFs
- `send_digest` handles empty match list
- SMTP failure returns False

Tracker test cases:
- Create application with matched status
- Get prior applications by company
- Update status
- Invalid status raises ValueError

Feedback test cases:
- Record applied/skipped/not_relevant feedback
- Invalid action raises ValueError

#### Success Criteria

**Automated (run by agent):**
- [ ] `uv run pytest tests/test_telegram.py tests/test_email_notifier.py tests/test_tracker.py tests/test_feedback.py -v` — all tests pass
- [ ] `uv run ruff check src/jobhunter/notifier/ src/jobhunter/tracker.py src/jobhunter/feedback.py` — no lint errors
- [ ] `uv run pytest tests/ -v` — all existing tests still pass

---

### Phase 7: Pipeline Orchestrator

Checkpoint system for crash recovery and the main pipeline runner that ties everything together.

**Implements:** FR-016, FR-017, FR-018, FR-019, FR-029, US-9, US-13, NFR-006

#### Changes Required

**File:** `src/jobhunter/checkpoint.py`
**What:** Checkpoint manager for crash recovery

```python
from __future__ import annotations

import json
from datetime import datetime
from pathlib import Path

import structlog

logger = structlog.get_logger("checkpoint")


class CheckpointManager:
    """Manages pipeline state for crash recovery.

    Tracks which stages have completed and which jobs have been processed.
    On restart, the pipeline can resume from the last completed checkpoint.
    """

    def __init__(self, checkpoint_dir: Path):
        self.checkpoint_dir = checkpoint_dir
        self.checkpoint_dir.mkdir(parents=True, exist_ok=True)
        self.state_file = checkpoint_dir / "last_run.json"
        self.state: dict = {}

    def load(self) -> dict | None:
        """Load the last checkpoint state. Returns None if no checkpoint exists."""
        if not self.state_file.exists():
            return None
        with open(self.state_file) as f:
            self.state = json.load(f)
        return self.state

    def save(self) -> None:
        """Save current state to disk."""
        self.state["updated_at"] = datetime.now().isoformat()
        with open(self.state_file, "w") as f:
            json.dump(self.state, f, indent=2)

    def start_run(self, run_id: str) -> None:
        """Mark a new pipeline run as started."""
        self.state = {
            "run_id": run_id,
            "status": "running",
            "started_at": datetime.now().isoformat(),
            "completed_stages": [],
            "scraped_sites": [],
            "scored_jobs": [],
            "tailored_jobs": [],
            "notified_jobs": [],
        }
        self.save()

    def complete_stage(self, stage: str) -> None:
        """Mark a pipeline stage as completed."""
        if stage not in self.state.get("completed_stages", []):
            self.state.setdefault("completed_stages", []).append(stage)
            self.save()

    def is_stage_complete(self, stage: str) -> bool:
        """Check if a stage was already completed."""
        return stage in self.state.get("completed_stages", [])

    def mark_site_scraped(self, site_id: str) -> None:
        """Mark a site as scraped (for crash recovery)."""
        self.state.setdefault("scraped_sites", []).append(site_id)
        self.save()

    def is_site_scraped(self, site_id: str) -> bool:
        return site_id in self.state.get("scraped_sites", [])

    def mark_job_scored(self, job_id: str) -> None:
        self.state.setdefault("scored_jobs", []).append(job_id)
        self.save()

    def is_job_scored(self, job_id: str) -> bool:
        return job_id in self.state.get("scored_jobs", [])

    def mark_job_tailored(self, job_id: str) -> None:
        self.state.setdefault("tailored_jobs", []).append(job_id)
        self.save()

    def is_job_tailored(self, job_id: str) -> bool:
        return job_id in self.state.get("tailored_jobs", [])

    def mark_job_notified(self, job_id: str) -> None:
        self.state.setdefault("notified_jobs", []).append(job_id)
        self.save()

    def is_job_notified(self, job_id: str) -> bool:
        return job_id in self.state.get("notified_jobs", [])

    def complete_run(self) -> None:
        """Mark the run as completed successfully."""
        self.state["status"] = "completed"
        self.state["completed_at"] = datetime.now().isoformat()
        self.save()

    def is_incomplete(self) -> bool:
        """Check if the last run was incomplete (crashed)."""
        return self.state.get("status") == "running"
```

**File:** `src/jobhunter/pipeline.py`
**What:** Main pipeline orchestrator

```python
from __future__ import annotations

from datetime import datetime
from pathlib import Path

import structlog

from jobhunter.checkpoint import CheckpointManager
from jobhunter.config import load_master_resume, load_settings, load_site_configs, merge_keywords
from jobhunter.db import Database
from jobhunter.dedup import DedupEngine
from jobhunter.feedback import FeedbackStore
from jobhunter.llm.client import LLMClient
from jobhunter.log import get_logger, setup_logging
from jobhunter.matcher.keyword_filter import KeywordFilter
from jobhunter.matcher.scorer import LLMScorer
from jobhunter.models import PipelineRun, generate_run_id
from jobhunter.notifier.email import EmailNotifier
from jobhunter.notifier.telegram import TelegramNotifier
from jobhunter.scraper.engine import ScraperEngine
from jobhunter.tailor.engine import ResumeTailor
from jobhunter.tailor.pdf import html_to_pdf, render_resume_html
from jobhunter.tailor.verifier import ResumeVerifier
from jobhunter.tracker import ApplicationTracker

logger = structlog.get_logger("pipeline")


class PipelineRunner:
    """Orchestrates the full job search pipeline.

    Stages:
    1. Poll for feedback (from previous notifications)
    2. Scrape all enabled sites (parallel)
    3. Deduplicate
    4. Keyword filter
    5. LLM scoring
    6. Resume tailoring + verification + PDF
    7. Send notifications
    8. Log results

    Each stage is checkpointed. On crash, the next run resumes
    from the last completed checkpoint.
    """

    def __init__(self, config_path: Path | None = None):
        self.settings = load_settings(config_path)
        self.project_root = Path(__file__).parent.parent.parent
        self.data_dir = self.project_root / self.settings.data_dir
        self.output_dir = self.project_root / self.settings.output_dir
        self.log_dir = self.project_root / self.settings.log_dir

        setup_logging(self.log_dir)

        self.db = Database(self.data_dir / "jobhunter.db")
        self.checkpoint = CheckpointManager(self.data_dir / "checkpoints")
        self.scraper = ScraperEngine(self.data_dir)
        self.dedup = DedupEngine(self.db)
        self.tracker = ApplicationTracker(self.db)
        self.feedback_store = FeedbackStore(self.db)

        # LLM client
        self.llm = LLMClient(
            primary=self.settings.llm_primary,
            fallback=self.settings.llm_fallback,
            timeout=self.settings.llm_timeout,
            max_retries=self.settings.llm_max_retries,
        )

        # Master resume
        self.master_resume = load_master_resume()

        # Scorer and tailor
        self.scorer = LLMScorer(self.llm, self.master_resume, self.settings.scoring_criteria)
        self.tailor = ResumeTailor(self.llm, self.master_resume)
        self.verifier = ResumeVerifier(self.llm, self.master_resume)

        # Notifiers (optional — only if configured)
        self.telegram = None
        if self.settings.telegram_bot_token and self.settings.telegram_chat_id:
            self.telegram = TelegramNotifier(
                self.settings.telegram_bot_token,
                self.settings.telegram_chat_id,
            )

        self.email = None
        if all([self.settings.email_smtp_host, self.settings.email_from,
                self.settings.email_to, self.settings.email_password]):
            self.email = EmailNotifier(
                self.settings.email_smtp_host,
                self.settings.email_smtp_port,
                self.settings.email_from,
                self.settings.email_to,
                self.settings.email_password,
            )

    def run(self) -> PipelineRun:
        """Execute the full pipeline. Returns the run summary."""
        run_id = generate_run_id()
        run = PipelineRun(
            run_id=run_id,
            started_at=datetime.now().isoformat(),
        )
        self.db.insert_run(run)

        # Check for incomplete previous run
        prev_state = self.checkpoint.load()
        if prev_state and self.checkpoint.is_incomplete():
            logger.info("resuming_incomplete_run", previous_run_id=prev_state.get("run_id"))
            # Use previous run's checkpoint state for resume logic
        else:
            self.checkpoint.start_run(run_id)

        try:
            # Stage 0: Poll feedback from previous notifications
            self._poll_feedback()

            # Stage 1: Scrape
            configs = load_site_configs()
            if not configs:
                logger.warning("no_site_configs_enabled")
                run.status = "completed"
                run.completed_at = datetime.now().isoformat()
                self.db.update_run(run)
                self.checkpoint.complete_run()
                return run

            run.sites_attempted = [c.site_id for c in configs]

            # Skip already-scraped sites (crash recovery)
            configs_to_scrape = [
                c for c in configs
                if not self.checkpoint.is_site_scraped(c.site_id)
            ]

            jobs, scrape_errors = self.scraper.scrape_all(configs_to_scrape, run_id)
            for config in configs_to_scrape:
                if config.site_id not in [e["site"] for e in scrape_errors]:
                    self.checkpoint.mark_site_scraped(config.site_id)
                    run.sites_succeeded.append(config.site_id)
                else:
                    run.sites_failed.extend(
                        [e for e in scrape_errors if e["site"] == config.site_id]
                    )

            run.jobs_scraped = len(jobs)

            # Save all jobs to DB
            for job in jobs:
                self.db.insert_job(job)
            self.checkpoint.complete_stage("scrape")

            # Stage 2: Dedup
            new_jobs = self.dedup.filter_new(jobs)
            run.jobs_new = len(new_jobs)
            self.checkpoint.complete_stage("dedup")

            # Stage 3: Keyword filter (per-site merged keywords)
            filtered_jobs = []
            for job in new_jobs:
                # Find the site config for this job to get per-site keywords
                site_cfg = next((c for c in configs if c.site_id == job.site_id), None)
                site_kw = site_cfg.keywords if site_cfg else None
                merged = merge_keywords(self.settings.keywords, site_kw)
                kf = KeywordFilter(merged)
                passed, _ = kf.filter([job])
                filtered_jobs.extend(passed)

            run.jobs_filtered_out = run.jobs_new - len(filtered_jobs)
            self.checkpoint.complete_stage("filter")

            # Stage 4: LLM Scoring
            scored_jobs = []
            for job in filtered_jobs:
                if self.checkpoint.is_job_scored(job.id):
                    # Already scored in a previous attempt — load from DB
                    score_data = self.db.get_score(job.id)
                    if score_data:
                        from jobhunter.models import ScoredJob
                        scored_jobs.append(ScoredJob(
                            job=job,
                            score=score_data["score"],
                            reasoning=score_data["reasoning"],
                            provider=score_data["provider"],
                            scored_at=score_data["scored_at"],
                        ))
                    continue

                scored = self.scorer.score(job)
                if scored:
                    self.db.insert_score(
                        job.id, scored.score, scored.reasoning,
                        scored.provider, scored.scored_at, run_id,
                    )
                    self.checkpoint.mark_job_scored(job.id)
                    scored_jobs.append(scored)
                    run.jobs_scored += 1
                else:
                    run.errors.append({"job_id": job.id, "stage": "scoring", "error": "LLM failed"})

            # Filter by threshold
            above_threshold = [s for s in scored_jobs if s.score >= self.settings.scoring_threshold]
            run.jobs_above_threshold = len(above_threshold)
            self.checkpoint.complete_stage("score")

            # Stage 5: Resume tailoring
            tailored_results = []  # list of (ScoredJob, pdf_path or None)
            for scored in above_threshold:
                if self.checkpoint.is_job_tailored(scored.job.id):
                    resume = self.db.get_resume(scored.job.id)
                    if resume:
                        tailored_results.append((scored, Path(resume.pdf_path)))
                    continue

                # Tailor
                html_content = self.tailor.tailor(scored.job)
                if not html_content:
                    run.errors.append({"job_id": scored.job.id, "stage": "tailor", "error": "LLM failed"})
                    tailored_results.append((scored, None))
                    continue

                # Verify (with retries)
                verified = False
                for attempt in range(3):  # initial + 2 retries
                    result = self.verifier.verify(html_content)
                    if result.passed:
                        verified = True
                        break
                    logger.warning("verification_failed", job_id=scored.job.id, attempt=attempt + 1, issues=result.issues)
                    if attempt < 2:
                        # Re-tailor with feedback
                        html_content = self.tailor.tailor(scored.job)
                        if not html_content:
                            break

                if verified and html_content:
                    # Render full HTML and convert to PDF
                    full_html = render_resume_html(html_content)
                    pdf_dir = self.output_dir / "resumes"
                    pdf_path = pdf_dir / f"{scored.job.company}_{scored.job.title}_{scored.job.id[:8]}.pdf"
                    html_path = pdf_dir / f"{scored.job.id[:8]}.html"

                    html_path.parent.mkdir(parents=True, exist_ok=True)
                    html_path.write_text(full_html, encoding="utf-8")
                    html_to_pdf(full_html, pdf_path)

                    from jobhunter.models import TailoredResume
                    resume = TailoredResume(
                        job_id=scored.job.id,
                        html_path=str(html_path),
                        pdf_path=str(pdf_path),
                        verified=True,
                        verification_issues=[],
                        generated_at=datetime.now().isoformat(),
                        run_id=run_id,
                    )
                    self.db.insert_resume(resume)
                    self.checkpoint.mark_job_tailored(scored.job.id)
                    run.resumes_generated += 1
                    tailored_results.append((scored, pdf_path))
                else:
                    run.errors.append({"job_id": scored.job.id, "stage": "verify", "error": "Verification failed after retries"})
                    tailored_results.append((scored, None))

            self.checkpoint.complete_stage("tailor")

            # Stage 6: Notifications
            for scored, pdf_path in tailored_results:
                if self.checkpoint.is_job_notified(scored.job.id):
                    continue

                # Create application record
                self.tracker.create_application(scored.job)
                prior = self.tracker.get_prior_applications(scored.job.company)
                # Exclude the one we just created
                prior = [a for a in prior if a.job_id != scored.job.id]

                prior_dicts = [{"role": a.role, "status": a.status} for a in prior] if prior else None

                # Notification routing by score
                telegram_sent = False
                if scored.score >= self.settings.notifications.instant_telegram and self.telegram:
                    try:
                        msg_id = self.telegram.send_match(
                            job_id=scored.job.id,
                            title=scored.job.title,
                            company=scored.job.company,
                            location=scored.job.location,
                            salary=scored.job.salary,
                            score=scored.score,
                            reasoning=scored.reasoning,
                            job_url=scored.job.url,
                            pdf_path=pdf_path,
                            prior_applications=prior_dicts,
                        )
                        telegram_sent = True
                    except Exception as e:
                        logger.error("telegram_failed", job_id=scored.job.id, error=str(e))

                self.db.mark_notified(
                    scored.job.id, telegram_sent, False, None,
                    datetime.now().isoformat(), run_id,
                )
                self.checkpoint.mark_job_notified(scored.job.id)
                run.notifications_sent += 1

            self.checkpoint.complete_stage("notify")

            # Send health alerts for any errors
            if run.sites_failed or run.errors:
                self._send_health_alerts(run)

            # Complete
            run.status = "completed"
            run.completed_at = datetime.now().isoformat()
            run.llm_providers_used = {"primary": self.llm.last_provider_used or "none"}
            self.db.update_run(run)
            self.checkpoint.complete_run()

            logger.info(
                "pipeline_complete",
                run_id=run_id,
                jobs_scraped=run.jobs_scraped,
                jobs_new=run.jobs_new,
                jobs_scored=run.jobs_scored,
                matches=run.jobs_above_threshold,
                resumes=run.resumes_generated,
                notifications=run.notifications_sent,
            )

        except Exception as e:
            run.status = "crashed"
            run.completed_at = datetime.now().isoformat()
            run.errors.append({"stage": "pipeline", "error": str(e)})
            self.db.update_run(run)
            logger.exception("pipeline_crashed", run_id=run_id)
            if self.telegram:
                self.telegram.send_health_alert(f"Pipeline crashed: {e}")
            raise

        finally:
            self.db.close()

        return run

    def _poll_feedback(self) -> None:
        """Poll Telegram for feedback from previous notifications."""
        if not self.telegram:
            return
        try:
            feedback_items = self.telegram.poll_feedback()
            for item in feedback_items:
                score_data = self.db.get_score(item["job_id"])
                score = score_data["score"] if score_data else 0
                self.feedback_store.record(
                    job_id=item["job_id"],
                    score=score,
                    action=item["action"],
                )
        except Exception as e:
            logger.warning("feedback_poll_failed", error=str(e))

    def _send_health_alerts(self, run: PipelineRun) -> None:
        """Send health alerts for pipeline issues."""
        if not self.telegram:
            return

        parts = []
        if run.sites_failed:
            for err in run.sites_failed:
                parts.append(f"Site `{err['site']}` failed: {err['error']}")
        if run.errors:
            parts.append(f"{len(run.errors)} job processing errors")

        summary = f"Run {run.run_id}: {run.jobs_scraped} scraped, {run.jobs_new} new, {run.jobs_above_threshold} matches, {run.resumes_generated} resumes"
        parts.append(summary)

        self.telegram.send_health_alert("\n".join(parts))
```

**File:** `src/jobhunter/main.py`
**What:** Replace stub with actual entry point

```python
from __future__ import annotations

import sys

from jobhunter.pipeline import PipelineRunner


def main():
    """Run the JobHunter pipeline."""
    try:
        runner = PipelineRunner()
        run = runner.run()
        print(f"Pipeline complete: {run.jobs_above_threshold} matches, {run.resumes_generated} resumes generated")
    except FileNotFoundError as e:
        print(f"Configuration error: {e}", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"Pipeline failed: {e}", file=sys.stderr)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

**File:** `tests/test_checkpoint.py`
**What:** Test checkpoint manager

Test cases:
- Start run, complete stages, verify state
- Load checkpoint from disk
- Incomplete run detected on restart
- Job-level checkpoints (scored, tailored, notified)
- Complete run clears incomplete flag

**File:** `tests/test_pipeline.py`
**What:** Test pipeline orchestrator with mocked dependencies

Test cases (all dependencies mocked — scraper, LLM, DB, Telegram):
- Full pipeline run with mocked components
- Crash recovery: simulate crash after scoring, verify resume on restart
- No site configs logs warning and exits cleanly
- Site failure doesn't block other sites
- Health alerts sent on errors
- Score below threshold doesn't trigger tailoring
- Verification failure after retries skips resume but still notifies

#### Success Criteria

**Automated (run by agent):**
- [ ] `uv run pytest tests/test_checkpoint.py tests/test_pipeline.py -v` — all tests pass
- [ ] `uv run ruff check src/jobhunter/` — no lint errors
- [ ] `uv run ruff format --check src/jobhunter/` — no format errors
- [ ] `uv run pytest tests/ -v` — ALL tests pass (full regression)
- [ ] `uv run python -c "from jobhunter.pipeline import PipelineRunner; print('OK')"` — import works

**Manual (verified by human):**
- [ ] `uv run python -m jobhunter` prints a config error (expected — no master resume yet) rather than crashing with an import error

---

### Phase 8: Site Configs & Integration Testing

Create YAML configs for the 5 target sites and write integration tests. Each site config requires inspecting the actual site's structure.

**Implements:** FR-001, FR-030, US-12, AC-10

#### Changes Required

**Important:** The site configs below are **initial drafts** based on expected site structure. Each config must be validated by:
1. Inspecting the actual site (HTML structure, network requests)
2. Running a test scrape to verify selectors work
3. Adjusting selectors/URLs as needed

The implementing agent should use web search and/or direct HTTP requests to inspect each site's actual structure before writing the config.

**File:** `site_configs/bayt.yaml`
**What:** Bayt.com (UAE) — likely HTML strategy. Inspect https://www.bayt.com/en/uae/jobs/ for actual selectors.

**File:** `site_configs/gulftalent.yaml`
**What:** GulfTalent (UAE) — likely HTML or API strategy. Inspect https://www.gulftalent.com for actual structure.

**File:** `site_configs/undutchables.yaml`
**What:** Undutchables (Netherlands) — likely HTML strategy. Inspect https://undutchables.nl/jobs for actual selectors.

**File:** `site_configs/stepstone_de.yaml`
**What:** StepStone.de (Germany) — likely API strategy. Inspect https://www.stepstone.de for API endpoints in network tab.

**File:** `site_configs/berlin_startup_jobs.yaml`
**What:** Berlin Startup Jobs (Germany) — likely HTML strategy. Inspect https://berlinstartupjobs.com for actual selectors.

For each config, fill in:
- site_id, name, url, country
- strategy (api/html/browser — determined by inspection)
- Strategy-specific config (selectors, API endpoints, etc.)
- detail_page config (if needed for full descriptions)
- max_pages
- Any per-site keyword overrides

**File:** `tests/test_site_configs.py`
**What:** Validate all site configs load correctly

```python
from pathlib import Path
from jobhunter.config import load_site_configs

SITE_CONFIGS_DIR = Path(__file__).parent.parent / "site_configs"


def test_all_configs_load():
    """All YAML configs in site_configs/ load without error."""
    configs = load_site_configs(SITE_CONFIGS_DIR)
    # At least some configs should be present
    assert len(configs) >= 1


def test_all_configs_have_required_fields():
    """Each config has all required fields."""
    configs = load_site_configs(SITE_CONFIGS_DIR)
    for config in configs:
        assert config.site_id, f"Missing site_id in {config.name}"
        assert config.name, f"Missing name"
        assert config.url, f"Missing url in {config.name}"
        assert config.country in ("UAE", "Netherlands", "Germany"), f"Invalid country in {config.name}"
        assert config.strategy in ("api", "html", "browser"), f"Invalid strategy in {config.name}"
        assert config.max_pages >= 1, f"Invalid max_pages in {config.name}"

        # Strategy-specific config must exist
        if config.strategy == "api":
            assert config.api is not None, f"Missing api config in {config.name}"
        elif config.strategy == "html":
            assert config.html is not None, f"Missing html config in {config.name}"
        elif config.strategy == "browser":
            assert config.browser is not None, f"Missing browser config in {config.name}"
```

**File:** `tests/test_integration.py`
**What:** Integration test with mocked HTTP responses

```python
"""Integration tests for the full pipeline with mocked external calls.

These tests verify the pipeline stages work together correctly without
making real HTTP requests or LLM calls.
"""

# Test: Full pipeline with mocked scraper, LLM, and Telegram
# - Mock httpx responses for 2 sites with sample HTML
# - Mock LLM to return scores and tailored resume
# - Mock Telegram to capture sent messages
# - Verify: jobs scraped → deduped → filtered → scored → tailored → notified

# Test: Dedup across runs
# - Run pipeline with jobs A, B, C
# - Run again with jobs B, C, D
# - Verify only D is processed in second run

# Test: Crash recovery
# - Start pipeline, mock a crash during scoring
# - Restart pipeline
# - Verify already-scored jobs are not re-scored
```

#### Success Criteria

**Automated (run by agent):**
- [ ] `uv run pytest tests/test_site_configs.py -v` — all tests pass
- [ ] `uv run pytest tests/test_integration.py -v` — all tests pass
- [ ] `uv run pytest tests/ -v` — ALL tests pass (full regression)
- [ ] `uv run ruff check .` — no lint errors
- [ ] `uv run ruff format --check .` — no format errors

**Manual (verified by human):**
- [ ] Each site config corresponds to the actual site structure (verified by manual inspection)
- [ ] `uv run python -m jobhunter` runs without import errors (will fail on missing config/network, but should not crash on missing modules)

---

## Open Questions

None — all resolved during spec clarification.

## Notes for Implementing Agents

1. **Use Context7 MCP** to look up library documentation (httpx, BeautifulSoup, structlog, WeasyPrint, python-telegram-bot, Playwright) before implementing.
2. **Run `uv sync` after modifying `pyproject.toml`** to install new dependencies.
3. **Run `playwright install chromium`** after installing playwright dependency.
4. **All tests should use mocks for external services** (HTTP, LLM, Telegram, SMTP). No real network calls in tests.
5. **Delete the old `main.py`** in the project root once `src/jobhunter/main.py` replaces it.
6. The `...` method stubs in `db.py` must be fully implemented — they are not optional.
7. For Phase 8 site configs: actually inspect each site's HTML/API structure before writing selectors. Don't guess.
