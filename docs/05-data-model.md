# Data Model and Payload Contracts

## 1. Purpose

This document defines the data model and payload contracts for the SchoolDiary Homework Automation system.

It establishes the structures used between:

- Google Cloud Run Job
- Playwright SchoolDiary Fetcher
- Google Cloud Storage temporary evidence handoff
- n8n webhook intake
- LLM-assisted homework itemization
- Google Sheets tracker
- Google Drive evidence storage
- Google Calendar due-date events
- Email notification workflows

This document is a contract document. Later implementation stages must use these structures unless explicitly changed through a design update.

## 2. Design Principles

The data model follows these principles:

1. Treat SchoolDiary Notice Board as a general notice feed, not a homework-only feed.
2. Capture the original notice before interpreting it.
3. Preserve evidence before creating homework tasks.
4. Separate raw source notices from structured homework items.
5. Classify notices before turning them into homework.
6. Do not create homework tasks from general announcements unless a clear action exists.
7. Treat missing or ambiguous information as `Needs Review`.
8. Use deterministic deduplication wherever possible.
9. Preserve publicly accessible links as references unless later design requires mirroring.
10. Store only operationally necessary data.
11. Keep the public repository free of real student, school, credential, portal, screenshot, or homework data.

## 3. Core Entities

The system uses the following core entities:

| Entity | Description | Created By | Used By |
|---|---|---|---|
| Capture Run | One execution of the Cloud Run Job | Cloud Run Job | n8n, logs, troubleshooting |
| Source Notice | One notice captured from SchoolDiary | Fetcher | n8n, tracker, evidence flow |
| Evidence Reference | Attachment, screenshot, PDF, public link, or extracted text reference | Fetcher / n8n | Google Drive, Google Sheet, notifications |
| Notice Classification | Classification of a notice as homework, announcement, resource notice, conditional action, or unknown | n8n / LLM parser | Tracker, notifications, filtering |
| Homework Item | One actionable homework task extracted from a notice | n8n / LLM parser | Google Sheet, Calendar, notifications |
| Tracker Row | Operational row in Google Sheets | n8n | Reminder and overdue workflows |
| Calendar Event Reference | Link between a homework item and a calendar event | n8n | Tracker, reminders |

## 4. Capture Run Object

A capture run represents one scheduled execution of the Cloud Run Job.

### 4.1 Required Fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `run_id` | string | Yes | Unique ID for the capture run |
| `source_system` | string | Yes | Expected value: `schooldiary` |
| `captured_at` | string | Yes | ISO-8601 timestamp for the run |
| `timezone` | string | Yes | Runtime timezone, default `Asia/Kolkata` |
| `status` | string | Yes | `success`, `partial_success`, or `failed` |
| `notices` | array | Yes | List of captured source notice objects |
| `errors` | array | Yes | List of safe error objects |

### 4.2 Example

```json
{
  "run_id": "run_2026-07-02T14-30-00_Asia-Kolkata",
  "source_system": "schooldiary",
  "captured_at": "2026-07-02T14:30:00+05:30",
  "timezone": "Asia/Kolkata",
  "status": "success",
  "notices": [],
  "errors": []
}
```

## 5. Source Notice Object

A source notice represents one captured SchoolDiary Notice Board entry.

The SchoolDiary Notice Board may contain homework, resource notices, general announcements, conditional instructions, or non-actionable information. The source notice object must therefore capture both the raw notice and classification fields.

### 5.1 Required Fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `notice_id` | string | Yes | Platform ID if available; otherwise generated ID |
| `notice_hash` | string | Yes | Deterministic hash used for deduplication |
| `source_system` | string | Yes | Expected value: `schooldiary` |
| `page_context` | object | Yes | Page-level context such as channel or class, if visible |
| `dom_order` | number/null | Yes | Order of notice on the page during capture |
| `posted_at_raw` | string/null | Yes | Raw timestamp text as shown in the portal |
| `posted_at` | string/null | Yes | Parsed ISO-8601 timestamp if available |
| `posted_date` | string/null | Yes | Parsed notice date in `YYYY-MM-DD` format |
| `captured_at` | string | Yes | ISO-8601 timestamp when captured |
| `sender_name` | string/null | Yes | Sender or teacher name if visible |
| `sender_context` | string/null | Yes | Full visible sender line |
| `class_context` | string/null | Yes | Class or section if visible |
| `channel_context` | string/null | Yes | Notice Board or channel context if visible |
| `title` | string/null | Yes | Formal title if visible; may be null |
| `generated_title` | string | Yes | System-generated short title |
| `body_text` | string | Yes | Extracted visible notice text |
| `notice_type` | string | Yes | Classification of notice |
| `actionability` | string | Yes | Whether action is required |
| `classification_confidence` | string | Yes | `High`, `Medium`, or `Low` |
| `classification_notes` | string | Yes | Reason for classification |
| `contains_homework` | boolean | Yes | Whether notice appears to contain homework |
| `requires_guardian_review` | boolean | Yes | Whether manual review is recommended |
| `attachments` | array | Yes | File attachments discovered or downloaded |
| `public_links` | array | Yes | Publicly accessible links found in the notice |
| `evidence_captures` | array | Yes | Screenshots or PDFs generated by the fetcher |
| `source_url` | string/null | Yes | Notice URL if available and safe to store |
| `extraction_status` | string | Yes | `success`, `partial_success`, or `failed` |
| `extraction_notes` | string | Yes | Safe processing notes |

### 5.2 Page Context Object

```json
{
  "school_label": null,
  "page_title": "Notice Board",
  "channel_id_present": true,
  "class_or_section": "Synthetic Class",
  "academic_year_label": null
}
```

The public repository must only use synthetic examples. Real school labels, class details, and portal identifiers must not be committed.

### 5.3 Notice Type Values

| Value | Meaning |
|---|---|
| `homework` | Notice contains an explicit homework task |
| `resource_notice` | Notice points to a study resource or uploaded material |
| `announcement` | General information with no direct homework action |
| `conditional_action` | Action depends on context, such as absence or teacher instruction |
| `event_notice` | Notice relates to an event, activity, or schedule |
| `unknown` | Classification is unclear |

### 5.4 Actionability Values

| Value | Meaning |
|---|---|
| `action_required` | Student or guardian must act |
| `for_information` | Informational only |
| `conditional` | Action depends on student-specific context |
| `needs_review` | System cannot confidently determine actionability |

### 5.5 Source Notice Example

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
  "captured_at": "2026-06-25T14:30:00+05:30",
  "sender_name": "Synthetic Teacher",
  "sender_context": "Synthetic Teacher (from Synthetic Class Notice Board)",
  "class_context": "Synthetic Class",
  "channel_context": "Notice Board",
  "title": null,
  "generated_title": "Mathematics homework",
  "body_text": "Good afternoon. Math homework: Complete Ex 3A.",
  "notice_type": "homework",
  "actionability": "action_required",
  "classification_confidence": "High",
  "classification_notes": "Notice contains explicit homework wording and a specific task.",
  "contains_homework": true,
  "requires_guardian_review": true,
  "attachments": [],
  "public_links": [],
  "evidence_captures": [],
  "source_url": null,
  "extraction_status": "success",
  "extraction_notes": ""
}
```

## 6. Attachment Object

An attachment object represents a file found in the notice.

### 6.1 Fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `attachment_id` | string | Yes | Generated unique ID |
| `file_name` | string | Yes | Sanitized file name |
| `mime_type` | string/null | Yes | MIME type if known |
| `source_attachment_url` | string/null | Yes | Source URL if safe to store |
| `download_status` | string | Yes | `downloaded`, `failed`, `skipped`, or `not_applicable` |
| `gcs_object_path` | string/null | Yes | Temporary GCS path if downloaded |
| `size_bytes` | number/null | Yes | File size if known |
| `hash_sha256` | string/null | Yes | File hash if downloaded |
| `error_message` | string | Yes | Safe error message if download failed |

### 6.2 Example

```json
{
  "attachment_id": "att_fake_001",
  "file_name": "homework_worksheet.pdf",
  "mime_type": "application/pdf",
  "source_attachment_url": null,
  "download_status": "downloaded",
  "gcs_object_path": "schooldiary-evidence/2026-07-02/notice_fake_001/homework_worksheet.pdf",
  "size_bytes": 102400,
  "hash_sha256": "sha256_fake_attachment_hash",
  "error_message": ""
}
```

## 7. Public Link Object

A public link object represents a URL included in a notice.

### 7.1 Fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `link_id` | string | Yes | Generated unique ID |
| `url` | string | Yes | Publicly accessible URL |
| `display_text` | string/null | Yes | Anchor text or surrounding text |
| `link_type` | string | Yes | `public_reference`, `assignment_link`, `resource_link`, `external_platform`, or `unknown` |
| `capture_required` | boolean | Yes | Whether later processing should mirror or capture it |
| `access_assumption` | string | Yes | `public`, `unknown`, or `requires_login` |
| `notes` | string | Yes | Safe processing notes |

### 7.2 Handling Rule

Public links should be preserved as references. They should not be downloaded or mirrored into file storage unless a later design or implementation stage explicitly requires that behavior.

### 7.3 Example

```json
{
  "link_id": "link_fake_001",
  "url": "https://example.com/sample-homework-resource",
  "display_text": "Reference worksheet",
  "link_type": "resource_link",
  "capture_required": false,
  "access_assumption": "public",
  "notes": "Preserved as source reference."
}
```

## 8. Evidence Capture Object

An evidence capture object represents a screenshot, PDF, or extracted text generated by the fetcher.

### 8.1 Fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `evidence_id` | string | Yes | Generated unique ID |
| `evidence_type` | string | Yes | `notice_screenshot`, `notice_pdf`, or `extracted_text` |
| `file_name` | string/null | Yes | File name if file-based evidence |
| `mime_type` | string/null | Yes | MIME type if file-based evidence |
| `gcs_object_path` | string/null | Yes | Temporary GCS path for file evidence |
| `text_content` | string/null | Yes | Extracted text if text evidence |
| `hash_sha256` | string/null | Yes | Hash if file-based |
| `capture_status` | string | Yes | `captured`, `failed`, or `skipped` |
| `error_message` | string | Yes | Safe error message if capture failed |

### 8.2 Example

```json
{
  "evidence_id": "evidence_fake_001",
  "evidence_type": "notice_screenshot",
  "file_name": "notice_fake_001.png",
  "mime_type": "image/png",
  "gcs_object_path": "schooldiary-evidence/2026-07-02/notice_fake_001/notice_fake_001.png",
  "text_content": null,
  "hash_sha256": "sha256_fake_evidence_hash",
  "capture_status": "captured",
  "error_message": ""
}
```

## 9. n8n Webhook Payload Contract

The Cloud Run Job posts a capture run payload to n8n.

### 9.1 Top-Level Payload

```json
{
  "webhook_token": "placeholder-only",
  "run": {
    "run_id": "run_2026-07-02T14-30-00_Asia-Kolkata",
    "source_system": "schooldiary",
    "captured_at": "2026-07-02T14:30:00+05:30",
    "timezone": "Asia/Kolkata",
    "status": "success",
    "notices": [],
    "errors": []
  }
}
```

### 9.2 Public Repo Rule

Public samples must not include real webhook tokens. In public sample payloads, use placeholder values only.

## 10. Notice Classification Output

Before creating homework tasks, n8n should classify each source notice.

### 10.1 Classification Output Fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `notice_hash` | string | Yes | Source notice hash |
| `notice_type` | string | Yes | Notice type classification |
| `actionability` | string | Yes | Actionability classification |
| `contains_homework` | boolean | Yes | Whether notice contains homework |
| `requires_guardian_review` | boolean | Yes | Whether manual review is recommended |
| `classification_confidence` | string | Yes | `High`, `Medium`, or `Low` |
| `classification_notes` | string | Yes | Explanation of classification |

### 10.2 Classification Rules

1. A notice should be classified as `homework` only when it contains an explicit student task.
2. A notice should be classified as `resource_notice` when it points to study material, classroom resources, links, or uploads without a clearly stated homework task.
3. A notice should be classified as `announcement` when it is informational only.
4. A notice should be classified as `conditional_action` when action depends on student-specific context.
5. A notice should be classified as `unknown` when the system cannot determine the type.
6. `Low` confidence or `unknown` classification should result in `Needs Review`.

## 11. Homework Item Object

A homework item represents one actionable task extracted from a source notice.

### 11.1 Fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `homework_id` | string | Yes | Generated task ID |
| `homework_hash` | string | Yes | Dedupe hash for the task |
| `notice_hash` | string | Yes | Source notice hash |
| `source_notice_id` | string | Yes | Source notice ID |
| `subject` | string/null | Yes | Extracted or inferred subject |
| `homework_task` | string | Yes | Actionable homework instruction |
| `due_date` | string/null | Yes | Due date in `YYYY-MM-DD`, if known |
| `due_time` | string/null | Yes | Optional due time in `HH:mm`, if known |
| `source_notice_date` | string/null | Yes | Source notice posted date |
| `teacher_name` | string/null | Yes | Sender or teacher name if available |
| `required_materials` | string/null | Yes | Materials required to complete task |
| `evidence_needed` | string/null | Yes | Expected completion evidence, if any |
| `is_conditional` | boolean | Yes | Whether action depends on context |
| `conditionality_notes` | string | Yes | Explanation of condition, if any |
| `confidence` | string | Yes | `High`, `Medium`, or `Low` |
| `status_recommendation` | string | Yes | Usually `New` or `Needs Review` |
| `ambiguity_notes` | string | Yes | Explanation if unclear |
| `source_evidence_refs` | array | Yes | Links or IDs of supporting evidence |

### 11.2 Example — Explicit Homework with Missing Due Date

```json
{
  "homework_id": "hw_fake_001",
  "homework_hash": "sha256_fake_homework_hash",
  "notice_hash": "sha256_fake_notice_hash",
  "source_notice_id": "notice_fake_001",
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
  "ambiguity_notes": "Due date is not stated in the notice.",
  "source_evidence_refs": [
    "evidence_fake_001"
  ]
}
```

### 11.3 Example — Conditional Action

```json
{
  "homework_id": "hw_fake_002",
  "homework_hash": "sha256_fake_conditional_homework_hash",
  "notice_hash": "sha256_fake_notice_hash_002",
  "source_notice_id": "notice_fake_002",
  "subject": "Science",
  "homework_task": "Collect the distributed journal if applicable.",
  "due_date": null,
  "due_time": null,
  "source_notice_date": "2026-06-25",
  "teacher_name": "Synthetic Teacher",
  "required_materials": "Journal",
  "evidence_needed": null,
  "is_conditional": true,
  "conditionality_notes": "Action applies only if the student did not receive the journal.",
  "confidence": "Medium",
  "status_recommendation": "Needs Review",
  "ambiguity_notes": "Applicability depends on student-specific context.",
  "source_evidence_refs": [
    "evidence_fake_002"
  ]
}
```

## 12. Google Sheet Tracker Model

Google Sheets is the master operational tracker.

### 12.1 Sheet Name

Recommended sheet name:

```text
Homework Tracker
```

### 12.2 Required Columns

| Column | Required | Description |
|---|---:|---|
| `Homework ID` | Yes | Unique generated task ID |
| `Homework Hash` | Yes | Dedupe hash for homework task |
| `Notice Hash` | Yes | Source notice dedupe key |
| `Source Notice ID` | Yes | Source notice ID |
| `Source System` | Yes | Expected value: `schooldiary` |
| `Notice Type` | Yes | `homework`, `resource_notice`, `announcement`, `conditional_action`, `event_notice`, or `unknown` |
| `Actionability` | Yes | `action_required`, `for_information`, `conditional`, or `needs_review` |
| `Notice Posted At` | No | Parsed notice timestamp |
| `Notice Posted Raw` | No | Raw timestamp shown in portal |
| `Captured At` | Yes | Timestamp when captured |
| `Sender Name` | No | Sender or teacher name |
| `Sender Context` | No | Full visible sender/context line |
| `Class Context` | No | Class or section context if visible |
| `Generated Title` | Yes | System-generated short title |
| `Subject` | Yes | Subject or category |
| `Homework Item` | Yes | Actionable task or review item |
| `Due Date` | No | Due date if available |
| `Due Time` | No | Optional due time |
| `Status` | Yes | Operational task status |
| `Owner` | Yes | Expected value: `student` for MVP |
| `Follow-up Owner` | Yes | Expected value: `guardians` or configured owner |
| `Confidence` | Yes | Extraction or classification confidence |
| `Is Conditional` | Yes | Whether action is conditional |
| `Conditionality Notes` | No | Explanation for conditional action |
| `Evidence Links` | Yes | Google Drive links, public links, or source references |
| `Public Links` | No | Preserved public links |
| `Calendar Event Link` | No | Link to Google Calendar event |
| `Original Notice Text` | Yes | Source notice text or safe excerpt |
| `Ambiguity Notes` | No | Explanation for unclear extraction |
| `Last Reminder Sent` | No | Timestamp |
| `Completed At` | No | Timestamp |
| `Completion Evidence Link` | No | Optional link if completion evidence is added |
| `Processing Notes` | No | Safe operational notes |

## 13. Status Values

The tracker supports the following statuses.

| Status | Meaning |
|---|---|
| `New` | Newly captured actionable homework item |
| `In Progress` | Student has started the task |
| `Completed` | Task is marked complete |
| `Needs Review` | System could not confidently extract required information or applicability |
| `Overdue` | Due date has passed and task is not completed |
| `Not Applicable` | Item was captured but later determined not to require action |
| `For Information` | Informational notice retained for visibility but not treated as homework |
| `Error` | Processing issue requires operator review |

## 14. Status Transition Rules

Recommended status transitions:

```text
New → In Progress → Completed
New → Completed
New → Needs Review
Needs Review → New
Needs Review → Not Applicable
Needs Review → For Information
New → Overdue
In Progress → Overdue
Overdue → Completed
Error → Needs Review
Error → Not Applicable
```

Rules:

1. `Completed` items should not be included in pending reminders.
2. `Not Applicable` items should not be included in pending reminders.
3. `For Information` items should not be included in homework reminders unless configured otherwise.
4. `Needs Review` items should be included in guardian-focused notifications.
5. `Overdue` should be derived from due date and status during reminder workflows.
6. Manual status updates in Google Sheets should be respected by n8n workflows.

## 15. Deduplication Model

The system uses multiple deduplication keys.

### 15.1 Notice Hash

Used to prevent the same notice from being processed repeatedly.

Recommended input:

```text
source_system
+ posted_at_raw
+ sender_context
+ normalized_body_text
+ normalized_attachment_names
+ normalized_public_links
```

If `posted_at_raw` is unavailable, fallback to:

```text
source_system
+ captured_date
+ sender_context
+ normalized_body_text
+ normalized_attachment_names
+ normalized_public_links
```

### 15.2 Homework Hash

Used to prevent duplicate homework tasks.

Recommended input:

```text
notice_hash
+ normalized_subject
+ normalized_homework_task
+ normalized_due_date
+ normalized_conditionality_notes
```

### 15.3 Evidence Hash

Used to identify duplicate downloaded or captured evidence files.

Recommended input:

```text
sha256(file_bytes)
```

### 15.4 Normalization Rules

Before hashing:

1. Trim leading and trailing whitespace.
2. Collapse repeated spaces.
3. Convert line endings consistently.
4. Lowercase fields used only for comparison.
5. Sort attachment file names.
6. Sort public links.
7. Use empty string for missing optional fields.
8. Do not include timestamps that change each run, except the original source timestamp shown in the portal.

## 16. Date and Time Handling

### 16.1 Timezone

Default timezone:

```text
Asia/Kolkata
```

### 16.2 Date Fields

| Field | Format | Notes |
|---|---|---|
| `captured_at` | ISO-8601 timestamp | Include timezone offset |
| `posted_at_raw` | string or null | Raw portal timestamp |
| `posted_at` | ISO-8601 timestamp or null | Parsed portal timestamp |
| `posted_date` | `YYYY-MM-DD` or null | Parsed portal date |
| `due_date` | `YYYY-MM-DD` or null | Derived or extracted due date |
| `due_time` | `HH:mm` or null | Optional |
| `last_reminder_sent` | ISO-8601 timestamp or blank | Tracker field |
| `completed_at` | ISO-8601 timestamp or blank | Tracker field |

### 16.3 Relative Dates

Relative dates such as `tomorrow`, `Friday`, or `next class` must be interpreted using:

1. Source posted date, if available.
2. Capture date, if posted date is unavailable.
3. Configured timezone.

If the due date cannot be confidently inferred, the homework item should be marked `Needs Review`.

### 16.4 Conservative Due-Date Rule

The system must not infer a due date unless the source notice explicitly states or strongly implies one.

Examples:

| Notice Wording | Due Date Handling |
|---|---|
| `Complete Ex 3A.` | `due_date = null`, `Needs Review` |
| `Complete Ex 3A by Friday.` | Parse Friday relative to posted date |
| `Submit tomorrow.` | Parse tomorrow relative to posted date |
| `Submit the homework by Friday, 26th June.` | Parse due date as `2026-06-26` using the posted year |
| `Bring it next class.` | `due_date = null`, `Needs Review` unless class schedule is later configured |

## 17. Validation Rules

### 17.1 Capture Payload Validation

n8n should reject or flag payloads when:

- `run_id` is missing.
- `captured_at` is missing.
- `source_system` is not `schooldiary`.
- `notices` is missing or not an array.
- Required notice fields are missing.
- Webhook token is missing or invalid.

### 17.2 Notice Validation

A notice is valid for processing when:

- `notice_hash` exists.
- `posted_at_raw`, `posted_at`, or `captured_at` exists.
- `body_text`, attachments, public links, or evidence captures exist.
- `attachments`, `public_links`, and `evidence_captures` are arrays, even if empty.

If the notice has no meaningful body text but has attachments or links, n8n may create a `Needs Review` row.

### 17.3 Homework Item Validation

A homework item is valid when:

- `homework_id` exists.
- `homework_hash` exists.
- `notice_hash` exists.
- `homework_task` is not empty.
- `confidence` is one of `High`, `Medium`, or `Low`.

If `due_date` is missing for an actionable homework item, the item may still be valid, but status should be `Needs Review`.

## 18. LLM Itemization Output Contract

The LLM itemization step should return only structured JSON.

### 18.1 Expected Output — Explicit Homework with Due Date

When the notice contains a clear due date, the LLM itemization step must populate `due_date`.

```json
[
  {
    "subject": "Chemistry",
    "homework_task": "Complete and submit the assigned selected questions from Chemistry Chapter 1.",
    "due_date": "2026-06-26",
    "due_time": null,
    "source_notice_date": "2026-06-24",
    "teacher_name": "Synthetic Teacher",
    "required_materials": "Chemistry Chapter 1 notes or assigned questions",
    "evidence_needed": "Completed written answers",
    "is_conditional": false,
    "conditionality_notes": "",
    "confidence": "Medium",
    "status_recommendation": "New",
    "ambiguity_notes": "The notice states that selected questions were assigned, but the specific question numbers are not included in the notice text."
  }
]
```

### 18.2 Expected Output — Explicit Homework with Missing Due Date

When the notice contains a clear homework task but no due date, the LLM itemization step should set `due_date` to `null` and recommend `Needs Review`.

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

### 18.3 LLM Rules

The itemizer must:

1. Create one item per distinct homework task.
2. Preserve page numbers, chapter names, worksheet names, file names, exercise numbers, and submission instructions.
3. Avoid creating tasks from general announcements unless there is a clear action.
4. Classify resource notices separately from homework tasks.
5. Mark conditional actions as conditional.
6. Use `null` for unknown due dates.
7. Add ambiguity notes when due date, subject, applicability, or task meaning is unclear.
8. Assign `Low` confidence when material details are missing.
9. Return JSON only.

## 19. Calendar Event Reference Model

Calendar event creation is covered in detail in `docs/06-google-calendar-design.md`.

The data model requires that each homework item with a due date may store:

| Field | Description |
|---|---|
| `calendar_event_id` | Calendar event ID if available |
| `calendar_event_link` | User-facing event link if available |
| `calendar_status` | `created`, `updated`, `skipped`, or `failed` |

If due date is missing, calendar event creation should be skipped.

## 20. Evidence Link Model

Evidence links may come from:

1. Google Drive final file links.
2. Public links preserved from the notice.
3. GCS temporary references used during processing.
4. Source notice references where safe and useful.

Tracker-facing evidence should prefer:

```text
Google Drive final file links > public links > source references > GCS temporary references
```

GCS temporary references should not be the long-term user-facing evidence link unless a processing failure requires temporary troubleshooting.

## 21. Error Object

Safe error objects may be included in run or notice payloads.

### 21.1 Fields

| Field | Type | Required | Description |
|---|---|---:|---|
| `error_code` | string | Yes | Machine-readable error code |
| `error_message` | string | Yes | Safe human-readable error |
| `component` | string | Yes | Component that generated the error |
| `severity` | string | Yes | `info`, `warning`, `error`, or `critical` |
| `recoverable` | boolean | Yes | Whether retry may help |

### 21.2 Example

```json
{
  "error_code": "ATTACHMENT_DOWNLOAD_FAILED",
  "error_message": "Attachment could not be downloaded.",
  "component": "schooldiary_fetcher",
  "severity": "warning",
  "recoverable": true
}
```

Errors must not include credentials, cookies, session tokens, real portal secrets, or sensitive personal information.

## 22. Public Repository Sample Data Rules

All sample data committed to the repository must be synthetic.

Allowed:

- Fake notice IDs.
- Fake homework text.
- Fake sender names.
- Fake class labels.
- Fake links using `example.com`.
- Fake hashes.
- Fake timestamps.
- Generic subjects.

Not allowed:

- Real SchoolDiary notice data.
- Real student names.
- Real guardian names.
- Real sender or teacher names.
- Real class or section details.
- Real school names.
- Real portal URLs.
- Real screenshots.
- Real attachments.
- Real credentials.
- Real webhook URLs.
- Real Google project IDs.

## 23. Open Decisions

The following decisions remain open for later documents:

1. Exact Google Sheet tab names.
2. Whether tracker row updates should use row IDs or homework hashes.
3. Whether public links should ever be mirrored into Google Drive.
4. Whether attachment text extraction is needed after MVP.
5. Whether completion updates happen directly in Google Sheets or through a form.
6. Whether Google Calendar event IDs can be reliably written back to the tracker.
7. Retention period for GCS temporary evidence.
8. Whether `Needs Review` alerts are sent immediately or batched.
9. Whether one notice with multiple homework items should create one evidence folder or one folder per task.
10. Whether informational notices should be retained in the tracker or ignored after classification.
11. Whether recurring non-homework notice types should be filtered out automatically.

## 24. Relationship to Later Documents

This document defines the data contracts used by later designs.

Later documents must reference this file when defining:

- Cloud Run Job output
- SchoolDiary fetcher payload generation
- n8n webhook validation
- Google Sheet tracker columns
- Google Calendar event creation
- Google Drive evidence handling
- Notification content
- Error handling and observability
- Test cases

## 25. Summary

The data model separates raw SchoolDiary notice capture from homework task creation. This is important because the SchoolDiary Notice Board may include homework, general announcements, resource notices, conditional actions, and non-actionable information.

The fetcher captures and preserves source notices and evidence. n8n validates, classifies, deduplicates, itemizes, tracks, calendarizes, and notifies. Google Sheets remains the operational system of record for MVP.
