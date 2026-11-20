# Stage 4 — Strategy Agent

Strategy is the first component allowed to have an opinion. Its job is to turn a goal and a pile of evidence into a plan a human can approve — and to show its work, because a plan without lineage is just taste.

---

## 1. Input

| Input | From | Used for |
|---|---|---|
| Approved goal v1 | stage 1 | Objectives, targets, stores, period, budget |
| Research report | stage 3 | Market and historical evidence, with gaps |
| Prior published content + performance | content registry, analytics | What worked, what to repeat |
| Historical campaign results | research `historical_performance` | Funnel rates, expected cost |
| Store positioning + brand rules | `_context/` via the positioning guide | Audience, message frame, tone limits |
| Inventory + offers | `inventory.json`, `offers.json` | What can actually be sold, at what price |
| Budget + channel constraints | `budgets.json`, `_sop/` | What is affordable and permitted |
| Accumulated learnings | `learnings` from stage 7 | Prior experiments and their outcomes |

---

## 2. The Dealer Store Positioning guide

This is a reusable skill, not free text. Per store it supplies: who we serve, price tier, three differentiators with proof, competitive frame, tone, and a list of claims we never make. Strategy must apply it and prove it did.

```
positioning_applied: [
  { rule: "STORE_002 framed as value/family store, not premium",
    shaped: "audience.segment SEG_1 and message MSG_2",
    where: "context/positioning.json#STORE_002" }
]
```

An empty `positioning_applied` list fails validation. That single rule prevents the most common failure in this kind of system: a generic car ad with the dealership's logo pasted on.

**Current state:** `_context/` is empty. So the guide's *contract* is defined here and the content must be authored before stage 4 can be trusted. Until then, Strategy runs with `positioning_source: "missing"` and every positioning claim is visibly `assumed` — it does not get to silently invent a brand voice.

---

## 3. What it decides

1. **Objective priority** — which objective leads when budget cannot serve all of them equally.
2. **Audience** — a segment defined by targetable criteria, with an honest statement of what we do *not* know about its size.
3. **Message and theme** — pillars, each tied to a proof point we actually hold.
4. **Channels and formats** — where each message belongs.
5. **Organic vs paid split** — with a principle, not a vibe.
6. **Budget allocation** — arithmetic across objectives and channels, plus a held reserve.
7. **KPIs** — per objective, plus guardrails.
8. **Task list** — what the Orchestrator must have built.

### The four rules

**Every recommendation carries lineage.** Each item cites research findings, goal fields, positioning rules, or inventory/offer facts. Something with no lineage is allowed but must be labelled `judgement` with low confidence. That label is what separates a designed plan from a confident guess.

**Organic tests, paid scales.** New or unproven messages start organic or in a small paid test; only a message with evidence behind it receives scale budget. Paying to find out whether a message works is how dealership budgets disappear in a week.

**Budget is arithmetic, and part of it is held back.** Every line is a number with an expected outcome; total ≤ envelope; a reserve is held and released against a checkpoint condition, not spent because it exists.

**Show the funnel before promising the target.** If the goal is unit-based, Strategy computes expected sales from historical rates and states the verdict. If the target is not reachable, it says so and escalates. It never quietly reports a plan that cannot hit the number.

---

## 4. Output schema

```json
{
  "plan_id": "PLAN_002",
  "run_id": "RUN_007",
  "goal_ref": { "goal_id": "GOAL_002", "version": 1 },
  "inputs_used": {
    "research": "RES_003",
    "positioning": "missing",
    "inventory_snapshot": "2026-11-20T10:55:00+05:30",
    "learnings": []
  },
  "objective_priority": [
    { "objective_id": "OBJ_2", "rank": 1,
      "reason": "Aged stock carries real holding cost; sales and leads objectives are served by the same demand",
      "evidence": ["F1", "inventory:VEH_003.days_in_stock=110"] }
  ],
  "audience": [
    {
      "segment_id": "SEG_1",
      "description": "Pune buyers shopping a compact-to-mid SUV, budget band 1.3M-1.9M",
      "criteria": { "geo": "10 km around STORE_001 and STORE_002", "interests": ["SUV", "car finance"],
                    "budget_band": { "min": 1300000, "max": 1900000 } },
      "size_estimate": null,
      "size_basis": "unknown — no audience data available",
      "evidence": ["leads.json (4 records)"]
    }
  ],
  "messages": [
    { "message_id": "MSG_1", "pillar": "Certainty of price",
      "claim": "Urban SUV at a stated on-road benefit, EMI explained line by line",
      "proof_refs": ["offer:OFFER_001"],
      "objective_ids": ["OBJ_1", "OBJ_2"] }
  ],
  "channel_plan": [
    {
      "line_id": "CHL_1", "channel": "google_ads", "organic_or_paid": "paid",
      "role": "capture existing intent", "objective_ids": ["OBJ_1", "OBJ_2"],
      "formats": ["search_rsa", "vehicle_listing"],
      "planned_spend": 55000,
      "expected": { "leads": 140, "cpl": 393, "qualified_rate": 0.35, "cost_per_sale": 8000 },
      "basis": ["F1"]
    },
    { "line_id": "CHL_2", "channel": "meta_ads", "organic_or_paid": "paid",
      "role": "create demand and retarget site visitors", "objective_ids": ["OBJ_1"],
      "formats": ["reel_ad", "carousel"], "planned_spend": 30000,
      "expected": { "leads": 60, "cpl": 500, "qualified_rate": 0.28, "cost_per_sale": 12000 },
      "basis": ["F2"] },
    { "line_id": "CHL_3", "channel": "instagram_organic", "organic_or_paid": "organic",
      "role": "test message frames at zero media cost", "objective_ids": ["OBJ_1", "OBJ_3"],
      "formats": ["reel", "carousel", "story"], "planned_spend": 0,
      "expected": { "leads": 15, "cpl": null }, "basis": ["F6"] }
  ],
  "budget_allocation": {
    "envelope": 100000,
    "lines": [
      { "line_id": "CHL_1", "amount": 55000 },
      { "line_id": "CHL_2", "amount": 30000 },
      { "line_id": "CHL_3", "amount": 0 }
    ],
    "reserve": { "amount": 15000, "release_condition": "after 2026-12-08 checkpoint if blended CPL <= 500" }
  },
  "kpis": [
    { "objective_id": "OBJ_1", "key": "completed_sales", "target": 3,
      "guardrail": { "key": "cost_per_sale", "max": 33000 } },
    { "objective_id": "OBJ_3", "key": "leads", "target": 40,
      "guardrail": { "key": "cost_per_lead", "max": 600 } }
  ],
  "feasibility": {
    "target_total_units": 5,
    "expected_units": 13,
    "worst_case_units": 10,
    "basis": "blended CPA 7,500 from CAMP_001; worst case uses meta-only CPA 10,000",
    "verdict": "feasible_with_buffer",
    "warning": "single prior campaign, n=8 sales"
  },
  "constraints_checked": [
    { "rule": "_sop.max_paid_share_per_channel", "limit": "60%", "actual": "55%", "result": "pass" }
  ],
  "positioning_applied": [],
  "assumptions": [
    { "text": "December demand is not materially different from October", "confidence": "low",
      "impact": "if demand is lower, paid lead volume falls and CPA rises" }
  ],
  "risks": [
    { "risk": "OFFER_001 expires 2026-12-31, same day as the campaign", "mitigation": "content must carry terms and a validity date" }
  ],
  "tasks_requested": [
    { "agent": "content", "purpose": "6 organic assets, 3 paid variants", "objective_ids": ["OBJ_1", "OBJ_2", "OBJ_3"] },
    { "agent": "distribution", "purpose": "publish and set up two paused campaigns", "objective_ids": ["OBJ_1"] }
  ]
}
```

---

## 5. Validation

| Code | Rule | On failure |
|---|---|---|
| S1 | Every plan item cites input, or is labelled `judgement` with a confidence | Item rejected |
| S2 | Allocated + reserve ≤ envelope | Plan rejected |
| S3 | Every objective has ≥1 channel line | Plan rejected |
| S4 | Every KPI exists in the metric registry and its source is reachable | Plan rejected |
| S5 | Every channel and spend rule is checked against `_sop` | Plan rejected |
| S6 | Every referenced offer is active and covers the store and period | Plan rejected |
| S7 | Unit targets come with funnel math and a verdict | Plan rejected |
| S8 | `positioning_applied` non-empty when `_context` exists; every item traceable | Plan rejected |
| S9 | Audience criteria are targetable, and unknown size is stated as unknown | Plan rejected |
| S10 | Guardrail exists for every paid objective | Warning |

---

## 6. Failure handling

| Failure | Handling |
|---|---|
| Research is `partial` with gaps | Plan proceeds, and each affected decision drops a confidence level and states the gap |
| Target not reachable | `TARGET_AT_RISK`, escalate to the user: raise budget, extend period, or amend target. Strategy does not bend the target |
| Constraint conflict (sop max spend < needed minimum) | Escalate with the two options, do not pick silently |
| Positioning missing | Run with `positioning_source: missing`; every positioning-dependent choice marked `assumed` |
| Inventory insufficient for the target | Flag `STOCK_CONSTRAINT` with the shortfall in units |
| Two objectives compete for the same budget | Priority order plus the trade-off written down |

---

## 7. Handoff

- **Content** receives the approved plan reference, with `objective_ids`, `messages`, `channel_plan` and the fact constraints attached per task.
- **Distribution** receives the channel and spend plan, but may not act until `G3`.
- **Orchestrator** receives `tasks_requested`, which becomes the task list.

Content does not read the research report directly — it reads the strategy that was approved. One hop, so there is one thing to audit.

---

## 8. Tabletop — GOAL_002

**Priority:** aged stock first. Reason: `VEH_003` sits 110 days at `STORE_002`; holding cost is real and the same demand serves all three objectives.

**Budget:** 55k search, 30k social, 15k held. Reserve releases only if blended CPL ≤ 500 at the 8 December checkpoint.

**Organic vs paid:** organic tests whether "EMI explained line by line" or "space for the family" moves people. Paid follows whichever wins at the checkpoint. No scale spend on an untested message.

**Feasibility:** 5 units to hit both sales objectives. At the prior campaign's blended ₹7,500 per sale, 100k buys ~13; worst case, social-only rates, ~10. Verdict `feasible_with_buffer`, with the caveat that this is one campaign and eight sales.

**What it refuses:** to state audience size. Four leads in `leads.json` is not an audience estimate. It says `size_basis: unknown`.

**What it escalates:** nothing, but it warns that `OFFER_001` expires on the campaign's last day, so every asset must carry the validity date.

---

## 9. Decisions I made (flag if you disagree)

- Strategy must prove positioning was applied; an empty list fails validation.
- Organic tests messages, paid scales proven ones; no scale spend without evidence.
- Budget is a set of numbered lines plus a held reserve, never a percentage split.
- Target infeasibility escalates to the user. The plan never quietly accepts a target it cannot reach.
- Content reads the approved strategy only, not the raw research, so there is one artefact to audit.
