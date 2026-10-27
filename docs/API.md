# apicraft API documentation

> Source: https://opencraft.cc/docs/ . Machine-readable schema: https://api.opencraft.cc/openapi.json . Index for AI assistants: https://opencraft.cc/llms.txt

# apicraft API

A small REST API that tells you whether a version is outdated and answers a few questions around a release. It runs on your own GitHub token, so your limits are yours alone.

> **Try it live**
>
> Use the request runner next to (or below) the text, or open the [Swagger UI](https://api.opencraft.cc/swagger/) for the full OpenAPI schema. Both need your API key.

It was built for the storytold Craft applications behind ArtCraft (WordCraft, GridCraft, PhotoCraft and others), but it works for any repository your token can read.

## Use these docs with an AI

Give an assistant one of these instead of this page. They are plain text, always match the live API and need no scraping.

| Resource | Address | Use it for |
| --- | --- | --- |
| llms.txt | [`/llms.txt`](https://opencraft.cc/llms.txt) | A short index plus the rules an agent must follow. Start here. |
| Docs as Markdown | [`/docs.md`](https://opencraft.cc/docs.md) | The whole reference in one file. Add `?section=assets` for a single part, `?download=1` to save it. |
| OpenAPI schema | [`/openapi.json`](https://api.opencraft.cc/openapi.json) | The machine-readable contract, for generating tools or function definitions. |
| Status as Markdown | [`/status.md`](https://opencraft.cc/status.md) | Whether the service is up before a long job. |

### Prompt to give your assistant

Paste this into a chat, a system prompt or an agent configuration. Replace `YOUR_KEY` with your key from the dashboard, or let the assistant ask you for it.

```
You can use the apicraft API to check software versions on GitHub.
Base URL: https://api.opencraft.cc
First read https://opencraft.cc/llms.txt, then https://opencraft.cc/docs.md if you need detail.

Rules:
- Send "Authorization: Bearer YOUR_KEY" on every /api/v1 request. Never guess or invent a key.
- HTTP 429 with a GitHub rate-limit message means the GitHub quota is used up (60 per hour without a saved token, 5,000 with one): tell me to save a GitHub token on the dashboard, do not retry in a loop.
- HTTP 429 means a limit is used up: wait for the Retry-After header, then continue.
- To check many repositories use POST /api/v1/check with up to 25 items instead of many single calls.
- Report what the API returned. Do not invent versions, release dates or download links.
```

### Rules an agent should follow

- **Authenticate every call.** The key goes in `Authorization: Bearer ac_…`. The only keyless endpoints are `/health`, `/public/*`, the badge and the docs files above.
- **Read errors literally.** Errors are JSON with `error`, `detail` and `status_code`; `detail` says what to do next.
- **Mind the limits.** 10 requests a second, 80 a minute, 4,000 a day per account. `GET /api/v1/usage` shows what is left before a large job.
- **Compare versions with the API.** It ignores a leading `v` and tag prefixes such as `app-v`. `latest` echoes the raw tag, so show it as given.
- **Ask before saving anything.** `POST` and `DELETE` on `/api/v1/watchlist` change the user's tracked list. Everything else is read-only.

### Typical jobs

| Goal | Call |
| --- | --- |
| Is my version outdated? | `GET /api/v1/check?owner=O&repo=R&current=1.2.3` |
| Check a whole list | `POST /api/v1/check` with `{"repos":[{"owner","repo","current"}]}` |
| Which file do I download? | `GET /api/v1/assets?owner=O&repo=R&os=windows&arch=x86_64`, then use `best.url` |
| What changed since my version? | `GET /api/v1/changelog?owner=O&repo=R&current=1.2.3`, read `markdown` and `breaking_changes` |
| Is anything unreleased on main? | `GET /api/v1/repos/O/R/compare`, read `ahead_by` |
| How is a project doing? | `GET /api/v1/repos/O/R/overview`, `/pulls`, `/issues` |
| Is the service up? | `GET /health` or `GET /status.md` |

## Quickstart

1. [Log in or sign up](https://opencraft.cc/login/) with your email and a password. The same form creates your account the first time.
2. On your [dashboard](https://opencraft.cc/account/), save a GitHub token. A fine-grained token with no permissions is enough for public repositories.
3. Copy your API key from the dashboard and call the API:

```
curl -H "Authorization: Bearer YOUR_KEY" \
  "https://api.opencraft.cc/api/v1/check?owner=storytold&repo=artcraft&current=0.1.0"
```

You get back the newest version, whether yours is outdated and a link to the release page.

## Limits

Every request needs an API key, and every key belongs to an account. There is no anonymous API access and no shared quota. Two limits apply:

| What | Limit | Notes |
| --- | --- | --- |
| Requests to apicraft | 10/s · 80/min · 4,000/day | Per account. Shown live on your dashboard and in `/api/v1/usage`. |
| Batch checks | 15/min · 25 repos each | `POST /api/v1/check`. A call that fans out counts as **one request per 5 repositories** (25 repositories = 5), so a batch cannot sidestep the per-minute limit. The same rule applies to `PUT /api/v1/watchlist`. |
| Keys that fail | 30/min · 200/hour per IP | Wrong, revoked or missing keys are throttled per address, so keys cannot be guessed. Honest clients never come near it. |
| GitHub requests | 5,000/hour | Your own GitHub token's allowance. Results are cached for 5 minutes, so repeated questions cost nothing. |

Every key you create (the main one and any [automation key](#auth-keys)) shares the account's limits, and so does the website while you are logged in. Limits are fixed windows per account; the response headers below always tell you what is left, so a script can pace itself instead of hitting 429. GitHub has its own rules on top (5,000 requests an hour per token, at most 100 concurrent): apicraft caches answers for 5 minutes and caps the calls it has in flight for you.

The homepage list of versions is the one place the server uses its own GitHub token: a background job refreshes it every 30 minutes and serves it from memory. Nothing you do can use that token up.

## Rate limit headers

Every API response tells you where you stand.

| Header | Example | Meaning |
| --- | --- | --- |
| X-RateLimit-Limit | `10` | Calls allowed in the tightest active window. |
| X-RateLimit-Remaining | `8` | Calls left before HTTP 429. |
| X-RateLimit-Reset | `1` | Seconds until that window resets. |
| X-RateLimit-Policy | `10;w=1, 80;w=60, 4000;w=86400` | Your whole budget: calls per window, the window in seconds. |
| Retry-After | `18` | Only on 429: seconds to wait. |
| X-Request-ID | `3f9a1c07d2e84b16` | On every response. Send your own (8 to 64 letters, digits, `.-_`) to trace a call end to end; it is echoed back and appears in error bodies. |
| ETag | `"9c1b…"` | On JSON GETs. Send it back as `If-None-Match` to get an empty `304`, see [caching](#caching). |

## Accounts and keys

An account is an email and a password, and logging in and signing up are one step ([`/auth/access`](#auth-access)). One mailbox holds one account: upper case, `+tags` and (for Gmail) dots do not create a second one. Once the server can send email, every log in, sign-up, password reset, email change and account deletion asks for a 6-digit code (`482 913`) that arrives **only by email**; until then the address is only checked for shape. On a development computer, accounts ending in `.test` accept `000000` and mails are saved to a folder instead of sent; a live server never does that.

- The **API key** (`ac_…`) authenticates your calls. Send it as `Authorization: Bearer ac_…` or `X-API-Key: ac_…`. Regenerate it any time; the old one stops working at once.
- The **session cookie** is what the website uses after you log in. It works for `/api/` calls from the browser (the docs runner and Swagger use it) and for the `/auth/` endpoints. Deleting an account needs this cookie plus your password and cannot be done with a key.

> **Security**
>
> Passwords are stored as salted scrypt hashes bound to a server secret. Failed log-ins are rate limited per IP and per mailbox, and the session cookie is HTTP-only and same-site strict. Your API key and GitHub token are encrypted at rest with AES-256-GCM.

## Your GitHub token

apicraft reads GitHub for you. Public repositories work without a token of yours, on GitHub's anonymous limit of 60 requests per hour; a saved token raises that to 5,000 per hour and lets you read private repositories. Save it once on the [dashboard](https://opencraft.cc/account/) (or with [`PUT /auth/github-token`](#auth-gh)). GitHub is asked first and a token it rejects is refused.

1. On GitHub open **Settings, Developer settings, Personal access tokens, Fine-grained tokens** and choose **Generate new token**. The dashboard has a link that pre-fills the form.
2. Name it (for example `apicraft`) and set an expiry.
3. For repository access pick **Public repositories (read-only)**. To track private repositories, choose those repositories and grant `Contents: Read-only` and `Metadata: Read-only`.
4. Paste the token (`github_pat_…`, `ghp_…`, `gho_…` and `ghu_…` are accepted) into the dashboard.

Without a saved token, public repositories are checked on GitHub's anonymous limit (60 requests per hour) and a call that needs a token of your own answers `403 github_token_required`. If GitHub revokes or expires the token, the dashboard shows it as *rejected* and you save a new one. What you see through your token is cached under your token only, so private data is never shown to anyone else.

## API key format

Keys are memorable and self-checking, so typos are caught before any lookup.

```
ac_otter-falcon-482913-a9f3c71be2d04a553d84fa12
```

| Part | Example | Purpose |
| --- | --- | --- |
| Prefix | `ac_` | Namespace that secret scanners can match. |
| Two animals | `otter-falcon` | Easy to recognise in a list of keys. |
| Six digits | `482913` | Random, for quick visual comparison. |
| Entropy | `a9f3c71be2d04a55` | 16 hex characters (64 bits) from a CSPRNG. |
| Checksum | `3d84fa12` | CRC32 of everything before it, checked in constant time. |

**GET `/api/v1/check` (API key)**

## Check one repository

Compares a version string against the newest release of a single repository.

| Parameter | Type |   | Description |
| --- | --- | --- | --- |
| owner | string | required | GitHub owner or organisation, for example `storytold`. |
| repo | string | required | Repository name, for example `artcraft`. |
| current | string | required | Version you run. A leading `v` is ignored, and so is a tag prefix such as `app-v` when comparing. |
| include_prereleases | boolean | optional | Count alpha, beta and rc releases. Default `false`. |

```
{
  "owner": "storytold",
  "repo": "artcraft",
  "current": "0.1.0",
  "latest": "artcraft-v0.41.0",
  "is_outdated": true,
  "release_url": "https://github.com/storytold/artcraft/releases/tag/artcraft-v0.41.0",
  "published_at": "2026-09-26T11:14:22Z",
  "prerelease": false
}
```

**POST `/api/v1/check` (up to 25 repos)**

## Check many repositories

Checks up to 25 repositories in parallel. A repository that fails returns its own `error` and does not abort the batch.

```
{
  "repos": [
    { "owner": "storytold", "repo": "wordcraft", "current": "0.8.0" },
    { "owner": "storytold", "repo": "gridcraft", "current": "0.8.0" }
  ]
}
```

Sending more than 25 returns `413`. Batches also have their own per-minute limit (see Limits).

**GET `/api/v1/latest` (API key)**

## Latest release

Returns the newest release tag and publication date without comparing versions. Query: `owner`, `repo` and optional `include_prereleases`.

**GET `/api/v1/repos` (API key)**

## Repository list

The catalog of repositories shown on the homepage, with display names, the software each one replaces, a short description and icon paths. It makes no GitHub request.

**GET `/api/v1/assets` (API key)**

## Downloads by OS and CPU

Lists the files attached to the newest release and tags each with the OS and CPU it is for. Give an `os` to get a `best` match: an exact CPU match first, then a universal build. Checksum and signature files are left out.

| Parameter | Type |   | Description |
| --- | --- | --- | --- |
| owner, repo | string | required | The repository. |
| os | enum | optional | `macos`, `windows`, `linux`, `freebsd` or `wasm`. |
| arch | enum | optional | `x86_64`, `aarch64` (or `arm64`), `riscv64`, `x86` or `universal`. |
| include_prereleases | boolean | optional | Use the newest pre-release if it is newer. |

Each asset has `name`, `kind` (msi, dmg, deb, appimage and so on), `os`, `arch`, `size`, `download_count`, `sha256` when GitHub reports it, and the direct `url`.

**GET `/api/v1/changelog` (API key)**

## Changelog since your version

Collects the notes of every release newer than `current`, so a jump over several versions does not hide the middle ones. `breaking_changes` is true when any notes mention breaking changes, migrations or deprecations. Query: `owner`, `repo`, `current`, `include_prereleases` and `limit` (1 to 50, default 20).

You get `entries[]` (tag, name, date, url, `breaking`, body) and one combined `markdown` string, newest first.

**GET `/api/v1/repos/{owner}/{repo}/overview` (API key)**

## Repository overview

Stars, forks, open issues, language, licence, last push and the newest release in one call.

**GET `/api/v1/repos/{owner}/{repo}/pulls` (API key)**

## Pull requests

Recently updated pull requests. Query: `state` (`open`, `closed`, `all`) and `limit` (1 to 50).

**GET `/api/v1/repos/{owner}/{repo}/issues` (API key)**

## Issues

Recently updated issues with labels and comment counts. GitHub mixes pull requests into its issue list, so they are filtered out here. Same `state` and `limit` query.

**GET `/api/v1/repos/{owner}/{repo}/compare` (API key)**

## Unreleased changes

Compares two refs. With no parameters it compares the latest release tag with the default branch, so `ahead_by` is the number of commits merged since the last release. Optional `base` and `head` take a tag, branch or commit. The answer includes `status`, `behind_by`, `files_changed` (GitHub lists at most 300), a `diff_url` and the 10 newest commits.

**GET `/api/v1/watchlist` (API key)**

## Tracked repositories

Track up to 50 repositories (the owner account 200), including private ones your token can read, together with the version you run. Every entry can carry a `label` and a `group` (a *track* such as `prod` or `tools`), and is checked against its newest release when you list it. The dashboard is a front end for this; a `read` [key](#auth-keys) can use the GETs, a `full` key or the website the rest.

| Call | What it does |
| --- | --- |
| GET /api/v1/watchlist | Your list with `latest`, `is_outdated` and `release_url` for each entry. Filters: `?group=prod` and `?only=outdated` (or `current`, `error`). |
| GET /api/v1/watchlist/summary | Counts instead of a list: `total`, `up_to_date`, `outdated`, `errors`, per-group numbers and the outdated entries. `all_current` is a single boolean for a pipeline gate. Accepts `?group=`. |
| POST /api/v1/watchlist | Add or update one. Body: `{"owner":"storytold","repo":"artcraft","current":"0.1.0","prereleases":false,"label":"Editor","group":"prod"}`. A repository your token cannot see is refused. |
| PUT /api/v1/watchlist | Add or update many (up to 200). Body: `{"items":[…same objects…],"replace":true}`. With `replace` everything not in `items` is removed, so a file in your repository can be the source of truth. Failing repositories come back in `rejected` with the reason; the rest is stored. |
| POST /api/v1/watchlist/{owner}/{repo}/acknowledge | Set the version you run to the newest release, for the moment after a deployment. |
| DEL /api/v1/watchlist/{owner}/{repo} | Stop tracking one repository. |

**GET `/api/v1/badge/{owner}/{repo}.svg` (No key)**

## Version badge

A live SVG badge for a README. Add `?current=0.1.0` to show whether that version is up to date. Badges are served from the homepage cache, so they cost no GitHub request, need no key and only know the repositories shown on the homepage (anything else says "not tracked").

```
![release](https://api.opencraft.cc/api/v1/badge/storytold/artcraft.svg)
```

**GET `/api/v1/usage` (API key)**

## Limits and usage

Shows your limits, the remaining quota in the tightest window and your last seven days of requests.

```
{
  "tier": "user",
  "limits": { "general": "10/second;80/minute;4000/day", "batch": "15/minute", "batch_size": 25 },
  "rate_limit": { "limit": 10, "remaining": 8, "reset_seconds": 1 },
  "usage": { "today": 12, "last7": [ { "day": "2026-10-26", "requests": 12 } ] }
}
```

**GET `/api/v1/usage/history` (API key)**

## Request history

Every call your account makes is stored (time, endpoint, query, status, duration) for 90 days, and the daily totals are kept for good. The [dashboard](https://opencraft.cc/account/) charts this; the endpoint gives you the same data. Reading the history is not itself logged.

| Parameter | Type |   | Description |
| --- | --- | --- | --- |
| days | integer | optional | Window, 1 to 90. Default 30. |
| limit | integer | optional | How many of the newest calls to list, 1 to 200. Default 50. |
| via | enum | optional | `all` (default), `apikey` (made with your key) or `session` (made by the website for you). |
| format | enum | optional | `json` (default) or `csv`, which downloads the raw rows of the window (up to 100,000). |

```
{
  "days": 30, "retention_days": 90, "stored_since": "2026-10-20T09:14:02+00:00",
  "totals": { "today": 12, "last7": 301, "last30": 1288, "all_time": 4410 },
  "daily": [ { "day": "2026-10-26", "requests": 12, "errors": 1, "apikey": 9 } ],
  "endpoints": [ { "path": "/api/v1/check", "count": 640, "errors": 4, "avg_ms": 212, "p95_ms": 480 } ],
  "statuses": { "200": 1270, "403": 14, "429": 4 },
  "hours": [ 0, 0, 3, 12, "… 24 numbers, UTC" ],
  "latency": { "avg": 205, "p50": 160, "p95": 480, "max": 1930 },
  "sources": [ { "name": "nightly-ci", "via": "apikey", "count": 610, "errors": 2, "last_at": "2026-10-26T03:00:12+00:00" },
               { "name": "Website", "via": "session", "count": 120, "errors": 0, "last_at": "2026-10-26T18:20:11+00:00" } ],
  "recent": [ { "time": "2026-10-26T18:20:11+00:00", "via": "apikey", "key": "nightly-ci", "method": "GET",
                "path": "/api/v1/check", "query": "owner=storytold&repo=artcraft&current=0.1.0", "status": 200, "ms": 187 } ]
}
```

**GET `/health` (No key)**

## Health and status

Three public endpoints, none of which needs a key:

| Endpoint | Use |
| --- | --- |
| GET /health | Plain liveness probe for load balancers and uptime monitors. Not rate limited. |
| GET /public/status | The full picture behind the [status page](https://opencraft.cc/status/): each component's state, 90 days of uptime per day and recent incidents. |
| GET /public/showcase | The newest version of every homepage repository, refreshed every 30 minutes. |

```
{ "status": "ok", "service": "apicraft", "accounts": true }
```

## Craft data: activity, verdicts and contributors

Besides your own repositories, apicraft knows the Craft apps. Three modules collect public GitHub data in the background and serve it from memory, so these calls **cost no GitHub request** and nothing you do can use anybody's quota up. They count as normal calls against your [account limits](#limits). The same payloads are available without a key under `/public/craft/…`; that is what the [Craft Watch page](https://opencraft.cc/craft/) uses.

| Module | What it gives you | Needs |
| --- | --- | --- |
| Craft Watch | Update verdicts, the alive meter, version check, advisories, compare, events, Atom feed, badge. | The server's `GITHUB_TOKEN`, used by a background job only. |
| Contributors | Leaderboard, per-person detail, daily and weekly charts, time-of-day heat map, time to merge, SonarCloud quality. | The same background token. |
| CraftSuite tracker | "Ready for real work" percentages with weekly history. | Nothing, but it is **off by default**; the owner switches it on (Admin page or `CRAFTSUITE_ENABLED=true`). |

A module the owner switched off answers `404 module_disabled`; one that is still collecting its first data answers `503 warming_up` with `Retry-After`. `GET /api/v1/craft/status` shows what is on, when each module last ran and how many GitHub requests the server token had left.

> **Signals, not a guarantee**
>
> Everything here is based on public GitHub records. Verdicts say what we saw ("A change mentions GHSA-…", "We found no published security advisories for this version") and never call a version safe or unsafe. Unofficial, not affiliated with ArtCraft or storytold.

### Credits

- **Craft Watch** by **Overprivileged**: the collector design, the plain-English verdicts, the wording rules, the alive meter and the "is my version fine" idea. Ported to Python here.
- **[ArtCraft Contributors](https://artcraft.apollonet.dev)**: the pull request and commit sync, the leaderboard, the charts and the quality numbers. Ported to Python here; its GitHub sign-in is not.
- **[CraftSuite (craftsuite.net)](https://craftsuite.net/tracker)**: the Crafting Apps replacement tracker, data under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). If you show or redistribute the readiness numbers, credit CraftSuite with a link; every response of the tracker carries the text to use in `attribution`.

`GET /api/v1/craft/credits` returns the same list as JSON.

**GET `/api/v1/craft/apps` (API key)**

## Apps, verdicts and compare

| Call | What it does |
| --- | --- |
| GET /api/v1/craft/apps | Every watched app: `latest`, `verdict` (`colour` green, yellow, red or grey, a `title` such as "Looks calm", "Wait a day", "Update now", and the sentence), the `alive` numbers (merges in 7 and 30 days, commits, releases, days since the last release, issues opened and closed, `pulse`: busy, active or quiet), `advisories_30d`, `scorecard`, `ai_tagged_pct` and, when the tracker module is on, `ready`. |
| GET /api/v1/craft/apps/{slug} | One app in detail: the verdict with its evidence, four stat cards, recent merged changes, releases, advisories, open problems and most wanted features (ranked by thumbs up), the report-spike check and a 30-day history. `slug` is the repository name in lower case, for example `photocraft` or `craft-launcher`. |
| GET /api/v1/craft/apps/{slug}/version?tag=… | What the public records show for the version you run: newer releases that mention hardening or name an advisory, open advisories, merged changes not released yet. Without `tag` it checks the newest. The answer also lists the known `releases`. |
| GET /api/v1/craft/compare?sort=… | All apps side by side plus the app that leads each measure. `sort` is `busiest`, `alive`, `fastest_fix`, `most_stable`, `most_ready`, `newest_release` or `stars`. |
| GET /api/v1/craft/status | Which modules run, last run, data age, remaining server quota, credits. |

**Verdict rules.** *Red, update now*: a merged change that names a GHSA advisory shipped in a release this week. *Yellow*: a change naming another advisory shipped, or at least 3 crash-like issue reports (and twice the usual) name the newest release in its first 48 hours. *Green*: nothing found. *Grey*: no release in the data. The thresholds for `pulse`: 10 or more merges in 7 days is busy, at least one is active, none is quiet.

```
curl -s "https://api.opencraft.cc/api/v1/craft/apps/photocraft/version" -H "Authorization: Bearer $APICRAFT_KEY" | jq '.verdict'
# { "colour": "green", "title": "Latest in our data", "text": "0.16.0 is the latest release in the data. We found no published security advisories for this version. Based on public GitHub records only. Signals, not a guarantee." }
```

**GET `/api/v1/craft/events` (API key)**

## Events, feed and badge

A newest-first feed across all apps for automations that react to new releases or published advisories. Parameters: `days` (1 to 30, default 7), `since` (ISO time; only newer events), `types` (comma list of `release`, `advisory`, `security_change`, `merged`), `app` (slug) and `limit` (1 to 200). `security_change` is a merged change whose title or labels mention hardening or an advisory. Poll with `since` set to the newest event you handled and `If-None-Match` for a body-less 304.

```
curl -s "https://api.opencraft.cc/api/v1/craft/events?types=release,advisory&days=3" -H "Authorization: Bearer $APICRAFT_KEY" | jq -r '.events[] | "\(.time)  \(.name)  \(.title)"'
```

- **Atom feed**, no key (feed readers cannot send headers): `https://api.opencraft.cc/public/craft/feed.atom` lists releases, advisories and hardening changes of the last 30 days.
- **Badge**, no key: `![photocraft](https://api.opencraft.cc/api/v1/badge/craft/photocraft.svg)` shows the newest version and the verdict, coloured. It turns grey with "not available" when the module is off.

**GET `/api/v1/craft/contributors` (API key)**

## Contributors

After [ArtCraft Contributors](https://artcraft.apollonet.dev), ported into apicraft. The [Contributors page](https://opencraft.cc/craft/contributors/) shows the same data the way the original does: merged per day by app (click the legend to hide apps), contributors over time, time to merge, when pull requests get merged, a sortable leaderboard (all time, 30 days, 7 days) with merge rate, reverts, median merge time and the apps a person built, SonarCloud code quality, and a page per person (`#/u/login`) with rank, pull requests, a per-day chart and commits per app. Not ported: its GitHub sign-in ("my pull requests").

Public facts only (login, counts and times); bots are left out. No avatars are loaded from other sites, so the page shows initials. While the first full sync runs, `syncing_repos` is above 0 and the numbers still grow.

| Call | What it does |
| --- | --- |
| GET /api/v1/craft/contributors | The leaderboard. `sort`: `merged` (default), `commits`, `opened`, `merge_rate`, `reverted`, `repos`, `median_merge`, `recent`, `name`. `repo` limits it to one repository, `days` to merges in the last N days, `q` searches a login, `limit` and `offset` page it. |
| GET /api/v1/craft/contributors/{login} | Rank, counts, the person's pull requests (newest 300, up to 1,000 with `prs`) and commits per repository. |
| GET /api/v1/craft/stats | Everything for charts: totals, 90 days of opened and merged, weekly contributor growth, merges by weekday and hour (UTC), the time-to-merge histogram, per-repository numbers with SonarCloud quality and the latest merges. |

**GET `/api/v1/craft/tracker` (API key)**

## Readiness tracker (optional)

How close each Crafting App is to replacing the original, as the project itself estimates in its own roadmap ("ready for real work"), with the weekly history, collected by CraftSuite. It is **off by default**; until the owner switches it on you get `404 module_disabled`. The server reads CraftSuite's public `tracker.json` (fixed address, size limited, every two hours) and shows it unchanged. Per app you get `rival`, `suite` (adobe, office or more), `stage`, `ready`, the week's `change`, the `reviewed` date, the latest version, releases in 30 days and stars. `summary` adds what the tracker page shows above its table: the app closest to ready, the average per suite (Adobe replacements, Office replacements), releases in the last 30 days, apps per stage and the biggest movers; `stages` explains pre-alpha, alpha and beta. Source of the data you see: [craftsuite.net/tracker.json](https://craftsuite.net/tracker.json).

**Attribution is required** (CC BY 4.0): credit "CraftSuite (craftsuite.net)" with a link to `https://craftsuite.net/tracker`. The response carries the exact text in `attribution`.

```
curl -s "https://api.opencraft.cc/api/v1/craft/tracker" -H "Authorization: Bearer $APICRAFT_KEY" | jq '.apps[] | [.name, .ready, .stage]'
```

## Automation guide

apicraft is built to be called by scripts, CI jobs and other tools, not just people. The contract is small and stable:

- **One key per job.** Create a named [automation key](#auth-keys) for each script. Your charts split calls by key name, so you see which job is noisy, and you can revoke one without touching the others.
- **One error shape.** Every error is JSON with a stable `error` code to branch on (`github_token_required`, `rate_limit_exceeded`, `not_found` ...), see [Errors](#errors).
- **Pace yourself.** Read `X-RateLimit-Remaining` and `X-RateLimit-Policy`; on `429` wait `Retry-After` seconds, then retry.
- **Ask cheaply.** Use `GET /api/v1/watchlist/summary` instead of looping over repositories, and send `If-None-Match` when you poll.
- **Trace calls.** Send an `X-Request-ID`; it comes back on the response and inside error bodies.

### Keep a list of repositories in sync

Commit a `watch.json` next to your code and let a job make apicraft match it (needs a `full` key):

```
curl -sS -X PUT https://api.opencraft.cc/api/v1/watchlist \
  -H "Authorization: Bearer $APICRAFT_KEY" -H "Content-Type: application/json" \
  -d '{"replace": true, "items": [
        {"owner":"storytold","repo":"artcraft","current":"0.41.0","group":"prod","label":"Editor"},
        {"owner":"storytold","repo":"photocraft","current":"0.12.1","group":"tools"} ]}'
```

### Fail a pipeline when something is outdated

A `read` key is enough. `all_current` is the only field you need:

```
curl -sS "https://api.opencraft.cc/api/v1/watchlist/summary?group=prod" \
  -H "Authorization: Bearer $APICRAFT_KEY" | jq -e '.all_current'
# exit code 0 = everything current, 1 = something is outdated or failing
```

### GitHub Actions example

```
name: release-check
on:
  schedule: [{ cron: "0 6 * * *" }]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - name: Any tracked app outdated?
        env:
          APICRAFT_KEY: ${{ secrets.APICRAFT_KEY }}
        run: |
          curl -fsS "https://api.opencraft.cc/api/v1/watchlist/summary" \
            -H "Authorization: Bearer $APICRAFT_KEY" -H "X-Request-ID: gha-${{ github.run_id }}" \
            | tee summary.json | jq -e '.all_current'
```

### After you deploy

Acknowledge a single repository or an entire track (environment) in one request:

```
# Mark a single repository as updated to latest:
curl -sS -X POST https://api.opencraft.cc/api/v1/watchlist/storytold/artcraft/acknowledge \
  -H "Authorization: Bearer $APICRAFT_KEY"     # needs a full key

# Mark all outdated repositories in a track (e.g. prod) as updated at once:
curl -sS -X POST https://api.opencraft.cc/api/v1/watchlist/track/prod/acknowledge \
  -H "Authorization: Bearer $APICRAFT_KEY"
```

### CI Gate Poller & Exit Codes

`GET /api/v1/watchlist/check` returns a `status` and an explicit `exit_code` (0 for clean, 1 for updates required or failure), making shell script integration effortless:

```
# Exit directly with Apicraft's exit code:
EXIT_CODE=$(curl -sS "https://api.opencraft.cc/api/v1/watchlist/check?group=prod" \
  -H "Authorization: Bearer $APICRAFT_KEY" | jq '.exit_code')
exit $EXIT_CODE
```

### Multi-Format Watchlist Export & Strategy Matrix

Export your tracked repositories in multiple formats with `GET /api/v1/watchlist/export?format=...` (supports `json`, `csv`, `yaml`, `markdown`, or `matrix`):

```
# Generate a GitHub Actions strategy matrix dynamically:
curl -sS "https://api.opencraft.cc/api/v1/watchlist/export?format=matrix&group=prod" \
  -H "Authorization: Bearer $APICRAFT_KEY" > matrix.json
```

### Real-Time Rate Limits & Guidance

Monitor your GitHub quota and Apicraft limits with `GET /api/v1/rate_limit`, or retrieve token security requirements with `GET /api/v1/tokens/guidance`.

```
curl -sS "https://api.opencraft.cc/api/v1/rate_limit" \
  -H "Authorization: Bearer $APICRAFT_KEY" | jq .github_rate_limit
```

> **Good to know**
>
> Keys cannot manage keys, change the password or delete the account; those are browser-session only on purpose, so a leaked key can do little harm. A `read` key can still send batch checks and CI exit-code queries because they change nothing.

**POST `/auth/keys` (Session)**

## Automation keys

Extra keys for scripts, each with a name, an access level, an optional expiry and an optional daily cap. Create, edit, rotate and delete them on the [dashboard](https://opencraft.cc/account/) or with these calls (browser session only). You can have up to 10. The dashboard also shows ready-to-paste snippets (curl, GitHub Actions, `.env`) when you create or rotate one.

| Call | What it does |
| --- | --- |
| POST /auth/keys | Body: `{"name":"nightly-ci","scope":"read","expires_in_days":90,"daily_limit":500}`. `scope` is `read` (default; every GET plus the batch check), `craft` (only `/api/v1/craft/`, the badge and your usage) or `full` (also changes your tracked repositories). `expires_in_days` (1 to 365) and `daily_limit` are optional. The response contains the key **once**; only a hash is stored. |
| PUT /auth/keys/{id} | Rename a key, change its access, set (`N`) or remove (`0`) its expiry, set or remove (`0`) its daily cap. The secret stays. |
| POST /auth/keys/{id}/rotate | A new secret for the same key; name, access, expiry and cap stay and the old secret stops working at once. Shown once. Use it for scheduled key rotation. |
| GET /auth/keys | Your keys with name, access, the start of the key, creation time and last use. Never the key itself. |
| DEL /auth/keys/{id} | Revoke one. It stops working within seconds. |

A read-only key that tries to change something gets `403 key_read_only`; a `craft` key outside the Craft data gets `403 key_scope`; a key past its daily cap gets `429 key_limit_reached` with `Retry-After` (other keys keep working); an expired key gets `401 invalid_api_key` saying when it expired. All keys, and the website, share your account's [limits](#limits) and your GitHub token. The dashboard charts each key by name.

## Caching with ETag

Every JSON `GET` under `/api/v1/` carries an `ETag`. Send it back and, while nothing changed, you get `304 Not Modified` with no body (it still counts as one request, but saves bandwidth and parsing):

```
curl -si "https://api.opencraft.cc/api/v1/watchlist" -H "Authorization: Bearer $KEY" -H 'If-None-Match: "9c1b2f7a0d3e44a1b6c8"'
# HTTP/1.1 304 Not Modified
```

GitHub answers are also cached on the server for 5 minutes per token, so polling more often than that only repeats the same answer.

**POST `/auth/access` (Browser session)**

## Log in or sign up

One call for both. If the email has an account, the password is checked and you are logged in. If not, an account is created with that password and you are logged in. The website's login page is a front end for this call.

| Field | Type | Rules |
| --- | --- | --- |
| email | string | A valid address. One account per mailbox: upper case, `+tags` and Gmail dots do not make a second one. |
| password | string | For a new account 10 to 128 characters and not equal to the email. For an existing one, whatever you set. |
| code | string | optional The 6-digit code, for example `482 913`. Only used while email codes are on. |

```
curl -X POST "https://api.opencraft.cc/auth/access" \
  -H "Content-Type: application/json" -c cookies.txt \
  -d '{"email":"you@example.com","password":"correct-horse-battery"}'
```

A wrong password gives `401`; a new account with a weak password gives `422`. Limits: 30 attempts per hour per IP and 20 per hour per mailbox.

### Email codes (switched off for now)

When the server turns email codes on, every log in and every sign-up needs a code from the user's mailbox. The password is checked first, so nothing is mailed for a wrong password:

1. Call `/auth/access` with email and password. The answer is `{"code_sent": true, "expires_in": 600, "resend_after": 60}` and a code like `482 913` is emailed.
2. Call it again with the same email and password plus `code`. The session cookie is set and, for a new address, the account is created.

Codes work once, expire after 10 minutes and allow 5 attempts. Calling without `code` again after the 60 second cooldown sends a fresh one. `GET /auth/options` (no key) tells a client whether codes are on.

**POST `/auth/password/forgot` (Switched off)**

## Forgot password

Two calls reset a forgotten password. Both answer `503` while email codes are off, which is the case for now.

| Call | What it does |
| --- | --- |
| POST /auth/password/forgot | Body `{"email":"you@example.com"}`. Emails a 6-digit code. The answer is identical whether or not the address has an account. |
| POST /auth/password/reset | Body `{"email","code","password"}`. Checks the code, sets the new password (10 or more characters), signs out every other session and logs you in. |

**GET `/auth/me` (Session)**

## Profile and key

Returns your email, API key, limits, whether a GitHub token is saved and the 7-day usage history. Needs the session cookie; an API key alone is not enough.

**PUT `/auth/github-token` (Session)**

## GitHub token

Three calls manage the token the API runs on. The token is never returned.

| Call | What it does |
| --- | --- |
| PUT /auth/github-token | Body `{"token":"github_pat_…"}`. GitHub is asked first; a token it rejects is refused with `422`. |
| GET /auth/github-token | Whether a token is saved, whether GitHub still accepts it, its kind and how many requests it has left this hour. |
| DEL /auth/github-token | Deletes the saved token. |

**POST `/auth/key/regenerate` (Session)**

## Regenerate key

Revokes the current key immediately and returns a new one. Limit: 5 per hour.

**POST `/auth/logout` (Session)**

## Log out

Deletes the server-side session and clears the cookie.

**POST `/auth/password/change` (Session)**

## Password and sessions

The website's [Settings](https://opencraft.cc/account/settings/) page uses these two calls. Both need the browser session; an API key is refused.

| Call | What it does |
| --- | --- |
| POST /auth/password/change | Body `{"current","password"}`. Checks the current password, sets the new one (10 or more characters), signs out every session and gives this browser a fresh one. Your API key does not change. |
| POST /auth/sessions/revoke-all | Ends every login on every device, including this one. Your API key keeps working. |

> **Deleting an account is not an API call**
>
> An account can only be deleted in Settings on the website, after logging in. It asks for your password, the word `DELETE` and, while email codes are on, a code emailed to you. There is no endpoint for it in this API or in the Swagger UI, and an API key can never do it.

## Homepage repositories

The Craft applications shown on the homepage. Their versions come from the server's own refresh job, not from your quota. Icons are served from `/icons/`.

| App | Repository | Replaces |
| --- | --- | --- |
| ArtCraft Studio | `artcraft` | AI image and video |
| PhotoCraft | `photocraft` | Raster image editing |
| VectorCraft | `vectorcraft` | Vector graphics |
| WordCraft | `wordcraft` | Word processing |
| GridCraft | `gridcraft` | Spreadsheets |
| DeckCraft | `deckcraft` | Presentations |
| FilmCraft | `filmcraft` | Video editing |
| LightCraft | `lightcraft` | RAW photo developing |
| PdfCraft | `pdfcraft` | PDF signing and OCR |
| EffectCraft | `effectcraft` | Motion graphics |
| DesignCraft | `designcraft` | Desktop publishing |
| FigCraft | `figcraft` | UI prototyping |
| CadCraft | `cadcraft` | CAD drafting |
| SoundCraft | `soundcraft` | Audio workstation |

## Errors

Every error, from every endpoint, is JSON in one shape. Branch on `error` (it never changes), show `detail` to people, and follow `hint` and `action` when they are there. The website turns those into a message and a button.

```
{
  "error": "github_token_required",
  "detail": "No GitHub token saved yet. Every API call runs on your own token.",
  "status_code": 403,
  "hint": "Save a GitHub token on your dashboard. A fine-grained token with no permissions is enough for public repositories.",
  "action": { "label": "Add your GitHub token", "path": "/account/" },
  "request_id": "b07683f9b7c843ca"
}
```

Optional fields (`hint`, `action`, `retry_after`, `rate_limit_remaining`, `rate_limit_reset`) are left out when they do not apply. `request_id` equals the `X-Request-ID` header.

| Status | Code | Meaning |
| --- | --- | --- |
| `401` | unauthorized | No API key or session. |
| `401` | invalid_api_key | The key is wrong, revoked, deleted or its account is disabled. |
| `403` | github_token_required | You have not saved a GitHub token yet. Add one on the dashboard. |
| `403` | github_token_rejected | GitHub no longer accepts your saved token (expired or revoked). Save a new one. |
| `403` | key_read_only | A read-only automation key tried to change something. |
| `403` | key_scope | A Craft-data-only key was used outside `/api/v1/craft/`. |
| `429` | key_limit_reached | That automation key used its own daily cap. Raise it or use another key. |
| `403` | github_forbidden · forbidden | GitHub refused the request (the message names the missing permission), a cross-site request was refused, or the account is disabled. |
| `404` | not_found | Repository not found (or private and not visible to your token), it has no releases, or the item does not exist. |
| `409` | conflict | An account with that email exists, a name is taken, or your tracked list or key list is full. |
| `413` | payload_too_large | Batch larger than 25 repositories (owner account 100). |
| `404` | module_disabled | The owner switched that Craft module off (the readiness tracker is off by default). |
| `422` | validation_error | A field broke its rules. `detail` names the field. |
| `429` | rate_limit_exceeded | Your apicraft limit. Wait `Retry-After` seconds. |
| `429` | github_rate_limited · github_quota_exhausted | Your own GitHub token used its hourly allowance. It resets at the top of the hour. |
| `502 · 504` | github_unreachable · github_timeout · github_bad_response | GitHub did not answer properly. Retry later; see the [status page](https://opencraft.cc/status/). |
| `503` | warming_up · module_unavailable | A Craft module has not finished its first collection (wait `Retry-After` seconds), or the server lacks its `GITHUB_TOKEN`. |
| `503` | unavailable | Accounts are not enabled or email codes are off (forgot password). |

## Swagger UI

The [Swagger UI](https://api.opencraft.cc/swagger/) lists every endpoint from the OpenAPI schema ([openapi.json](https://api.opencraft.cc/openapi.json)) and lets you send requests. Press **Authorize** and paste your `ac_` key; it is remembered in your browser. If you are logged in, requests also work through your session.

## Running it yourself

Local, with Python 3.11 or newer. Accounts use a local SQLite file by default, no database server needed.

```
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
python scripts/gen_secrets.py   # paste the output into .env
uvicorn app.main:app --port 8000
```

For production, set `DATABASE_URL` to use PostgreSQL or run `docker compose up -d`, put TLS in front, and set `ALLOWED_HOSTS`, `CORS_ORIGINS` and `COOKIE_SECURE=true`. The optional `GITHUB_TOKEN` only feeds the homepage list. See `SETUP.md` for the full checklist.
