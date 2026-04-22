# Curated Endpoint Catalog

All endpoints below are x402 **v2** pay-per-call, settle in USDC on Base, return JSON, and are paid via the awal CLI:

```bash
npx awal@latest x402 pay <url> -X POST -d '<json>' --json
```

Prices are per single call. They may drift — treat as estimates. Every claim about behavior here was verified by direct calls in 2026-04 — the warnings about broken endpoints reflect actual server responses, not theoretical concerns.

## Table of contents

1. [Exa (PRIMARY for both sourcing and content extraction)](#exa)
2. [Apollo (universe scan only — see warnings)](#apollo)
3. [Hunter (email verification — pair with guess patterns)](#hunter)
4. [Minerva (deep enrichment when you have an email/phone)](#minerva)
5. [Firecrawl (scrape JS-heavy candidate pages)](#firecrawl)
6. [Serper (recency / news signal)](#serper)
7. [Reddit (community signal, sparingly)](#reddit)
8. [StableSocial (audience-building roles only)](#stablesocial)
9. [Broken endpoints — do not call](#broken-endpoints)
10. [Bazaar fallback](#bazaar-fallback)

---

## Exa

Base URL: `https://stableenrich.dev`

**Exa is the workhorse of this skill.** Verified working: returns real names + LinkedIn URLs in ~800ms for $0.01. Use it for both tech and GTM sourcing.

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

Response shape: `{"results": [{"id": <linkedin_url>, "title": "<full name + headline>", "url": <linkedin_url>, "publishedDate": "...", "author": "<full name>", "image": "..."}]}`. Title typically formatted as `"Full Name - Title @ Company"`.

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

AI-generated answer over web search. Useful for one-shot priors like "who are the most-cited authors on retrieval-augmented generation 2024-2025" — but always verify hits with Exa search before trusting.

---

## Apollo

Base URL: `https://stableenrich.dev`

**Apollo via stableenrich is partially working**: search endpoints work for universe scanning, but the enrichment endpoint does NOT currently return contact data (verified 2026-04). Use Apollo for company discovery and "which orgs have these titles" — do not rely on it for emails.

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
- ❌ `pagination.total_entries` is always `0` even when results are present. Trust `people.length`.

Working call:

```json
{"person_titles": ["Account Executive", "Enterprise Account Executive"], "person_locations": ["San Francisco"], "per_page": 25}
```

How to actually use the obfuscated results: take `first_name + organization.name` and run an Exa LinkedIn search to recover the full name and LinkedIn URL.

### `POST /api/apollo/org-search` — $0.02

Find target companies by location and (weakly) keyword.

```json
{"q_keywords": "fintech series b", "organization_locations": ["New York"], "per_page": 25}
```

Returns rich firmographics (`name`, `website_url`, `linkedin_url`, `founded_year`, `organization_revenue`, headcount growth).

**Verified gotcha**: `q_keywords` filtering is weak — orgs come back filtered mostly by location, with industry keywords only marginally affecting ranking. Don't trust it for tight industry targeting; instead use Exa LinkedIn search with the company type baked into the query.

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

Cost: ~$0.09 per candidate (3 verifies). Pattern-detection accuracy is high on US tech companies; lower on EMEA/legacy enterprises.

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

## Broken endpoints

These are documented because earlier versions of this skill recommended them. **Do not call** unless and until verified working again:

### ❌ `POST /api/apollo/people-enrich` — $0.0495 (broken via stableenrich for contact data)

Returns `status: "matched"` with `name` and `employment_history[].organization_name`, but **`email`, `linkedin_url`, `title`, `seniority`, `personal_emails` all come back null** — even when `reveal_personal_emails: true` is passed in the body. The `id` returned is also different from the input `person_id`, suggesting the wrapper isn't preserving the Apollo identity. Use Hunter guess+verify instead.

### ❌ `POST /api/clado/contacts-enrich` — $0.20 (server-side rejection)

Returns `"Payment was authorized but rejected by server"` on every call attempt, with multiple LinkedIn URL formats. Endpoint exists in `awal x402 details` but execution is broken. Do not call.

### ⚠️ x402.orth.sh endpoints (Tomba, Fiber, Nyne, Sixtyfour, Apollo via Orthogonal) — incompatible with awal

There is a much richer recruitment stack at `https://x402.orth.sh/...` — Tomba `/v1/linkedin` ($0.01) returns email from a LinkedIn URL, Fiber `/v1/natural-language-search/profiles` accepts a recruiter brief in plain English, Apollo via Orthogonal exposes a working `reveal_personal_emails` flow, Nyne offers AI-scored search, Sixtyfour does deep find-email/find-phone.

**These all return x402 v1 payment requirements, and the awal CLI (any version) cannot pay them — the wallet signs and submits a payment which the orth.sh facilitator rejects.** Verified with awal 2.0.3 and awal 2.8.0.

If the recruiter explicitly asks to use this stack, see `references/advanced-orthogonal.md` — the orth.sh path requires a small Node script using `x402-fetch` with a raw `PRIVATE_KEY` env var, not awal.

---

## Bazaar fallback

This skill ships with a curated list by design — predictable cost, fast, vetted. Do **not** call the Bazaar discovery endpoints during normal sourcing.

The one exception: if the recruiter's brief is genuinely outside this catalog (e.g. "find me drone-pilot candidates with FAA Part 107 certs"), then run `npx awal@latest x402 bazaar search "<query>"` to see if a specialized endpoint exists. Surface the new endpoint to the user before paying.
