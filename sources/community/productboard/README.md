# Productboard (Community)

**Version:** 0.1.0
**Backend:** HTTP (Productboard API v2)
**Tables:** 4
**Base URL:** `https://api.productboard.com`

Query notes, features, members, and teams from Productboard. Designed for 
product roadmap visibility, feedback analysis, and cross-source joins with 
bundled sources like **Slack**, **Intercom**, and **Jira**.

## Install

Community sources are not bundled with the Coral binary. Add the manifest from
this directory:

```bash
coral source add --file sources/community/productboard/manifest.yaml
```

Or copy `manifest.yaml` into your workspace and pass that path to
`coral source add --file`.

Reference the linked GitHub issue in your PR so maintainers can connect the
contribution to the prior discussion.

## Authentication and setup

Requires `PRODUCTBOARD_ACCESS_TOKEN` (Personal Access Token).

1. In Productboard, go to **Workspace Settings → Integrations → Public APIs**.
2. Click **Add Token** (requires at least a Pro plan).
3. Ensure the token has the `members:pii:read` scope if you need to query actual names and emails.
4. Copy the access token.

```bash
export PRODUCTBOARD_ACCESS_TOKEN=eyJhb...
coral source add --file sources/community/productboard/manifest.yaml
```

See [Productboard API Token Documentation](https://developer.productboard.com/reference/api-token).

## Table categories

### Product data

| Table | Description |
| --- | --- |
| `notes` | Feedback and conversations. Primary join key `owner_id`. |
| `features` | Product features and roadmap items. |
| `members` | Workspace members. |
| `teams` | Workspace teams. |

## Filters and pagination

- List tables (`notes`, `features`, `members`, `teams`) use Productboard cursor pagination (`pageCursor`). Always use `LIMIT` on large workspaces.
- **Note on pagination limitations**: Productboard API v2 returns full URLs in `links.next`. Pagination might experience issues until Coral engine parses these out natively, so use `LIMIT` to keep query bounds small for now.
- `features` restricts results strictly to entities where `type=feature`. Other entities like components or objectives are not currently exposed in this v1 source.

## Example relationships

```text
productboard.notes.owner_id
  → productboard.members.id

productboard.members.email
  → slack.users.email
  → intercom.contacts.email
```

## Example queries

### Notes and owners

```sql
SELECT n.id, n.name, n.created_at, m.email AS owner_email
FROM productboard.notes n
LEFT JOIN productboard.members m ON n.owner_id = m.id
WHERE n.archived = false
LIMIT 20;
```

### Search features by status

```sql
SELECT id, name, status_name, timeframe_start
FROM productboard.features
WHERE status_name = 'In Progress'
LIMIT 10;
```

### Cross-tool member join (requires Slack)

```sql
SELECT p.name AS pb_name, p.role, s.name AS slack_name
FROM productboard.members p
JOIN slack.users s ON LOWER(p.email) = LOWER(s.profile_email)
WHERE p.disabled = false
LIMIT 20;
```

### Pipeline summary

```sql
SELECT status_name, COUNT(*) AS feature_count
FROM productboard.features
GROUP BY status_name
ORDER BY feature_count DESC;
```

## Validation

```bash
# YAML style (requires: cargo install ryl --locked)
make lint-sources

# Manifest structure and smoke queries (requires Coral CLI)
coral source lint sources/community/productboard/manifest.yaml
export PRODUCTBOARD_ACCESS_TOKEN=eyJhb...
coral source add --file sources/community/productboard/manifest.yaml
coral source test productboard
```

## Limitations

- **Read-only** v1 (no creates/updates).
- **Entities scoping**: The `features` table only exposes features (not components, objectives, or products).
- **PII redaction**: Without the `members:pii:read` scope, the `members` table will return `[redacted]` for `name` and `email` fields.
- **Content field**: The `notes.content` field is typed as `Json` because its schema is polymorphic (can be a string for `textNote` or an array for `conversationNote`).
- **Pagination gap**: Productboard returns a full URL in `links.next` instead of a raw cursor. This may require future DSL enhancements in Coral to fully support native fetching of >100 records.
- Community sources are maintained separately from bundled core sources.

## Contributing

Follow [CONTRIBUTING.md](../../../CONTRIBUTING.md): discuss on the issue first,
sign the CLA if this is your first contribution, run `make lint-sources`, and
open a focused PR titled `feat(sources/community/productboard): add productboard community source`.
