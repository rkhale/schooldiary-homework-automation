# Notification Design

## 1. Purpose

This document defines how the SchoolDiary Homework Automation system sends family-facing notifications and operational alerts.

The notification design is intentionally low-noise. The MVP does not send an email for every captured notice or every homework item. Instead, it sends one consolidated daily digest after the final scheduled SchoolDiary capture of the day.

## 2. Scope

This document covers:

- Daily family-facing digest
- Digest timing
- Digest sections
- Notification eligibility rules
- Recipient configuration
- Configuration source model
- Evidence link handling
- Missing due date handling
- Overdue item handling
- AI usage boundaries
- Deduplication rules
- Operational alerts
- Weekly summary direction
- Public repository safety

## 3. Out of Scope

This document does not define:

- SchoolDiary notice extraction
- Google Calendar event creation
- Google Drive evidence storage
- Google Sheet full schema
- n8n credential setup
- WhatsApp integration
- SMS integration
- Push notification integration
- Student completion evidence collection

WhatsApp, SMS, and push notifications are out of MVP.

## 4. Relationship to Other Documents

| Document | Relevance |
|---|---|
| `docs/01-architecture.md` | Defines email as MVP notification channel |
| `docs/04-n8n-workflow-design.md` | Defines n8n workflows and digest cadence |
| `docs/05-data-model.md` | Defines homework status and tracker fields |
| `docs/06-google-calendar-design.md` | Defines calendar event creation rules |
| `docs/07-google-drive-evidence-design.md` | Defines evidence links used in digest |
| `docs/09-security-and-secrets.md` | Will define safe handling of private configuration |
| `docs/10-error-handling-and-observability.md` | Will define operational alerting in more detail |
| `docs/13-google-oauth-and-n8n-credential-setup.md` | Will define Google/n8n credential setup |

## 5. Design Position

Notifications are not the system of record.

The Google Sheet tracker remains the system of record for homework status. Notifications are a family-facing summary layer over the tracker.

The daily digest should help answer:

1. What needs attention now?
2. What is due today or tomorrow?
3. What is overdue?
4. What needs parent review because the automation could not confidently extract details?
5. What new homework was captured since the last digest?

## 6. MVP Notification Channel

MVP notification channel:

```text
Email
```

Out of MVP:

```text
WhatsApp
SMS
Push notifications
Per-item instant alerts
```

Rationale:

- Email is easier to implement through n8n.
- Email supports structured sections and evidence links.
- Email avoids unnecessary family notification noise.
- WhatsApp requires additional integration and policy handling.

## 7. Daily Digest Cadence

Daily family-facing digest should run once per day at:

```text
20:45 Asia/Kolkata
```

Reason:

- The final SchoolDiary capture is scheduled for `20:30`.
- A 15-minute gap gives n8n time to process newly captured notices.
- The family receives one consolidated end-of-day homework summary.

Related scheduled captures:

```text
07:00 Asia/Kolkata
15:30 Asia/Kolkata
20:30 Asia/Kolkata
```

Digest workflow:

```text
20:30 final capture
    ↓
n8n intake processing
    ↓
20:40 overdue status update
    ↓
20:45 daily family digest
```

## 8. Daily Digest Workflow

Recommended n8n workflow name:

```text
02 - Daily Homework Digest
```

Trigger:

```text
Schedule Trigger
Daily at 20:45 Asia/Kolkata
```

High-level flow:

```text
Read private configuration
    ↓
Check Daily Digest Log
    ↓
Read Homework Tracker
    ↓
Filter eligible rows
    ↓
Group rows into digest sections
    ↓
Generate deterministic digest structure
    ↓
Optionally use AI to improve wording
    ↓
Send one family-facing email
    ↓
Write Daily Digest Log
```

## 9. One Email Per Day Rule

The daily digest must send at most one family-facing email per local date.

n8n should check the `Daily Digest Log` tab before sending.

Recommended log fields:

| Field | Purpose |
|---|---|
| `Digest Date` | Local date in `Asia/Kolkata` |
| `Sent At` | Timestamp of email send |
| `Recipient Group` | Logical recipient group |
| `Item Count` | Number of tracker rows included |
| `Status` | `sent`, `skipped`, or `failed` |
| `Message ID` | Email provider message ID if available |
| `Notes` | Safe processing notes |

If a digest was already sent for the same local date and recipient group, n8n must skip sending another family-facing digest unless manually overridden.

## 10. Configuration Source Model

Configuration is split into public templates and private runtime values.

The public repository may contain a placeholder configuration template:

```text
config/config.example.json
```

The real configuration must not be committed.

Private runtime values may be stored in one of the following locations depending on execution context:

| Runtime Area | Recommended Config Source |
|---|---|
| n8n workflows | Private Google Sheet `Config` tab or n8n private variables |
| Local development | `config/config.local.json` |
| Public repo documentation | `config/config.example.json` only |

For MVP, the recommended source for n8n-controlled notification settings is:

```text
Private Google Sheet Config tab
```

Reason:

- n8n Cloud can read the Config tab directly.
- Recipients can be changed without redeploying Cloud Run.
- Digest behavior can be adjusted without editing workflow code deeply.
- Real email addresses remain outside the public repository.

## 11. Recipient Configuration

Recipient configuration must be private runtime configuration.

Recipient email addresses must not be hardcoded in:

- Source code
- Public documentation
- n8n workflow exports committed to Git
- `config/config.example.json`
- `.env.example`

Recommended logical configuration shape:

```json
{
  "notifications": {
    "digest_email_to": ["<family-recipient@example.com>"],
    "digest_email_cc": [],
    "digest_email_bcc": [],
    "operational_alert_email_to": ["<admin@example.com>"],
    "digest_send_time": "20:45",
    "digest_timezone": "Asia/Kolkata",
    "digest_lookahead_days": 7,
    "digest_send_empty": false,
    "digest_use_ai_formatting": true,
    "digest_max_items_per_section": 10
  }
}
```

In a Google Sheet `Config` tab, the same values can be represented as key-value rows.

Example with placeholders only:

| Key | Value |
|---|---|
| `notifications.digest_email_to` | `<family-recipient-list>` |
| `notifications.digest_email_cc` |  |
| `notifications.digest_email_bcc` |  |
| `notifications.operational_alert_email_to` | `<admin-email>` |
| `notifications.digest_send_time` | `20:45` |
| `notifications.digest_timezone` | `Asia/Kolkata` |
| `notifications.digest_lookahead_days` | `7` |
| `notifications.digest_send_empty` | `false` |
| `notifications.digest_use_ai_formatting` | `true` |
| `notifications.digest_max_items_per_section` | `10` |

## 12. Family-Facing Digest Sections

Recommended digest sections:

1. Needs Review
2. Overdue
3. Due Today
4. Due Tomorrow
5. Upcoming
6. Newly Captured Since Last Digest
7. Evidence or Attachment Issues

Digest sections with no items may be omitted.

## 13. Section Logic

### 13.1 Needs Review

Include rows where:

```text
Status = Needs Review
```

Typical reasons:

- Missing due date
- Ambiguous subject
- Ambiguous instruction
- Conditional action
- Evidence required to interpret task
- Attachment/link issue

This section should appear near the top because it requires human judgment.

### 13.2 Overdue

Include rows where:

```text
Status = Overdue
AND Status not in Completed, Not Applicable, For Information
```

The overdue status should be updated by the scheduled overdue workflow before digest generation.

### 13.3 Due Today

Include rows where:

```text
Due Date = today
AND Status not in Completed, Not Applicable, For Information
```

### 13.4 Due Tomorrow

Include rows where:

```text
Due Date = tomorrow
AND Status not in Completed, Not Applicable, For Information
```

### 13.5 Upcoming

Include rows where:

```text
Due Date > tomorrow
AND Status in New, In Progress
```

Recommended MVP lookahead:

```text
7 days
```

### 13.6 Newly Captured Since Last Digest

Include rows where:

```text
Captured At > previous_digest_sent_at
AND Status not in Completed, Not Applicable
```

This section helps the family see what changed since the previous digest.

### 13.7 Evidence or Attachment Issues

Include rows where:

```text
Evidence Status in partial, failed
```

or where attachment download/finalization failed.

This section should be concise and should point to the tracker for details.

## 14. Excluded Items

The daily digest should exclude rows where:

```text
Status = Completed
Status = Not Applicable
Status = For Information
```

These items may appear in a future weekly summary if useful, but they should not clutter the daily action digest.

## 15. Missing Due Date Handling

If a homework item does not have a confident due date, the item should not be placed in Due Today, Due Tomorrow, or Upcoming.

It should appear under:

```text
Needs Review
```

Example digest wording:

```text
Mathematics — Complete Ex 3A.
Due date: Not found in notice
Action: Please review the original notice/evidence.
```

The automation should not invent due dates.

## 16. Digest Item Format

Recommended item format:

```text
Subject — Homework task
Due: <due date or "Needs review">
Status: <status>
Evidence: <Drive folder/file link if available>
Notes: <ambiguity notes if relevant>
```

Example:

```text
Chemistry — Complete selected questions from Chapter 1
Due: 2026-06-26
Status: New
Evidence: Drive folder link
Notes: Specific question numbers were not included in the notice text.
```

## 17. Email Subject Line

Recommended subject format:

```text
SchoolDiary Homework Digest — yyyy-mm-dd
```

Example:

```text
SchoolDiary Homework Digest — 2026-07-03
```

If there are overdue or review items, subject can include a signal:

```text
SchoolDiary Homework Digest — 2026-07-03 — Review Needed
```

Avoid alarmist wording unless there is a genuine urgent issue.

## 18. Email Body Structure

Recommended digest body:

```text
SchoolDiary Homework Digest
Date: yyyy-mm-dd

Summary
- Needs Review: <count>
- Overdue: <count>
- Due Today: <count>
- Due Tomorrow: <count>
- Upcoming: <count>
- New Since Last Digest: <count>

Needs Review
<items>

Overdue
<items>

Due Today
<items>

Due Tomorrow
<items>

Upcoming
<items>

New Since Last Digest
<items>

Evidence / Attachment Issues
<items>

Tracker
<Google Sheet link>

Notes
This digest is generated from SchoolDiary notices. Please verify ambiguous items in the original evidence.
```

## 19. Digest Tone

The digest should be:

- Clear
- Action-oriented
- Calm
- Parent/student friendly
- Short enough to read quickly

Avoid:

- Alarmist wording
- Overly technical processing details
- Long raw notice dumps
- Internal error traces
- Model confidence explanations unless needed

## 20. Evidence Link Handling

The digest should use final Google Drive links, not GCS paths.

Preferred evidence order:

1. Notice-level Drive folder link
2. Specific attachment link, if directly relevant
3. Public link from notice, if applicable
4. Tracker link for full context

Do not expose temporary GCS paths in family-facing emails.

If evidence is unavailable:

```text
Evidence: Not available
```

and include the row under Evidence or Attachment Issues if needed.

## 21. Public Link Handling

Public links from SchoolDiary notices may be included in the digest.

They should be labelled clearly:

```text
External link from notice
```

If access is unknown:

```text
External link from notice — access not verified
```

Do not claim a link is publicly accessible unless the workflow has actually verified it.

## 22. AI Usage Boundary

AI may be used to format or summarize the digest.

AI must not decide:

- Whether to send the digest
- Who receives the digest
- Whether an item is overdue
- Whether a task is completed
- Whether to suppress a row
- Whether a due date should be invented
- Whether evidence is valid

n8n deterministic logic controls send eligibility, filtering, grouping, and deduplication.

Recommended AI role:

```text
Format the already-selected digest items into a concise, family-readable email.
Do not add tasks.
Do not remove tasks.
Do not change dates.
Do not infer completion.
Do not invent due dates.
```

## 23. Completion Handling

Completion is human-confirmed through the Google Sheet tracker.

If a row is marked:

```text
Completed
```

then it should be excluded from the daily digest.

AI must not infer that homework is completed based on age, lack of reminders, or wording.

## 24. Overdue Handling

A separate scheduled workflow updates overdue statuses before the digest.

Recommended workflow:

```text
03 - Overdue Status Update
```

Schedule:

```text
20:40 Asia/Kolkata
```

Logic:

```text
Due Date < today
AND Status not in Completed, Not Applicable, For Information
```

Set:

```text
Status = Overdue
```

The daily digest then reads the updated tracker state.

## 25. Operational Alerts

Operational alerts are separate from the family-facing daily digest.

Operational alerts should go to the parent/admin only.

Examples:

- SchoolDiary login failure
- CAPTCHA or manual verification required
- n8n webhook authentication failure
- Google Drive permission failure
- Google Sheet writeback failure
- Google Calendar creation failure
- GCS read/write failure
- Digest send failure

Operational alerts should not be sent to the full family recipient list unless explicitly configured.

## 26. Operational Alert Format

Recommended subject:

```text
SchoolDiary Automation Alert — <error category>
```

Recommended body:

```text
SchoolDiary Automation Alert

Time: <timestamp>
Workflow: <workflow name>
Severity: <info/warning/error>
Error Category: <category>
Safe Details: <summary>
Action Needed: <recommended action>

No secrets, cookies, tokens, or private URLs are included in this alert.
```

## 27. Alert Severity

Recommended severity values:

| Severity | Meaning |
|---|---|
| `info` | Non-blocking informational event |
| `warning` | Partial failure or recoverable issue |
| `error` | Workflow failed or action required |
| `critical` | Repeated failure or blocked automation |

MVP can use only:

```text
warning
error
```

if simpler.

## 28. Digest Failure Handling

If the digest fails to send:

1. Record failure in `Daily Digest Log`.
2. Send operational alert to admin.
3. Do not retry repeatedly without control.
4. Avoid duplicate family-facing emails.

If retry is implemented, it should check the digest log before sending.

## 29. Empty Digest Handling

If there are no actionable items, two options exist.

Recommended MVP behavior:

```text
Do not send an empty family-facing digest.
Write skipped status to Daily Digest Log.
```

Reason:

- Avoids unnecessary email.
- Keeps the system low-noise.

Optional future behavior:

```text
Send "No pending homework items found today."
```

This can be added later if the family wants explicit confirmation.

## 30. Weekly Summary

Weekly summary is optional but useful after MVP stabilizes.

Recommended workflow name:

```text
04 - Weekly Summary
```

Potential contents:

- Completed this week
- Pending items
- Overdue items
- Needs Review items
- Subjects with repeated homework
- Evidence issues
- Manual follow-up list

Weekly summary should not replace the daily digest.

## 31. Deduplication Rules

The notification layer must respect tracker-level deduplication.

Rules:

1. Same homework row should not appear twice in the same digest.
2. Same digest date and recipient group should not be sent twice.
3. Duplicate notices should not produce duplicate digest items.
4. Multiple homework items from one notice may appear separately if they are distinct tracker rows.
5. Multiple tracker rows may share the same evidence folder link.

## 32. Timezone

All digest scheduling and date comparisons use:

```text
Asia/Kolkata
```

This applies to:

- Today
- Tomorrow
- Overdue calculation
- Digest date
- Previous digest window
- Daily Digest Log

## 33. Notification Configuration

Recommended logical configuration:

| Key | Required | Private | Description |
|---|---:|---:|---|
| `notifications.digest_email_to` | Yes | Yes | Family digest recipients |
| `notifications.digest_email_cc` | No | Yes | Optional CC recipients |
| `notifications.digest_email_bcc` | No | Yes | Optional BCC recipients |
| `notifications.operational_alert_email_to` | Yes | Yes | Admin alert recipient |
| `notifications.digest_send_time` | No | No | Default `20:45` |
| `notifications.digest_timezone` | Yes | No | Default `Asia/Kolkata` |
| `notifications.digest_lookahead_days` | No | No | Default `7` |
| `notifications.digest_send_empty` | No | No | Default `false` |
| `notifications.digest_use_ai_formatting` | No | No | Default `true` |
| `notifications.digest_max_items_per_section` | No | No | Optional section cap; default `10` |

Real recipient values must be stored only in private runtime configuration.

## 34. Recommended `config/config.example.json` Addition

Public committed template:

```json
{
  "notifications": {
    "digest_email_to": ["<family-recipient@example.com>"],
    "digest_email_cc": [],
    "digest_email_bcc": [],
    "operational_alert_email_to": ["<admin@example.com>"],
    "digest_send_time": "20:45",
    "digest_timezone": "Asia/Kolkata",
    "digest_lookahead_days": 7,
    "digest_send_empty": false,
    "digest_use_ai_formatting": true,
    "digest_max_items_per_section": 10
  }
}
```

The committed template must contain placeholders only.

## 35. Private Local Config Option

For local development only, the same structure may exist in:

```text
config/config.local.json
```

This file must be gitignored.

Recommended `.gitignore` entries:

```gitignore
config/config.local.json
config/*.private.json
config/*.secrets.json
```

n8n does not need to read `config/config.local.json` in MVP. For n8n, the private Google Sheet `Config` tab or n8n private variables are preferred.

## 36. Public Repository Safety

Do not commit:

- Real recipient email addresses
- Real Google Sheet links
- Real Google Drive links
- Real SchoolDiary notice content
- Real student names
- Real guardian names
- Real teacher names
- Real school names
- Real message IDs
- Real workflow execution logs with private content
- SMTP credentials
- Gmail OAuth credentials
- n8n credential exports
- `config/config.local.json`
- Private config files

Safe placeholders:

```text
<family-recipient@example.com>
<admin@example.com>
https://docs.google.com/spreadsheets/d/fake_sheet_id
https://drive.google.com/drive/folders/fake_folder_id
```

## 37. Open Decisions

The following decisions remain open:

1. Exact family digest recipient list.
2. Whether digest should be sent when there are zero actionable items.
3. Whether the student receives the same digest as parents or a reduced version.
4. Whether weekly summary should be enabled in MVP or deferred.
5. Whether digest should include completed items from the same day.
6. Whether operational alerts should go only to parent/admin or also to a backup recipient.
7. Whether digest should include direct Google Calendar links.
8. Whether digest should limit each section to a maximum number of items.

## 38. Acceptance Criteria

The notification design is implemented correctly when:

1. The system sends at most one family-facing digest per day.
2. The digest runs after the final scheduled SchoolDiary capture.
3. The digest uses `Asia/Kolkata` for date logic.
4. The digest excludes `Completed`, `Not Applicable`, and `For Information` rows.
5. Missing due date items appear under `Needs Review`.
6. Overdue items appear under `Overdue`.
7. Due today and due tomorrow items are clearly separated.
8. Evidence links point to Google Drive or preserved public links, not GCS.
9. AI formatting does not change eligibility, dates, status, or task content.
10. Empty digests are skipped by default.
11. Operational alerts are separate from family-facing digest emails.
12. Digest send attempts are logged.
13. Duplicate family-facing emails are prevented.
14. Real recipients and private links are not committed to the public repo.
15. Notification configuration uses private runtime values, not hardcoded workflow/code values.

## 39. Summary

The MVP notification model uses one consolidated daily family-facing email at `20:45 Asia/Kolkata`.

The digest focuses on actionable homework, overdue items, due-soon items, newly captured homework, and items needing review. It avoids per-item email noise.

n8n controls deterministic filtering, grouping, deduplication, and send rules. AI may help format the final email but must not invent, remove, suppress, or modify homework facts.

Operational alerts are handled separately and sent only to the configured admin recipient.

Recipient configuration is private runtime configuration. The public repository may include only placeholders in `config/config.example.json`.
