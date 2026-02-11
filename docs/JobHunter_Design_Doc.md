# JobHunter: Automated Job Search Pipeline

## Problem

Job hunting across multiple countries is a full-time job. Every day: open 15 tabs, scroll through the same boards, skim 50 listings that don't match, copy-paste resume details into slightly different formats, lose track of what you applied to. The interesting jobs get buried in noise. The tailored resume never gets written because it takes 20 minutes per application. You either apply to everything with a generic CV (low conversion) or carefully tailor 3 per day (too slow, miss opportunities).

## Solution

A personal job search pipeline that watches niche job boards, scores matches against your profile, auto-tailors your resume, and pings you on Telegram/email when something worth applying to shows up. You teach it one site at a time — each site gets its own scraper config. The app does the grunt work. You do the applying.

No generic platforms at V1 (no LinkedIn, no Indeed — they fight scrapers and you already check them manually). Focus on the smaller, specialized boards where the signal-to-noise ratio is high and scraping is straightforward.

**V1 scope: Netherlands, Germany, UAE only.**

---

## Architecture Overview

```
                    ┌──────────────────────────────────────────────────────┐
                    │                    SCHEDULER                         │
                    │              (cron — every 4-6 hours)                │
                    └──────────────┬───────────────────────────────────────┘
                                   │
                    ┌──────────────▼───────────────────────────────────────┐
                    │                SCRAPER LAYER                         │
                    │    site_configs/                                     │
                    │      ├── bayt.*                                     │
                    │      ├── gulftalent.*                               │
                    │      ├── undutchables.*                             │
                    │      ├── stepstone_de.*                             │
                    │      └── ...                                        │
                    │                                                     │
                    │    For each site: fetch → parse → normalize         │
                    └──────────────┬───────────────────────────────────────┘
                                   │
                          [raw job listings]
                                   │
                    ┌──────────────▼───────────────────────────────────────┐
                    │              CHECKPOINT: raw data saved to disk      │
                    │              (never lose scraped data, even on crash)│
                    └──────────────┬───────────────────────────────────────┘
                                   │
                    ┌──────────────▼───────────────────────────────────────┐
                    │               DEDUP ENGINE                          │
                    │    Hash(title + company + location) → skip if seen  │
                    └──────────────┬───────────────────────────────────────┘
                                   │
                          [new listings only]
                                   │
                    ┌──────────────▼───────────────────────────────────────┐
                    │              MATCHING ENGINE                         │
                    │                                                     │
                    │    Pass 1: Keyword filter (cheap, instant)           │
                    │            → kill obvious mismatches                 │
                    │                                                     │
                    │    Pass 2: LLM scoring (per surviving listing)       │
                    │            → score 0-100 + reasoning                 │
                    └──────────────┬───────────────────────────────────────┘
                                   │
                          [scored listings, threshold configurable]
                                   │
                    ┌──────────────▼───────────────────────────────────────┐
                    │              RESUME TAILOR                           │
                    │                                                     │
                    │    LLM call 1: Generate tailored resume (HTML)       │
                    │    LLM call 2: Verify (no fabrication, good match)   │
                    │    Convert: HTML → PDF                               │
                    └──────────────┬───────────────────────────────────────┘
                                   │
                          [job + score + tailored resume PDF]
                                   │
                    ┌──────────────▼───────────────────────────────────────┐
                    │              NOTIFIER                                │
                    │    Telegram bot message + email                      │
                    │    Includes: job link, score, reasoning,             │
                    │              tailored resume PDF attached            │
                    └──────────────┬───────────────────────────────────────┘
                                   │
                          [user sees notification]
                                   │
                    ┌──────────────▼───────────────────────────────────────┐
                    │              FEEDBACK + TRACKER                      │
                    │    User reacts: applied / skipped / not relevant     │
                    │    Tracker logs: company, role, date, resume used    │
                    │    Feedback trains future scoring                    │
                    └─────────────────────────────────────────────────────┘
```

---

## Scraper Layer

### The Hard Part (and why it's manageable)

Scraping is the hardest module — not because parsing HTML is hard, but because every site is different. The solution: **one config file per site**, each describing how to fetch and extract listings from that specific board.

Three scraping strategies, tried in order of preference:

```
1. API        — site has a public/undocumented API → use it (cleanest, most reliable)
2. HTML Parse — static HTML rendered server-side → fetch + parse
3. Browser    — JS-rendered content → headless browser (heaviest, last resort)
```

For V1 target sites, most fall into category 1 or 2. The big platforms (LinkedIn, Indeed) that would need category 3 + anti-bot evasion are explicitly out of scope.

### Site Config

Each site gets a config file that teaches the app how to scrape it. The config defines: where to fetch, how to find job cards, how to extract fields, and how to paginate.

**Config format — to be decided:**

```
┌────────┬──────────────────────────────────────────────────────────────┐
│ YAML   │ Human-readable, clean for nested structures, comments.      │
│        │ Requires PyYAML or ruamel.yaml.                             │
├────────┼──────────────────────────────────────────────────────────────┤
│ JSON   │ No dependencies, native Python support. No comments.        │
│        │ Verbose for nested configs but universally understood.      │
├────────┼──────────────────────────────────────────────────────────────┤
│ TOML   │ Good for flat configs, Python 3.11+ has built-in support.   │
│        │ Less ideal for deeply nested structures.                    │
├────────┼──────────────────────────────────────────────────────────────┤
│ Python │ Maximum flexibility, no parsing needed. Configs are code.   │
│        │ Harder to validate, easier to break.                        │
└────────┴──────────────────────────────────────────────────────────────┘
```

**What the config needs to contain (regardless of format):**

```
Site Config:
  site_id            unique identifier ("bayt", "gulftalent", etc.)
  name               display name
  url                base URL
  country            which country this board covers
  enabled            on/off switch

  strategy           "api" | "html" | "browser"

  # If API strategy:
  api config         base URL, method, params, headers, auth, response path, pagination

  # If HTML strategy:
  html config        list page URL, job card CSS selector, pagination method,
                     field selectors (title, company, location, url, salary, date)

  # If browser strategy:
  browser config     load URL, wait-for selector, scroll behavior,
                     then same field selectors as HTML

  # Optional detail page:
  detail page        whether to follow each job URL for full description,
                     and how to extract description from that page

  # Keyword pre-filter:
  keywords           must_have_any, must_not_have, title_must_have_any
```

### Adding a New Site

1. Open the site in browser, inspect the HTML/network tab
2. Copy the template config, fill in selectors
3. Run test mode to validate
4. Enable it

No code changes needed. The scraper engine reads all configs and runs them.

### Normalized Job Schema

Every site's raw data gets normalized into this common format before storage:

```
Job:
  id: string              auto-generated hash(site_id + title + company + location)
  site_id: string         "bayt"
  title: string           "Senior Cloud Engineer"
  company: string         "Careem"
  location: string        "Dubai, UAE"
  country: string         "UAE"
  url: string             full URL to the listing
  salary: string | null   raw text, not parsed
  description: string     full job description text
  requirements: string    requirements section if separate
  posted_date: date|null  when the job was posted
  scraped_at: datetime    when we found it
  is_new: bool            true if first time seeing this listing
```

---

## Checkpoint System

Raw data is the most expensive thing to re-acquire (network calls, rate limits, sites going down). The pipeline saves state at every critical boundary so a crash never loses work.

### Checkpoint Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CHECKPOINT POINTS                            │
│                                                                     │
│  CP1: After scraping each site                                      │
│       → save raw HTML/JSON response to disk immediately             │
│       → even if parsing fails, you have the raw data to retry       │
│                                                                     │
│  CP2: After normalization                                           │
│       → save normalized job list to DB                              │
│       → if dedup/matching crashes, re-process from DB not network   │
│                                                                     │
│  CP3: After LLM scoring                                             │
│       → save score + reasoning per job                              │
│       → if resume tailoring crashes, don't re-score                 │
│                                                                     │
│  CP4: After resume generation                                       │
│       → save HTML + PDF to disk                                     │
│       → if notification fails, resume is already built              │
│                                                                     │
│  CP5: After notification sent                                       │
│       → mark job as "notified" in DB                                │
│       → prevents duplicate notifications on re-run                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Raw Data Storage

```
data/
├── raw/
│   ├── 2026-02-11/
│   │   ├── bayt_page1.html          # raw HTTP response body
│   │   ├── bayt_page2.html
│   │   ├── gulftalent_page1.html
│   │   └── undutchables_page1.html
│   ├── 2026-02-12/
│   │   └── ...
│   └── ...
├── checkpoints/
│   ├── last_run.json                # which stage completed, which jobs processed
│   └── pending_scores.json          # jobs that passed filter but haven't been scored
└── jobhunter.db                     # main database
```

### Resume on Crash

```
Pipeline starts:
  1. Check last_run.json
  2. If previous run didn't complete:
     → identify which stage crashed
     → resume from the last completed checkpoint
     → don't re-scrape sites that already completed
     → don't re-score jobs that already have scores
  3. If previous run completed normally:
     → start fresh run
```

This means: if the LLM scoring crashes halfway through 12 jobs (6 scored, 6 remaining), the next run picks up from job 7. No wasted LLM calls, no re-scraping.

---

## Dedup Engine

Simple and effective. Before any listing enters the matching pipeline:

```
1. Compute hash = sha256(normalize(title) + normalize(company) + normalize(location))
2. Check if hash exists in seen_jobs table
3. If yes → skip (already processed)
4. If no  → insert hash + proceed to matching
```

`normalize()` lowercases, strips whitespace, removes punctuation. This catches: same job posted on multiple boards, same job re-scraped on next run, slightly different formatting of same listing.

---

## Matching Engine

Two passes. The first is free and instant. The second costs an LLM call.

### Pass 1: Keyword Filter

Uses the keyword config per site. Three checks:

```
1. Job text must contain at least one relevant keyword (cloud, api, apigee, etc.)
2. Job text must NOT contain dealbreakers (intern, junior, student, etc.)
3. Job title must signal the right role level (engineer, architect, lead, etc.)
```

This kills 60-80% of listings before any LLM call.

### Pass 2: LLM Scoring

Each surviving listing gets scored by the LLM via `claude-max-api-proxy`. The prompt includes the full candidate profile, the job description, and scoring criteria (technical match, seniority match, role relevance, visa feasibility, growth potential). Returns a JSON score 0-100 with reasoning.

**Threshold** is configurable. Suggested starting point: score >= 60 proceeds to resume tailoring. Tune based on feedback.

### Feedback Integration

Over time, user feedback adjusts the system:

```
User marks job as "not relevant" → system logs which keywords/patterns to penalize
User marks job as "applied"      → system logs what a good match looks like
User marks job as "skipped"      → neutral, no adjustment
```

V1: Feedback stored in DB. Manual review of patterns. No auto-retraining yet — just data collection. V2: feed aggregated feedback patterns into the LLM scoring prompt.

---

## Resume Tailor

The killer feature. For each job scoring above threshold, automatically generate a tailored resume.

### Master Resume

A comprehensive file containing ALL your experience, skills, and achievements — more than would ever fit on one resume. This is the source of truth the LLM draws from.

**Format — to be decided:**

```
┌──────────┬──────────────────────────────────────────────────────────┐
│ YAML     │ Readable, supports nested structures well.              │
│          │ Natural for tagged achievements.                        │
├──────────┼──────────────────────────────────────────────────────────┤
│ JSON     │ Native Python support. Easy to validate with schemas.   │
│          │ More verbose but strict structure.                       │
├──────────┼──────────────────────────────────────────────────────────┤
│ Markdown │ Most human-friendly to write and maintain.              │
│          │ Harder to parse programmatically for structured data.   │
└──────────┴──────────────────────────────────────────────────────────┘
```

**What it needs to contain:**

```
Master Resume:
  personal info       name, email, phone, linkedin, location, languages
  summary variants    multiple versions for different role types
  experience          all roles, with achievements tagged by skill category
  certifications      all certs with tags
  education           all degrees
  skills pool         every skill, grouped by category (cloud, api, devops, etc.)
```

### Tailoring Flow

```
                    ┌───────────────────────────────┐
                    │        Master Resume           │
                    └───────────────┬───────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │         Job Description        │
                    └───────────────┬───────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │      LLM Call 1: TAILOR        │
                    │                               │
                    │  "Given this master resume     │
                    │   and this job description,    │
                    │   produce a tailored resume.   │
                    │   Emphasize relevant skills,   │
                    │   reorder experience, pick     │
                    │   the right summary variant.   │
                    │   Output: HTML"                │
                    └───────────────┬───────────────┘
                                    │
                              [tailored HTML]
                                    │
                    ┌───────────────▼───────────────┐
                    │      LLM Call 2: VERIFY        │
                    │                               │
                    │  "Compare this tailored resume │
                    │   against the master resume.   │
                    │   Flag any fabricated skills,   │
                    │   inflated claims, or missing  │
                    │   critical info. Return JSON   │
                    │   with pass/fail + issues."    │
                    └───────────────┬───────────────┘
                                    │
                          [pass or fail + issues]
                                    │
                    ┌───────────────▼───────────────┐
                    │   If pass → HTML → PDF          │
                    │   If fail → fix issues → retry   │
                    └────────────────────────────────┘
```

### Why HTML → PDF?

- Full CSS control over styling (fonts, spacing, colors, layout)
- Easy to generate from an LLM (just structured HTML)
- Template stays consistent, only content changes
- Can preview in browser before converting

**HTML → PDF converter — to be decided:**

```
┌──────────────┬─────────────────────────────────────────────────────────┐
│ WeasyPrint   │ Pure Python, pip install. Good CSS support.            │
│              │ Handles flexbox, grid. Active project.                 │
├──────────────┼─────────────────────────────────────────────────────────┤
│ wkhtmltopdf  │ CLI tool, wraps WebKit. Very mature.                   │
│              │ Great CSS support but external binary dependency.      │
├──────────────┼─────────────────────────────────────────────────────────┤
│ Playwright   │ Full browser rendering, best CSS fidelity.             │
│              │ Heavier dependency (downloads Chromium).               │
│              │ Already needed if using browser scraping strategy.     │
├──────────────┼─────────────────────────────────────────────────────────┤
│ pdfkit       │ Python wrapper around wkhtmltopdf. Simpler API.        │
│              │ Same external dependency as wkhtmltopdf.               │
└──────────────┴─────────────────────────────────────────────────────────┘
```

---

## Notification Layer

### Telegram Bot

Primary notification channel. A simple bot created via BotFather.

```
Message format:

Match Score: 82/100

Senior Cloud Engineer — Booking.com
Amsterdam, Netherlands
EUR 75,000 - 90,000

Why it matched:
"Strong Apigee X + GCP alignment. Role requires API gateway
migration which is core to candidate's experience. Booking.com
is an IND-recognized HSM sponsor."

Concerns:
"Listing mentions Dutch preferred but not required"

[Tailored Resume attached]
[View Listing](https://...)

React:
  Applied  |  Skip  |  Not Relevant
```

User reacts with an inline button → feedback stored in DB.

### Email (Secondary)

Daily digest email grouping all matches from the last 24 hours. Same info as Telegram but in a digest format. Resume PDFs attached.

### Notification Rules (configurable)

```
Score 80+  → Telegram instant + include in daily email
Score 60-79 → Daily email digest only
Score 40-59 → Logged, no notification (reviewable later)
Score < 40  → Discarded after logging
```

Thresholds and routing are configurable.

---

## Application Tracker

### Schema

```
Application:
  id: string
  job_id: string            → FK to Job
  company: string
  role: string
  country: string
  applied_date: date
  resume_version: string    path to the tailored resume PDF used
  status: enum
    "matched"               → system found it, pending user action
    "applied"               → user applied
    "phone_screen"          → got a callback
    "interview"             → interview scheduled/completed
    "offer"                 → received offer
    "rejected"              → rejected at any stage
    "withdrawn"             → user withdrew
    "expired"               → listing disappeared before applying
  status_updated: datetime
  notes: string             free-text user notes
  source_site: string       which board it came from

CompanyLog:
  company: string
  country: string
  applications: [Application]
  notes: string
```

### Duplicate Application Guard

Before sending a notification, check if you've already applied to the same role at the same company. If you applied to the same company but for a different role, flag it in the notification — different tailored resumes for different roles at the same company is perfectly normal and expected.

---

## Feedback Loop

### Data Collection (V1)

Every notification gets a user reaction:

```
Applied      → positive signal, strong match
Skip         → neutral, maybe timing/mood
Not Relevant → negative signal, bad match — prompt user for 1-word reason
```

"Not Relevant" triggers a follow-up: "Why? [Wrong role] [Wrong level] [Bad company] [Location] [Salary] [Other]"

### Storage

```
Feedback:
  job_id: string
  score: int               what the LLM gave it
  action: enum             applied | skipped | not_relevant
  reason: string | null    why it wasn't relevant
  timestamp: datetime
```

### Usage (V1 — manual)

After 2-3 weeks of data: review feedback patterns, adjust keyword filters, tune score thresholds, identify blind spots.

### Usage (V2 — future)

Feed aggregated feedback patterns into the LLM scoring prompt automatically.

---

## Error Handling

### Philosophy

The pipeline must be **resilient by default**. A failure in one site, one job, or one LLM call should never kill the entire run. Every error is logged with enough context to diagnose and fix without re-running.

### Error Handling Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                     LAYER 1: SITE-LEVEL ISOLATION                   │
│                                                                     │
│  Each site scraper runs independently. If Bayt crashes, GulfTalent  │
│  and Undutchables still complete. Failed sites are logged and       │
│  retried on the next scheduled run.                                 │
│                                                                     │
│  On site failure:                                                   │
│    1. Log: site_id, error type, stack trace, HTTP status, timestamp │
│    2. Save partial data if any pages succeeded (checkpoint)         │
│    3. Continue to next site                                         │
│    4. Alert via Telegram: "Scraper failed for {site_id}: {error}"  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     LAYER 2: JOB-LEVEL ISOLATION                    │
│                                                                     │
│  Each job is processed independently through the pipeline. If one   │
│  job's detail page fails or its LLM scoring returns garbage, the    │
│  other jobs continue processing normally.                           │
│                                                                     │
│  On job processing failure:                                         │
│    1. Log: job_id, stage (scrape/score/tailor), error, raw data    │
│    2. Mark job as "error" in DB with error details                  │
│    3. Continue to next job                                          │
│    4. Errored jobs can be retried manually or on next run           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     LAYER 3: LLM CALL RESILIENCE                    │
│                                                                     │
│  LLM calls are the most failure-prone component (rate limits,       │
│  proxy crashes, malformed responses, timeouts).                     │
│                                                                     │
│  Fallback chain per call:                                           │
│    1. claude-max-api-proxy (primary)                                │
│       ↓ on failure (timeout, 429, 500, connection refused)          │
│    2. OpenRouter free tier                                          │
│       ↓ on failure                                                  │
│    3. Skip this job, log it, move on                                │
│       (for scoring — never send a resume without LLM quality)       │
│                                                                     │
│  On malformed LLM response (bad JSON, missing fields):             │
│    1. Retry same provider once with "respond only with valid JSON"  │
│    2. If still bad → try next provider in chain                     │
│    3. If all fail → log raw response for debugging, skip job        │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     LAYER 4: NOTIFICATION RESILIENCE                │
│                                                                     │
│  If Telegram fails, try email. If email fails, the job is still     │
│  marked as scored/tailored in the DB — it won't be lost.            │
│                                                                     │
│  Fallback chain:                                                    │
│    1. Telegram instant message                                      │
│       ↓ on failure                                                  │
│    2. Email                                                         │
│       ↓ on failure                                                  │
│    3. Log to "unsent_notifications" table                           │
│       → retry on next run                                           │
│       → or review manually                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Logging

Every pipeline run produces a structured log with enough detail to understand exactly what happened:

```
Log entry per run:
  run_id             unique ID for this pipeline execution
  started_at         timestamp
  completed_at       timestamp (or null if crashed)
  sites_attempted    ["bayt", "gulftalent", "undutchables"]
  sites_succeeded    ["bayt", "undutchables"]
  sites_failed       [{"site": "gulftalent", "error": "HTTP 403", "stage": "fetch"}]
  jobs_scraped       42
  jobs_new           8
  jobs_filtered_out  5
  jobs_scored        3
  jobs_above_threshold  2
  resumes_generated  2
  notifications_sent 2
  errors             [{job_id, stage, error, timestamp}]
  llm_provider_used  {"scoring": "proxy", "tailoring": "proxy"}  # or "openrouter" if fallback

Log entry per job:
  job_id             hash
  site_id            where it came from
  title, company     identifying info
  stages_completed   ["scraped", "normalized", "deduped", "filtered", "scored"]
  score              72
  resume_generated   true/false
  notified           true/false
  error              null or {stage, message, stack_trace}
```

**Logging library — to be decided:**

```
┌─────────────────┬──────────────────────────────────────────────────────┐
│ Python logging  │ Built-in, zero dependencies. Standard approach.     │
│                 │ Configurable handlers (file, stdout, etc.)          │
├─────────────────┼──────────────────────────────────────────────────────┤
│ structlog       │ Structured logging (JSON output). Great for parsing │
│                 │ logs programmatically. More queryable.              │
├─────────────────┼──────────────────────────────────────────────────────┤
│ loguru          │ Simple API, pretty console output, file rotation.   │
│                 │ Popular for personal projects. One extra dependency.│
└─────────────────┴──────────────────────────────────────────────────────┘
```

### Alerting

Pipeline health alerts go to Telegram (same bot, different message format):

```
Pipeline alert examples:

"Run completed: 8 new jobs, 2 matches, 0 errors"

"WARNING: GulfTalent scraper failed (HTTP 403). Other sites OK.
  2 matches found from Bayt and Undutchables."

"ERROR: LLM proxy unreachable. Fell back to OpenRouter for 3 scoring calls.
  Resume tailoring skipped (quality too important for fallback).
  3 jobs queued for tailoring on next run."

"CRITICAL: All sites failed. Check internet/config."
```

---

## Tech Stack (Options)

Most choices are left open for implementation time. Here are the options for each component:

### Language

Python 3.11+ (settled — best library ecosystem for scraping + LLM + PDF generation)

### Scraping

```
┌───────────────────┬────────────────────────────────────────────────────┐
│ HTTP Client       │                                                    │
├───────────────────┼────────────────────────────────────────────────────┤
│ httpx             │ Async support, modern API, HTTP/2. Active project.│
│ requests          │ Most popular, simplest API. No async.             │
│ aiohttp           │ Async-native. Good if pipeline is fully async.    │
├───────────────────┼────────────────────────────────────────────────────┤
│ HTML Parser       │                                                    │
├───────────────────┼────────────────────────────────────────────────────┤
│ BeautifulSoup     │ Most popular, forgiving parser. Slower.           │
│ selectolax        │ Fastest, CSS selectors only. Less forgiving.      │
│ lxml              │ Fast, supports XPath + CSS. C dependency.         │
│ parsel            │ Scrapy's parser. XPath + CSS. Good middle ground. │
├───────────────────┼────────────────────────────────────────────────────┤
│ Browser (if needed)│                                                   │
├───────────────────┼────────────────────────────────────────────────────┤
│ Playwright        │ Modern, multi-browser, async. Chromium download.  │
│ Selenium          │ Older, more docs/examples. WebDriver dependency.  │
└───────────────────┴────────────────────────────────────────────────────┘
```

### LLM

```
┌────────────────────────┬───────────────────────────────────────────────┐
│ claude-max-api-proxy   │ PRIMARY. Uses Claude subscription. $0 cost.  │
│                        │ OpenAI-compatible API on localhost.           │
│                        │ github.com/atalovesyou/claude-max-api-proxy  │
├────────────────────────┼───────────────────────────────────────────────┤
│ OpenRouter free tier   │ FALLBACK. Free credits, multiple models.     │
│                        │ Standard API.                                │
├────────────────────────┼───────────────────────────────────────────────┤
│ Ollama (local)         │ EMERGENCY FALLBACK. Llama 3 / Mistral.      │
│                        │ Scoring only — not reliable enough for       │
│                        │ resume tailoring.                            │
└────────────────────────┴───────────────────────────────────────────────┘
```

### Notifications

```
┌─────────────────────────┬─────────────────────────────────────────────┐
│ Telegram                │                                             │
├─────────────────────────┼─────────────────────────────────────────────┤
│ python-telegram-bot     │ Most popular, feature-rich. Async.         │
│ aiogram                 │ Async-native, modern API. Less docs.       │
│ telepot                 │ Simple, synchronous. Minimal.              │
├─────────────────────────┼─────────────────────────────────────────────┤
│ Email                   │                                             │
├─────────────────────────┼─────────────────────────────────────────────┤
│ smtplib                 │ Built-in Python. Zero dependencies.        │
│ resend                  │ Modern email API. Free tier: 100 emails/day│
│ SendGrid                │ Free tier: 100 emails/day. More features.  │
└─────────────────────────┴─────────────────────────────────────────────┘
```

### Storage

```
┌───────────────────┬────────────────────────────────────────────────────┐
│ SQLite            │ Zero setup, single file, no server. Backed up by  │
│                   │ copying one file. Enough for personal use.        │
├───────────────────┼────────────────────────────────────────────────────┤
│ DuckDB            │ SQLite-like but columnar. Better for analytics.   │
│                   │ Overkill for V1 but interesting if you want to    │
│                   │ run queries on job market trends later.           │
├───────────────────┼────────────────────────────────────────────────────┤
│ JSON files        │ Simplest possible. No queries, just read/write.   │
│                   │ Works for checkpoints, not great for tracking.    │
└───────────────────┴────────────────────────────────────────────────────┘
```

### Scheduler

```
┌───────────────────┬────────────────────────────────────────────────────┐
│ cron (system)     │ Native on any Linux/Mac. Simple, reliable.        │
├───────────────────┼────────────────────────────────────────────────────┤
│ APScheduler       │ Python library. In-process scheduling.            │
│                   │ Good if app runs as a long-lived process.         │
├───────────────────┼────────────────────────────────────────────────────┤
│ GitHub Actions    │ Free, no server needed. Stateless (need external  │
│                   │ storage). Good for prototyping.                   │
└───────────────────┴────────────────────────────────────────────────────┘
```

---

## LLM Call Strategy

### Claude Max API Proxy (Subscription-Powered)

Instead of paying for API keys, we use the `claude-max-api-proxy` — an open-source tool that wraps the authenticated Claude Code CLI and exposes a standard OpenAI-compatible HTTP API. This lets the app make proper API calls using your existing Claude subscription.

**How it works:**

```
Your app                 claude-max-api-proxy            Anthropic
   │                            │                           │
   ├── POST /v1/chat ──────────►│                           │
   │   (standard OpenAI format) │                           │
   │                            ├── (uses OAuth token  ────►│
   │                            │    from subscription)     │
   │                            │                           │
   │                            │◄── response ──────────────┤
   │◄── JSON response ─────────┤                           │
   │   (standard format)        │                           │
```

**Setup (one-time):**

```bash
# 1. Ensure Claude Code CLI is authenticated
claude auth login

# 2. Clone and run the proxy
git clone https://github.com/atalovesyou/claude-max-api-proxy
cd claude-max-api-proxy
npm install
npm start
# Proxy now running on http://localhost:3456/v1
```

**Usage in Python (via standard OpenAI SDK):**

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:3456/v1",
    api_key="not-needed"    # proxy handles auth via subscription
)

response = client.chat.completions.create(
    model="claude-sonnet-4-5-20250929",
    messages=[{"role": "user", "content": "Score this job..."}],
    response_format={"type": "json_object"}
)
```

Clean, standard API calls. Structured JSON responses. No API key costs.

**Note:** This is a community tool (github.com/atalovesyou/claude-max-api-proxy), not officially supported by Anthropic. It works by wrapping the CLI's OAuth authentication. For a personal tool, this risk is acceptable — worst case, you switch to API keys or OpenRouter.

### Call Distribution

```
                         ┌─────────────────────────┐
                         │   ~30 new listings/day   │
                         │   (3 countries, ~5 sites) │
                         └────────────┬────────────┘
                                      │
                              keyword filter (free)
                                      │
                         ┌────────────▼────────────┐
                         │    ~8 survive filter     │
                         └────────────┬────────────┘
                                      │
                          LLM scoring (via proxy)
                            ~8 calls × ~500 tokens
                                      │
                         ┌────────────▼────────────┐
                         │   ~3 score above threshold│
                         └────────────┬────────────┘
                                      │
                         resume tailoring (via proxy)
                           ~3 calls × ~2000 tokens
                                      │
                         resume verification (via proxy)
                           ~3 calls × ~1000 tokens
                                      │
                         ┌────────────▼────────────┐
                         │  ~2-3 notifications/day  │
                         └─────────────────────────┘
```

**Daily LLM cost: $0** — all calls go through your Claude subscription via the proxy.

**Fallback chain:**
1. Claude subscription via proxy (primary — free)
2. OpenRouter free tier (if proxy is down)
3. Local Ollama (if everything is down — scoring only, not resume tailoring)

---

## Hosting Options (Free Tier)

```
┌───────────────────┬──────────────┬───────────────┬─────────────────────────────┐
│ Option            │ Cost         │ Cron Support  │ Notes                       │
├───────────────────┼──────────────┼───────────────┼─────────────────────────────┤
│ GitHub Actions    │ Free         │ Yes (schedule) │ 2000 min/month free.        │
│                   │              │               │ Stateless — need external   │
│                   │              │               │ storage for DB. Good for    │
│                   │              │               │ prototyping.                │
├───────────────────┼──────────────┼───────────────┼─────────────────────────────┤
│ Oracle Cloud VM   │ Free forever │ Yes (crontab) │ 1 GB RAM, 1 OCPU.          │
│ (always-free)     │              │               │ Real VM, real cron, real    │
│                   │              │               │ filesystem. Can also host   │
│                   │              │               │ the LLM proxy on same box. │
├───────────────────┼──────────────┼───────────────┼─────────────────────────────┤
│ Railway           │ Free trial   │ Yes (cron)    │ $5 credit/month on free     │
│                   │ then $5/mo   │               │ plan. Easy deploy. Good DX. │
├───────────────────┼──────────────┼───────────────┼─────────────────────────────┤
│ Render            │ Free tier    │ Background    │ Free web services spin down │
│                   │              │ workers       │ after 15 min idle. Cron via │
│                   │              │               │ external trigger needed.    │
├───────────────────┼──────────────┼───────────────┼─────────────────────────────┤
│ Raspberry Pi      │ One-time     │ Yes (crontab) │ At home, always on, full    │
│ (at home)         │ ~$50-80      │               │ control. No vendor lock-in. │
│                   │              │               │ Needs stable internet.      │
├───────────────────┼──────────────┼───────────────┼─────────────────────────────┤
│ Vercel            │ Free tier    │ Cron (limited)│ Serverless functions.       │
│                   │              │               │ Good for notification       │
│                   │              │               │ endpoints, less ideal for   │
│                   │              │               │ long-running scrape jobs.   │
└───────────────────┴──────────────┴───────────────┴─────────────────────────────┘
```

---

## Directory Structure (suggested, not final)

```
jobhunter/
├── config/
│   ├── settings.*             # global settings (thresholds, notification prefs)
│   └── master_resume.*        # comprehensive resume data
├── site_configs/
│   ├── _template.*            # blank template for adding new sites
│   ├── bayt.*                 # UAE
│   ├── gulftalent.*           # UAE
│   ├── undutchables.*         # Netherlands
│   ├── stepstone_de.*         # Germany
│   └── ...                    # more as added
├── src/
│   ├── scraper                # reads site configs, fetches + parses listings
│   ├── dedup                  # hash-based deduplication
│   ├── matcher                # keyword filter + LLM scoring
│   ├── tailor                 # resume tailoring + verification
│   ├── notifier               # telegram + email notifications
│   ├── tracker                # application tracking
│   ├── feedback               # feedback collection + storage
│   ├── llm                    # LLM abstraction (proxy / openrouter / ollama)
│   ├── checkpoint             # checkpoint save/resume logic
│   └── db                     # database operations
├── templates/
│   └── resume.html            # HTML resume template (CSS included)
├── output/
│   └── resumes/               # generated tailored resume PDFs
├── data/
│   ├── raw/                   # raw scraped HTML/JSON by date
│   ├── checkpoints/           # pipeline state for crash recovery
│   └── jobhunter.db           # main database (or whatever storage chosen)
├── logs/                      # structured pipeline logs
├── main                       # orchestrator — runs the full pipeline
└── requirements.txt
```

### Module Independence

Each module is independent and testable on its own. You can test just the scraper for one site, or generate a tailored resume from a saved job description, or send a test notification — without running the full pipeline. This is ideal for building module by module with Claude Code.

---

## V1 Target Sites

```
┌──────────────┬───────────────────────┬────────────────────────────────────┐
│ Country      │ Site                  │ Notes                              │
├──────────────┼───────────────────────┼────────────────────────────────────┤
│ UAE          │ Bayt.com              │ Largest MENA job board             │
│ UAE          │ GulfTalent            │ Premium Gulf roles                 │
│ Netherlands  │ Undutchables          │ Expat-focused, HSM visa roles      │
│ Germany      │ StepStone.de          │ Largest German board               │
│ Germany      │ Berlin Startup Jobs   │ English-friendly startup roles     │
└──────────────┴───────────────────────┴────────────────────────────────────┘
```

Start with 2-3 sites. Add more as confidence grows. Other countries (Japan, Ireland, Canada, Saudi Arabia) are future expansions.

---

## Pipeline Run: Step by Step

```
T=0s     Cron triggers main
         │
T=0s     Check last_run checkpoint — resume if previous run crashed
         │
T=0s     Load all enabled site configs
         │
T=1s     For each site (parallel where possible):
         │  ├── Fetch listing pages
         │  ├── CHECKPOINT: save raw response to data/raw/
         │  ├── Parse job cards
         │  ├── Fetch detail pages (if configured)
         │  ├── CHECKPOINT: save raw detail pages
         │  ├── Normalize into Job schema
         │  └── On error: log, alert, continue to next site
         │
T=15s    All sites scraped. ~30 raw listings collected.
         │
T=15s    CHECKPOINT: all normalized jobs saved to DB
         │
T=16s    Dedup: Check hashes against seen_jobs table.
         │
T=16s    Keyword filter.
         │
T=17s    LLM scoring via proxy (with fallback chain):
         │       On proxy fail → try OpenRouter → skip + log
         │       CHECKPOINT: save each score as it completes
         │
T=40s    Scoring complete.
         │
T=40s    Resume tailoring for above-threshold jobs:
         │       On proxy fail → queue for next run (don't use fallback for quality)
         │       CHECKPOINT: save each HTML + PDF as it completes
         │
T=80s    Send notifications (Telegram → Email → unsent_notifications table)
         │
T=85s    CHECKPOINT: mark run as complete
         │
         Pipeline complete. Next run in 4-6 hours.
```

---

## Edge Cases

### Site goes down or changes layout
Scraper catches HTTP errors and parsing failures. Logs the error, skips the site, continues with others. Telegram alert sent. Raw response saved to disk regardless (for manual inspection or re-parsing after fix).

### LLM returns garbage
Verify step catches malformed JSON. Retry once with stricter prompt. If still bad, try next provider in fallback chain. If all fail, skip the job and log the raw response for debugging.

### Same job, different titles
"Cloud Engineer" vs "Cloud Infrastructure Engineer" at the same company. Dedup won't catch this (different hash). The tracker will flag prior interactions with the same company.

### Claude subscription rate limits
Proxy returns a rate-limit error. App falls back to OpenRouter for scoring. Resume tailoring is queued for the next run (quality too important for fallback models). Telegram alert sent.

### Proxy goes down
The `llm` module catches the connection error and falls through to OpenRouter. Telegram alert: "LLM proxy down — using fallback. Check auth." Resume tailoring queued (not generated on fallback).

### Job listing disappears before you apply
The scraper stores the full description at scrape time (checkpoint). Even if the listing is taken down, you have the data. The notification includes the full description.

### Pipeline crashes mid-run
Checkpoint system handles this. On next run, the pipeline detects incomplete state and resumes from the last completed checkpoint. No re-scraping, no re-scoring of already-processed jobs.

### Network failure during scraping
Each site is independent. If the network drops during site 3 of 5, sites 1 and 2 are already checkpointed. Sites 3-5 retry on next run.

---

## Open Questions / Future Work

- **LinkedIn integration**: Parse LinkedIn job alert emails instead of scraping the site directly? Email arrives → extract job data → run through pipeline. Avoids all anti-bot issues.
- **Indeed integration**: Same email-parsing approach. Or use Indeed's API if approved.
- **More countries (Japan, Ireland, Canada, Saudi Arabia)**: Add site configs as V1 stabilizes.
- **Company intelligence layer**: Auto-pull Glassdoor rating, IND sponsor list (NL), check if company has sponsored visas before.
- **Dashboard**: Simple web UI to browse matches, manage tracker, review feedback.
- **Cover letter generation**: Same LLM approach as resume tailoring but with a cover letter template.
- **Salary parsing**: Parse salary strings into numbers, convert currencies, compare against your target. Flag underpaying roles.
- **Multi-language resume support**: German CV format, Japanese Rirekisho — may need separate templates per country.
- **Auto-apply (maybe never)**: Ethically questionable and likely to get flagged. The value is in the tailored resume + notification, not in automating the click.
