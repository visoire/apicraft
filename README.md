# apicraft

**A simple API that tells you when there is a new release.**

You build with an app, or you ship one. You want to know *"is my version outdated?"* and *"did a new release just come out?"* without watching GitHub by hand, scraping release pages or fighting rate limits. apicraft answers that with one HTTP call.

- Website and dashboard: **https://opencraft.cc**
- API: **https://api.opencraft.cc**
- Full documentation: [docs/API.md](docs/API.md) (also at https://opencraft.cc/docs/)
- Machine-readable schema: [docs/openapi.json](docs/openapi.json) (live: https://api.opencraft.cc/openapi.json)
- Live status: https://opencraft.cc/status/

> This repository holds the **public documentation, examples and the issue tracker** of apicraft. The service itself is **not open source** (see [License](#license)).

---

## How it works

1. **Create a free account** at opencraft.cc. You get an API key that looks like `ac_otter-falcon-482913-...`.
2. **Optional: save your own GitHub token** on the dashboard. With it your calls use your own 5,000 requests per hour and can read private repositories your token can see. Without it, public repositories work on GitHub's anonymous limit (60 per hour).
3. **Call the API** with your key. apicraft asks GitHub on your behalf, compares versions the smart way (tag prefixes such as `app-v` are understood), caches the answer for a few minutes and returns clean JSON.

```bash
curl -H "Authorization: Bearer YOUR_KEY" \
  "https://api.opencraft.cc/api/v1/check?owner=storytold&repo=artcraft&current=0.1.0"
```

```json
{
  "owner": "storytold",
  "repo": "artcraft",
  "current": "0.1.0",
  "latest": "0.41.0",
  "is_outdated": true,
  "release_url": "https://github.com/storytold/artcraft/releases/tag/artcraft-v0.41.0",
  "published_at": "2026-10-25T14:01:11Z"
}
```

Send the key as `Authorization: Bearer ac_...` or `X-API-Key: ac_...`. A key belongs to one account and shares that account's limits.

## What you can do with it

| Need | Call |
| :--- | :--- |
| Is my version outdated? | `GET /api/v1/check?owner=&repo=&current=` |
| Check up to 25 repositories at once | `POST /api/v1/check` |
| Newest release only | `GET /api/v1/latest` |
| Download link for my OS and CPU | `GET /api/v1/assets?owner=&repo=&os=&arch=` |
| What changed since my version | `GET /api/v1/changelog` |
| Repository facts, open pull requests and issues, unreleased commits | `GET /api/v1/repos/{owner}/{repo}/overview`, `/pulls`, `/issues`, `/compare` |
| Keep a list of repositories and ask "is anything outdated?" | `GET/POST/PUT /api/v1/watchlist`, `GET /api/v1/watchlist/summary` |
| Your limits and your full request history | `GET /api/v1/usage`, `GET /api/v1/usage/history` |
| A live version badge for your README | `GET /api/v1/badge/{owner}/{repo}.svg` (no key) |
| Craft app activity, update verdicts, events, contributors | `GET /api/v1/craft/*` (see the docs) |

### Use it in CI

```bash
curl -fsS "https://api.opencraft.cc/api/v1/watchlist/summary?group=prod" \
  -H "Authorization: Bearer $APICRAFT_KEY" | jq -e '.all_current'
# exit code 0 = everything is current, 1 = something is outdated or failing
```

Create **one named key per script** (read-only, Craft-data-only or full access, with an optional expiry and daily cap) on the dashboard, so a leaked or runaway job stays small.

## Good to know

- **Limits** per account: 10 requests per second, 80 per minute, 4,000 per day. A batch counts as one request per 5 repositories. Every response carries `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` and `X-RateLimit-Policy`, and `429` answers include `Retry-After`.
- **Caching:** GitHub answers are cached for 5 minutes per token. JSON `GET`s carry an `ETag`; send `If-None-Match` to get an empty `304` while nothing changed.
- **Errors** always have the same shape: a stable `error` code to branch on, a human `detail`, often a `hint`, an `action` and a `request_id` (also the `X-Request-ID` header). Quote the request ID when you report a problem.
- **Privacy:** your GitHub token and API key are encrypted at rest; only a hash of each key is used to look it up. Account deletion is possible only in the website's Settings, never through the API.
- **Public data only.** Verdicts and statistics say what the public records show and never call a version "safe" or "unsafe". Signals, not a guarantee.

## For AI assistants

Give your assistant https://opencraft.cc/llms.txt first, then https://opencraft.cc/docs.md or [docs/API.md](docs/API.md). They are plain text and list the rules an agent must follow (never invent a key, wait for `Retry-After` on `429`, prefer the batch call).

## Report a problem or ask for something

- **Bug?** [Open a bug report](../../issues/new?template=bug_report.yml)
- **Idea?** [Suggest a feature](../../issues/new?template=feature_request.yml)
- **Docs wrong or unclear?** [Report a docs issue](../../issues/new?template=docs_issue.yml)
- **Security problem?** Do not open an issue: see [SECURITY.md](SECURITY.md).

How issues and contributions work is in [CONTRIBUTING.md](CONTRIBUTING.md).

## License

The **service, its source code and its design are not open source** and are not published here. Copyright (c) 2026 visoire, all rights reserved. What you may do with this repository:

- read, share and quote the documentation with a link back to this repository,
- copy the **code examples** into your own projects, no credit needed,
- use the hosted API under the terms shown when you create an account.

See [LICENSE](LICENSE) for the exact wording.

## Credits and disclaimer

apicraft is an independent project by [visoire](https://github.com/visoire). It is **not affiliated with, endorsed by or sponsored by ArtCraft or storytold**; their names are used only to describe what the API reads. The Craft data features build on ideas and data from **Craft Watch** (Overprivileged), **ArtCraft Contributors** (artcraft.apollonet.dev) and the **CraftSuite** replacement tracker (craftsuite.net, CC BY 4.0, credit and link shown wherever its numbers appear).
