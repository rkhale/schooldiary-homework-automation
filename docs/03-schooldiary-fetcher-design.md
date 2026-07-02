# SchoolDiary Fetcher Design

## 1. Purpose

This document defines the design for the Playwright-based SchoolDiary fetcher used by the SchoolDiary Homework Automation system.

The fetcher runs inside the Google Cloud Run Job. It is responsible for authenticating to SchoolDiary, opening the configured Notice Board page, extracting raw notice blocks, preserving attachments and public links, capturing evidence, and returning structured source notice data that conforms to `docs/05-data-model.md`.

The fetcher must treat the SchoolDiary Notice Board as a general notice feed. It must not assume every notice is homework.

## 2. Scope

This document covers:

- SchoolDiary login flow
- Session handling
- Notice Board navigation
- Notice block extraction
- Sender and context extraction
- Posted timestamp extraction
- Notice body extraction
- Attachment detection and download
- Public link detection and preservation
- Screenshot and PDF evidence capture
- Notice hash input preparation
- Fetcher output contract
- Error handling
- Selector isolation
- Public repository safety

## 3. Out of Scope

This document does not define:

- Google Cloud Run Job deployment
- Cloud Scheduler setup
- n8n workflow node design
- LLM itemization prompt
- Google Sheet tracker implementation
- Google Drive final evidence upload
- Google Calendar event creation
- Email notification format
- CAPTCHA or OTP bypass
- Reverse engineering private APIs
- Persisting browser session state

Those topics are handled in other design documents.

## 4. Relationship to Other Documents

| Document | Relevance |
|---|---|
| `docs/00-high-level-project-design.md` | Defines the overall system objective |
| `docs/01-architecture.md` | Defines component boundaries |
| `docs/02-cloud-run-job-design.md` | Defines the runtime container and job behavior |
| `docs/05-data-model.md` | Defines the payload contract the fetcher must support |
| `docs/04-n8n-workflow-design.md` | Defines downstream processing after the fetcher posts payloads |
| `docs/10-error-handling-and-observability.md` | Expands operational error handling and run monitoring |

## 5. Design Position

The fetcher is a browser automation component.

It owns:

- Logging in to SchoolDiary.
- Opening the configured Notice Board.
- Reading visible notice content.
- Capturing raw source notice details.
- Downloading attachments where available.
- Preserving public links.
- Capturing evidence screenshots or PDFs.
- Returning structured source notice objects.

It does not own:

- Final homework classification decisions.
- Final homework item creation.
- Deduplication against Google Sheets.
- Google Drive final storage.
- Google Calendar creation.
- Email reminders.
- Completion tracking.

Those responsibilities belong to n8n.

## 6. Key Design Assumption

The SchoolDiary Notice Board is a mixed feed.

A single page may include:

- Homework notices.
- Resource notices.
- Announcements.
- Conditional instructions.
- Event notices.
- Non-actionable information.

Therefore, the fetcher must extract **source notices**, not final homework tasks.

The fetcher should capture each notice block with enough context for n8n and the LLM itemizer to classify and process it later.

## 7. High-Level Fetcher Flow

```text
Start fetcher
    ↓
Load runtime config and secrets
    ↓
Launch Playwright browser
    ↓
Open SchoolDiary login page or configured Notice Board URL
    ↓
Authenticate if required
    ↓
Navigate to configured Notice Board URL
    ↓
Wait for Notice Board content
    ↓
Identify visible notice blocks
    ↓
For each notice block:
        Extract sender/context line
        Extract posted timestamp
        Extract body text
        Extract attachments
        Extract public links
        Capture screenshot/PDF evidence where configured
        Build source notice object
    ↓
Return captured notices to Cloud Run Job
    ↓
Close browser
```

## 8. Login Flow

### 8.1 Login Inputs

The fetcher requires:

| Input | Source |
|---|---|
| SchoolDiary username | Secret Manager or injected environment variable |
| SchoolDiary password | Secret Manager or injected environment variable |
| SchoolDiary Notice Board URL | Secret Manager or injected environment variable |
| Headless mode | Runtime config |
| Timeout settings | Runtime config |

### 8.2 Login Behavior

The fetcher should:

1. Start a fresh browser context.
2. Open the configured SchoolDiary Notice Board URL.
3. Detect whether login is required.
4. If login page is shown, enter username and password.
5. Submit login form.
6. Wait for authenticated landing page or target Notice Board page.
7. Navigate to the configured Notice Board URL after login if necessary.
8. Confirm that Notice Board content is visible.

### 8.3 Failed Login Behavior

If login fails, the fetcher should:

1. Stop extraction.
2. Return a safe error object.
3. Avoid repeated aggressive login attempts.
4. Avoid logging credentials.
5. Avoid capturing screenshots that expose credentials.

Recommended error code:

```text
SCHOOLDIARY_LOGIN_FAILED
```

### 8.4 CAPTCHA or OTP Behavior

If CAPTCHA, OTP, or additional human verification appears, the fetcher must not attempt bypass.

Expected behavior:

1. Stop the run.
2. Return a safe error object.
3. Mark the run as failed.
4. Surface the issue for manual intervention.

Recommended error code:

```text
SCHOOLDIARY_INTERACTIVE_VERIFICATION_REQUIRED
```

## 9. Session Handling

### 9.1 MVP Session Model

For MVP, the fetcher should log in fresh on each scheduled run.

Reason:

- Avoids storing cookies or browser session state.
- Reduces public-repo and runtime security risk.
- Makes behavior easier to reason about.

### 9.2 Session State Persistence

Persisting browser session state is out of scope for MVP.

The fetcher should not write or reuse:

- Cookies.
- Browser storage state.
- Local storage.
- Session storage.
- Authentication tokens.

Any future decision to persist session state must be documented in `docs/09-security-and-secrets.md`.

## 10. Notice Board Navigation

The fetcher should navigate to the configured Notice Board URL after authentication.

The configured URL should not be hardcoded.

Expected behavior:

1. Read `SCHOOLDIARY_NOTICEBOARD_URL` from runtime configuration.
2. Open URL in Playwright page.
3. Wait for network idle or a stable page-ready condition.
4. Confirm the Notice Board feed is visible.
5. Continue only when notice content or a valid empty state is detected.

If the Notice Board cannot be loaded, return a safe error.

Recommended error code:

```text
NOTICE_BOARD_NAVIGATION_FAILED
```

## 11. Notice Block Extraction

### 11.1 Notice Block Definition

A notice block is one visible feed item on the SchoolDiary Notice Board.

Based on observed page behavior, a notice block may include:

- Sender line.
- Sender context.
- Class or channel context.
- Posted timestamp.
- Body text.
- Attachments.
- Public links.
- Informal title or no title.
- Notice order on the page.

The fetcher must not assume a formal notice title exists.

### 11.2 Fields to Extract Per Notice

Each notice block should attempt to extract:

| Field | Description |
|---|---|
| `dom_order` | Position of notice block on the page |
| `sender_name` | Sender or teacher name if visible |
| `sender_context` | Full visible sender/context line |
| `class_context` | Class or section if visible |
| `channel_context` | Notice Board or channel context if visible |
| `posted_at_raw` | Timestamp exactly as shown in the portal |
| `posted_at` | Parsed ISO timestamp where possible |
| `posted_date` | Parsed date where possible |
| `title` | Formal title if visible; otherwise null |
| `generated_title` | Short generated title based on body text/context |
| `body_text` | Visible notice body text |
| `attachments` | File attachment objects |
| `public_links` | Public link objects |
| `evidence_captures` | Screenshot/PDF/text evidence objects |
| `source_url` | Notice-specific URL if available and safe |
| `extraction_status` | `success`, `partial_success`, or `failed` |
| `extraction_notes` | Safe processing notes |

### 11.3 Extraction Granularity

The fetcher should extract one object per notice block.

It should not merge multiple visible notices into one object.

It should not split one notice into homework items. That is n8n’s responsibility.

## 12. Sender and Context Extraction

The sender/context area may include text similar to:

```text
Synthetic Teacher (from Synthetic Class Notice Board)
```

The fetcher should preserve the full line as `sender_context`.

Where possible, it should parse:

| Parsed Field | Example |
|---|---|
| `sender_name` | `Synthetic Teacher` |
| `class_context` | `Synthetic Class` |
| `channel_context` | `Notice Board` |

If parsing is unreliable, preserve `sender_context` and set parsed fields to null.

## 13. Timestamp Extraction

### 13.1 Raw Timestamp

The fetcher must capture the raw timestamp text as shown in the portal.

Example:

```text
Thursday, June 25, 2026 12:33:39 PM
```

This value should be stored as:

```text
posted_at_raw
```

### 13.2 Parsed Timestamp

The fetcher should attempt to parse the raw timestamp into ISO-8601 format using the configured timezone.

Example:

```text
2026-06-25T12:33:39+05:30
```

This value should be stored as:

```text
posted_at
```

### 13.3 Parsed Date

The fetcher should also derive:

```text
posted_date = YYYY-MM-DD
```

If timestamp parsing fails, `posted_at` and `posted_date` may be null, but `posted_at_raw` should still be preserved.

## 14. Body Text Extraction

The fetcher should extract visible notice body text from each notice block.

Rules:

1. Preserve meaningful line breaks where possible.
2. Remove repeated UI-only whitespace.
3. Do not remove subject labels, exercise numbers, chapter names, due dates, or instructions.
4. Do not attempt to classify final homework in the fetcher.
5. Do not translate or rewrite notice content.
6. Preserve original wording as much as possible.

The fetcher may produce a normalized version only for hashing, but the original extracted text should remain available.

## 15. Generated Title

Because notice blocks may not have formal titles, the fetcher should generate a short title for operational use.

Examples:

| Notice Text | Generated Title |
|---|---|
| `Math homework: Complete Ex 3A.` | `Mathematics homework` |
| `Learning aid document has been uploaded...` | `Resource notice` |
| `Science journals have been distributed...` | `Science conditional action` |

The generated title should be brief and should not replace the original body text.

If title generation is uncertain, use:

```text
SchoolDiary notice
```

Final classification remains with n8n.

## 16. Attachment Detection and Download

### 16.1 Attachment Detection

The fetcher should detect file attachments linked from a notice block.

Potential file types:

- PDF
- Image
- Document
- Audio
- Other downloadable school resource

### 16.2 Attachment Download Behavior

When `DOWNLOAD_ATTACHMENTS=true`, the fetcher should attempt to download attachments.

For each attachment, capture:

| Field | Description |
|---|---|
| `attachment_id` | Generated ID |
| `file_name` | Sanitized file name |
| `mime_type` | MIME type if known |
| `source_attachment_url` | Source URL if safe to store |
| `download_status` | `downloaded`, `failed`, `skipped`, or `not_applicable` |
| `local_path` | Temporary local path for Cloud Run internal use |
| `size_bytes` | File size if known |
| `hash_sha256` | File hash if downloaded |
| `error_message` | Safe error message if download failed |

The final payload sent to n8n should include the GCS object path after the Cloud Run Job uploads the file.

### 16.3 Attachment Failure

If an attachment fails to download, the fetcher should:

1. Continue processing the notice where possible.
2. Mark the attachment as `failed`.
3. Add a safe error message.
4. Preserve the notice body and public links.
5. Avoid failing the entire run unless attachment failure blocks the complete capture process.

Recommended error code:

```text
ATTACHMENT_DOWNLOAD_FAILED
```

## 17. Public Link Detection and Preservation

The fetcher should detect links inside each notice block.

A public link may appear as:

- Anchor tag.
- Raw URL in text.
- External platform link.
- Resource link.
- Assignment link.

### 17.1 Public Link Object

For each link, capture:

| Field | Description |
|---|---|
| `link_id` | Generated ID |
| `url` | URL |
| `display_text` | Anchor text or surrounding text |
| `link_type` | `public_reference`, `assignment_link`, `resource_link`, `external_platform`, or `unknown` |
| `capture_required` | Default false |
| `access_assumption` | `public`, `unknown`, or `requires_login` |
| `notes` | Safe notes |

### 17.2 Handling Rule

Public links should be preserved as references.

The fetcher should not download or mirror public links into file storage unless a later design explicitly requires that behavior.

## 18. Evidence Capture

The fetcher should capture evidence so guardians can later review what the SchoolDiary page showed.

### 18.1 Preferred Evidence

Preferred evidence per notice:

1. Notice block screenshot.
2. Downloaded original attachments.
3. Public links as references.
4. Extracted text.

### 18.2 Screenshot Capture

When `CAPTURE_SCREENSHOT=true`, the fetcher should attempt to capture a screenshot of each notice block.

Recommended behavior:

1. Capture screenshot at notice-block level.
2. Use safe file names.
3. Store in local temporary evidence directory.
4. Return local path to Cloud Run Job for GCS upload.
5. Add evidence object to notice payload.

### 18.3 Screenshot Fallback

If notice-block screenshot fails, the fetcher may capture:

1. Full page screenshot.
2. Viewport screenshot.
3. Extracted text evidence only.

The failure should be recorded as a warning, not necessarily a fatal error.

### 18.4 PDF Capture

PDF capture is optional for MVP.

If enabled later, it should follow the same evidence object model as screenshots.

## 19. Notice Hash Preparation

The fetcher should prepare enough stable input for the Cloud Run Job or downstream code to calculate `notice_hash`.

Recommended hash input:

```text
source_system
+ posted_at_raw
+ sender_context
+ normalized_body_text
+ normalized_attachment_names
+ normalized_public_links
```

Fallback when `posted_at_raw` is unavailable:

```text
source_system
+ captured_date
+ sender_context
+ normalized_body_text
+ normalized_attachment_names
+ normalized_public_links
```

Rules:

1. Do not include `captured_at` in the primary hash if a source timestamp exists.
2. Do not include local file paths.
3. Do not include GCS paths.
4. Do not include generated run IDs.
5. Sort attachment names and public links before hashing.
6. Normalize whitespace before hashing.

## 20. Fetcher Output Contract

The fetcher should return a structured result to the Cloud Run Job.

### 20.1 Fetcher Result Object

```json
{
  "source_system": "schooldiary",
  "fetch_status": "success",
  "notices": [],
  "errors": []
}
```

### 20.2 Source Notice Object

Each notice should align with `docs/05-data-model.md`.

Synthetic example:

```json
{
  "notice_id": "notice_fake_001",
  "notice_hash": "sha256_fake_notice_hash",
  "source_system": "schooldiary",
  "page_context": {
    "school_label": null,
    "page_title": "Notice Board",
    "channel_id_present": true,
    "class_or_section": "Synthetic Class",
    "academic_year_label": null
  },
  "dom_order": 2,
  "posted_at_raw": "Thursday, June 25, 2026 12:33:39 PM",
  "posted_at": "2026-06-25T12:33:39+05:30",
  "posted_date": "2026-06-25",
  "captured_at": "2026-06-25T15:30:00+05:30",
  "sender_name": "Synthetic Teacher",
  "sender_context": "Synthetic Teacher (from Synthetic Class Notice Board)",
  "class_context": "Synthetic Class",
  "channel_context": "Notice Board",
  "title": null,
  "generated_title": "Mathematics homework",
  "body_text": "Good afternoon. Math homework: Complete Ex 3A.",
  "notice_type": "unknown",
  "actionability": "needs_review",
  "classification_confidence": "Low",
  "classification_notes": "Classification deferred to n8n.",
  "contains_homework": false,
  "requires_guardian_review": true,
  "attachments": [],
  "public_links": [],
  "evidence_captures": [],
  "source_url": null,
  "extraction_status": "success",
  "extraction_notes": ""
}
```

### 20.3 Classification Defaults

The fetcher should not make final classification decisions.

Default values may be:

| Field | Default |
|---|---|
| `notice_type` | `unknown` |
| `actionability` | `needs_review` |
| `classification_confidence` | `Low` |
| `classification_notes` | `Classification deferred to n8n.` |
| `contains_homework` | `false` |
| `requires_guardian_review` | `true` |

n8n may overwrite or enrich these fields during classification.

## 21. Selector Strategy

All SchoolDiary selectors must be isolated.

Recommended structure:

```text
src/
├── schooldiary_fetcher.py
├── schooldiary_selectors.py
└── selector_utils.py
```

Rules:

1. Do not scatter selectors throughout the codebase.
2. Keep selectors named by business meaning, not CSS shape only.
3. Prefer stable semantic selectors where available.
4. Use fallback selectors where the UI is inconsistent.
5. Log selector failure safely.
6. Do not commit real page HTML unless sanitized.

Example selector names:

```text
LOGIN_USERNAME_INPUT
LOGIN_PASSWORD_INPUT
LOGIN_SUBMIT_BUTTON
NOTICE_BOARD_CONTAINER
NOTICE_CARD
NOTICE_SENDER_CONTEXT
NOTICE_TIMESTAMP
NOTICE_BODY
NOTICE_ATTACHMENT_LINK
NOTICE_PUBLIC_LINK
```

Actual selector values are implementation details and should be verified during coding.

## 22. Page Readiness and Empty State

The fetcher should distinguish between:

1. Notice Board loaded with notices.
2. Notice Board loaded with no notices.
3. Notice Board failed to load.
4. Login page still visible.
5. Error page or unauthorized page.

If no notices are found but the page appears valid, return success with an empty notices array.

If the page structure cannot be recognized, return an extraction or navigation failure.

## 23. Error Handling

### 23.1 Fetcher-Level Errors

| Error Code | Meaning |
|---|---|
| `SCHOOLDIARY_LOGIN_FAILED` | Login failed |
| `SCHOOLDIARY_INTERACTIVE_VERIFICATION_REQUIRED` | CAPTCHA, OTP, or manual verification required |
| `NOTICE_BOARD_NAVIGATION_FAILED` | Notice Board page could not be opened |
| `NOTICE_BOARD_NOT_RECOGNIZED` | Expected page structure not found |
| `NOTICE_EXTRACTION_FAILED` | Notice extraction failed |
| `NOTICE_PARTIAL_EXTRACTION` | Some fields could not be extracted |
| `ATTACHMENT_DOWNLOAD_FAILED` | Attachment download failed |
| `PUBLIC_LINK_EXTRACTION_FAILED` | Link extraction failed |
| `EVIDENCE_CAPTURE_FAILED` | Screenshot or PDF capture failed |
| `TIMESTAMP_PARSE_FAILED` | Raw timestamp could not be parsed |

### 23.2 Error Severity

| Severity | Usage |
|---|---|
| `critical` | Login failure, blocked access, page inaccessible |
| `error` | Notice extraction cannot proceed |
| `warning` | Partial extraction, attachment failure, screenshot failure |
| `info` | Non-blocking observations |

### 23.3 Partial Extraction

Partial extraction should be allowed when the fetcher can still preserve useful notice text or evidence.

Example:

- Timestamp parse fails but body text is captured.
- Attachment download fails but body text and link are captured.
- Screenshot fails but extracted text is available.

## 24. Timeout and Retry Behavior

Retry behavior should be conservative.

| Operation | Retry Guidance |
|---|---|
| Login page load | Retry once |
| Login submit | Retry once only if clearly transient |
| Notice Board navigation | Retry once or twice |
| Notice extraction | Do not retry indefinitely |
| Attachment download | Limited retry |
| Screenshot capture | Retry once, then mark failed |
| Timestamp parse | Do not retry; preserve raw value |
| Public link extraction | Do not fail entire notice |

Avoid aggressive retries that could lock the account or create portal abuse risk.

## 25. Logging Rules

The fetcher should produce safe structured logs.

Safe to log:

- Run ID.
- Component name.
- Number of notice blocks found.
- Number of attachments found.
- Number of public links found.
- Safe error codes.
- Extraction status.
- Duration.

Not safe to log:

- Username.
- Password.
- Cookies.
- Session tokens.
- Full authenticated URLs.
- Raw credentials.
- Browser storage state.
- Real notice content in public examples.
- Screenshots or file content.

## 26. Public Repository Safety

The repository must not include:

- Real SchoolDiary URLs.
- Real login credentials.
- Real student names.
- Real guardian names.
- Real teacher names.
- Real school names.
- Real class or section details.
- Real notice text.
- Real screenshots.
- Real PDFs.
- Real downloaded attachments.
- Browser cookies.
- Browser storage state.
- Real n8n webhook URLs.
- Real Google Cloud project IDs.
- Real service account keys.

Allowed:

- Synthetic examples.
- Placeholder URLs.
- Fake notice text.
- Fake sender names.
- Fake class labels.
- Fake timestamps.
- Fake hashes.
- `example.com` public links.

## 27. Local Development Behavior

The fetcher should support local development with placeholder or real private configuration loaded outside git.

Local development should support:

- `HEADLESS=true` or `HEADLESS=false`
- `POST_TO_N8N=false`
- Local output directory
- Synthetic fixtures
- Optional screenshot capture
- Optional attachment download

PowerShell example:

```powershell
$env:POST_TO_N8N = "false"
$env:HEADLESS = "false"
$env:OUTPUT_DIR = "C:\\Temp\\schooldiary-homework"

python -m src.main
```

Do not commit local output.

## 28. Testing Strategy for Fetcher

Detailed testing is covered in `docs/12-testing-strategy.md`, but the fetcher should support:

1. Unit tests for timestamp parsing.
2. Unit tests for notice hash normalization.
3. Unit tests for public link extraction.
4. Unit tests for generated title fallback.
5. Unit tests for safe error object creation.
6. Fixture-based tests with sanitized HTML.
7. Local smoke test against configured private environment.

Real SchoolDiary HTML must not be committed unless fully sanitized.

## 29. Open Decisions

The following decisions remain open:

1. Exact login selectors.
2. Exact notice card selector.
3. Whether page-level screenshot is always captured.
4. Whether notice-block screenshot is technically reliable.
5. Whether source notice URLs are available and safe to store.
6. Whether some links require login and should be marked `requires_login`.
7. Whether attachments open in new tabs or download directly.
8. Whether SchoolDiary rate limits repeated logins.
9. Whether session persistence is required later.
10. Whether `contains_homework` should remain default false or be lightly inferred by the fetcher.

## 30. Acceptance Criteria

The fetcher design is implemented correctly when:

1. It logs in using runtime secrets.
2. It navigates to the configured Notice Board URL.
3. It extracts one object per visible notice block.
4. It captures sender/context, timestamp, body text, attachments, public links, and evidence where available.
5. It preserves raw timestamp text and parsed timestamp where possible.
6. It does not assume every notice is homework.
7. It does not create final homework items.
8. It returns source notice objects aligned to `docs/05-data-model.md`.
9. It records partial failures without discarding useful notice data.
10. It keeps selectors isolated.
11. It avoids logging or committing secrets, cookies, real screenshots, or real portal data.

## 31. Summary

The SchoolDiary fetcher is the capture component inside the Cloud Run Job. Its job is to authenticate, read the Notice Board feed, preserve source notices and evidence, and return structured raw notice data.

It should remain narrow, conservative, and public-repo safe. Final homework classification, task creation, tracking, calendarization, and notifications remain downstream responsibilities owned by n8n.
