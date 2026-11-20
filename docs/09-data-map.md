# Data Map — What Each Dataset Feeds, and What Is Genuinely Missing

You said the eleven files are reference and runtime data. Here is exactly what each one feeds, what builds up over time, and what is missing that a stage cannot work without. Nothing is proposed that the workflow does not use.

---

## 1. The eleven files, mapped

| File | Stages | What it is read for | Mode |
|---|---|---|---|
| `stores.json` | 1, 4, 6, 7 | Resolve city → store; store identity; account ownership; per-store reporting | read |
| `inventory.json` | 1, 4, 5, 6 | Validate model exists; stock and `days_in_stock` for aged-stock reasoning; verified facts for copy; re-check at publish | read |
| `offers.json` | 1, 4, 5, 6 | Covering offer check; message proof; discount and terms in copy; expiry re-check at execution | read |
| `budgets.json` | 1, 4, 6 | Budget ceiling flag; spend envelope and reserve; activation guard | read |
| `campaign_history.json` | 3, 4, 7 | Internal findings; expected CPA; baseline for comparison | read |
| `channel_data.json` | 3, 7 | Channel-level findings; trend over time | read |
| `leads.json` | 3, 7 | Buyer signals; funnel; `allowed_lead_statuses` as an enum for validation | read |
| `follow_ups.json` | 3, 7 | Buyer concerns from notes; lead status history | read |
| `sales.json` | 7 | **Attribution** — `campaign_id` + `lead_id` is what makes "verified" possible | read |
| `competitor_data.json` | 3 | Competitor activity observations | read |
| `business_goals.json` | 1 | Seed of the goal contract; `GOAL_001` migrates into `objectives[]` | read, then extended |

---

## 2. What builds up, and why each one exists

| Store | Written by stage | Why it cannot be avoided |
|---|---|---|
| `campaign_requests` | 1 | A request must be stored before it becomes a goal; otherwise clarification has nowhere to attach |
| `business_goals` (+ `amendments`) | 1 | The approved thing, with its history |
| `campaign_runs` + `run_events` | 2 | "Track status" is impossible without it; events are the audit trail |
| `artifact_registry` | 2 | Approvals bind to hashes, so the exact bytes must be kept |
| `content_registry` | 5 → 6 | Per-asset performance needs a per-asset identity |
| `execution_registry` | 6 | Platform IDs and spend actually committed — not planned |
| `performance_reports` | 7 | Outcomes, with maturity |
| `metric_definitions` | 7 (versioned) | Month-to-month comparability |
| `learnings` | 7 | The only source of "what we learned last time" |
| `capability_matrix` | human | Which platform × store is `api`, `assisted` or `manual` |

All of these are derived from what already exists, except the capability matrix and metric definitions, which are configuration.

---

## 3. Genuinely missing

**Blocking** means a stage cannot honestly complete without it. **Degrading** means it runs, with a stated weakness.

| # | Missing | Blocks | Consequence today | Fix |
|---|---|---|---|---|
| 1 | Store positioning + brand voice (`_context/` is empty) | Stage 4 `S8`, stage 5 voice | Strategy cannot prove positioning was applied; every tone choice is `assumed` | Author one brief per store: audience, price tier, 3 differentiators with proof, tone, banned claims |
| 2 | Business SOPs (`_sop/` is empty) | Stage 5 claim checks, stage 6 spend limits | No prohibited-claim list, no channel spend caps, no consent policy | Author: banned claims, spend rules, consent rules, escalation contacts |
| 3 | Vehicle spec sheet | Content types `feature_highlight`, `comparison` | Copy cannot say mileage, features or safety — so content can only sell price and offer | Add `specs{}` to `inventory`, or a `vehicle_specs` file keyed by model+variant |
| 4 | Store × platform account registry | Stage 6 `api` and `assisted` modes | Nothing can be published or set up; every row would be `manual` | One row per store × platform: handle, page/ad account id, token status |
| 5 | Metric definitions registry | Stage 7 `A1`, comparability | Numbers cannot be compared across months | Seed ~10 definitions; version them |
| 6 | Platform insights (reach, watch time, engagement) | Organic evaluation | Organic content is unmeasurable — already visible as a gap in the tabletop | Not "missing data" — a connection that does not exist yet |
| 7 | Audience data (size, interest overlap, geo demand) | Budget confidence | Strategy sets criteria but cannot size an audience, so the split is judged, not computed | Export from ad platforms once connected; until then `size_basis: unknown` |
| 8 | Inventory ageing history (snapshots) | Aged-stock reasoning | `days_in_stock` is a snapshot; cannot tell "always slow" from "recently stuck" | Nightly snapshot of `days_in_stock` and `stock_quantity` |
| 9 | Prior-year same-period baseline | Seasonal comparison | Baseline is one campaign, so seasonality is invisible | Acceptable to start; add when a year of data exists |
| 10 | Consent records for people on camera | Delivery moments, salesperson videos | Person-on-camera assets block at `C7` | A simple consent register, or an SOP rule that the human confirms at `G2` |

Items 1–4 are worth fixing before any code. 5 can be a seed file. 6–10 are known weaknesses to state, not blockers.

---

## 4. What I deliberately did **not** add

| Not added | Why not |
|---|---|
| A separate "campaign plan" table | The goal plus the strategy artefact already carry it |
| A competitor tracking database | Research records sightings with dates; a database would imply coverage we do not have |
| An asset store or render pipeline | Content produces specs; rendering is a separate, optional concern |
| An approvals table | Approvals are events in `run_events`, bound to hashes — one source of truth |
| A "budget request" table | `budgets.json` plus the goal budget covers it |
| An audience segment store | Derived from lead records at planning time; storing it invites staleness |
| A free-text insights table | `learnings` covers it, and it is derived rather than typed |

---

## 5. Seed fixes

You said the data can be corrected. These are the specific edits:

| File | Change | Why |
|---|---|---|
| `business_goals.json` | `target_revenue` 500,000 → 4,350,000, or drop it | 3 × Urban SUV ≈ 4.35M; the two targets disagree |
| `business_goals.json` | Reshape `GOAL_001` into `objectives[]` | Matches the nested contract; one objective to start |
| `offers.json` | `OFFER_003` status `expired` → `active`, or `valid_until` → 2026-10-31 | Status contradicts the date |
| `leads.json` | `LEAD_001` status → `sold` | `SALE_001` exists for that lead |
| `campaign_history.json` | `CAMP_001` revenue 9,000,000 → 1,450,000, or regenerate `channel_data` to reconcile | Channel rows sum to 1.45M |
| `budgets.json` | `spent_amount` before the period starts → either `committed_amount` or move the period | December has not begun; spend cannot precede it |
| `competitor_data.json` | Replace `"will be number"` placeholders | Present records look usable but carry no data; this is what forced the contradiction in the tabletop |
| `channel_data.json` | Document per-channel fields (`visitors` vs `impressions`) | Mixed shape is correct but must be declared |

One fix is a judgement call, not a correction: the per-sale revenue of 1.5M against a 1.45M list price. Probably add-ons and insurance. Flag it, don't "fix" it — that is a business fact nobody supplied.

---

## 6. Shape notes worth keeping

- `leads.json` carries `allowed_lead_statuses`. That is a schema, not decoration — validation should use it rather than hard-coding statuses.
- `competitor_data.json` currently nests one channel object per competitor. If more channels appear, it becomes an array. The research contract expects a list of dated sightings, so this file needs reshaping before it can feed stage 3 properly.
- `sales.json` having both `campaign_id` and `lead_id` is the single most valuable thing in the dataset. It is the only reason stage 7 can say "verified" instead of "correlated". Do not remove either field.
