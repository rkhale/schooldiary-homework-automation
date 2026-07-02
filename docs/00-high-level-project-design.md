# High-Level Project Design

## 1. Objective

Build an automated homework tracking system that captures homework notices from a configured SchoolDiary Notice Board, converts them into structured homework tasks, stores supporting evidence, creates calendar entries for due dates, and sends reminders to configured recipients.

The captured homework notices may include plain text, attachments, or publicly accessible links. The system should preserve these references as evidence so that the student and guardians can review the original homework instructions when needed.

## 2. Problem Statement

Homework information is often published through a web portal that requires manual checking, manual interpretation, and manual follow-up. This creates a risk of missed assignments, unclear due dates, incomplete evidence, and inconsistent communication to the student and guardians.

## 3. Target Users

- `student`
- `guardians`
- `configured recipients`
- repository maintainers and operators

## 4. Success Criteria

- New homework notices are captured on schedule from the configured SchoolDiary Notice Board.
- Homework items are converted into structured records in a configured Google Sheet.
- Due dates are added to a configured Google Calendar when available.
- Evidence is preserved through configured evidence links, stored files, or source references.
- Configured recipients receive timely email notifications and reminders.
- Duplicate notices do not create duplicate homework rows, evidence records, or calendar events.

## 5. Scope

- Scheduled capture of homework notices from the configured SchoolDiary Notice Board
- Extraction of notice text and metadata
- Preservation of attachments, screenshots, PDFs, and publicly accessible links as evidence
- Posting structured JSON from the fetcher to n8n
- Deduplication and itemization of notices into homework tasks
- Tracking tasks in a configured Google Sheet
- Creating due-date events in a configured Google Calendar
- Storing final file-based evidence in a configured Google Drive folder
- Sending email notifications and reminder messages to configured recipients

## 6. Out of Scope for MVP

- WhatsApp notifications
- Native mobile applications
- Multi-school or multi-portal abstraction
- Manual portal data entry UI
- Advanced analytics dashboards
- Full OCR or deep content extraction from attachments is out of scope for MVP unless required later for reliable homework itemization. MVP preserves attachments and links as evidence first.
- Always-on infrastructure

## 7. High-Level Architecture

The system uses Google Cloud Scheduler to trigger a Google Cloud Run Job on a defined capture schedule. The Cloud Run Job hosts a Playwright-based SchoolDiary fetcher that logs in using secrets, opens the configured SchoolDiary Notice Board URL, extracts notices, downloads attachments where available, preserves publicly accessible links where included, and captures screenshots or PDFs where applicable.

Downloaded files and generated captures are temporarily stored in Google Cloud Storage when needed as a handoff layer. The fetcher posts structured JSON to an n8n webhook. n8n Cloud then handles validation, deduplication, LLM-based itemization, Google Drive evidence storage, Google Sheet tracking, Google Calendar event creation, and notification delivery. Daily reminder and overdue workflows run inside n8n.

## 8. End-to-End Workflow

1. Google Cloud Scheduler triggers the capture job.
2. Google Cloud Run Job starts the Playwright-based fetcher.
3. The fetcher authenticates using stored secrets.
4. The fetcher opens the configured SchoolDiary Notice Board URL and extracts relevant notices.
5. Notice text, attachments, publicly accessible links, and screenshots or PDFs are captured or preserved.
6. Downloaded or captured files are temporarily written to Google Cloud Storage when applicable.
7. The fetcher posts structured JSON and evidence references to n8n.
8. n8n validates the payload and checks for duplicates.
9. n8n uses LLM-assisted logic to itemize homework tasks where needed.
10. n8n stores final file evidence in the configured Google Drive folder where applicable.
11. n8n writes or updates rows in the configured Google Sheet.
12. n8n creates or updates events in the configured Google Calendar when due dates exist.
13. n8n sends email notifications to configured recipients.
14. Reminder and overdue workflows continue inside n8n on their own schedules.

## 9. Component Responsibilities

- Google Cloud Scheduler: triggers capture runs on schedule
- Google Cloud Run Job: runs stateless capture execution on demand
- Playwright fetcher: logs in, navigates, extracts notices, downloads attachments, preserves links, and captures screenshots or PDFs
- Google Cloud Storage: temporary handoff storage for downloaded or captured evidence
- n8n Cloud: validation, deduplication, enrichment, itemization, orchestration, and downstream updates
- configured Google Drive folder: final storage for user-facing file evidence
- configured Google Sheet: master tracker for homework records and status
- configured Google Calendar: due-date visibility and reminder context
- email channel: MVP notification delivery to configured recipients

## 10. Scheduling Model

- Capture schedule is initiated by Google Cloud Scheduler.
- Capture runs are periodic rather than always-on.
- Reminder and overdue schedules run separately inside n8n.
- Scheduling should support future adjustment without changing the core architecture.

## 11. Data Model Overview

At a high level, the system tracks:

- source notice identity
- captured notice text
- extracted homework items
- subject or category when available
- assigned or published date when available
- due date when available
- evidence references
- processing timestamps
- deduplication status
- notification status
- calendar status

The configured Google Sheet acts as the operational master tracker for these records.

## 12. Evidence Management

Evidence may include:

1. Original attachments such as PDFs or images.
2. Publicly accessible links included in the notice.
3. Screenshot or PDF capture of the SchoolDiary notice.
4. Extracted notice text where useful.

Evidence flow:

- SchoolDiary notice text, attachments, public links, and screenshots are captured or preserved.
- Downloaded or captured files are temporarily handed off through Google Cloud Storage.
- Final user-facing file evidence is stored in Google Drive where applicable.
- Publicly accessible links may be stored directly in Google Sheets and notifications if no file capture is needed.

Configured evidence links should make it easy for the student and guardians to review the original source material later.

## 13. Notification Model

- MVP notifications are email-based.
- Notifications are sent to configured recipients.
- Initial notifications summarize newly captured homework items.
- Reminder notifications highlight upcoming due dates.
- Overdue notifications identify incomplete or overdue tasks based on tracker state.
- Notification content should include structured task details plus configured evidence links or preserved public links where relevant.

## 14. Error Handling and Exception Handling

- Authentication failures should stop the run and raise an actionable alert.
- Portal structure changes should be treated as capture failures and surfaced for review.
- Partial evidence failures should be logged with enough context to retry or manually inspect.
- Invalid or incomplete payloads should be rejected by n8n validation logic.
- Duplicate notices should be identified before creating duplicate tracker rows or calendar events.
- If a due date, subject, or homework instruction cannot be confidently extracted, n8n should create a tracker row with status `Needs Review` and include the original evidence link for manual guardian review.
- Downstream failures for Drive, Sheets, Calendar, or email should be logged and retried where practical.

## 15. Security and Privacy Principles

- Never commit credentials, secrets, portal-specific identifiers, or personal data to git.
- Store credentials and secrets in Secret Manager, n8n credentials, or environment variables.
- Keep the public repository generic and public-safe.
- Minimize stored personal data and only retain operationally necessary information.
- Preserve evidence safely without exposing confidential data unnecessarily.
- Separate temporary evidence handling from final user-facing storage.

## 16. Deployment Model

- Google Cloud Scheduler triggers the system.
- Google Cloud Run Job provides the scheduled execution environment.
- n8n Cloud provides orchestration and downstream automation.
- Google Drive, Google Sheets, and Google Calendar provide the user-facing workspace integrations.

This model avoids dependency on a local machine and supports managed, on-demand execution.

## 17. Cost Model

- Cloud Run Job costs are usage-based per run.
- Cloud Scheduler costs are low and predictable.
- Google Cloud Storage costs are temporary evidence storage related.
- n8n Cloud introduces workflow platform subscription cost.
- Google Workspace-related usage may add storage and API-related operational cost depending on scale.

The design favors low idle cost by using scheduled jobs instead of always-on compute.

## 18. Build Stages

1. Stage 0 — Repo Skeleton. Expected output: the repository contains the base folder structure and public-safe starter files.
2. Stage 1 — High-Level Project Design. Expected output: a concise public-safe design that defines objectives, scope, architecture, and delivery stages.
3. Stage 2 — Architecture Design. Expected output: a clearer system-level breakdown of components, responsibilities, and service interactions.
4. Stage 3 — Data Model and Payload Contracts. Expected output: documented schemas for notices, homework rows, evidence references, and inter-system payloads.
5. Stage 4 — Cloud Run Job Design. Expected output: a design for job lifecycle, runtime behavior, configuration, and execution boundaries.
6. Stage 5 — SchoolDiary Fetcher Design. Expected output: a design for login, notice extraction, evidence capture, and source handling without implementation code.
7. Stage 6 — n8n Workflow Design. Expected output: a workflow design for intake, validation, deduplication, itemization, tracking, and notifications.
8. Stage 7 — Security, Error Handling, and Observability Design. Expected output: documented controls for secrets, failure handling, logging, alerts, and recovery.
9. Stage 8 — Deployment Runbook. Expected output: an operator-facing deployment and update procedure for the planned system.
10. Stage 9 — Implementation: Cloud Run Job Scaffold. Expected output: the initial runnable project scaffold for the scheduled job environment.
11. Stage 10 — Implementation: SchoolDiary Fetcher. Expected output: the first working fetcher flow for authentication, navigation, and notice capture.
12. Stage 11 — Implementation: Evidence Handoff. Expected output: working handling for attachments, screenshots, PDFs, links, and temporary storage references.
13. Stage 12 — Implementation: n8n Intake. Expected output: working n8n intake and downstream tracker orchestration for structured homework processing.
14. Stage 13 — Implementation: Reminders and Overdue Follow-up. Expected output: reminder and overdue workflows that operate from tracker state and evidence references.

## 19. Risks and Mitigations

- Portal UI changes may break selectors.
  Mitigation: isolate portal logic, add observability, and keep selectors maintainable.
- Duplicate captures may create duplicate tasks.
  Mitigation: use deterministic deduplication logic in n8n and stable source identifiers where possible.
- Due dates may be ambiguous or missing.
  Mitigation: support incomplete dates and preserve source evidence for manual review.
- Attachment handling may fail intermittently.
  Mitigation: preserve notice text and source references even when file download fails.
- LLM itemization may misinterpret a notice.
  Mitigation: keep source evidence linked and allow conservative structured output design.

## 20. Open Decisions

- Exact deduplication strategy for repeated or edited notices
- Calendar event update behavior when source notices change
- Whether public links should ever be mirrored into file storage
- Whether attachment text extraction is needed beyond file preservation
- Preferred alerting channel for operator failures beyond email

## 21. Codex Implementation Guidance

- Codex must not implement all stages at once. Each later implementation step must reference this high-level design and the relevant stage-specific design document.
- When asked to work on a specific stage, Codex must modify only the files explicitly named in that stage prompt unless instructed otherwise.
- Keep the public repository generic and free of real credentials, real portal details, and real personal data.
- Prefer staged implementation with clear boundaries between fetch, evidence handling, orchestration, and downstream integrations.
- Preserve traceability from source notice to structured task to evidence reference.

## 22. Summary

This project is a scheduled, cloud-hosted homework automation system built around Google Cloud Scheduler, Google Cloud Run Job, and n8n Cloud. It captures notices from a configured SchoolDiary Notice Board, preserves evidence, structures homework into a configured Google Sheet, creates due-date visibility in a configured Google Calendar, and sends email reminders to configured recipients while keeping the repository public-safe.
