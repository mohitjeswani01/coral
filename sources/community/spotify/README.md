# Spotify community source

The `spotify` community source exposes read-only Spotify data through Coral
SQL. Query your playlists, saved tracks, top artists, and top tracks using
the Spotify Web API with OAuth 2.0 authorization-code and PKCE.

## Setup

### 1. Create a Spotify app

> **Spotify Premium required**: the app owner must hold an active Spotify
> Premium subscription (including Family or Duo plans) to use the Web API in
> Development Mode. If the subscription lapses, the app will stop working.

Open the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard)
and create an app (or use an existing one).

In the app settings, add the following redirect URI exactly as shown:

```
http://127.0.0.1:53682/oauth/callback
```

> **Important**: Spotify no longer accepts `localhost` aliases. The loopback
> IP literal `127.0.0.1` is required. Using `localhost` will cause
> authorization to fail.

Copy the **Client ID** from the app dashboard. Do not use the client secret —
this source uses the PKCE public-client flow and does not need one.

### 2. Set your Client ID

```sh
export SPOTIFY_CLIENT_ID="<your_client_id>"
```

### 3. Add the source

```sh
cargo run -p coral-cli -- source add --file sources/community/spotify/manifest.yaml
```

Coral will open your browser to the Spotify authorization page. After you
approve the requested scopes, Coral stores the access token and refresh token
automatically. Token refresh is handled transparently — no manual intervention
is needed.

**Manual token path (short-lived)**: To paste a token instead of using the
browser flow, run the source add command and choose **Paste access token** when
prompted. `SPOTIFY_CLIENT_ID` is not required for this path. However, the paste
path bypasses the OAuth flow entirely — no refresh token is issued. Spotify
access tokens expire after **one hour** with no way to refresh them. This path
is intended for quick testing only.

### 4. Verify

```sh
cargo run -p coral-cli -- source test spotify
```

## Required scopes

| Scope | Tables |
|---|---|
| `playlist-read-private` | `spotify.playlists` |
| `playlist-read-collaborative` | `spotify.playlists` |
| `user-library-read` | `spotify.saved_tracks` |
| `user-top-read` | `spotify.top_artists`, `spotify.top_tracks` |

Coral requests all four scopes in a single authorization. If you scope your
token manually, include every scope for the tables you plan to query.

## Tables

| Table | Description | Optional filters | Pagination |
|---|---|---|---|
| `spotify.playlists` | Playlists owned by or followed by the authenticated user | none | offset |
| `spotify.saved_tracks` | Tracks saved to the user's Liked Songs library | none | offset |
| `spotify.top_artists` | Top artists by listening affinity | `time_range` | offset |
| `spotify.top_tracks` | Top tracks by listening affinity | `time_range` | offset |

All tables are read-only. This source does not create, modify, or delete any
Spotify data.

The `time_range` filter for `top_artists` and `top_tracks` accepts:

| Value | Window |
|---|---|
| `short_term` | approximately last 4 weeks |
| `medium_term` | approximately last 6 months (default) |
| `long_term` | approximately last 1 year |

## Example queries

List your playlists with track counts:

```sql
SELECT id, name, tracks_total, owner_name, public, collaborative
FROM spotify.playlists
ORDER BY name
LIMIT 20;
```

Find your longest collaborative playlists:

```sql
SELECT id, name, tracks_total, owner_name
FROM spotify.playlists
WHERE collaborative IS TRUE
ORDER BY tracks_total DESC
LIMIT 10;
```

Most recently saved tracks with primary artist:

```sql
SELECT added_at, track_name, artist_name, album_name, popularity
FROM spotify.saved_tracks
ORDER BY added_at DESC
LIMIT 20;
```

Saved tracks by a specific artist (note: `artist_name` is not pushed down
to Spotify — Coral fetches pages until the result is complete or the fetch
limit is reached):

```sql
SELECT track_name, album_name, added_at, duration_ms
FROM spotify.saved_tracks
WHERE artist_name = 'Radiohead'
ORDER BY added_at DESC
LIMIT 50;
```

Explicit saved tracks sorted by popularity:

```sql
SELECT track_name, artist_name, popularity, album_name
FROM spotify.saved_tracks
WHERE explicit IS TRUE
ORDER BY popularity DESC
LIMIT 20;
```

Your all-time top artists with genre and follower data:

```sql
SELECT name, popularity, followers_total, genres_joined
FROM spotify.top_artists
WHERE time_range = 'long_term'
ORDER BY popularity DESC
LIMIT 20;
```

Compare your top artists across time ranges:

```sql
SELECT 'short_term' AS period, name, popularity FROM spotify.top_artists WHERE time_range = 'short_term'
UNION ALL
SELECT 'long_term'  AS period, name, popularity FROM spotify.top_artists WHERE time_range = 'long_term'
ORDER BY period, popularity DESC;
```

Your recent top tracks with album context:

```sql
SELECT name, artist_name, album_name, popularity, duration_ms
FROM spotify.top_tracks
WHERE time_range = 'short_term'
ORDER BY popularity DESC
LIMIT 20;
```

Cross-reference top tracks against saved library:

```sql
SELECT t.name, t.artist_name, t.popularity,
       s.added_at IS NOT NULL AS in_library
FROM spotify.top_tracks t
LEFT JOIN spotify.saved_tracks s ON t.id = s.track_id
WHERE t.time_range = 'medium_term'
ORDER BY t.popularity DESC
LIMIT 20;
```

## Validation

Lint the manifest:

```sh
cargo run -p coral-cli -- source lint sources/community/spotify/manifest.yaml
```

Install and run the built-in test queries:

```sh
export SPOTIFY_CLIENT_ID="<your_client_id>"
cargo run -p coral-cli -- source add --file sources/community/spotify/manifest.yaml
cargo run -p coral-cli -- source test spotify
```

Inspect registered tables and columns:

```sh
cargo run -p coral-cli -- sql "SELECT table_name, description FROM coral.tables WHERE schema_name = 'spotify'"
cargo run -p coral-cli -- sql "SELECT table_name, column_name, data_type FROM coral.columns WHERE schema_name = 'spotify' ORDER BY table_name, ordinal_position"
```

## Notes

- **Spotify Premium required**: the app owner must hold an active Spotify
  Premium subscription to use the Web API in Development Mode. This includes
  Family and Duo plans. If the subscription lapses, API calls will fail.
- **Token expiry and refresh**: Spotify access tokens expire after one hour.
  The OAuth browser flow issues a refresh token, which Coral uses automatically
  for transparent long-lived access. The **Paste access token** path bypasses
  the OAuth flow and issues no refresh token — the token expires after one hour
  with no refresh capability regardless of whether `SPOTIFY_CLIENT_ID` is set.
  Use the browser flow for any usage beyond quick testing.
- **OAuth PKCE**: Spotify requires PKCE for public clients. No client secret is
  used or stored. The `SPOTIFY_CLIENT_ID` variable holds the app's Client ID,
  which is not a credential. It can be left blank for the paste-token path,
  but no refresh token is available on that path regardless.
- **Redirect URI**: Spotify requires the loopback IP literal
  `http://127.0.0.1:53682/oauth/callback`. Add this URI exactly in your
  Spotify app dashboard. `localhost` and `127.0.0.1:0` are not accepted.
- **Development Mode limit**: Spotify apps in Development Mode allow at most
  5 authenticated users (the app owner plus 4 others). Each additional user
  must be manually added to the **Users and Access** allowlist in the Spotify
  Developer Dashboard. Moving beyond 5 users requires Extended Quota Mode,
  which Spotify currently grants only to registered organizations.
- **saved_tracks fetch limit**: `saved_tracks` has a default fetch limit of
  100 tracks. Filters such as `artist_name` are not pushed down to Spotify —
  Coral fetches pages until the limit is reached or pages are exhausted.
  Always include a `LIMIT` clause when querying with non-pushdown filters to
  avoid scanning a large library and hitting Spotify's rolling rate limit.
- **Offset pagination**: All four tables use offset-based pagination with a
  maximum page size of 50. The `playlists` table supports a maximum offset
  of 100,000 (per the Spotify API spec). The `saved_tracks` and `top_artists`
  / `top_tracks` tables do not document a hard offset cap in the current API
  reference for the access levels this source uses.
- **Nested fields**: Several columns are sourced from nested API response
  objects. `tracks_total` in `playlists` comes from `items.total` on each
  `SimplifiedPlaylistObject` — this is the `PlaylistTracksRefObject` that
  holds a link and the track count for that playlist. The `tracks` field
  carrying the same data is marked deprecated in the Spotify OpenAPI schema
  (February 2026) with "Use `items` instead". `followers_total` in
  `top_artists` comes from `followers.total`. These are not flat fields.
- **Primary artist**: `artist_name` and `artist_id` in `saved_tracks` and
  `top_tracks` reflect the first element of the API's `artists` array. Use
  the `artists` Json column for the complete artist list.
- **album_release_date format**: Spotify returns release dates with variable
  precision — YYYY, YYYY-MM, or YYYY-MM-DD. The column is exposed as `Utf8`
  to preserve the original string.
- **description and public nullability**: `description` is null for
  unmodified or unverified playlists. `public` can be null when the status is
  not applicable.
- **Rate limits**: The Spotify Web API enforces rate limits on a rolling
  30-second window. On a `429 Too Many Requests` response Spotify returns a
  `Retry-After` header. Coral will surface the error; add retries or reduce
  query frequency if you hit limits.

## Out of scope for v1

- Track audio features (endpoint restricted to pre-approved apps by Spotify)
- Artist related tracks and related artists (restricted endpoint)
- Track recommendations (restricted endpoint)
- Playlist track contents (requires per-playlist requests)
- Recently played tracks (`user-read-recently-played` scope)
- Followed artists (`user-follow-read` scope)
- Saved albums (`user-library-read` scope, separate endpoint)
- User profile (`user-read-private`, `user-read-email` scopes)
- Write operations of any kind
