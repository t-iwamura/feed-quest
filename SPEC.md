# feed-quest — Implementation Spec

## Context

Greenfield RSS reader for a single user who accesses it from multiple devices. Deployed as a portable Docker image (target host TBD). All architectural choices below were made explicitly by the user during interview; alternatives are not re-litigated here. Scope is deliberately narrow — this is a personal tool, not a product.

## Product summary

Self-hosted, single-user RSS reader. React SPA + Node/Hono API in one container, SQLite on a mounted volume. Mobile-first single-column UI. Feeds are polled server-side on a fixed interval; article content is shown as delivered by the feed, with an on-demand "fetch full article" action.

## Non-goals (intentionally cut)

- OPML import/export
- Server-side search (client-side substring filter on current list only)
- Podcast / enclosure playback
- Web Push notifications
- Multi-user accounts / user management
- Auto-discovery of feeds from a site URL
- Starred / read-later / bookmarks
- Always-on full-article extraction (only on user action)
- Per-feed retention or polling overrides (global settings only)

## Stack

- **Backend:** Node 20 LTS, TypeScript (ESM), Hono (HTTP), better-sqlite3, `@rowanmanning/feed-parser` (RSS 2.0 / Atom / JSON Feed), sanitize-html, argon2, JSDOM + `@mozilla/readability` (for on-demand full-article extraction), pino (logs).
- **Frontend:** React + Vite + TypeScript, React Router, TanStack Query, Tailwind CSS.
- **Testing:** Vitest (both sides).
- **Container:** multi-stage Dockerfile; single runtime image serves the SPA static bundle and the API from the same origin on one port.

All code uses ES modules per `CLAUDE.md`.

## Repo layout

```
feed-quest/
  server/
    src/
      index.ts           # Hono app + startup + poller boot
      db.ts              # better-sqlite3 handle + migrations
      auth.ts            # argon2, session cookies
      poller.ts          # fixed-interval fetch loop + prune job
      ingest.ts          # parse + dedupe + insert
      sanitize.ts        # sanitize-html allowlist config
      readability.ts     # on-demand full-article extraction
      routes/
        auth.ts
        feeds.ts
        items.ts
        settings.ts
  web/                   # Vite React app
    src/
      routes/            # /login, /feeds, /feeds/:id, /items/:id, /settings
      api/               # fetch wrappers + query keys
      components/
      lib/
  shared/                # shared TS types (Feed, Item, Settings)
  Dockerfile
  docker-compose.yml     # dev only
  SPEC.md
  CLAUDE.md
```

## Data model (SQLite)

```sql
-- one row expected; bootstrapped from env on first start
CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  username TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  created_at TEXT NOT NULL
);

CREATE TABLE sessions (
  id TEXT PRIMARY KEY,             -- opaque random token, stored in cookie
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  last_used_at TEXT NOT NULL,      -- sliding expiry: session valid while last_used_at + 30d > now
  created_at TEXT NOT NULL
);

CREATE TABLE feeds (
  id INTEGER PRIMARY KEY,
  feed_url TEXT NOT NULL UNIQUE,
  title TEXT NOT NULL,             -- user-overridable
  site_url TEXT,
  etag TEXT,
  last_modified TEXT,              -- HTTP Last-Modified header, verbatim
  last_polled_at TEXT,
  last_error TEXT,                 -- null if last poll succeeded
  consecutive_failures INTEGER NOT NULL DEFAULT 0,
  next_retry_at TEXT,              -- backoff gate; null = eligible now
  created_at TEXT NOT NULL
);

CREATE TABLE items (
  id INTEGER PRIMARY KEY,
  feed_id INTEGER NOT NULL REFERENCES feeds(id) ON DELETE CASCADE,
  guid TEXT NOT NULL,              -- feed <guid>/<id>; missing → hash(link+title+pubdate)
  url TEXT,
  title TEXT NOT NULL,
  author TEXT,
  published_at TEXT,               -- feed-provided; falls back to ingested_at
  content_html TEXT,               -- ALREADY SANITIZED at ingest; nullable for title-only feeds
  fetched_full_html TEXT,          -- populated only on user action
  is_read INTEGER NOT NULL DEFAULT 0,
  read_at TEXT,
  ingested_at TEXT NOT NULL,
  UNIQUE (feed_id, guid)
);
CREATE INDEX items_feed_ingested ON items (feed_id, ingested_at DESC);
CREATE INDEX items_unread ON items (is_read, ingested_at DESC);
CREATE INDEX items_prune ON items (is_read, read_at);

-- single-row settings table
CREATE TABLE settings (
  id INTEGER PRIMARY KEY CHECK (id = 1),
  poll_interval_minutes INTEGER NOT NULL DEFAULT 30,
  retention_days INTEGER NOT NULL DEFAULT 30
);
```

**Configuration precedence:** `POLL_INTERVAL_MINUTES` / `RETENTION_DAYS` env vars are used **only to seed the `settings` row on first start** (when the row does not yet exist). After bootstrap the DB row is the sole source of truth; changing the env var on restart has no effect. `PATCH /api/settings` writes to this row and the poller re-reads it at the start of each tick, so changes take effect from the next tick — no process restart needed.

Dedup rule: `UNIQUE (feed_id, guid)` — same GUID across two different feeds is intentionally treated as two items (per interview).

## HTTP API

All routes JSON. All non-`/api/auth/login` routes require a valid session cookie; state-changing routes also require `Origin` to match the server's origin (lightweight CSRF check).

**Auth**
- `POST /api/auth/login` `{username, password}` → sets `Set-Cookie: sid=…; HttpOnly; Secure; SameSite=Lax`
- `POST /api/auth/logout` — deletes session
- `GET  /api/auth/me` — `{username}` or 401

**Feeds**
- `GET    /api/feeds` — `[{id, title, site_url, unread_count, has_error, last_error, last_polled_at}]`
- `POST   /api/feeds` `{feed_url}` — normalizes the URL (lowercase scheme/host, strip trailing slash, drop URL fragment), then fetches and parses; **rejects with 400 if it's not a valid feed** (no autodiscovery); returns 409 if the normalized URL already exists.
- `PATCH  /api/feeds/:id` `{title?}`
- `DELETE /api/feeds/:id` — cascades items
- `POST   /api/feeds/:id/refresh` — trigger an immediate **synchronous** poll (resets backoff); returns the updated feed row on success, `502 { last_error }` on fetch/parse failure.

**Items**
- `GET  /api/items?feed_id=&filter=unread|all&cursor=&limit=50` — cursor is `ingested_at,id`
- `GET  /api/items/:id`
- `POST /api/items/:id/read`
- `POST /api/items/:id/unread`
- `POST /api/items/read-all?feed_id=` — mark all as read (optionally per feed)
- `POST /api/items/:id/fetch-full` — runs Readability, stores into `fetched_full_html`, returns it

**Settings**
- `GET   /api/settings`
- `PATCH /api/settings` `{poll_interval_minutes?, retention_days?}` — changes take effect from the next poller tick (poller re-reads settings each tick).
- `POST  /api/auth/password` `{current, new}`

## Poll worker (in-process)

Single `setInterval` inside the API process, tick every minute; on each tick pick feeds where `next_retry_at IS NULL OR next_retry_at <= now` AND `last_polled_at IS NULL OR last_polled_at + poll_interval <= now`. Poll them with a **maximum of 4 concurrent fetches** (`p-limit` or equivalent).

Per feed:
1. `safeFetch(feed_url, { headers: { 'If-None-Match': etag, 'If-Modified-Since': last_modified, 'User-Agent': USER_AGENT }})` (see **Outbound HTTP guards** below).
   - Conditional GET is added even though the interview picked "fixed interval" — it costs nothing and materially reduces the chance of being rate-limited or blocked by upstream hosts.
2. On `304 Not Modified`: update `last_polled_at`, clear error state, done.
3. On `2xx`: parse with `@rowanmanning/feed-parser`; for each item:
   - Compute `guid`:
     - Feed-provided `<guid>` / `<id>` if present.
     - Else if `pubDate` present: `sha1(link|title|pubDate)`.
     - Else if `link` present: `sha1(link)`.
     - Else: skip the item and log a warning (insufficient identity — can't safely dedupe).
   - Sanitize `content_html` with the strict allowlist.
   - `INSERT OR IGNORE` on `(feed_id, guid)`.
   - Save new `etag`/`last_modified` from response headers.
   - Wrap all inserts for one feed in a single transaction so a mid-tick shutdown cannot leave partial rows.
4. On error (network, 4xx/5xx, malformed XML): `consecutive_failures++`, `last_error = message`, `next_retry_at = now + min(30m * 2^(failures-1), 24h)`.

**UI signal:** a feed shows an error badge in the sidebar when `consecutive_failures >= 3`; hover/tap shows `last_error`.

**Prune job:** runs once per day (also on startup if never run). `DELETE FROM items WHERE is_read = 1 AND read_at < now - retention_days`.

## Ingestion & sanitization

- Parser accepts RSS 2.0, Atom 1.0, JSON Feed 1.1.
- Sanitize server-side on ingest with a strict allowlist:
  - Tags: `p a code pre blockquote ul ol li h2 h3 h4 h5 h6 img em strong br hr table thead tbody tr td th figure figcaption`
  - Attrs: `a[href,title]`, `img[src,alt,title]`, no `class`/`style`/`id`
  - Force `rel="noopener noreferrer"` and `target="_blank"` on all anchors
  - Strip `<script>`, `<style>`, `<iframe>`, event handlers, `javascript:` URLs. `data:` URLs allowed **only** for `data:image/{png,jpeg,gif,webp}` — `data:image/svg+xml` is explicitly rejected (SVG can carry `<script>`).
  - Resolve relative URLs against the item's URL
- Cap total sanitized `content_html` size at **1 MB** per item; anything larger is truncated with a trailing `<p>…</p>` marker, so a base64-heavy feed can't blow up the DB.
- Stored content is already-clean; the React renderer uses `dangerouslySetInnerHTML` directly with no runtime sanitization. Trade-off accepted: changing the allowlist requires re-ingesting to update existing rows.
- Full-article path: server `safeFetch(item.url)` → `new JSDOM(html, {url})` → `new Readability(...).parse()` → run the same sanitizer → persist to `fetched_full_html`. Uses the same `safeFetch` wrapper as the poller (see below).

### Outbound HTTP guards (`safeFetch`)

Every server-initiated fetch (feed registration, poller, full-article extraction) goes through a single wrapper with these guards:

- Scheme allowlist: `http`, `https` only. Reject `file:`, `gopher:`, `ftp:`, etc.
- **SSRF filter:** resolve the hostname to IP(s); reject if any resolved address is loopback (`127.0.0.0/8`, `::1`), link-local (`169.254.0.0/16`, `fe80::/10`), private (`10/8`, `172.16/12`, `192.168/16`, `fc00::/7`), or CGN (`100.64/10`). Reject cloud metadata (`169.254.169.254`) explicitly. Recheck on every redirect hop.
- Redirect cap: follow at most **5** redirects.
- Timeout: **15 s** total (connect + body).
- Response size cap: **5 MB**; abort the stream once exceeded.
- Fixed `User-Agent` from env `USER_AGENT`.

## UI (React, single-column mobile-first)

**Routes**
- `/login`
- `/` → redirects to `/feeds`
- `/feeds` — flat sidebar-less list: "All (n unread)", then each subscription with its unread count and error badge; footer link to `/settings`; "Add feed" input.
- `/feeds/:id` — item list ordered by `ingested_at DESC` (matches the cursor key so pagination is stable); header shows feed title + unread-only toggle (default on); infinite scroll via cursor.
- `/feeds/all` — same list, unioned across feeds.
- `/items/:id` — reader view. On mount, POST `/api/items/:id/read` (idempotent). Header: back, feed title, timestamp, "Mark unread", "Open original", "Fetch full article". Body renders `fetched_full_html` if present, else `content_html`.
- `/settings` — poll interval (minutes), retention (days), change password, logout.

**Behavior**
- Marking read fires **only** on entering `/items/:id` (per interview).
- Document title shows total unread count: `(n) feed-quest`; TanStack Query re-fetches unread count every 60s while tab is visible.
- Dark mode follows `prefers-color-scheme`.
- Keyboard shortcuts (desktop): `j`/`k` next/prev item in list, `o`/`Enter` open, `u` toggle unread, `g r` refresh current feed, `?` shortcut help.
- No service worker, no PWA install, no offline cache.

## Auth & security

- Password hashed with argon2id (`argon2` npm).
- Sessions: 32-byte random **opaque token**, stored server-side. Cookie: `HttpOnly; Secure; SameSite=Lax; Path=/`. Sliding expiry: session is valid while `last_used_at + 30d > now`; `last_used_at` is refreshed at most **once per hour** to avoid per-request write pressure. No cookie signing — the token is opaque and validated against the DB row on every request, so there is nothing meaningful to sign.
- CSRF: `SameSite=Lax` + `Origin` header allowlist check on `POST/PATCH/DELETE` (Lax already blocks cross-site form POSTs; Origin check covers the residual holes for non-idempotent GETs — we have none).
- **Bootstrap:** on startup, if `users` is empty, read `FEED_QUEST_USERNAME` + `FEED_QUEST_PASSWORD` from env and create the user. Startup fails hard if either is missing and the table is empty.
- No password reset flow — change password from settings while logged in; direct DB edit if locked out.
- **Login rate-limit is delegated to the reverse proxy** (nginx/caddy `limit_req` or equivalent). The app itself does not throttle `POST /api/auth/login`; the operator is responsible for wiring an edge-level rate limit. Failed logins are logged via pino so the proxy has something to correlate against.
- **Request body size limit:** Hono `bodyLimit` middleware set to **100 KB** for all `POST` / `PATCH` routes.

**Deployment assumption:** operator terminates TLS in front of the container (reverse proxy). App refuses to set `Secure` cookies over plain HTTP unless `TRUST_PROXY=1` and `X-Forwarded-Proto: https` is present.

## Docker

Multi-stage:
1. `node:20-alpine` builder installs deps (both `server/` and `web/`), runs `vite build`, then `tsc` for the server.
2. Runtime `node:20-alpine`: copies `server/dist` + `web/dist` + production `node_modules`. Runs as the image's built-in non-root `node` user (`USER node`). Command: `node server/dist/index.js`.

**Route registration order** (matters — SPA fallback must not swallow `/api`):
1. API routes under `/api/*`.
2. `serve-static` for `web/dist`.
3. SPA fallback `GET /*` **with an explicit exclusion for paths under `/api`** — so an unknown `/api/…` returns a 404 JSON response instead of `index.html`.

**Graceful shutdown:** on `SIGTERM` / `SIGINT`:
1. Stop the poller `setInterval` and `await` the in-flight tick.
2. `close()` the HTTP server (drain in-flight requests).
3. Close the SQLite handle.
4. `process.exit(0)`.

Per-feed ingest is wrapped in a single SQLite transaction, so a mid-tick kill cannot leave partial rows.

**Volume:** `/data` for `feed-quest.db`. **Backups:** use `sqlite3 /data/feed-quest.db ".backup /data/backups/feed-quest-$(date +%F).db"` on a host-side cron — a plain file copy while WAL is active can corrupt the snapshot.
**Env:** `PORT` (default 3000), `FEED_QUEST_USERNAME`, `FEED_QUEST_PASSWORD`, `POLL_INTERVAL_MINUTES` (default 30, **seed-only** — see Configuration precedence), `RETENTION_DAYS` (default 30, **seed-only**), `USER_AGENT` (default `feed-quest/0.1`), `TRUST_PROXY` (default 0).

## Tradeoffs worth naming

1. **Fixed 30-min polling** — chatty on quiet feeds, laggy on fast ones. Conditional GET added on top of the interview choice makes it mostly free.
2. **No search** — fine for tens of feeds; will feel limiting past a hundred. Easy to add SQLite FTS5 later against `items(title, content_html)`.
3. **No starred / read-later** — nothing survives the 30-day prune once read. If you want to keep something, mark it unread.
4. **In-process poller** — restart drops in-flight fetches. Acceptable at this scale; re-poll is idempotent.
5. **Sanitize-on-ingest** — allowlist changes need a backfill migration. Simpler render, faster page load.
6. **Single container** — the poller and the API share memory; a runaway feed parse can affect API latency. Fine for one user.

## Verification (end-to-end)

1. `docker compose up`, hit the URL, log in with bootstrap creds.
2. Add a real feed URL (e.g. `https://news.ycombinator.com/rss`) → items appear.
3. Try adding `https://news.ycombinator.com/` (site URL, not feed) → 400.
4. Open an item → returns to list → item shown read; reload → still read.
5. Restart the container mid-poll → no duplicate items after next tick.
6. Add a test feed whose content includes `<script>alert(1)</script>` and `<a href="javascript:...">` → confirm both stripped in rendered DOM.
7. Point a feed URL at a server returning 500 → after 3 ticks, error badge appears; `next_retry_at` visibly backs off in DB.
8. Set `RETENTION_DAYS=1`, manually backdate a read item's `read_at` by 2 days, wait for daily prune → row deleted.
9. Log in on device A, check unread count in tab title updates within 60s of new items arriving.
10. `POST /api/items/:id/fetch-full` on an item whose feed only gave a summary → response includes extracted body, DB has `fetched_full_html` populated.
11. Try adding `http://127.0.0.1:3000/api/feeds` or `http://169.254.169.254/` as a feed URL → rejected by the SSRF filter (never reaches the fetch stage).
12. Add `https://news.ycombinator.com/rss`, then attempt to add `HTTPS://news.ycombinator.com/rss/#anchor` → second request returns 409 (normalized to the same URL).
13. Add a feed whose `content_html` contains an `<img src="data:image/svg+xml,<svg…<script>…">` → sanitized output drops the `data:` URI entirely; no SVG reaches the DB.
14. `PATCH /api/settings` with `poll_interval_minutes: 5` → observe in logs that the next tick uses the new interval (no restart needed).

## Implementation order (suggested)

1. Repo scaffolding, Vite + Hono skeletons, Dockerfile.
2. DB + migrations + settings row + bootstrap user.
3. Auth (login/logout/me, argon2, cookie sessions, Origin check).
4. Feed CRUD + ingest (parse + sanitize + insert) + manual refresh endpoint.
5. Poller loop + prune job + conditional GET + backoff.
6. Items list + read-toggle endpoints.
7. React shell, routing, login page, feeds page, items page, reader page.
8. Read-on-open, unread count in title, keyboard shortcuts.
9. Fetch-full-article path (JSDOM + Readability).
10. Settings page, change password.
11. Verification checklist above.
