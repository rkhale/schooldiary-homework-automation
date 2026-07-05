# Security and Secrets Design

## 1. Purpose

This document defines the security and secrets handling model for the SchoolDiary Homework Automation project.

The project is a public repository. Therefore, all credentials, private identifiers, real family information, SchoolDiary data, evidence files, and runtime-only configuration must remain outside Git.

The goal is to make the repository safe to share while still allowing local development, Cloud Run execution, and n8n workflow operation.

## 2. Scope

This document covers:

- Public repository safety rules
- Secret versus private configuration classification
- Approved configuration locations
- SchoolDiary credential handling
- n8n webhook and MCP token handling
- Google OAuth credential handling
- Google Drive, Calendar, and Sheet identifier handling
- Evidence privacy
- Log redaction
- Git hygiene checks
- Incident response if a secret is exposed

## 3. Out of Scope

This document does not define:

- Exact Google Cloud IAM setup
- Detailed n8n workflow implementation
- Google OAuth consent screen setup
- Google Workspace administration
- SchoolDiary portal implementation details
- Production-grade key rotation automation
- Enterprise DLP tooling

Those may be added in deployment and credential setup documentation.

## 4. Relationship to Other Documents

| Document | Relevance |
|---|---|
| `docs/01-architecture.md` | Defines the overall system architecture |
| `docs/02-cloud-run-job-design.md` | Defines Cloud Run runtime and temporary GCS handoff |
| `docs/03-schooldiary-fetcher-design.md` | Defines portal login and evidence capture behavior |
| `docs/04-n8n-workflow-design.md` | Defines n8n workflow responsibilities |
| `docs/05-data-model.md` | Defines payload, tracker, and evidence fields |
| `docs/06-google-calendar-design.md` | Defines private Calendar ID handling |
| `docs/07-google-drive-evidence-design.md` | Defines Drive evidence privacy |
| `docs/08-notification-design.md` | Defines private notification recipient configuration |
| `docs/10-error-handling-and-observability.md` | Will define logging and operational monitoring in more detail |
| `docs/11-deployment-runbook.md` | Will define deployment commands and runtime setup |
| `docs/13-google-oauth-and-n8n-credential-setup.md` | Will define credential setup steps |

## 5. Security Position

The project must follow this principle:

```text
The public repository contains design, templates, code, and synthetic examples only.
Real credentials, real private configuration, and real evidence never enter Git.
```

The public repository may describe configuration keys, but it must not contain actual runtime values.

## 6. Classification Model

Configuration and data are classified into four categories:

| Category | Meaning | Can Commit? |
|---|---|---:|
| Public | Generic documentation, source code, synthetic samples | Yes |
| Placeholder | Fake values used in examples/templates | Yes |
| Private Config | Runtime identifiers or personal values not directly reusable as credentials | No |
| Secret | Credentials, tokens, passwords, OAuth secrets, session material | No |

## 7. Secret Values

The following are secrets and must never be committed:

- SchoolDiary username
- SchoolDiary password
- n8n webhook token
- n8n MCP token
- Google OAuth client secret
- Google refresh tokens
- Google service account key JSON files
- Gmail SMTP credentials, if ever used
- Browser cookies
- Browser session state
- Playwright storage state
- Any token embedded in a URL
- Any exported n8n credential value
- Any local `.env` file containing real values
- Any private `config.local.json` file containing real values

## 8. Private Configuration Values

The following are private configuration values.

They may not be full credentials, but they still must not be committed:

- Real SchoolDiary Notice Board URL
- Google Cloud project ID
- GCS evidence bucket name
- Google Sheet ID
- Google Calendar ID
- Google Drive root folder ID
- n8n webhook URL
- Real Google Drive links
- Real Google Sheet links
- Real Calendar links
- Digest recipient email addresses
- Operational alert email addresses
- Real student, guardian, teacher, class, or school names
- Real workflow execution IDs if they expose private context

Private configuration should be treated as runtime-only.

## 9. Public Values

The following may be committed:

- Source code
- Design documents
- Synthetic test data
- Synthetic payload samples
- Placeholder IDs
- Placeholder URLs
- Placeholder email addresses
- `.env.example`
- `config/config.example.json`
- `.codex/config.example.toml`
- README instructions
- Generic deployment command templates

Examples of safe placeholder values:

```text
<your-drive-folder-id>
<your-calendar-id>
<family-recipient@example.com>
<admin@example.com>
https://docs.google.com/spreadsheets/d/fake_sheet_id
https://drive.google.com/drive/folders/fake_folder_id
```

## 10. Approved Configuration Locations

The project uses different configuration sources for different runtime areas.

| Runtime Area | Approved Location |
|---|---|
| Public repo examples | `config/config.example.json`, `.env.example`, docs placeholders |
| Local development | `config/config.local.json`, local `.env`, local ignored files |
| Cloud Run secrets | Google Secret Manager |
| Cloud Run non-secret flags | Environment variables |
| n8n workflow runtime config | Private Google Sheet `Config` tab or n8n private variables |
| n8n credentials | n8n credential store |
| Codex local MCP token | `.codex/config.toml`, gitignored |
| Codex MCP template | `.codex/config.example.toml`, placeholders only |

## 11. Public Configuration Template

The public repository may include:

```text
config/config.example.json
```

This file must contain placeholders only.

Recommended structure:

```json
{
  "runtime": {
    "timezone": "Asia/Kolkata",
    "max_notices_per_run": 20,
    "download_attachments": true,
    "capture_screenshot": true
  },
  "google_drive": {
    "evidence_root_folder_id": "<your-drive-folder-id>",
    "finalize_evidence": true,
    "create_notice_text_file": true,
    "folder_date_basis": "posted_date",
    "link_sharing_mode": "restricted"
  },
  "google_calendar": {
    "homework_calendar_id": "<your-calendar-id>",
    "create_events": true,
    "default_event_type": "all_day",
    "timezone": "Asia/Kolkata",
    "create_for_needs_review": false,
    "enable_reminders": false
  },
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

Even in the example file, values must remain synthetic placeholders.

## 12. Private Local Configuration

Local development may use:

```text
config/config.local.json
```

This file must be ignored by Git.

Recommended `.gitignore` entries:

```gitignore
config/config.local.json
config/*.private.json
config/*.secrets.json
.env
.env.*
!.env.example
```

The public repository should not depend on the real local file.

## 13. Cloud Run Secret Handling

Cloud Run should not receive secrets from committed files.

Recommended handling:

| Value | Storage |
|---|---|
| `SCHOOLDIARY_USERNAME` | Google Secret Manager |
| `SCHOOLDIARY_PASSWORD` | Google Secret Manager |
| `SCHOOLDIARY_NOTICEBOARD_URL` | Google Secret Manager or private runtime config |
| `N8N_WEBHOOK_URL` | Google Secret Manager or private runtime config |
| `N8N_WEBHOOK_TOKEN` | Google Secret Manager |

Cloud Run should read secrets at runtime.

Non-secret flags may be environment variables:

| Value | Example |
|---|---|
| `TIMEZONE` | `Asia/Kolkata` |
| `MAX_NOTICES_PER_RUN` | `20` |
| `DOWNLOAD_ATTACHMENTS` | `true` |
| `CAPTURE_SCREENSHOT` | `true` |
| `POST_TO_N8N` | `true` |

## 14. n8n Secret Handling

n8n secrets must be stored in n8n credentials or n8n private variables.

Do not commit:

- n8n credential exports containing secrets
- n8n webhook production URLs with tokens
- workflow JSON exports containing real private URLs
- Google OAuth credential values
- Gmail credential values

If workflow exports are committed later, they must be sanitized.

Sanitized workflow exports may contain:

```text
<PLACEHOLDER_WEBHOOK_URL>
<PLACEHOLDER_GOOGLE_CREDENTIAL>
<PLACEHOLDER_SHEET_ID>
<PLACEHOLDER_DRIVE_FOLDER_ID>
```

## 15. n8n MCP Token Handling

The n8n MCP token is a secret.

For local Codex use, the real token may be stored in:

```text
.codex/config.toml
```

This file must be gitignored.

The public repository may contain:

```text
.codex/config.example.toml
```

with placeholders only.

Safe template:

```toml
[mcp_servers.n8n]
url = "https://<your-n8n-domain>/mcp-server/http"
http_headers = { "Authorization" = "Bearer <PASTE_N8N_MCP_TOKEN_HERE>" }
```

The real token must never be pasted into chat, committed to Git, or included in screenshots.

## 16. n8n MCP Smoke Test

A successful MCP connection can be verified by asking Codex to run an n8n MCP health check.

Expected smoke test:

```text
search_workflows returns successfully
```

A response like this is acceptable:

```json
{"data":[],"count":0}
```

This means:

```text
n8n MCP connection is live
authentication works
no workflows are currently visible or created
```

If the n8n tools do not appear in Codex, trigger tool discovery for:

```text
get_sdk_reference
create_workflow_from_code
validate_workflow
search_workflows
```

## 17. Google OAuth Credential Handling

Google OAuth credentials must be stored in n8n's credential store or other approved private credential storage.

Do not commit:

- OAuth client secret
- refresh token
- access token
- downloaded OAuth credential JSON
- n8n credential export containing Google credential material

Google OAuth setup steps should be documented using placeholders only.

## 18. Google Drive, Calendar, and Sheet IDs

Google Drive folder IDs, Calendar IDs, and Sheet IDs are private configuration.

They are not passwords, but they can expose private resources or reveal family-specific structure.

Do not commit real values.

Use placeholders in examples:

```text
<your-drive-folder-id>
<your-calendar-id>
<your-google-sheet-id>
```

Runtime values should live in:

- Private Google Sheet `Config` tab
- n8n private variables
- Secret Manager where appropriate
- Local ignored configuration for development

## 19. SchoolDiary Credential Handling

SchoolDiary credentials are high-risk secrets.

Rules:

1. Never commit username or password.
2. Never log username or password.
3. Never include credentials in screenshots.
4. Never store browser session state in Git.
5. Never commit Playwright storage state.
6. Prefer fresh login per scheduled run for MVP.
7. If session persistence is introduced later, store session material as secret runtime state only.

If SchoolDiary introduces CAPTCHA, OTP, or interactive verification, the automation must not bypass it.

The system should fail safely with an operational alert.

## 20. Evidence Privacy

SchoolDiary evidence can contain private educational and family information.

Do not commit:

- Real notice screenshots
- Real attachments
- Real homework PDFs
- Real images
- Real notice text
- Real downloaded files
- Real student names
- Real teacher names
- Real school names
- Real class names

Evidence should be stored only in the private Google Drive evidence folder.

Temporary evidence may pass through GCS but should not be made public.

## 21. Google Drive Sharing Rules

Google Drive evidence folder access should remain restricted.

Recommended model:

| Principal | Access |
|---|---|
| Parent/owner | Owner |
| Guardian | Viewer or Editor, depending on responsibility |
| Student | Viewer or Commenter |
| n8n connected Google account | Editor |
| Public internet | No access |

General access should remain:

```text
Restricted
```

Do not create public links unless explicitly required later.

## 22. GCS Security

GCS is a temporary handoff layer.

Rules:

1. GCS bucket must not be public.
2. Cloud Run service account should have only required object access.
3. n8n should receive only the paths it needs.
4. GCS paths should not be shown in family-facing emails.
5. Lifecycle cleanup should be configured later.
6. Evidence in GCS should be treated as private.

Recommended future retention:

```text
Delete temporary GCS objects after 7 to 30 days.
```

## 23. Log Redaction Rules

Logs must be safe by default.

Do not log:

- Passwords
- Tokens
- OAuth secrets
- Cookies
- Authorization headers
- Full webhook URLs with tokens
- Full private Google links when avoidable
- Raw SchoolDiary notice text in high-volume logs
- Real attachment contents
- Browser storage state
- Full stack traces containing secret environment variables

Safe logs may include:

- Run ID
- Workflow name
- Error category
- Count of notices captured
- Count of homework items created
- Count of duplicates skipped
- Count of evidence uploads
- Count of failed items
- Redacted IDs
- Hash prefixes
- Safe status values

Example safe log:

```json
{
  "run_id": "run_2026-07-03T20-30-00_Asia-Kolkata",
  "event": "notice_processing_complete",
  "notices_seen": 12,
  "homework_items_created": 3,
  "duplicates_skipped": 2,
  "evidence_failures": 0
}
```

## 24. Redaction Format

When a private value must be referenced for troubleshooting, redact it.

Recommended redaction:

```text
abcd1234...redacted
```

For hashes, short prefixes may be acceptable:

```text
notice_hash_short = ab12cd34
```

Do not expose full IDs unless necessary in a private operational context.

## 25. Email Safety

Family-facing digest emails must not include:

- Internal error traces
- GCS object paths
- n8n execution URLs
- Secret values
- OAuth details
- Browser session details
- Full raw logs

Operational alerts may include safe technical summaries but must still redact secrets.

## 26. Public Repository Safety

The following must not be committed:

```text
.env
.env.*
config/config.local.json
config/*.private.json
config/*.secrets.json
credentials/
secrets/
google-credentials*.json
service-account*.json
storage_state.json
.browser-state/
output/
downloads/
evidence/
logs/
*.log
.codex/config.toml
.codex/secrets.*
```

The following may be committed:

```text
.env.example
config/config.example.json
.codex/config.example.toml
docs/
src/
tests/
samples/synthetic/
README.md
LICENSE
```

## 27. Required `.gitignore` Rules

The repository `.gitignore` should include at minimum:

```gitignore
# Environment / secrets
.env
.env.*
!.env.example
credentials/
secrets/
google-credentials*.json
service-account*.json
storage_state.json

# Runtime output
output/
downloads/
evidence/
logs/
*.log

# Local config
config/config.local.json
config/*.private.json
config/*.secrets.json

# Codex local MCP config / secrets
.codex/config.toml
.codex/secrets.*
*.secrets.json
*.secrets.toml

# Python
__pycache__/
*.py[cod]
.venv/
venv/
.pytest_cache/

# Playwright
playwright-report/
test-results/
.browser-state/

# OS / IDE
.DS_Store
.vscode/
.idea/
```

## 28. Git Hygiene Before Commit

Before every commit, run:

```powershell
git status --short
git diff --cached --name-only
git diff --cached
```

Check that no private files are staged.

For ignored-file validation, use:

```powershell
git status --ignored
git check-ignore -v .codex/config.toml
git check-ignore -v config/config.local.json
git check-ignore -v .env
```

Expected result: private files should be ignored.

## 29. Secret Scanning

Before major commits or releases, run secret scanning locally if available.

Examples:

```powershell
git diff --cached
```

Optional tools may be added later:

```text
gitleaks
trufflehog
detect-secrets
```

These tools are useful but do not replace manual review.

## 30. Screenshot Safety

Screenshots can leak private information.

Before committing screenshots, verify that they do not contain:

- Real student names
- Guardian names
- Teacher names
- School names
- Real homework content
- URLs with tokens
- Browser cookies or extension data
- Email addresses
- Google file/folder IDs
- n8n workflow URLs
- n8n credentials
- SchoolDiary session data

MVP should avoid committing real screenshots entirely.

Use synthetic screenshots only if visual test fixtures are needed.

## 31. Sample Data Safety

Sample data must be synthetic.

Safe sample values:

```text
Synthetic Student
Synthetic Teacher
Synthetic School
Synthetic Class
Mathematics
Complete Ex 3A.
2026-06-25
```

Unsafe sample values:

```text
Real student name
Real teacher name
Real school name
Real portal text
Real attachment filename if identifying
Real Drive link
Real Calendar link
```

## 32. Least Privilege

Each component should receive only the access it needs.

### Cloud Run Job

Needs:

- Read SchoolDiary credentials
- Access SchoolDiary portal
- Write temporary evidence to GCS
- Post payload to n8n webhook
- Write logs

Does not need:

- Google Sheets access
- Google Calendar access
- Google Drive final evidence access
- Gmail access

### n8n

Needs:

- Receive Cloud Run payload
- Read temporary GCS evidence if configured
- Write final evidence to Google Drive
- Read/write Google Sheets
- Create Google Calendar events
- Send emails
- Read private Config tab

Does not need:

- SchoolDiary password
- Browser cookies
- Playwright session state
- Cloud Run runtime secrets unrelated to workflow execution

## 33. Separation of Duties

MVP separation:

```text
Cloud Run = capture and handoff
n8n = orchestration and Google workspace actions
Google Sheets = tracker and config
Google Drive = evidence
Google Calendar = visibility
Email = notification
```

This separation limits credential spread.

## 34. Incident Response: Secret Accidentally Committed

If a secret or private value is accidentally committed:

1. Stop using the exposed value immediately.
2. Rotate the affected secret or token.
3. Remove the value from the repository.
4. Rewrite Git history if the repository has not been widely cloned, or follow GitHub secret exposure guidance if already pushed.
5. Check GitHub/security alerts if available.
6. Review logs for unauthorized use.
7. Add or improve `.gitignore` rules.
8. Add a safer placeholder template if needed.
9. Document the incident privately.

Examples:

| Exposed Value | Required Action |
|---|---|
| SchoolDiary password | Change password |
| n8n webhook token | Rotate webhook token |
| n8n MCP token | Revoke/regenerate MCP token |
| Google OAuth client secret | Rotate OAuth client secret |
| Service account key | Disable/delete key and create a new one if still needed |
| Google Drive folder ID | Move to private config; review sharing settings |
| Recipient email | Remove from repo; consider privacy impact |

## 35. Incident Response: Evidence Accidentally Committed

If real evidence is committed:

1. Remove evidence files from the repository.
2. Remove from Git history if pushed.
3. Confirm public repository no longer exposes the file.
4. Check whether GitHub cached or indexed the file.
5. Replace with synthetic sample evidence if needed.
6. Review `.gitignore` coverage for evidence directories.
7. Review local output paths to prevent recurrence.

## 36. Runtime Failure Safety

If a credential is missing or invalid, the system should fail closed.

Examples:

| Failure | Expected Behavior |
|---|---|
| Missing SchoolDiary password | Cloud Run exits; no portal access attempted |
| Invalid n8n token | Cloud Run does not retry indefinitely |
| Missing Drive folder ID | n8n records operational error |
| Google credential expired | n8n sends operational alert |
| Digest recipient missing | n8n skips digest and sends admin alert if configured |

The system should not expose secrets in error messages.

## 37. Access Review

At minimum, periodically review:

- Google Drive folder sharing
- Google Calendar sharing
- Google Sheet sharing
- n8n credentials
- n8n MCP tokens
- Google Cloud service accounts
- Google Secret Manager access
- GitHub repository collaborators
- Local ignored config files

Suggested cadence:

```text
Monthly during build phase
Quarterly after MVP stabilizes
```

## 38. Open Decisions

The following decisions remain open:

1. Whether Cloud Run uses Google Secret Manager exclusively or a mix of Secret Manager and environment variables.
2. Whether n8n runtime config is stored primarily in Google Sheet `Config` tab or n8n variables.
3. Exact GCS lifecycle retention period.
4. Whether a separate automation Google account should be used after MVP.
5. Whether secret scanning tooling should be added to CI.
6. Whether workflow JSON exports will be committed and, if so, how they will be sanitized.
7. Whether Google Drive folder IDs should be stored in the Sheet `Config` tab or n8n private variables.
8. Whether access review should be documented as a recurring manual checklist.

## 39. Acceptance Criteria

The security and secrets design is implemented correctly when:

1. No real credentials are committed to Git.
2. No real SchoolDiary data is committed to Git.
3. No real evidence files are committed to Git.
4. No real Google Drive, Calendar, or Sheet IDs are committed to Git.
5. No real recipient emails are committed to Git.
6. Public templates contain placeholders only.
7. `config/config.local.json` is gitignored.
8. `.codex/config.toml` is gitignored.
9. Cloud Run secrets are stored in Secret Manager or equivalent private runtime storage.
10. n8n credentials are stored in n8n credential storage.
11. n8n runtime config is stored in private configuration, not hardcoded public files.
12. Logs redact secrets and avoid private content.
13. Family-facing emails do not expose internal technical details.
14. Operational alerts redact secrets.
15. Git hygiene checks are documented and followed before commits.
16. There is a defined response process for exposed secrets or evidence.

## 40. Summary

The SchoolDiary Homework Automation repository is public, so the security design is built around strict separation between public artifacts and private runtime values.

The public repository may contain source code, documentation, and placeholder templates. Real credentials, tokens, Google IDs, recipient emails, SchoolDiary data, and evidence must remain in private runtime configuration, Google Secret Manager, n8n credentials, ignored local files, or private Google Sheets.

The system should fail closed, redact logs, prevent accidental data exposure, and keep evidence in private Google Drive storage only.
