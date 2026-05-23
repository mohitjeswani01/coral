# Google Tasks Community Source

Exposes user task lists and individual task items as standard SQL tables using the official Google Tasks REST API v1 via a local-first SQL runtime.

## Authentication

This source requires setting up an OAuth 2.0 Desktop Application client inside your personal Google Cloud Console.

* **GOOGLE_CLIENT_ID**: Generated Desktop application Client ID.
* **GOOGLE_CLIENT_SECRET**: Accompanying Client Secret.

Make sure to enable the **Google Tasks API** in your Google API Library before authenticating.

## Setup

To link this source schema locally to your Coral workspace, run:
```bash
coral source add --file sources/community/google_tasks/manifest.yaml
```

## Tables

### `task_lists`
Lists metadata for all task lists owned by or shared with the authenticated user. Entry-point table; requires no filters.

### `tasks`
Lists specific task items. 

**Crucial Note:** This table requires a mandatory filter clause on `tasklist_id` due to upstream Google API design constraints.

**API Pushdown Filters:**
You can efficiently filter tasks at the API layer by utilizing the following optional filters in your `WHERE` clauses:
* `show_completed` (Boolean) - Include completed tasks.
* `show_hidden` (Boolean) - Include hidden tasks (required to see tasks completed in Google first-party clients).
* `show_deleted` (Boolean) - Include deleted tasks.
* `due_min` / `due_max` (String) - Filter by due date bounds (RFC 3339).
* `completed_min` / `completed_max` (String) - Filter by completion date bounds (RFC 3339).
* `updated_min` (String) - Filter by last updated date bound (RFC 3339).

## Example Queries

### 1. Discover your task list IDs
```sql
SELECT id, title, updated FROM google_tasks.task_lists;
```

### 2. Query incomplete tasks from a targeted list
```sql
SELECT id, title, due 
FROM google_tasks.tasks 
WHERE tasklist_id = 'your_list_id_here' 
  AND status = 'needsAction';
```

### 3. Track completed items across lists via a JOIN
```sql
SELECT 
    l.title AS list_name, 
    t.title AS task_name, 
    t.completed
FROM google_tasks.tasks t
JOIN google_tasks.task_lists l ON t.tasklist_id = l.id
WHERE t.status = 'completed';
```

### 4. Fetch hidden and completed tasks from the API layer
```sql
SELECT id, title, completed, hidden
FROM google_tasks.tasks 
WHERE tasklist_id = 'your_list_id_here' 
  AND show_completed = true 
  AND show_hidden = true;
```

### 5. Filter by specific due dates using pushdown API parameters
```sql
SELECT title, due 
FROM google_tasks.tasks 
WHERE tasklist_id = 'your_list_id_here' 
  AND due_min = '2023-10-01T00:00:00Z'
  AND due_max = '2023-10-31T23:59:59Z';
```

### 6. Extremely fast single-item lookups (Point Queries)
```sql
-- This routes directly to the `/tasks/v1/users/@me/lists/{id}` endpoint
SELECT * FROM google_tasks.task_lists WHERE id = 'your_list_id_here';

-- This routes directly to the `/tasks/v1/lists/{tasklist}/tasks/{id}` endpoint
SELECT * FROM google_tasks.tasks 
WHERE tasklist_id = 'your_list_id_here' 
  AND id = 'your_task_id_here';
```

## API Documentation
Reference: [Google Tasks REST API v1 Reference](https://developers.google.com/tasks/api/reference/rest/v1)
