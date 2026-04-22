# Curated Endpoint Catalog

All endpoints called via the awal CLI:

```bash
npx awal@latest x402 pay <url> -X POST -d '<json>' --json
```

Prices are per single call. Behavior here was **verified by direct call** in 2026-04 — both via awal (for stableenrich endpoints) and via `x402-fetch` with a raw key (to distinguish endpoint bugs from awal bugs). Warnings reflect actual server responses, not theoretical concerns.

## Table of contents

1. [Exa (PRIMARY for sourcing and content extraction)](#exa)
2. [Apollo (universe scan + company enrich)](#apollo)
3. [Hunter (email verification — pair with guess patterns)](#hunter)
4. [Minerva (deep enrichment when you have an email/phone)](#minerva)
5. [Firecrawl (scrape JS-heavy candidate pages)](#firecrawl)
6. [Serper (recency / news signal)](#serper)
7. [Reddit (community signal, sparingly)](#reddit)
8. [StableSocial (audience-building roles only)](#stablesocial)
9. [Verified working but awal can't pay yet (x402 v1 — Tomba, Apollo via Orthogonal)](#x402-v1-blocked-on-awal)
10. [Server-side broken — do not call](#server-broken)
11. [Bazaar fallback](#bazaar-fallback)

---

## Exa

Base URL: `https://stableenrich.dev`

**Exa is the workhorse of this skill.** Verified working via awal: returns real names + LinkedIn URLs in ~800ms for $0.01.

### `POST /api/exa/search` — $0.01

Semantic web search with a category filter.

| Field | Type | Notes |
|-------|------|-------|
| `query` | string | Natural language. Include role + skill/industry + location/company. |
| `numResults` | number | Default 10, up to 100. Use 25 for first pass. |
| `category` | string | One of: `"company"`, `"research paper"`, `"news"`, `"pdf"`, `"github"`, `"tweet"`, `"personal site"`, `"linkedin profile"`, `"financial report"`. |

Patterns that work well:

```json
{"query": "senior account executive developer tools San Francisco", "numResults": 25, "category": "linkedin profile"}
{"query": "staff backend engineer distributed systems fintech", "numResults": 25, "category": "linkedin profile"}
{"query": "senior account executive CodeRabbit", "numResults": 10, "category": "linkedin profile"}
{"query": "open source contributors Rust async runtime", "numResults": 25, "category": "github"}
{"query": "ML engineer transformers inference optimization blog", "numResults": 15, "category": "personal site"}
```

Response shape: `{"results": [{"id": <url>, "title": "<full name + headline>", "url": <url>, "publishedDate": "...", "author": "<full name>", "image": "..."}]}`. Title typically formatted as `"Full Name - Title @ Company"`.

### `POST /api/exa/contents` — $0.002

Extract clean content from URLs you already have. **Dirt cheap — use it freely.**

```json
{"urls": ["https://www.linkedin.com/in/edanharel", "https://github.com/alice/repo"]}
```

Use it to:
- Pull a candidate's LinkedIn headline / current company after Exa search returns the URL
- Read the top of a GitHub README for the "what they build" snippet
- Extract a blog post abstract for tech ranking signals

### `POST /api/exa/find-similar` — $0.01

Given one strong-fit candidate URL, find similar profiles. Useful when the recruiter says "more like this person".

```json
{"url": "https://www.linkedin.com/in/alice-engineer", "numResults": 15}
```

### `POST /api/exa/answer` — $0.01

AI-generated answer over web search. Useful for one-shot priors — but always verify hits with Exa search before trusting.

---

## Apollo

Base URL: `https://stableenrich.dev`

**Apollo via stableenrich works for search but NOT for enrichment.** Use search endpoints to scan for who has these titles where; do not rely on `people-enrich` (it's server-broken — see [server-side broken](#server-broken)).

### `POST /api/apollo/people-search` — $0.02

Universe scan: find people by title + location + employer. Returns obfuscated names (`first_name` + `last_name_obfuscated` like `"O'***n"`) plus `id`, `title`, `organization.name`, `has_email`, `has_direct_phone`.

| Field | Type | Notes |
|-------|------|-------|
| `person_titles` | string[] | **Required for title filtering.** Array of exact-ish titles, e.g. `["Account Executive", "Senior Account Executive"]`. |
| `person_locations` | string[] | City names or regions. |
| `organization_ids` | string[] | Apollo org IDs from `org-search`. |
| `per_page` | number | Up to 100. Use 25 for first pass. |

**Critical pitfalls verified in testing:**

- ❌ Using `q_keywords` for titles returns **0 hits**. Use `person_titles` array.
- ❌ Combining `person_titles` + `q_keywords` returns **0 hits**. Use one or the other.
- ❌ `pagination.total_entries` is always `0` even when results are present (the stableenrich wrapper drops the count). Trust `people.length`.

Working call:

```json
{"person_titles": ["Account Executive", "Enterprise Account Executive"], "person_locations": ["San Francisco"], "per_page": 25}
```

How to use the obfuscated results: take `first_name + organization.name` and run an Exa LinkedIn search to recover the full name and LinkedIn URL.

### `POST /api/apollo/org-search` — $0.02

Find target companies by location and (weakly) keyword.

```json
{"q_keywords": "fintech series b", "organization_locations": ["New York"], "per_page": 25}
```

Returns rich firmographics (`name`, `website_url`, `linkedin_url`, `founded_year`, `organization_revenue`, headcount growth).

**Verified gotcha**: `q_keywords` filtering is weak — orgs come back filtered mostly by location. Don't trust it for tight industry targeting; use Exa LinkedIn search with the company type baked into the query instead.

### `POST /api/apollo/org-enrich` — $0.0495

Pull firmographics for one company by domain. Use sparingly — once per unique candidate-company in the shortlist, mostly to grab the email domain for guess-and-verify.

```json
{"domain": "stripe.com"}
```

---

## Hunter

Base URL: `https://stableenrich.dev`

**Hunter is the email validation backbone.** Pair it with email pattern guessing.

### `POST /api/hunter/email-verifier` — $0.03

Check if a discovered or guessed email is deliverable.

```json
{"email": "edan.harel@coderabbit.ai"}
```

Response includes `result` (`deliverable` / `risky` / `undeliverable`) and `score`. **Only report emails to the recruiter when `result === "deliverable"`.**

### Recommended email-discovery pattern (no LinkedIn → email endpoint works via awal yet)

For each top-N candidate:

1. Get company domain from Apollo `org-enrich` once per unique company.
2. Generate 2–3 guess patterns:
   - `firstname.lastname@domain`
   - `firstname@domain`
   - `flastname@domain` (first initial + last name)
3. Verify each guess in parallel via `hunter/email-verifier`.
4. Keep the first `deliverable`. If none verify, mark "LinkedIn only".

Cost: ~$0.09 per candidate (3 verifies). Pattern accuracy is high on US tech companies; lower on EMEA/legacy enterprises.

---

## Minerva

Base URL: `https://stableenrich.dev`

Minerva is a consumer identity graph. Use only when you **already have an email or phone** for the candidate — name + LinkedIn alone is not sufficient input (verified: returns `validation_errors: "No valid emails or phones were provided"`).

### `POST /api/minerva/resolve` — $0.02

Resolve a person to a Minerva PID + LinkedIn URL.

```json
{"records": [{"record_id": "1", "first_name": "Jane", "last_name": "Doe", "emails": ["jane@stripe.com"]}]}
```

### `POST /api/minerva/enrich` — $0.05

Full enrichment given a `minerva_pid`, `linkedin_url` (with email also present), or name+email/phone.

```json
{"records": [{"record_id": "1", "minerva_pid": "p-..."}], "return_fields": ["work_history", "education", "personal_emails", "phones"]}
```

Use only on the top 5 ranked candidates after a verified email is in hand. Otherwise skip.

### `POST /api/minerva/validate-emails` — $0.01

Bulk check emails against Minerva DB. Cheaper than Hunter for batch validation if checking >5 at once, but Hunter is more authoritative on deliverability — prefer Hunter for the final go/no-go.

---

## Firecrawl

Base URL: `https://stableenrich.dev`

### `POST /api/firecrawl/scrape` — $0.0126

Scrape a single URL to clean markdown. Use when a candidate's portfolio site is JS-heavy and Exa `contents` returns thin text.

```json
{"url": "https://alice.dev/about"}
```

### `POST /api/firecrawl/search` — $0.0252

Web search + scrape combined. Generally Exa `search` + `contents` is cheaper and better — only use Firecrawl `search` when Exa returns nothing relevant.

---

## Serper

Base URL: `https://stableenrich.dev`

### `POST /api/serper/news` — $0.04

Google News search. Useful when checking if a senior candidate has been quoted, founded a company, or made the news (recency signal for ranking).

```json
{"q": "Edan Harel CodeRabbit senior account executive", "num": 10, "gl": "us", "hl": "en"}
```

Use only on top 5 candidates — too expensive for full shortlist.

---

## Reddit

Base URL: `https://stableenrich.dev`

### `POST /api/reddit/search` — $0.02

Search Reddit for tech-community signals (r/ExperiencedDevs, r/MachineLearning, r/sales). Most hits are noise; use sparingly to spot under-the-radar contributors or domain influencers.

```json
{"query": "best engineers I've worked with at fintech", "sort": "relevance", "timeframe": "year", "maxResults": 20}
```

---

## StableSocial

Base URL: `https://stablesocial.dev`

For most recruiting, StableSocial is **optional** — only pull social data when the role explicitly involves audience-building (DevRel, content, community, founding marketing). All endpoints are $0.06.

| Endpoint | Use case |
|----------|----------|
| `POST /api/instagram/profile` | Verify a candidate's IG follower count for content/community roles |
| `POST /api/tiktok/profile` | Same, for TikTok-native creators |
| `POST /api/reddit/search-profiles` | Find Reddit handles for community-manager candidates |

Skip StableSocial entirely for IC engineering / standard GTM roles.

---

## x402 v1 — blocked on awal

These endpoints are **verified working** via direct testing (`x402-fetch` SDK with a raw private key in 2026-04), but the awal CLI cannot pay them today. Awal signs and submits the payment — the on-chain USDC transfer **does happen** (visible on basescan), but the orth.sh facilitator returns a response format awal cannot parse, so awal reports `"Payment was authorized but rejected by server"` and never surfaces the data. The fetched response sits orphaned.

**Why include them in this catalog?** When awal adds x402 v1 support (or the orth.sh response format issue is resolved), these endpoints become the canonical recruiting flow — one $0.01 Apollo call replaces the entire `org-enrich + Hunter guess+verify` chain. The skill is structured so swapping them in is one playbook edit. Do not call them today via awal.

If the recruiter insists on using these endpoints today, see `references/advanced-orthogonal.md` for the `x402-fetch` + raw `PRIVATE_KEY` workaround.

### `GET https://x402.orth.sh/tomba/v1/linkedin?url=<linkedin-url>` — $0.01

Returns verified email, full name, current company + domain, position, department, email pattern, and confidence score for a LinkedIn profile URL.

Verified response shape on Satya Nadella:
```json
{"data": {"email": "satyan@microsoft.com", "first_name": "Satya", "last_name": "Nadella",
  "company": "Microsoft", "website_url": "microsoft.com", "pattern": "{f}{last}",
  "department": "executive", "position": "ceo", "score": 100,
  "verification": {"status": "valid", "date": "2026-02-14"}}}
```

This is the cleanest LinkedIn → email path in the entire catalog. Coverage varies by candidate: famous executives are well-covered; mid-level ICs at small startups may not be in Tomba's database (returns nulls + score 0).

### `POST https://x402.orth.sh/apollo/api/v1/people/match` — $0.01

Apollo's real people-match endpoint with **working** `reveal_personal_emails: true`. Returns verified business email + full profile in one call.

Body (LinkedIn URL or `email` or `first_name`+`last_name`+`organization_name`):
```json
{"linkedin_url": "https://www.linkedin.com/in/satyanadella", "reveal_personal_emails": true}
```

**Do not** also pass `reveal_phone_number: true` without also passing a `webhook_url` — Apollo returns a 400 because phone reveal is async-only.

Verified response shape:
```json
{"person": {"email": "satyan@microsoft.com", "email_status": "verified",
  "first_name": "Satya", "last_name": "Nadella", "title": "Chairman and CEO",
  "headline": "Chairman and CEO at Microsoft", "linkedin_url": "...",
  "organization": {"name": "Microsoft", "primary_domain": "microsoft.com", ...},
  "seniority": "c_suite", "departments": ["c_suite"], "personal_emails": [],
  "employment_history": [...]}}
```

When awal v1 lands, this single call replaces the entire enrichment loop in `playbooks.md` for the top 20 — cuts per-candidate cost from ~$0.10 to $0.01 and skips email guessing entirely.

### `POST https://x402.orth.sh/apollo/api/v1/mixed_people/api_search` — FREE

Apollo's people search through the Orthogonal proxy. **No payment required.** Returns the same `total_entries` count and obfuscated-name `people` array as stableenrich's people-search, **but `total_entries` actually populates** (verified: 7271 vs stableenrich's broken 0).

Same body shape as stableenrich `people-search`:
```json
{"person_titles": ["Account Executive"], "person_locations": ["San Francisco"], "per_page": 25}
```

Useful as a free universe scan when awal v1 lands. Today, blocked by the same v1 issue.

### Other orth.sh endpoints

The Orthogonal proxy fronts a much larger catalog (Tomba's 20+ endpoints, Fiber, Nyne, Sixtyfour, more Apollo). Spec lives in the user's local docs at `/Users/ashnouruzi/bundle-maker/enrich/`. Not all are tested:

- **Fiber `v1/natural-language-search/profiles`**: ❌ verified broken — returns 500 `"Failed to process payment middleware"`. Skip Fiber until fixed.
- Other orth.sh endpoints (Nyne, Sixtyfour, other Apollo orth, other Tomba): not yet tested. Apply the same "blocked on awal v1" reasoning.

---

## Server-side broken

Verified empty-body responses via **both** awal (when payment settles) and x402-fetch with a raw private key. **Do not call.** USDC is charged, no usable data is returned.

### ❌ `POST https://stableenrich.dev/api/apollo/people-enrich` — $0.0495

Returns empty body via x402-fetch (verified 2026-04 with `linkedin_url` and `email` inputs, with and without `reveal_personal_emails: true`). Earlier awal calls returned partial JSON with name + employment_history but null contact data — so the wrapper is intermittent at best, broken at worst. Not usable for contact resolution either way.

Alternatives: use `apollo/api/v1/people/match` via Orthogonal (when awal v1 lands), or stay with the Hunter guess+verify pattern documented above.

### ❌ `POST https://stableenrich.dev/api/clado/contacts-enrich` — $0.20

Returns empty body via both awal and x402-fetch. Verified across multiple LinkedIn URL formats (with/without `www`, trailing slash, public profile URLs). Endpoint exists in `awal x402 details` but every actual call settles payment and returns nothing. Do not call.

Alternatives: Tomba `/v1/linkedin` ($0.01, 95% cheaper, when awal v1 lands), or Hunter guess+verify pattern today.

---

## Bazaar fallback

This skill ships with a curated list by design — predictable cost, fast, vetted. Do **not** call the Bazaar discovery endpoints during normal sourcing.

The one exception: if the recruiter's brief is genuinely outside this catalog (e.g. "find me drone-pilot candidates with FAA Part 107 certs"), then run `npx awal@latest x402 bazaar search "<query>"` to see if a specialized endpoint exists. Surface the new endpoint to the user before paying.
