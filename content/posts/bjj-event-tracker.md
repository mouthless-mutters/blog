+++
title = "Building and Deploying a BJJ Event Tracker with GitHub Copilot and GCP"
date = "2026-04-02T09:00:00-05:00"
draft = false
+++

I wanted to build something useful for the grappling community while also getting hands-on experience with GitHub Copilot as a coding assistant and with deploying a real app to GCP. The result is a BJJ Event Tracker: a live web app that aggregates upcoming Brazilian Jiu-Jitsu and grappling events from multiple sources, deduplicates them, and shows them in a filterable UI with direct watch links.

Here is the story of how it came together.

## The General Idea

There is no single place to find upcoming BJJ events, such as ADCC, UFC BJJ, PGF, WNO, IBJJF, etc. I wanted one feed that pulled from all of them, cleaned up duplicates, and showed me what was coming up in a browser.

The core loop is:

1. Scrape each source
2. Normalize and deduplicate events
3. Store in SQLite
4. Serve through a FastAPI API
5. Render in a simple browser UI

## Using GitHub Copilot

I built the entire thing using GitHub Copilot in agent mode inside VS Code. Rather than writing code manually, I described what I wanted and let Copilot generate, iterate, and debug. A few things stood out:

Copilot was useful for scaffolding repetitive scraper patterns. Each source has slightly different HTML or API behavior, and Copilot could generate a new scraper that matched the existing patterns in the project quickly.

When things broke in production, I pasted error output directly into the chat and Copilot diagnosed the root cause and ran fix commands in the terminal for me. For example, when the Cloud Run Job was crashing with:

    AttributeError: 'str' object has no attribute 'model_dump'

Copilot identified that `main.py` was iterating over the wrong return type from `run_ingestion()` and fixed it without me needing to trace through the code myself.

It also handled GCP CLI commands that had PowerShell-specific quirks. For example, the URI `https://.../$JOB:run` gets parsed incorrectly in PowerShell because `:run` is treated as a scoped variable. Copilot caught this and rewrote it as `$($JOB):run`.

The workflow I found most effective: describe the goal at a high level, let Copilot propose the approach, confirm the plan, then let it execute. I stepped in when its first attempt at something failed (usually IAM or CLI syntax) and pasted back the error so it could self-correct.

## Tech Stack

**Scrapers** (in `scrapers/`)
- `flograppling.py` - FloGrappling HTML scraper
- `ibjjf.py` - IBJJF schedule scraper
- `pgf.py` - PGF schedule scraper
- `ufc_fightpass.py` - UFC Fight Pass upcoming events API
- `smoothcomp.py` - Smoothcomp page parser
- `youtube.py` - YouTube upcoming livestream search (optional, requires API key)

**Pipeline** (in `pipeline/`)
- Promotion name normalization
- Ruleset detection (gi / no-gi)
- Event ID generation from normalized fields
- Exact deduplication by ID
- Fuzzy deduplication by name + date proximity

**Storage**
- SQLite for local development (`data/events.db`)
- `storage/gcs_sync.py` syncs the SQLite file to a Cloud Storage bucket for production persistence

**API** - FastAPI (`app.py`)
- `GET /events` - all events with filters
- `GET /events/upcoming` - upcoming only
- `GET /events/{event_id}` - single event
- `GET /promotions` - distinct promotion list
- `GET /health` - source freshness check
- `POST /ingest` - manual trigger (disabled in production)

**Web UI** (`web/`)
- Vanilla JS, no framework
- Search, promotion filter, ruleset filter, date sort
- Event cards with direct watch links

## Inspecting the UFC Fight Pass API

One concrete example of using Copilot to dig into a data source: the UFC Fight Pass scraper was setting every watch URL to the generic schedule page:

    https://welcome.ufcfightpass.com/schedule

I suspected there was a per-event link available. I asked Copilot to inspect the API response and it ran this:

    python -c "
    from scrapers.ufc_fightpass import _get_auth_token, _fetch_page
    import httpx, json
    c = httpx.Client(timeout=30.0)
    t = _get_auth_token(c)
    p = _fetch_page(c, t, 1)
    ev = p.get('events', [])
    flt = [e for e in ev if any(k in (e.get('title') or '').lower() for k in ('cffc bjj','ufc bjj','submission hunter pro'))]
    print(json.dumps(flt[0], indent=2, default=str))
    "

The response included an `id` field (e.g. `301012`). Copilot then verified:

    Invoke-WebRequest -Uri "https://app.ufcfightpass.com/live/301012" -Method Head -UseBasicParsing

Returned `200`. So it patched the scraper to build deep links using the event ID:

    https://app.ufcfightpass.com/live/{id}

## Deploying to GCP

### Step 1: Enable required services

    gcloud services enable `
      run.googleapis.com `
      cloudbuild.googleapis.com `
      artifactregistry.googleapis.com `
      cloudscheduler.googleapis.com `
      secretmanager.googleapis.com

### Step 2: Build and push the container image

The project already had a `Dockerfile`. Cloud Build handled the rest:

    gcloud builds submit --tag gcr.io/PROJECT_ID/bjj-event-tracker

### Step 3: Deploy to Cloud Run

    gcloud run deploy bjj-event-tracker \
      --image gcr.io/PROJECT_ID/bjj-event-tracker \
      --platform managed \
      --region northamerica-northeast1 \
      --allow-unauthenticated \
      --port 8080 \
      --set-env-vars ALLOW_PUBLIC_INGEST=false,GCS_BUCKET=PROJECT_ID-bjj-events-db

### Step 4: Create a Cloud Storage bucket for DB persistence

Cloud Run containers have ephemeral filesystems, so the SQLite database disappears on restart. The fix was a small GCS sync layer already in the codebase: the service downloads the DB at startup and the ingestion job uploads it after each run.

    gcloud storage buckets create gs://PROJECT_ID-bjj-events-db \
      --location=northamerica-northeast1 \
      --uniform-bucket-level-access

    gcloud storage buckets add-iam-policy-binding gs://PROJECT_ID-bjj-events-db \
      --member="serviceAccount:PROJECT_NUM-compute@developer.gserviceaccount.com" \
      --role="roles/storage.objectAdmin"

### Step 5: Create a Cloud Run Job for ingestion

Rather than triggering ingestion through a public HTTP endpoint, a Cloud Run Job runs `python main.py` on a schedule. This keeps the scraping logic completely off the public API surface.

    gcloud run jobs create bjj-ingest-job \
      --image gcr.io/PROJECT_ID/bjj-event-tracker \
      --region northamerica-northeast1 \
      --command python \
      --args main.py \
      --set-env-vars GCS_BUCKET=PROJECT_ID-bjj-events-db

### Step 6: Schedule the job with Cloud Scheduler

    gcloud iam service-accounts create scheduler-run-job \
      --display-name "Scheduler invokes Cloud Run Job"

    gcloud run jobs add-iam-policy-binding bjj-ingest-job \
      --region northamerica-northeast1 \
      --member="serviceAccount:scheduler-run-job@PROJECT_ID.iam.gserviceaccount.com" \
      --role="roles/run.invoker"

    gcloud scheduler jobs create http bjj-ingest-daily-private \
      --location=northamerica-northeast1 \
      --schedule="0 6 * * *" \
      --uri="https://run.googleapis.com/v2/projects/PROJECT_ID/locations/northamerica-northeast1/jobs/bjj-ingest-job:run" \
      --http-method=POST \
      --message-body="{}" \
      --oauth-service-account-email=scheduler-run-job@PROJECT_ID.iam.gserviceaccount.com \
      --oauth-token-scope="https://www.googleapis.com/auth/cloud-platform"

### Step 7: Disable the public ingest endpoint

With the job handling ingestion, the `POST /ingest` API route was locked down:

    ALLOW_PUBLIC_INGEST=false

Any request to `POST /ingest` now returns 404. The Refresh Sources button was also removed from the UI entirely since no end user can trigger it.

## Things That Broke Along the Way

**Cloud Scheduler sending requests to the wrong URL**

The first attempt at creating the Cloud Scheduler job used `--headers` instead of `--update-headers` on an `update http` command. Also, in PowerShell, `$JOB:run` in a URI string is silently parsed as a variable, stripping the job name. The final working URI had to be built like this:

    $RUN_URI = "https://run.googleapis.com/v2/projects/$PROJECT_ID/locations/$REGION/jobs/$($JOB):run"

**Ingestion job crashing with AttributeError**

`main.py` was iterating over the return value of `run_ingestion()` and calling `.model_dump()` on each item, but `run_ingestion()` returns a summary dict, not a list of Event objects. The fix was to stop iterating and just print the result:

    result = run_ingestion()
    print(result)

**Events not showing up after deploy**

Because Cloud Run containers are stateless, a fresh deployment started with an empty database. The GCS sync fixed this, but it required both the ingestion job and the web service to have `GCS_BUCKET` set as an environment variable. Once both were configured and a job execution completed, events appeared on the live site.

## Final Result

- Live at: https://bjj-event-tracker-615762110730.northamerica-northeast1.run.app. Will eventually move it to a different domain.
- Events from: FloGrappling (WNO, IBJJF, Fight-to-Win, etc.), PGF, UFC Fight Pass (UFC BJJ, CFFC BJJ, etc.)
- Ingestion runs daily at 06:00 UTC via Cloud Scheduler + Cloud Run Job
- SQLite persisted to Cloud Storage between runs
- Public ingest endpoint disabled; ingestion is private and scheduled
- Codebase on GitHub: https://github.com/mouthless-mutters/bjj-event-tracker