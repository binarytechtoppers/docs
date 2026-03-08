---
layout: default
title: User Guide - FlakeTriage
permalink: /flaketriage/user-guide
---

# User Guide — FlakeTriage

> **Audience:** Developers and engineering managers who install FlakeTriage on their GitHub repositories.  
> **Prerequisites:** A GitHub account, at least one repository with GitHub Actions workflows.  

---

## Quick Start

Get FlakeTriage running on your repository in under 5 minutes:

1. **Install the GitHub App** — Go to the [GitHub Marketplace listing](https://github.com/marketplace/flaketriage), click **Install**, and grant access to your organization or specific repositories.

2. **Done.** FlakeTriage works immediately with safe defaults — no config file required.

3. **(Optional) Customize** — Create `.github/flaketriage.yml` in your repository to adjust the daily rerun budget or allow/deny specific workflows.

4. **Verify** — Push a change that triggers a CI workflow. If it fails, FlakeTriage will automatically rerun the failed jobs and post a **Check Run** on the commit with the decision and evidence.

> **Zero-config:** If you skip step 3, FlakeTriage uses safe defaults: enabled, 10 reruns/day budget (capped by your plan tier), built-in deny patterns for deploy/release workflows.

---

## Installation

### Organization vs Repository Install

When you install FlakeTriage from the Marketplace, GitHub asks you to choose the installation scope:

| Scope | Behavior |
|-------|----------|
| **All repositories** | FlakeTriage monitors every repository in the organization (including future ones). |
| **Only select repositories** | FlakeTriage monitors only the repositories you choose. You can add or remove repositories later from your organization's **Installed GitHub Apps** settings. |

**Recommendation:** Start with "Only select repositories" for a controlled rollout, then expand to "All repositories" once you're comfortable.

**To change scope later:**
1. Go to your organization's **Settings → Integrations → GitHub Apps → FlakeTriage → Configure**.
2. Change "Repository access" to your preference.

### Required Permissions

FlakeTriage requests the minimum permissions necessary to function:

| Permission | Scope | Why It's Needed |
|---|---|---|
| `actions:write` | Repository | Trigger re-run of failed jobs via the GitHub API |
| `checks:write` | Repository | Create and update Check Runs with evidence summaries |
| `contents:read` | Repository | Read `.github/flaketriage.yml` configuration file |
| `issues:write` | Repository | Post reactions and reply comments for `/flaketriage retry` feedback |
| `metadata:read` | Repository | Required by GitHub for all App installations (automatic) |

> **Privacy:** FlakeTriage reads only workflow metadata and one small config file. It never reads source code, build logs, test output, or personal data. See [docs/PRIVACY_POLICY.md](PRIVACY_POLICY.md) for full details.

### Webhook Events

FlakeTriage subscribes to these webhook events:

| Event | Purpose |
|-------|---------|
| `workflow_run` | Detect completed workflow runs (trigger automatic rerun) |
| `push` | Invalidate config cache on default-branch push (instant config reload) |
| `issue_comment` | Listen for `/flaketriage retry` manual retry commands |
| `marketplace_purchase` | Process plan changes (purchased, changed, cancelled) |

---

## Configuration

### Config File Location

```
.github/flaketriage.yml
```

This file is optional. If it's missing, FlakeTriage uses safe defaults. The file is read from the **exact commit SHA** being processed (not the latest on the default branch), so config changes take effect on the next workflow run after the config is pushed.

**Config caching:** The config file is cached in memory for **120 seconds**. Pushing to the default branch triggers **instant cache invalidation** — you don't have to wait.

### Configuration Reference

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enabled` | `boolean` | `true` | Enable or disable FlakeTriage for this repo. Set to `false` to pause all automatic reruns without uninstalling. |
| `budget_daily` | `integer` | `10` | Maximum number of automatic reruns per UTC day for this repo. Must be a non-negative integer (`0` = no reruns). The effective budget is `min(budget_daily, plan tier limit)`. |
| `allow_workflows` | `string[]` | `[]` | Workflow names to **always allow** for automatic rerun, even if they match a deny pattern. Highest priority — overrides all deny rules. |
| `deny_workflows` | `string[]` | `[]` | Additional workflow name patterns to deny. These are added on top of the built-in deny patterns. |

**Validation rules:**

- **Unknown keys** produce a warning in the Check Run output but do not block processing. Typos like `budgetDaily` (camelCase) will be flagged.
- **Invalid types** (e.g., `enabled: "yes"` instead of `enabled: true`) produce a warning, and the default value is used for that field.
- **Non-string items** in `allow_workflows` or `deny_workflows` arrays are silently ignored.
- A file that is empty or not a valid YAML mapping falls back to defaults with a config error noted in the Check Run.

**Built-in deny patterns** (always active, case-insensitive substring match):

| Pattern | Protects against |
|---------|-----------------|
| `release` | Release workflows |
| `deploy` | Deployment workflows |
| `publish` | Package publish workflows |
| `production` | Production-targeted workflows |

> **How matching works:** FlakeTriage performs a **case-insensitive substring match** against the workflow name (or file path). A workflow named "Deploy to staging" matches the `deploy` pattern. Use `allow_workflows` to override if needed.

### Example Configs

#### Minimal — Use Defaults

```yaml
# .github/flaketriage.yml
# No config needed — FlakeTriage uses safe defaults:
#   enabled: true
#   budget_daily: 10
#   allow_workflows: []
#   deny_workflows: []
```

Or simply don't create the file at all.

#### Standard — Custom Budget

```yaml
# .github/flaketriage.yml
enabled: true
budget_daily: 20
```

Allows up to 20 automatic reruns per day (or your plan tier limit, whichever is lower).

#### Monorepo / High-Volume

```yaml
# .github/flaketriage.yml
# Large repo with many CI workflows — higher budget + selective allows
enabled: true
budget_daily: 50
allow_workflows:
  - "unit-tests"
  - "integration-tests"
  - "e2e-smoke"
deny_workflows:
  - "nightly-benchmark"
  - "scheduled-cleanup"
```

This config:
- Allows up to 50 reruns/day (capped by plan tier).
- Explicitly allows `unit-tests`, `integration-tests`, and `e2e-smoke` — these will be rerun even if they happen to contain a built-in deny pattern.
- Explicitly denies `nightly-benchmark` and `scheduled-cleanup` on top of the built-in deny patterns.

#### Strict / Enterprise — Conservative

```yaml
# .github/flaketriage.yml
# Conservative setup for a regulated environment
enabled: true
budget_daily: 3
allow_workflows: []
deny_workflows:
  - "compliance"
  - "security-scan"
  - "migration"
```

This config:
- Keeps automatic reruns to a strict minimum (3/day).
- Denies additional workflow patterns on top of the built-in deny list.
- No allow overrides — all deny patterns are enforced strictly.

---

## How It Works

### Automatic Rerun Flow

```
1. A GitHub Actions workflow FAILS (conclusion: failure or timed_out)
         │
2. GitHub sends a webhook (workflow_run.completed) to FlakeTriage
         │
3. FlakeTriage validates the webhook signature (HMAC-SHA256)
         │
4. Checks: has FlakeTriage seen this delivery before? (dedup guard)
         │
5. Reads .github/flaketriage.yml from the commit (or uses defaults)
         │
6. POLICY CHECK: Is this workflow allowed?
   │  ├─ If workflow matches allow_workflows → YES
   │  ├─ If workflow matches deny pattern   → NO (Check Run: "policy_denied")
   │  └─ Otherwise                          → YES
   │
7. BUDGET CHECK: Are reruns available today?
   │  ├─ Effective budget = min(config budget_daily, plan tier limit)
   │  ├─ If used >= effective budget → NO (Check Run: "budget_exhausted")
   │  └─ Otherwise                   → YES
   │
8. RERUN GUARD: Has this exact run already been rerun?
   │  ├─ If already rerun → NO (Check Run: "already_rerun")
   │  └─ Otherwise        → YES
   │
9. TRIGGER: POST /repos/{owner}/{repo}/actions/runs/{run_id}/rerun-failed-jobs
         │
10. POST Check Run on the commit with the full decision evidence
         │
    ─── Later, when the rerun completes ───
         │
11. GitHub sends another webhook (workflow_run.completed, run_attempt > 1)
         │
12. FlakeTriage looks up the stored state → classifies outcome:
    ├─ Rerun PASSED  → "Suspected Flake"
    └─ Rerun FAILED  → "Likely Legit Failure"
         │
13. PATCH the Check Run with the outcome classification
```

### What Triggers a Rerun

A rerun is triggered when **all** of these conditions are true:

| Condition | Detail |
|-----------|--------|
| Workflow conclusion is `failure` or `timed_out` | Other conclusions (success, cancelled, skipped, etc.) are ignored |
| This is the first attempt (`run_attempt = 1`) | Rerun completions (attempt > 1) are used for classification, not re-rerunning |
| FlakeTriage is enabled for this repo | `enabled: true` in config (default) |
| Workflow passes the policy check | Not matching any deny pattern, or listed in `allow_workflows` |
| Daily budget has not been exhausted | `used_today < min(config budget_daily, tier limit)` |
| This run has not already been rerun | Single-rerun idempotency guard prevents double reruns |

### What Gets Skipped

| Skip Reason | Reason Code | What Happens |
|-------------|-------------|--------------|
| Workflow matches a deny pattern | `policy_denied` | Check Run created with deny evidence |
| Daily budget exhausted | `budget_exhausted` | Check Run created with budget details |
| Already rerun by FlakeTriage | `already_rerun` | Check Run created (dedup) |
| FlakeTriage disabled in config | `disabled_by_config` | Check Run created noting disabled |
| Webhook burst rate limited | `rate_limited_webhook` | Check Run created noting rate limit |
| Not a failure/timed_out conclusion | *(not processed)* | No action, no Check Run |

**Every decision produces a Check Run** — whether FlakeTriage takes action or not. This gives you full visibility into what happened and why.

### Outcome Classification

When a rerun completes, FlakeTriage updates the original Check Run with one of these outcomes:

| Rerun Result | Classification | Meaning |
|--------------|----------------|---------|
| Rerun **passed** | **Suspected Flake** | The original failure was likely transient (flaky test or infrastructure issue) |
| Rerun **failed** | **Likely Legit Failure** | The failure persisted, suggesting a real regression |

### Categorization Heuristic

FlakeTriage includes a metadata-only categorization heuristic (no log parsing):

| Category | Confidence | When Assigned |
|----------|-----------|---------------|
| `test_flake` | `med` | Rerun passed after original failure |
| `unknown` | `low` | Rerun failed, was skipped, or hasn't completed yet |
| `infra` | — | Reserved for future rules |
| `dependency` | — | Reserved for future rules |

> **Note:** This heuristic is intentionally conservative. It never claims certainty without log-level insight. The `test_flake` at `med` confidence is the strongest signal currently produced.

---

## Reading Check Run Results

After FlakeTriage processes a workflow run, look for a Check Run named **"FlakeTriage"** on the commit. Here's what each outcome looks like:

### Rerun Triggered

When FlakeTriage successfully triggers a rerun of failed jobs:

```markdown
## FlakeTriage — Automation Evidence

| Field | Value |
|---|---|
| Event | `workflow_run.completed` |
| Run ID | `12345678` |
| Conclusion | `failure` |
| Workflow | CI Tests |
| Reason Code | `rerun_triggered` |
| Reason | Rerun of failed jobs triggered successfully |

### Configuration
- **Source:** REPO
- **Path:** `.github/flaketriage.yml`
- **Enabled:** true
- **Budget Daily:** 10

### Policy Decision
- **Allowed:** true
- **Reason:** No deny patterns matched

### Budget
- **Daily Limit:** 10
- **Used Today:** 3
- **Remaining:** 6
- **Budget Decision:** Within budget

### Rerun Action
- **Triggered:** true
- **Outcome:** rerun_triggered
- **Summary:** Rerun of failed jobs triggered successfully
```

### Suspected Flake

When the rerun succeeds, the Check Run is updated:

```markdown
## FlakeTriage — Suspected Flake

| Field | Value |
|---|---|
| Classification | **Suspected Flake** |
| Run ID | `12345678` |
| Workflow | CI Tests |
| Head SHA | `abc1234` |

### Timeline
| Step | Detail |
|---|---|
| Original conclusion | `failure` |
| Rerun requested | 2026-03-08T10:00:00Z |
| Rerun attempt | 2 |
| Rerun conclusion | `success` |
| Rerun completed | 2026-03-08T10:05:30Z |

### Interpretation
The rerun **succeeded**, which suggests the original failure was caused by a
flaky test or transient infrastructure issue. The original failure is likely
**not a real regression**.

### Categorization (Heuristic)
- **Category:** test_flake
- **Confidence:** med
- **Rationale:** `rerun_passed`
```

### Likely Legit Failure

When the rerun also fails:

```markdown
## FlakeTriage — Likely Legit Failure

| Field | Value |
|---|---|
| Classification | **Likely Legit Failure** |
| Run ID | `12345678` |
| Workflow | CI Tests |
| Head SHA | `abc1234` |

### Timeline
| Step | Detail |
|---|---|
| Original conclusion | `failure` |
| Rerun requested | 2026-03-08T10:00:00Z |
| Rerun attempt | 2 |
| Rerun conclusion | `failure` |
| Rerun completed | 2026-03-08T10:06:00Z |

### Interpretation
The rerun **failed again**, which suggests the original failure is a
**real regression** — not a flake. Manual investigation is recommended.

### Categorization (Heuristic)
- **Category:** unknown
- **Confidence:** low
- **Rationale:** `rerun_failed`
```

### Skipped — Budget Exhausted

```markdown
## FlakeTriage — Automation Evidence

| Field | Value |
|---|---|
| Reason Code | `budget_exhausted` |
| Reason | Daily rerun budget exhausted (10/10 used) |

### Budget
- **Daily Limit:** 10
- **Used Today:** 10
- **Remaining:** 0
- **Budget Decision:** Budget exhausted for today
```

### Skipped — Policy Denied

```markdown
## FlakeTriage — Automation Evidence

| Field | Value |
|---|---|
| Reason Code | `policy_denied` |
| Reason | Workflow matches deny pattern |

### Policy Decision
- **Allowed:** false
- **Reason:** Denied by pattern match
- **Matched deny patterns:** `deploy`
```

### Skipped — Disabled by Config

```markdown
## FlakeTriage — Automation Evidence

| Field | Value |
|---|---|
| Reason Code | `disabled_by_config` |
| Reason | FlakeTriage is disabled for this repository |

### Configuration
- **Source:** REPO
- **Path:** `.github/flaketriage.yml`
- **Enabled:** false
```

---

## Manual Retry (Comment Command)

FlakeTriage supports manually triggering a rerun by posting a comment on any issue or pull request.

### Syntax

```
/flaketriage retry <run_id>
```

Where `<run_id>` is the numeric workflow run ID you want to rerun.

**Example:**

```
/flaketriage retry 12345678
```

> **Important:** The syntax is `/flaketriage retry <run_id>` — not `run=<run_id>`. The run ID is a positional argument separated by a space.

### How to Find the Run ID

1. Go to the **Actions** tab of your repository.
2. Click on the failed workflow run.
3. The run ID is in the URL: `github.com/{owner}/{repo}/actions/runs/{run_id}`

Alternatively, the FlakeTriage Check Run on your commit includes the `Run ID` in its evidence table.

### Permission Requirements

| Requirement | Detail |
|-------------|--------|
| **Minimum permission** | `write`, `admin`, or `maintain` on the repository |
| **Verification method** | GitHub Collaborator API (`GET /repos/{owner}/{repo}/collaborators/{username}/permission`) |
| **Cache** | Permission results cached in-memory for 120 seconds (configurable via `FLAKETRIAGE_PERMISSION_CACHE_TTL_SECONDS`) |
| **Fallback on API failure** | **Denied by default** — if the permission API is unreachable, access is denied |
| **Plan requirement** | `/flaketriage retry` is **not available on the Free plan** |

### Rate Limits

| Limit | Value |
|-------|-------|
| Window | 60 seconds (sliding window) |
| Max requests | 3 per repo × actor per window |
| Behavior when exceeded | Command silently ignored — **no reaction, no reply, no Check Run** |

> **Why silent?** The rate-limit check runs *before* token exchange (step 2 in the pipeline), so FlakeTriage cannot post reactions or replies when rate-limited. The same applies to budget-exhausted manual retries. The event is still logged server-side and counted by the `flaketriage_manual_retry_requests_total{outcome="denied",reason="rate_limit"}` Prometheus metric.

### What You Will See

When you post `/flaketriage retry <run_id>` and the command passes rate-limit and budget checks:

1. **👀 Reaction on your comment** — FlakeTriage acknowledges receipt (best-effort; silently skipped on 403).
2. **Check Run** — A Check Run named **"FlakeTriage — Manual Retry"** is posted on the commit with the outcome.
3. **Reply comment** — FlakeTriage posts a reply with ✅ (triggered) or ❌ (failed) plus 😕 reaction for errors (best-effort).

> **Silent outcomes:** If the command is **rate-limited**, **budget-exhausted**, on the **Free plan**, or has a **missing run ID**, FlakeTriage returns early *before* acquiring an API token. You will see **no reaction, no reply, and no Check Run**. Check server logs or Prometheus metrics to diagnose these cases.

### Manual Retry Outcomes

| Outcome | Check Run Created? | Description |
|---------|-------------------|-------------|
| `rerun_triggered` | ✅ (neutral) | Rerun of failed jobs triggered successfully |
| `rerun_failed` | ✅ (failure) | Rerun API call failed (e.g., 403, 404) |
| `permission_denied` | ✅ (failure) | Commenter lacks write/admin/maintain access |
| `permission_check_unavailable` | ✅ (failure) | Permission API unreachable — denied by default |
| `budget_exhausted` | — | Daily rerun budget exceeded |
| `rate_limited` | — | Too many commands in 60-second window |
| `auth_failed` | — | Installation token exchange failed |
| `missing_run_id` | — | No run ID provided in command |
| `manual_retry_denied_free_tier` | — | Free plan does not include manual retry |

---

## Plans & Limits

### Plan Tiers

| Plan | Reruns/Day | Manual Retry | Monthly | Annual |
|------|-----------|--------------|---------|--------|
| **Free** | Up to 5 | No | $0 | $0 |
| **Starter** | Up to 50 | Yes | $10 | $100 |
| **Team** | Up to 300 | Yes | $30 | $300 |
| **Business** | Up to 1000 | Yes | $99 | $990 |

All plans include: Check Run evidence, policy engine, config caching, structured logging, Prometheus metrics, impact/export/hotspot endpoints.

> For full pricing details be sure to see the Marketplace listing pricing section.

### How Budget Enforcement Works

The daily rerun budget is **usage-based** and resets at **midnight UTC**:

```
effective_budget = min(config budget_daily, plan tier limit)
```

**Example:** If your config says `budget_daily: 50` but you're on the Free plan (limit: 5), your effective budget is **5 reruns/day**.

- Each successful rerun trigger increments the daily counter for that repo.
- Budget is tracked **per repo per UTC day**.
- Max reruns per individual workflow run: **1** (single-rerun idempotency — FlakeTriage will never rerun the same run twice).

### What Happens When Budget Is Exhausted

- FlakeTriage still processes the webhook and creates a Check Run.
- The Check Run shows `budget_exhausted` with the used/total counts.
- No rerun is triggered.
- The budget resets at the next UTC midnight.

### Tier Resolution

FlakeTriage resolves your plan tier using this priority order:

1. **Database lookup** — If your installation has a plan persisted from a Marketplace purchase/change webhook, that tier is used.
2. **Environment variable** — Falls back to `FLAKETRIAGE_PLAN_TIER` env var (useful for self-hosted deployments).
3. **Default** — Falls back to `free`.

Per-repo overrides are available via `FLAKETRIAGE_TIER_OVERRIDES` (JSON map, environment variable). See the Marketplace listing pricing section for details.

---

## Impact & Time Saved

FlakeTriage provides delivery-impact visibility directly inside GitHub — no separate dashboard required.

### Check Run Mini-Card

Every Check Run created by FlakeTriage includes a **Time Saved** mini-card at the bottom. The mini-card shows:

| Field | Description |
|-------|-------------|
| **Auto rerun triggered** | Whether FlakeTriage triggered an automatic rerun |
| **Result** | Outcome: Suspected Flake, Likely Legit Failure, Skipped, or Rerun Pending |
| **Time-to-green** | Minutes between rerun request and successful completion (suspected flake only) |
| **Estimated developer time saved** | Estimated minutes saved by avoiding manual investigation |

**Initial triage (rerun pending):**
- Shows "Rerun Pending" result — time savings appear after rerun completes.

**Outcome update (rerun completed):**
- **Suspected Flake:** Shows time-to-green and estimated dev time saved.
- **Likely Legit Failure:** No time savings — manual investigation may be needed.

### Configuring Time Estimates

All estimates use configurable values from `.github/flaketriage.yml`:

```yaml
roi:
  estimated_manual_rerun_minutes: 3     # How long a manual rerun takes (default: 3)
  estimated_context_switch_minutes: 2   # Context-switch overhead (default: 2)
```

**Formula:** `estimated_dev_time_saved = 1 × (manual_rerun + context_switch)` per suspected flake.

FlakeTriage shows time savings only — no dollar or cost estimates.

### Weekly Impact Digest

FlakeTriage can post a **weekly impact digest** as a GitHub Issue in your repository. The digest summarizes:

- Suspected flakes detected
- Total reruns triggered and classified
- Estimated developer time saved
- Skip breakdown by reason (budget, policy, etc.)
- Budget exhaustion events

The digest uses an **idempotent issue** — the same issue is updated each week rather than creating new ones. Look for issues with the `flaketriage-digest` label.

### `/impact` API Endpoint

The `/impact` endpoint returns a comprehensive JSON impact summary. See [API / Telemetry Endpoints](#api--telemetry-endpoints) for usage and examples. The `/roi` endpoint is kept as a deprecated alias.

> **Important:** All "Estimated" values are based on configured assumptions, not certainties. FlakeTriage uses the word "Estimated" wherever counterfactual savings are shown.

---

## API / Telemetry Endpoints

FlakeTriage exposes five telemetry endpoints for monitoring and analytics. These are available on the running FlakeTriage server (not on GitHub).

### Authentication

All telemetry endpoints require a **bearer token** set via the `FLAKETRIAGE_METRICS_TOKEN` environment variable.

| Scenario | Behavior |
|----------|----------|
| Token **not configured** | All endpoints return **403 Forbidden** (`telemetry_disabled`) |
| Token configured, **no Authorization header** | Returns **401 Unauthorized** |
| Token configured, **wrong token** | Returns **401 Unauthorized** |
| Token configured, **correct token** | Returns data (200) |

> **Security:** Token comparison uses constant-time `timingSafeEqual` to prevent timing attacks.

### Endpoints Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `GET /metrics` | GET | Prometheus-format metrics (all `flaketriage_*` counters and gauges) |
| `GET /impact` | GET | JSON impact summary (reruns attempted, successful, skipped, estimated time saved) |
| `GET /roi` | GET | Deprecated alias for `/impact` — returns identical data |
| `GET /export` | GET | Export impact + outcome data. Query: `?format=json` (default) or `?format=csv` |
| `GET /hotspots` | GET | Top N hotspot entries by count. Query: `?limit=N` (default 10, max 50) |

**Additional operational endpoints (no auth required):**

| Endpoint | Method | Description |
|----------|--------|-------------|
| `GET /healthz` | GET | Liveness probe — always 200 (includes `degraded` flag) |
| `GET /readyz` | GET | Readiness probe — 200 if healthy, 503 if storage degraded |

### Request Examples (PowerShell)

```powershell
# Set your bearer token
$headers = @{ Authorization = "Bearer $env:FLAKETRIAGE_METRICS_TOKEN" }
$baseUrl = "https://your-flaketriage-instance.fly.dev"

# Prometheus metrics
Invoke-RestMethod -Uri "$baseUrl/metrics" -Headers $headers

# Impact summary
Invoke-RestMethod -Uri "$baseUrl/impact" -Headers $headers

# Export as JSON
Invoke-RestMethod -Uri "$baseUrl/export?format=json" -Headers $headers

# Export as CSV
Invoke-WebRequest -Uri "$baseUrl/export?format=csv" -Headers $headers | 
  Select-Object -ExpandProperty Content

# Hotspots (top 10)
Invoke-RestMethod -Uri "$baseUrl/hotspots?limit=10" -Headers $headers

# Health check (no auth required)
Invoke-RestMethod -Uri "$baseUrl/healthz"
```

**Example `/impact` response:**

```json
{
  "suspected_flake_count": 31,
  "total_reruns_attempted": 47,
  "reruns_classified": 31,
  "reruns_pending_or_failed": 16,
  "reruns_skipped": {
    "budget": 5,
    "policy": 12,
    "already_rerun": 2
  },
  "reruns_skipped_total": 19,
  "estimated_manual_minutes_avoided": 155,
  "ci_minutes_added": 0,
  "budget_exhaustion_count": 5,
  "impact_config": {
    "estimated_manual_rerun_minutes": 3,
    "estimated_context_switch_minutes": 2
  },
  "formulas": {
    "estimated_manual_minutes_avoided": "suspected_flake_count(31) × (estimated_manual_rerun_minutes(3) + estimated_context_switch_minutes(2)) = 155"
  },
  "window": "since_start",
  "disclaimer": "All saved-time figures are estimates based on configured assumptions, not certainties."
}
```

**Example `/hotspots` response:**

```json
{
  "hotspots": [
    { "category": "skip_reason", "label": "policy_denied", "count": 12 },
    { "category": "skip_reason", "label": "budget_exhausted", "count": 5 },
    { "category": "tier_attempt", "label": "free/triggered", "count": 47 }
  ],
  "limit": 10,
  "total_entries": 3,
  "window": "since_start"
}
```

---

## Troubleshooting

### Installed but not triggering

| Check | How |
|-------|-----|
| Is the App installed on the repo? | Go to **Settings → Integrations → GitHub Apps** — you should see FlakeTriage listed |
| Is the workflow conclusion `failure` or `timed_out`? | Other conclusions (`success`, `cancelled`, `skipped`) are intentionally ignored |
| Is FlakeTriage enabled? | Check `.github/flaketriage.yml` — if `enabled: false`, FlakeTriage won't act |
| Is the workflow denied by policy? | Check the Check Run for `policy_denied`. Add it to `allow_workflows` to override |
| Is the budget exhausted? | Check the Check Run for `budget_exhausted`. Wait for UTC midnight reset or increase `budget_daily` |
| Is this a first attempt? | FlakeTriage only reruns on `run_attempt = 1`. Rerun completions (attempt > 1) are used for classification |

### Webhooks failing / delivery errors

1. Go to your GitHub App settings → **Advanced** → **Recent Deliveries**.
2. Look for the webhook delivery corresponding to the workflow run.
3. Check the **response code** returned by FlakeTriage:

| Response Code | Meaning | What to Check |
|---|---|---|
| **202** | Accepted — webhook queued for processing | Normal operation. Check Runs will appear shortly. |
| **401** | Signature verification failed | Webhook secret mismatch between GitHub App settings and `WEBHOOK_SECRET` env var. |
| **400** | Invalid JSON payload | Unusual — may indicate a proxy or middleware corrupting the request body. |
| **503** | Server shutting down | Transient — the server is draining during a deployment. Redeliver the webhook. |

4. Try **Redelivery** from the webhook delivery page if it was a transient issue.
5. On the server side, check the application logs (Fly.io: `fly logs --app <app-name>`).

> **Note:** Signature failures return **401** (not 400). If you see 401 responses, verify that the `WEBHOOK_SECRET` environment variable matches the secret configured in your GitHub App settings.

### Budget exhausted

- The Check Run will show the exact used/total counts.
- Budget resets at **midnight UTC** each day.
- You can increase `budget_daily` in `.github/flaketriage.yml` (still capped by your plan tier limit).
- Consider upgrading your plan for a higher tier limit.

### Manual retry denied

| Outcome | Meaning | Fix |
|---------|---------|-----|
| `permission_denied` | You don't have write access to the repo | Ask a repo admin to grant write/maintain/admin access |
| `permission_check_unavailable` | GitHub API is unreachable | Wait and retry after GitHub status recovers |
| `manual_retry_denied_free_tier` | Free plan does not include manual retry | Upgrade to Starter or above |
| `budget_exhausted` | Daily rerun budget exceeded | Wait for UTC midnight reset |
| `rate_limited` | Too many commands in 60 seconds | Wait 60 seconds and retry |
| `missing_run_id` | No run ID in command | Use `/flaketriage retry <run_id>` with the numeric run ID |

### Tier missing → free fallback

If FlakeTriage cannot determine your plan tier (no database record, no environment variable), it falls back to the **free** tier. Signs this is happening:

- The `flaketriage_installation_tier_missing_total` Prometheus metric is incrementing.
- Check Runs show a budget limit of 5 (the free tier default).
- Manual retry commands return `manual_retry_denied_free_tier`.

**Fix:** Ensure your Marketplace subscription is active, or set `FLAKETRIAGE_PLAN_TIER` environment variable explicitly.

### Where to look for diagnostics

| Source | What it shows |
|--------|---------------|
| **Check Run on the commit** | Full decision evidence (config, policy, budget, outcome) |
| **GitHub webhook deliveries** | App settings → Advanced → Recent Deliveries (request/response payloads) |
| **Server logs** | Fly.io: `fly logs --app <app-name>` — structured JSON logs with event types |
| **`/metrics` endpoint** | Prometheus counters: webhook totals, reruns triggered/skipped, queue depth |
| **`/healthz` endpoint** | Liveness + storage mode (sqlite vs memory) + degraded flag |

---

## FAQ

**Q: Does FlakeTriage read my source code or build logs?**
A: No. FlakeTriage operates on **workflow metadata only** — event payloads, conclusion status, workflow names, run IDs. It never fetches source code, build logs, or test output. See [docs/PRIVACY_POLICY.md](PRIVACY_POLICY.md).

**Q: What happens if I don't create a `.github/flaketriage.yml` file?**
A: FlakeTriage works with safe defaults: enabled, 10 reruns/day budget (capped by your plan tier), built-in deny patterns for `release`/`deploy`/`publish`/`production` workflows.

**Q: Can FlakeTriage rerun the same workflow run multiple times?**
A: No. FlakeTriage has a single-rerun idempotency guard. Each workflow run can be rerun at most once. This is enforced regardless of configuration.

**Q: Does FlakeTriage rerun the entire workflow?**
A: No. FlakeTriage uses `rerun-failed-jobs`, which reruns **only the failed jobs** within the workflow — not the entire run. This saves CI minutes and is faster.

**Q: What if my workflow fails during the rerun too?**
A: FlakeTriage classifies it as **"Likely Legit Failure"** and updates the Check Run accordingly. It does not attempt a second rerun.

**Q: Are deploy/release workflows automatically rerun?**
A: No. FlakeTriage has built-in deny patterns for `release`, `deploy`, `publish`, and `production`. These workflows are never automatically rerun unless you explicitly add them to `allow_workflows` in your config.

**Q: Does the budget count manual retries?**
A: Yes. Both automatic reruns and manual retries (`/flaketriage retry`) count toward the daily budget for that repo.

**Q: What happens to my data if FlakeTriage's storage becomes unavailable?**
A: FlakeTriage degrades gracefully to in-memory mode. Delivery deduplication may miss duplicates, and budget counts may reset, but the service continues to function. The `/healthz` endpoint reports `degraded: true` in this state.

**Q: Can I use FlakeTriage on private repositories?**
A: Yes. FlakeTriage works on both public and private repositories. It only requires the permissions listed in the [Required Permissions](#required-permissions) section.

**Q: How do I completely stop FlakeTriage for one repo without uninstalling?**
A: Set `enabled: false` in `.github/flaketriage.yml`:
```yaml
enabled: false
```

---

## Upgrade / Downgrade Notes

### Upgrading (Marketplace)

- Go to [GitHub Marketplace → FlakeTriage](https://github.com/marketplace/flaketriage) and select a higher plan.
- The new tier takes effect **immediately** — FlakeTriage processes the `marketplace_purchase` (action: `changed`) webhook and updates the installation's tier in the database.
- Your daily budget limit increases to the new tier's limit right away.

### Downgrading

- Downgrades initiated through GitHub Marketplace may trigger a `pending_change` event.
- **Pending changes** are logged but do **not** change your tier immediately — the change takes effect at the end of the current billing cycle (as determined by GitHub Marketplace).
- When the billing cycle ends, GitHub sends a `changed` event with the new plan, and FlakeTriage updates the tier.
- After downgrade, if your `budget_daily` config exceeds the new tier's limit, the effective budget is automatically capped: `effective = min(config budget_daily, new tier limit)`.

### Cancelling

- Cancellation triggers a `cancelled` event.
- FlakeTriage removes the installation's plan from the database.
- The tier reverts to **free** (5 reruns/day, no manual retry).
- FlakeTriage continues to function on the free tier — it does not uninstall itself.

### Self-Hosted / Environment Variable Configuration

If you're not using GitHub Marketplace (e.g., self-hosted deployment), you can change tiers by updating the `FLAKETRIAGE_PLAN_TIER` environment variable and restarting the server:

```powershell
fly secrets set FLAKETRIAGE_PLAN_TIER="team" --app your-flaketriage-app
```

Valid values: `free`, `starter`, `team`, `business`.

---

*For privacy and data policy, see [Privacy & Data Policy](privacy-policy). For source code and architecture details, see the [FlakeTriage repository](https://github.com/marketplace/flaketriage).*
