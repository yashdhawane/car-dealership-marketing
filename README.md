# Agentic Marketing Workflow for a Car Dealership

A design for a multi-agent marketing system that takes a dealership's request in plain language and carries it through research, strategy, content, publishing and measurement — without inventing facts, spending money unattended, or claiming credit for sales it cannot prove.

**Current status: design complete, implementation not started.** This repository contains the full specification (`docs/`), seed datasets (`data/`), and an explorable architecture diagram. There is no application code yet; `docs/10-implementation-options.md` describes what building it would involve.

---

## Contents

- [Why this exists](#why-this-exists)
- [The workflow](#the-workflow)
- [The seven stages](#the-seven-stages)
- [How to use this repository](#how-to-use-this-repository)
- [A campaign end to end](#a-campaign-end-to-end)
- [What data is needed](#what-data-is-needed)
- [Rules the system enforces](#rules-the-system-enforces)
- [Repository layout](#repository-layout)
- [Known limitations](#known-limitations)
- [Next step](#next-step)

---

## Why this exists

A dealership marketing manager says something like:

> "December is quiet. Push 3 Urban SUVs in Pune, around 1 lakh. Wakad branch has too much old stock. Do something on Instagram."

Today that sentence becomes a series of ad-hoc decisions: someone guesses the audience, someone writes a caption, someone boosts a post, and a month later nobody can say which part worked. The common failure modes are predictable:

- A discount is advertised after the offer expired.
- A number in a caption cannot be traced to any source.
- A budget is spent on an untested message.
- A sale is claimed by a campaign that has no join to it.
- A missing data source is quietly replaced by a confident guess.

This design treats those as structural problems, not discipline problems. Every stage has a contract, every fact has an owner, every irreversible action needs an approval bound to exact bytes, and missing data becomes a visible gap instead of a sentence.

---

## The workflow

A request can arrive two ways, and both enter at the same place:

1. **Manual request** from a dealership user (chat, form, email, phone notes).
2. **Scheduled trigger** for automatic campaign planning (recorded as a system requester).

```
 human request / scheduled trigger
              │
              ▼
 ┌────────────────────────────┐
 │ 1  Request → Business Goal │  extract · resolve · ask · approve
 └─────────────┬──────────────┘
               │ approved goal snapshot + hash
               ▼
 ┌────────────────────────────┐
 │ 2  Orchestrator            │  route · delegate · track · gate
 └─────────────┬──────────────┘
               │ task envelope (one agent at a time)
               ▼
 ┌─────────────┐  findings  ┌─────────────┐   G1    ┌─────────────┐
 │ 3 Research  │───────────▶│ 4 Strategy  │────────▶│ 5 Content   │
 └─────────────┘            └─────────────┘         └──────┬──────┘
       ▲                                                   │ G2 (per asset)
       │                                                   ▼
       │                                            ┌─────────────┐
       │                                            │ 6 Distrib.  │── G3 ──▶ platforms
       │                                            │  / Execution│          (Meta, Google,
       │                                            └──────┬──────┘           website, CRM)
       │                                                   │ execution records
       │                                                   ▼
       │                                            ┌─────────────┐
       └─────────────── learnings ──────────────────│ 7 Analytics │
                                                    └─────────────┘
```

The Orchestrator is the only router. Agents never call each other directly, because a side channel is an unaudited decision.

---

## The seven stages

### 1. Campaign request → Business Goal

**Job:** turn a sentence into a structured, checkable goal, and refuse to invent anything the user did not say.

- Captures the request verbatim before anything else happens.
- Extracts objective type, models, stores, target, timeframe and budget.
- Resolves names to IDs using `stores.json` and `inventory.json` ("Pune" → `STORE_001`, `STORE_002`).
- Asks blocking clarification questions when a required field is missing. Nothing moves until they are answered.
- Proposes a measurable success metric with a one-line reason, and shows it to the human.
- Tags every field as `extracted`, `user_confirmed` or `inferred`. An `inferred` value is always shown as an assumption.

**Output:** an approved `Business Goal` with objectives, stores, period, budget, provenance, flags and amendments.
**Gate:** the requester approves the goal.

### 2. Orchestrator

**Job:** decide who runs next, with what, and whether they are allowed to run. It coordinates; it does not write marketing.

- Verifies the goal is approved and its snapshot hash matches.
- Builds a task DAG from the objectives (routing table, not an LLM planner).
- Gives each agent a scoped input bundle, not the whole goal.
- Passes references to artifacts, never its own paraphrases.
- Appends every state change to an append-only event log; the run state is derived from that log.
- Enforces gates, budget ceilings, retries and idempotency.
- Classifies failures: retry reads, never blind-retry a publish, escalate business preconditions.

**Output:** `CampaignRun` (state, plan, tasks, artifacts, gates, open issues) plus `run_events`.

### 3. Research Agent

**Job:** stop the rest of the system from guessing.

Investigates, as scoped by the Orchestrator:

- Historical campaign performance and prior published content.
- Buyer questions and concerns (CRM notes, comments, forums, search).
- Competitor marketing activity that is publicly observable.
- Industry trends and market opportunities.

Every finding carries `source`, `observed_at`, `access` and one of four evidence statuses: `verified`, `single_source`, `unverified`, `contradicted`. Interpretations are a separate field and must cite finding IDs. A requested section with no data produces a `gap`, not a paragraph.

**Output:** a research report with dated observations, interpretations, gaps and `valid_until` (competitor sightings go stale in 14 days, most sections in 30, historical performance in 90).

### 4. Strategy Agent

**Job:** turn the goal and evidence into a plan a human can approve, and show its work.

Decides:

- Which objective leads when the budget cannot serve all of them.
- Audience segment, message, campaign theme and proof points.
- Channels and formats, and what is organic versus paid.
- Budget allocation as numbered lines plus a held reserve.
- KPIs per objective, plus guardrails for paid lines.
- The task list the Orchestrator must run.

It applies the **Dealer Store Positioning** guide (audience, price tier, differentiators with proof, competitive frame, tone, banned claims) and records which rule shaped which decision. An empty `positioning_applied` list fails validation, so a generic ad with the dealership logo pasted on cannot pass as strategy.

Every recommendation cites an input (research finding, goal field, positioning rule, inventory or offer fact) or is explicitly labelled `judgement` with a confidence. Unit targets come with funnel math and a feasibility verdict. If the target is unreachable, it escalates rather than quietly lowering it.

**Output:** `PLAN_...` with audience, messages, channel plan, budget, KPIs, feasibility, assumptions and risks.
**Gate:** `G1_strategy` — approves the plan before any content work.

### 5. Content Agent

**Job:** turn the approved strategy into production-ready creative specs.

Produces hooks, scripts, on-screen text, captions, CTAs, disclaimers, shot lists, durations and aspect ratios for organic and paid formats. Every factual claim maps to a verified inventory or offer field; a discount or EMI always travels with its terms.

It does **not** render video. Each asset declares its production mode:

| Mode | Who produces the file | Used for |
|---|---|---|
| `template_render` | deterministic renderer | offer cards, price posts, carousels |
| `human_shoot` | salesperson or videographer | walkarounds, delivery moments |
| `stock_edit` | editor cuts supplied footage | trend reels, comparisons |

Offer validity is checked at generation and again at publish. A missing vehicle spec sheet blocks feature highlights and comparisons rather than being filled with invented mileage or safety claims.

**Output:** a content pack of per-asset specs with hashes, claim checks and gaps.
**Gate:** `G2_content` — approves each asset individually, by hash.

### 6. Distribution / Execution Agent

**Job:** the only component that touches the outside world, and it assumes the outside world is hostile.

- Re-validates the offer, stock, store status and asset hash at execution time.
- Builds the exact platform payload and attaches an idempotency key.
- Publishes only assets whose hash matches an approved `G2`.
- Creates paid campaigns **paused**; activation is a separate action under `G3`.
- Reads back before retrying after a timeout, so a reel is not posted three times.
- On partial success, records per-item status and reports `partial`.

Every action declares its mode: `api`, `assisted` (agent prepares, human clicks, record says so) or `manual` (brief and checklist only). Nothing claims to have published something a human published.

**Gate:** `G3_publish` — approves the schedule, platform accounts and planned spend.

### 7. Analytics Agent

**Job:** measure with defined metrics and honest attribution.

- Reads the execution registry, not the plan, for what actually went live.
- Reports reach, impressions, watch time, engagement, traffic, leads, qualified leads, test drives, sales, cost per lead and cost per sale when the data exists.
- Uses versioned metric definitions so months stay comparable.
- Labels maturity: `T+7` is an operational check, `T+30` a first sales read, `T+45` close-out.
- Separates attribution into three tiers: **verified** (sale joined by `campaign_id` and `lead_id`), **correlated** (rose during the campaign, no join), **unattributed** (organic, walk-in, referral).
- Says "unknown" when the cause is unknown.

Findings feed back to Research and Strategy as learnings, and per-asset performance feeds the content registry, so the next cycle starts from evidence rather than memory.

---

## How to use this repository

### Today: as a design package

1. **Read the overview** — `docs/00-overview.md` for the five invariants, the approval model and where the LLM does and does not sit.
2. **Read the stage contracts in order** — `docs/01` through `docs/07`, then `docs/08` for the shared handoff, error and gate rules.
3. **Check the data plan** — `docs/09-data-map.md` maps every dataset to the stages that read it, lists what must be authored, and lists the seed-data corrections.
4. **Choose an implementation path** — `docs/10-implementation-options.md`. The recommendation is a deterministic contract harness (option B) before any live platform integration.
5. **Open the diagram** — `.archify/architecture-dealership-marketing-20261120-003227/dealership-marketing-architecture-v2.html` for an explorable view of the same architecture.

### Before any implementation: supply the missing configuration

Four gaps are worth closing first because a stage cannot honestly complete without them:

| Missing | Blocks | What to author |
|---|---|---|
| Store positioning and brand voice | Strategy, Content | One brief per store: audience, price tier, three differentiators with proof, competitive frame, tone, banned claims |
| Business SOPs | Content, Distribution | Prohibited claims, channel spend caps, consent rules, escalation contacts, approval authority |
| Vehicle spec sheet | Feature highlights, comparisons | Mileage, features, safety data keyed by model and variant — or add `specs{}` to inventory |
| Store × platform account registry | Publishing | One row per store × platform: handle, page or ad account ID, token status, mode (`api` / `assisted` / `manual`) |

### Once built: running a campaign

1. Submit a request (manual or scheduled). The system stores it before parsing it.
2. Answer clarification questions. The goal cannot leave `awaiting_clarification` until every required field is resolved.
3. Confirm the proposed success metric and approve the goal.
4. Review the research report: observations, sources, dates, gaps and contradictions.
5. Approve the strategy at `G1`: audience, message, channels, organic/paid split, budget lines, KPIs and assumptions.
6. Review each content asset at `G2`. Editing an approved caption returns only that asset to the gate, not the campaign.
7. Approve the schedule and spend at `G3`. Paid campaigns launch paused and are activated only after this gate.
8. Watch execution per item. Partial success is normal and reported honestly.
9. Read the `T+7`, `T+30` and `T+45` reports. Treat `T+7` as operational, not a sales verdict.
10. Let the learnings feed the next request. Nothing in the loop is typed in by hand.

### Approvals in one line

`G1` strategy → `G2` each asset → `G3` publish and spend. Approvals bind to artifact hashes; a changed byte returns that item only. Silence never approves anything.

---

## A campaign end to end

The docs carry a worked example, `GOAL_002`, across every stage. The short version:

| Step | What happens |
|---|---|
| Request | "Push 3 Urban SUVs in Pune, around 1 lakh. Wakad has old stock. Do something on Instagram." |
| Clarify | Which December? Is 1 lakh total? How many aged cars, and is Instagram paid? |
| Goal | Sales: 3 units. Aged stock: 2 units. Instagram: 40 leads. Budget 100,000 INR, 1–31 Dec 2026. |
| Research | Search converted at roughly half the cost per sale of social in `CAMP_001`; competitor offer values are placeholders, so that comparison is a gap; no platform analytics connection exists. |
| Strategy | Aged stock first. 55k search, 30k social, 15k reserve. Organic tests two message frames; paid scales the winner. Verdict: feasible with a thin evidence base. |
| `G1` | Plan approved at 12:10. |
| Content | Six organic assets per store, paid variants, all facts bound to `OFFER_001` and inventory. |
| `G2` | One asset references expired `OFFER_003`; that branch blocks, the rest proceed. Caption fixed, re-approved. |
| `G3` | Two campaigns created paused, then activated. A Wakad token expires; one post blocks and retries once after refresh. |
| Analytics | 95,000 spent, 253 leads, 28 test drives, 5 sales. Only 3 sales join to the campaign; 2 are reported as correlated. Organic performance is reported as unavailable, not estimated. |

This walkthrough is the acceptance test for the design: gates block, failures stay local, and gaps reach the human.

---

## What data is needed

All currency is INR and all times are Asia/Kolkata.

### A. Reference and runtime data (seeded in `data/`)

| File | What it must contain | Read by |
|---|---|---|
| `stores.json` | `store_id`, `name`, `city`, `address`, `pincode`, `status` | 1, 4, 6, 7 |
| `inventory.json` | `vehicle_id`, `store_id`, `make`, `model`, `variant`, `price`, `stock_quantity`, `days_in_stock`, `status` | 1, 4, 5, 6 |
| `offers.json` | `offer_id`, `name`, `store_ids`, `vehicle_model`, `discount_amount`, `terms`, `valid_from`, `valid_until`, `status` | 1, 4, 5, 6 |
| `budgets.json` | `budget_id`, `store_id`, `campaign_id`, `period`, `total_budget`, `spent_amount`, `currency` | 1, 4, 6 |
| `campaign_history.json` | Campaigns with per-channel spend, impressions, clicks, leads, qualified leads, test drives, sales, revenue | 3, 4, 7 |
| `channel_data.json` | Per-day per-channel rows; paid rows use `impressions`, website rows use `visitors` | 3, 7 |
| `leads.json` | Lead identity, city, store, interested model, budget, source, `campaign_id`, status, consent; plus `allowed_lead_statuses` | 3, 7 |
| `follow_ups.json` | `lead_id`, date, channel, action, outcome, next date, notes (buyer concerns live here) | 3, 7 |
| `sales.json` | `sale_id`, `lead_id`, `campaign_id`, `store_id`, `vehicle_id`, model, amount, date, status | 7 |
| `competitor_data.json` | Competitor name, city, channel, theme, model, message, offer, price, ad URL, `observed_date` | 3 |
| `business_goals.json` | Seeded goals that migrate into the runtime goal store | 1 |

`sales.json` carrying both `campaign_id` and `lead_id` is the single most important field pair in the dataset: it is the only reason Analytics can say "verified" instead of "correlated". Do not remove either.

### B. Configuration you must author (empty today)

| Item | Why it is needed |
|---|---|
| `_context/` positioning briefs | Audience, price tier, differentiators with proof, tone, banned claims, per store. Strategy must prove it applied them; Content reads voice from here. |
| `_sop/` business rules | Prohibited claims, channel spend caps, consent policy, escalation contacts, approval authority. |
| Vehicle spec sheet | Mileage, features, safety. Without it, feature highlights and comparisons cannot be written honestly. |
| Store × platform account registry | Which handle/page/ad account belongs to which store, token status, supported mode. |
| Capability matrix | `api`, `assisted` or `manual` per store × platform. This is the honest API table. |
| Metric definitions (~10, versioned) | Formula, source, filters, timezone and owner for each KPI. Without versions, month-to-month comparison is fiction. |
| Consent register | Required before any asset with a person on camera can pass `G2`. |

### C. Data the system creates

| Store | Written by | Purpose |
|---|---|---|
| `campaign_requests` | Stage 1 | Raw truth, never edited; clarification attaches here |
| `business_goals` + `amendments` | Stage 1 | What was agreed, and its history |
| `campaign_runs` + `run_events` | Orchestrator | Audit trail; run state is derived from events |
| `artifact_registry` | Orchestrator | Exact approved bytes and hashes |
| `content_registry` | Content → Distribution | Per-asset identity and performance |
| `execution_registry` | Distribution | What actually went live, platform IDs, spend committed |
| `performance_reports` | Analytics | Outcomes with maturity and attribution tier |
| `metric_definitions` | Analytics | Comparability across periods |
| `learnings` | Analytics | The only source of "what we learned last time" |

Everything here is derived from the datasets above, except the capability matrix and metric definitions, which are configuration.

### D. Missing data and what it blocks

| Missing | Blocks | Current consequence |
|---|---|---|
| Positioning and brand voice | Strategy `S8`, Content tone | Every positioning choice would be marked `assumed` |
| SOPs and prohibited-claim list | Content claim checks, spend limits | No enforceable claim or spend policy |
| Vehicle spec sheet | Feature highlights, comparisons | Copy can only sell price and offer |
| Platform account registry | Publishing | Every row would fall back to `manual` |
| Metric definitions | Comparable reporting | Numbers cannot be compared month to month |
| Platform insights connection | Organic evaluation | Reach and watch time are unmeasurable |
| Audience size data | Budget confidence | Audience sizing stays `unknown` |
| Inventory ageing snapshots | Aged-stock reasoning | `days_in_stock` is only a snapshot |
| Prior-year baseline | Seasonal comparison | Seasonality is invisible |

The first four — positioning, SOPs, spec sheet and the account registry — are worth fixing before writing code. The rest can be shipped as declared gaps.

### E. Seed data that needs correction

The seed files contain contradictions that the system will deliberately flag rather than silently fix. The full list is in `docs/09-data-map.md`. The main ones:

| File | Issue |
|---|---|
| `business_goals.json` | `target_revenue` 500,000 disagrees with 3 units at ~1.45M each |
| `offers.json` | `OFFER_003` is `expired` but `valid_until` is in the future |
| `leads.json` | `LEAD_001` is `test_drive_booked` while `SALE_001` exists for that lead |
| `campaign_history.json` | `CAMP_001` revenue 9,000,000 disagrees with 1,450,000 in `channel_data` |
| `budgets.json` | December spend is recorded before December starts |
| `competitor_data.json` | `"will be number"` placeholders where offer and price should be |

These are intentional: they exercise the contradiction, flag and data-quality paths.

---

## Rules the system enforces

1. **No fact without a source.** An untraceable claim is a gap or a labelled assumption.
2. **No number without a definition.** Metrics are versioned artefacts.
3. **Nothing irreversible without a matching hash.** Approval binds to bytes, not to a conversation.
4. **One writer per fact.** Every value has exactly one owning component.
5. **Missing data produces a gap, never a sentence.** The system says "we did not check" out loud.
6. **Paid campaigns start paused.** Money defaults to unspent.
7. **Publishing is never blindly retried.** Read back first; deletion needs approval.
8. **"Caused" requires a join.** Otherwise the wording becomes "cannot attribute".
9. **Contradictions are shown, not averaged.**
10. **Personal data stays in the CRM.** Lead names and contact details never enter creative or research output.

What the design deliberately refuses to do:

| Not done | Because |
|---|---|
| Invent business facts | A goal is the user's intent, not the model's guess |
| Assume an API exists | Each store × platform declares `api`, `assisted` or `manual` |
| Render video | Content writes specs; a camera still needs a person |
| Auto-publish or auto-spend | Gates bind to hashes; spend defaults to zero |
| Attribute a sale without a join | Attribution is verified, correlated or unattributed |
| Lower a target to make a plan pass | Infeasibility escalates to the user |

---

## Repository layout

```
cardealership/
├─ README.md                      ← this file
├─ docs/                          design specification, one file per stage
│  ├─ 00-overview.md              flow, invariants, LLM boundaries
│  ├─ 01-request-to-business-goal.md
│  ├─ 02-orchestrator.md
│  ├─ 03-research-agent.md
│  ├─ 04-strategy-agent.md
│  ├─ 05-content-agent.md
│  ├─ 06-distribution-agent.md
│  ├─ 07-analytics-agent.md
│  ├─ 08-handoffs-state-errors.md
│  ├─ 09-data-map.md
│  └─ 10-implementation-options.md
├─ data/                          eleven seed datasets (reference + runtime)
├─ _context/                      store positioning and brand voice — empty, to author
├─ _sop/                          business rules and claim policy — empty, to author
└─ .archify/                      generated interactive architecture diagram
```

---

## Known limitations

- **Platform access will be partial.** Tokens expire, ad accounts get restricted, insights lag. Much of a real deployment runs in `assisted` or `manual` mode.
- **Attribution will be incomplete.** Most dealership sales do not carry a campaign ID. The three-tier model is built for that reality, not around it.
- **Samples will be small.** One campaign and eight sales is a hypothesis, not a trend. Confidence caps are enforced for a reason.
- **Humans will approve late.** The system stalls visibly rather than publishing unattended.
- **No code exists yet.** The contracts are specified in prose with example JSON payloads; they are not executable.
- **No UI is planned for the first build.** The interesting behaviour is in the gates, retries and attribution, and a dashboard would hide it.

---

## Next step

The recommended path is **option B: a contract harness**.

- Zod schemas for every artifact.
- All seven stages with deterministic logic and stubbed reasoning.
- Fake platform adapters that can publish, reject, time out and duplicate.
- A dry run of `GOAL_002` that prints the audit trail.

It is credible when it demonstrates five things:

1. An asset whose hash no longer matches `G2` is refused by Distribution.
2. A publish timeout triggers read-back, not a duplicate post.
3. A creative referencing `OFFER_003` blocks that branch while organic keeps running.
4. A sale without `campaign_id` is reported as correlated, and the words "produced" and "caused" never appear next to it.
5. A goal with a missing period never leaves `awaiting_clarification`.

If you want live LLM judgement, option D adds real model calls at exactly three points: request extraction, research interpretation and content drafting. Everything else stays deterministic.

Read `docs/10-implementation-options.md` before choosing.
