# digest

A personal notification-digest service. It collects new messages from the
owner's own Telegram groups and new notifications from the owner's own
X/Twitter account, summarizes what's new with Claude, and delivers a single
digest every six hours. Delivery is multi-channel — SMTP email, a Telegram
TL;DR ping, and a JSON publish to a news-site ingest endpoint — with each
channel independently switchable.
Telegram collection uses the official MTProto API (Telethon). X collection
uses `twifork` (a maintained fork of the dead `twikit`, still imported as
`twikit`), an unofficial scraper driven by a cookie session — this carries
ToS and account-ban risk, which the owner has explicitly accepted.

## Architecture

```
Telegram (Telethon) ─┐                                              ┌─> Telegram TL;DR ping
X/Twitter (twifork)  │                                              │
RSS / news feeds     ├─> collectors ─> SQLite state ─> Claude ──────┼─> news site (ingest endpoint)
Reddit               │                  (items, cursors, summarize  │
Polymarket           │                    digests, deltas, + trans- ├─> SMTP email
Hacker News         ─┘                    arc keys/context)  late   │
                                                                    └─> markdown archive

Patreon (own collector) ─┐
Telegram/X/keyword       ├─> own run ─> Claude ─> its own Telegram topic
  position matches ──────┘             (never the site — see below)
```

Each run mode is designed to run on its own scheduler (systemd timers, cron —
scheduling lives outside this repo). The cadences below are what the reference
deployment uses:

| Mode | Command | Cadence | Input | Telegram topic |
|---|---|---|---|---|
| window | `python -m digest` | every 6h (00/06/12/18 UTC) | raw items | TL;DR |
| daily | `python -m digest daily` | 20:30 Budapest | that day's window digests | daily |
| weekly | `python -m digest weekly` | Sun 21:45 Budapest | the week's daily briefs | weekly |
| patreon | `python -m digest patreon` | hourly (:50) | one paid post each | patreon |
| positions | `python -m digest positions` | every 4h (:25) | raw items matching the tracked channels, accounts or keywords | positions |
| backfill/ops | `scripts/*.py` | manual | — | — |

The first three form a cascade: each rung consumes the one below, so the
daily never sees raw items and the weekly never sees anything but daily
briefs.

`patreon` and `positions` are NOT part of that cascade. Each owns its items
end to end — collected, summarized and delivered into its own topic, and
structurally excluded from the window sweep so nothing is ever covered
twice. `positions` additionally stays SILENT when the window held only
chatter (see `prompts/positions.md`), which is what lets it run every 4
hours without becoming noise.

### Run-mode arguments

```
python -m digest                       # window run, all channels
python -m digest hide:telegram         # window run, no Telegram ping
python -m digest hide:telegram,site    # window run, stored + archived only
python -m digest daily --force         # bypass the daily duplicate-fire guard
```

`hide:<channel>` (`email`, `site`, `telegram`, comma-separated) produces and
stores the digest exactly as normal — the row, its stamped items, the archive
copy and every downstream consumer are untouched — and only suppresses the
reader-facing channel. `hide:site` without `hide:telegram` is rejected at
startup: the Telegram TL;DR links to the digest's own site page, so hiding
only the site would ship a real ping pointing at a page that was never
published. It is accepted on the window path only.

Which runs use it is a scheduling decision, not this repo's. The reference
deployment splits the 6-hourly window into three timers — 06/12 UTC plain,
00 UTC `hide:telegram` (no overnight ping, but the site still carries the
night's news for morning readers), and 18 UTC `hide:telegram,site` (the
daily brief republishes that same window 30 minutes later).

`--force` is the owner's manual escape hatch past `run_daily`'s duplicate-fire
guard, for a deliberate second daily run on the same day. It is read only
alongside `daily`.

Runs as a one-shot container (`docker compose run --rm digest`) on a systemd
timer, not a long-running service.

## Quickstart (local dev)

```
uv sync
cp .env.example .env        # fill in the values below
uv run python scripts/telegram_login.py   # one-time Telethon login, Phase 1
uv run python -m digest
```

## Environment variables

Read exclusively in `config.py` — no other module touches `os.environ`.
Defaults in parentheses; **(secret)** means it should be injected at deploy
time (a secrets manager, not a committed file) and never logged.

**Core**

| Variable | Description |
|---|---|
| `TG_API_ID` / `TG_API_HASH` | Telegram API credentials from my.telegram.org. **(secret)** |
| `TG_SESSION` | Telethon user session string. **(secret)** |
| `TG_CHAT_ALLOWLIST` | Comma-separated chat/group IDs to collect from. |
| `STATE_DB_PATH` | SQLite state database (`./state.db`). |
| `ARCHIVE_DIR` | Where markdown digest copies are archived (`./archive`). |

**Collectors** — each is off unless enabled; `NEWS_FEEDS`, `PATREON_*`, `POSITIONS_TG_CHANNELS`, `POSITIONS_X_ACCOUNTS` and `POSITIONS_KEYWORDS` use an empty-means-disabled shape instead of a flag.

| Variable | Description |
|---|---|
| `X_ENABLED` | Master switch for the X collector (`false`). |
| `X_COOKIES_PATH` / `X_COOKIES` | X session cookies — supply exactly one. **(secret)** |
| `NEWS_FEEDS` | Comma-separated RSS/Atom URLs. Enables the news collector iff non-empty. |
| `REDDIT_ENABLED` | Reddit collector switch (`false`). |
| `REDDIT_SESSION_COOKIE` | Required when Reddit is enabled. **(secret)** |
| `REDDIT_SUBREDDITS` | Comma-separated names, no `r/` prefix. Required when enabled. |
| `REDDIT_POSTS_PER_SUB` | Posts pulled per subreddit per run. |
| `POLYMARKET_ENABLED` | Polymarket collector switch (`false`). |
| `POLYMARKET_API_BASE` / `POLYMARKET_PROXY_KEY` | Endpoint and its key. **(secret)** |
| `POLYMARKET_TOP_N` / `POLYMARKET_SWING_THRESHOLD` | How many markets, and the probability move that makes one notable. |
| `HACKERNEWS_ENABLED` / `HACKERNEWS_TOP_N` | Hacker News front-page collector (`false`) and how many front-page stories per run (`15`, range 1–30). No auth. |
| `PATREON_CAMPAIGN_ID` / `PATREON_SESSION_COOKIE` | Patreon collector, used only by the `patreon` run mode. Enabled iff **both** are set — setting exactly one is a startup error, not a half-disabled collector. The cookie is the owner's own `session_id` for a paid account. **(secret)** |
| `POSITIONS_TG_CHANNELS` | Telegram usernames (no `@`) claimed by the positions tracker instead of the window digest. |
| `POSITIONS_X_ACCOUNTS` | X screen names (`@` optional) claimed the same way. **Requires post notifications (the bell) enabled for each account in the X app** — the collector only fetches an account's posts when a notification names it. |
| `POSITIONS_KEYWORDS` | Free-text terms (min 4 chars) that claim an item from **any** source, so the project's news is pulled out of general channels and feeds too. Keep them distinctive — cashtags and proper names, never a bare `ASI`/`FET`; an over-broad term can make unrelated stories vanish from every channel. |

**Summarization**

| Variable | Description |
|---|---|
| `ANTHROPIC_MODEL` | Model for headless summarization (`claude-opus-5`). |
| `CLAUDE_TIMEOUT_SECONDS` | Per summarize call (`300`; the deployment sets 600). |
| `CLAUDE_EFFORT` | `low`/`medium`/`high`/`xhigh`/`max` (`high`). |
| `TRANSLATE_HU_ENABLED` | Hungarian translation pass (`false`). |
| `TRANSLATE_MODEL` / `TRANSLATE_MODEL_FALLBACK` | Primary (`sonnet`) and the model retried on a safeguards refusal (`claude-sonnet-4-6`; empty disables the retry). |
| `TRANSLATE_TIMEOUT_SECONDS` | Per translate leg (`300`) — **applies to each leg, so the worst case is 2×**. |
| `VERIFY_DAILY_ENABLED` | Web-verification pass over the daily brief (`false`). |
| `VERIFY_DAILY_TIMEOUT_SECONDS` / `VERIFY_DAILY_MAX_WEB_OPS` | Its budget (`600`) and its self-policed tool-call guidance (`20`). |
| `VERIFY_DAILY_MODEL` / `VERIFY_DAILY_EFFORT` | Default to `ANTHROPIC_MODEL` / `CLAUDE_EFFORT`. |
| `CONTEXT_ENABLED` | Story-arc context primers, generated after the daily brief ships (`false`). |
| `CONTEXT_MAX_PER_RUN` / `CONTEXT_MODEL` / `CONTEXT_TIMEOUT_SECONDS` | Primer bounds: calls per daily run (`3`), model (`sonnet`), timeout (`120`). |

**Delivery** — the app refuses to start with every channel disabled.

| Variable | Description |
|---|---|
| `EMAIL_ENABLED` | Email channel (`true`). |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` | iCloud SMTP (`smtp.mail.me.com`, `587` STARTTLS). |
| `SMTP_PASSWORD` | iCloud app-specific password. **(secret)** |
| `DIGEST_FROM` / `DIGEST_TO` / `DIGEST_FROM_NAME` | Envelope fields (`Digest`). |
| `SITE_PUBLISH_URL` / `SITE_INGEST_KEY` | News-site ingest endpoint and its key. **(secret)** |
| `SITE_PUBLIC_BASE` | Public base URL used to build reader-facing links. |
| `ARC_KEYS_SITE_ENABLED` | Send story-arc keys with the site payload (`true`). |
| `TELEGRAM_NOTIFY_BOT_TOKEN` | Bot token for the TL;DR ping. **(secret)** |
| `TELEGRAM_NOTIFY_CHAT_ID` | Target chat/group. |
| `TELEGRAM_NOTIFY_THREAD_ID` | Forum topic for window digests. |
| `TELEGRAM_DAILY_THREAD_ID` / `TELEGRAM_WEEKLY_THREAD_ID` | Separate topics for the daily and weekly briefs; unset means they land in the window topic. |
| `TELEGRAM_PATREON_THREAD_ID` / `TELEGRAM_POSITIONS_THREAD_ID` | Same, for the Patreon posts and the positions tracker. `0` is a real value (post to the group root), so unset is the only "not configured" state. |

**Operations**

| Variable | Description |
|---|---|
| `STALE_BACKLOG_WARN_HOURS` | Age at which an unsummarized item makes a run log a `stale_backlog` WARNING naming its source (`24`). Report-only — it never fails a run. Detects a source that has stopped draining, which is otherwise invisible because the run still succeeds. |

## Container

```
docker compose build          # build the image locally
docker compose run --rm -it --entrypoint claude digest   # ONE-TIME: /login (Max subscription)
docker compose run --rm digest   # one-shot run (no daemon, no ports)
```

The interactive `claude` step is required once per fresh `digest-data`
volume: the summarizer authenticates via the persisted subscription login
in `CLAUDE_CONFIG_DIR` (no API key), and a brand-new volume has no
credentials — without the login, the first run that collects any items
fails at summarization. Complete `/login` in the prompt, exit, and the
credentials persist in the volume for every later run.

Local state (SQLite db, markdown archive, Claude CLI config dir) lands in
the named Docker volume `digest-data`, mounted into the container at
`/data`. A named volume is used instead of a `./local-data:/data` bind mount
because Docker initializes a named volume's contents (and ownership) from
the image on first run: `/data` in the image is already chowned to the
container's uid-1000 (non-root) user, so the volume is writable from the
very first `docker compose run`. A bind mount, by contrast, would have
Docker auto-create the host directory as root on first run, which shadows
that chown and leaves the uid-1000 process unable to write — silently
breaking the container. `compose.yml` here is for local dev; a production
deployment would pin a published image tag in its own compose file.

To inspect the archive without a shell in the running container:

```
docker compose run --rm --entrypoint ls digest -la /data/archive
```

or copy files out via a temporary container:

```
docker run --rm -v digest-data:/data -v "$PWD":/backup busybox \
  cp -r /data/archive /backup/
```

To reset all local state (start clean, e.g. after a schema change):

```
docker volume rm digest-data
```

(`compose.yml` pins the volume's literal name via `name: digest-data`, so
it is *not* project-prefixed — the commands above address the exact volume
the service uses).

## Deploying it yourself

This repo is the application only. A production deployment needs, outside
this repo: a scheduler firing each run mode (systemd timers work well —
one oneshot service + timer per mode, exit code as the health signal), a
pinned image (build via `docker build`, or wire your own registry push on
tag), secret injection at deploy time (never a committed env file), and a
one-time interactive `claude` login persisted in the state volume (see
Container above).

Two operational notes worth stealing: serialize all run modes behind one
lock (e.g. `flock`) so two modes never share the Telethon session
concurrently, and alert on the process exit code rather than parsing logs
— every run ends with a structured JSON `run_summary` line for debugging,
but 0/1 is the only health contract.

## Origin

Code comments and docstrings cite `PLAN.md` and `CLAUDE.md` section numbers
throughout — those are the source project's internal design/plan documents,
not included in this mirror. Read the citations as "this was a deliberate,
documented decision", with the reasoning usually restated in the comment
itself.

Extracted from a personal deployment that runs all five modes on systemd
timers with the Telegram ping and news-site channels live. History was not
carried over. The X collector's ToS risk, the cookie-based Reddit/Patreon
collectors, and the prompt files' editorial choices all reflect that
single-owner origin — read them as a working reference, not a product.
