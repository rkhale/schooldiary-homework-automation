# Deployment Runbook

## 1. Purpose

This runbook defines the deployment and operational setup steps for the SchoolDiary Homework Automation system.

The runbook uses placeholders only. It must not contain real SchoolDiary URLs, credentials, Google IDs, Google Drive links, Google Sheet links, Calendar IDs, n8n webhook URLs, tokens, recipient emails, or evidence files.

The commands are written for PowerShell.

## 2. Scope

This runbook covers:

- Local prerequisite checks
- Google Cloud project setup
- Required Google Cloud APIs
- Service account setup
- Secret Manager setup
- GCS temporary evidence bucket setup
- Artifact Registry image setup
- Cloud Run Job deployment
- Cloud Scheduler setup
- Google Workspace resource setup checklist
- n8n workflow setup checklist
- Manual execution and smoke tests
- Enablement checklist
- Pause, rollback, and recovery procedures
- Public repository safety checks

## 3. Out of Scope

This runbook does not define:

- SchoolDiary selector implementation
- Full n8n workflow implementation
- Google OAuth consent screen details
- n8n credential creation details
- Full test strategy
- Production monitoring dashboards
- WhatsApp/SMS/push notification setup
- OCR/deep attachment extraction

Those are covered in other design documents or future implementation stages.

## 4. Relationship to Other Documents

| Document | Relevance |
|---|---|
| `docs/01-architecture.md` | Defines end-to-end architecture |
| `docs/02-cloud-run-job-design.md` | Defines Cloud Run Job behavior |
| `docs/03-schooldiary-fetcher-design.md` | Defines SchoolDiary fetcher behavior |
| `docs/04-n8n-workflow-design.md` | Defines n8n workflow design |
| `docs/05-data-model.md` | Defines payload and tracker model |
| `docs/06-google-calendar-design.md` | Defines Google Calendar event behavior |
| `docs/07-google-drive-evidence-design.md` | Defines Google Drive evidence storage |
| `docs/08-notification-design.md` | Defines daily digest and operational alerts |
| `docs/09-security-and-secrets.md` | Defines secret/private config handling |
| `docs/10-error-handling-and-observability.md` | Defines run logs, error logs, and health checks |
| `docs/12-testing-strategy.md` | Will define formal testing |
| `docs/13-google-oauth-and-n8n-credential-setup.md` | Will define Google OAuth and n8n credential steps |

## 5. Deployment Principle

The deployment model is:

```text
Google Cloud Scheduler
  → Google Cloud Run Job
  → Playwright SchoolDiary Fetcher
  → Google Cloud Storage temporary evidence handoff
  → n8n Webhook
  → n8n workflows
  → Google Sheets / Google Drive / Google Calendar / Email
```

The public repository stores only code, docs, and placeholders.

Runtime secrets and private configuration are stored outside Git.

## 6. Environment Placeholders

Use these placeholders throughout this runbook.

| Placeholder | Meaning |
|---|---|
| `<gcp-project-id>` | Google Cloud project ID |
| `<gcp-project-number>` | Google Cloud project number |
| `<region>` | Google Cloud region, for example `asia-south1` |
| `<scheduler-region>` | Cloud Scheduler region |
| `<artifact-repo-name>` | Artifact Registry repository name |
| `<image-name>` | Container image name |
| `<job-name>` | Cloud Run Job name |
| `<gcs-bucket-name>` | Temporary GCS evidence bucket |
| `<cloud-run-service-account>` | Runtime service account |
| `<scheduler-service-account>` | Scheduler invoker service account |
| `<n8n-webhook-url>` | Private n8n webhook URL |
| `<schooldiary-noticeboard-url>` | Private SchoolDiary Notice Board URL |
| `<google-sheet-id>` | Private Google Sheet ID |
| `<drive-folder-id>` | Private Google Drive root folder ID |
| `<calendar-id>` | Private Google Calendar ID |

Do not replace these placeholders in committed files.

## 7. Recommended Naming

Recommended non-secret resource names:

```text
Cloud Run Job: schooldiary-homework-fetcher
Cloud Run runtime service account: schooldiary-cloudrun-job
Cloud Scheduler service account: schooldiary-scheduler-invoker
Artifact Registry repository: schooldiary-automation
GCS bucket: <gcp-project-id>-schooldiary-evidence-tmp
GCS prefix: schooldiary-evidence
```

Resource names may be adjusted during deployment, but private IDs and secrets must remain outside Git.

## 8. Local Prerequisites

Install and authenticate:

1. Git
2. Python
3. Google Cloud CLI
4. Docker or compatible container build path, if building locally
5. Access to the Google Cloud project
6. Access to n8n
7. Access to the Google account used for Sheets, Drive, Calendar, and email

Check tools:

```powershell
git --version
python --version
gcloud --version
```

Authenticate to Google Cloud:

```powershell
gcloud auth login
gcloud auth application-default login
```

Set the project:

```powershell
gcloud config set project "<gcp-project-id>"
```

## 9. Local Repository Safety Check

Before any deployment work, confirm private files are ignored:

```powershell
git status --short
git status --ignored
git check-ignore -v .codex/config.toml
git check-ignore -v config/config.local.json
git check-ignore -v .env
```

Expected behavior:

- `.codex/config.toml` is ignored.
- `config/config.local.json` is ignored.
- `.env` is ignored.
- `config/config.example.json` is not ignored and contains placeholders only.

## 10. Set Local PowerShell Variables

Use local variables to reduce copy/paste mistakes.

```powershell
$PROJECT_ID = "<gcp-project-id>"
$PROJECT_NUMBER = "<gcp-project-number>"
$REGION = "<region>"
$SCHEDULER_REGION = "<scheduler-region>"
$ARTIFACT_REPO = "<artifact-repo-name>"
$IMAGE_NAME = "<image-name>"
$JOB_NAME = "<job-name>"
$GCS_BUCKET = "<gcs-bucket-name>"
$GCS_PREFIX = "schooldiary-evidence"
$CLOUD_RUN_SA_NAME = "<cloud-run-service-account>"
$SCHEDULER_SA_NAME = "<scheduler-service-account>"
```

Example region choice for Pune/India deployments:

```powershell
$REGION = "asia-south1"
$SCHEDULER_REGION = "asia-south1"
```

Use the actual region only in private/local commands, not in public committed files if the region is considered private for your setup.

## 11. Enable Google Cloud APIs

Enable required APIs:

```powershell
gcloud services enable run.googleapis.com
gcloud services enable cloudscheduler.googleapis.com
gcloud services enable secretmanager.googleapis.com
gcloud services enable storage.googleapis.com
gcloud services enable artifactregistry.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable logging.googleapis.com
```

Optional later:

```powershell
gcloud services enable monitoring.googleapis.com
```

## 12. Create Service Accounts

Create Cloud Run runtime service account:

```powershell
gcloud iam service-accounts create $CLOUD_RUN_SA_NAME --display-name="SchoolDiary Cloud Run Job"
```

Create Scheduler invoker service account:

```powershell
gcloud iam service-accounts create $SCHEDULER_SA_NAME --display-name="SchoolDiary Scheduler Invoker"
```

Set service account email variables:

```powershell
$CLOUD_RUN_SA_EMAIL = "$CLOUD_RUN_SA_NAME@$PROJECT_ID.iam.gserviceaccount.com"
$SCHEDULER_SA_EMAIL = "$SCHEDULER_SA_NAME@$PROJECT_ID.iam.gserviceaccount.com"
```

## 13. Create Secret Manager Secrets

Create secret containers:

```powershell
gcloud secrets create schooldiary-username --replication-policy="automatic"
gcloud secrets create schooldiary-password --replication-policy="automatic"
gcloud secrets create schooldiary-noticeboard-url --replication-policy="automatic"
gcloud secrets create n8n-webhook-url --replication-policy="automatic"
gcloud secrets create n8n-webhook-token --replication-policy="automatic"
```

Add secret versions without putting real values in Git.

PowerShell pattern for a plain text secret:

```powershell
$secretValue = Read-Host "Enter secret value"
$secretValue | gcloud secrets versions add schooldiary-username --data-file=-
$secretValue = $null
```

PowerShell pattern for a password-like secret:

```powershell
$secureValue = Read-Host "Enter secret value" -AsSecureString
$bstr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureValue)
$plainValue = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($bstr)
$plainValue | gcloud secrets versions add schooldiary-password --data-file=-
[Runtime.InteropServices.Marshal]::ZeroFreeBSTR($bstr)
$plainValue = $null
$secureValue = $null
```

Repeat for each required secret, changing the secret name.

Do not paste real secrets into committed files or chat.

## 14. Grant Cloud Run Access to Secrets

Grant the Cloud Run runtime service account access to required secrets.

Project-level option:

```powershell
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$CLOUD_RUN_SA_EMAIL" --role="roles/secretmanager.secretAccessor"
```

A stricter per-secret grant can be used later if needed.

## 15. Create Temporary GCS Evidence Bucket

Create the GCS bucket:

```powershell
gcloud storage buckets create "gs://$GCS_BUCKET" --location="$REGION" --uniform-bucket-level-access
```

Recommended bucket posture:

```powershell
gcloud storage buckets update "gs://$GCS_BUCKET" --public-access-prevention
```

Grant Cloud Run runtime service account object access:

```powershell
gcloud storage buckets add-iam-policy-binding "gs://$GCS_BUCKET" --member="serviceAccount:$CLOUD_RUN_SA_EMAIL" --role="roles/storage.objectAdmin"
```

If n8n will read GCS directly using Google credentials, grant the n8n Google principal appropriate access privately. For MVP, this detail is finalized in the n8n credential setup document.

## 16. Configure GCS Lifecycle

GCS is temporary handoff only.

Recommended lifecycle JSON:

```json
{
  "rule": [
    {
      "action": {
        "type": "Delete"
      },
      "condition": {
        "age": 30
      }
    }
  ]
}
```

Save this as a local file only if needed:

```text
infra/gcs-lifecycle.example.json
```

Apply lifecycle configuration:

```powershell
gcloud storage buckets update "gs://$GCS_BUCKET" --lifecycle-file="infra/gcs-lifecycle.example.json"
```

Do not apply lifecycle deletion until the evidence finalization workflow has been tested.

## 17. Create Artifact Registry Repository

Create Docker repository:

```powershell
gcloud artifacts repositories create $ARTIFACT_REPO --repository-format=docker --location=$REGION --description="SchoolDiary automation container images"
```

Configure Docker authentication for the region:

```powershell
gcloud auth configure-docker "$REGION-docker.pkg.dev"
```

Set image URI:

```powershell
$IMAGE_URI = "$REGION-docker.pkg.dev/$PROJECT_ID/$ARTIFACT_REPO/$IMAGE_NAME:latest"
```

## 18. Build and Push Container Image

When the implementation and Dockerfile are ready, build using Cloud Build:

```powershell
gcloud builds submit --tag $IMAGE_URI .
```

If the implementation is not ready yet, skip this step.

## 19. Deploy Cloud Run Job

Deploy the Cloud Run Job after the container image exists.

```powershell
gcloud run jobs deploy $JOB_NAME --image=$IMAGE_URI --region=$REGION --service-account=$CLOUD_RUN_SA_EMAIL --tasks=1 --max-retries=0 --task-timeout=900s --set-env-vars="TIMEZONE=Asia/Kolkata,OUTPUT_DIR=/tmp/schooldiary-homework,GCP_PROJECT_ID=$PROJECT_ID,GCS_EVIDENCE_BUCKET=$GCS_BUCKET,GCS_EVIDENCE_PREFIX=$GCS_PREFIX,HEADLESS=true,MAX_NOTICES_PER_RUN=20,DOWNLOAD_ATTACHMENTS=true,CAPTURE_SCREENSHOT=true,POST_TO_N8N=true" --set-secrets="SCHOOLDIARY_USERNAME=schooldiary-username:latest,SCHOOLDIARY_PASSWORD=schooldiary-password:latest,SCHOOLDIARY_NOTICEBOARD_URL=schooldiary-noticeboard-url:latest,N8N_WEBHOOK_URL=n8n-webhook-url:latest,N8N_WEBHOOK_TOKEN=n8n-webhook-token:latest"
```

For initial dry-run deployment, use:

```powershell
gcloud run jobs deploy $JOB_NAME --image=$IMAGE_URI --region=$REGION --service-account=$CLOUD_RUN_SA_EMAIL --tasks=1 --max-retries=0 --task-timeout=900s --set-env-vars="TIMEZONE=Asia/Kolkata,OUTPUT_DIR=/tmp/schooldiary-homework,GCP_PROJECT_ID=$PROJECT_ID,GCS_EVIDENCE_BUCKET=$GCS_BUCKET,GCS_EVIDENCE_PREFIX=$GCS_PREFIX,HEADLESS=true,MAX_NOTICES_PER_RUN=5,DOWNLOAD_ATTACHMENTS=false,CAPTURE_SCREENSHOT=true,POST_TO_N8N=false" --set-secrets="SCHOOLDIARY_USERNAME=schooldiary-username:latest,SCHOOLDIARY_PASSWORD=schooldiary-password:latest,SCHOOLDIARY_NOTICEBOARD_URL=schooldiary-noticeboard-url:latest,N8N_WEBHOOK_URL=n8n-webhook-url:latest,N8N_WEBHOOK_TOKEN=n8n-webhook-token:latest"
```

## 20. Manually Execute Cloud Run Job

Run manually before enabling the schedule:

```powershell
gcloud run jobs execute $JOB_NAME --region=$REGION --wait
```

Check executions:

```powershell
gcloud run jobs executions list --job=$JOB_NAME --region=$REGION
```

Read logs through Google Cloud Console or gcloud logging commands.

Do not proceed to scheduler enablement until manual execution is understood.

## 21. Create Cloud Scheduler Jobs

The target schedule is:

```text
07:00 Asia/Kolkata
15:30 Asia/Kolkata
20:30 Asia/Kolkata
```

Recommended scheduler job names:

```text
schooldiary-fetch-0700
schooldiary-fetch-1530
schooldiary-fetch-2030
```

Grant scheduler service account permission to invoke Cloud Run jobs.

Project-level option:

```powershell
gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:$SCHEDULER_SA_EMAIL" --role="roles/run.invoker"
```

Create scheduler jobs:

```powershell
gcloud scheduler jobs create http "schooldiary-fetch-0700" --location=$SCHEDULER_REGION --schedule="0 7 * * *" --time-zone="Asia/Kolkata" --uri="https://run.googleapis.com/v2/projects/$PROJECT_ID/locations/$REGION/jobs/$JOB_NAME:run" --http-method=POST --oauth-service-account-email=$SCHEDULER_SA_EMAIL
```

```powershell
gcloud scheduler jobs create http "schooldiary-fetch-1530" --location=$SCHEDULER_REGION --schedule="30 15 * * *" --time-zone="Asia/Kolkata" --uri="https://run.googleapis.com/v2/projects/$PROJECT_ID/locations/$REGION/jobs/$JOB_NAME:run" --http-method=POST --oauth-service-account-email=$SCHEDULER_SA_EMAIL
```

```powershell
gcloud scheduler jobs create http "schooldiary-fetch-2030" --location=$SCHEDULER_REGION --schedule="30 20 * * *" --time-zone="Asia/Kolkata" --uri="https://run.googleapis.com/v2/projects/$PROJECT_ID/locations/$REGION/jobs/$JOB_NAME:run" --http-method=POST --oauth-service-account-email=$SCHEDULER_SA_EMAIL
```

Create the scheduler jobs paused first if supported in your deployment approach, or create them and immediately pause until smoke tests are complete.

Pause scheduler jobs:

```powershell
gcloud scheduler jobs pause "schooldiary-fetch-0700" --location=$SCHEDULER_REGION
gcloud scheduler jobs pause "schooldiary-fetch-1530" --location=$SCHEDULER_REGION
gcloud scheduler jobs pause "schooldiary-fetch-2030" --location=$SCHEDULER_REGION
```

Resume scheduler jobs after smoke tests pass:

```powershell
gcloud scheduler jobs resume "schooldiary-fetch-0700" --location=$SCHEDULER_REGION
gcloud scheduler jobs resume "schooldiary-fetch-1530" --location=$SCHEDULER_REGION
gcloud scheduler jobs resume "schooldiary-fetch-2030" --location=$SCHEDULER_REGION
```

## 22. Manual Scheduler Test

Run a scheduler job manually:

```powershell
gcloud scheduler jobs run "schooldiary-fetch-2030" --location=$SCHEDULER_REGION
```

Then verify:

1. Cloud Run execution started.
2. Cloud Run completed.
3. n8n received payload if `POST_TO_N8N=true`.
4. Google Sheet `Run Log` was updated.
5. No duplicate rows were created.

## 23. Google Workspace Resource Setup Checklist

These resources are created outside Git.

### 23.1 Google Sheet

Create the SchoolDiary tracker spreadsheet with these tabs:

```text
Homework Tracker
Notice Index
Run Log
Processing Errors
Daily Digest Log
Config
```

Optional later:

```text
Calendar Event Log
Evidence Log
Operational Metrics
```

Record the Google Sheet ID privately.

Do not commit the real Sheet ID.

### 23.2 Google Drive Evidence Folder

Create root folder:

```text
SchoolDiary Homework Evidence
```

Record the root folder ID privately.

The user creates only the root folder. n8n creates date and notice child folders automatically.

### 23.3 Google Calendar

Create or use dedicated calendar:

```text
SchoolDiary Homework
```

Record the Calendar ID privately.

Do not commit the real Calendar ID.

### 23.4 Email Sending

Configure n8n Gmail or email credential privately.

Do not commit real recipients or OAuth credential material.

## 24. Google Sheet Config Tab

For MVP, n8n runtime configuration should be stored in the private Google Sheet `Config` tab or in n8n private variables.

Recommended Config tab key-value rows:

| Key | Value |
|---|---|
| `google_sheets.tracker_spreadsheet_id` | `<google-sheet-id>` |
| `google_drive.evidence_root_folder_id` | `<drive-folder-id>` |
| `google_calendar.homework_calendar_id` | `<calendar-id>` |
| `notifications.digest_email_to` | `<family-recipient-list>` |
| `notifications.digest_email_cc` |  |
| `notifications.digest_email_bcc` |  |
| `notifications.operational_alert_email_to` | `<admin-email>` |
| `notifications.digest_send_time` | `20:45` |
| `notifications.digest_timezone` | `Asia/Kolkata` |
| `notifications.digest_lookahead_days` | `7` |
| `notifications.digest_send_empty` | `false` |

Do not commit the real Config tab values.

## 25. n8n Workflow Setup Checklist

Recommended workflows:

```text
01 - SchoolDiary Homework Intake
02 - Daily Homework Digest
03 - Overdue Status Update
04 - Weekly Summary
```

### 25.1 Intake Workflow

Trigger:

```text
Webhook
```

Responsibilities:

1. Validate webhook token.
2. Validate payload.
3. Write or update `Run Log`.
4. Check `Notice Index`.
5. Dedupe notices and homework items.
6. Classify/itemize notices.
7. Write `Homework Tracker`.
8. Finalize evidence to Google Drive.
9. Create Google Calendar events where eligible.
10. Record errors.

### 25.2 Daily Digest Workflow

Trigger:

```text
Schedule at 20:45 Asia/Kolkata
```

Responsibilities:

1. Read Config.
2. Check `Daily Digest Log`.
3. Read eligible tracker rows.
4. Group digest sections.
5. Send one family-facing email if applicable.
6. Write `Daily Digest Log`.

### 25.3 Overdue Status Update Workflow

Trigger:

```text
Schedule at 20:40 Asia/Kolkata
```

Responsibilities:

1. Read tracker.
2. Find rows with `Due Date < today`.
3. Exclude `Completed`, `Not Applicable`, and `For Information`.
4. Set `Status = Overdue`.

### 25.4 Weekly Summary Workflow

Optional for MVP.

Trigger:

```text
Weekly schedule
```

Can remain disabled until daily digest stabilizes.

## 26. n8n MCP Development Setup

Codex can use n8n MCP for workflow creation and validation.

Local MCP config:

```text
.codex/config.toml
```

This file must remain gitignored.

Public template:

```text
.codex/config.example.toml
```

Smoke test:

```text
search_workflows returns successfully
```

A response such as this means the MCP connection is live:

```json
{"data":[],"count":0}
```

`count: 0` means no workflows are currently visible or created. It does not mean the MCP connection failed.

## 27. Local Development Run

Local dry-run should avoid posting to n8n unless explicitly enabled.

Example:

```powershell
$env:POST_TO_N8N = "false"
$env:HEADLESS = "true"
$env:OUTPUT_DIR = "C:\Temp\schooldiary-homework"
$env:TIMEZONE = "Asia/Kolkata"
$env:MAX_NOTICES_PER_RUN = "5"
$env:DOWNLOAD_ATTACHMENTS = "false"
$env:CAPTURE_SCREENSHOT = "true"

python -m src.main
```

Do not use real credentials in committed files.

## 28. Deployment Smoke Tests

Run these before enabling scheduled execution.

### 28.1 Git Safety

```powershell
git status --short
git status --ignored
git diff --cached --name-only
```

Confirm no private files are staged.

### 28.2 GCS Write Test

Create a synthetic test file:

```powershell
"synthetic test" | Set-Content -Path ".\tmp-gcs-test.txt"
gcloud storage cp ".\tmp-gcs-test.txt" "gs://$GCS_BUCKET/$GCS_PREFIX/smoke-test/tmp-gcs-test.txt"
Remove-Item ".\tmp-gcs-test.txt"
```

Confirm upload, then delete it if desired:

```powershell
gcloud storage rm "gs://$GCS_BUCKET/$GCS_PREFIX/smoke-test/tmp-gcs-test.txt"
```

### 28.3 Cloud Run Dry Run

Deploy with:

```text
POST_TO_N8N=false
MAX_NOTICES_PER_RUN=5
DOWNLOAD_ATTACHMENTS=false
```

Execute:

```powershell
gcloud run jobs execute $JOB_NAME --region=$REGION --wait
```

Expected:

1. Job starts.
2. Job exits cleanly or fails with a clear expected error.
3. Logs contain no secrets.
4. No n8n workflow is called.

### 28.4 n8n Webhook Test

After n8n intake is ready, use synthetic payload only.

Expected:

1. Webhook accepts valid payload.
2. Invalid token is rejected.
3. Synthetic run appears in `Run Log`.
4. Synthetic notice appears in `Notice Index`.
5. No real data is required.

### 28.5 Google Drive Test

Expected:

1. n8n can create child folder under `SchoolDiary Homework Evidence`.
2. n8n can upload a synthetic file.
3. n8n can write the Drive link to the test tracker row.
4. General access remains restricted.

### 28.6 Google Calendar Test

Expected:

1. n8n can create a synthetic all-day event.
2. Event appears in `SchoolDiary Homework` calendar.
3. Calendar event ID/link is written back.
4. Duplicate event is not created on re-run.

### 28.7 Daily Digest Test

Expected:

1. Digest uses synthetic tracker rows.
2. One email is sent to configured test recipients.
3. `Daily Digest Log` is updated.
4. Re-running same digest does not send duplicate email.
5. Empty digest is skipped if configured.

## 29. Enablement Checklist

Do not enable the production schedule until all are true:

```text
Cloud Run dry run completed
n8n webhook validated with synthetic payload
Google Sheet tracker tabs created
Google Sheet Config tab populated privately
Google Drive root folder created
n8n can write Drive evidence
Google Calendar created
n8n can create calendar event
Daily digest tested with synthetic data
Operational alert tested
No private values committed
Scheduler jobs created and paused
Manual scheduler run tested
```

After all tests pass:

```powershell
gcloud scheduler jobs resume "schooldiary-fetch-0700" --location=$SCHEDULER_REGION
gcloud scheduler jobs resume "schooldiary-fetch-1530" --location=$SCHEDULER_REGION
gcloud scheduler jobs resume "schooldiary-fetch-2030" --location=$SCHEDULER_REGION
```

## 30. Normal Operations Checklist

Daily/weekly check during early MVP:

1. Check `Run Log`.
2. Check `Processing Errors`.
3. Check `Daily Digest Log`.
4. Confirm no repeated login failures.
5. Confirm no evidence permission failures.
6. Confirm no duplicate tracker rows.
7. Confirm digest did not send duplicate emails.
8. Confirm Google Drive evidence remains restricted.

## 31. Pause Procedure

Pause all scheduled captures:

```powershell
gcloud scheduler jobs pause "schooldiary-fetch-0700" --location=$SCHEDULER_REGION
gcloud scheduler jobs pause "schooldiary-fetch-1530" --location=$SCHEDULER_REGION
gcloud scheduler jobs pause "schooldiary-fetch-2030" --location=$SCHEDULER_REGION
```

Use pause when:

- SchoolDiary login is repeatedly failing.
- Google credentials expired.
- n8n workflow is being changed.
- Duplicate row issue is detected.
- Evidence privacy issue is suspected.
- Secret exposure incident is being investigated.

## 32. Resume Procedure

Resume only after the issue is resolved and one manual execution succeeds:

```powershell
gcloud run jobs execute $JOB_NAME --region=$REGION --wait
```

Then resume schedules:

```powershell
gcloud scheduler jobs resume "schooldiary-fetch-0700" --location=$SCHEDULER_REGION
gcloud scheduler jobs resume "schooldiary-fetch-1530" --location=$SCHEDULER_REGION
gcloud scheduler jobs resume "schooldiary-fetch-2030" --location=$SCHEDULER_REGION
```

## 33. Rollback Procedure

If a new Cloud Run image causes failure:

1. Pause scheduler jobs.
2. Identify the previous working image tag.
3. Redeploy the job with the previous image.
4. Execute manually.
5. Confirm logs and n8n behavior.
6. Resume scheduler only after successful manual test.

Template:

```powershell
$PREVIOUS_IMAGE_URI = "<previous-working-image-uri>"

gcloud run jobs deploy $JOB_NAME --image=$PREVIOUS_IMAGE_URI --region=$REGION --service-account=$CLOUD_RUN_SA_EMAIL
gcloud run jobs execute $JOB_NAME --region=$REGION --wait
```

## 34. Secret Rotation Procedure

If a secret changes:

1. Add a new secret version.
2. Confirm Cloud Run references `latest` or update job to reference the intended version.
3. Manually execute Cloud Run Job.
4. Confirm n8n intake if applicable.
5. Resume schedules if paused.

Example:

```powershell
$secretValue = Read-Host "Enter new secret value"
$secretValue | gcloud secrets versions add n8n-webhook-token --data-file=-
$secretValue = $null
```

If a secret was exposed, rotate immediately and follow the incident response process in `docs/09-security-and-secrets.md`.

## 35. Disable Procedure

To stop scheduled automation without deleting resources:

```powershell
gcloud scheduler jobs pause "schooldiary-fetch-0700" --location=$SCHEDULER_REGION
gcloud scheduler jobs pause "schooldiary-fetch-1530" --location=$SCHEDULER_REGION
gcloud scheduler jobs pause "schooldiary-fetch-2030" --location=$SCHEDULER_REGION
```

To delete scheduler jobs:

```powershell
gcloud scheduler jobs delete "schooldiary-fetch-0700" --location=$SCHEDULER_REGION
gcloud scheduler jobs delete "schooldiary-fetch-1530" --location=$SCHEDULER_REGION
gcloud scheduler jobs delete "schooldiary-fetch-2030" --location=$SCHEDULER_REGION
```

To delete the Cloud Run Job:

```powershell
gcloud run jobs delete $JOB_NAME --region=$REGION
```

Do not delete GCS or Google Drive evidence until retention and evidence requirements are reviewed.

## 36. Cost Control Notes

Expected MVP cost drivers:

- Cloud Run Job executions
- Cloud Build image builds
- Artifact Registry image storage
- GCS temporary evidence storage
- Cloud Scheduler jobs
- n8n Cloud subscription/usage
- Google Workspace storage
- AI/LLM usage, if enabled

Cost controls:

1. Keep Cloud Run scheduled runs to three per day.
2. Limit `MAX_NOTICES_PER_RUN`.
3. Keep GCS lifecycle cleanup enabled after evidence finalization is proven.
4. Avoid large attachment mirroring unless needed.
5. Keep digest to one email per day.
6. Track AI usage if AI formatting/classification is used.

## 37. Public Repository Safety Before Commit

Before committing deployment-related files:

```powershell
git status --short
git diff --cached --name-only
git diff --cached
```

Do not commit:

```text
real SchoolDiary URL
real SchoolDiary credentials
real n8n webhook URL/token
real n8n MCP token
real Google IDs
real Google Drive links
real Google Sheet links
real Calendar IDs
real recipient emails
real evidence
config/config.local.json
.codex/config.toml
.env
```

Safe to commit:

```text
docs/11-deployment-runbook.md
config/config.example.json
infra/*.example.json
synthetic examples
placeholder commands
```

## 38. Open Decisions

The following decisions remain open:

1. Final Google Cloud project and region.
2. Final GCS bucket name.
3. Whether n8n reads GCS directly or Cloud Run provides signed/private handoff another way.
4. Whether GCS lifecycle retention should be 7, 14, or 30 days.
5. Whether Cloud Run uses per-secret IAM grants instead of project-level Secret Accessor.
6. Whether a separate automation Google account should be used after MVP.
7. Whether scheduler jobs should be created through console or command line.
8. Whether weekly summary is enabled in MVP or deferred.
9. Whether monitoring dashboards are added before or after MVP.

## 39. Acceptance Criteria

Deployment is complete when:

1. Required Google Cloud APIs are enabled.
2. Cloud Run runtime service account exists.
3. Scheduler invoker service account exists.
4. Required secrets exist in Secret Manager.
5. Cloud Run runtime service account can access required secrets.
6. Temporary GCS bucket exists and is not public.
7. Cloud Run Job is deployed.
8. Manual Cloud Run execution works.
9. n8n intake webhook works with synthetic payload.
10. Google Sheet tracker tabs exist.
11. Private Config tab is populated.
12. Google Drive root folder exists.
13. n8n can create Drive child folders and upload synthetic evidence.
14. Google Calendar exists.
15. n8n can create a synthetic calendar event.
16. Daily digest workflow sends one test digest and prevents duplicate send.
17. Scheduler jobs exist and are initially paused.
18. Manual scheduler run works.
19. Scheduler jobs are resumed only after smoke tests pass.
20. No real private values are committed to Git.

## 40. Summary

This runbook deploys the SchoolDiary Homework Automation system as a scheduled Cloud Run Job orchestrated by Cloud Scheduler, with n8n handling downstream workflow automation.

The deployment must start with private configuration and security controls, then proceed through GCS, Cloud Run, n8n, Google Sheets, Google Drive, Calendar, and digest testing.

The scheduler should remain paused until manual Cloud Run execution, n8n intake, evidence finalization, calendar creation, and daily digest behavior have all passed smoke testing.
