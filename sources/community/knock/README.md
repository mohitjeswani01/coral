# Knock

**Version:** 0.1.0
**Backend:** HTTP
**Tables:** 4
**Base URL:** `https://api.knock.app`

Query notification messages, users, tenants, and schedules from Knock — the notification infrastructure platform for in-app, email, SMS, push, and chat notifications.

## Authentication

Requires a `KNOCK_API_KEY`. Find it in the Knock dashboard:
**Platform → API Keys → copy secret key**

Secret keys start with `sk_test_` (test environment) or `sk_live_` (production environment). Use the key matching the environment you want to query.

> **Environment note:** Knock API keys are environment-scoped. A `sk_test_` key only returns data from your test environment; a `sk_live_` key only returns production data.

```bash
coral source add --file sources/community/knock/manifest.yaml
```

You will be prompted to enter your API key interactively.

API docs: https://docs.knock.app/api-reference/overview

## Tables

| Table | Description | Required filters | Optional filters |
|---|---|---|---|
| `messages` | Notification delivery records with status tracking | — | `channel_id`, `workflow`, `tenant`, `status` |
| `users` | Notification recipients/subscribers | — | — |
| `tenants` | Multi-tenant organization records | — | `tenant_id` |
| `schedules` | Scheduled workflow notification runs | — | `workflow`, `tenant` |

### Key design notes

- **All tables use cursor pagination.** Knock uses `page_info.after` cursors
  with a max page size of 50. The source handles pagination automatically.
- **`messages` is the core table.** It tracks delivery status (`queued`,
  `sent`, `delivered`, `undelivered`, `not_sent`) and engagement timestamps
  (`read_at`, `seen_at`, `archived_at`).
- **`users` are flat objects.** Standard fields (name, email, phone_number)
  are columns; custom properties synced to Knock are not exposed as fixed
  columns.
- **`tenants` enable multi-tenant scoping.** Join `messages.tenant` with
  `tenants.id` to enrich notification data with tenant metadata.

```text
messages   → notification delivery records with engagement tracking
users      → recipients/subscribers with contact details
tenants    → multi-tenant organization records
schedules  → scheduled workflow notification runs
```

### Not included in v1

- **`message_events`, `message_activities`, `delivery_logs`** — require a
  `message_id` path parameter (sub-resource pattern). Deferred to v2.
- **`workflows`** — workflow definitions live on the Knock Management API
  (`control.knock.app`) which uses separate Service Token authentication,
  not the standard API secret key.
- **`objects`** — require a `collection` path parameter. Deferred to v2.

### messages filter values

| Filter | Description |
|---|---|
| `channel_id` | Filter by channel ID (e.g. a specific email or push channel) |
| `workflow` | Filter by workflow key (e.g. `welcome-email`) |
| `tenant` | Filter by tenant ID |
| `status` | Filter by delivery status (`queued`, `sent`, `delivered`, `undelivered`, `not_sent`) |

### schedules filter values

| Filter | Description |
|---|---|
| `workflow` | Filter by workflow key |
| `tenant` | Filter by tenant ID |

## Quick start

```bash
# Step 1 — list recent notification messages
coral sql "
  SELECT id, status, workflow, tenant, inserted_at
  FROM knock.messages
  LIMIT 20
"

# Step 2 — list all users
coral sql "
  SELECT id, name, email, timezone
  FROM knock.users
  LIMIT 20
"

# Step 3 — list all tenants
coral sql "
  SELECT id, name, created_at
  FROM knock.tenants
"

# Step 4 — list schedules for a specific workflow
coral sql "
  SELECT id, workflow, tenant, inserted_at
  FROM knock.schedules
  WHERE workflow = 'welcome-series'
"
```

## Example queries

### List recent messages with status

```sql
SELECT
  id,
  status,
  workflow,
  channel_id,
  tenant,
  inserted_at
FROM knock.messages
ORDER BY inserted_at DESC
LIMIT 50;
```

### Find undelivered messages by workflow

```sql
SELECT
  id,
  workflow,
  channel_id,
  tenant,
  status,
  inserted_at
FROM knock.messages
WHERE status = 'undelivered'
LIMIT 50;
```

### List all users in the workspace

```sql
SELECT
  id,
  name,
  email,
  phone_number,
  timezone,
  created_at
FROM knock.users
LIMIT 100;
```

### List schedules for a specific workflow

```sql
SELECT
  id,
  workflow,
  tenant,
  inserted_at,
  updated_at
FROM knock.schedules
WHERE workflow = 'digest-daily'
LIMIT 50;
```

### Join messages with users on recipient

```sql
SELECT
  m.id AS message_id,
  m.status,
  m.workflow,
  m.inserted_at,
  u.name AS recipient_name,
  u.email AS recipient_email
FROM knock.messages m
JOIN knock.users u ON m.tenant = u.id
LIMIT 50;
```
