# getHire AI Features

> Source-audited overview of AI-powered capabilities in the employee-side getHire application.
>
> **Audit scope:** `employee-api`, `employee-ui`, `employee-worker`, and `employee-pubsub`.
> **Explicit exclusions:** `employer-ui`, `employer-api`, and `employer-worker` were not included in this audit.

## 1. Scope and confidence

This document describes implementation paths found during a source-code audit. It distinguishes:

- **Implemented in source:** a feature or call path is present in the audited code.
- **Configuration-dependent:** operation depends on provider credentials, cloud configuration, database capabilities, or service deployment.
- **Optional/legacy:** an alternate or older implementation exists, but its active use is not established.
- **Not verified at runtime:** source code alone does not prove that a production/local deployment has the required settings, credentials, subscriptions, or populated data.

This is not a claim that every feature is enabled in every environment.

## 2. AI provider architecture

### Central LLM router

The primary provider abstraction is `employee-api/app/services/llm.py`. It centralizes text generation, structured JSON generation, and embeddings for services that call the router.

Audited source defaults:

| Setting | Default |
|---|---|
| `LLM_PROVIDER` | `gemini` |
| `GEMINI_LOCATION` | `us-central1` |
| `GEMINI_MODEL` | `gemini-2.5-flash` |
| `GEMINI_FAST_MODEL` | `gemini-2.5-flash-lite` |
| `GEMINI_RESUME_MODEL` | `gemini-2.5-flash-lite` |
| `GEMINI_EMBEDDING_MODEL` | `gemini-embedding-001` |
| `GEMINI_EMBEDDING_DIMENSIONS` | `768` |
| `OLLAMA_MODEL` | `llama3.2` |
| Ollama embedding model | `nomic-embed-text` |

These are **code defaults**, not proof of the effective values in a running environment.

### Supported and alternate routes

- Gemini is the default configured provider in `app/config.py`.
- Gemini generation can use Vertex AI authentication/configuration or a Gemini API key, depending on environment configuration. Therefore, the source audit does **not** establish that all Gemini requests are Vertex AI requests.
- Optional local Ollama generation/embedding support exists through `LLM_PROVIDER=ollama` and `app/services/llm_ollama.py`.
- Account-level BYO provider settings support Claude and Gemini for supported AI uses such as replies, matching, summaries, intelligence, and fallback.
- Copilot and Cursor can appear in account settings as provider choices, but the audited backend usable-provider list is Claude and Gemini; do not interpret storing those choices as proof of model API integration.
- `timesheet_reader.py` uses a direct Google GenAI client path rather than necessarily going through the central router.
- `embeddings.py` contains both a direct Ollama embedding path (`generate_email_embeddings`) and a central-router embedding path (`generate_embedding`). Callers determine which path is used; do not assume they are interchangeable.

### Usage, cost and controls

Relevant implementation includes `app/services/ai_usage.py`, account settings, AI usage migrations, and tests such as `test_ai_cost.py`, `test_ai_tiers.py`, and `test_llm_byo.py`. The code includes usage tracking, feature attribution, limits/caps, caching and fallback-related behavior. Actual limits and effective provider behavior depend on runtime configuration and account state.

## 3. Employee-facing AI features

### 3.1 Email intelligence

**Purpose:** enrich synced emails with structured job/recruitment context and make that information available in the Inbox and Intelligence experiences.

Audited implementation includes:

- `app/services/intelligence.py`
- `app/services/genai_http_client.py`
- `app/services/email_pipeline.py`
- `app/routers/intelligence.py`
- `app/routers/emails.py`

Capabilities found in source include structured email analysis such as job opportunity signals, company, role/job context, employment type, confidence and related metadata. The UI loads intelligence in batches and can request analysis for an individual email.

UI entry points include:

- `employee-ui/app/dashboard/inbox/page.tsx`
- `employee-ui/components/inbox/email-detail.tsx`
- `employee-ui/components/inbox/email-intelligence-modal.tsx`
- `employee-ui/components/inbox/intelligence-panel.tsx`
- `employee-ui/components/inbox/intelligence-overview.tsx`

The UI exposes email summary and intelligence tabs/panels. Manual email/thread summary endpoints exist. Automatic/background batch summarization has been disabled in the audited code path; do not describe all emails as being automatically summarized.

### 3.2 Email categorization and triage

**Purpose:** categorize incoming mail and help distinguish relevant recruitment/job emails from other mail.

- `app/services/mail_categorizer.py` uses rule-based classification first and Gemini fast-model classification for applicable unmatched/batch cases.
- `app/services/protocol_classification.py` uses keyword/rule matching first and Gemini fallback for unmatched cases in the applicable path.
- The bulk `classify_all_emails` path was identified as keyword-only in the reviewed source.
- Inbox UI exposes a reclassification action through `PROTOCOLS_RECLASSIFY_ALL`.

The actual classification mix depends on the path invoked and the available provider configuration. Avoid describing every classification as an LLM call.

### 3.3 Tracks and resume skill extraction

**Purpose:** extract skills from a user's resume and use them to organize/recommend relevant job tracks.

Relevant backend paths:

- `app/services/skill_extraction.py`
- `app/routers/protocols.py`
- `app/services/genai_http_client.py`
- worker task `employee-worker/tasks/extract_resume_skills.py`

The extraction service calls the GenAI client to extract resume skills and stores valid results. The reviewed implementation protects existing stored skills when extraction is empty/invalid rather than blindly replacing them.

The re-extract endpoint in `protocols.py` was recently changed to call `extract_and_store_skills(protocol.id)` synchronously. This is the audited local implementation and should be rechecked if the endpoint is later made asynchronous.

UI entry points include:

- `employee-ui/app/dashboard/protocols/page.tsx`
- `employee-ui/components/protocols/protocol-modal.tsx`
- `employee-ui/app/dashboard/intelligence/page.tsx`

The UI exposes a Re-extract action and track-based matching views.

### 3.4 Resume-to-job matching and job intelligence

**Purpose:** surface likely job opportunities using resume skills, tracked email/job context, and uploaded resumes.

Relevant backend paths include:

- `app/routers/jobs.py`
- `app/routers/resume_matches.py`
- `app/services/resume_matches.py`
- `app/services/resume_gap_analysis.py`
- `app/services/resume_profile.py`
- `app/services/jd_alignment.py`

Observed behavior:

- Track/resume skill overlap can be scored deterministically; not every match requires an LLM call.
- Stored `ResumeMatch` records are preferred where available; a quick skill-overlap fallback exists.
- Resume gap analysis uses LLM-backed comparison when useful resume/job information is available and can cache repeated results.
- JD alignment includes structured comparison/report functionality; some skill extraction/comparison steps are deterministic.
- Resume profile generation is an LLM-backed path.
- UI combines track-derived matches and uploaded-resume job results.

UI entry points include:

- `employee-ui/app/dashboard/intelligence/page.tsx`
- `employee-ui/components/intelligence/hooks/use-track-matches.ts`
- `employee-ui/components/intelligence/hooks/use-resume-upload.ts`
- `employee-ui/components/intelligence/matches-section.tsx`
- `employee-ui/components/inbox/resume-gap-analysis.tsx`
- `employee-ui/components/inbox/jd-alignment-report.tsx`

### 3.5 Resume generation and automatic resume updates

The intelligence router contains structured resume generation/update paths using the configured resume model (`GEMINI_RESUME_MODEL` in the reviewed source). The audited generation path uses structured output and has a high output-token allowance. Resume auto-update UI is exposed from the Inbox detail experience.

Relevant UI:

- `employee-ui/components/inbox/resume-auto-update.tsx`
- `employee-ui/components/inbox/email-detail.tsx`

Generated document formatting and page layout are document-generation concerns in addition to model generation; a model response alone does not guarantee a fixed page count or page-break layout.

### 3.6 Email replies and automatic sender

The API has an AI-assisted automatic email sender path (`app/services/auto_sender.py`) and the UI includes auto-email/auto-sender surfaces.

The audited intended safety policy is restrictive:

- Only consider verified HR/recruitment or verified company recruitment/job mail.
- Skip newsletters, no-reply, promotional messages, forms and rejection messages.
- Require a resume attachment where configured.
- Draft/review behavior is supported.

The reviewed runtime defaults were dry-run enabled and live sending disabled (`AUTO_EMAIL_DRY_RUN=True`, `AUTO_EMAIL_SEND_ENABLED=False`). These are source/runtime defaults noted during audit; confirm deployment environment values before assuming current send behavior. Do not present live sending as enabled by this documentation.

### 3.7 Contact classification and extraction

Relevant implementation:

- `app/services/contact_classification.py`
- `app/services/contact_extraction.py`
- `app/routers/contacts.py`

The classification path uses structured LLM output with the configured fast model for contact roles such as recruiter, HR, or hiring manager. Contact extraction uses synced emails and enriched email intelligence. Batch/retry paths exist.

UI entry point: `employee-ui/app/dashboard/contacts/page.tsx`.

### 3.8 Submission signal classification and tracking

Relevant implementation:

- `app/services/submission_signals.py`
- `app/routers/submissions.py`
- `employee-worker/tasks/track_submissions.py`

The signal classifier uses Gemini batch classification for recruitment/vendor messages and signals such as interview scheduling or rejection. Submission tracking consumes these signals and maintains application stages, replies, vendor replies, calendar links, follow-up tracks and stale-record handling.

### 3.9 Vendor job extraction

`app/services/vendor_job_scraper.py` uses the configured Gemini fast model to extract structured job listings from career pages. The employee worker schedules vendor job scraping periodically. This is distinct from the excluded employer repositories.

### 3.10 Employment type inference

`app/services/employment_types.py` uses structured generation with the configured fast model for employment-type inference/backfill.

### 3.11 Timesheet/document reading

`app/services/timesheet_reader.py` uses a direct Google GenAI client path for timesheet/document reading. This path is not necessarily routed through `app/services/llm.py`; its authentication and model configuration must be checked separately.

### 3.12 Support assistant

`app/routers/support.py` exposes a support assistant path that calls the central LLM service. The central provider router determines the actual provider. Older comments/docstrings referencing Ollama should not be taken as proof that Ollama is the active provider.

## 4. Embeddings and semantic search

Relevant implementation:

- `app/services/embeddings.py`
- `app/services/intelligence.py`
- `app/services/llm.py`

The source supports email embeddings and semantic email search. Semantic search can use vector similarity/cosine distance when embeddings are present, with keyword fallback when embeddings are unavailable.

Database support includes an optional pgvector extension and a 768-dimensional email embedding column. Migrations and model definitions establish schema support, not proof that the extension is enabled or vectors are populated in a particular environment.

The embedding model configured in source defaults is `gemini-embedding-001` with 768 dimensions. The code also contains an Ollama embedding route. Switching embedding models/providers may require re-embedding because vector spaces are not guaranteed to be compatible.

## 5. Background processing architecture

### 5.1 Employee worker

The separate `employee-worker` service runs background jobs independently from the API process.

Core files:

- `employee-worker/main.py`
- `employee-worker/scheduler.py`
- `employee-worker/tasks/categorize_emails.py`
- `employee-worker/tasks/classify_protocols.py`
- `employee-worker/tasks/extract_contacts.py`
- `employee-worker/tasks/extract_resume_skills.py`
- `employee-worker/tasks/generate_embeddings.py`
- `employee-worker/tasks/run_intelligence.py`
- `employee-worker/tasks/track_submissions.py`

The worker uses APScheduler and PostgreSQL `LISTEN/NOTIFY` on `sync_settings_changed` to react to sync-setting changes. A periodic polling fallback checks settings every five minutes.

Reviewed scheduler intervals include:

| Job | Interval |
|---|---:|
| Sync settings meta-dispatcher | 5 minutes |
| Vendor job scraper | 1 hour |
| Email backfill | 2 minutes |
| Email categorization | 2 minutes |
| Submission tracking | 3 minutes |
| Sync tick | 15 minutes |

Per-account email sync settings are evaluated by the sync task; the 15-minute tick is not necessarily the actual per-account sync interval.

After a successful email sync, the worker chains these processing tasks:

1. Email intelligence processing
2. Protocol/track classification
3. Contact extraction
4. Email embedding generation

Each chained task has separate error handling/logging. Other sync tasks (contacts, calendar and drive) are part of the worker's broader sync architecture.

### 5.2 Employee Pub/Sub service

The separate `employee-pubsub` service handles provider notification delivery and queued email processing. It is event plumbing, not an AI model.

Audited files:

- `employee-pubsub/main.py`
- `employee-pubsub/subscriber.py`
- `employee-pubsub/outlook_listener.py`
- `employee-pubsub/watch_manager.py`

Responsibilities found in source:

- Google Gmail notifications through Google Cloud Pub/Sub streaming pull.
- Outlook/Yahoo event handling through PostgreSQL `LISTEN/NOTIFY`.
- Processing API-created email queue entries.
- Triggering `run_email_pipeline` for queued email events.
- Gmail watch setup/maintenance and Outlook subscription renewal.
- Retry of failed queue items and polling fallback.

Reviewed housekeeping intervals:

| Job | Interval |
|---|---:|
| Renew expiring watches/subscriptions | 6 hours |
| Ensure Gmail watches | 10 minutes |
| Retry failed queue items | 5 minutes |
| Poll provider accounts (fallback) | 2 minutes |

Google streaming subscription startup is conditional on required configuration (including project and subscription identifiers) and the Pub/Sub client package. If streaming subscriber startup fails or is not configured, the service has a polling fallback. Source presence does not prove Pub/Sub is enabled in the current deployment.

## 6. UI feature map

The following employee UI files expose or support the audited AI features:

| UI area | Representative source files | Capability |
|---|---|---|
| Inbox | `app/dashboard/inbox/page.tsx`, `components/inbox/email-detail.tsx` | Email intelligence, manual analysis, summaries, reclassification, AI detail tabs |
| Intelligence | `app/dashboard/intelligence/page.tsx` | Track matches, uploaded-resume job results, job details |
| Tracks/Protocols | `app/dashboard/protocols/page.tsx`, `components/protocols/protocol-modal.tsx` | Resume skill extraction/re-extraction and track management |
| Contacts | `app/dashboard/contacts/page.tsx` | Contact classification/extraction workflows |
| Resume analysis | `components/inbox/resume-gap-analysis.tsx`, `components/inbox/jd-alignment-report.tsx` | Gap analysis and JD alignment |
| Resume update | `components/inbox/resume-auto-update.tsx` | AI-assisted resume update |
| Reply | `components/inbox/reply-composer.tsx` | Reply composition support |
| Auto sender | `app/dashboard/auto-sender/page.tsx` | Review-oriented automatic email workflow |
| Support | `components/support/support-chat.tsx` | Support assistant UI |
| AI settings | `components/settings/ai-keys-tab.tsx` | User provider/key settings |

The UI source audit also identified API endpoint maps in `lib/endpoints.ts`, `lib/endpoints-inbox.ts`, `lib/endpoints-intelligence.ts`, and `lib/endpoints-onboarding.ts`.

## 7. Data and persistence components

Relevant persistence concepts found in the audited API source/migrations include:

- Email intelligence records and analysis metadata.
- Optional email embedding vector column (768 dimensions).
- Resume match records.
- AI usage tracking records.
- Account-level AI provider/key settings.
- Protocol/track skill records.
- Contact classification/enrichment fields.
- Submission signal/stage data.

Migrations are evidence of schema support and historical evolution, not proof that every table/column is present in every deployed database.

## 8. Important operational qualifications

1. **Provider is environment-specific.** The source default is Gemini, but runtime environment variables and per-user BYO settings can change routing.
2. **Vertex AI is not guaranteed for every Gemini call.** Some paths support API-key authentication; timesheet reading has a direct GenAI client path.
3. **Ollama remains in source.** It is optional/alternate; its presence does not prove it is running.
4. **Not every feature is generative AI.** Rule-based categorization, keyword matching, skill overlap scoring, vector similarity, and report formatting are deterministic or algorithmic components.
5. **Background services must be deployed and configured.** Worker and Pub/Sub code being present does not prove those processes are running.
6. **Pub/Sub is conditional.** Missing subscription configuration or client dependencies can lead to polling fallback.
7. **Auto-send safety must be verified at runtime.** The reviewed defaults are dry-run on and sending off; inspect deployment flags before enabling any live email sending.
8. **Embedding availability is conditional.** pgvector support and a 768-dimensional column do not prove vectors have been generated.
9. **Credentials are intentionally not documented.** Never place API keys, service-account JSON, OAuth secrets, or user tokens in this README.

## 9. Audit inventory

The source audit was restricted to these four modules:

- `employee-api`
- `employee-ui`
- `employee-worker`
- `employee-pubsub`

The consolidated source grep audit scanned 609 source files across those modules and produced 2,560 matches. The UI source collection captured 24 selected files (10,222 lines). Worker/Pub/Sub source review covered the scheduler, task modules, subscriber, listeners and watch manager.

Search matches are discovery evidence, not a guarantee that every matched file is an active execution path. The representative source paths above are the primary navigation points for future code review.

## 10. Suggested follow-up verification for each deployment

Before describing a deployment as fully operational, verify without exposing secret values:

- Effective `LLM_PROVIDER` and model names.
- Whether Gemini is authenticated through Vertex AI or an API key.
- Whether Ollama is intentionally enabled and reachable.
- Whether Pub/Sub project/subscription configuration is present and the subscriber is connected.
- Whether the employee worker and Pub/Sub processes are running.
- Whether pgvector is enabled and email embeddings are populated.
- Whether automatic email sending remains disabled or has been explicitly approved and configured.

---

*Prepared from a source-code audit of the employee-side getHire modules. Update this document when provider routing, task schedules, UI endpoints, or feature flags change.*
