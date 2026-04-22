---
name: recruit
description: Run a deep candidate search for a recruiter using x402 pay-per-call endpoints (Apollo, Exa, Clado, Hunter, Minerva, social) on Base USDC via the awal wallet. Use when the user asks to source candidates, find applicants, build a shortlist, run a recruiting search, find engineers/developers/ML hires, find sales/marketing/BD hires, or "research who could fit role X". Output is a ranked, evidence-backed candidate report. Tech (engineering, ML) and GTM (sales, marketing, BD) recruiting are first-class; do not use for executive-only searches without the recruiter confirming.
---

# Recruit

## Overview

Source and enrich candidate shortlists for a recruiter by orchestrating a curated set of x402 pay-per-call endpoints. Payments are made automatically in USDC on Base from the user's awal wallet. Output is a ranked markdown report with evidence and contact data per candidate.

## Prerequisites

Before sourcing, confirm:

1. **Wallet authenticated**: `npx awal@2.0.3 status`. If not authed, run the `authenticate-wallet` skill.
2. **USDC balance sufficient**: `npx awal@2.0.3 balance`. A typical 25-candidate search costs **$2–$8 USDC** depending on enrichment depth. If balance is low, run the `fund` skill.

Do not proceed without both.

## Workflow

### Step 1 — Parse the brief

Extract from the recruiter's request:

- **Role type**: tech (engineer, ML, data, infra, security) or GTM (AE, SDR, marketing, BD, CSM, RevOps)
- **Seniority**: IC / senior / staff / lead / manager / director
- **Must-haves**: skills, domains, frameworks, industries, languages
- **Nice-to-haves**: tenure, education, prior employers
- **Location & work model**: city/region, remote/hybrid/onsite, work authorization if mentioned
- **Headcount target**: how many candidates to surface (default: 15–25)

If any must-have is ambiguous, ask **one** clarifying question before spending — the rest can be inferred.

### Step 2 — Pick the playbook

| Role type | Playbook |
|-----------|----------|
| Tech | `references/playbooks.md#tech` — Exa-first (GitHub, personal sites, LinkedIn), Apollo for filling in employer/title, Clado for LinkedIn → contact |
| GTM | `references/playbooks.md#gtm` — Apollo-first (people-search by title/company/location), Apollo enrich, Hunter verify |

Read the relevant playbook section before running calls. Each playbook lists the recommended call sequence, input templates, and ranking signals.

### Step 3 — Source (wide net)

Run 2–4 parallel sourcing calls based on the playbook. Aim for **40–80 raw hits** before filtering. Track running USDC spend in an internal counter.

### Step 4 — Filter and dedupe

- Drop hits missing the must-haves
- Dedupe by (LinkedIn URL || email || full name + current employer)
- Keep top **20–30** by initial fit signals (see playbook ranking heuristics)

### Step 5 — Enrich top N

For each survivor:

- **Contact**: Apollo `people-enrich` (cheapest at $0.0495) → if missing, Clado `contacts-enrich` ($0.20) as fallback
- **Verify email**: Hunter `email-verifier` ($0.03) on any discovered email before reporting it
- **Cross-reference** (tech only): Exa `contents` ($0.002) on the candidate's GitHub or personal site URL to pull a 1-line "what they build" signal

### Step 6 — Rank

Score each candidate 0–10 against the brief using the playbook's signal weights. Sort descending. Surface ties by recency of relevant activity.

### Step 7 — Write the report

Use `assets/report-template.md` as the structure. Include the running total spend at the bottom. Save to the user's working directory as `candidates-<role>-<YYYYMMDD>.md`.

## Calling endpoints with awal

All recruitment endpoints are called via:

```bash
npx awal@2.0.3 x402 pay <url> -X POST -d '<json-body>' --json
```

**Critical rules**:

- Always pass `--json` so the response parses cleanly. The awal CLI rejects text/markdown responses even when payment settles, so JSON-returning endpoints are mandatory — every endpoint in `references/endpoints.md` returns JSON.
- Single-quote JSON bodies to avoid shell expansion: `-d '{"q_keywords":"staff engineer"}'`.
- Do **not** pass `--max-amount` — the user has opted out of caps. Spend freely but report the total.
- Run independent calls in parallel via separate Bash tool invocations in one message.

Example — Apollo people search:

```bash
npx awal@2.0.3 x402 pay https://stableenrich.dev/api/apollo/people-search \
  -X POST \
  -d '{"q_keywords":"senior react engineer","person_locations":["San Francisco"],"per_page":25}' \
  --json
```

The full endpoint catalog (URLs, methods, input schemas, prices) lives in **`references/endpoints.md`** — read it before constructing any call.

## Cost tracking

Maintain a running tally as `Σ = sum of per-call prices from references/endpoints.md`. Surface the total in the final report. If the running total crosses $5 mid-search, briefly note it to the user but continue (no caps).

## Output

Always produce a markdown report following `assets/report-template.md`. Do not return raw JSON dumps to the user — synthesize.

## What this skill is not for

- **Outreach / email sending** — research only. If the recruiter asks to send messages, decline and suggest they handle outreach themselves.
- **Background checks / criminal records** — not in scope; the curated endpoints don't cover this.
- **Executive search** without recruiter confirmation — the sourcing depth tuned here is for IC through director, not C-suite.
