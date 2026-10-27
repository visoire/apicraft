# Contributing to apicraft

Thank you for helping. apicraft is a closed-source service with a public issue tracker and public documentation. This is how you can help, and what to expect.

## What is welcome

| You can | How |
| :--- | :--- |
| Report a bug in the API or the website | [Bug report](../../issues/new?template=bug_report.yml) |
| Ask for a feature or a new endpoint | [Feature request](../../issues/new?template=feature_request.yml) |
| Point out a mistake or a gap in the docs | [Docs issue](../../issues/new?template=docs_issue.yml) |
| Fix the docs yourself | A pull request that changes **only** `README.md` or `docs/` (see below) |
| Share a good example | A pull request that adds a tested snippet to `docs/` |
| Report a security problem | Privately, see [SECURITY.md](SECURITY.md). Never in a public issue. |

## What is not accepted

The service source code is not in this repository, so there is nothing to send code changes to. Pull requests that add application code, vendored third-party code or a copy of the service will be closed. Ideas for the service itself belong in a **feature request**, where they can be discussed first.

## Before you open an issue

1. Search the existing issues, open and closed.
2. Check the [status page](https://opencraft.cc/status/): the problem may already be known.
3. Read the matching part of [docs/API.md](docs/API.md), including the error codes.
4. Reproduce it with a single `curl` command if you can.

## A good bug report contains

- the exact request (method, URL, query, body) with **your API key and GitHub token removed**,
- what you expected and what you got (status code and the JSON body),
- the **`request_id`** from the error body or the `X-Request-ID` response header (this lets the issue be found in the logs),
- the time (with time zone) and, if relevant, the response headers `X-RateLimit-*`.

**Never paste an API key (`ac_...`), a GitHub token (`github_pat_...`, `ghp_...`) or a password into an issue, a comment or a pull request.** If you did, revoke it at once: regenerate your key on the dashboard and remove the token on GitHub. Issues containing secrets may be edited or deleted to protect you.

## Pull requests for the docs

- One topic per pull request, small and focused.
- Plain, friendly English. No em dashes, no marketing words, no emoji walls.
- Keep examples working: use `YOUR_KEY` as the placeholder, `https://api.opencraft.cc` as the API address and `https://opencraft.cc` for pages.
- Do not edit `docs/API.md` structure or `docs/openapi.json` by hand for new endpoints; they are generated from the service and replaced on every release. Open an issue instead and it will be fixed at the source.
- By opening a pull request you agree that your change may be used, modified and published as part of the documentation under the terms in [LICENSE](LICENSE), and that you wrote it yourself or have the right to share it.

## How issues are handled

- New issues are read, labelled (`bug`, `enhancement`, `docs`, `question`, `needs-info`) and answered as time allows. This is a one-person project, so there is no guaranteed response time.
- An issue marked `needs-info` that stays unanswered for 14 days may be closed. You can always reopen it with the missing details.
- Not every request will be built. Reasons are given when an issue is declined.
- Be kind. See the [Code of Conduct](CODE_OF_CONDUCT.md).

## Credits

apicraft builds on ideas from Craft Watch (Overprivileged), ArtCraft Contributors and the CraftSuite tracker. If your report touches those parts, say so in the issue and name the source.
