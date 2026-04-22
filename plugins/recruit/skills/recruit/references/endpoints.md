# Curated Endpoint Catalog

All endpoints below are x402 pay-per-call, settle in USDC on Base, and return JSON. Call them via `npx awal@2.0.3 x402 pay <url> -X POST -d '<json>' --json`.

Prices are per single call. They may drift — treat as estimates.

## Table of contents

1. [Apollo (people & company search/enrich)](#apollo)
2. [Exa (semantic web search & content extraction)](#exa)
3. [Clado (LinkedIn → contact enrichment)](#clado)
4. [Hunter (email verification)](#hunter)
5. [Minerva (person identity resolution)](#minerva)
6. [Firecrawl (web scraping)](#firecrawl)
7. [Serper (Google news search)](#serper)
8. [Reddit (StableEnrich)](#reddit-stableenrich)
9. [Social profile data (StableSocial)](#stablesocial)
10. [When to add Bazaar discovery](#bazaar-fallback)

---

## Apollo

Base URL: `https://stableenrich.dev`

Apollo is the workhorse for **GTM** sourcing (titles, companies, locations) and a strong assist for **tech** sourcing once you have a name or company.

### `POST /api/apollo/people-search` — $0.02

Find people by title / keyword / location / employer. **Primary GTM sourcing call.**

Body fields:

| Field | Type | Notes |
|-------|------|-------|
| `q_keywords` | string | Free text. Combine title + skill, e.g. `"staff backend engineer python"` |
| `person_locations` | string[] | City names or regions, e.g. `["San Francisco", "Remote"]` |
| `organization_ids` | string[] | Apollo org IDs (get from `org-search` first) |
| `per_page` | number | Up to 100. Use 25 for first pass, 100 for deep dives |
| `page` | number | Pagination |

Example:

```json
{"q_keywords": "enterprise account executive cybersecurity", "person_locations": ["New York", "Boston"], "per_page": 50}
```

Response shape (truncated): array of people with `name`, `title`, `organization`, `linkedin_url`, sometimes masked email.

### `POST /api/apollo/people-enrich` — $0.0495

Pull full profile + contact for a known person. **Cheapest enrichment — try this first.**

Body: pass at least one of `email`, `linkedin_url`, `person_id`, or `first_name`+`last_name`+`organization_name`.

```json
{"linkedin_url": "https://www.linkedin.com/in/janedoe"}
```

### `POST /api/apollo/org-search` — $0.02

Find target companies. Useful when sourcing "engineers at fintech Series B in NYC" — find the orgs first, then `people-search` with `organization_ids`.

```json
{"q_keywords": "fintech series b", "organization_locations": ["New York"], "per_page": 25}
```

### `POST /api/apollo/org-enrich` — $0.0495

Pull firmographics (headcount, funding, tech stack) for one company by domain.

```json
{"domain": "stripe.com"}
```

---

## Exa

Base URL: `https://stableenrich.dev`

Exa is the **primary tech sourcing** tool. Its `category` filter lets you target GitHub, LinkedIn profiles, personal sites, and research papers directly.

### `POST /api/exa/search` — $0.01

Semantic web search with a category filter. **Cheap — run several variants.**

Body:

| Field | Type | Notes |
|-------|------|-------|
| `query` | string | Natural language, e.g. `"engineers building Rust crypto wallets blog"` |
| `numResults` | number | Default 10, up to 100 |
| `category` | string | One of: `"company"`, `"research paper"`, `"news"`, `"pdf"`, `"github"`, `"tweet"`, `"personal site"`, `"linkedin profile"`, `"financial report"` |

Tech sourcing patterns:

```json
{"query": "senior ML engineer transformers inference optimization", "numResults": 20, "category": "linkedin profile"}
{"query": "open source contributors Rust async runtime", "numResults": 20, "category": "github"}
{"query": "engineer blog distributed systems observability", "numResults": 15, "category": "personal site"}
```

### `POST /api/exa/contents` — $0.002

Extract clean content from URLs you already have. **Dirt cheap — use it to read the top of a candidate's GitHub README or blog before ranking.**

```json
{"urls": ["https://github.com/alice/awesome-repo", "https://alice.dev"]}
```

### `POST /api/exa/find-similar` — $0.01

Given one strong-fit candidate URL, find similar profiles. Useful to expand a shortlist when the recruiter says "more like this person".

```json
{"url": "https://www.linkedin.com/in/alice-engineer", "numResults": 15}
```

### `POST /api/exa/answer` — $0.01

AI-generated answer over web search. Useful for one-shot questions like "who are the most-cited authors on retrieval-augmented generation 2024-2025" to seed a search — but verify hits with other endpoints before trusting.

---

## Clado

Base URL: `https://stableenrich.dev`

### `POST /api/clado/contacts-enrich` — $0.20

Resolve LinkedIn URL → email + phone. **Use as the fallback when Apollo `people-enrich` returns no email.** Pricier but specifically tuned for LinkedIn → contact.

Body: pass exactly one of `linkedin_url`, `email`, or `phone`.

```json
{"linkedin_url": "https://www.linkedin.com/in/janedoe"}
```

---

## Hunter

Base URL: `https://stableenrich.dev`

### `POST /api/hunter/email-verifier` — $0.03

Check if a discovered email is deliverable. **Always run before reporting an email to the recruiter** — bad emails make the recruiter look unprofessional.

```json
{"email": "jane@stripe.com"}
```

Response includes `result` (deliverable / risky / undeliverable) and `score`.

---

## Minerva

Base URL: `https://stableenrich.dev`

Minerva is broader than Apollo — it pulls demographics, work history, contact, sometimes financial signals. Use for **deep enrichment** on the top 5 candidates only (keeps spend reasonable).

### `POST /api/minerva/resolve` — $0.02

Resolve a person to a Minerva PID + LinkedIn URL.

```json
{"records": [{"first_name": "Jane", "last_name": "Doe", "emails": ["jane@stripe.com"]}]}
```

### `POST /api/minerva/enrich` — $0.05

Full enrichment given a `minerva_pid`, `linkedin_url`, or name+email/phone.

```json
{"records": [{"linkedin_url": "https://www.linkedin.com/in/janedoe"}], "return_fields": ["work_history", "education", "demographics"]}
```

### `POST /api/minerva/validate-emails` — $0.01

Bulk check emails against Minerva DB. Cheaper than Hunter for batch validation if you're checking >5 at once.

```json
{"records": ["jane@stripe.com", "alice@vercel.com"]}
```

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
{"q": "Jane Doe Stripe staff engineer", "num": 10, "gl": "us", "hl": "en"}
```

---

## Reddit (StableEnrich)

Base URL: `https://stableenrich.dev`

### `POST /api/reddit/search` — $0.02

Search Reddit for tech-community signals — devs talking shop in r/ExperiencedDevs, r/ChatGPTCoding, etc. Sometimes surfaces under-the-radar contributors.

```json
{"query": "best engineers I've worked with at fintech", "sort": "relevance", "timeframe": "year", "maxResults": 20}
```

Use sparingly — most hits are noise.

---

## StableSocial

Base URL: `https://stablesocial.dev`

For most recruiting, StableSocial is **optional** — only pull social data when the role explicitly involves audience-building (DevRel, content, community, founding marketing). All endpoints are $0.06.

Useful endpoints:

| Endpoint | Use case |
|----------|----------|
| `POST /api/instagram/profile` | Verify a candidate's IG follower count for content/community roles |
| `POST /api/tiktok/profile` | Same, for TikTok-native creators |
| `POST /api/reddit/search-profiles` | Find Reddit handles for community-manager candidates |

Skip StableSocial entirely for IC engineering / standard GTM roles.

---

## Bazaar fallback

This skill ships with a **curated list** by design — predictable cost, fast, vetted. Do **not** call the Bazaar discovery endpoints during normal sourcing.

The one exception: if the recruiter's brief is genuinely outside this catalog (e.g. "find me drone-pilot candidates with FAA Part 107 certs"), then and only then run `npx awal@2.0.3 x402 bazaar search "<query>"` to see if a specialized endpoint exists. Note the new endpoint to the user before paying.
