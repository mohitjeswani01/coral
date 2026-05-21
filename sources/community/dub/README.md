# Dub

**Version:** 0.1.0
**Backend:** HTTP
**Tables:** 4
**Base URL:** `https://api.dub.co`

Query links, domains, tags, and workspace metadata from Dub.co — the modern link attribution platform for short links, conversion tracking, and affiliate programs.

## Authentication

Requires a `DUB_API_KEY`. Generate one from:
**Workspace Settings → API Keys → Create API Key**

Keys start with `dub_` and are scoped to a single workspace — no `workspaceId`
parameter is needed in queries.

```bash
coral source add --file sources/community/dub/manifest.yaml
```

You will be prompted to enter your API key interactively.

API docs: https://dub.co/docs/api-reference/introduction

## Tables

| Table | Description | Required filters | Optional filters |
|---|---|---|---|
| `workspaces` | Workspace metadata, plan, and team members | — | — |
| `links` | Short links with click, lead, and sales analytics | — | `domain`, `search`, `show_archived`, `tag_ids`, `folder_id` |
| `domains` | Custom domains and their verification status | — | — |
| `tags` | Tags for organizing and categorizing links | — | `search` |

### Key design notes

- **No workspace scoping filter needed.** Dub API keys are workspace-scoped
  since July 2024. The API key determines which workspace data is returned.
- **`links` is the richest table.** It includes `clicks`, `leads`, `sales`,
  `sale_amount`, and `conversions` directly in the list response.
- **`workspaces` is the entry point.** Query it first to verify connectivity
  and discover workspace identity and plan.
- **`tags` are referenced by `links`.** The `tags` column on links is a JSON
  array of `{id, name, color}` objects.

```text
workspaces → workspace metadata, plan, usage
links      → short links with analytics (filterable by domain, tags)
domains    → custom domains and DNS verification
tags       → tag definitions (id, name, color)
```

### links filter values

| Filter | Description |
|---|---|
| `domain` | Filter by domain (e.g. `dub.sh`, `yourdomain.com`) |
| `search` | Search link slugs and destination URLs |
| `show_archived` | Set to `true` to include archived links (default: false) |
| `tag_ids` | Filter by tag ID(s) |
| `folder_id` | Filter by folder ID |

## Quick start

```bash
# Step 1 — verify API key and discover workspace info
coral sql "SELECT id, name, slug, plan FROM dub.workspaces"

# Step 2 — list links with click analytics
coral sql "
  SELECT id, domain, key, url, clicks, leads, sales, created_at
  FROM dub.links
  ORDER BY clicks DESC
  LIMIT 20
"

# Step 3 — list all custom domains
coral sql "SELECT slug, verified, primary, archived FROM dub.domains"

# Step 4 — list all tags
coral sql "SELECT id, name, color FROM dub.tags"

# Step 5 — filter links by domain
coral sql "
  SELECT id, key, url, clicks
  FROM dub.links
  WHERE domain = 'yourdomain.com'
  LIMIT 20
"
```

## Example queries

### All links with click analytics

```sql
SELECT
  id,
  domain,
  key,
  url,
  short_link,
  clicks,
  leads,
  sales,
  sale_amount,
  created_at
FROM dub.links
ORDER BY clicks DESC
LIMIT 50;
```

### Top performing links by clicks

```sql
SELECT
  short_link,
  url,
  clicks,
  leads,
  sales,
  conversions,
  last_clicked,
  created_at
FROM dub.links
ORDER BY clicks DESC
LIMIT 10;
```

### Filter links by domain

```sql
SELECT
  id,
  key,
  url,
  clicks,
  leads,
  archived,
  created_at
FROM dub.links
WHERE domain = 'yourdomain.com'
LIMIT 50;
```

### List all custom domains

```sql
SELECT
  slug,
  verified,
  primary,
  archived,
  target,
  type,
  created_at
FROM dub.domains;
```

### List tags and their colors

```sql
SELECT
  id,
  name,
  color
FROM dub.tags;
```
