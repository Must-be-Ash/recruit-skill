---
name: recruit
description: Run a deep candidate search for a recruiter using x402 pay-per-call endpoints (Exa, Apollo, Hunter, Minerva, Firecrawl) on Base USDC via the awal wallet. Use when the user asks to source candidates, find applicants, build a shortlist, run a recruiting search, find engineers/developers/ML hires, find sales/marketing/BD hires, or "research who could fit role X". Output is a ranked, evidence-backed candidate report. Tech (engineering, ML) and GTM (sales, marketing, BD) recruiting are first-class; do not use for executive-only searches without the recruiter confirming.
---

# Recruit

## Overview

Source and enrich candidate shortlists for a recruiter by orchestrating a curated set of x402 pay-per-call endpoints. Payments are made automatically in USDC on Base from the user's awal wallet. Output is a ranked markdown report with evidence and contact data per candidate.

## Prerequisites

Before sourcing, confirm:

1. **Wallet authenticated**: `npx awal@latest status`. If not authed, run the `authenticate-wallet` skill.
2. **USDC balance sufficient**: `npx awal@latest balance`. A typical 20-candidate search costs **$1–$3 USDC**. If balance is low, run the `fund` skill.

Do not proceed without both.

## Compatibility note

**The awal CLI pays x402 v2 endpoints. The vetted v2 endpoints are at `https://stableenrich.dev/...`** — those are what this skill uses end-to-end.

There is a richer set of recruitment endpoints (Tomba LinkedIn→email, Fiber natural-language profile search, Apollo via Orthogonal with a working contact reveal, Nyne, Sixtyfour) at `https://x402.orth.sh/...`, but those are x402 **v1** and currently fail when paid via awal. If the agent wants to use them, see `references/advanced-orthogonal.md` — that path requires running a small Node script with `x402-fetch` and a raw `PRIVATE_KEY`, not awal. Skip unless the recruiter explicitly asks.

## Workflow

### Step 1 — Parse the brief

Extract from the recruiter's request:

- **Role type**: tech (engineer, ML, data, infra, security) or GTM (AE, SDR, marketing, BD, CSM, RevOps)
- **Seniority**: IC / senior / staff / lead / manager / director
- **Must-haves**: skills, domains, frameworks, industries, languages
- **Nice-to-haves**: tenure, education, prior employers
- **Location & work model**: city/region, remote/hybrid/onsite
- **Headcount target**: how many candidates to surface (default: 15–20)

If any must-have is ambiguous, ask **one** clarifying question before spending — the rest can be inferred.

### Step 2 — Pick the playbook

| Role type | Playbook |
|-----------|----------|
| Tech | `references/playbooks.md#tech` — Exa-first across LinkedIn, GitHub, personal sites; Apollo for universe scan; Hunter guess+verify for emails |
| GTM | `references/playbooks.md#gtm` — Exa LinkedIn search lead; Apollo people-search for company/title scan; Hunter guess+verify for emails |

Both playbooks lead with **Exa LinkedIn search** because it's the only call that reliably returns full real names + LinkedIn URLs through stableenrich. Apollo `people-search` is useful as a universe scan but returns obfuscated names (`Du***g`, `O'***n`) and its enrichment endpoint does not currently expose contact data via stableenrich.

Read the relevant playbook section before running calls.

### Step 3 — Source (wide net)

Run 2–4 parallel sourcing calls based on the playbook. Aim for **40–80 raw hits** before filtering. Track running USDC spend in an internal counter.

### Step 4 — Filter and dedupe

- Drop hits missing the must-haves
- Dedupe by (LinkedIn URL || full name + current employer)
- Keep top **20** by initial fit signals (see playbook ranking heuristics)

### Step 5 — Enrich top N

For each survivor:

- **Confirm current company / title**: `exa/contents` ($0.002) on the LinkedIn URL → pull headline & current role
- **Discover email** via guess + verify (no working LinkedIn → email endpoint via awal):
  1. Get the company domain (via Apollo `org-enrich` $0.0495 once per unique company, then cache)
  2. Generate 2–3 likely email patterns: `firstname.lastname@domain`, `firstname@domain`, `flastname@domain`
  3. `hunter/email-verifier` ($0.03) on each guess
  4. Keep the first `result: "deliverable"`. If none verify, mark candidate as "LinkedIn only — recruiter to use InMail"
- **Optional cross-reference** (tech only): `exa/contents` on the candidate's GitHub or blog ($0.002) for a 1-line "what they build" snippet

For the **top 5 only**: `serper/news` ($0.04) — surfaces recent quotes, conference talks, or hiring news that strengthens ranking.

### Step 6 — Rank

Score each candidate 0–10 against the brief using the playbook's signal weights. Sort descending. Surface ties by recency of relevant activity.

### Step 7 — Write the report

Use `assets/report-template.md` as the structure. Include the running total spend at the bottom. Save to the user's working directory as `candidates-<role>-<YYYYMMDD>.md`.

## Calling endpoints with awal

All recruitment endpoints are called via:

```bash
npx awal@latest x402 pay <url> -X POST -d '<json-body>' --json
```

**Critical rules**:

- Always pass `--json` so the response parses cleanly. The awal CLI rejects text/markdown responses even when payment settles, so JSON-returning endpoints are mandatory — every endpoint in `references/endpoints.md` returns JSON.
- Single-quote JSON bodies to avoid shell expansion: `-d '{"person_titles":["Account Executive"]}'`.
- Do **not** pass `--max-amount` — the user has opted out of caps. Spend freely but report the total.
- Run independent calls in parallel via separate Bash tool invocations in one message.
- The awal CLI writes results to stdout but the format includes a leading status line; parse JSON starting at the first `{`.

Example — Exa LinkedIn search:

```bash
npx awal@latest x402 pay https://stableenrich.dev/api/exa/search \
  -X POST \
  -d '{"query":"senior account executive developer tools San Francisco","numResults":25,"category":"linkedin profile"}' \
  --json
```

The full endpoint catalog (URLs, methods, input schemas, prices, known bugs) lives in **`references/endpoints.md`** — read it before constructing any call.

## Cost tracking

Maintain a running tally as `Σ = sum of per-call prices from references/endpoints.md`. Surface the total in the final report. If the running total crosses $5 mid-search, briefly note it to the user but continue (no caps).

## Output

Always produce a markdown report following `assets/report-template.md`. Do not return raw JSON dumps to the user — synthesize.

## What this skill is not for

- **Outreach / email sending** — research only. If the recruiter asks to send messages, decline and suggest they handle outreach themselves.
- **Background checks / criminal records** — not in scope; the curated endpoints don't cover this.
- **Executive search** without recruiter confirmation — the sourcing depth tuned here is for IC through director, not C-suite.
