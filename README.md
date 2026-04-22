# recruit — a Claude Code skill for x402-powered candidate sourcing

A Claude Code plugin that gives an agent the ability to source candidates for a recruiter by orchestrating pay-per-call x402 endpoints (Apollo, Exa, Clado, Hunter, Minerva, and more). Payments settle automatically in USDC on Base from the user's [awal](https://www.npmjs.com/package/awal) agent wallet.

The agent's output is a ranked, evidence-backed markdown shortlist — not raw API dumps.

## What you get

- **Tech recruiting** — Exa-led sourcing across LinkedIn, GitHub, and personal sites; Apollo for "which orgs have these titles" universe scan.
- **GTM recruiting** — Exa LinkedIn search led; Apollo for company/title scan; Hunter guess-and-verify for emails.
- **Curated endpoint catalog** — vetted x402 v2 endpoints (Exa, Apollo, Hunter, Minerva, Firecrawl, Serper) with prices, body schemas, known bugs, and call patterns. Every endpoint behavior verified by direct test.
- **Cost transparency** — running USDC spend tally surfaced in the final report. Typical 20-candidate search: **$1–$3 USDC**.
- **Research only** — the skill does not send outreach. The recruiter handles that.
- **Optional advanced stack** — `references/advanced-orthogonal.md` documents the richer Tomba/Fiber/Nyne/Sixtyfour endpoints at `x402.orth.sh`, which require a raw private key + `x402-fetch` SDK (awal can't pay them yet — they're x402 v1).

## Install

```
npx skills add Must-be-Ash/recruit-skill
```

Then start a new session and the `recruit` skill is available.

## Prerequisites

This skill requires the **awal** agent wallet. If you don't have it, install it first:

```
npx skills add coinbase/agentic-wallet-skills
```

This ships the `authenticate-wallet` and `fund` skills. Run them to set up and top up your wallet before using `recruit`.

Then confirm you're ready:

- `npx awal@latest status` — confirm authenticated
- `npx awal@latest balance` — confirm USDC > $3 (typical 20-candidate search costs $1–$3)

## Usage

Just describe the role. Examples:

> Find me 20 staff backend engineers in NYC with strong distributed-systems chops at fintechs.

> Source enterprise AEs in Boston who've sold cybersecurity to F500 in the last 3 years.

> Build me a shortlist of 15 ML engineers shipping inference-optimization work — open-source contributors preferred.

The agent will:

1. Parse the brief (role type, seniority, location, must-haves).
2. Pick a playbook (tech vs GTM).
3. Run 2–4 sourcing calls in parallel (Exa LinkedIn-led for both).
4. Filter, dedupe, and rank.
5. Enrich the top 20 — `exa/contents` for headline + tenure, `apollo/org-enrich` once per company for the email domain, then guess email patterns and verify each via Hunter.
6. Deep-enrich the top 5 with `serper/news` and (if a verified email exists) Minerva for full work history.
7. Hand back a ranked markdown report with evidence per candidate and total spend.

## What's inside the skill

```
plugins/recruit/skills/recruit/
├── SKILL.md                       # workflow + when to use
├── references/
│   ├── endpoints.md               # curated x402 v2 endpoint catalog (awal-payable)
│   ├── playbooks.md               # tech vs GTM sourcing playbooks
│   └── advanced-orthogonal.md     # optional: x402.orth.sh stack via x402-fetch SDK
└── assets/
    └── report-template.md         # output report structure
```

## Limitations

- **Not for outreach.** Research only. If you need to send emails, do that yourself.
- **Not tuned for executive search.** IC through director is the sweet spot. C-suite searches need a different signal mix; the skill will warn the recruiter to confirm before proceeding.
- **No spending caps by default.** The skill trusts the awal wallet's balance as the ceiling and reports total spend in the output. Add `--max-amount` to individual calls if you want per-call limits.
- **awal is x402 v2 only.** The richer recruitment endpoints at `x402.orth.sh` (Tomba LinkedIn→email, Fiber NL search, etc.) are x402 v1 and cannot be paid by awal — see `references/advanced-orthogonal.md` for the workaround using `x402-fetch` + raw private key.
- **Endpoint prices and behavior may drift.** Two stableenrich endpoints currently return broken contact data (`apollo/people-enrich`) or 5xx-equivalent errors (`clado/contacts-enrich`); the skill works around both. If you find a fix, PR welcome.

## Contributing

Pull requests welcome. Especially:

- New curated endpoints (must be x402, return JSON, settle on Base).
- Playbook tweaks based on real recruiter feedback.
- Better ranking heuristics.

## License

MIT
