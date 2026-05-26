# Asana

**Version:** 0.1.0
**Backend:** HTTP
**Tables:** 4
**Base URL:** `https://app.asana.com/api/1.0`

Query workspaces, projects, tasks, and sections from Asana via the Asana REST API v1.0.

## Authentication

Asana supports two authentication methods: OAuth (recommended for interactive use) and
Personal Access Tokens (PAT) for scripts and CI pipelines.

### Option 1 — OAuth (recommended)

1. Go to [https://app.asana.com/0/my-apps](https://app.asana.com/0/my-apps) and create a new app.
2. Under **Redirect URIs**, add exactly: `http://127.0.0.1:53682/oauth/callback`
3. Copy the **Client ID** into `ASANA_CLIENT_ID` and the **Client Secret** into `ASANA_CLIENT_SECRET`.
4. Run `coral source add --file sources/community/asana/manifest.yaml` and click **Sign in with Asana** to complete the browser-based flow.

> **Scope:** The `default` scope grants read access to all workspaces, projects, tasks, and
> sections visible to the authorized user.

### Option 2 — Personal Access Token

1. Go to [https://app.asana.com/0/my-apps](https://app.asana.com/0/my-apps) → **Create new token**.
2. Give the token a name and copy it.
3. Run `coral source add --file sources/community/asana/manifest.yaml` and paste the token when prompted for `ASANA_TOKEN`.

```bash
coral source add --file sources/community/asana/manifest.yaml
```

API docs: [https://developers.asana.com/reference/rest-api-reference](https://developers.asana.com/reference/rest-api-reference)

## Tables

| Table | Description | Required filters | Optional filters |
|---|---|---|---|
| `workspaces` | All workspaces visible to the authenticated user | — | — |
| `projects` | Projects across all workspaces or within one workspace | — | `workspace_gid` |
| `tasks` | Tasks within a specific project | `project_gid` | — |
| `sections` | Sections (columns / swim-lanes) within a specific project | `project_gid` | — |

### Key design notes

- **Start with `workspaces`.** It requires no filters and returns the GIDs you'll use everywhere else.
- **`projects` is optionally scoped.** Without `workspace_gid` the API returns projects across all
  workspaces visible to the token — useful for a global view. Supply `workspace_gid` to limit scope.
- **`tasks` and `sections` require `project_gid`.** Asana's API models these as project-scoped
  resources. Get the GID from `asana.projects.gid` first.
- **Typical flow:** `workspaces` → `projects` (filter by `workspace_gid`) → `tasks` or `sections`
  (filter by `project_gid`).
- **`resource_subtype` on tasks** distinguishes milestones (`milestone`), approvals (`approval`),
  and regular tasks (`default_task`). Filter locally after fetching.
- **`due_on` vs `due_at` on tasks.** Asana sets one or the other, never both.
  `due_on` is a date string (`YYYY-MM-DD`); `due_at` is a full timestamp for time-specific deadlines.
- **Rate limits.** The Asana free tier allows 150 requests per minute per user. All tables fetch
  100 items per page to minimize round-trips.

```text
workspaces   → all workspaces (entry point, no filter needed)
projects     → projects in a workspace  (optional workspace_gid filter)
tasks        → tasks in a project       (requires project_gid)
sections     → sections in a project    (requires project_gid)
```

### `projects` optional filter

| Filter | Description |
|---|---|
| `workspace_gid` | Limit projects to a specific workspace. Get the GID from `asana.workspaces`. |

### `tasks` required filter

| Filter | Description |
|---|---|
| `project_gid` | The project whose tasks to fetch. Get the GID from `asana.projects`. |

### `sections` required filter

| Filter | Description |
|---|---|
| `project_gid` | The project whose sections to fetch. Get the GID from `asana.projects`. |

## Quick start

```bash
# Step 1 — list all workspaces
coral sql "SELECT gid, name, is_organization FROM asana.workspaces"

# Step 2 — list projects in a workspace
coral sql "
  SELECT gid, name, archived, due_on, status_type
  FROM asana.projects
  WHERE workspace_gid = 'your-workspace-gid'
  LIMIT 20
"

# Step 3 — list open tasks in a project
coral sql "
  SELECT gid, name, due_on, assignee_name
  FROM asana.tasks
  WHERE project_gid = 'your-project-gid'
    AND completed = false
  ORDER BY due_on ASC
  LIMIT 50
"

# Step 4 — list sections in a project
coral sql "
  SELECT gid, name, created_at
  FROM asana.sections
  WHERE project_gid = 'your-project-gid'
"
```

## Example queries

### List all workspaces

```sql
SELECT
  gid,
  name,
  is_organization,
  email_domains
FROM asana.workspaces;
```

### List projects in a workspace

```sql
SELECT
  gid,
  name,
  archived,
  color,
  due_on,
  status_type,
  status_title,
  owner_name,
  created_at
FROM asana.projects
WHERE workspace_gid = 'your-workspace-gid'
ORDER BY name;
```

### List open tasks in a project sorted by due date

```sql
SELECT
  gid,
  name,
  due_on,
  due_at,
  assignee_name,
  resource_subtype
FROM asana.tasks
WHERE project_gid = 'your-project-gid'
  AND completed = false
ORDER BY due_on ASC NULLS LAST
LIMIT 100;
```

### Find overdue incomplete tasks

```sql
SELECT
  gid,
  name,
  due_on,
  assignee_name,
  modified_at
FROM asana.tasks
WHERE project_gid = 'your-project-gid'
  AND completed = false
  AND due_on < current_date
ORDER BY due_on ASC;
```

### List sections in a project

```sql
SELECT
  gid,
  name,
  created_at
FROM asana.sections
WHERE project_gid = 'your-project-gid'
ORDER BY created_at ASC;
```
