# Spec 0001: Job Search Pipeline V1
**Date:** 2026-02-12
**Status:** Draft
**Branch:** feature/0001-job-search-pipeline-v1

## Problem

Job hunting across Netherlands, Germany, and UAE requires daily manual checking of multiple niche job boards, skimming dozens of irrelevant listings, and spending 20+ minutes per application tailoring a resume. This results in either low-conversion generic applications or too-few carefully tailored ones. An automated pipeline that watches these boards, scores matches against a candidate profile, auto-tailors resumes, and sends notifications would eliminate the grunt work and let the user focus only on applying to high-quality matches.

## User Stories

### US-1: Scrape Job Listings from Configured Sites (P1)
**As a** job seeker
**I want to** have niche job boards automatically scraped on a schedule
**So that** I never miss a new listing without manually checking each site
**Independent test:** Run the scraper for a single configured site and verify it returns normalized job listings with all required fields.

### US-2: Avoid Processing Duplicate Listings (P1)
**As a** job seeker
**I want to** never see the same job listing twice
**So that** I only spend time on genuinely new opportunities
**Independent test:** Feed the same listing twice and verify the second is skipped. Feed two different listings and verify both are processed.

### US-3: Filter Out Irrelevant Jobs by Keywords (P1)
**As a** job seeker
**I want to** automatically discard listings that obviously don't match (wrong level, wrong domain)
**So that** only plausible matches reach the LLM scoring step (saving time and LLM calls)
**Independent test:** Feed a set of listings with known relevant/irrelevant keywords and verify correct filtering.

### US-4: Score Remaining Jobs with LLM (P1)
**As a** job seeker
**I want to** each surviving listing scored 0-100 against my full candidate profile
**So that** I can prioritize the best matches
**Independent test:** Provide a job description and candidate profile, get back a score (0-100) and reasoning string.

### US-5: Auto-Tailor Resume for High-Scoring Jobs (P1)
**As a** job seeker
**I want to** a tailored resume PDF generated automatically for each high-scoring job
**So that** I can apply immediately with a role-specific resume without manual editing
**Independent test:** Provide a master resume + job description, get back an HTML resume and a PDF file. Verify the PDF is valid and the content is tailored (not a carbon copy of the master).

### US-6: Verify Tailored Resume for Accuracy (P1)
**As a** job seeker
**I want to** each tailored resume checked for fabricated skills or inflated claims
**So that** I never submit a resume with false information
**Independent test:** Provide a tailored resume + master resume, get a pass/fail result. Feed a resume with a fabricated skill and verify it fails.

### US-7: Receive Telegram Notifications for Matches (P1)
**As a** job seeker
**I want to** receive instant Telegram messages for high-scoring matches (with score, reasoning, job link, and tailored resume attached)
**So that** I can act on opportunities quickly from my phone
**Independent test:** Trigger a notification for a test job and verify the Telegram message arrives with correct format and PDF attachment.

### US-8: Receive Daily Email Digest (P2)
**As a** job seeker
**I want to** receive a daily email summarizing all matches from the last 24 hours
**So that** I have a backup channel and can review matches in bulk
**Independent test:** Trigger a digest for a set of test jobs and verify the email contains all matches with resume PDFs attached.

### US-9: Recover from Pipeline Crashes (P1)
**As a** job seeker
**I want to** the pipeline to resume from where it left off after a crash
**So that** I never lose scraped data or waste LLM calls re-scoring already-processed jobs
**Independent test:** Simulate a crash after scoring 3 of 6 jobs. Restart the pipeline and verify it resumes from job 4, not job 1.

### US-10: Track Application Status (P2)
**As a** job seeker
**I want to** track which jobs I've applied to and their status (applied, interview, rejected, etc.)
**So that** I have a single place to see my job search progress and avoid duplicate applications
**Independent test:** Mark a job as "applied", then verify the tracker shows the correct status. Verify that a new match at the same company flags the prior application.

### US-11: Provide Feedback on Matches (P2)
**As a** job seeker
**I want to** react to each notification (applied / skip / not relevant) with an optional reason
**So that** the system collects data to improve future matching
**Independent test:** Submit feedback for a job and verify it's stored with the correct action and reason.

### US-12: Configure Sites Without Code Changes (P1)
**As a** job seeker (or developer maintaining the tool)
**I want to** add a new job board by creating a YAML config file (no code changes)
**So that** expanding to new sites is fast and doesn't risk breaking existing scrapers
**Independent test:** Create a new site config YAML, run the scraper, and verify it fetches and parses listings from the new site.

### US-13: Receive Pipeline Health Alerts (P1)
**As a** job seeker
**I want to** be alerted via Telegram when something goes wrong (scraper failure, LLM proxy down, etc.)
**So that** I know when the system needs attention without checking logs manually
**Independent test:** Simulate a site scraper failure and verify a Telegram alert is sent with the error details.

### US-14: Use LLM with Fallback Chain (P1)
**As a** job seeker
**I want to** LLM calls to automatically fall back from the primary proxy to OpenRouter if the proxy is down
**So that** the pipeline doesn't stall when one LLM provider has issues
**Independent test:** Simulate primary proxy failure, make an LLM call, and verify it succeeds via the fallback provider.

## Requirements

### Functional

- **FR-001:** The system shall load site scraper configurations from YAML files in `site_configs/` at startup.
- **FR-002:** Each site config shall specify one of three scraping strategies: `api`, `html`, or `browser`. All three strategies must be implemented — which strategy each site needs will be determined during site inspection.
- **FR-003:** The scraper shall normalize all listings into a common Job schema (id, site_id, title, company, location, country, url, salary, description, requirements, posted_date, scraped_at). Detail page fetching (following each job URL for full description) is configurable per site config — some sites provide enough data on listing pages, others require it.
- **FR-004:** The job ID shall be a SHA-256 hash of `normalize(title) + normalize(company) + normalize(location)` where normalize lowercases, strips whitespace, and removes punctuation.
- **FR-005:** The dedup engine shall skip any job whose hash already exists in the `seen_jobs` store.
- **FR-006:** The keyword filter shall support three rule types: `must_have_any`, `must_not_have`, and `title_must_have_any`. Global defaults are defined in `settings.yaml`. Each site config can extend or override global keywords.
- **FR-007:** The LLM scoring shall accept the master resume + scoring criteria (from `settings.yaml`) and job description, and return a JSON object with `score` (0-100) and `reasoning` (string). No separate candidate profile document — the master resume IS the profile, and scoring weights/preferences/visa needs are in settings.
- **FR-008:** Jobs scoring at or above a configurable threshold (default: 60) shall proceed to resume tailoring.
- **FR-009:** Resume tailoring shall take a master resume (YAML) and job description, and produce an HTML resume.
- **FR-010:** Resume verification shall compare a tailored resume against the master resume and return pass/fail with a list of issues (fabricated skills, inflated claims, missing critical info).
- **FR-011:** If verification fails, the system shall attempt to fix the issues and re-verify (max 2 retries before skipping).
- **FR-012:** The system shall convert verified HTML resumes to PDF.
- **FR-013:** Telegram notifications shall include: match score, job title, company, location, salary (if available), match reasoning, concerns, tailored resume PDF, and job listing URL.
- **FR-014:** Telegram notifications shall include inline reaction buttons: Applied, Skip, Not Relevant.
- **FR-015:** Notification routing shall be configurable by score range (e.g., 80+ instant Telegram, 60-79 email digest only, 40-59 logged only, <40 discarded after logging).
- **FR-016:** The checkpoint system shall save pipeline state after: each site scrape, normalization, each LLM score, each resume generation, and each notification sent.
- **FR-017:** On startup, the pipeline shall check for incomplete previous runs and resume from the last completed checkpoint.
- **FR-018:** Each site scraper shall run independently — a failure in one site shall not prevent other sites from completing.
- **FR-019:** Each job shall be processed independently — a failure in one job shall not prevent other jobs from processing.
- **FR-020:** Raw HTTP responses shall be saved to disk immediately after fetching (before parsing), organized by date.
- **FR-021:** The LLM abstraction shall implement a fallback chain: primary proxy → OpenRouter → skip and log.
- **FR-022:** On malformed LLM response, the system shall retry once with a stricter prompt before trying the next provider.
- **FR-023:** The application tracker shall record: job_id, company, role, country, applied_date, resume_version, status, and notes.
- **FR-024:** The tracker shall support statuses: matched, applied, phone_screen, interview, offer, rejected, withdrawn, expired.
- **FR-025:** Before sending a notification, the system shall check the tracker for prior applications to the same company and flag them in the notification.
- **FR-026:** User feedback (applied/skip/not_relevant + optional reason) shall be stored with the job_id, score, action, reason, and timestamp.
- **FR-027:** The daily email digest shall group all matches from the last 24 hours with resume PDFs attached.
- **FR-028:** Pipeline health alerts (site failures, LLM fallbacks, critical errors) shall be sent via Telegram.
- **FR-029:** Every pipeline run shall produce a structured log with: run_id, timestamps, sites attempted/succeeded/failed, jobs counts at each stage, errors, and LLM provider used.
- **FR-030:** Each site config shall specify a `max_pages` value for pagination (default: 3). The scraper paginates up to this limit per site.

### Non-Functional

- **NFR-001:** A full pipeline run (5 sites, ~30 listings) shall complete within 5 minutes under normal conditions.
- **NFR-002:** The system shall handle intermittent network failures gracefully (retry with backoff for HTTP requests, max 3 retries per request).
- **NFR-003:** All credentials (Telegram bot token, email password, API keys) shall be loaded from environment variables or a secrets file, never hardcoded.
- **NFR-004:** Logs shall be structured JSON (parseable for debugging and monitoring).
- **NFR-005:** The SQLite database and raw data files shall be excluded from version control.
- **NFR-006:** The pipeline shall be runnable via a single command (`uv run python -m jobhunter.main`) and via system cron.

## Acceptance Criteria

### AC-1: End-to-End Pipeline Run
**Given** at least one enabled site config and a valid master resume
**When** the pipeline runs
**Then** it scrapes listings, deduplicates, filters, scores with LLM, tailors resumes for high-scoring jobs, sends notifications, and logs results — all without manual intervention.

### AC-2: Site Isolation on Failure
**Given** 3 configured sites where site 2 returns HTTP 500
**When** the pipeline runs
**Then** sites 1 and 3 complete successfully, site 2's error is logged and an alert is sent, and raw response data for site 2 is saved.

### AC-3: Crash Recovery
**Given** a pipeline run that crashed after scoring 3 of 6 jobs
**When** the pipeline is restarted
**Then** it resumes from job 4, does not re-scrape sites that already completed, and does not re-score jobs 1-3.

### AC-4: Dedup Prevents Reprocessing
**Given** a job listing that was scraped and processed in a previous run
**When** the same listing appears in a subsequent scrape
**Then** it is skipped before the keyword filter stage.

### AC-5: Keyword Filter Reduces LLM Calls
**Given** 30 scraped listings where 22 contain dealbreaker keywords or lack required keywords
**When** the keyword filter runs
**Then** only the 8 matching listings proceed to LLM scoring.

### AC-6: LLM Scoring Returns Valid JSON
**Given** a job description and candidate profile
**When** the LLM scoring call completes
**Then** the response is valid JSON with integer `score` (0-100) and string `reasoning`.

### AC-7: Resume Tailoring Produces Valid PDF
**Given** a master resume and a job description for a high-scoring match
**When** resume tailoring completes
**Then** a PDF file is generated that: opens correctly, contains tailored content (not identical to master), and passes verification (no fabricated skills).

### AC-8: LLM Fallback Chain Works
**Given** the primary LLM proxy is unreachable
**When** an LLM call is made
**Then** the system falls back to OpenRouter, completes the call, and logs the fallback.

### AC-9: Telegram Notification Format
**Given** a scored and tailored job match
**When** the notification is sent
**Then** the Telegram message contains: score, title, company, location, reasoning, concerns, job link, and has the PDF attached. Inline reaction buttons (Applied, Skip, Not Relevant) are present.

### AC-10: New Site Config Works Without Code Changes
**Given** a new YAML site config following the template
**When** the pipeline runs
**Then** the new site is scraped alongside existing sites without any code modifications.

### AC-11: Checkpoint Saves Raw Data Before Parsing
**Given** a site that returns valid HTML but the parser crashes
**When** the error is logged
**Then** the raw HTML response is already saved to `data/raw/{date}/` and can be re-parsed manually.

### AC-12: Feedback Storage
**Given** a user taps "Not Relevant" on a Telegram notification
**When** the feedback is received
**Then** it is stored with the job_id, original score, action ("not_relevant"), and timestamp.

## Edge Cases

- **Empty scrape results:** Site returns 200 OK but no job listings — log a warning, don't treat as error, continue to next site.
- **Site layout changes:** Parser finds 0 job cards where it normally finds 10+ — log a warning, save raw HTML for manual inspection, alert via Telegram.
- **LLM returns score outside 0-100 range:** Clamp to valid range and log a warning.
- **LLM returns non-JSON response:** Retry once with stricter prompt. If still invalid, try next provider. If all fail, skip job and log raw response.
- **Master resume file missing or malformed:** Pipeline aborts with a clear error message (can't proceed without resume data).
- **No site configs enabled:** Pipeline logs a warning and exits cleanly (nothing to do).
- **Network timeout during scraping:** Retry up to 3 times with exponential backoff. After 3 failures, log error, save any partial data, continue to next site.
- **Duplicate company, different role:** Tracker flags prior applications to the same company but doesn't block — user decides.
- **Resume verification fails after retries:** Skip resume generation for this job, log the failure, still send a notification without the PDF (so the user knows about the match).
- **Telegram API rate limit:** Queue messages and send with delays. Don't drop notifications.
- **Database locked (concurrent access):** Use WAL mode for SQLite to allow concurrent reads. Pipeline runs are sequential (cron), so write conflicts should not occur.
- **Very long job descriptions:** Truncate to a reasonable token limit before sending to LLM. Log if truncation occurred.

## Assumptions

- **V1 targets 5 specific sites** (Bayt, GulfTalent, Undutchables, StepStone.de, Berlin Startup Jobs). No generic "add any site" guarantee needed beyond these. — change if wrong
- **V1 covers Netherlands, Germany, UAE only.** No other countries. — change if wrong
- **All three scraping strategies (api, html, browser) must be implemented for V1** — we don't know which strategy each target site will require until we inspect them during implementation. Each site config specifies its strategy. — change if wrong
- **Single user system** — no multi-tenancy, no auth, no user accounts. Config and master resume belong to one person. — change if wrong
- **Pipeline runs sequentially via cron** — no concurrent pipeline runs. No need for distributed locking. — change if wrong
- **The claude-max-api-proxy is already set up and running** when the pipeline executes. The pipeline does not manage the proxy lifecycle. — change if wrong
- **Telegram bot is already created via BotFather** and the token is available as an environment variable. — change if wrong
- **The master resume YAML format will be defined during implementation** — the spec does not prescribe a schema, only that it must contain: personal info, summary variants, experience with tagged achievements, certifications, education, and skills pool. — change if wrong

## Out of Scope

- LinkedIn, Indeed, or any site requiring anti-bot evasion
- Multi-user support or authentication
- Web dashboard or UI
- Cover letter generation
- Salary parsing or currency conversion
- Auto-apply functionality
- Automatic retraining of scoring based on feedback (V1 collects feedback data only)
- Browser-based scraping beyond headless Playwright (no anti-bot evasion, no CAPTCHA solving)
- Multi-language resume formats (German CV, Japanese Rirekisho)
- Company intelligence layer (Glassdoor ratings, visa sponsor lists)
- Proxy lifecycle management (starting/stopping the LLM proxy)

## Success Criteria

- Pipeline runs end-to-end unattended, producing Telegram notifications with tailored resume PDFs for matching jobs.
- Adding a new job board requires only creating a YAML config file — zero code changes.
- A crash at any pipeline stage loses zero scraped data and zero completed LLM scores.
- Tailored resumes contain zero fabricated skills or inflated claims (verified by the LLM verification step).
- The pipeline processes 30 listings across 5 sites in under 5 minutes.
- Pipeline health issues (site failures, LLM fallbacks) are surfaced via Telegram alerts within the same run.

## Open Questions

None — all clarified.
