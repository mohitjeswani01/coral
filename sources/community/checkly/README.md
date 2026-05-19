# Checkly

Query synthetic monitoring checks, check results, alert channels, and
dashboards from [Checkly](https://checklyhq.com/) — the synthetic monitoring
platform for API and browser checks.

## Authentication

Checkly requires **two credentials** for every API request:

1. **API Key** — authenticates the request
2. **Account ID** — scopes the request to your account

### Step 1 — Get your API key

- **Personal key**: Checkly dashboard → User Settings → [API Keys](https://app.checklyhq.com/settings/user/api-keys) → Create API key
- **Service key** (recommended for CI/CD): Checkly dashboard → Account Settings → API Keys → Create API key

A read-only key is sufficient for all tables in this source.

### Step 2 — Get your Account ID

Checkly dashboard → Account Settings → General → **Account ID** (shown at the top of the page).

### Step 3 — Add the source

```sh
export CHECKLY_API_KEY="cu_..."
export CHECKLY_ACCOUNT_ID="12345678"
coral source add --file sources/community/checkly/manifest.yaml
```

See the [Checkly authentication docs](https://developers.checklyhq.com/docs/authentication) for details.

## Tables

| Table | Description | Required filters |
|---|---|---|
| `checkly.checks` | All API and browser checks in the account — inventory, type, frequency, locations | — |
| `checkly.check_results` | Recent run results for a specific check — pass/fail, response time, run location | `check_id` (required) |
| `checkly.alert_channels` | Notification channels — Slack, Email, PagerDuty, Webhook, etc. — and their routing config | — |
| `checkly.dashboards` | Public and private status page dashboards with domain and tag configuration | — |

### `checkly.checks`

Lists all synthetic monitoring checks in the account, newest first. No filter required — the `X-Checkly-Account` header scopes results automatically.

Key columns:

| Column | Type | Description |
|---|---|---|
| `id` | `Utf8` | Check UUID — use this as `check_id` in `check_results` |
| `name` | `Utf8` | Check display name |
| `check_type` | `Utf8` | `API` · `BROWSER` · `HEARTBEAT` · `DNS` · `TCP` |
| `frequency` | `Int64` | Run interval in minutes |
| `activated` | `Boolean` | `false` = check is paused |
| `muted` | `Boolean` | `true` = alerts suppressed |
| `locations` | `Json` | Array of AWS region strings |
| `tags` | `Json` | Array of tag strings |
| `degraded_response_time` | `Int64` | Degraded threshold in ms |
| `max_response_time` | `Int64` | Failure threshold in ms |
| `created_at` | `Timestamp` | Check creation time |

### `checkly.check_results`

Recent run results for one specific check. **`check_id` is required.** Get check IDs from `checkly.checks`. Raw results are retained for **30 days**.

| Filter | Required | Description |
|---|---|---|
| `check_id` | ✅ Yes | UUID of the check to fetch results for |
| `result_type` | No | `FINAL` (completed runs) or `ATTEMPT` (retry attempts) |
| `has_failures` | No | `true` to return only failing runs |
| `from` | No | UNIX millisecond timestamp — lower time bound |
| `to` | No | UNIX millisecond timestamp — upper time bound |

Key columns:

| Column | Type | Description |
|---|---|---|
| `id` | `Utf8` | Result ID |
| `check_run_id` | `Int64` | Monotonic run sequence number |
| `result_type` | `Utf8` | `FINAL` or `ATTEMPT` |
| `has_failures` | `Boolean` | `true` if assertions failed or timeout occurred |
| `has_errors` | `Boolean` | `true` if a Checkly platform error occurred |
| `run_location` | `Utf8` | AWS region where the check ran |
| `started_at` | `Timestamp` | Run start time |
| `stopped_at` | `Timestamp` | Run completion time |
| `response_time` | `Int64` | Execution time in ms |

### `checkly.alert_channels`

All notification channels in the account. `config` is a polymorphic JSON object — its shape depends on `type`.

| `type` | `config` fields |
|---|---|
| `EMAIL` | `address` |
| `SLACK` | `channel`, `url` |
| `WEBHOOK` | `method`, `url`, `headers`, `template` |
| `SMS` | `number` |
| `PAGERDUTY` | `service_key`, `account`, `service_name` |
| `OPSGENIE` | `api_key`, `name`, `priority`, `region` |
| `CALL` | `name`, `number` |

### `checkly.dashboards`

Status page dashboards for the account. `tags` controls which checks appear on each dashboard — only checks with matching tags are displayed.

## Example queries

### Inventory all active checks

```sql
SELECT
  id,
  name,
  check_type,
  frequency,
  locations,
  tags
FROM checkly.checks
WHERE activated = true
ORDER BY name;
```

### Find paused or muted checks

```sql
SELECT
  id,
  name,
  check_type,
  activated,
  muted
FROM checkly.checks
WHERE activated = false OR muted = true;
```

### Get recent failing results for a specific check

```sql
SELECT
  id,
  has_failures,
  run_location,
  response_time,
  started_at,
  stopped_at
FROM checkly.check_results
WHERE check_id = '<your-check-id>'
  AND has_failures = 'true'
  AND result_type = 'FINAL'
LIMIT 50;
```

### Audit alert channel routing

```sql
SELECT
  id,
  type,
  send_failure,
  send_recovery,
  send_degraded,
  ssl_expiry,
  config
FROM checkly.alert_channels
ORDER BY type;
```

### List public status page dashboards

```sql
SELECT
  id,
  custom_url,
  custom_domain,
  is_private,
  tags
FROM checkly.dashboards
WHERE is_private = false;
```

## Auth

This source uses two headers on every request:

| Header | Value |
|---|---|
| `Authorization` | `Bearer <CHECKLY_API_KEY>` |
| `X-Checkly-Account` | `<CHECKLY_ACCOUNT_ID>` |

Both credentials are required. Requests without the `X-Checkly-Account` header will be rejected by the API regardless of the API key's validity.

See the [Checkly API reference](https://developers.checklyhq.com/reference) for full documentation.
