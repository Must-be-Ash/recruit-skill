# Endpoint Catalog

All endpoints below are x402 pay-per-call, settle in USDC on Base, return JSON, and are paid via awal:

```bash
npx awal@latest x402 pay <url> -X POST -d '<json>' --json
# GET endpoints:
npx awal@latest x402 pay '<url-with-query-string>' -X GET --json
```

## Table of contents

1. [Apollo via Orthogonal — primary enrichment](#apollo-via-orthogonal)
2. [Tomba — LinkedIn → email backup](#tomba)
3. [Exa — sourcing and content extraction](#exa)
4. [Apollo via stableenrich — company enrich](#apollo-via-stableenrich)
5. [Hunter — email verification](#hunter)
6. [Minerva — deep enrichment when you have an email](#minerva)
7. [Firecrawl — scrape JS-heavy candidate pages](#firecrawl)
8. [Serper — recency / news signal](#serper)
9. [Reddit — community signal](#reddit)
10. [StableSocial — audience-building roles only](#stablesocial)
11. [Bazaar fallback](#bazaar-fallback)

---

## Apollo via Orthogonal

Base URL: `https://x402.orth.sh/apollo`

**The primary enrichment surface in this skill.** One call returns verified email + full profile.

### `POST /api/v1/mixed_people/api_search` — FREE

Universe scan. Returns up to 100 people per page matching structured filters. Names come back partly obfuscated (`first_name` + `last_name_obfuscated`) plus `id`, `title`, `organization.name`, `has_email`. Pair with `/people/match` to get full names + verified emails.

| Field | Type | Notes |
|-------|------|-------|
| `person_titles` | string[] | Required for title filtering. Array of exact-ish titles. |
| `person_seniorities` | string[] | `owner`, `founder`, `c_suite`, `partner`, `vp`, `head`, `director`, `manager`, `senior`, `entry`. |
| `person_locations` | string[] | City names or regions. |
| `organization_locations` | string[] | Filter by company HQ location. |
| `organization_num_employees_ranges` | string[] | E.g. `["1,10"]`, `["50,100"]`, `["1000,5000"]`. |
| `q_keywords` | string | Free text. **Do not combine with `person_titles`** — returns 0 hits. |
| `per_page` | number | Up to 100. Use 25 for first pass. |
| `page` | number | Pagination. |

```json
{"person_titles": ["Account Executive", "Enterprise Account Executive"], "person_locations": ["San Francisco"], "per_page": 25}
```

### `POST /api/v1/people/match` — $0.01

The enrichment workhorse. Pass a LinkedIn URL (preferred), email, or `first_name`+`last_name`+`organization_name` and get back:

- `email` — verified business email
- `email_status` — `"verified"` when present
- `first_name`, `last_name`, `headline`, `title`
- `linkedin_url`, `photo_url`, `twitter_url`, `github_url`
- `organization` — full firmographics (`name`, `primary_domain`, `website_url`, headcount, revenue)
- `seniority`, `departments`, `subdepartments`, `functions`
- `employment_history` — full work history
- `personal_emails` — when available

Body:

```json
{"linkedin_url": "https://www.linkedin.com/in/edanharel", "reveal_personal_emails": true}
```

**Do not** also pass `reveal_phone_number: true` without `webhook_url` — Apollo returns 400 because phone reveal is async-only.

### `GET /api/v1/organizations/enrich?domain=<domain>` — $0.01

Cheaper than the stableenrich version. Returns full firmographics by domain.

### `POST /api/v1/organizations/{organization_id}/job_postings` — $0.01

Get current job postings for a company. Useful when scoping target companies — "who's hiring AEs right now".

### `POST /api/v1/news_articles/search` — $0.01

News articles related to specific Apollo organizations. Pass `organization_ids` (required) plus optional `q_keywords`.

---

## Tomba

Base URL: `https://x402.orth.sh/tomba`

LinkedIn → email backup when Apollo `/people/match` doesn't return one. All endpoints are $0.01.

### `GET /v1/linkedin?url=<linkedin-url>` — $0.01

Returns email + pattern + verification status from a LinkedIn profile URL.

```bash
npx awal@latest x402 pay 'https://x402.orth.sh/tomba/v1/linkedin?url=https://www.linkedin.com/in/satyanadella' -X GET --json
```

Response shape:
```json
{"data": {"email": "satyan@microsoft.com", "first_name": "Satya", "last_name": "Nadella",
  "company": "Microsoft", "website_url": "microsoft.com", "pattern": "{f}{last}",
  "department": "executive", "position": "ceo", "score": 100,
  "verification": {"status": "valid"}}}
```

Coverage: well-known professionals are covered with `score: 100`; unknown candidates return all nulls + `score: 0` — fall back to the next strategy.

### `GET /v1/email-finder?domain=<d>&company=<c>&first_name=<f>&last_name=<l>` — $0.01

Construct the most likely email when you have name + company. Useful when neither Apollo nor Tomba LinkedIn lookup returns an email.

### `GET /v1/email-format?domain=<domain>` — $0.01

Get the email pattern at a company (e.g. `{first}.{last}` or `{f}{last}`). Use to seed manual email construction in batch.

### `GET /v1/email-verifier?email=<email>` — $0.01

Cheaper Hunter alternative. Use this for routine verification; reserve Hunter for the final go/no-go on top candidates.

### `GET /v1/phone-finder?linkedin=<url>` — $0.01

Find phone numbers from a LinkedIn URL. Optional — most recruiters care about email first.

### `GET /v1/domain-search?domain=<d>&company=<c>&department=<dept>` — $0.01

Find ALL emails at a company in a department. Powerful when scoping "who at Stripe is in engineering".

---

## Exa

Base URL: `https://stableenrich.dev`

Sourcing workhorse for finding candidates with full names + URLs.

### `POST /api/exa/search` — $0.01

Semantic web search with category filter. Use 25 results for first pass.

| Field | Type | Notes |
|-------|------|-------|
| `query` | string | Natural language. Include role + skill/industry + location/company. |
| `numResults` | number | Default 10, up to 100. |
| `category` | string | `"linkedin profile"`, `"github"`, `"personal site"`, `"company"`, `"news"`, `"research paper"`, `"pdf"`, `"tweet"`, `"financial report"`. |

Patterns that work:

```json
{"query": "senior account executive developer tools San Francisco", "numResults": 25, "category": "linkedin profile"}
{"query": "open source contributors Rust async runtime", "numResults": 25, "category": "github"}
{"query": "ML engineer transformers inference optimization blog", "numResults": 15, "category": "personal site"}
```

Title format on LinkedIn results: `"Full Name - Title @ Company"`. Parse the title for current employer when present.

### `POST /api/exa/contents` — $0.002

Extract clean content from URLs. Dirt cheap — use freely.

```json
{"urls": ["https://www.linkedin.com/in/edanharel", "https://github.com/alice/repo"]}
```

### `POST /api/exa/find-similar` — $0.01

```json
{"url": "https://www.linkedin.com/in/alice-engineer", "numResults": 15}
```

Use when the recruiter says "more like this person".

### `POST /api/exa/answer` — $0.01

AI-generated answer over web search. Useful for one-shot priors — verify hits with Exa search before trusting.

---

## Apollo via stableenrich

Base URL: `https://stableenrich.dev`

Use for `org-search` and `org-enrich` when working through the stableenrich path. For people-search, prefer the Orthogonal `mixed_people/api_search` (free) instead.

### `POST /api/apollo/org-search` — $0.02

Find target companies by location and (weakly) keyword.

```json
{"q_keywords": "fintech series b", "organization_locations": ["New York"], "per_page": 25}
```

Returns full firmographics. Note: `q_keywords` filtering is weak; orgs come back filtered mostly by location. Use Exa LinkedIn search with the company type baked into the query for tighter industry targeting.

### `POST /api/apollo/org-enrich` — $0.0495

Pull firmographics for one company by domain.

```json
{"domain": "stripe.com"}
```

---

## Hunter

Base URL: `https://stableenrich.dev`

### `POST /api/hunter/email-verifier` — $0.03

Authoritative deliverability check. Use for the final go/no-go on top candidate emails. Tomba's verifier ($0.01) is cheaper for batch validation.

```json
{"email": "edan.harel@coderabbit.ai"}
```

Response: `result` (`deliverable` / `risky` / `undeliverable`) + `score`. Only report emails to the recruiter when `result === "deliverable"`.

---

## Minerva

Base URL: `https://stableenrich.dev`

Consumer identity graph. Use only when you **already have an email or phone** for the candidate — name + LinkedIn alone is not sufficient input.

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

Use only on the top 5 ranked candidates after a verified email is in hand.

### `POST /api/minerva/validate-emails` — $0.01

Bulk email check against Minerva DB. Cheaper than Hunter for pre-screening lists; Hunter is more authoritative for the final call.

---

## Firecrawl

Base URL: `https://stableenrich.dev`

### `POST /api/firecrawl/scrape` — $0.0126

Scrape a single URL to clean markdown. Use when a candidate's portfolio site is JS-heavy and Exa `contents` returns thin text.

```json
{"url": "https://alice.dev/about"}
```

### `POST /api/firecrawl/search` — $0.0252

Web search + scrape combined. Generally Exa `search` + `contents` is cheaper — use Firecrawl when Exa returns nothing relevant.

---

## Serper

Base URL: `https://stableenrich.dev`

### `POST /api/serper/news` — $0.04

Google News search. Useful for checking quotes, conference talks, recent moves on top 5 candidates.

```json
{"q": "Edan Harel CodeRabbit senior account executive", "num": 10, "gl": "us", "hl": "en"}
```

---

## Reddit

Base URL: `https://stableenrich.dev`

### `POST /api/reddit/search` — $0.02

Search Reddit for tech-community signals (r/ExperiencedDevs, r/MachineLearning, r/sales). Most hits are noise — use sparingly.

```json
{"query": "best engineers I've worked with at fintech", "sort": "relevance", "timeframe": "year", "maxResults": 20}
```

### `POST /api/reddit/post-comments` — $0.02

Get full post text + comments for a Reddit URL. Use for posts where the search preview is truncated.

---

## StableSocial

Base URL: `https://stablesocial.dev`

Optional. Use only when the role explicitly involves audience-building (DevRel, content, community, founding marketing). All endpoints are $0.06.

| Endpoint | Use case |
|----------|----------|
| `POST /api/instagram/profile` | Verify a candidate's IG follower count for content/community roles |
| `POST /api/tiktok/profile` | Same, for TikTok-native creators |
| `POST /api/reddit/search-profiles` | Find Reddit handles for community-manager candidates |

Skip entirely for IC engineering / standard GTM roles.

---

## Bazaar fallback

This skill ships with a curated list by design. Do **not** call Bazaar discovery during normal sourcing.

Exception: if the recruiter's brief is genuinely outside this catalog (e.g. "find drone-pilot candidates with FAA Part 107 certs"), run `npx awal@latest x402 bazaar search "<query>"` to see if a specialized endpoint exists. Surface the new endpoint to the user before paying.
