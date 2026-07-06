# Error Handling and Observability Design

## 1. Purpose

This document defines the error handling and observability model for the SchoolDiary Homework Automation system.

The system is scheduled, unattended, and family-facing. It must therefore fail safely, avoid duplicate processing, record enough operational evidence to diagnose issues, and notify the parent/admin when action is required.

The design separates:

- family-facing homework notifications
- operational alerts
- runtime logs
- tracker status
- processing errors
- evidence finalization status

## 2. Scope

This document covers:

- Error handling principles
- Error categories
- Retry and skip behavior
- Cloud Run Job observability
- SchoolDiary fetcher observability
- n8n workflow observability
- Google Sheets operational tabs
- Google Drive evidence finalization errors
- Google Calendar creation errors
- Daily digest errors
- Operational alerting
- Log redaction
- Health checks
- Metrics
- Acceptance criteria

## 3. Out of Scope

This document does not define:

- Google Cloud Logging setup in detail
- Google Cloud Monitoring dashboards
- n8n workflow implementation internals
- Google OAuth setup
- SchoolDiary selector implementation
- Production incident management tooling
- Enterprise DLP tooling
- PagerDuty, Slack, Teams, or WhatsApp alerting

Those may be added after MVP if needed.

## 4. Relationship to Other Documents

| Document | Relevance |
|---|---|
| `docs/01-architecture.md` | Defines system components and data flow |
| `docs/02-cloud-run-job-design.md` | Defines Cloud Run exit codes and runtime behavior |
| `docs/03-schooldiary-fetcher-design.md` | Defines fetcher error codes |
| `docs/04-n8n-workflow-design.md` | Defines n8n workflows and processing tabs |
| `docs/05-data-model.md` | Defines run, notice, evidence, and tracker payload fields |
| `docs/06-google-calendar-design.md` | Defines calendar event creation behavior |
| `docs/07-google-drive-evidence-design.md` | Defines evidence finalization and Drive error handling |
| `docs/08-notification-design.md` | Defines family digest and operational alert separation |
| `docs/09-security-and-secrets.md` | Defines redaction, private config, and secret handling |
| `docs/11-deployment-runbook.md` | Will define deployment and runtime operations |
| `docs/12-testing-strategy.md` | Will define test coverage and failure-mode testing |

## 5. Design Position

Observability has three goals:

1. Confirm that the automation ran.
2. Confirm what it processed.
3. Identify what requires human intervention.

The system should not require reading raw logs for normal family use. The Google Sheet tracker and operational tabs should provide the primary day-to-day visibility.

Logs are for technical troubleshooting. They must be safe and redacted.

## 6. Error Handling Principles

The system follows these rules:

1. Fail closed when credentials, tokens, or required configuration are missing.
2. Do not expose secrets in logs, tracker rows, emails, or alerts.
3. Do not create duplicate homework rows, evidence records, calendar events, or digest emails.
4. Continue processing independent notices when one notice fails.
5. Preserve raw evidence where safe and available.
6. Create `Needs Review` rows when homework is actionable but ambiguous.
7. Send operational alerts only when action is required.
8. Keep family-facing digest emails free from technical noise.
9. Record all meaningful processing failures in a structured location.
10. Make retry behavior controlled and idempotent.

## 7. Component Responsibilities

| Component | Error Handling Responsibility |
|---|---|
| Cloud Scheduler | Trigger Cloud Run Job on schedule |
| Cloud Run Job | Login, fetch notices, capture evidence, upload temporary GCS files, post payload to n8n |
| SchoolDiary Fetcher | Detect portal, extract notices, identify extraction failures |
| GCS | Temporary handoff storage |
| n8n Intake Workflow | Validate payload, dedupe, classify/itemize, write tracker rows, finalize evidence, create calendar events |
| Google Sheets | Store tracker rows, run logs, error records, digest logs, config |
| Google Drive | Store final evidence |
| Google Calendar | Store due-date visibility events |
| Email | Send daily digest and operational alerts |

## 8. Error Categories

Recommended top-level categories:

| Category | Meaning |
|---|---|
| `configuration` | Required runtime configuration missing or invalid |
| `authentication` | Login, OAuth, token, or credential failure |
| `navigation` | SchoolDiary page could not be reached or recognized |
| `extraction` | Notice content could not be extracted reliably |
| `classification` | Notice/homework classification failed or returned unusable output |
| `deduplication` | Hash/index logic failed or duplicate state could not be checked |
| `evidence` | Screenshot, attachment, GCS, or Drive evidence issue |
| `tracker_write` | Google Sheet row write/update failed |
| `calendar` | Calendar event creation or update failed |
| `notification` | Digest or operational alert send failed |
| `runtime` | Unexpected exception or infrastructure failure |
| `permission` | Google Drive/Sheet/Calendar/GCS permission failure |
| `rate_limit` | API quota or throttling issue |
| `manual_review` | Automation could not safely decide and needs human review |

## 9. Severity Levels

Recommended severity values:

| Severity | Meaning | Example |
|---|---|---|
| `info` | Normal operational event | Run completed with no issues |
| `warning` | Partial failure or review needed | Attachment failed but notice text captured |
| `error` | Workflow/run failed or action required | SchoolDiary login failed |
| `critical` | Repeated failure or blocked automation | Multiple consecutive daily capture failures |

MVP may use only:

```text
info
warning
error
```

`critical` can be introduced after consecutive-failure detection is implemented.

## 10. Cloud Run Job Exit Codes

Cloud Run Job should use stable exit codes.

Recommended exit codes:

| Exit Code | Meaning |
|---:|---|
| `0` | Success |
| `10` | Configuration failure |
| `11` | Secret retrieval failure |
| `20` | SchoolDiary authentication failure |
| `21` | SchoolDiary navigation failure |
| `22` | Notice extraction failure |
| `30` | Evidence upload failure |
| `40` | n8n webhook post failure |
| `50` | Unexpected runtime failure |
| `60` | Partial success |

Partial success means at least some notices were captured or posted, but one or more non-fatal steps failed.

## 11. Cloud Run Structured Logs

Cloud Run logs should be structured JSON where possible.

Safe example:

```json
{
  "run_id": "run_2026-07-03T20-30-00_Asia-Kolkata",
  "component": "cloud_run_job",
  "event": "run_complete",
  "severity": "info",
  "notices_seen": 12,
  "notices_extracted": 12,
  "attachments_downloaded": 2,
  "evidence_uploaded_to_gcs": 3,
  "payload_posted_to_n8n": true,
  "duration_ms": 182000
}
```

Do not log:

- SchoolDiary credentials
- n8n webhook token
- Authorization headers
- OAuth tokens
- Browser cookies
- Full private Google links
- Raw SchoolDiary notice text in technical logs
- Downloaded attachment contents

## 12. Run ID

Every scheduled execution should have a unique `run_id`.

Recommended format:

```text
run_<yyyy-mm-dd>T<hh-mm-ss>_<timezone>
```

Example:

```text
run_2026-07-03T20-30-00_Asia-Kolkata
```

The same `run_id` should appear in:

- Cloud Run logs
- n8n intake payload
- Google Sheet `Run Log`
- Google Sheet `Processing Errors`
- GCS temporary evidence path
- Google Drive evidence metadata/notes where practical

## 13. Correlation IDs

Recommended identifiers:

| Identifier | Purpose |
|---|---|
| `run_id` | Correlates one scheduled execution |
| `notice_hash` | Correlates one SchoolDiary notice |
| `notice_hash_short` | Safe short reference for folders/logs |
| `homework_hash` | Correlates one generated homework item |
| `evidence_id` | Correlates one evidence object |
| `calendar_event_id` | Correlates one calendar event |
| `digest_date` | Correlates one daily digest |

For logs and alerts, prefer short hashes unless full values are required in private troubleshooting.

## 14. Google Sheet Observability Tabs

The Google Sheet should include operational tabs.

Recommended tabs:

```text
Homework Tracker
Notice Index
Run Log
Processing Errors
Daily Digest Log
Config
```

Optional later tabs:

```text
Calendar Event Log
Evidence Log
Operational Metrics
```

## 15. Run Log Tab

The `Run Log` tab records one row per Cloud Run/n8n intake run.

Recommended fields:

| Field | Purpose |
|---|---|
| `Run ID` | Unique run identifier |
| `Triggered At` | Scheduler trigger timestamp |
| `Received At` | n8n webhook receipt timestamp |
| `Completed At` | n8n processing completion timestamp |
| `Status` | `success`, `partial_success`, `failed` |
| `Notices Received` | Count from payload |
| `Notices Processed` | Count processed by n8n |
| `Duplicates Skipped` | Count skipped due to notice/homework dedupe |
| `Homework Items Created` | New tracker rows created |
| `Needs Review Created` | Rows marked `Needs Review` |
| `Evidence Finalized` | Evidence files copied to Drive |
| `Evidence Failures` | Evidence failures |
| `Calendar Events Created` | Calendar events created |
| `Calendar Failures` | Calendar failures |
| `Digest Impact` | Whether new items may appear in digest |
| `Error Count` | Count of processing errors |
| `Notes` | Safe summary |

## 16. Processing Errors Tab

The `Processing Errors` tab stores structured errors.

Recommended fields:

| Field | Purpose |
|---|---|
| `Error ID` | Unique error identifier |
| `Run ID` | Related run |
| `Timestamp` | Error timestamp |
| `Component` | `cloud_run`, `fetcher`, `n8n`, `drive`, `calendar`, `sheets`, `email` |
| `Workflow Name` | n8n workflow if applicable |
| `Error Category` | Category from Section 8 |
| `Severity` | `info`, `warning`, `error`, `critical` |
| `Notice Hash Short` | Related notice if applicable |
| `Homework Hash Short` | Related homework item if applicable |
| `Error Code` | Stable code |
| `Safe Message` | Redacted human-readable summary |
| `Action Required` | `yes` or `no` |
| `Resolved` | `yes` or `no` |
| `Resolved At` | Timestamp |
| `Resolution Notes` | Safe notes |

## 17. Daily Digest Log Tab

The `Daily Digest Log` tab prevents duplicate family-facing emails.

Recommended fields:

| Field | Purpose |
|---|---|
| `Digest Date` | Local date in `Asia/Kolkata` |
| `Recipient Group` | Logical group |
| `Started At` | Digest workflow start |
| `Sent At` | Email sent timestamp |
| `Status` | `sent`, `skipped`, `failed` |
| `Item Count` | Count of included rows |
| `Needs Review Count` | Count |
| `Overdue Count` | Count |
| `Due Today Count` | Count |
| `Due Tomorrow Count` | Count |
| `Upcoming Count` | Count |
| `New Since Last Digest Count` | Count |
| `Message ID` | Provider message ID if available |
| `Notes` | Safe notes |

If `Status = sent` for the same `Digest Date` and `Recipient Group`, the workflow must not send another digest unless explicitly overridden.

## 18. Notice Index Tab

The `Notice Index` tab supports notice-level deduplication.

Recommended fields:

| Field | Purpose |
|---|---|
| `Notice Hash` | Primary notice dedupe key |
| `Notice Hash Short` | Human-readable short reference |
| `First Seen Run ID` | First run that saw the notice |
| `First Seen At` | Timestamp |
| `Last Seen At` | Timestamp |
| `Seen Count` | Number of times encountered |
| `Posted At Raw` | Raw SchoolDiary timestamp |
| `Posted Date` | Parsed posted date |
| `Sender Context` | Safe or redacted context |
| `Generated Title` | Synthetic/generated operational title |
| `Contains Homework` | `true`/`false` |
| `Processing Status` | `processed`, `duplicate`, `needs_review`, `failed` |
| `Drive Folder Link` | Final evidence folder link if available |
| `Notes` | Safe notes |

## 19. Homework Tracker Observability Fields

The `Homework Tracker` should include fields that support troubleshooting without exposing unnecessary technical detail.

Recommended observability fields:

| Field | Purpose |
|---|---|
| `Source Notice ID` | Link to source notice object |
| `Notice Hash` | Notice-level dedupe |
| `Homework Hash` | Homework-level dedupe |
| `Captured At` | Capture timestamp |
| `Confidence` | Extraction/classification confidence |
| `Status` | Current homework status |
| `Evidence Status` | `available`, `partial`, `failed`, `not_required`, `link_only` |
| `Processing Notes` | Safe notes |
| `Calendar Status` | `created`, `skipped`, `failed`, `not_required` |
| `Last Reminder Sent` | Reserved/future |
| `Completed At` | Human completion timestamp |

## 20. Error Codes

Stable error codes make troubleshooting easier.

### 20.1 Configuration

| Error Code | Meaning |
|---|---|
| `CONFIG_MISSING_REQUIRED_VALUE` | Required config value missing |
| `CONFIG_INVALID_VALUE` | Config value invalid |
| `CONFIG_PRIVATE_VALUE_EXPOSED` | Real private value detected in unsafe place |

### 20.2 SchoolDiary / Fetcher

| Error Code | Meaning |
|---|---|
| `SCHOOLDIARY_LOGIN_FAILED` | Login failed |
| `SCHOOLDIARY_INTERACTIVE_VERIFICATION_REQUIRED` | CAPTCHA/OTP/manual verification required |
| `NOTICE_BOARD_NAVIGATION_FAILED` | Notice Board could not be opened |
| `NOTICE_BOARD_NOT_RECOGNIZED` | Page structure not recognized |
| `NOTICE_EXTRACTION_FAILED` | Notice extraction failed |
| `NOTICE_PARTIAL_EXTRACTION` | Notice extracted with missing fields |
| `TIMESTAMP_PARSE_FAILED` | Posted timestamp could not be parsed |
| `ATTACHMENT_DOWNLOAD_FAILED` | Attachment download failed |
| `EVIDENCE_CAPTURE_FAILED` | Screenshot/text capture failed |

### 20.3 n8n / Processing

| Error Code | Meaning |
|---|---|
| `WEBHOOK_AUTH_FAILED` | Webhook token invalid |
| `PAYLOAD_VALIDATION_FAILED` | Payload does not match expected contract |
| `CLASSIFICATION_FAILED` | AI/deterministic classification failed |
| `HOMEWORK_ITEMIZATION_FAILED` | Notice could not be converted to homework items |
| `DEDUPLICATION_CHECK_FAILED` | Existing item lookup failed |
| `TRACKER_WRITE_FAILED` | Google Sheet write failed |
| `NOTICE_INDEX_WRITE_FAILED` | Notice Index write failed |
| `RUN_LOG_WRITE_FAILED` | Run Log write failed |

### 20.4 Evidence / Drive / GCS

| Error Code | Meaning |
|---|---|
| `GCS_OBJECT_READ_FAILED` | Temporary GCS object could not be read |
| `GCS_OBJECT_WRITE_FAILED` | Cloud Run could not write temporary evidence |
| `DRIVE_ROOT_FOLDER_MISSING` | Drive root folder config missing |
| `DRIVE_FOLDER_CREATE_FAILED` | Drive folder creation failed |
| `DRIVE_FILE_UPLOAD_FAILED` | Drive upload failed |
| `DRIVE_PERMISSION_DENIED` | n8n lacks Drive access |
| `EVIDENCE_LINK_WRITEBACK_FAILED` | Tracker evidence link update failed |

### 20.5 Calendar / Notification

| Error Code | Meaning |
|---|---|
| `CALENDAR_ID_MISSING` | Calendar ID missing |
| `CALENDAR_EVENT_CREATE_FAILED` | Calendar event creation failed |
| `CALENDAR_PERMISSION_DENIED` | n8n lacks calendar access |
| `DIGEST_RECIPIENT_MISSING` | Digest recipient config missing |
| `DIGEST_SEND_FAILED` | Family digest email failed |
| `OPERATIONAL_ALERT_SEND_FAILED` | Admin alert failed |
| `DUPLICATE_DIGEST_PREVENTED` | Digest already sent and duplicate was skipped |

## 21. Retry Strategy

Retries must be controlled and idempotent.

| Operation | Retry? | Notes |
|---|---:|---|
| SchoolDiary login | Limited | Avoid account lockout |
| Notice Board navigation | Limited | Useful for transient network issues |
| Notice extraction | No automatic deep retry | Selector issue likely requires fix |
| Attachment download | Limited | Can retry per attachment |
| GCS upload | Limited | Transient failures possible |
| n8n webhook post | Limited | Must include stable `run_id` and hashes |
| Google Sheet write | Limited | Must avoid duplicate rows |
| Google Drive upload | Limited | Reuse folder/file if already created |
| Calendar event create | Limited | Must check existing event by `homework_hash` |
| Daily digest send | Very limited | Must check `Daily Digest Log` before retry |

## 22. Idempotency Rules

Idempotency is mandatory.

Rules:

1. Same `notice_hash` must not create duplicate notice records.
2. Same `homework_hash` must not create duplicate tracker rows.
3. Same evidence hash/file must not create duplicate Drive evidence where avoidable.
4. Same `homework_hash` must not create duplicate calendar events.
5. Same digest date and recipient group must not send duplicate daily digests.
6. Retried workflow executions must check existing state before creating new records.

## 23. Partial Failure Behavior

The system should process what it can safely process.

Examples:

| Scenario | Expected Behavior |
|---|---|
| One attachment fails | Create tracker row if notice text is sufficient; mark evidence `partial` |
| Screenshot fails | Continue if text is captured |
| One notice extraction fails | Process other notices; record error |
| Calendar event fails | Tracker row remains; mark calendar status `failed` |
| Drive upload fails | Tracker row remains; mark evidence `failed` or `partial` |
| Digest fails | Record digest failure; send operational alert if possible |
| AI classification fails | Mark notice/homework `Needs Review` if actionable text exists |

## 24. Needs Review as Safe Fallback

When the automation cannot safely decide, it should prefer `Needs Review`.

Typical triggers:

- Missing due date
- Ambiguous subject
- Ambiguous instruction
- Conditional notice
- Attachment required but unavailable
- Public link required but inaccessible
- AI classification uncertainty
- Conflicting due dates
- Incomplete notice extraction

`Needs Review` is not a system failure. It is a controlled fallback state.

## 25. Family-Facing Versus Operational Errors

Family-facing digest should include only actionable homework context.

Operational alerts should include system health and failures.

| Error Type | Family Digest? | Operational Alert? |
|---|---:|---:|
| Missing homework due date | Yes, as Needs Review | No, unless repeated/high volume |
| SchoolDiary login failed | No | Yes |
| Drive permission denied | No or concise evidence issue | Yes |
| Calendar event creation failed | No by default | Yes if repeated |
| Attachment missing | Yes, as evidence issue if relevant | Warning if needed |
| Digest send failed | No | Yes |
| n8n credential expired | No | Yes |

## 26. Operational Alerts

Operational alerts go to the configured admin recipient.

Recommended recipient config:

```text
notifications.operational_alert_email_to
```

Operational alerts should be sent for:

- Cloud Run total failure
- SchoolDiary login failure
- Interactive verification required
- n8n webhook authentication failure
- Payload validation failure
- Google Sheet write failure
- Google Drive permission failure
- Google Calendar permission failure
- Daily digest send failure
- Repeated partial failures

## 27. Operational Alert Format

Recommended subject:

```text
SchoolDiary Automation Alert — <error category>
```

Recommended body:

```text
SchoolDiary Automation Alert

Time: <timestamp>
Run ID: <run_id>
Workflow: <workflow name>
Severity: <warning/error/critical>
Error Category: <category>
Error Code: <error_code>
Safe Details: <redacted summary>
Action Required: <yes/no>
Suggested Action: <safe next step>

No secrets, tokens, cookies, or private credential values are included in this alert.
```

## 28. Consecutive Failure Detection

After MVP, the system should detect repeated failures.

Example rule:

```text
If the same scheduled capture fails 3 times consecutively, raise severity to critical.
```

Potential repeated failure checks:

- Consecutive SchoolDiary login failures
- Consecutive n8n intake failures
- Consecutive Google Sheet write failures
- Consecutive digest send failures
- Multiple days without successful capture

This can be implemented using `Run Log` and `Processing Errors`.

## 29. Health Checks

Recommended MVP health checks:

| Health Check | Method |
|---|---|
| Cloud Run completed today | Check `Run Log` |
| Final daily capture completed | Check run at/after `20:30` |
| n8n intake working | Check latest `Run Log` status |
| Google Sheet writable | Check successful tracker/write log |
| Drive evidence writable | Check recent evidence finalization |
| Calendar writable | Check recent calendar status |
| Digest sent or skipped intentionally | Check `Daily Digest Log` |
| n8n MCP connection for development | Run `search_workflows` smoke test in Codex |

## 30. n8n MCP Development Health Check

For local development with Codex, n8n MCP can be checked by asking Codex to run an n8n MCP health check.

Success condition:

```text
search_workflows returns successfully
```

Acceptable successful response:

```json
{"data":[],"count":0}
```

This means the MCP connection is live even if no workflows exist yet.

If n8n MCP tools are not visible, trigger tool discovery for:

```text
get_sdk_reference
create_workflow_from_code
validate_workflow
search_workflows
```

This check is a development-time health check, not a production runtime dependency.

## 31. Metrics

Recommended MVP metrics:

| Metric | Source |
|---|---|
| Scheduled runs expected | Schedule config |
| Runs completed | Run Log |
| Runs failed | Run Log |
| Notices received | Run Log |
| Notices processed | Run Log |
| Duplicate notices skipped | Run Log / Notice Index |
| Homework rows created | Run Log / Tracker |
| Needs Review rows created | Run Log / Tracker |
| Evidence finalized | Run Log / Evidence fields |
| Evidence failures | Run Log / Processing Errors |
| Calendar events created | Run Log / Tracker |
| Calendar failures | Processing Errors |
| Digest sent | Daily Digest Log |
| Digest skipped | Daily Digest Log |
| Digest failed | Daily Digest Log / Processing Errors |

## 32. Operational Dashboard Direction

MVP can use Google Sheets as the operational dashboard.

Potential dashboard sections:

- Last successful capture
- Last successful digest
- Open operational errors
- Needs Review count
- Evidence failure count
- Overdue count
- Runs in last 7 days
- Failure count in last 7 days

A dedicated dashboard is optional for MVP.

## 33. Data Freshness Indicators

The tracker or dashboard should make data freshness visible.

Recommended freshness fields:

| Field | Purpose |
|---|---|
| `Last Successful Capture At` | Confirms notice fetch is working |
| `Last Successful Intake At` | Confirms n8n processing is working |
| `Last Successful Digest At` | Confirms digest is working |
| `Last Error At` | Indicates unresolved operational issue |
| `Open Error Count` | Indicates whether admin review is needed |

## 34. Log Redaction

All logs, alerts, and technical notes must follow the security design.

Do not log:

- Passwords
- Tokens
- OAuth credentials
- Cookies
- Authorization headers
- Full webhook URLs with tokens
- Raw SchoolDiary notice text in high-volume technical logs
- Attachment contents
- Browser session state
- Private config files

Safe logs may include:

- Run ID
- Component
- Error code
- Error category
- Severity
- Counts
- Redacted identifiers
- Hash prefixes
- Safe status values

## 35. Error Message Style

Error messages should be clear and actionable.

Poor:

```text
Failed.
```

Better:

```text
SchoolDiary login failed. Check stored username/password or whether interactive verification is required.
```

Poor:

```text
Google error.
```

Better:

```text
Google Drive upload failed due to permission error. Check that the n8n Google credential has access to the evidence root folder.
```

Do not include secrets or full private URLs.

## 36. AI Failure Handling

If AI is used for classification or digest formatting, failures must be handled safely.

Rules:

1. AI failure must not crash the whole workflow if deterministic fallback is possible.
2. If classification fails but notice text exists, create `Needs Review` where appropriate.
3. AI must not invent missing due dates.
4. AI must not mark homework completed.
5. AI must not suppress tracker rows.
6. AI output should be validated before use.
7. Invalid AI output should be recorded in `Processing Errors`.

Recommended AI error codes:

| Error Code | Meaning |
|---|---|
| `AI_CLASSIFICATION_FAILED` | AI call failed |
| `AI_OUTPUT_INVALID_JSON` | AI output could not be parsed |
| `AI_OUTPUT_SCHEMA_MISMATCH` | AI output failed schema validation |
| `AI_DIGEST_FORMATTING_FAILED` | Digest formatting failed |

## 37. Payload Validation

n8n should validate incoming payloads from Cloud Run.

Validation should check:

- Webhook token
- `run_id`
- `captured_at`
- `timezone`
- `notices` array
- Required notice fields
- Attachment/evidence structure
- Basic timestamp validity

If payload validation fails:

1. Do not process the payload.
2. Record error.
3. Send operational alert if appropriate.
4. Return failure to Cloud Run if synchronous response is used.

## 38. Calendar Error Handling

Calendar event creation failures should not block tracker row creation.

Calendar behavior:

| Scenario | Behavior |
|---|---|
| Missing due date | Do not create event; status `not_required` |
| Needs Review | Do not create event by default |
| Calendar ID missing | Record operational error |
| Calendar permission denied | Record operational error |
| Event already exists | Reuse existing event/link |
| Event creation fails | Mark calendar status `failed` |

Calendar failures should be visible in tracker fields and/or `Processing Errors`.

## 39. Evidence Error Handling

Evidence finalization failures should not block tracker row creation when notice text is sufficient.

Evidence behavior:

| Scenario | Behavior |
|---|---|
| Screenshot missing but text exists | Continue; evidence `partial` or `not_required` |
| Attachment download failed | Continue if task text exists; mark evidence `partial` |
| GCS object unreadable | Record evidence error |
| Drive folder creation failed | Record operational error |
| Drive upload failed | Mark evidence `failed` or `partial` |
| Public link malformed | Preserve safe note; do not fail whole notice |

## 40. Digest Error Handling

Digest workflow must avoid duplicate emails.

Rules:

1. Check `Daily Digest Log` before sending.
2. If no actionable items and `digest_send_empty=false`, skip and log.
3. If send fails, record `failed`.
4. Send operational alert if possible.
5. If retrying, re-check `Daily Digest Log` first.
6. Do not send repeated family-facing emails without explicit override.

## 41. Manual Override

Manual override should be rare.

Potential manual override scenarios:

- Re-send digest after confirmed failed send.
- Reprocess a specific run.
- Reprocess a specific notice.
- Clear an operational error after resolution.
- Mark a failed evidence item as accepted.

Manual overrides should leave notes in the relevant Sheet tab.

## 42. Recovery Playbooks

Detailed recovery steps belong in the deployment runbook, but this design expects future playbooks for:

- SchoolDiary login failure
- Interactive verification required
- n8n credential expired
- Google OAuth reauthorization
- Drive permission denied
- Sheet write failure
- Calendar creation failure
- Digest send failure
- Duplicate rows created accidentally
- Secret accidentally committed
- Evidence accidentally exposed

## 43. Open Decisions

The following decisions remain open:

1. Whether Google Cloud Monitoring dashboards are needed for MVP.
2. Whether n8n execution errors should be summarized into Google Sheets automatically.
3. Whether operational alerts should include a backup recipient.
4. Whether consecutive failure detection is included in MVP or deferred.
5. Whether a dedicated `Evidence Log` tab should be added in MVP.
6. Whether a dedicated `Calendar Event Log` tab should be added in MVP.
7. Whether digest failure should trigger an immediate retry or next-schedule retry only.
8. Whether secret scanning should be included in CI.
9. Whether an operational dashboard tab should be created in the first implementation.

## 44. Acceptance Criteria

The error handling and observability design is implemented correctly when:

1. Every scheduled run has a unique `run_id`.
2. Cloud Run writes safe structured logs.
3. n8n records run outcomes in `Run Log`.
4. n8n records processing failures in `Processing Errors`.
5. Daily digest sends/skips/failures are recorded in `Daily Digest Log`.
6. Duplicate homework rows are prevented.
7. Duplicate calendar events are prevented.
8. Duplicate daily digests are prevented.
9. Partial failures do not block unrelated notices.
10. Ambiguous actionable items are marked `Needs Review`.
11. Evidence failures are visible but do not block tracker creation when text is sufficient.
12. Calendar failures do not block tracker creation.
13. Operational alerts are separate from family-facing digest emails.
14. Logs and alerts redact secrets.
15. Family-facing emails do not expose internal technical details.
16. A failed credential/config state fails closed.
17. Health checks exist for capture, intake, evidence, calendar, and digest.
18. Manual recovery paths are identified for common failures.

## 45. Summary

The SchoolDiary Homework Automation system must be observable without exposing private data.

Cloud Run captures and logs safe run-level metrics. n8n records run status, processing errors, digest outcomes, evidence status, and calendar status in Google Sheets. Family-facing notifications remain focused on homework, while technical issues are routed to operational alerts.

The default error-handling posture is controlled continuation: process what can be processed safely, dedupe aggressively, mark ambiguity as `Needs Review`, and alert the parent/admin only when action is required.
