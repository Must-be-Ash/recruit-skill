# Advanced: Orthogonal x402 v1 endpoints (Tomba, Fiber, Nyne, Sixtyfour, Apollo)

**Read only when the recruiter explicitly asks for endpoints not available via awal — e.g. "find me an email from this LinkedIn URL", "do natural-language profile search", or "use Fiber/Tomba/Nyne".**

The default skill workflow uses `https://stableenrich.dev/...` paid via the awal CLI. Awal cannot pay the richer recruitment stack hosted at `https://x402.orth.sh/...` (verified: x402 v1 payment requirements, awal signs but the orth.sh facilitator rejects). To use the orth.sh stack, the agent must run a small Node script with `x402-fetch` and a raw `PRIVATE_KEY` env var.

## When this stack is worth the setup

| Use case | Endpoint | Why this beats the awal-stableenrich path |
|----------|----------|------------------------------------------|
| LinkedIn URL → email in one call | `tomba/v1/linkedin?url=<linkedin>` ($0.01) | The default flow guesses email patterns and verifies — Tomba just returns it directly. |
| Find email + phone for a person | `tomba/v1/email-finder` + `tomba/v1/phone-finder` ($0.01 each) | Same: avoids guess+verify entirely. |
| Natural-language profile search | `fiber/v1/natural-language-search/profiles` (Dynamic) | Pass a recruiter brief verbatim, get matching profiles. Avoids hand-crafting Exa queries. |
| Apollo with working contact reveal | `apollo/api/v1/people/match` ($0.01, with `reveal_personal_emails: true`) | Returns actual email/phone — stableenrich's wrapper does not. |
| Free Apollo people search | `apollo/api/v1/mixed_people/api_search` (Free, but returns no payment-gated data) | Same data shape as paid search but free. |
| Deep enrichment with social profiles | `nyne/person/enrichment` (Dynamic) | Returns AI-enhanced search across LinkedIn/Twitter/Instagram/GitHub. |
| Find verified contact for a lead | `sixtyfour/find-email`, `sixtyfour/find-phone` ($0.30 phone, dynamic email) | Sixtyfour-style "find" semantics, designed for recruiting/sales. |

## Setup the agent must do once

The recruiter's awal wallet has no exportable private key, so the agent **must** ask the recruiter to provide a raw EVM private key for a separate funded wallet. Make the ask explicit:

> "To use the Orthogonal endpoints (Tomba, Fiber, Nyne, etc.), I need a raw EVM private key (0x...) from a wallet funded with USDC on Base. This is separate from your awal wallet because awal can't pay the v1 facilitator. If you'd rather skip those endpoints, I can stay with the working awal stack (Exa LinkedIn + Hunter guess+verify) and you'll lose direct LinkedIn→email lookups."

If the recruiter approves, capture the key into a session env var (never log it):

```bash
export ORTH_PRIVATE_KEY=0x...
```

## Calling pattern

Write a one-shot Node script per call (or batch). The agent should write the script to a temp file, run it, parse the output, then delete the script.

```bash
# Install once per session
npm install --silent --no-save x402-fetch viem 2>&1 | tail -3
```

Template script (`/tmp/orth-call.mjs`):

```javascript
import { wrapFetchWithPayment } from "x402-fetch";
import { privateKeyToAccount } from "viem/accounts";

const account = privateKeyToAccount(process.env.ORTH_PRIVATE_KEY);
const fetchWithPayment = wrapFetchWithPayment(fetch, account);

const url = process.argv[2];
const method = process.argv[3] || "GET";
const body = process.argv[4] || null;

const opts = { method, headers: {} };
if (body) {
  opts.headers["Content-Type"] = "application/json";
  opts.body = body;
}

const res = await fetchWithPayment(url, opts);
const data = await res.json();
console.log(JSON.stringify(data));
```

Invoke:

```bash
# GET (Tomba LinkedIn → email)
node /tmp/orth-call.mjs 'https://x402.orth.sh/tomba/v1/linkedin?url=https://www.linkedin.com/in/edanharel'

# POST (Apollo people match with reveal)
node /tmp/orth-call.mjs 'https://x402.orth.sh/apollo/api/v1/people/match' POST '{"linkedin_url":"https://www.linkedin.com/in/edanharel","reveal_personal_emails":true}'

# POST (Fiber NL profile search)
node /tmp/orth-call.mjs 'https://x402.orth.sh/fiber/v1/natural-language-search/profiles' POST '{"query":"senior account executives at developer tools companies in San Francisco","pageSize":10}'
```

## Endpoint catalog (orth.sh stack)

Full schemas live in the per-provider docs the user shipped at `/Users/ashnouruzi/bundle-maker/enrich/`:

- **Tomba** (`tomba-x402-SKILL.md`) — 20+ endpoints, all $0.01. Email discovery from any signal (domain, name, LinkedIn URL, blog post URL). Email format detection, employee count by domain, phone finder.
- **Fiber AI** (`fiber-x402-SKILL.md`) — natural-language and structured profile/company/job/investor search; LinkedIn live-fetch for profile + company; kitchen-sink endpoints that accept any combination of identifiers.
- **Nyne.ai** (`nyne-x402-SKILL.md`) — async polling pattern (POST starts a job, GET polls). AI-scored search with `show_emails: true`, `profile_scoring: true`. Person enrichment + interactions + interests + newsfeed.
- **Sixtyfour** (`sixtyfour-x402-SKILL.md`) — find-email (PROFESSIONAL/PERSONAL modes), find-phone, enrich-lead, enrich-company with `find_people: true` and `people_focus_prompt` (powerful for "find devs at this company who do X").
- **Apollo via Orthogonal** (`apollo-x402-SKILL.md`) — `/api/v1/people/match` with working `reveal_personal_emails: true`, plus `/mixed_people/api_search` (FREE), `/organizations/{id}/job_postings`, `/news_articles/search`.
- **Exa via Orthogonal** (`exa-x402-SKILL.md`) — same Exa search/findSimilar/contents/answer; also exposes `/research/v1` async deep research with `outputSchema` for structured candidate output.

## Cost guardrails

The orth.sh stack contains "Dynamic" priced endpoints (Fiber, Nyne `person/search`). **Always start with the smallest possible page size** (`pageSize: 5` or `limit: 5`) to discover the actual price before scaling up. The agent must surface the per-call cost from the response before running larger searches.

## When NOT to use this stack

- The recruiter has no separate funded wallet to share a private key from.
- The brief is small enough that the awal stack handles it adequately (most 20-candidate searches are fine without orth.sh).
- The recruiter is uncomfortable with private-key handling — this path requires shell env-var management. Default to the awal stack if there's any hesitation.
