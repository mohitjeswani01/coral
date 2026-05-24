# Google Tasks Community Source

Exposes user task lists and individual task items as standard SQL tables using the official Google Tasks REST API v1.

## Authentication

This source uses **OAuth 2.0 authorization-code flow with PKCE**. You need a Google Cloud project with the Google Tasks API enabled.

### Required inputs

| Input | Kind | Description |
|---|---|---|
| `GOOGLE_CLIENT_ID` | variable | OAuth 2.0 Client ID from Google Cloud Console |
| `GOOGLE_CLIENT_SECRET` | secret | OAuth 2.0 Client Secret from Google Cloud Console |
| `GOOGLE_TASKS_TOKEN` | secret | OAuth access token — Coral opens a browser flow automatically |

### Setup steps

1. Go to [Google Cloud Console](https://console.cloud.google.com/) → **APIs & Services** → **Enabled APIs** and enable the **Google Tasks API**.
2. Go to **APIs & Services** → **Credentials** → **Create Credentials** → **OAuth 2.0 Client ID**.
3. Choose **Desktop app** as the application type.
4. Copy the **Client ID** and **Client Secret**.
5. Add the source — Coral will open a browser window to complete the OAuth consent flow:

```sh
coral source add --file sources/community/google_tasks/manifest.yaml
```

Coral requests the `tasks.readonly` scope. The authorization URL includes `access_type=offline&prompt=consent` so Google issues a refresh token on first consent, keeping the token alive across sessions.

### Required scope

| Scope | Tables |
|---|---|
| `https://www.googleapis.com/auth/tasks.readonly` | `google_tasks.task_lists`, `google_tasks.tasks` |

## Tables

### `google_tasks.task_lists`

Lists metadata for all task lists owned by or shared with the authenticated user. Entry-point table; no required filters.

Filtering by `id` routes directly to the single-object endpoint (`GET /tasks/v1/users/@me/lists/{id}`) and returns exactly one row.

### `google_tasks.tasks`

Lists individual task items from a specific task list.

**`tasklist_id` is required** — obtain it from `google_tasks.task_lists`.

Filtering by both `tasklist_id` and `id` routes directly to the single-object endpoint (`GET /tasks/v1/lists/{tasklist}/tasks/{id}`) and returns exactly one row.

**API-level pushdown filters** (sent as query parameters to Google's API):

| Filter | Type | Default | Description |
|---|---|---|---|
| `show_completed` | Boolean | false | Include completed tasks |
| `show_hidden` | Boolean | false | Include hidden tasks (needed for tasks completed in Google apps) |
| `show_deleted` | Boolean | false | Include deleted tasks |
| `show_assigned` | Boolean | false | Include tasks assigned from Google Docs or Google Chat |
| `due_min` | String | — | Lower bound for due date (RFC 3339) |
| `due_max` | String | — | Upper bound for due date (RFC 3339) |
| `completed_min` | String | — | Lower bound for completion date (RFC 3339) |
| `completed_max` | String | — | Upper bound for completion date (RFC 3339) |
| `updated_min` | String | — | Lower bound for last modification date (RFC 3339) |

## Rate limits

Google Tasks enforces a courtesy quota of **50,000 queries per day** per project (see [Google Tasks usage limits](https://developers.google.com/workspace/tasks/limits)). Both tables default to `fetch_limit_default: 100` rows per query. Use `LIMIT` and date-range filters to keep individual queries bounded on large task lists.

## Example queries

### 1. Discover your task list IDs

```sql
SELECT id, title, updated FROM google_tasks.task_lists;
```

### 2. Query incomplete tasks from a specific list

```sql
SELECT id, title, due
FROM google_tasks.tasks
WHERE tasklist_id = 'your_list_id_here'
  AND status = 'needsAction';
```

### 3. Find completed tasks in a single list (with metadata join)

```sql
SELECT
    l.title AS list_name,
    t.title AS task_name,
    t.completed
FROM google_tasks.tasks t
JOIN google_tasks.task_lists l ON t.tasklist_id = l.id
WHERE t.tasklist_id = 'your_list_id_here'
  AND t.status = 'completed';
```

### 4. Include hidden and completed tasks via API pushdown

```sql
SELECT id, title, completed, hidden
FROM google_tasks.tasks
WHERE tasklist_id = 'your_list_id_here'
  AND show_completed = true
  AND show_hidden = true;
```

### 5. Include tasks assigned from Google Docs or Chat

```sql
SELECT id, title, status, web_view_link
FROM google_tasks.tasks
WHERE tasklist_id = 'your_list_id_here'
  AND show_assigned = true;
```

### 6. Filter tasks by due date range

```sql
SELECT title, due
FROM google_tasks.tasks
WHERE tasklist_id = 'your_list_id_here'
  AND due_min = '2024-01-01T00:00:00Z'
  AND due_max = '2024-01-31T23:59:59Z';
```

### 7. Single-item point queries (route directly to GET endpoints)

```sql
-- Routes to GET /tasks/v1/users/@me/lists/{id} — returns one row
SELECT * FROM google_tasks.task_lists WHERE id = 'your_list_id_here';

-- Routes to GET /tasks/v1/lists/{tasklist}/tasks/{id} — returns one row
SELECT * FROM google_tasks.tasks
WHERE tasklist_id = 'your_list_id_here'
  AND id = 'your_task_id_here';
```

## API reference

- [Google Tasks REST API v1](https://developers.google.com/tasks/reference/rest/v1)
- [tasks.tasklists.list](https://developers.google.com/workspace/tasks/reference/rest/v1/tasklists/list)
- [tasks.tasklists.get](https://developers.google.com/workspace/tasks/reference/rest/v1/tasklists/get)
- [tasks.tasks.list](https://developers.google.com/workspace/tasks/reference/rest/v1/tasks/list)
- [tasks.tasks.get](https://developers.google.com/workspace/tasks/reference/rest/v1/tasks/get)
- [OAuth and authorization](https://developers.google.com/workspace/tasks/auth)
- [Usage limits](https://developers.google.com/workspace/tasks/limits)
