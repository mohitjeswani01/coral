# Semaphore CI Source

Query CI/CD data from [Semaphore CI](https://semaphoreci.com) — workflows,
pipelines, and promotions for your projects. Read-only; uses the
[v1alpha REST API](https://docs.semaphoreci.com/reference/api).

## Setup

### 1. Get your API token

Generate an API token from your Semaphore account:
[Account Settings → API Tokens](https://me.semaphoreci.com/account).

### 2. Configure environment variables

```bash
export SEMAPHORE_API_TOKEN="your-semaphore-api-token"
export SEMAPHORE_ORG_SLUG="mycompany"
```

The org slug is the subdomain part of your Semaphore dashboard URL.
For example, if your URL is `https://mycompany.semaphoreci.com`, your slug
is `mycompany`.

### 3. Add the source

```bash
coral source add --file sources/community/semaphore_ci/manifest.yaml
```

## Authentication

| Input | Kind | Description |
|---|---|---|
| `SEMAPHORE_API_TOKEN` | Secret | Semaphore CI API token |
| `SEMAPHORE_ORG_SLUG` | Variable | Organization slug (subdomain) |

Auth uses `Authorization: Token <TOKEN>` per the Semaphore v1alpha API spec.

## Tables

> **Note:** The Semaphore v1alpha API does not expose a list-projects
> endpoint. To get your `project_id`, use the Semaphore dashboard URL
> (visible in project settings) or the CLI: `sem get project <name>`.
> All tables below require a `project_id` or `pipeline_id` filter.

### workflows

Workflow runs for a project. Each row is one workflow triggered by a push,
tag, pull request, or manual rerun.

| Column | Type | Description |
|---|---|---|
| `project_id` | Utf8 | Project ID (required filter) |
| `wf_id` | Utf8 | Workflow UUID |
| `requester_id` | Utf8 | User or system that requested this workflow |
| `repository_id` | Utf8 | Repository UUID |
| `organization_id` | Utf8 | Organization UUID |
| `initial_ppl_id` | Utf8 | Initial pipeline UUID |
| `hook_id` | Utf8 | Webhook event UUID |
| `branch_name` | Utf8 | Git branch name |
| `branch_id` | Utf8 | Semaphore branch UUID |
| `commit_sha` | Utf8 | Git commit SHA |
| `triggered_by` | Utf8 | Trigger source (integer enum or string) |
| `rerun_of` | Utf8 | Original workflow UUID if rerun |
| `created_at` | Timestamp | Workflow creation time (UTC) |

**Required filter:** `project_id`
**Optional filter:** `branch_name`
**Pagination:** Link header

---

### pipelines

Pipelines for a project. Each row is one pipeline execution within a
workflow.

| Column | Type | Description |
|---|---|---|
| `project_id` | Utf8 | Project ID (required filter) |
| `ppl_id` | Utf8 | Pipeline UUID (may be absent in list) |
| `name` | Utf8 | Pipeline name from YAML config |
| `yaml_file_name` | Utf8 | YAML file defining this pipeline |
| `working_directory` | Utf8 | Pipeline working directory |
| `wf_id` | Utf8 | Parent workflow UUID |
| `state` | Utf8 | State (running, done, stopping) |
| `result` | Utf8 | Result (passed, failed, stopped, canceled) |
| `branch_name` | Utf8 | Git branch name |
| `created_at` | Timestamp | Pipeline creation time (UTC) |

**Required filter:** `project_id`
**Optional filters:** `wf_id`, `branch_name`
**Pagination:** Link header

---

### promotions

Promotions triggered from a pipeline (e.g. deploy to staging/production).

| Column | Type | Description |
|---|---|---|
| `pipeline_id` | Utf8 | Pipeline ID (required filter) |
| `name` | Utf8 | Promotion name (e.g. production) |
| `status` | Utf8 | Promotion status (passed, failed) |
| `triggered_by` | Utf8 | What triggered the promotion |

**Required filter:** `pipeline_id`
**Pagination:** None

---

## Typical Query Flow

Since there is no projects list endpoint, the typical workflow is:

1. **Get your project ID** from the Semaphore UI or CLI
2. **Query workflows** for that project
3. **Drill into pipelines** using the project ID or a specific workflow ID
4. **Check promotions** for a specific pipeline

## Example Queries

```sql
-- List recent workflows for a project
SELECT wf_id, branch_name, commit_sha, created_at
FROM semaphore_ci.workflows
WHERE project_id = 'your-project-uuid'
LIMIT 10;

-- Find failed pipelines
SELECT name, state, result, branch_name, created_at
FROM semaphore_ci.pipelines
WHERE project_id = 'your-project-uuid'
  AND result = 'FAILED';

-- Filter pipelines by workflow ID
SELECT name, state, result, yaml_file_name
FROM semaphore_ci.pipelines
WHERE project_id = 'your-project-uuid'
  AND wf_id = 'your-workflow-uuid';

-- Find pipelines by branch name
SELECT name, state, result, created_at
FROM semaphore_ci.pipelines
WHERE project_id = 'your-project-uuid'
  AND branch_name = 'main';

-- List promotions for a pipeline
SELECT name, status, triggered_by
FROM semaphore_ci.promotions
WHERE pipeline_id = 'your-pipeline-uuid';
```

## Pagination

| Table | Mode | Default Page Size |
|---|---|---|
| `workflows` | Link header | 30 |
| `pipelines` | Link header | 30 |
| `promotions` | None | — |

## Notes

- **No projects endpoint**: The Semaphore v1alpha API does not provide a
  `GET /projects` endpoint. Obtain your `project_id` from the Semaphore
  dashboard (Project Settings) or CLI (`sem get project`).
- **Read-only**: This source only uses GET endpoints. No create, update,
  or delete operations.
- **Timestamps**: The API returns `created_at` as a protobuf-style
  `{seconds, nanos}` object. This source extracts the `seconds` field
  and exposes it as a UTC `Timestamp` column.
- **Case inconsistency**: The `state` and `result` fields may use different
  casing between list and describe responses (e.g. `DONE` vs `done`,
  `FAILED` vs `failed`). Use case-insensitive comparisons when filtering.
- **API docs**: https://docs.semaphoreci.com/reference/api
