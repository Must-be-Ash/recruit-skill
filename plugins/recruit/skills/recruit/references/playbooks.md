# Recruiting Playbooks

Two playbooks, picked by role type. Read only the one that matches the brief.

## Tech

### Sourcing call sequence

For tech roles (engineering, ML, data, infra, security), Apollo alone undersells signal. The web — GitHub, blogs, LinkedIn — is where real evidence of capability lives. Lead with Exa.

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
4. **Apollo people-search** — backfill employer/title structure
   ```json
   {"q_keywords": "<seniority> <role> <skills>", "person_locations": [<location>], "per_page": 50}
   ```

Aim for **60–100 raw hits** across all four.

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
| Tenure at a recognized company doing similar work | 2 | Apollo / LinkedIn |
| Conference talks, meaningful OSS maintainership | 2 | Exa search + Serper news |
| Adjacent stack (e.g. Go when you wanted Rust) | 0.5–1 | LinkedIn / GitHub |
| Years of experience in range | 1 | Apollo / LinkedIn |

### Enrichment pass

For the **top 20** by ranking:

1. `apollo/people-enrich` (linkedin_url) — cheap baseline
2. If no email returned → `clado/contacts-enrich` (linkedin_url)
3. `hunter/email-verifier` on any email before including it
4. `exa/contents` on the candidate's top GitHub repo or blog post → 1-line "what they build" summary

For the **top 5** only:

5. `minerva/enrich` for full work history + education detail

### Tech-specific gotchas

- **Recency over prestige**: a recent contributor to a relevant OSS project beats an ex-FAANG engineer who hasn't shipped in 3 years.
- **GitHub != real code**: lots of orgs do private work. Treat sparse GitHub as neutral, not negative, for senior+ candidates with strong employer signals.
- **Exa LinkedIn search returns public profile snippets, not full data** — you'll still need Apollo or Clado to confirm current role.

---

## GTM

### Sourcing call sequence

For GTM roles (AE, SDR, marketing, BD, CSM, RevOps, partnerships), Apollo is the workhorse. Title + company + location filters are precise enough that you rarely need to leave it.

Run **in parallel**:

1. **Apollo org-search** — find the right target companies first
   ```json
   {"q_keywords": "<industry/segment> <stage>", "organization_locations": [<location>], "per_page": 25}
   ```
   Take the top ~15 org IDs from the response.

2. **Apollo people-search with `organization_ids`** — pull people at those orgs
   ```json
   {"q_keywords": "<title>", "organization_ids": [<top org IDs>], "per_page": 50}
   ```

3. **Apollo people-search by title alone** (no org filter) — catches good people at orgs you didn't pre-select
   ```json
   {"q_keywords": "<exact title> <industry>", "person_locations": [<location>], "per_page": 50}
   ```

4. **Exa LinkedIn profile search** (optional) — sometimes catches founders, advisors, and people Apollo hasn't fully indexed
   ```json
   {"query": "<title> <industry> <location>", "numResults": 15, "category": "linkedin profile"}
   ```

Aim for **50–80 raw hits**.

### Filtering signals (drop these)

- People whose current title doesn't match the seniority (e.g. searched "enterprise AE", got "BDR")
- Tenure < 12 months at current role AND < 18 months at previous role (job-hopper unless brief calls for it)
- Wrong industry segment when industry is in the must-haves
- Location mismatch (no "Remote" tag and wrong city)

### Ranking signals (weight ~)

| Signal | Weight | Source |
|--------|--------|--------|
| Closed enterprise deals at a peer company (inferred from title progression + employer) | 3 | Apollo enrich + Minerva |
| Tenure ≥ 2 years at most recent role (stability) | 2 | Apollo |
| Worked at a recognized peer / target-list company | 2 | Apollo enrich (org-enrich peer companies) |
| Title progression upward over career | 1.5 | Minerva work_history |
| Adjacent industry (e.g. cybersecurity when you wanted devtools — both technical B2B) | 1 | Apollo |
| Education / pedigree (only if recruiter flagged it) | 0.5 | Minerva |

### Enrichment pass

For the **top 20**:

1. `apollo/people-enrich` (linkedin_url or person_id)
2. If no email → `clado/contacts-enrich`
3. `hunter/email-verifier` on every email
4. `apollo/org-enrich` on each candidate's current employer (only if recruiter wants firmographics in the report)

For the **top 5**:

5. `minerva/enrich` with `return_fields: ["work_history", "education"]` for the full career arc
6. `serper/news` to check if the candidate has been quoted or made news in the role's industry — strong relevance signal

### GTM-specific gotchas

- **Title inflation is real**: an "AE" at a 50-person startup may be doing SDR work; a "BDR" at a $1B public co may be ramping for enterprise AE. Look at company stage, not just the title string.
- **Apollo email coverage varies by region**: US is ~80%+, EMEA is patchier. Don't be surprised if Clado fallback is needed more for international searches.
- **Org IDs are the unlock**: a single `org-search` call ($0.02) followed by people searches scoped to those orgs is dramatically better than free-text-only people search.
