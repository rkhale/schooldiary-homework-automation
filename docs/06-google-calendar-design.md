# Google Calendar Design

## 1. Purpose

This document defines how the SchoolDiary Homework Automation system uses Google Calendar.

The calendar is used for due-date visibility only. It is not the primary notification channel, not the source of completion status, and not the master homework tracker.

The MVP uses a dedicated Google Calendar named:

```text
SchoolDiary Homework
```

n8n creates calendar events for eligible homework items after the SchoolDiary notice has been captured, classified, itemized, deduplicated, and written to the Google Sheet tracker.

## 2. Scope

This document covers:

- Calendar purpose
- Dedicated calendar configuration
- Calendar ID handling
- Event creation rules
- Event title format
- Event date and time handling
- All-day event behavior
- Event description format
- Event idempotency
- Calendar writeback to Google Sheets
- Handling of missing due dates
- Handling of `Needs Review`
- Handling of completed homework
- Calendar reminders
- Calendar sharing assumptions
- Error handling
- Public repository safety
- Setup documentation TODOs

## 3. Out of Scope

This document does not define:

- Google Calendar creation steps
- Google OAuth setup
- n8n credential setup
- Google Sheet tracker schema
- SchoolDiary fetcher behavior
- LLM itemization prompt
- Daily digest email format
- Completion UI
- WhatsApp reminders
- Student-facing dashboard

Those are covered in other design documents or later implementation steps.

## 4. Relationship to Other Documents

| Document | Relevance |
|---|---|
| `docs/01-architecture.md` | Defines Google Calendar as part of downstream workflow |
| `docs/04-n8n-workflow-design.md` | Defines when n8n creates calendar events |
| `docs/05-data-model.md` | Defines homework item fields and calendar writeback fields |
| `docs/08-notification-design.md` | Defines daily digest email as the primary notification mechanism |
| `docs/10-error-handling-and-observability.md` | Will expand calendar failure handling |

## 5. Design Position

Google Calendar is used for due-date visibility.

It is not:

- The master homework tracker
- The completion source of truth
- The primary notification mechanism
- The evidence repository
- The deduplication database
- The workflow engine

The Google Sheet tracker remains the MVP system of record. n8n writes calendar metadata back to the tracker.

## 6. Dedicated Calendar

MVP uses a dedicated Google Calendar:

```text
SchoolDiary Homework
```

Using a dedicated calendar avoids mixing homework automation events into a personal or family calendar.

Benefits:

1. Homework events can be visually separated.
2. The calendar can be shared with family members.
3. n8n can target a single known calendar.
4. Calendar visibility can be controlled independently.
5. Future changes can be made without affecting other calendars.

## 7. Calendar ID Handling

n8n should use the Calendar ID, not only the display name.

Recommended runtime configuration key:

```text
SCHOOLDIARY_HOMEWORK_CALENDAR_ID=<your-calendar-id>
```

The actual Calendar ID is private runtime configuration.

It must not be committed to:

- `.env.example`
- `README.md`
- `docs/*.md`
- `samples/*.json`
- `n8n/workflows/*.json`
- Any public repository file

Public examples should use placeholder values only.

Example placeholder:

```text
SCHOOLDIARY_HOMEWORK_CALENDAR_ID=schooldiary-homework-calendar@example-calendar-id
```

## 8. Calendar Sharing Assumptions

The calendar may be shared with family members.

Recommended access model:

| User Type | Recommended Permission |
|---|---|
| Student | See all event details |
| Guardian / parent | See all event details or Make changes to events |
| n8n connected account | Make changes to events |
| Public access | Disabled |

Public sharing should remain off.

## 9. Event Creation Principle

n8n should create a calendar event only when a homework item is actionable and has a valid due date.

MVP rule:

```text
Create calendar event only when:
Status = New
AND due_date is present
AND homework item is not duplicate
AND calendar event does not already exist
```

Calendar events should not be created directly from raw SchoolDiary notices. Events are created only after notice classification and homework itemization.

## 10. Calendar Eligibility

### 10.1 Create Event

Create a calendar event when all of these are true:

| Condition | Required |
|---|---:|
| Homework item has `homework_hash` | Yes |
| Homework item has `due_date` | Yes |
| Status is `New` | Yes |
| Item is actionable | Yes |
| Item is not duplicate | Yes |
| No calendar event already exists | Yes |

### 10.2 Skip Event

Skip calendar creation when:

| Condition | Reason |
|---|---|
| `due_date` is null | Calendar needs a date |
| Status is `Needs Review` | Item requires human review first |
| Status is `For Information` | Not actionable homework |
| Status is `Not Applicable` | No action required |
| Status is `Completed` | Already done |
| Item is duplicate | Avoid duplicate events |
| Calendar event already exists | Preserve idempotency |

## 11. Needs Review Handling

MVP does not create calendar events for `Needs Review` rows.

Reason:

- The item may have an unclear task.
- The due date may be missing or uncertain.
- The applicability may be conditional.
- Creating calendar events for ambiguous rows can pollute the calendar.

`Needs Review` items are handled through the daily digest email and the Google Sheet tracker.

Future enhancement:

```text
Create optional review events for Needs Review items
```

This is out of scope for MVP.

## 12. Missing Due Date Handling

If a homework item does not have a due date, n8n must not create a calendar event.

The item should remain in the tracker and appear in the daily digest under a review or missing-due-date section.

Example:

| Homework Item | Due Date | Calendar Event |
|---|---:|---|
| Complete Ex 3A | null | No |
| Submit Chemistry homework by 2026-06-26 | 2026-06-26 | Yes |

## 13. Event Type

MVP uses all-day events by default.

Reason:

1. SchoolDiary homework due dates often do not specify a due time.
2. All-day events reduce false precision.
3. The daily digest email remains the main planning mechanism.
4. All-day events are easier to scan in calendar views.

Default event model:

```text
All-day event on due_date
```

If a due time is explicitly available in a later notice, timed events may be considered in a future enhancement.

## 14. Event Date Handling

For MVP:

| Field | Handling |
|---|---|
| `due_date` | Used as the all-day event date |
| `due_time` | Ignored for MVP unless explicitly enabled later |
| Timezone | `Asia/Kolkata` |
| Missing date | Skip event |
| Relative date | Must already be resolved before calendar creation |

n8n should not invent due dates during calendar creation.

Due-date extraction and resolution happens during classification/itemization, based on `docs/05-data-model.md`.

## 15. Event Title Format

Recommended MVP event title:

```text
Homework Due: <Subject> — <Short Task>
```

Examples:

```text
Homework Due: Mathematics — Complete Ex 3A
Homework Due: Chemistry — Submit Chapter 1 selected questions
Homework Due: English — Complete worksheet
```

### 15.1 Title Rules

1. Keep titles short.
2. Include subject when available.
3. Include a brief task summary.
4. Do not include full notice text.
5. Do not include private identifiers.
6. Do not include raw SchoolDiary URLs.
7. Do not include the student name unless explicitly required later.

### 15.2 Fallback Title

If subject is missing:

```text
Homework Due: SchoolDiary Task
```

If task is too long, truncate safely.

Recommended maximum title length:

```text
120 characters
```

## 16. Event Description Format

The event description should provide enough context without becoming the master record.

Recommended description structure:

```text
Homework Task:
<homework_task>

Due Date:
<due_date>

Status at Creation:
<status>

Source:
SchoolDiary Notice

Evidence:
<Google Drive evidence link, if available>

Tracker:
<Google Sheet tracker reference or link, if available>

Notes:
<ambiguity_notes, if any>
```

### 16.1 Description Rules

The description may include:

- Homework task
- Subject
- Due date
- Sender/teacher name, if available
- Evidence link
- Tracker link
- Ambiguity notes

The description must not include:

- Credentials
- Tokens
- Cookies
- Authenticated raw URLs
- Full private portal URLs
- Sensitive implementation configuration
- Real data in public examples

## 17. Calendar Event Metadata

n8n should write calendar information back to the Google Sheet tracker.

Recommended tracker fields:

| Field | Purpose |
|---|---|
| `Calendar Event ID` | Google Calendar event ID |
| `Calendar Event Link` | User-facing event link, if available |
| `Calendar Status` | `created`, `updated`, `skipped`, or `failed` |
| `Calendar Created At` | Timestamp when event was created |
| `Calendar Updated At` | Timestamp when event was updated |
| `Calendar Notes` | Safe processing notes |

`docs/05-data-model.md` already defines the core calendar reference fields. Additional columns may be added during implementation if useful.

## 18. Idempotency

n8n must create at most one calendar event per homework item.

Primary idempotency key:

```text
homework_hash
```

Calendar event creation should check:

1. Does the tracker row already have `Calendar Event ID`?
2. Does the tracker row already have `Calendar Event Link`?
3. Does another row with the same `homework_hash` already have a calendar event?
4. Is the item duplicate?

If any check indicates an event already exists, n8n must skip event creation.

## 19. Calendar Event Updates

MVP should minimize event updates.

### 19.1 Update Event

Update an existing calendar event only when:

| Condition | Action |
|---|---|
| Due date changed manually in tracker | Update event date |
| Homework task corrected manually | Optionally update title/description |
| Evidence link added after event creation | Optionally update description |
| Calendar event exists but tracker metadata is missing | Repair tracker metadata if possible |

### 19.2 Do Not Update Event

Do not update the calendar event for every minor workflow run.

Avoid repeated updates caused by:

- Duplicate SchoolDiary captures
- Reprocessing same notice
- Unchanged tracker rows
- Digest workflow runs

## 20. Completion Handling

Homework completion is controlled by the Google Sheet tracker, not Google Calendar.

MVP rule:

```text
When homework is marked Completed in Google Sheets, do not delete the calendar event.
```

Reason:

1. The tracker is the source of truth.
2. Deleting calendar events can remove historical visibility.
3. Updating/deleting events adds unnecessary workflow complexity.
4. The daily digest excludes completed items, so operational noise is already reduced.

### 20.1 Existing Events for Completed Homework

If a homework item is later marked `Completed`:

- Keep the calendar event unchanged for MVP.
- Exclude the item from pending digest sections.
- Preserve completion state in the tracker.

### 20.2 Future Enhancement

Future behavior may update the event title to:

```text
Completed: <original title>
```

or add a completion note to the event description.

This is out of scope for MVP.

## 21. Not Applicable Handling

If a homework item is marked `Not Applicable`, n8n should not create a new calendar event.

If a calendar event already exists, MVP should leave it unchanged unless a later cleanup workflow is explicitly designed.

## 22. Calendar Reminders

MVP should not rely on Google Calendar reminders as the primary reminder mechanism.

Primary reminder mechanism:

```text
Daily Homework Digest email at 20:45 Asia/Kolkata
```

Recommended MVP calendar reminder setting:

```text
No default event reminders
```

or

```text
Minimal reminder only if manually configured by family members
```

Reason:

1. Avoid notification flooding.
2. Shared-calendar notifications can be inconsistent across users.
3. Daily digest provides consolidated context.
4. Calendar is for visibility, not notification control.

### 22.1 Adding Calendar Reminders Later

Calendar reminders can be added after MVP.

Because n8n writes `Calendar Event ID` back to the tracker, future workflows can update existing calendar events and add reminder settings later if needed.

Later reminder options:

1. Add reminders only to newly created events after `CALENDAR_ENABLE_REMINDERS=true`.
2. Run a one-time backfill workflow to update existing future events.
3. Add reminders only for homework due tomorrow or overdue.
4. Allow each family member to rely on their own Google Calendar notification settings.

Calendar reminders should remain secondary to the daily digest unless the notification strategy is explicitly changed.

## 23. Event Creation Flow in n8n

Recommended flow:

```text
Homework row created or updated
    ↓
Check calendar eligibility
    ↓
Check duplicate/homework_hash
    ↓
Check existing Calendar Event ID/Link
    ↓
Build event title
    ↓
Build event description
    ↓
Create all-day event in SchoolDiary Homework calendar
    ↓
Write Calendar Event ID/Link/Status back to Homework Tracker
    ↓
Record error if creation fails
```

## 24. Error Handling

Calendar failures should not block homework tracker creation.

If calendar creation fails:

1. Keep the homework tracker row.
2. Set `Calendar Status = failed`.
3. Record safe error in `Processing Errors`.
4. Include the item in the daily digest if otherwise eligible.
5. Do not retry indefinitely in the same workflow run.

Recommended error code:

```text
CALENDAR_EVENT_CREATE_FAILED
```

If calendar update fails:

```text
CALENDAR_EVENT_UPDATE_FAILED
```

If calendar configuration is missing:

```text
CALENDAR_CONFIGURATION_MISSING
```

## 25. Failure Behavior

| Failure | Behavior |
|---|---|
| Calendar ID missing | Skip calendar creation, record error |
| Calendar access denied | Skip calendar creation, record error |
| Invalid due date | Skip calendar creation, mark item Needs Review if appropriate |
| Calendar API failure | Keep tracker row, record error |
| Duplicate event detected | Skip event creation |
| Event created but writeback fails | Record error; avoid duplicate creation on rerun where possible |

## 26. Public Repository Safety

Public repository files must not include the real Calendar ID.

Do not commit:

- Real Calendar ID
- Real calendar event IDs
- Real calendar event links
- Real family email addresses
- Real student names
- Real homework content
- Real evidence links
- Google credential data

Safe public placeholders:

```text
SCHOOLDIARY_HOMEWORK_CALENDAR_ID=<your-calendar-id>
```

```text
calendar_event_id_fake_001
```

```text
https://calendar.google.com/calendar/event?eid=fake_event_id
```

## 27. Configuration

Recommended runtime configuration:

| Key | Required | Secret | Description |
|---|---:|---:|---|
| `SCHOOLDIARY_HOMEWORK_CALENDAR_ID` | Yes | Yes/Private Config | Dedicated homework calendar ID |
| `CALENDAR_CREATE_EVENTS` | Yes | No | Whether n8n should create events |
| `CALENDAR_DEFAULT_EVENT_TYPE` | Yes | No | `all_day` for MVP |
| `CALENDAR_TIMEZONE` | Yes | No | `Asia/Kolkata` |
| `CALENDAR_CREATE_FOR_NEEDS_REVIEW` | No | No | Default `false` |
| `CALENDAR_ENABLE_REMINDERS` | No | No | Default `false` |

The real Calendar ID should be stored in n8n credentials, environment config, or private runtime configuration.

## 28. Setup Documentation TODO

A separate setup file must be created for Google OAuth and n8n credential setup.

Recommended file:

```text
docs/13-google-oauth-and-n8n-credential-setup.md
```

This setup file should cover:

1. Google Cloud project OAuth consent configuration.
2. Google OAuth client creation.
3. Required Google scopes for Sheets, Drive, Calendar, and email.
4. n8n Google credential setup.
5. n8n credential testing.
6. Calendar ID private configuration.
7. Google Sheet ID private configuration.
8. Google Drive root folder ID private configuration.
9. Credential rotation guidance.
10. Public repository safety rules.

The setup file must not include real credentials, real Calendar IDs, real Sheet IDs, real Drive folder IDs, OAuth secrets, refresh tokens, or private email addresses.

## 29. Open Decisions

The following decisions remain open:

1. Whether completed homework should eventually update event titles.
2. Whether `Needs Review` items should ever create review placeholder events.
3. Whether due-time-specific timed events are needed later.
4. Whether calendar event descriptions should include direct tracker row links.
5. Whether event color should be controlled by n8n or calendar default.
6. Whether calendar reminders should be enabled later.
7. Whether existing calendar events should be cleaned up for `Not Applicable` items.
8. Whether events should include attachments directly or only Drive links.

## 30. Acceptance Criteria

The Google Calendar design is implemented correctly when:

1. n8n writes events only to the dedicated `SchoolDiary Homework` calendar.
2. The real Calendar ID is not committed to the public repository.
3. Events are created only for actionable `New` homework with a valid due date.
4. No events are created for missing due dates.
5. No events are created for `Needs Review` rows in MVP.
6. No events are created for `Completed`, `Not Applicable`, or `For Information` rows.
7. Events are all-day events by default.
8. Event titles are short and clear.
9. Event descriptions include useful evidence/tracker context without secrets.
10. Calendar Event ID/Link/Status are written back to the tracker.
11. Duplicate homework items do not create duplicate calendar events.
12. Calendar creation failure does not block tracker row creation.
13. Calendar reminders are not the primary notification mechanism.
14. Calendar reminders can be added later by updating existing or future events.
15. Completed homework remains controlled by the Google Sheet tracker.
16. Google OAuth and n8n credential setup is documented separately.

## 31. Summary

The `SchoolDiary Homework` calendar provides due-date visibility for actionable homework items.

n8n creates all-day events only for eligible `New` homework items with valid due dates. The Google Sheet tracker remains the source of truth for homework status and completion. The daily digest email remains the primary family-facing notification mechanism.

The real Calendar ID is private runtime configuration and must not be committed to the public repository. Calendar reminders are intentionally deferred for MVP and can be added later because event IDs are retained in the tracker.
