# schooldiary-homework-automation

Public repository scaffold for automating homework retrieval from a SchoolDiary-style system and routing normalized outputs into downstream workflows.

## Planned architecture

- `cloud-run-job/`: planned Python service for scheduled fetch and normalization logic
- `n8n/`: planned workflow definitions and prompt assets for orchestration
- `infra/`: planned infrastructure and deployment configuration
- `docs/`: design notes, decision records, and operational runbooks
- `samples/`: sanitized example inputs and outputs only
- `scripts/`: helper scripts for local development and maintenance

## Current status

This repository currently contains the initial project skeleton and documentation placeholders only.

Fetcher code, browser automation, workflow implementations, infrastructure definitions, and integrations are intentionally not implemented yet.

## Security notice

This repository is intended to remain public-safe.

Do not commit real credentials, real SchoolDiary URLs, real student data, screenshots, PDFs, browser cookies, session files, or any other sensitive artifacts.
