# Project 5 — Closed-Won Lookalike Account Extractor

HubSpot deal hits Closed-Won, lookalike pipeline builds itself — free-tier only, no Clay, no custom backend.

## Overview

```text
HubSpot deal -> Closed-Won webhook -> n8n gateway
  -> OpenRouter minimax (firmographic extract: industry, headcount, tech stack, region)
  -> Apollo company search (5-10 non-customer lookalikes, exclusion by domain)
  -> n8n waterfall per lookalike: Apollo people-search -> Hunter -> Prospeo -> Dropcontact -> ZeroBounce verify
  -> HubSpot Company + Contact + Deal association into `Target Lookalikes` pipeline
  -> Slack Block Kit summary to #gtm-lookalikes
```

Only accounts with valid verified email and `lookalike_score >= 50` enter the pipeline. Scoring gates are strict; everything else is quarantined with reason.

> **Constraint:** No Clay subscription. Portfolio §6 clay rule waived for this project — see docs/decisions.md D0. All enrichment is n8n-direct API calls, same pattern as 04-job-posting-intent-engine.

## Results

| Metric | Value |
|--------|-------|
| Trigger | HubSpot deal stage = Closed-Won (webhook) |
| Lookalikes per closed deal | 5-10 non-customers |
| Waterfall (free-tier) | Apollo (unlimited free) -> Hunter 50/mo -> Prospeo 75/mo -> Dropcontact 50 trial -> ZeroBounce 100 |
| Scoring | 0-100: industry 30 + headcount 25 + tech 20 + region 15 + decision-maker 10; >=70 immediate, 50-69 nurture, <50 discard |
| CRM write | `Target Lookalikes` pipeline: lookalike_score, source_deal_id, lookalike_tier, enrichment_status |
| Slack | Block Kit card top-3 + Review in HubSpot + qualify/disqualify callbacks |
| Demo scope | 2 fake Closed-Won deals x 5 lookalikes — well within all free tiers |

*Metrics populated after live run. See docs/decisions.md for credit math.*

## Stage 1 — Trigger & firmographic extract

**Objective.** Turn Closed-Won into structured firmographics without manual mapping.

**Step-by-step.**

1. Create fake Closed-Won deal in sandbox with rich props (industry, employee_count, tech_stack, region, company_domain).
2. Webhook fires on dealstage change.
3. n8n normalizes payload; OpenRouter minimax extracts firmographics into strict JSON; validate before Apollo query.

**Deliverables.** Webhook fixture, firmographic JSON fixture, extraction prompt in docs/schema.md.

**Output example.**

```json
{
  "industry": "511210",
  "headcount": 120,
  "tech_stack": ["HubSpot", "Salesforce"],
  "region": "US",
  "company_domain": "example.com"
}
```

## Stage 2 — Apollo lookalike sourcing

**Objective.** 5-10 non-customer companies matching the Closed-Won profile.

**Step-by-step.**

1. Apollo company search with NAICS + headcount ±20% + region + tech overlap filters.
2. Exclude existing customers by domain list.
3. If 0 results, widen (±40%, parent NAICS, drop tech) up to 2 rounds; else log no_results and skip.

**Deliverables.** Apollo request/response fixtures, widening log.

## Stage 3 — Person + email waterfall

**Objective.** One verified decision-maker per lookalike.

**Step-by-step.** Apollo people-search at lookalike domain -> Hunter domain search -> Prospeo via LinkedIn URL -> Dropcontact enrich -> ZeroBounce verify (api-us endpoint). First valid email wins.

**Deliverables.** Per-tier fixtures, quarantine on invalid/catch-all.

**Output example.** `apollo-people-search.json`, `zerobounce-verify.json` in tests/fixtures/.

## Stage 4 — CRM sync

**Objective.** Idempotent writes with dedup.

**Step-by-step.** Check Company by domain, Contact by email -> create/update -> associate Contact->Company -> create Deal in `Target Lookalikes` with custom props.

**Deliverables.** HubSpot property definitions, association calls (labels are Pro-gated, use default).

## Stage 5 — Slack dispatch

**Objective.** Sales-ready summary, not spam.

**Step-by-step.** Block Kit card to #gtm-lookalikes with top-3, scores, source deal, tier hit; buttons callback to n8n to update deal stage.

**Deliverables.** Slack payload fixture, callback handler.

## Failure-mode evidence

- Apollo zero results -> widening logged, skip after 2 rounds.
- Duplicate domain -> update-in-place, no duplicate Company.
- Invalid/catch-all email -> quarantined, zero CRM writes.
- ZeroBounce 1020 -> use api-us.zerobounce.net.
- (populate with screenshots/logs after live run)

## Repo map

```text
workflows/                  n8n workflow JSON (importable, scrubbed)
tests/fixtures/             real captured payloads (apollo, hunter, prospeo, zerobounce, hubspot)
docs/schema.md              field mappings: HubSpot Deal -> Apollo -> HubSpot Company/Contact
docs/runbook.md             rebuild-from-zero guide
docs/decisions.md           D0-D11 tradeoffs
docs/credentials.md         per-tool signup -> credential -> scopes
clay/                       empty — Clay waived (free-tier constraint)
assets/                     Loom link, Slack screenshots
```

## Build order

Per §5.2: (1) fake Closed-Won deal + firmographic props, (2) hand-run Apollo lookalike search, (3) lock thresholds numerically. Then n8n gateway per §7.3 passes.
