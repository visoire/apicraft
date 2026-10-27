# Security policy

## Report a vulnerability

Please **do not** open a public issue for a security problem.

Report it privately with GitHub's **"Report a vulnerability"** button on the [Security tab](../../security/advisories/new) of this repository. If that is not available, open an issue titled "Security contact request" with **no details** and ask for a private channel; you will be answered there.

Please include what you found, the steps to reproduce it, the address it affects (`opencraft.cc` or `api.opencraft.cc`) and what you think the impact is. Use test accounts only.

## What to expect

- An acknowledgement within a few days. This is a one-person project, so there is no guaranteed time frame.
- A fix or a clear answer, and credit in the release notes if you want it.
- Please give me reasonable time to fix the problem before you publish anything.

## Rules for testing

- Only test your own account and your own data.
- No denial-of-service, no load testing, no automated scanning that floods the service, no social engineering, no physical attacks.
- Do not access, change or delete other people's data. If you reach it by accident, stop and report it.

## Out of scope

Missing best-practice headers without a demonstrated impact, rate limits being "too low" or "too high", reports about third-party services (GitHub, your mail provider), and issues that need a compromised device of the victim.

## Keys and tokens

If you ever leak an API key (`ac_...`) or a GitHub token, revoke it right away: regenerate the key on the dashboard and delete the token on GitHub. Treat anything you pasted in public as compromised.
