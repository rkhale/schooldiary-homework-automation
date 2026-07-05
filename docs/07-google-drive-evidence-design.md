# Google Drive Evidence Design

## 1. Purpose

This document defines how the SchoolDiary Homework Automation system stores and manages evidence in Google Drive.

Evidence includes captured notice screenshots, downloaded SchoolDiary attachments, extracted notice text, and preserved public links. Google Drive is the final family-facing evidence store for MVP.

Google Cloud Storage is used only as a temporary handoff layer between the Cloud Run Job and n8n.

## 2. Scope

This document covers:

- Google Drive evidence purpose
- Dedicated evidence root folder
- Folder structure
- Folder creation responsibility
- Evidence file types
- File naming rules
- GCS-to-Google-Drive finalization flow
- Google Sheet evidence link writeback
- Public link handling
- Access and sharing model
- Duplicate evidence handling
- Failure handling
- Retention model
- Public repository safety

## 3. Out of Scope

This document does not define:

- SchoolDiary browser extraction selectors
- Cloud Run Job implementation
- Google Cloud Storage bucket setup
- Google OAuth setup
- n8n credential setup
- Google Sheet tracker schema in full
- Daily digest email format
- Completion evidence upload
- OCR or deep attachment content extraction

Those are covered in other design documents or future implementation stages.

## 4. Relationship to Other Documents

| Document | Relevance |
|---|---|
| `docs/01-architecture.md` | Defines Google Drive as final evidence storage |
| `docs/02-cloud-run-job-design.md` | Defines GCS temporary handoff |
| `docs/03-schooldiary-fetcher-design.md` | Defines notice screenshots, attachments, and links captured by the fetcher |
| `docs/04-n8n-workflow-design.md` | Defines evidence finalization inside n8n |
| `docs/05-data-model.md` | Defines attachment, public link, evidence, and tracker fields |
| `docs/08-notification-design.md` | Will define how evidence links appear in daily digest emails |
| `docs/13-google-oauth-and-n8n-credential-setup.md` | TODO for OAuth and n8n credential setup |

## 5. Design Position

Google Drive is the final evidence repository.

It is not:

- The master homework tracker
- The notification engine
- The deduplication database
- The temporary file handoff layer
- The OCR processing engine
- The source of homework completion status

The Google Sheet tracker remains the system of record. Google Drive stores supporting evidence and provides family-facing links.

## 6. Dedicated Evidence Root Folder

MVP uses a dedicated Google Drive folder:

```text
SchoolDiary Homework Evidence
```

The real Google Drive folder ID is private runtime configuration.

Recommended runtime key:

```text
SCHOOLDIARY_HOMEWORK_DRIVE_ROOT_FOLDER_ID=<your-drive-folder-id>
```

The real folder ID must not be committed to Git.

## 7. Folder Creation Responsibility

The user creates only the dedicated root folder once:

```text
SchoolDiary Homework Evidence
```

n8n is responsible for creating all child folders under the root folder.

The user should not manually create date folders, notice folders, or attachment folders.

Examples of folders that n8n should create automatically:

```text
2026-07-03/
notice_ab12cd34/
```

This keeps the evidence structure consistent and prevents manual folder naming drift.

## 8. Evidence Storage Principle

The evidence flow is:

```text
SchoolDiary source notice
    ↓
Cloud Run Job captures text, screenshots, attachments, links
    ↓
Cloud Run Job uploads files to temporary GCS handoff
    ↓
n8n copies file evidence from GCS to Google Drive
    ↓
n8n writes final Drive links to Google Sheets
    ↓
Daily digest uses Google Sheet evidence links
```

Google Drive links should be the final user-facing evidence references.

GCS paths should remain internal processing references and should not be used as long-term family-facing evidence links.

## 9. Evidence Types

MVP evidence types:

| Evidence Type | Source | Final Storage |
|---|---|---|
| Notice screenshot | Cloud Run / Playwright | Google Drive |
| Downloaded attachment | Cloud Run / Playwright | Google Drive |
| Extracted notice text | Cloud Run / n8n | Google Drive or Google Sheet |
| Public link | SchoolDiary notice | Preserved as URL reference |
| GCS object path | Cloud Run upload | Temporary only |

## 10. Folder Structure

Recommended MVP folder structure:

```text
SchoolDiary Homework Evidence/
└── yyyy-mm-dd/
    └── notice_<notice_hash_short>/
        ├── notice_screenshot.png
        ├── notice_text.txt
        ├── attachment_001_<safe_file_name>
        └── attachment_002_<safe_file_name>
```

Example with synthetic values:

```text
SchoolDiary Homework Evidence/
└── 2026-06-25/
    └── notice_ab12cd34/
        ├── notice_screenshot.png
        ├── notice_text.txt
        └── attachment_001_worksheet.pdf
```

## 11. Folder Date Basis

Use the SchoolDiary notice posted date where available.

Priority order:

1. `posted_date`
2. Parsed date from `posted_at`
3. Capture date from `captured_at`

Reason:

- Posted date is more meaningful for homework review.
- Capture run date may differ from the school notice date.
- Family users usually think in terms of when the notice was posted, not when automation captured it.

## 12. Notice Folder Naming

Recommended notice folder format:

```text
notice_<notice_hash_short>
```

Where:

```text
notice_hash_short = first 8 to 12 characters of notice_hash
```

Rules:

1. Do not use real teacher names in folder names.
2. Do not use student names in folder names.
3. Do not use full notice text in folder names.
4. Avoid characters that can cause path or sync issues.
5. Use stable identifiers to support idempotency.

## 13. File Naming Rules

File names should be safe, short, and deterministic.

### 13.1 Notice Screenshot

Recommended:

```text
notice_screenshot.png
```

### 13.2 Extracted Notice Text

Recommended:

```text
notice_text.txt
```

### 13.3 Attachments

Recommended:

```text
attachment_<sequence>_<sanitized_original_file_name>
```

Example:

```text
attachment_001_worksheet.pdf
```

### 13.4 Sanitization Rules

File names should:

1. Trim whitespace.
2. Replace unsupported characters.
3. Collapse repeated spaces.
4. Avoid very long names.
5. Preserve useful extensions when available.
6. Avoid names containing real student names where possible.
7. Use generated fallback names when source file names are missing.

Fallback:

```text
attachment_001_file
```

## 14. GCS Temporary Handoff

Cloud Run uploads file evidence to GCS before posting the payload to n8n.

n8n receives GCS object paths in attachment and evidence objects.

n8n should:

1. Read the GCS object.
2. Upload or copy it to Google Drive.
3. Store final Drive link.
4. Write final Drive link to tracker.
5. Preserve safe processing notes.
6. Leave GCS cleanup to lifecycle policy unless a later workflow deletes temporary objects.

## 15. Google Drive Finalization Flow

Recommended n8n flow:

```text
For each source notice:
    Determine evidence folder path
    Create date folder if missing
    Create notice folder if missing
    For each evidence file:
        Check if equivalent file already exists
        Upload/copy file to Drive if missing
        Capture Drive file ID/link
    Write evidence links back to tracker
    Record errors if any
```

n8n should create missing child folders automatically. Manual child-folder creation is not required.

## 16. Evidence Link Writeback

n8n should write final evidence references to the Google Sheet tracker.

Recommended tracker fields:

| Field | Purpose |
|---|---|
| `Evidence Links` | Primary family-facing evidence links |
| `Public Links` | Preserved external links from notice |
| `Evidence Status` | `available`, `partial`, `failed`, or `not_required` |
| `Evidence Notes` | Safe notes |
| `Drive Folder Link` | Link to notice-level Drive folder |
| `Attachment Links` | Links to copied attachments, if separated |

The exact column set may be refined during implementation, but the tracker must include enough evidence context for the daily digest.

## 17. Public Link Handling

Public links found in SchoolDiary notices should be preserved as references.

MVP rule:

```text
Preserve public links. Do not mirror them into Drive by default.
```

Reason:

1. Public links may point to live external resources.
2. Some links may require login or context.
3. Mirroring external content creates unnecessary complexity.
4. The evidence model already preserves source reference metadata.

n8n should mark public links with access assumptions:

| Access Assumption | Meaning |
|---|---|
| `public` | Appears publicly accessible |
| `unknown` | Access has not been verified |
| `requires_login` | Link likely requires login |

## 18. Evidence Status Values

Recommended evidence status values:

| Status | Meaning |
|---|---|
| `available` | All expected evidence finalized to Drive |
| `partial` | Some evidence finalized; some failed or unavailable |
| `failed` | Evidence finalization failed |
| `not_required` | No file evidence was required or available |
| `link_only` | Evidence consists of preserved public links only |

## 19. Access Model

Recommended Drive access model:

| User Type | Recommended Access |
|---|---|
| Owner / parent | Owner |
| Guardian | Editor or Viewer depending on responsibility |
| Student | Viewer or Commenter |
| n8n connected Google account | Editor |
| Public access | Restricted |

For MVP, the student usually needs view access, not edit access.

General access should remain:

```text
Restricted
```

Only explicitly shared users should access the folder.

## 20. n8n Google Account Access

n8n accesses Google Drive through the Google credential configured in n8n.

MVP recommendation:

```text
Use the owner's Google OAuth credential in n8n.
```

If n8n uses the owner's Google account and the owner owns the Drive folder, no additional folder sharing is required for n8n.

If n8n later uses a separate automation Google account, share the root evidence folder with that account as:

```text
Editor
```

Service account usage is out of scope for MVP unless explicitly selected later.

## 21. Duplicate Evidence Handling

n8n should avoid uploading duplicate evidence files.

Primary duplicate checks:

1. `notice_hash`
2. `hash_sha256` for file evidence
3. Existing Drive folder for `notice_hash_short`
4. Existing file name inside notice folder

If the same notice is reprocessed, n8n should reuse existing Drive links where possible.

Duplicate notice behavior:

```text
Duplicate notices do not create duplicate evidence records.
```

## 22. Evidence and Homework Item Relationship

One source notice may produce one or more homework items.

MVP folder model:

```text
One evidence folder per source notice.
```

Reason:

1. Evidence usually belongs to the notice, not to a single item.
2. One notice may contain multiple homework tasks.
3. Duplicating the same screenshot/attachment per homework item is wasteful.
4. Tracker rows can reference the same evidence folder/link.

If one notice creates multiple homework rows, each row may reference the same Drive folder.

## 23. Extracted Text Evidence

The notice body text is already stored in the payload and tracker.

Optional MVP behavior:

```text
Create notice_text.txt in the Drive notice folder.
```

This can help family users and debugging, but it is not strictly required if the tracker preserves the original notice text.

Recommended MVP decision:

```text
Store notice_text.txt when screenshot capture is enabled or when attachments/links exist.
```

This gives a plain-text fallback when screenshots are hard to read.

## 24. Attachment Handling

Downloaded attachments should be copied from GCS into the notice folder.

For each attachment:

1. Preserve sanitized file name.
2. Preserve MIME type where possible.
3. Preserve hash where available.
4. Write Drive file link back to tracker or evidence metadata.
5. Record failures safely.

Attachment download happens in the Cloud Run Job. n8n only finalizes already-downloaded temporary files.

## 25. Screenshot Handling

Notice screenshots should be copied from GCS into the notice folder.

Recommended screenshot file name:

```text
notice_screenshot.png
```

If notice-block screenshot is not available, page-level or viewport screenshots may be stored with clear names:

```text
page_screenshot.png
viewport_screenshot.png
```

If screenshot capture fails but notice text is available, evidence status should be:

```text
partial
```

or:

```text
not_required
```

depending on configuration.

## 26. Completion Evidence

Completion evidence upload is out of scope for MVP.

Future options:

1. Student/guardian uploads photo of completed homework.
2. Google Form collects completion evidence.
3. n8n stores completion evidence in a separate subfolder.
4. Tracker stores `Completion Evidence Link`.

Future folder pattern:

```text
notice_<notice_hash_short>/
└── completion/
    └── completion_evidence_001.jpg
```

## 27. Retention Model

### 27.1 Google Drive

Google Drive is the final evidence store.

Drive evidence should be retained unless manually deleted or a later retention policy is defined.

### 27.2 Google Cloud Storage

GCS is temporary handoff only.

Recommended future lifecycle policy:

```text
Delete temporary GCS evidence after 7 to 30 days.
```

Exact GCS retention should be defined in deployment/runbook documentation.

### 27.3 Google Sheets

Google Sheet tracker rows should retain evidence links as long as the tracker is active.

## 28. Failure Handling

Evidence finalization failure should not automatically block homework tracking.

| Failure | Behavior |
|---|---|
| GCS object missing | Record error; continue if notice text exists |
| Drive folder create failure | Record error; continue tracker processing if possible |
| Drive upload failure | Mark evidence status `partial` or `failed` |
| Public link malformed | Preserve safe note; do not fail notice |
| Attachment missing | Mark attachment failed; continue |
| Screenshot missing | Continue if text evidence exists |
| Drive permission failure | Record error; flag for review |

If evidence is required to understand the homework and cannot be finalized, the related tracker row should be marked:

```text
Needs Review
```

or should include evidence failure notes.

## 29. Error Codes

Recommended error codes:

| Error Code | Meaning |
|---|---|
| `DRIVE_ROOT_FOLDER_MISSING` | Root folder ID/config missing |
| `DRIVE_FOLDER_CREATE_FAILED` | n8n could not create date or notice folder |
| `DRIVE_FILE_UPLOAD_FAILED` | n8n could not upload/copy evidence file |
| `DRIVE_PERMISSION_DENIED` | n8n credential lacks folder access |
| `GCS_OBJECT_READ_FAILED` | Temporary GCS evidence could not be read |
| `EVIDENCE_LINK_WRITEBACK_FAILED` | Tracker update failed |
| `PUBLIC_LINK_INVALID` | Public link could not be parsed safely |

## 30. Public Repository Safety

Do not commit:

- Real Drive folder IDs
- Real Drive file IDs
- Real Drive links
- Real evidence screenshots
- Real SchoolDiary attachments
- Real notice text
- Real student names
- Real guardian names
- Real teacher names
- Real school names
- Real private email addresses
- Google OAuth credentials
- n8n credential exports

Safe placeholders:

```text
SCHOOLDIARY_HOMEWORK_DRIVE_ROOT_FOLDER_ID=<your-drive-folder-id>
```

```text
https://drive.google.com/file/d/fake_file_id/view
```

```text
https://drive.google.com/drive/folders/fake_folder_id
```

## 31. Configuration

Recommended runtime configuration:

| Key | Required | Secret | Description |
|---|---:|---:|---|
| `SCHOOLDIARY_HOMEWORK_DRIVE_ROOT_FOLDER_ID` | Yes | Yes/Private Config | Dedicated Drive root folder ID |
| `DRIVE_FINALIZE_EVIDENCE` | Yes | No | Whether n8n copies evidence to Drive |
| `DRIVE_CREATE_NOTICE_TEXT_FILE` | No | No | Whether to create `notice_text.txt` |
| `DRIVE_FOLDER_DATE_BASIS` | No | No | Default `posted_date` |
| `DRIVE_LINK_SHARING_MODE` | No | No | Default `restricted` |
| `GCS_DELETE_AFTER_DRIVE_COPY` | No | No | Default `false` for MVP |

The real Drive folder ID must be stored in n8n credential/config, not in Git.

## 32. Setup Documentation TODO

The Google Drive evidence setup should be referenced in:

```text
docs/13-google-oauth-and-n8n-credential-setup.md
```

That setup file should include:

1. How to create the Google Drive evidence root folder.
2. How to capture the root folder ID privately.
3. How to share the folder with family members.
4. How to grant n8n access through Google OAuth.
5. How to test n8n folder access.
6. How to avoid committing real folder IDs or links.

## 33. Open Decisions

The following decisions remain open:

1. Exact GCS lifecycle retention period.
2. Whether `notice_text.txt` is created for every notice or only when needed.
3. Whether student should remain Viewer or Commenter instead of Editor.
4. Whether completion evidence upload is added after MVP.
5. Whether external public links should ever be mirrored into Drive.
6. Whether date folders should use notice posted date or due date in later versions.
7. Whether Drive folder links or individual file links are preferred in the daily digest.
8. Whether evidence should be compressed or archived after a school term.

## 34. Acceptance Criteria

The Google Drive evidence design is implemented correctly when:

1. Final evidence is stored under `SchoolDiary Homework Evidence`.
2. The real Drive folder ID is not committed to Git.
3. The user only creates the root evidence folder manually.
4. n8n automatically creates date folders and notice folders under the root folder.
5. GCS remains a temporary handoff layer only.
6. n8n copies screenshots and downloaded attachments from GCS to Drive.
7. n8n writes final Drive evidence links to Google Sheets.
8. Public links are preserved but not mirrored by default.
9. One source notice has one evidence folder.
10. Multiple homework rows from one notice can reuse the same evidence folder.
11. Duplicate notices do not create duplicate evidence records.
12. Evidence failure does not block tracker row creation when notice text is sufficient.
13. Evidence failures are recorded safely.
14. General Drive access remains restricted.
15. No real evidence files, links, IDs, or private data are committed to the public repository.

## 35. Summary

Google Drive is the final evidence repository for the SchoolDiary Homework Automation system.

The user creates only the dedicated root folder: `SchoolDiary Homework Evidence`. n8n creates the date-level and notice-level child folders automatically.

The Cloud Run Job captures notice screenshots and downloads attachments, then places them in temporary GCS handoff. n8n finalizes those files into the Drive evidence folder and writes Drive links back to the Google Sheet tracker.

MVP uses one evidence folder per source notice, organized by notice posted date and notice hash. Public links are preserved as references and are not mirrored by default. Completion evidence is deferred until after MVP.
