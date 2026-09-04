# Closed-Won Lookalike Engine

HubSpot deal hits Closed-Won, lookalike pipeline builds itself — free-tier only, no Clay, no custom backend.

## Project Overview

Sales teams close a deal and then manually hunt for similar companies. That list is slow, biased, and never makes it back into the CRM as a system.

This automation solves that by watching HubSpot for a Closed-Won deal, extracting firmographics (industry, headcount band, tech stack, region), searching Tavily/Apollo for 5-10 non-customer lookalikes, running a cheap-first email waterfall per lookalike, scoring 0-100, and creating Company+Contact+Deal in a dedicated Target Lookalikes pipeline with a Slack summary.

Only accounts with valid verified email and lookalike_score >=50 enter the pipeline.

## The Problem

Without an automated engine:

- Closed-Won learnings die in a spreadsheet
- Reps hand-pick "similar" accounts with no scoring
- No exclusion of existing customers by domain
- Unverified emails enter CRM and bounce
- No pipeline to track lookalike conversion
- Sales gets no Slack context (source deal, tier, score)

## The Solution

This n8n workflow automatically:

1. Receives HubSpot webhook on dealstage = Closed-Won
2. Extracts firmographics via OpenRouter minimax into strict JSON (industry NAICS, headcount, tech_stack, region) — validate before query
3. Searches lookalikes: Tavily Search Lookalikes + Parse + Has Lookalikes? (fallback seed if 0). Apollo variant 403 on free tier, so Tavily is primary.
4. Per lookalike (first extracted):
   - Hunter domain search (limit 10, 50/mo free) -> Select Best Person (first email)
   - Hunter Found? --NO--> Prospeo via LinkedIn URL (fallback, 429 handled)
   - ZeroBounce verify via api-us.zerobounce.net (only valid proceeds)
   - Gate: valid OR hunter_raw.verification.status==valid else quarantine email_not_valid
5. Scores 0-100: industry 30 + headcount 25 (within 20%) + tech 20 + region 15 + decision-maker 10. >=70 immediate, 50-69 nurture, <50 discard
6. HubSpot dedup: Search Company by domain -> Prepare Upsert (standard props only to avoid 400) -> Upsert Company -> Search Contact by email -> Upsert Contact -> Associate -> Respond HubSpot Synced
7. Slack summary to #all-fafo (C0BN1L6KDBR, #gtm-lookalikes fallback) — text mode, onError continueRegularOutput so 429 never blocks HubSpot

## Workflow

HubSpot Closed-Won webhook
      -> Firmographic Extract (minimax)
      -> Tavily Search Lookalikes (5-10)
      -> Extract First Lookalike
      -> Hunter Domain Search -> Select Best Person
      -> ZeroBounce Verify -> Valid?
           YES          NO
            |            |
         Score 0-100   Quarantine
            |
      HubSpot Company+Contact+Deal (Target Lookalikes)
            |
      Slack Summary (#all-fafo)

## Technologies Used

- n8n — 33 nodes, workflow RIhpPf9paOwX2sNs (live on Railway)
- OpenRouter minimax/m3:free — Firmographic extraction
- Tavily — Lookalike company search + parse (primary, free tier)
- Apollo — People search (fallback, unlimited free but mixed_companies blocked on free)
- Hunter — Domain search (50/mo)
- Prospeo — LinkedIn URL email fallback (429 handled)
- Dropcontact — Enrichment fallback
- ZeroBounce — Verify (api-us endpoint, 100 free)
- HubSpot — Sandbox 247137733, Target Lookalikes pipeline, Company+Contact+Deal
- Slack — Summary card (slackApi, text mode)

## HubSpot Fields

Company: name, domain
Deal (Target Lookalikes): lookalike_score, source_deal_id, lookalike_tier, enrichment_status, plus standard dealstage
Contact: email, name, lookalike_score reference

Scoring gates are strict; everything else quarantined with reason.

## Example Output

```
{
  "industry": "511210",
  "headcount": 120,
  "tech_stack": ["HubSpot", "Salesforce"],
  "region": "US",
  "lookalike_score": 90,
  "lookalike_tier": "immediate",
  "company_domain": "linear.app",
  "contact_email": "conor@linear.app",
  "email_status": "valid"
}
```

Quarantine: `quarantined: true, reason: email_not_valid`

## Follow-Up Logic

- Apollo 403 or 0 results -> widen filters (+-40%, parent NAICS) up to 2 rounds else log no_results and use fallback seed.
- Duplicate domain -> update-in-place, no duplicate Company.
- Invalid/catch-all -> quarantined, zero CRM writes.
- ZeroBounce 1020 on Railway -> use api-us.zerobounce.net.

## Example Scenario

Deal linear.app closes Won with Software/120/HubSpot/US -> Tavily finds 5 lookalikes -> notion.so extracted -> Hunter finds d.okonkwo@notion.so (conf 84) -> ZeroBounce invalid -> quarantined. Next run linear.app first -> Hunter conor@linear.app valid -> score 90 -> Company+Contact created -> Deal in Target Lookalikes -> Slack posted (exec 366).

## Demo

- Closed-Won webhook (firmographics_ready 0.95)
- Tavily lookalikes (5) + Hunter (10 candidates) + ZeroBounce
- Scoring (90 immediate)
- HubSpot synced + Slack posted
- Quarantine path (invalid does_not_accept_mail)

Add Loom + screenshots here.

## Benefits

- Turns one Closed-Won into 5-10 pipeline accounts systematically
- Cheap-first waterfall protects free-tier credits
- Strict scoring prevents low-fit spam
- Idempotent dedup (exact domain/email)
- Slack top-3 + Review in HubSpot + qualify/disqualify callbacks deferred to polish

## Security

Never upload credentials. HubSpot Service Key, OpenRouter, Tavily bearer, Hunter, Prospeo, Dropcontact, ZeroBounce, Slack bot tokens live in n8n only. Export workflows/p5-final-closed-won-lookalike.json scrubbed.

## Possible Improvements

- Add Apollo company search when paid tier available
- Add Block Kit + callback buttons for sales to qualify/disqualify
- Add pagination + 100-item chunking for >100 lookalikes
- Add analytics per source deal

## Project Status

Completed — Demo / Portfolio Version — Live workflow 33 nodes, tested synced (score 90) + quarantine paths. Custom props + Block Kit deferred to polish.

## Author

Chiranjeev Sahu — GTM Engineering
Skills Demonstrated: n8n HubSpot Tavily Apollo Hunter ZeroBounce Scoring GTM Expansion
