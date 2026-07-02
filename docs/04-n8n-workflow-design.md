# n8n Workflow Design

## 1. Purpose

This document defines the n8n workflow design for the SchoolDiary Homework Automation system.

n8n receives the structured capture payload from the Cloud Run Job, validates it, deduplicates notices, classifies SchoolDiary notices, itemizes homework tasks, finalizes evidence into Google Drive, updates Google Sheets, creates Google Calendar events where appropriate, and sends a consolidated daily digest email.

n8n is the orchestration layer for the household workflow. The Cloud Run Job captures source notices. n8n turns those notices into trackable homework operations.

## 2. Scope

This document covers:

- n8n workflow inventory
- Intake webhook behavior
- Webhook token validation
- Payload validation
- Notice deduplication
- Homework item deduplication
- Notice classification
- LLM-assisted homework itemization
- Evidence finalization from GCS to Google Drive
- Google Sheets tracker updates
- Google Calendar event creation
- Manual completion handling
- Daily digest notification model
- Overdue status update
- Weekly summary workflow
- Error handling
- Idempotency rules
- Public repository safety

## 3. Out of Scope

This document does not define:

- SchoolDiary browser automation
- Cloud Run Job implementation
- Google Cloud deployment commands
- Exact Playwright selectors
- Full LLM prompt text
- Google Cloud Storage lifecycle policy
- Google Workspace credential setup steps
- Production monitoring dashboards
- WhatsApp integration
- Mobile app UI
- Student-facing web dashboard
- Automated proof-of-completion detection

Those are covered in other design or implementation documents.

## 4. Relationship to Other Documents

| Document | Relevance |
|---|---|
| `docs/00-high-level-project-design.md` | Defines overall project objective |
| `docs/01-architecture.md` | Defines component boundaries |
| `docs/02-cloud-run-job-design.md` | Defines the job that posts payloads to n8n |
| `docs/03-schooldiary-fetcher-design.md` | Defines source notice capture behavior |
| `docs/05-data-model.md` | Defines source notice, homework item, and tracker data structures |
| `docs/06-google-calendar-design.md` | Will define calendar event details |
| `docs/07-google-drive-evidence-design.md` | Will define evidence folder/file structure |
| `docs/08-notification-design.md` | Will define email templates and notification rules |
| `docs/10-error-handling-and-observability.md` | Will expand operational visibility and alerting |

## 5. Design Position

n8n owns downstream workflow orchestration.

It owns:

- Webhook intake.
- Payload validation.
- Deduplication.
- Notice classification.
- LLM itemization.
- Evidence finalization.
- Tracker updates.
- Calendar event creation.
- Daily digest email.
- Overdue status update.
- Weekly summary.

n8n does not own:

- SchoolDiary browser login.
- Notice Board scraping.
- Attachment download from the authenticated SchoolDiary page.
- Cloud Run runtime execution.
- Google Cloud Scheduler execution.
- Automatic homework completion inference.
- Public web UI.

## 6. MVP Notification Principle

MVP uses a daily digest notification model to avoid email flooding.

The intake workflow must not send one family-facing email per homework item. It should create or update tracker rows, finalize evidence, and mark rows as digest-eligible where appropriate.

The daily digest workflow sends one consolidated family-facing email per day.

The AI may help format and summarize the digest, but deterministic n8n logic must enforce:

1. One family-facing digest email per day.
2. No duplicate homework rows.
3. No duplicate calendar events.
4. No duplicate digest entries.
5. Exclusion of completed and not-applicable items.
6. Preservation of manual tracker updates.

## 7. Workflow Inventory

The MVP should include four n8n workflows.

| Workflow | Purpose | Trigger |
|---|---|---|
| `01 - SchoolDiary Homework Intake` | Receive Cloud Run payload, process notices, update tracker, create eligible calendar events | Webhook |
| `02 - Daily Homework Digest` | Send one consolidated daily family-facing email | Scheduled |
| `03 - Overdue Status Update` | Mark overdue rows before digest processing | Scheduled |
| `04 - Weekly Summary` | Send weekly homework summary | Scheduled |

The intake workflow is mandatory for MVP. The daily digest workflow is also required for useful household operation. The overdue status update workflow should run before the daily digest. Weekly summary can be added after the intake and daily digest paths are stable.

## 8. Shared Data Stores

n8n uses Google Sheets as the MVP operational database.

Recommended workbook tabs:

| Tab | Purpose |
|---|---|
| `Homework Tracker` | Master operational homework tracker |
| `Notice Index` | Deduplication index for captured SchoolDiary notices |
| `Run Log` | One row per Cloud Run capture run processed by n8n |
| `Processing Errors` | Safe operational errors |
| `Daily Digest Log` | One row per digest email attempt |
| `Config` | Non-secret workflow configuration |

Secrets must not be stored in Google Sheets.

## 9. Google Sheets Role

Google Sheets is the MVP system of record.

It stores:

- Homework rows.
- Status.
- Due dates.
- Evidence links.
- Calendar links.
- Digest eligibility.
- Reminder/digest timestamps.
- Completion timestamps.
- Completion notes.
- Processing notes.

Manual status updates in Google Sheets must be respected by n8n.

n8n should not overwrite manually changed fields unless the workflow is explicitly designed to update that field.

## 10. Google Drive Role

Google Drive is the final evidence repository for family-facing review.

Google Drive stores:

- Notice screenshots.
- Downloaded attachments copied from temporary GCS handoff.
- Any future generated PDFs.
- Optional extracted text files, if enabled later.

Google Drive links are written back to the Google Sheet tracker.

## 11. Google Calendar Role

Google Calendar is used for due-date visibility.

n8n creates calendar events only when the homework item has a valid due date and the item is actionable.

Calendar creation should be skipped when:

- `due_date` is null.
- Status is `Needs Review`.
- Status is `For Information`.
- Status is `Not Applicable`.
- Status is `Completed`.
- A calendar event already exists for the homework item.

## 12. Email Role

Email is the MVP notification channel.

MVP sends one consolidated family-facing daily digest email.

The daily digest may include:

- New homework captured since the previous digest.
- Items needing review.
- Overdue items.
- Items due today or tomorrow.
- Upcoming homework.
- Evidence or attachment issues, if relevant.

The intake workflow should not send immediate family-facing emails per homework item.

System or operational failure alerts may be added later, but they should be separate from family-facing homework notifications.

WhatsApp is out of scope for MVP.

## 13. Workflow 01 — SchoolDiary Homework Intake

### 13.1 Purpose

This workflow receives the capture payload from the Cloud Run Job and converts captured SchoolDiary notices into tracker rows, evidence records, and calendar events.

It does not send family-facing homework emails in MVP.

### 13.2 Trigger

Trigger type:

```text
n8n Webhook
```

Expected caller:

```text
Google Cloud Run Job
```

Expected payload:

```text
Cloud Run capture payload matching docs/05-data-model.md
```

### 13.3 High-Level Flow

```text
Webhook receives payload
    ↓
Validate webhook token
    ↓
Validate payload structure
    ↓
Create or update Run Log row
    ↓
For each source notice:
        Check Notice Index by notice_hash
        Skip already-processed notices unless update behavior is enabled
        Finalize evidence from GCS to Google Drive
        Classify notice
        Itemize homework where applicable
        For each homework item:
            Calculate or verify homework_hash
            Check Homework Tracker for existing homework_hash
            Create or update tracker row
            Create calendar event if eligible
            Mark row as digest-eligible
        Record Notice Index row
    ↓
Record safe errors
    ↓
Return webhook response
```

### 13.4 No Immediate Family-Facing Email

The intake workflow must not send a separate email for every new homework item.

Instead, it should:

1. Create or update tracker rows.
2. Set digest-related fields.
3. Allow the daily digest workflow to send one consolidated email.

Recommended tracker fields:

| Field | Purpose |
|---|---|
| `Digest Eligible` | Whether item should appear in daily digest |
| `First Digest Sent At` | Prevent repeated first-notification treatment |
| `Last Digest Included At` | Track last digest inclusion |
| `Digest Notes` | Optional safe notes |

These may be added during implementation if useful.

## 14. Webhook Validation

### 14.1 Token Validation

For MVP, the webhook token may be provided in the request body as defined in `docs/05-data-model.md`.

Expected field:

```json
{
  "webhook_token": "placeholder-only"
}
```

n8n must compare this value against the token stored in n8n credentials or environment configuration.

Invalid token behavior:

1. Stop workflow.
2. Return unauthorized response.
3. Do not process payload.
4. Do not log the provided token.

Recommended response:

```text
401 Unauthorized
```

### 14.2 Future Header-Based Token

A later production hardening step may move the token to an HTTP header.

Preferred future header:

```text
X-SchoolDiary-Webhook-Token
```

The token must not be included in the webhook URL.

## 15. Payload Validation

n8n must validate the incoming payload before processing.

### 15.1 Required Top-Level Fields

| Field | Required | Description |
|---|---:|---|
| `webhook_token` | Yes | Token used for webhook validation |
| `run` | Yes | Capture run object |

### 15.2 Required Run Fields

| Field | Required | Description |
|---|---:|---|
| `run.run_id` | Yes | Unique capture run identifier |
| `run.source_system` | Yes | Expected value: `schooldiary` |
| `run.captured_at` | Yes | ISO-8601 capture timestamp |
| `run.timezone` | Yes | Runtime timezone |
| `run.status` | Yes | `success`, `partial_success`, or `failed` |
| `run.notices` | Yes | Array of source notice objects |
| `run.errors` | Yes | Array of safe error objects |

### 15.3 Invalid Payload Behavior

If the payload is invalid:

1. Stop notice processing.
2. Write a safe error to `Processing Errors` if enough safe context exists.
3. Return a failure response.
4. Do not create homework rows.
5. Do not send family-facing notifications.

Recommended response:

```text
400 Bad Request
```

## 16. Run Log Handling

n8n should record each received capture run in the `Run Log` tab.

Recommended columns:

| Column | Description |
|---|---|
| `Run ID` | Capture run ID |
| `Source System` | Expected value: `schooldiary` |
| `Captured At` | Cloud Run capture timestamp |
| `Processed At` | n8n processing timestamp |
| `Run Status` | Incoming run status |
| `n8n Processing Status` | `success`, `partial_success`, or `failed` |
| `Notice Count` | Number of notices in payload |
| `Created Homework Count` | Number of new homework rows created |
| `Skipped Duplicate Count` | Number of duplicate notices/items skipped |
| `Needs Review Count` | Number of review rows created |
| `Digest Eligible Count` | Number of rows eligible for daily digest |
| `Error Count` | Number of safe errors recorded |
| `Processing Notes` | Safe notes |

Run Log must not contain secrets or raw authenticated URLs.

## 17. Notice Deduplication

n8n should use `notice_hash` as the primary notice dedupe key.

### 17.1 Notice Index

The `Notice Index` tab should store one row per processed source notice.

Recommended columns:

| Column | Description |
|---|---|
| `Notice Hash` | Primary dedupe key |
| `Source Notice ID` | Notice ID from payload |
| `Run ID` | First run that processed this notice |
| `First Seen At` | First processing timestamp |
| `Last Seen At` | Most recent processing timestamp |
| `Notice Posted At` | Parsed notice timestamp |
| `Notice Posted Raw` | Raw notice timestamp |
| `Sender Context` | Sender/context line |
| `Generated Title` | Generated title |
| `Notice Type` | Classification |
| `Actionability` | Classification |
| `Processing Status` | `processed`, `duplicate`, `error`, or `ignored` |
| `Tracker Rows Created` | Count |
| `Processing Notes` | Safe notes |

### 17.2 Duplicate Notice Behavior

If `notice_hash` already exists in `Notice Index`:

1. Do not create duplicate homework rows.
2. Do not create duplicate evidence records.
3. Do not create duplicate calendar events.
4. Do not create duplicate digest entries.
5. Update `Last Seen At`.
6. Record duplicate skip count in `Run Log`.

This implements the core rule:

```text
Duplicate notices do not create duplicate homework rows, evidence records, or calendar events.
```

## 18. Evidence Finalization

n8n finalizes evidence from temporary GCS storage into Google Drive.

### 18.1 Evidence Inputs

Evidence may arrive as:

- GCS object paths for screenshots.
- GCS object paths for downloaded attachments.
- Public links.
- Extracted text references.

### 18.2 Evidence Flow

```text
GCS temporary object
    ↓
n8n reads object using configured credential
    ↓
n8n uploads/copies object into Google Drive
    ↓
n8n stores final Google Drive link
    ↓
n8n writes evidence link to Homework Tracker
```

### 18.3 Google Drive Folder Pattern

Detailed folder structure is defined in `docs/07-google-drive-evidence-design.md`.

Recommended MVP pattern:

```text
SchoolDiary Homework Evidence/
└── yyyy-mm-dd/
    └── run_id/
        └── notice_id/
            ├── notice_screenshot.png
            └── attachment_files
```

### 18.4 Evidence Failure Behavior

If evidence finalization fails:

1. Continue processing the notice if body text is available.
2. Mark the evidence item as failed.
3. Record safe error.
4. Create tracker row with `Needs Review` if evidence is required to understand the homework.
5. Do not fail the entire run unless evidence failure affects all notices.

## 19. Notice Classification

n8n classifies each source notice before itemization.

### 19.1 Classification Values

Classification must use values defined in `docs/05-data-model.md`.

Notice type values:

```text
homework
resource_notice
announcement
conditional_action
event_notice
unknown
```

Actionability values:

```text
action_required
for_information
conditional
needs_review
```

### 19.2 Classification Method

MVP classification should use:

1. Lightweight deterministic pre-checks.
2. LLM classification for ambiguous or content-rich notices.
3. Conservative fallback to `Needs Review`.

### 19.3 Deterministic Pre-Checks

Examples:

| Condition | Likely Classification |
|---|---|
| Empty body with attachment/link | `unknown`, `needs_review` |
| Contains clear homework wording | Candidate `homework` |
| Contains only uploaded resource wording | Candidate `resource_notice` |
| Contains absence/applicability condition | Candidate `conditional_action` |
| Contains only general information | Candidate `announcement` |

Deterministic pre-checks should not overrule unclear content.

### 19.4 Classification Output

Expected output:

```json
{
  "notice_hash": "sha256_fake_notice_hash",
  "notice_type": "homework",
  "actionability": "action_required",
  "contains_homework": true,
  "requires_guardian_review": false,
  "classification_confidence": "High",
  "classification_notes": "Notice contains explicit homework wording and a specific task."
}
```

## 20. LLM Homework Itemization

n8n uses LLM itemization only after classification.

### 20.1 When to Itemize

Itemization should run for:

| Notice Type | Itemization Behavior |
|---|---|
| `homework` | Yes |
| `conditional_action` | Yes, usually with `Needs Review` |
| `resource_notice` | Only if there is an implied student action or configured retention |
| `announcement` | No, unless action is detected |
| `event_notice` | No for homework tracker unless action exists |
| `unknown` | Create `Needs Review` row or run conservative itemization |

### 20.2 Itemization Rules

The LLM must:

1. Return JSON only.
2. Create one item per distinct homework task.
3. Use only the source notice and evidence metadata provided.
4. Preserve exercise numbers, chapter names, worksheet names, file names, submission instructions, and due dates.
5. Use `null` when due date is not stated.
6. Not invent due dates.
7. Not infer relative dates unless source notice date is available.
8. Mark unclear items as `Needs Review`.
9. Add ambiguity notes.
10. Keep conditionality explicit.

### 20.3 Expected Itemization Output

Example:

```json
[
  {
    "subject": "Mathematics",
    "homework_task": "Complete Ex 3A.",
    "due_date": null,
    "due_time": null,
    "source_notice_date": "2026-06-25",
    "teacher_name": "Synthetic Teacher",
    "required_materials": null,
    "evidence_needed": "Completed exercise",
    "is_conditional": false,
    "conditionality_notes": "",
    "confidence": "Medium",
    "status_recommendation": "Needs Review",
    "ambiguity_notes": "Due date is not stated in the notice."
  }
]
```

## 21. Homework Item Deduplication

n8n should calculate or verify `homework_hash` for each homework item.

Primary dedupe key:

```text
homework_hash
```

If an item with the same `homework_hash` already exists in `Homework Tracker`:

1. Do not create another tracker row.
2. Do not create another calendar event.
3. Do not send another notification or duplicate digest entry.
4. Optionally update evidence links if missing.
5. Record duplicate skip.

## 22. Tracker Row Creation

n8n writes actionable or review-required items to `Homework Tracker`.

### 22.1 Row Creation Rules

Create a row when:

| Condition | Create Tracker Row |
|---|---|
| Clear homework task | Yes |
| Conditional action | Yes, usually `Needs Review` |
| Unknown but possibly actionable | Yes, `Needs Review` |
| Resource notice with no action | Optional; default no |
| Pure announcement | No by default |
| Event notice with no homework/action | No by default |

### 22.2 Status Mapping

| Item Condition | Status |
|---|---|
| Clear homework and sufficient details | `New` |
| Missing due date but task exists | `Needs Review` |
| Conditional applicability | `Needs Review` |
| Ambiguous task | `Needs Review` |
| Informational resource retained for visibility | `For Information` |
| Processing failure | `Error` |

### 22.3 Manual Update Protection

n8n must not overwrite:

- `Status`
- `Completed At`
- `Completion Evidence Link`
- `Completion Notes`
- `Completed By`
- Manually edited `Due Date`
- Manually edited `Homework Item`

unless the workflow explicitly detects that the existing value is blank and safe to populate.

## 23. Completion Handling

Homework completion is human-confirmed.

For MVP, Google Sheets is the source of truth. A homework item is considered complete only when the `Status` field is changed to:

```text
Completed
```

AI must not infer completion automatically.

### 23.1 Recommended Completion Columns

Recommended tracker columns:

| Column | Purpose |
|---|---|
| `Status` | Manual status field |
| `Completed At` | Timestamp when item was marked completed |
| `Completion Evidence Link` | Optional link to photo/file/proof if added later |
| `Completion Notes` | Optional completion notes |
| `Completed By` | Optional value such as `student` or `guardian` |

### 23.2 MVP Completion Flow

```text
Student or guardian updates Google Sheet
        ↓
Status = Completed
        ↓
n8n detects completed status in scheduled workflow
        ↓
n8n populates Completed At if blank
        ↓
Item is excluded from digest, overdue update, and pending reminders
```

### 23.3 Completion Rules

When a row is marked `Completed`:

1. n8n should exclude it from daily digest pending sections.
2. n8n should exclude it from overdue status update.
3. n8n should preserve the completed status during future intake runs.
4. n8n may populate `Completed At` if the field is blank.
5. n8n must not revert a completed item to `New`, `Needs Review`, or `Overdue`.
6. n8n must not infer completion based on age, lack of response, or absence from later notices.

### 23.4 Future Completion Options

Future options may include:

1. Google Form completion.
2. Secure one-click completion link in daily email.
3. Completion evidence upload.

These are out of scope for MVP.

## 24. Calendar Event Creation

n8n creates Google Calendar events for actionable homework items.

### 24.1 Calendar Eligibility

Create calendar event when:

1. `due_date` is present.
2. Status is `New`.
3. Homework item is actionable.
4. Calendar event does not already exist.
5. Homework item is not duplicate.

Skip calendar event when:

1. `due_date` is null.
2. Status is `Needs Review`.
3. Status is `For Information`.
4. Status is `Not Applicable`.
5. Status is `Completed`.
6. Item already has `Calendar Event Link`.

### 24.2 Calendar Writeback

After creating an event, n8n should write back:

- Calendar event ID, if available.
- Calendar event link, if available.
- Calendar status.

Detailed event naming and reminder rules will be defined in `docs/06-google-calendar-design.md`.

## 25. Workflow 02 — Daily Homework Digest

### 25.1 Purpose

The daily digest workflow sends one consolidated family-facing homework email per day.

This workflow replaces per-item immediate email notifications for MVP.

### 25.2 Recommended Schedule

The daily digest should run after the final scheduled Cloud Run capture.

Recommended MVP schedule:

```text
Daily at 20:45 Asia/Kolkata
```

This follows the final 20:30 capture run and gives a single evening planning email.

### 25.3 Digest Query

Select rows where:

- Status is `New`, `In Progress`, `Needs Review`, or `Overdue`.
- Status is not `Completed`.
- Status is not `Not Applicable`.
- Status is not `For Information`, unless configured for visibility.
- Row is newly captured since previous digest, pending, due soon, missing due date, needs review, or overdue.

### 25.4 Digest Sections

The daily digest should group items by:

1. Needs Review.
2. Overdue.
3. Due today.
4. Due tomorrow.
5. Upcoming.
6. New homework captured since previous digest.
7. Evidence or attachment issues, if relevant.

### 25.5 Digest Content

Each item should include:

- Subject.
- Homework task.
- Due date, if available.
- Status.
- Teacher/sender, if available.
- Evidence link, if available.
- Calendar link, if available.
- Ambiguity notes, if any.

### 25.6 AI Usage in Digest

AI may be used to format and summarize the digest email.

The AI should:

1. Group items by urgency.
2. Preserve subject, task, due date, status, and evidence links.
3. Highlight missing due dates.
4. Highlight conditional or ambiguous items.
5. Avoid inventing dates, subjects, or completion status.
6. Keep the summary concise.
7. Avoid duplicating the same item in multiple sections unless explicitly useful.

The AI must not decide whether duplicate rows exist or whether the one-email-per-day rule applies. Those controls must be enforced by n8n logic.

### 25.7 One-Email-Per-Day Enforcement

The workflow must check `Daily Digest Log` before sending.

If a digest was already successfully sent for the same local date:

1. Do not send another family-facing digest.
2. Record a skipped duplicate digest attempt.
3. Exit successfully.

Recommended `Daily Digest Log` columns:

| Column | Description |
|---|---|
| `Digest Date` | Local date for digest |
| `Digest Sent At` | Timestamp |
| `Recipient Group` | Family/guardian recipient group |
| `Included Item Count` | Number of rows included |
| `Needs Review Count` | Count |
| `Overdue Count` | Count |
| `Due Today Count` | Count |
| `Due Tomorrow Count` | Count |
| `Send Status` | `sent`, `skipped`, or `failed` |
| `Processing Notes` | Safe notes |

## 26. Workflow 03 — Overdue Status Update

### 26.1 Purpose

The overdue status update workflow identifies pending homework past due date and updates status to `Overdue`.

It does not send a separate family-facing email in MVP. Overdue items are included in the daily digest.

### 26.2 Recommended Schedule

Recommended MVP schedule:

```text
Daily at 20:40 Asia/Kolkata
```

This allows the daily digest at 20:45 to include up-to-date overdue status.

### 26.3 Overdue Logic

An item is overdue when:

```text
due_date < today
AND status not in Completed, Not Applicable, For Information
```

n8n should update status to:

```text
Overdue
```

only when the row is not already completed or not applicable.

### 26.4 Overdue Rules

1. Do not mark `Completed` rows as overdue.
2. Do not mark `Not Applicable` rows as overdue.
3. Do not mark `For Information` rows as overdue.
4. Do not overwrite manually edited completion fields.
5. Include overdue items in the next daily digest.

## 27. Workflow 04 — Weekly Summary

### 27.1 Purpose

The weekly summary gives a household view of homework volume and completion.

### 27.2 Recommended Schedule

Recommended MVP schedule:

```text
Sunday 18:00 Asia/Kolkata
```

### 27.3 Weekly Summary Content

The weekly summary should include:

- New homework captured this week.
- Completed homework this week.
- Pending homework.
- Overdue homework.
- Needs Review items.
- Items with missing due dates.
- Evidence or attachment failures, if any.

Weekly summary can be added after workflows 01–03 are stable.

## 28. Error Handling

### 28.1 Error Log

n8n should write safe processing errors to the `Processing Errors` tab.

Recommended columns:

| Column | Description |
|---|---|
| `Error ID` | Generated ID |
| `Run ID` | Related run ID, if available |
| `Notice Hash` | Related notice hash, if available |
| `Homework Hash` | Related homework hash, if available |
| `Workflow Name` | n8n workflow name |
| `Component` | Validation, dedupe, evidence, LLM, Sheets, Calendar, Email |
| `Error Code` | Machine-readable code |
| `Error Message` | Safe message |
| `Severity` | `info`, `warning`, `error`, or `critical` |
| `Recoverable` | Boolean |
| `Created At` | Timestamp |
| `Processing Notes` | Safe notes |

### 28.2 Failure Handling

| Failure | Behavior |
|---|---|
| Invalid token | Stop, return 401 |
| Invalid payload | Stop, return 400 |
| Duplicate notice | Skip downstream item creation |
| Evidence copy failure | Continue if text is sufficient; mark warning |
| LLM failure | Create `Needs Review` row if notice may be actionable |
| Google Sheet failure | Stop row-dependent processing |
| Calendar failure | Keep tracker row; record calendar error |
| Daily digest email failure | Keep tracker rows; record notification error |
| Partial notice failure | Preserve available information |

## 29. Idempotency Rules

n8n workflows must be idempotent.

Rules:

1. Same `notice_hash` must not create duplicate notice processing.
2. Same `homework_hash` must not create duplicate tracker rows.
3. Same homework row must not create duplicate calendar events.
4. Intake workflow must not send per-item family-facing emails.
5. Daily digest workflow must send no more than one family-facing digest per local date.
6. Daily digest workflow must respect `Daily Digest Log`.
7. Overdue workflow must not overwrite manually completed rows.
8. Manual tracker updates must be preserved.
9. Completed items must not reappear in pending digest sections.

## 30. LLM Safety and Grounding

The LLM must be grounded only in captured notice content and evidence metadata.

Rules:

1. Do not invent due dates.
2. Do not invent subjects where unclear; use null or `Needs Review`.
3. Do not invent teacher names.
4. Do not infer student-specific applicability unless stated.
5. Preserve original homework wording.
6. Return JSON only for classification and itemization steps.
7. Mark uncertainty clearly in `ambiguity_notes`.
8. Do not infer homework completion.

## 31. Public Repository Safety

n8n workflow exports committed to the public repository must be placeholder-only.

Do not commit:

- Real n8n webhook URLs.
- Real n8n credentials.
- Real Google credentials.
- Real email addresses.
- Real student names.
- Real guardian names.
- Real SchoolDiary content.
- Real Drive folder IDs.
- Real Sheet IDs.
- Real Calendar IDs.
- Real LLM API keys.
- Real payloads.

Safe to commit:

- Placeholder workflow JSON files.
- Placeholder node names.
- Synthetic sample payloads.
- Fake email addresses using example domains.
- Documentation.
- Prompt templates with synthetic examples.

## 32. n8n Credentials

n8n credentials should be configured inside n8n Cloud.

Required credentials:

| Credential | Purpose |
|---|---|
| Webhook token/config value | Validate Cloud Run payload |
| Google Sheets credential | Read/write tracker |
| Google Drive credential | Write final evidence |
| Google Calendar credential | Create events |
| Email credential | Send daily digest and weekly summary |
| LLM provider credential | Classification, itemization, and digest formatting |
| GCS access credential or equivalent | Read temporary evidence objects |

Credential values must not be exported into the public repository.

## 33. Open Decisions

The following decisions remain open:

1. Whether webhook token remains in body for MVP or moves to header before implementation.
2. Exact Google Sheet workbook and tab names.
3. Exact Google Drive root folder name.
4. Whether pure announcements are ignored or retained in a separate notice log only.
5. Whether resource notices create `For Information` rows.
6. Whether `Needs Review` items with due dates should create review calendar events.
7. Exact family-facing email recipients.
8. Exact digest email format.
9. Exact LLM provider and model.
10. Whether failed payloads should be replayed manually or automatically.
11. Whether n8n should delete GCS temporary evidence after Drive copy or rely on GCS lifecycle policy.
12. Whether Google Form completion should be added after MVP.

## 34. Acceptance Criteria

The n8n workflow design is implemented correctly when:

1. Cloud Run payloads are accepted only with a valid token.
2. Invalid payloads do not create homework rows.
3. Duplicate notices do not create duplicate homework rows, evidence records, calendar events, or digest entries.
4. Notices are classified before itemization.
5. Homework itemization follows the data model contract.
6. Missing due dates are handled conservatively.
7. `Needs Review` rows are created for ambiguous or conditional items.
8. Evidence is finalized into Google Drive where possible.
9. Google Sheet tracker rows are created or updated safely.
10. Calendar events are created only for eligible actionable homework items.
11. Intake workflow does not send one email per item.
12. Daily digest workflow sends no more than one family-facing email per local date.
13. Manual completion through Google Sheets is respected.
14. Completed items are excluded from digest, overdue update, and pending reminders.
15. AI does not infer completion.
16. No real credentials, URLs, names, payloads, or evidence are committed to git.

## 35. Summary

n8n is the orchestration layer for the SchoolDiary Homework Automation system.

It receives captured SchoolDiary notices from the Cloud Run Job, validates and deduplicates them, classifies notices, itemizes homework, stores evidence, updates the Google Sheet tracker, creates calendar events, and sends one consolidated daily family-facing digest.

The design keeps capture and workflow responsibilities separate. The Cloud Run Job captures source notices. n8n turns those notices into a controlled, reviewable homework process.

MVP uses manual completion in Google Sheets and a one-email-per-day digest model to keep the workflow simple, auditable, and low-noise.
