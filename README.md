# recruit — a Claude Code skill for x402-powered candidate sourcing

A Claude Code plugin that gives an agent the ability to source candidates for a recruiter by orchestrating pay-per-call x402 endpoints (Apollo, Exa, Clado, Hunter, Minerva, and more). Payments settle automatically in USDC on Base from the user's [awal](https://www.npmjs.com/package/awal) agent wallet.

The agent's output is a ranked, evidence-backed markdown shortlist — not raw API dumps.

## What you get

- **Tech recruiting** — Exa-led sourcing across GitHub, personal sites, and LinkedIn, with Apollo backfilling employer/title and Clado resolving LinkedIn → contact.
- **GTM recruiting** — Apollo-led sourcing (org-search → people-search by `organization_ids`), with Hunter verifying every email before it's reported.
- **Curated endpoint catalog** — vetted list of recruiting-relevant x402 endpoints with prices, body schemas, and call patterns. No live Bazaar fishing during normal sourcing.
- **Cost transparency** — running USDC spend tally surfaced in the final report.
- **Research only** — the skill does not send outreach. The recruiter handles that.

## Install

```
/plugin marketplace add Must-be-Ash/recruit-skill
/plugin install recruit@must-be-ash
```

Then start a new session (or `/reload-plugins`) and the `recruit` skill is available.

## Prerequisites

This plugin assumes the agent has the **awal** wallet skills installed and an authenticated wallet with USDC on Base:

- `npx awal@2.0.3 status` — confirm authenticated
- `npx awal@2.0.3 balance` — confirm USDC > $5 (typical 25-candidate search costs $2–$8)

If the wallet isn't set up, the agent should run the `authenticate-wallet` and `fund` skills first (both ship as part of the standard awal skill bundle).

## Usage

Just describe the role. Examples:

> Find me 20 staff backend engineers in NYC with strong distributed-systems chops at fintechs.

> Source enterprise AEs in Boston who've sold cybersecurity to F500 in the last 3 years.

> Build me a shortlist of 15 ML engineers shipping inference-optimization work — open-source contributors preferred.

The agent will:

1. Parse the brief (role type, seniority, location, must-haves).
2. Pick a playbook (tech vs GTM).
3. Run 2–4 sourcing calls in parallel.
4. Filter, dedupe, and rank.
5. Enrich the top 20 (Apollo → Clado fallback for contact, Hunter for email verification).
6. Deep-enrich the top 5 (Minerva work history + education).
7. Hand back a ranked markdown report with evidence per candidate and total spend.

## What's inside the skill

```
plugins/recruit/skills/recruit/
├── SKILL.md                  # workflow + when to use
├── references/
│   ├── endpoints.md          # curated x402 endpoint catalog
│   └── playbooks.md          # tech vs GTM sourcing playbooks
└── assets/
    └── report-template.md    # output report structure
```

## Limitations

- **Not for outreach.** Research only. If you need to send emails, do that yourself.
- **Not tuned for executive search.** IC through director is the sweet spot. C-suite searches need a different signal mix; the skill will warn the recruiter to confirm before proceeding.
- **No spending caps by default.** The skill trusts the awal wallet's balance as the ceiling and reports total spend in the output. Add `--max-amount` to individual calls if you want per-call limits.
- **Endpoint prices may drift.** The catalog reflects prices at the time of writing. Treat them as estimates.

## Contributing

Pull requests welcome. Especially:

- New curated endpoints (must be x402, return JSON, settle on Base).
- Playbook tweaks based on real recruiter feedback.
- Better ranking heuristics.

## License

MIT
