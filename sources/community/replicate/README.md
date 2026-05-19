# Replicate

Query predictions, models, deployments, and collections from
[Replicate](https://replicate.com/) — the cloud API for running machine
learning models.

## Authentication

Requires a **Replicate API token**.

1. Log in to Replicate → [Account → API tokens](https://replicate.com/account/api-tokens) → **New API token**.
2. Give it a name (e.g. "coral-read") and copy the generated token.
3. A read-only token is sufficient for all tables in this source.

```sh
export REPLICATE_API_TOKEN="r8_..."
coral source add --file sources/community/replicate/manifest.yaml
```

See the [Replicate API authentication docs](https://replicate.com/docs/reference/http)
for details on token management and permissions.

## Tables

| Table | Description | Filters |
|---|---|---|
| `replicate.predictions` | Your AI inference history — every model run under the authenticated account | — |
| `replicate.models` | Global public ML model catalog — browse and discover models by owner, latest updates, or creation date | `sort_by`, `sort_direction` (optional) |
| `replicate.deployments` | Deployed models configured for the authenticated account, with hardware and scaling config | — |
| `replicate.collections` | Curated model groups maintained by Replicate (e.g. "super-resolution", "text-to-image") | — |
| `replicate.hardware` | Available GPU and CPU hardware options and their SKU identifiers | — |

### `replicate.predictions`

Returns all predictions made by the authenticated user, newest first.
No filter is required — the API token scopes results to the account automatically.

Key columns:

| Column | Type | Description |
|---|---|---|
| `id` | `Utf8` | Unique prediction ID |
| `model` | `Utf8` | Model in `owner/name` format (e.g. `stability-ai/sdxl`) |
| `version` | `Utf8` | 64-char version ID, or `"hidden"` for official models |
| `status` | `Utf8` | `starting` · `processing` · `succeeded` · `failed` · `canceled` · `aborted` |
| `created_at` | `Timestamp` | When the prediction was created |
| `started_at` | `Timestamp` | When the model began processing |
| `completed_at` | `Timestamp` | When the prediction finished |
| `input` | `Json` | Model-specific input parameters |
| `output` | `Json` | Model-specific output (URL, string, or object) |
| `error` | `Utf8` | Error message when `status = 'failed'` |
| `metrics__total_time` | `Float64` | Total wall-clock duration in seconds |
| `data_removed` | `Boolean` | True when output data has been deleted by retention policy |

### `replicate.models`

Global public model catalog. All public models are returned; private models
are excluded. Supports optional `sort_by` and `sort_direction` filters.

| Filter | Values | Description |
|---|---|---|
| `sort_by` | `model_created_at`, `latest_version_created_at` | Sort field (default: `latest_version_created_at`) |
| `sort_direction` | `asc`, `desc` | Sort direction (default: `desc`) |

### `replicate.deployments`

Account-scoped deployments with current release details. The `release_hardware`
column contains the hardware SKU — join against `replicate.hardware` on `sku`
to get the human-readable name.

### `replicate.collections`

Curated collections grouping models by theme. Only three fields are returned
by the list endpoint (`name`, `slug`, `description`). Browse models in a
collection at `replicate.com/collections/<slug>`.

### `replicate.hardware`

Static reference list of hardware tiers. Typically fewer than 20 entries.
`sku` values correspond to `release_hardware` in `replicate.deployments`.

## Example queries

### List recent predictions with status

```sql
SELECT
  id,
  model,
  version,
  status,
  created_at,
  completed_at,
  metrics__total_time
FROM replicate.predictions
ORDER BY created_at DESC
LIMIT 20;
```

### Find failed predictions with error details

```sql
SELECT
  id,
  model,
  status,
  created_at,
  error,
  logs
FROM replicate.predictions
WHERE status = 'failed'
ORDER BY created_at DESC
LIMIT 50;
```

### Browse recently updated public models

```sql
SELECT
  owner,
  name,
  description,
  run_count,
  is_official,
  url
FROM replicate.models
WHERE sort_by = 'latest_version_created_at'
  AND sort_direction = 'desc'
LIMIT 25;
```

### List your deployments with hardware and scaling configuration

```sql
SELECT
  owner,
  name,
  release_model,
  release_version,
  release_hardware,
  release_min_instances,
  release_max_instances,
  release_created_at
FROM replicate.deployments
ORDER BY release_created_at DESC;
```

### Show available hardware options

```sql
SELECT
  name,
  sku
FROM replicate.hardware;
```

## Auth

This source uses the `Authorization: Bearer <token>` header with a Replicate
API token. See the
[Replicate API reference](https://replicate.com/docs/reference/http) for full
documentation on authentication and API tokens.
