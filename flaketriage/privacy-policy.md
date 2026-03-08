---
layout: default
title: FlakeTriage - Privacy and Data Policy
permalink: /flaketriage/privacy-policy
---

# FlakeTriage — Privacy & Data Policy

> **Last updated:** 2026-03-05  
> **Version:** 1.1

---

## What FlakeTriage Does

FlakeTriage is a GitHub App that automatically re-runs failed CI jobs to detect flaky tests. It monitors `workflow_run.completed` events and, when a failure is detected, triggers a re-run of the failed jobs and posts a Check Run with an evidence summary.

FlakeTriage also supports manual retries via `/flaketriage retry <run_id>` comments on issues/PRs (`issue_comment` events), and processes GitHub Marketplace plan-change webhooks (`marketplace_purchase` events) to keep installation tiers current.

## What Data We Collect

FlakeTriage collects and processes **only workflow and operational metadata** — never your source code, build logs, test output, or personal data beyond your GitHub username.

### Data collected — workflow_run and push events:

| Data | Purpose | Retention |
|---|---|---|
| Repository name (`owner/repo`) | Route events, apply per-repo config | In-memory + SQLite (see below) |
| Workflow run ID | Track rerun decisions, prevent duplicates | SQLite, 30 days |
| Workflow name and path | Policy evaluation (deny sensitive workflows) | In-memory only |
| Run conclusion (`failure`, `success`, etc.) | Decide whether to trigger rerun | In-memory only |
| Head SHA | Pin config reads, create Check Runs | In-memory only |
| Installation ID | Authenticate API calls | In-memory only |
| GitHub delivery ID | Deduplicate webhook deliveries | SQLite, 30 days |
| Budget usage counters | Enforce daily rerun limits | SQLite, per UTC day |

### Data collected — issue_comment events (`/flaketriage retry`):

| Data | Purpose | Retention |
|---|---|---|
| Commenter login (`comment.user.login`) | Permission check (write/admin gate) | In-memory only (permission cache, 120s TTL) |
| Repository and org context (`repository.full_name`) | Route command, scope permission check | In-memory only |
| Comment ID (`comment.id`) | Idempotency / dedupe in structured logs | Structured log output only (not persisted in SQLite) |
| Command outcome (`rerun_triggered`, `permission_denied`, etc.) | Audit trail, Prometheus counter | Structured log + in-memory Prometheus counter |
| Requested workflow run ID | Target of the rerun action | In-memory only |

> **Note:** FlakeTriage does **not** store the full comment body. The comment text is parsed in-memory to extract the `/flaketriage retry <run_id>` command and then discarded.

### Data collected — marketplace_purchase events:

| Data | Purpose | Retention |
|---|---|---|
| Installation ID | Link plan to installation | SQLite (`installation_plans` table), until cancelled or overwritten |
| Plan name / slug | Map to internal tier | Logged (structured log); normalized slug stored in SQLite |
| Mapped internal tier (`free`/`starter`/`team`/`business`) | Enforce usage budget | SQLite (`installation_plans` table) |
| Account login | Audit trail | Structured log output only (not persisted in SQLite) |
| Action (`purchased`/`changed`/`cancelled`/`pending_change`) | Determine tier update | Structured log output only |
| Webhook source timestamp | Audit trail | Structured log output only |

### Operational telemetry (`/metrics` endpoint):

| Data | Scope | Protection |
|---|---|---|
| Aggregate Prometheus counters | Process-level totals (e.g., reruns attempted, plan changes processed) | Bearer-token protected; no repository names or user identifiers in metric labels by default |

### Data **NOT** collected:

- **Source code** — FlakeTriage never reads, downloads, or stores your source code (the only file read is `.github/flaketriage.yml` for configuration)
- **Build logs or test output** — We do not access or store CI logs, JUnit XML, or artifacts
- **Comment body text** — Full comment text is not stored; only the parsed command and outcome are logged
- **Pull request content** — Only metadata (run IDs, SHAs) is processed
- **Personal information** — No email addresses, names, or profile data beyond GitHub login
- **Secrets or tokens** — Installation tokens are ephemeral and never logged or stored
- **Workflow logs** — FlakeTriage does not ingest, download, or parse workflow log output

## How Data Is Stored

- **SQLite database**: A single-file database on the server stores run metadata, delivery IDs, and budget counters. The database is local to the FlakeTriage server instance.
- **In-memory caches**: Config cache (120s TTL), token cache, and run state are held in memory and lost on process restart.
- **No external analytics**: FlakeTriage does not send data to third-party analytics, tracking, or advertising services.
- **No cloud storage**: Data is not replicated to external databases or cloud object stores.

## Data Retention

| Data Type | Retention Period |
|---|---|
| Delivery ID deduplication records | 30 days |
| Rerun records | 30 days |
| Run state (pending outcome correlation) | 24 hours |
| Budget counters | Per UTC day (current day only) |
| Config cache entries | 120 seconds (TTL) |
| Prometheus metrics | Since last process restart (in-memory) |

Plan-tier retention targets (metrics history — aspirational, not yet code-enforced):
- **Free**: 7 days
- **Starter**: 30 days
- **Team**: 90 days
- **Business**: 365 days

> **Note:** Prometheus metrics are currently held in-memory and reset on process restart regardless of plan tier. The per-tier retention targets above represent planned behavior for a future persistent metrics store (see WI-039). Actual data retention today is governed by the SQLite retention periods in the table above and the process lifecycle.

## GitHub Permissions

FlakeTriage requests the minimum permissions necessary:

| Permission | Scope | Purpose |
|---|---|---|
| `actions:write` | Repository | Trigger re-run of failed jobs |
| `checks:write` | Repository | Create/update Check Runs with evidence |
| `contents:read` | Repository | Read `.github/flaketriage.yml` config file |
| `issues:write` | Repository | Post reaction/reply on `/flaketriage retry` commands |

FlakeTriage subscribes to these webhook events:

| Event | Purpose |
|---|---|
| `workflow_run` | Detect workflow completions and trigger reruns |
| `push` | Invalidate config cache on default-branch changes |
| `issue_comment` | Process `/flaketriage retry <run_id>` manual retry commands |
| `marketplace_purchase` | Ingest plan purchase, change, and cancellation events to update installation tier |

## Security

- All webhook payloads are verified using HMAC SHA-256 signature verification (`x-hub-signature-256` header) before any processing occurs. Events with invalid or missing signatures are rejected.
- The `/metrics` telemetry endpoint is protected by a bearer token (`FLAKETRIAGE_METRICS_TOKEN`). Requests without a valid token receive HTTP 401.
- Installation tokens are exchanged per-request with short TTL and never persisted.
- Structured JSON logging excludes secrets, tokens, and PII by design.
- See [SECURITY.md](SECURITY.md) for the full security model.

## GDPR & Data Subject Rights

FlakeTriage processes only GitHub-public workflow metadata, not personal data in the GDPR sense. However, if you believe FlakeTriage holds data about you:

- **Access**: Contact us to request a summary of data associated with your repository.
- **Deletion**: Uninstalling the GitHub App removes all webhook subscriptions. Server-side SQLite data for your repository (run records, delivery IDs, budget counters) expires per the retention policy above (30 days maximum). In-memory caches (config, token, run state) are cleared on process restart.
- **Portability**: Data can be exported in JSON format upon request.

## Data Deletion on Uninstall

When you uninstall the FlakeTriage GitHub App:
1. GitHub stops delivering webhook events immediately.
2. No new data is collected for your repositories.
3. Existing data (run records, budget counters) expires per the retention policy above.
4. Config cache entries expire within 120 seconds.

## Changes to This Policy

We will update this document when our data practices change. The "Last updated" date at the top reflects the most recent revision.

## Contact

For privacy questions or data requests:
- **Primary:** Open an issue in the FlakeTriage GitHub repository
- **Email:** binarytechtoppers@gmail.com
