# Testing log

Verification notes for the `recruit` skill. **Not loaded into the agent's context** — this is for skill maintainers to track what works, what doesn't, and where awal compatibility differs from endpoint reality.

Last verified: 2026-04-22.

## How to test an endpoint

The skill assumes awal pays any working x402 endpoint. To distinguish "endpoint broken" from "awal client bug", we test directly with `x402-fetch` + a raw private key. If a call works via x402-fetch but awal reports failure, the endpoint stays in the skill (we'll fix awal); if it fails both ways, the endpoint is genuinely broken.

```bash
# One-time setup in /tmp/x402-test
mkdir -p /tmp/x402-test && cd /tmp/x402-test
npm init -y && npm install --silent x402-fetch viem
```

Save this as `/tmp/x402-test/call.mjs`:

```javascript
import { wrapFetchWithPayment } from "x402-fetch";
import { privateKeyToAccount } from "viem/accounts";

const pk = process.env.WALLET_PRIVATE_KEY;
const account = privateKeyToAccount(pk.startsWith("0x") ? pk : `0x${pk}`);
const fetchPay = wrapFetchWithPayment(fetch, account);

const url = process.argv[2];
const method = process.argv[3] || "GET";
const body = process.argv[4] || null;

const opts = { method, headers: {} };
if (body) {
  opts.headers["Content-Type"] = "application/json";
  opts.body = body;
}

const res = await fetchPay(url, opts);
const text = await res.text();
let parsed;
try { parsed = JSON.parse(text); } catch { parsed = text; }
console.log(JSON.stringify({ status: res.status, ok: res.ok, data: parsed }, null, 2));
```

Run via:

```bash
set -a; source <path-to-env>/.env.local; set +a
cd /tmp/x402-test && node call.mjs '<url>' POST '<json-body>'
```

## Endpoint status

### ✅ Working — included in skill

| Endpoint | Surface | Price | Notes |
|----------|---------|-------|-------|
| `apollo/api/v1/people/match` (with `reveal_personal_emails:true`) | `x402.orth.sh` | $0.01 | Verified: Satya Nadella → `email: satyan@microsoft.com`, `email_status: "verified"`, full profile + departments + seniority. **Best one-call enrichment in catalog.** |
| `apollo/api/v1/mixed_people/api_search` | `x402.orth.sh` | FREE | Verified: returned 7271 total entries for SF Account Executives. Stableenrich's wrapper reports `total_entries: 0` even when results are present — Orthogonal version reports the real count. |
| `apollo/api/v1/organizations/enrich` | `x402.orth.sh` | $0.01 | Cheaper than stableenrich's `/api/apollo/org-enrich` ($0.0495). |
| `tomba/v1/linkedin?url=<linkedin>` | `x402.orth.sh` | $0.01 | Verified: Satya Nadella → email + pattern `{f}{last}` + score 100 + verification valid. Coverage gap: unknown candidates return all nulls. |
| `tomba/v1/email-finder` | `x402.orth.sh` | $0.01 | Untested directly but same proxy infra as `v1/linkedin`. |
| `tomba/v1/email-verifier` | `x402.orth.sh` | $0.01 | Untested. Cheaper than Hunter ($0.03). |
| `exa/search` | `stableenrich.dev` | $0.01 | Verified: returned 5 real CodeRabbit AEs with full names + LinkedIn URLs in 800ms. **The best LinkedIn discovery endpoint in the catalog.** |
| `exa/contents` | `stableenrich.dev` | $0.002 | Untested directly but standard Exa surface. |
| `apollo/people-search` | `stableenrich.dev` | $0.02 | Verified working with `person_titles` array. Three pitfalls: (1) `q_keywords` for titles returns 0 hits, (2) combining `person_titles` + `q_keywords` returns 0, (3) `pagination.total_entries` is always 0. **Prefer Orthogonal's free `mixed_people/api_search` when you don't need stableenrich-specific quirks.** |
| `apollo/org-search` | `stableenrich.dev` | $0.02 | Works. `q_keywords` filtering is weak — use Exa for tighter industry targeting. |
| `apollo/org-enrich` | `stableenrich.dev` | $0.0495 | Works. Apollo via Orthogonal's `organizations/enrich` does the same thing for $0.01 — prefer that. |
| `hunter/email-verifier` | `stableenrich.dev` | $0.03 | Works. Authoritative deliverability check. |
| `minerva/resolve`, `minerva/enrich`, `minerva/validate-emails` | `stableenrich.dev` | $0.02 / $0.05 / $0.01 | Verified `resolve` works but requires `emails` or `phones` in input — name + linkedin alone is insufficient (returns `validation_errors: "No valid emails or phones were provided"`). |
| `firecrawl/scrape`, `firecrawl/search` | `stableenrich.dev` | $0.0126 / $0.0252 | Untested directly. |
| `serper/news` | `stableenrich.dev` | $0.04 | Untested directly. |
| `reddit/search`, `reddit/post-comments` | `stableenrich.dev` | $0.02 each | Untested directly. |

### ❌ Server-side broken — excluded from skill

| Endpoint | Surface | Verified failure |
|----------|---------|------------------|
| `apollo/people-enrich` | `stableenrich.dev` | Empty body via x402-fetch with `linkedin_url`, `email`, and `person_id` inputs. Earlier awal calls returned partial JSON (name + employment_history) but null contact data — wrapper is intermittent at best. **Replaced by Apollo via Orthogonal's `/people/match`.** |
| `clado/contacts-enrich` | `stableenrich.dev` | Empty body via x402-fetch on multiple LinkedIn URL formats. Endpoint exists in `awal x402 details` but every actual call charges $0.20 and returns nothing. |
| `fiber/v1/natural-language-search/profiles` | `x402.orth.sh` | Returns 500 `"Failed to process payment middleware"`. Other Fiber endpoints presumed broken on the same proxy infra (untested). |

### ⚠️ awal-specific bug (endpoint works, awal client cannot render)

Verified 2026-04 with awal versions 2.0.3, 2.8.0, and 2.8.1.

awal reports `"Payment was authorized but rejected by server"` for any endpoint hosted at `https://x402.orth.sh/...`. Basescan confirms USDC **does transfer on-chain** for these "Failed" calls — payment settles cleanly. The orth.sh facilitator is x402 v1; awal's response handling appears not to recognize the v1 response shape, so the data is fetched but never surfaced to the caller.

When awal adds v1 response handling, all the orth.sh endpoints in the "Working" table above become awal-payable with no skill changes required.

Until then: the skill catalog already includes them (the assumption being awal will catch up). If you need to use them today, run the Node `x402-fetch` script above.

## Test wallet info

- Test wallet address: `0x16CA9e69E97EF3E740f573E79b913183BF500C18` (loaded from `WALLET_PRIVATE_KEY` in `bundle-maker/.env.local`)
- awal wallet address: `0xC160EaEFEb8CabB7D276EB92582755808e286581`
- Total spent across all verification: ~$0.75 USDC

## Things still untested (worth covering before next release)

- Other Tomba endpoints (`email-finder`, `email-format`, `domain-search`, `phone-finder`, `combined/find`)
- Nyne, Sixtyfour endpoints (any of them — same proxy infra as Tomba so likely work)
- `firecrawl/scrape`, `firecrawl/search`, `serper/news`, `serper/shopping`, `reddit/search`
- `exa/find-similar`, `exa/answer`
- `apollo/news_articles/search`, `apollo/organizations/{id}/job_postings` (orth.sh)
- Influencer endpoints (`influencer/enrich-by-email`, `influencer/enrich-by-social`) — for content-creator recruiting
- Whitepages endpoints (likely irrelevant to recruiting)
- Cloudflare browser rendering (likely irrelevant to recruiting)

## How to run a fresh end-to-end smoke test

A 5-candidate GTM brief exercising the full path:

```bash
# 1. Free universe scan
node call.mjs 'https://x402.orth.sh/apollo/api/v1/mixed_people/api_search' POST \
  '{"person_titles":["Account Executive"],"person_locations":["San Francisco"],"per_page":5}'

# 2. Enrich one of the returned ids (or use a known LinkedIn URL)
node call.mjs 'https://x402.orth.sh/apollo/api/v1/people/match' POST \
  '{"linkedin_url":"https://www.linkedin.com/in/<known-person>","reveal_personal_emails":true}'

# 3. Backup email lookup if Apollo missed
node call.mjs 'https://x402.orth.sh/tomba/v1/linkedin?url=https://www.linkedin.com/in/<known-person>' GET

# 4. Final verification
npx awal@latest x402 pay https://stableenrich.dev/api/hunter/email-verifier \
  -X POST -d '{"email":"<email-from-step-2>"}' --json
```

Total cost for the smoke test: ~$0.05. Run before any release.
