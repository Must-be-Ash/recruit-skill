# Recruiting Playbooks

Two playbooks, picked by role type. Read only the one that matches the brief.

Both playbooks lead with **Exa LinkedIn search** because — verified through testing 2026-04 — Apollo's enrich endpoint via stableenrich does not currently return contact data, and Clado is server-side broken. Exa returns real names + LinkedIn URLs that subsequent calls (Apollo `org-enrich` for the email domain, Hunter for verification) can build on.

## Tech

### Sourcing call sequence

For tech roles (engineering, ML, data, infra, security), web evidence (GitHub, blogs, LinkedIn) beats firmographic-only signals.

Run **in parallel** (one Bash message, multiple `awal x402 pay` calls):

1. **Exa LinkedIn profile search** — primary credentialing signal
   ```json
   {"query": "<seniority> <role> <key skills> <location?>", "numResults": 25, "category": "linkedin profile"}
   ```
2. **Exa GitHub search** — primary capability signal
   ```json
   {"query": "<key skills/frameworks> <domain>", "numResults": 25, "category": "github"}
   ```
3. **Exa personal site search** — finds engineers who write (a strong positive signal)
   ```json
   {"query": "<role> <skills> blog", "numResults": 15, "category": "personal site"}
   ```
4. **Apollo people-search** (universe scan) — see which orgs have these titles in this region
   ```json
   {"person_titles": ["<Title 1>", "<Title 2>"], "person_locations": [<location>], "per_page": 25}
   ```
   Names come back obfuscated — use the `first_name + organization.name` to seed an Exa LinkedIn lookup for the full name.

Aim for **60–100 raw hits** across all four. Total sourcing cost ~$0.06.

### Filtering signals (drop these)

- LinkedIn profiles with no listed work history or <2 years total experience (when sourcing senior+)
- GitHub accounts with no commits in the last 18 months and no pinned repos
- Personal sites that haven't been touched since 2021
- Apollo hits whose `title` is wildly off (e.g. you searched "ML engineer", got "ML manager") — keep adjacency only if seniority matches

### Ranking signals (weight ~)

| Signal | Weight | Source |
|--------|--------|--------|
| Recent (≤12mo) commits in directly relevant repos | 3 | Exa GitHub + `contents` on top repo |
| Public writing on the exact problem domain | 2.5 | Exa personal site + `contents` on top post |
| Tenure at a recognized company doing similar work | 2 | LinkedIn (via Exa) + Apollo |
| Conference talks, meaningful OSS maintainership | 2 | Exa search + Serper news |
| Adjacent stack (e.g. Go when you wanted Rust) | 0.5–1 | LinkedIn / GitHub |
| Years of experience in range | 1 | LinkedIn (via Exa contents) |

### Enrichment pass

For the **top 20** by ranking:

1. `exa/contents` ($0.002) on the LinkedIn URL → headline, current company, brief summary
2. **Email discovery via guess + verify** (no working LinkedIn → email endpoint via awal):
   - `apollo/org-enrich` ($0.0495) once per unique current company → get the email domain
   - Build 2–3 guesses: `firstname.lastname@domain`, `firstname@domain`, `flastname@domain`
   - `hunter/email-verifier` ($0.03) on each guess in parallel
   - Keep the first `result: "deliverable"`. If none verify, mark "LinkedIn only — recruiter to InMail"
3. `exa/contents` ($0.002) on the candidate's top GitHub repo or blog → 1-line "what they build" summary

For the **top 5** only:

4. `serper/news` ($0.04) → check for conference talks, quotes, or recent moves
5. `minerva/resolve` + `minerva/enrich` ($0.07 total) only **after** a verified email is in hand → full work history + education

Per-candidate enrichment cost in the top 20: ~$0.10 typical. Top 5 deep dive: another ~$0.20 per.

### Tech-specific gotchas

- **Recency over prestige**: a recent contributor to a relevant OSS project beats an ex-FAANG engineer who hasn't shipped in 3 years.
- **GitHub != real code**: lots of orgs do private work. Treat sparse GitHub as neutral, not negative, for senior+ candidates with strong employer signals.
- **Exa `linkedin profile` returns rich snippets**: title strings often follow `"Full Name - Role @ Company"`. Parse the title for current employer when present — saves an `exa/contents` call.

---

## GTM

### Sourcing call sequence

For GTM roles (AE, SDR, marketing, BD, CSM, RevOps, partnerships), the working playbook is **Exa LinkedIn search led**, with Apollo as a universe-scan supplement. Apollo via stableenrich can't reveal contact data right now (see `endpoints.md`), so Exa drives candidate discovery.

Run **in parallel**:

1. **Exa LinkedIn profile search** — primary call, returns full real names + LinkedIn URLs
   ```json
   {"query": "<title> <industry/company-type> <location>", "numResults": 25, "category": "linkedin profile"}
   ```
   Run 2–3 variants with different industry phrasings (e.g. "developer tools", "API platform", "infrastructure") to broaden the net.

2. **Exa LinkedIn search per known target company** — when the recruiter has a target list
   ```json
   {"query": "<title> <CompanyName> LinkedIn", "numResults": 10, "category": "linkedin profile"}
   ```

3. **Apollo people-search by title** — universe scan: which companies in this region have these titles?
   ```json
   {"person_titles": ["<Title 1>", "<Title 2>"], "person_locations": [<location>], "per_page": 25}
   ```
   **Critical**: use `person_titles` (array), NOT `q_keywords`. Combining them returns 0 hits. Names come back obfuscated — feed `first_name + organization.name` back into an Exa LinkedIn search to recover the full name.

4. **Apollo org-search** (optional) — only when scoping to specific company stages/industries before searching people
   ```json
   {"q_keywords": "<industry/segment>", "organization_locations": [<location>], "per_page": 15}
   ```
   Note: keyword filtering is weak. Don't rely on it for tight industry targeting.

Aim for **50–80 raw hits**. Total sourcing cost ~$0.05–$0.08.

### Filtering signals (drop these)

- Current title doesn't match the seniority (e.g. searched "enterprise AE", got "BDR")
- Tenure < 12 months at current role AND < 18 months at previous role (job-hopper unless brief calls for it)
- Wrong industry segment when industry is in the must-haves
- Location mismatch (no "Remote" tag and wrong city)

### Ranking signals (weight ~)

| Signal | Weight | Source |
|--------|--------|--------|
| Title progression at peer companies (proxies for closed enterprise deals) | 3 | LinkedIn title parsing + Apollo employer signal |
| Tenure ≥ 2 years at most recent role | 2 | Exa contents on LinkedIn |
| Worked at a recognized peer / target-list company | 2 | Apollo `org-enrich` on current company |
| Title progression upward over career | 1.5 | Exa contents on LinkedIn |
| Adjacent industry (e.g. cybersecurity when you wanted devtools) | 1 | LinkedIn / Apollo |
| Education / pedigree (only if recruiter flagged it) | 0.5 | Minerva (after email verified) |

### Enrichment pass

For the **top 20**:

1. `exa/contents` ($0.002) on the LinkedIn URL → headline, current company, tenure
2. `apollo/org-enrich` ($0.0495) once per unique current company → email domain + firmographics
3. **Email guess + verify**:
   - Build 2–3 guesses: `firstname.lastname@domain`, `firstname@domain`, `flastname@domain`
   - `hunter/email-verifier` ($0.03) on each in parallel
   - Keep the first deliverable; mark "LinkedIn only" if none

For the **top 5**:

4. `serper/news` ($0.04) → quotes, recent moves, industry mentions
5. `minerva/resolve` + `minerva/enrich` ($0.07) only after verified email → full work history + demographics

Per-candidate enrichment cost in the top 20: ~$0.10 typical (less if multiple candidates share a company, since `org-enrich` caches).

### GTM-specific gotchas

- **Title inflation is real**: an "AE" at a 50-person startup may be doing SDR work; a "BDR" at a $1B public co may be ramping for enterprise AE. Look at company stage (from `org-enrich`), not just the title string.
- **Email patterns vary by company size**: Big-co (Salesforce, Oracle) almost always uses `firstname.lastname@`. Smaller startups often use `firstname@` or `flastname@`. Try `firstname@` first for sub-100-person startups, `firstname.lastname@` first for everything else.
- **Apollo org-search keyword filter is weak** — when scoping to a specific industry, prefer running Exa LinkedIn searches with the industry phrase in the query rather than trying to filter Apollo orgs by keyword.
