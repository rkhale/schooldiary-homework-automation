# Cloud Run Job Design

## 1. Purpose

This document defines the design for the Google Cloud Run Job used by the SchoolDiary Homework Automation system.

The Cloud Run Job is responsible for running the scheduled capture process. It starts on a configured schedule, executes the SchoolDiary fetcher, captures or preserves notice evidence, uploads temporary file evidence to Google Cloud Storage, posts a structured payload to n8n, and exits.

This document focuses on job runtime behavior, configuration, secrets, execution flow, logging, exit behavior, and operational expectations.

## 2. Scope

This document covers:

- Cloud Run Job responsibilities
- Runtime execution flow
- Configuration and secret handling
- Cloud Scheduler trigger model
- Google Cloud Storage temporary evidence handoff
- n8n webhook posting
- Safe logging rules
- Error handling behavior
- Exit codes
- Local execution model
- Runtime permissions
- Operational assumptions

## 3. Out of Scope

This document does not define:

- SchoolDiary browser selectors
- SchoolDiary login page implementation
- Notice extraction selector strategy
- n8n node-by-node workflow design
- Google Sheet column definitions
- Google Calendar event structure
- Google Drive final evidence folder structure
- LLM itemization prompt
- Deployment commands
- Production monitoring dashboards

Those details are covered in later stage-specific design documents.

## 4. Relationship to Other Documents

This document depends on:

| Document | Relevance |
|---|---|
| `docs/00-high-level-project-design.md` | Defines the overall project objective and staged approach |
| `docs/01-architecture.md` | Defines component boundaries and runtime architecture |
| `docs/05-data-model.md` | Defines the payload contracts the Cloud Run Job must produce |

Later documents that depend on this file:

| Document | Relevance |
|---|---|
| `docs/03-schooldiary-fetcher-design.md` | Defines the browser automation implementation inside the job |
| `docs/04-n8n-workflow-design.md` | Defines the receiving webhook and downstream workflow |
| `docs/09-security-and-secrets.md` | Expands secret and access-control design |
| `docs/10-error-handling-and-observability.md` | Expands failure handling, alerts, and run visibility |
| `docs/11-deployment-runbook.md` | Defines deployment and setup steps |

## 5. Design Position

The Cloud Run Job is a short-lived scheduled execution unit.

It is not:

- An always-on service
- A public web application
- A database
- The master tracker
- The final evidence repository
- The notification engine
- The homework interpretation engine

The job owns capture execution only. n8n owns downstream orchestration.

## 6. High-Level Runtime Flow

```text
Google Cloud Scheduler
        ↓
Cloud Run Job starts
        ↓
Load configuration
        ↓
Load secrets
        ↓
Initialize run context
        ↓
Run Playwright SchoolDiary fetcher
        ↓
Capture notices, attachments, links, and evidence
        ↓
Upload downloaded/captured file evidence to GCS
        ↓
Build payload matching docs/05-data-model.md
        ↓
Post payload to n8n webhook
        ↓
Log safe run summary
        ↓
Exit with meaningful code
```

## 7. Cloud Scheduler Trigger Model

Google Cloud Scheduler triggers the Cloud Run Job on a configured schedule.

Recommended capture schedule:

| Local Time | Purpose |
|---:|---|
| 07:00 | Capture overnight or early morning notices |
| 15:30 | Capture notices posted after school closing time |
| 20:30 | Capture evening updates after study planning time |

Default timezone:

```text
Asia/Kolkata
```

The schedule should be configurable outside code.

Cloud Scheduler is responsible only for starting the job. It does not pass credentials, parse notices, or manage downstream workflow logic.

## 8. Cloud Run Job Responsibilities

The Cloud Run Job owns:

1. Runtime startup and shutdown.
2. Configuration loading.
3. Secret retrieval.
4. Run ID generation.
5. Safe logging initialization.
6. Playwright fetcher execution.
7. Temporary local file handling.
8. Temporary GCS upload for downloaded or captured evidence.
9. Capture payload construction.
10. n8n webhook posting.
11. Safe run summary logging.
12. Exit status reporting.

## 9. Cloud Run Job Non-Responsibilities

The Cloud Run Job does not own:

1. Final homework itemization.
2. Final Google Sheet updates.
3. Final Google Drive storage.
4. Google Calendar event creation.
5. Email notification delivery.
6. Reminder workflows.
7. Overdue workflows.
8. Manual completion tracking.
9. Long-term evidence retention.
10. Public UI or family dashboard.

These belong to n8n or later workflow layers.

## 10. Runtime Configuration

Configuration should be externalized through environment variables or Secret Manager references.

### 10.1 Required Runtime Configuration

| Variable | Required | Secret | Description |
|---|---:|---:|---|
| `SCHOOLDIARY_NOTICEBOARD_URL` | Yes | Yes/Recommended | Configured SchoolDiary Notice Board URL |
| `TIMEZONE` | Yes | No | Runtime timezone, default `Asia/Kolkata` |
| `OUTPUT_DIR` | Yes | No | Local temporary output directory |
| `GCP_PROJECT_ID` | Yes | No | Google Cloud project identifier |
| `GCS_EVIDENCE_BUCKET` | Yes | No | Temporary evidence handoff bucket |
| `GCS_EVIDENCE_PREFIX` | Yes | No | Prefix for evidence objects in GCS |
| `N8N_WEBHOOK_URL` | Yes | Yes | n8n webhook endpoint |
| `MAX_NOTICES_PER_RUN` | Yes | No | Maximum notices to process in one run |
| `HEADLESS` | Yes | No | Whether Playwright runs headless |
| `DOWNLOAD_ATTACHMENTS` | Yes | No | Whether attachments should be downloaded |
| `CAPTURE_SCREENSHOT` | Yes | No | Whether notice screenshots should be captured |
| `POST_TO_N8N` | Yes | No | Whether payload should be posted to n8n |

### 10.2 Required Secrets

| Secret | Required | Purpose |
|---|---:|---|
| `SCHOOLDIARY_USERNAME` | Yes | SchoolDiary authentication |
| `SCHOOLDIARY_PASSWORD` | Yes | SchoolDiary authentication |
| `N8N_WEBHOOK_TOKEN` | Yes | Payload authentication / validation |
| `N8N_WEBHOOK_URL` | Yes | n8n webhook endpoint if treated as sensitive |
| `SCHOOLDIARY_NOTICEBOARD_URL` | Recommended | Sensitive portal-specific URL |

### 10.3 Public Repository Rule

The public repository may include `.env.example` with placeholder values only.

It must not include:

- Real SchoolDiary username
- Real SchoolDiary password
- Real SchoolDiary Notice Board URL
- Real n8n webhook URL
- Real n8n webhook token
- Real Google Cloud project ID
- Real service account keys
- Real browser session data

## 11. Secret Handling Model

Secrets must be loaded at runtime and never hardcoded.

Preferred hierarchy:

1. Google Secret Manager for Cloud Run Job runtime.
2. Environment variables injected from managed secrets.
3. Local `.env` file for local development only.

Local `.env` files must be excluded from git.

The job must not log:

- Passwords
- Tokens
- Webhook URLs
- Session cookies
- Browser storage state
- Full authenticated URLs
- Raw secret values

## 12. Service Account and Permissions

The Cloud Run Job should run under a dedicated service account.

### 12.1 Minimum Required Permissions

| Permission Area | Purpose |
|---|---|
| Read selected secrets | Retrieve SchoolDiary and webhook secrets |
| Write to configured GCS bucket/prefix | Upload temporary evidence |
| Read from configured GCS bucket/prefix | Validate or reference uploaded evidence if needed |
| Write logs | Emit Cloud Logging records |

### 12.2 Explicit Non-Permissions

The Cloud Run Job should not need direct permissions to:

- Google Sheets
- Google Drive final evidence folder
- Google Calendar
- Gmail or email sending
- n8n credentials
- unrelated GCS buckets
- broad project administration

Those integrations are owned by n8n.

## 13. Local Temporary Storage

The job may use local temporary storage during execution.

Recommended local structure:

```text
/tmp/schooldiary-homework/
├── run_<run_id>/
│   ├── notices/
│   ├── attachments/
│   ├── evidence/
│   └── payload/
```

Rules:

1. Local files are temporary.
2. Files must not be assumed to survive after job completion.
3. Downloaded or captured evidence should be uploaded to GCS before the job exits.
4. Local temporary paths should not be sent to n8n unless used only for debugging in local mode.
5. Sensitive files such as cookies or browser state must not be written unless explicitly required in a later design.

## 14. GCS Temporary Evidence Handoff

Google Cloud Storage is used only as a temporary handoff layer.

### 14.1 Object Path Convention

Recommended path pattern:

```text
<gcs_evidence_prefix>/<yyyy>/<mm>/<dd>/<run_id>/<notice_id>/<file_name>
```

Example with synthetic values:

```text
schooldiary-evidence/2026/07/02/run_fake_001/notice_fake_001/notice_screenshot.png
```

### 14.2 GCS Usage Rules

1. Upload downloaded attachments where available.
2. Upload generated screenshots or PDFs where configured.
3. Do not use GCS as the final family-facing evidence store in MVP.
4. n8n should move or copy final evidence to Google Drive where applicable.
5. GCS object references may be included in the n8n payload.
6. GCS retention period should be defined in a later evidence or deployment document.

### 14.3 Public Links

Publicly accessible links found in notices should generally be preserved as references.

They should not be downloaded or mirrored to GCS unless a later design explicitly requires that behavior.

## 15. Capture Run Payload

The Cloud Run Job must produce a payload that conforms to `docs/05-data-model.md`.

At minimum, the top-level payload must include:

```json
{
  "webhook_token": "placeholder-only",
  "run": {
    "run_id": "run_2026-07-02T15-30-00_Asia-Kolkata",
    "source_system": "schooldiary",
    "captured_at": "2026-07-02T15:30:00+05:30",
    "timezone": "Asia/Kolkata",
    "status": "success",
    "notices": [],
    "errors": []
  }
}
```

Public samples must use placeholder values only.

## 16. Run ID Generation

Each job run must generate a unique `run_id`.

Recommended format:

```text
run_<yyyy-mm-ddThh-mm-ss>_<timezone-safe-label>
```

Example:

```text
run_2026-07-02T15-30-00_Asia-Kolkata
```

Rules:

1. `run_id` must be generated once at job start.
2. The same `run_id` must be used across logs, local paths, GCS paths, and payload.
3. `run_id` must not include secrets or personal data.
4. `run_id` should be deterministic enough for troubleshooting but unique per run.

## 17. Execution Flow Detail

### 17.1 Startup

At startup, the job should:

1. Initialize safe structured logging.
2. Generate `run_id`.
3. Load configuration.
4. Validate required configuration.
5. Retrieve required secrets.
6. Create local temporary run directory.
7. Initialize an empty capture payload.

### 17.2 Fetch Execution

The job should call the Playwright fetcher.

The fetcher should return:

- Source notices
- Attachments
- Public links
- Evidence captures
- Safe extraction notes
- Safe errors where applicable

The fetcher design is defined in `docs/03-schooldiary-fetcher-design.md`.

### 17.3 Evidence Upload

For each downloaded or captured file, the job should:

1. Validate file exists locally.
2. Generate or confirm file hash.
3. Upload file to the configured GCS bucket and prefix.
4. Store the GCS object path in the related attachment or evidence object.
5. Record upload errors as safe error objects.

### 17.4 Payload Construction

The job should construct a payload using the data model.

The payload should include:

- `run_id`
- `source_system`
- `captured_at`
- `timezone`
- `status`
- `notices`
- `errors`

The payload should not include:

- Passwords
- Tokens other than the required webhook token
- Cookies
- Browser session data
- Local file contents
- Real secrets
- Raw service account keys

### 17.5 n8n Posting

The job should post the payload to the configured n8n webhook when `POST_TO_N8N=true`.

Expected behavior:

1. Include webhook token according to the n8n contract.
2. Use a reasonable timeout.
3. Retry transient failures where safe.
4. Treat repeated webhook failure as job failure or partial failure based on whether capture completed.
5. Log only safe response metadata.

The job should not log the full webhook URL if it contains sensitive tokens or identifiers.

### 17.6 Shutdown

At shutdown, the job should:

1. Log final run summary.
2. Remove or leave local temporary files according to runtime behavior.
3. Exit with the appropriate exit code.

## 18. Status Model

The capture run status must be one of:

| Status | Meaning |
|---|---|
| `success` | Capture completed, payload posted to n8n, no blocking errors |
| `partial_success` | Some notices or evidence failed, but payload was posted |
| `failed` | Run could not complete or payload could not be posted |

## 19. Exit Codes

Recommended exit codes:

| Exit Code | Meaning |
|---:|---|
| `0` | Success |
| `10` | Configuration validation failure |
| `11` | Secret retrieval failure |
| `20` | SchoolDiary authentication failure |
| `21` | Notice Board navigation failure |
| `22` | Notice extraction failure |
| `30` | Evidence upload failure |
| `40` | n8n webhook post failure |
| `50` | Unexpected runtime error |
| `60` | Partial success with recoverable errors |

Exit code `60` may be used when the run posts a payload successfully but includes warning or recoverable error objects.

## 20. Retry Behavior

Retry behavior should be conservative.

| Operation | Retry Guidance |
|---|---|
| Secret retrieval | Retry only if transient error is detected |
| SchoolDiary login | Limited retry; avoid account lockout risk |
| Notice Board page load | Retry once or twice |
| Attachment download | Retry limited times, then mark attachment failed |
| GCS upload | Retry transient failures |
| n8n webhook post | Retry transient failures |
| Invalid payload | Do not retry until corrected |
| Bad credentials | Do not retry repeatedly |

The job should avoid aggressive retry loops.

## 21. Logging Rules

Logs should be structured, safe, and operationally useful.

### 21.1 Log Fields

Recommended log fields:

| Field | Purpose |
|---|---|
| `run_id` | Correlate all logs for one run |
| `component` | Runtime component emitting the log |
| `event` | Event name |
| `severity` | Log severity |
| `notice_count` | Number of notices captured |
| `error_count` | Number of safe errors |
| `duration_ms` | Runtime duration where applicable |

### 21.2 Safe to Log

Safe log examples:

- Run started
- Run completed
- Number of notices captured
- Number of attachments discovered
- Number of files uploaded to GCS
- n8n post succeeded or failed
- Safe error codes
- Exit code

### 21.3 Not Safe to Log

Do not log:

- SchoolDiary password
- SchoolDiary username unless explicitly sanitized
- n8n webhook token
- Full n8n webhook URL
- Cookies
- Browser storage state
- Raw authenticated URLs
- Real student names in public examples
- Full sensitive notice content in public test data

Runtime logs may contain limited operational notice metadata only where required for troubleshooting.

## 22. Error Object Handling

The job should represent recoverable and non-recoverable problems as safe error objects.

Example:

```json
{
  "error_code": "ATTACHMENT_DOWNLOAD_FAILED",
  "error_message": "Attachment could not be downloaded.",
  "component": "schooldiary_fetcher",
  "severity": "warning",
  "recoverable": true
}
```

Errors must not expose credentials, cookies, tokens, or personal data unnecessarily.

## 23. Local Development Mode

The job should support local execution for development.

### 23.1 Local Mode Requirements

Local mode should allow:

- Loading configuration from `.env`
- Running without Cloud Scheduler
- Running Playwright locally
- Writing temporary output under local `OUTPUT_DIR`
- Optionally skipping n8n post with `POST_TO_N8N=false`
- Optionally writing the payload to a local JSON file
- Using synthetic test fixtures

### 23.2 Local PowerShell Example

Synthetic example only:

```powershell
$env:POST_TO_N8N = "false"
$env:HEADLESS = "true"
$env:OUTPUT_DIR = "C:\Temp\schooldiary-homework"

python -m src.main
```

Do not commit `.env`, local output, screenshots, browser state, or real payloads.

## 24. Container Runtime Expectations

The Cloud Run Job container should include:

- Python runtime
- Playwright package
- Browser dependencies required for Chromium
- Application source code
- Minimal runtime utilities needed by the fetcher

The container should not include:

- Real credentials
- `.env` files
- Browser session state
- Real screenshots
- Real attachments
- Real payloads
- Google service account key files

Service account identity should be provided by Google Cloud runtime, not committed files.

## 25. Resource Assumptions

Initial MVP assumptions:

| Resource | Initial Assumption |
|---|---|
| CPU | Modest; browser automation is the main workload |
| Memory | Enough for Chromium-based Playwright execution |
| Runtime duration | Short-lived scheduled execution |
| Concurrency | One capture job run at a time |
| Storage | Temporary local storage plus GCS handoff |
| Network | Outbound access to SchoolDiary, GCS, and n8n |

Concurrency should remain low for MVP. Parallel capture runs are not required.

## 26. Concurrency and Idempotency

The job should be designed to tolerate repeated runs.

Rules:

1. Cloud Scheduler should avoid overlapping runs where possible.
2. If overlapping runs occur, n8n deduplication should prevent duplicate tracker rows.
3. `notice_hash` and `homework_hash` remain the primary deduplication controls.
4. GCS paths include `run_id`, so two runs do not overwrite each other.
5. The job should not assume it is the only execution ever running.

## 27. Security Requirements

The Cloud Run Job must follow these security requirements:

1. Use a dedicated service account.
2. Use least-privilege permissions.
3. Retrieve secrets securely.
4. Never commit secrets to git.
5. Never log secrets.
6. Do not persist browser session state unless a later design explicitly approves it.
7. Do not expose a public HTTP endpoint for the job.
8. Keep public samples synthetic.
9. Avoid storing personal data beyond operational need.
10. Use GCS only for temporary evidence handoff.

## 28. Operational Run Summary

At the end of each run, the job should produce a safe summary.

Recommended fields:

| Field | Description |
|---|---|
| `run_id` | Run identifier |
| `status` | `success`, `partial_success`, or `failed` |
| `started_at` | Start timestamp |
| `ended_at` | End timestamp |
| `duration_ms` | Total runtime |
| `notice_count` | Number of notices captured |
| `attachment_count` | Number of attachments discovered |
| `evidence_count` | Number of evidence files generated |
| `gcs_upload_count` | Number of files uploaded to GCS |
| `public_link_count` | Number of public links preserved |
| `error_count` | Number of safe errors |
| `n8n_post_status` | `posted`, `skipped`, or `failed` |
| `exit_code` | Process exit code |

## 29. Public Repository Sample Rules

Any examples in this repository must use only synthetic values.

Allowed:

- Fake run IDs
- Fake notice IDs
- Fake timestamps
- Fake GCS object paths
- Fake webhook token placeholders
- Fake subjects
- Fake homework text
- Fake `example.com` links

Not allowed:

- Real SchoolDiary URLs
- Real login credentials
- Real student names
- Real guardian names
- Real school names
- Real teacher names
- Real portal screenshots
- Real homework text
- Real PDFs or attachments
- Real service account files
- Real webhook URLs

## 30. Open Decisions

The following decisions remain open for later design stages:

1. Exact Cloud Run memory and CPU allocation.
2. Exact timeout configuration.
3. Whether the job should capture screenshots for every notice or only actionable notices.
4. Whether the job should write a local payload JSON artifact in production.
5. Exact GCS retention policy.
6. Whether n8n webhook token is sent in body or header.
7. Whether source URL is safe to include in payload.
8. Whether browser session state may ever be persisted.
9. Whether failed payloads should be retried by job rerun or stored for replay.
10. Whether payload posting should be synchronous only or support fallback storage.

## 31. Acceptance Criteria

The Cloud Run Job design is implemented correctly when:

1. The job can start and exit without a local machine dependency.
2. Required configuration is validated at startup.
3. Required secrets are loaded securely.
4. The job generates a unique `run_id`.
5. The job can invoke the fetcher.
6. Downloaded or captured file evidence is uploaded to GCS.
7. The job builds a payload matching `docs/05-data-model.md`.
8. The job posts the payload to n8n when enabled.
9. The job logs a safe run summary.
10. The job exits with meaningful status codes.
11. No credentials, cookies, session state, real screenshots, or real payloads are committed to git.

## 32. Summary

The Cloud Run Job is the scheduled capture execution unit for the system. It is intentionally short-lived, stateless, and narrowly scoped.

It owns runtime execution, secure configuration loading, SchoolDiary fetcher invocation, temporary evidence handoff to GCS, n8n payload posting, and safe operational logging.

It does not own final homework tracking, calendar creation, evidence finalization, or notifications. Those responsibilities remain with n8n and the connected Google Workspace tools.
