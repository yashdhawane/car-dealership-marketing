# Stage 7 — Analytics Agent

Analytics has one hard rule: **a number is only as good as its definition, and a cause is only as good as its join.** Everything else follows from that.

---

## 1. Input

| Input | Why |
|---|---|
| Execution registry | What actually went live: platform IDs, timestamps, spend (not the plan) |
| Platform insights | Reach, impressions, views, watch time, engagement, clicks, spend |
| Website data | Sessions, source, landing page, form submits |
| CRM data | Leads, status history (`leads.json`, `follow_ups.json`), test drives |
| Sales data | `sales.json` with `campaign_id` and `lead_id` — this is what makes attribution possible |
| Goal KPIs and guardrails | What "good" means for this run |
| Metric definitions registry | So this month's number is comparable to last month's |
| Prior campaign results | Baseline, not zero |

---

## 2. Metric definitions are versioned artefacts

Every metric has one stored definition: formula, source, filters, timezone, and owner.

| Metric | Definition |
|---|---|
| `lead` | A record in `leads.json` created in the period with `source` matching the channel |
| `qualified_lead` | A lead whose status reached `qualified` or beyond |
| `test_drive_booked` | A lead with status `test_drive_booked` or beyond |
| `completed_sale` | A `sales.json` row with `status: completed` |
| `cost_per_lead` | channel spend ÷ leads attributed to that channel in period |
| `cost_per_sale` | channel spend ÷ completed sales joined to that channel |

If the definition changes, it changes with a version. Otherwise January's "lead" and December's "lead" are different animals and every trend is fiction.

---

## 3. Attribution — three tiers, never blurred

| Tier | Evidence required | Permitted language |
|---|---|---|
| **Verified** | `sales.campaign_id` = this campaign, and the lead is joined by `lead_id` | "produced 3 sales" |
| **Correlated** | Campaign ran, sales rose, no join exists | "sales rose during the campaign; we cannot attribute them to it" |
| **Unattributed** | Organic, walk-in, referral, no campaign id | reported as a total, never credited to a channel |

This is why `sales.json` having both `campaign_id` and `lead_id` matters: it is the difference between a report and a guess. When `campaign_id` is null, that sale is real, but the campaign's name does not go on it.

---

## 4. Maturity

Sales lag the campaign. Every report carries a maturity date and a label:

- `T+7` — operational check: reach, CPL, creative response. **Not** a sales verdict.
- `T+30` — first sales read, still incomplete.
- `T+45` — close-out, when the sales cycle has mostly landed.

Comparing a T+7 report to a T+45 report and calling it a decline is the most common analytical error in dealership marketing. The schema makes the maturity impossible to ignore.

---

## 5. Diagnostic discipline

| Claim type | Requirement |
|---|---|
| "Channel A beat channel B" | Rates from the same period, same definition, sample size stated |
| "Creative X won" | Per-asset performance, split by variant, not by idea |
| "This caused the lift" | A join, or the sentence becomes "we cannot attribute" |
| "The market changed" | A dated external source; otherwise it is a hypothesis |

Every diagnosis carries `what`, `evidence`, `confidence`, and — when the cause is unknown — **"unknown"**. An explanation with no evidence is worse than a gap, because someone will act on it.

---

## 6. Output schema

```json
{
  "report_id": "PERF_004",
  "run_id": "RUN_007",
  "period": { "start": "2026-12-01", "end": "2026-12-31" },
  "maturity": { "label": "T+30", "as_of": "2027-01-30", "complete": false, "expected_lag_days": 45 },
  "metrics": [
    { "key": "spend", "value": 95000, "unit": "INR", "definition_ref": "def:spend@v1", "source": "execution_registry", "evidence_status": "verified" },
    { "key": "leads", "value": 253, "unit": "count", "definition_ref": "def:lead@v1", "source": "leads.json", "evidence_status": "verified" },
    { "key": "qualified_leads", "value": 78, "unit": "count", "definition_ref": "def:qualified@v1", "source": "leads.json", "evidence_status": "verified" },
    { "key": "test_drives_booked", "value": 28, "unit": "count", "definition_ref": "def:td@v1", "source": "leads.json", "evidence_status": "verified" },
    { "key": "completed_sales", "value": 5, "unit": "count", "definition_ref": "def:sale@v1", "source": "sales.json", "evidence_status": "verified" },
    { "key": "cost_per_lead", "value": 375, "unit": "INR", "definition_ref": "def:cpl@v1", "evidence_status": "verified" },
    { "key": "cost_per_sale", "value": 31667, "unit": "INR", "definition_ref": "def:cpa@v1", "note": "verified sales only (3); all-sales CPA is 19,000", "evidence_status": "verified" },
    { "key": "organic_watch_time", "value": null, "unit": "seconds", "evidence_status": "unavailable",
      "note": "no platform connection; GAP" }
  ],
  "by_channel": [
    { "channel": "google_ads", "spend": 65000, "leads": 168, "qualified": 57, "test_drives": 19, "sales_verified": 3, "cpl": 387, "qualified_rate": 0.339 },
    { "channel": "meta_ads", "spend": 30000, "leads": 74, "qualified": 18, "test_drives": 7, "sales_verified": 0, "cpl": 405, "qualified_rate": 0.243 },
    { "channel": "instagram_organic", "spend": 0, "leads": 11, "qualified": 3, "test_drives": 2, "sales_verified": 0, "cpl": null }
  ],
  "funnel": {
    "leads": 253, "qualified": 78, "test_drives": 28, "sales": 5,
    "lead_to_qualified": 0.308, "qualified_to_td": 0.359, "td_to_sale": 0.179,
    "baseline": { "source": "CAMP_001", "lead_to_qualified": 0.342, "qualified_to_td": 0.40, "td_to_sale": 0.308 }
  },
  "attribution": {
    "verified_sales": 3, "correlated_sales": 2, "unattributed_sales": 0,
    "revenue_verified": 4350000, "revenue_total": 7250000,
    "note": "2 sales came from walk-in and website leads with no campaign_id"
  },
  "kpi_results": [
    { "objective_id": "OBJ_1", "key": "completed_sales", "target": 3, "actual": 3, "result": "met" },
    { "objective_id": "OBJ_2", "key": "aged_stock_units", "target": 2, "actual": 2, "result": "met" },
    { "objective_id": "OBJ_1", "key": "cost_per_sale", "guardrail_max": 33000, "actual": 31667, "result": "within_guardrail" },
    { "objective_id": "OBJ_3", "key": "leads", "target": 40, "actual": 11, "result": "missed" }
  ],
  "diagnosis": [
    { "what": "Search produced qualified demand at roughly 1.4x the rate of social",
      "evidence": ["by_channel.google_ads.qualified_rate=0.339", "by_channel.meta_ads.qualified_rate=0.243", "CAMP_001 same direction"],
      "confidence": "medium",
      "caveat": "Two campaigns now agree, which is stronger than one; still under 100 sales total" },
    { "what": "The family/space frame outperformed the EMI frame on social",
      "evidence": ["AST_12.qualified_rate=0.31", "AST_11.qualified_rate=0.21", "n=74 leads"],
      "confidence": "low",
      "caveat": "Small sample, one store, one month. Test on search before scaling" },
    { "what": "Organic Instagram contribution cannot be judged",
      "evidence": [], "confidence": "unknown",
      "caveat": "no watch time or reach data; 11 leads is not enough to evaluate creative" }
  ],
  "experiments_for_next_cycle": [
    { "hypothesis": "The family/space frame beats EMI on search too",
      "test": "Split RSAs 50/50 for two weeks", "metric": "qualified_rate",
      "min_sample": "300 clicks per arm", "cost": "no incremental spend" },
    { "hypothesis": "Retargeting site visitors converts cheaper than cold social",
      "test": "10,000 on retargeting once the audience exceeds 5,000", "metric": "cost_per_sale",
      "min_sample": "audience 5,000", "cost": 10000 }
  ],
  "learnings_for_next_cycle": [
    { "text": "In Pune, search intent converts at ~1.4x the qualified rate of feed placement", "confidence": "medium", "basis": ["PERF_004", "RES_003:F1"] },
    { "text": "Aged-stock urgency did not harm qualified rate", "confidence": "low", "basis": ["PERF_004"] },
    { "text": "Two of five December sales cannot be attributed to any campaign", "confidence": "verified", "basis": ["sales.json"] }
  ]
}
```

---

## 7. Validation

| Code | Rule | On failure |
|---|---|---|
| A1 | Every metric has a definition ref, source and timestamp | Report rejected |
| A2 | A causal claim requires a join | Wording downgraded to "cannot attribute" |
| A3 | Denominator present; no CPL/CPC on zero spend | Metric set to `null` |
| A4 | Period and timezone aligned across sources | Report rejected |
| A5 | Sample size stated when < 30 | Confidence capped at low |
| A6 | Maturity label present | Report rejected |
| A7 | Baseline stated (prior campaign, prior period, or "none available") | Field required |
| A8 | Channel totals reconcile with execution registry spend | Report rejected |

---

## 8. Failure handling

| Failure | Handling |
|---|---|
| Platform API missing | Metric `unavailable`, gap listed. Never estimated |
| CRM sync delayed | Re-run later; do not interpolate a sales count |
| Two sources disagree on spend | Show both, use the execution registry as authority, flag |
| Attribution join fails for some sales | Report them as correlated, not verified |
| Campaign too small to conclude | Say so, cap confidence, recommend a test |
| Report requested at T+3 | Produce an operational check labelled as such, not a sales verdict |

---

## 9. The feedback loop

Three destinations, one direction — findings never overwrite facts:

1. **Research** receives `learnings_for_next_cycle` as *internal findings* with a source and a date. Next time, they enter the report as `verified` internal evidence rather than model opinion.
2. **Strategy** receives the same learnings plus per-channel rates, so budget allocation starts from evidence instead of the previous plan.
3. **The content registry** receives per-asset performance, so a creative that lost is not rebuilt next month by an agent that forgot.

This is the only new store in the whole design, and it exists because the spec requires learning across cycles. Everything in it is derived from the datasets above; nothing is typed in by hand.

---

## 10. Tabletop — GOAL_002 close-out

At T+30 the report reads: 95,000 spent, 253 leads, 78 qualified, 28 test drives, 5 sales, 4.35M verified revenue.

What it says carefully:

- **Search beat social on quality, again** — 33.9% versus 24.3% qualified. Two campaigns now point the same way, so confidence moves from low to medium. Not to "proven": that needs more sales than exist.
- **The family frame beat EMI on social** — 31% versus 21% qualified — but on 74 leads, so it is a reason to test, not a reason to reallocate.
- **Two of five sales are not attributable.** They came from walk-in and website leads with no `campaign_id`. The campaign gets credit for three.
- **Cost per sale is reported two ways** — 31,667 on verified sales, 19,000 blended — because collapsing them into one number hides exactly the thing the human needs to know.
- **Organic is honestly blank.** No watch time, no reach. Eleven leads is not an evaluation.
- **The aged-stock objective was met** (2 of 2), and the sales objective was met (3 of 3), while the Instagram leads objective missed badly (11 of 40) — which is the correct outcome for an objective set without a plan behind it, and a lesson for stage 1.

---

## 11. Decisions I made (flag if you disagree)

- Metric definitions are versioned; changing one is a visible event.
- Attribution is three-tier, and the report always shows both verified and blended cost per sale.
- Maturity labelling is mandatory, because T+7 sales verdicts are always wrong.
- "Unknown" is a valid diagnosis. Fabricated causality is not.
- The learnings store is the only new dataset, and it is derived, not authored.
