# Stage 1 — Campaign Request to Business Goal

The only job of this stage: turn a human's sentence into a structured, checkable goal — and refuse to invent anything the human did not say.

---

## 1. Input

| Field | Example | Notes |
|---|---|---|
| `raw_text` | "December is quiet. Push 3 Urban SUVs in Pune, around 1 lakh. Wakad branch has too much old stock. Do something on Instagram." | Stored verbatim, never rewritten |
| `requester` | `user_id`, name, role, store (may be null) | Determines who can approve later |
| `channel` | chat, form, email, phone-notes | Free text arrives from a person, not a system |
| `received_at` | timestamp with timezone | Asia/Kolkata |

A scheduled trigger is not a separate kind of input. It produces the same record with `requester.role = "system"` and a generated sentence. Stage 1 treats both identically.

---

## 2. What the stage does

1. **Capture.** Write the request to `campaign_requests` unchanged. Nothing downstream can start without this record.
2. **Extract.** Pull candidate values for: objective type, models, location, metric, target, timeframe, budget, constraints. Extraction never guesses — it either finds a value or records it as missing.
3. **Resolve.** Map names to IDs using reference data: "Pune" → `STORE_001`, `STORE_002`; "Urban SUV" → the model that exists in `inventory`. Unresolvable names become clarification questions, not silent defaults.
4. **Completeness check.** Compare against the required-field list. Any gap stops the goal at `awaiting_clarification` and sends questions to the requester.
5. **Wait.** Nothing moves. No research, no strategy, no content. The goal is parked until every question is answered.
6. **Propose the metric.** Once fields are complete, the AI chooses the success metric for each objective and writes a one-line reason. This is shown to the human, not hidden.
7. **Confirm.** Human accepts or edits the metric. On acceptance the goal becomes `approved` and is handed to the Orchestrator.

### The two rules that make this stage honest

**Provenance per field.** Every value in the goal is tagged `extracted` (it was in the text), `user_confirmed` (it came from an answer), or `inferred` (the AI's own reading). `inferred` values are always shown to the user as assumptions. A field with no source stays `null`. There is no fourth option where the model fills a gap quietly.

**Measurability gate.** A success metric is only accepted if the Analytics Agent can actually measure it from data we will have. "Increase brand love" is not a metric; "reach ≥ 250,000 on Instagram" is. This is checked at goal time, not discovered at reporting time three weeks later.

---

## 3. Data read

| Source | Used for | Not used for |
|---|---|---|
| `stores.json` | Resolve city/name → `store_id`; confirm store is active | — |
| `inventory.json` | Confirm the model exists at a target store | Deciding what to sell |
| `offers.json` | Flag offers that expire before the campaign ends | Choosing the offer |
| `budgets.json` | Flag when the requested budget exceeds what is left | Setting the budget |
| `_context/` | Brand rules that restrict a target (e.g. a locked tone or claim) | — |
| `_sop/` | Business rules: approval authorities, prohibited claims, lead-handling SLAs | — |

Rule: stage 1 reads reference data to **validate and flag**, never to **fill in** the goal.

---

## 4. Output schema

```json
{
  "goal_id": "GOAL_002",
  "version": 1,
  "status": "pending_confirmation",
  "origin": {
    "request_ids": ["REQ_014"],
    "requester": { "user_id": "U_17", "role": "marketing_manager", "store_id": null },
    "received_at": "2026-11-19T10:12:00+05:30",
    "channel": "chat",
    "raw_text": "December is quiet. Push 3 Urban SUVs in Pune, around 1 lakh. Wakad branch has too much old stock. Do something on Instagram."
  },
  "objectives": [
    {
      "objective_id": "OBJ_1",
      "type": "sales",
      "source_request_id": "REQ_014",
      "statement": "Sell 3 Urban SUVs in Pune during December",
      "success_metric": {
        "key": "completed_sales",
        "target": 3,
        "unit": "units",
        "rationale": "Requester stated 3 cars; sales is the closest measurable outcome in sales.json",
        "decision": "proposed"
      },
      "models": ["Urban SUV"],
      "store_ids": ["STORE_001", "STORE_002"],
      "period": { "start": "2026-12-01", "end": "2026-12-31" },
      "budget": { "amount": 100000, "currency": "INR", "scope": "shared_across_objectives" },
      "provenance": {
        "success_metric.target": "extracted",
        "period": "user_confirmed",
        "budget.amount": "extracted",
        "budget.scope": "inferred"
      }
    }
  ],
  "constraints": [
    { "text": "Wakad has ageing Urban SUV stock", "source": "extracted" }
  ],
  "assumptions": [
    { "text": "1 lakh is a shared ceiling for both objectives", "impact": "strategy cannot spend more than this without a new request" }
  ],
  "open_questions": [],
  "flags": [
    { "code": "BUDGET_EXCEEDS_REMAINING", "severity": "warning", "detail": "Requested 100000; 145000 remains across STORE_001 and STORE_002" }
  ],
  "amendments": [],
  "approval": { "approved_by": null, "approved_at": null }
}
```

**Status values:** `draft` → `awaiting_clarification` → `pending_confirmation` → `approved`. Terminal: `rejected`, `expired`. Post-approval edits re-enter at `pending_confirmation` with the version incremented.

**Mutability:** the object is edited only when a human asks for a change or an addition. Each edit appends to `amendments[]` (`what`, `why`, `by`, `at`, `version`) so the approved history is still readable. No autonomous rewrites.

**Nesting:** one goal holds many objectives. "Also increase test drives" is a new entry in `objectives[]` on the same goal, pointing at the request that asked for it — not a second goal.

---

## 5. Validation rules

| Code | Rule | On failure |
|---|---|---|
| V1 | `raw_text` and `requester` present | Reject (`rejected: no_request`) |
| V2 | At least one objective exists | Clarify |
| V3 | Every objective has type, metric, target value, period, ≥1 store | Clarify |
| V4 | Model-specific objectives name a model | Clarify |
| V5 | Period start ≤ end, start is in the future | Clarify |
| V6 | Target value is numeric and > 0 | Clarify |
| V7 | Metric is in the measurable-metric registry | Clarify with the closest measurable alternative |
| V8 | Every `store_id` exists and is `active` | Reject (`rejected: unknown_store`) with the active list |
| V9 | Every model exists in inventory at a target store | Flag `MODEL_NOT_IN_STOCK` |
| V10 | Budget ≥ 0, currency known | Clarify |
| V11 | Budget ≤ remaining (`total_budget − spent_amount`) for the period | Flag `BUDGET_EXCEEDS_REMAINING` |
| V12 | Active offer exists and covers the period | Flag `NO_COVERING_OFFER` / `OFFER_EXPIRES_BEFORE_PERIOD_END` |
| V13 | No `open_questions` left unanswered | Stay `awaiting_clarification` |
| V14 | Approval comes from the requester, authenticated as `origin.requester.user_id` | Reject the approval, not the goal |

Flags never block. Only V1, V2, V3 (in the sense of staying parked) and V8 stop the goal.

---

## 6. Failure handling

| Failure | Handling |
|---|---|
| Request is unreadable or not a campaign request | `rejected` with the reason; requester can resubmit |
| Model name ambiguous ("SUV") | Clarify with the known model list |
| Two targets that disagree (3 units vs 5 lakh) | Clarify. Do not pick one |
| City has no active store | `rejected: unknown_store` with the list of cities we serve |
| Metric not measurable | Clarify, offering the measurable proxy |
| Requester goes silent | Reminder at day 2, `expired` at day 5, request stays on record |
| Same request arrives twice | Link to the existing goal inside `amendments` instead of creating a second |
| Reference data contradicts itself | Raise `DATA_QUALITY` flag; do not "correct" the goal to match the data |

---

## 7. Handoff

To the Orchestrator, a **read-only snapshot**:

```json
{
  "goal_id": "GOAL_002",
  "version": 1,
  "objectives": [ /* as approved */ ],
  "snapshot_hash": "sha256:…"
}
```

The Orchestrator never writes to the goal. If the goal is amended later, the Orchestrator is notified with the new version and decides whether the running campaign is still valid.

---

## 8. Tabletop run — REQ_014

**Incoming (chat, 2026-11-19 10:12, marketing manager):**

> "December is quiet. Push 3 Urban SUVs in Pune, around 1 lakh. Wakad branch has too much old stock. Do something on Instagram."

**Turn 1 — extraction**

| Field | Value | Status |
|---|---|---|
| Objective 1 | Sell 3 Urban SUVs | `extracted` |
| Objective 2 | Clear ageing stock at Wakad | `extracted`, no target → gap |
| Objective 3 | Instagram activity | `extracted`, type unclear → gap |
| Stores | Pune → STORE_001, STORE_002 | `resolved` |
| Period | December | `inferred` — year and exact dates unstated → gap |
| Budget | ~1 lakh | `extracted`, scope unclear → gap |
| Metric | units → completed_sales | `proposed` |

**Turn 2 — status `awaiting_clarification`, three questions sent async:**

1. December of which year, and does the campaign run 1–31 Dec?
2. Is 1 lakh the total for all three objectives?
3. For the Wakad stock — how many cars, by when, and is Instagram organic or paid?

**Turn 3 — answers arrive (next morning):** 1–31 Dec 2026; 1 lakh total; clear at least 2 of the 6 ageing Urban SUVs; Instagram can be paid out of the same 1 lakh.

**Turn 4 — validation:**

- V11 passes but is tight: 100,000 requested, 145,000 remaining (`STORE_001` 75,000 + `STORE_002` 70,000).
- V12 warns: `OFFER_001` covers both stores to 2026-12-31 — fine. `OFFER_003` is `expired` despite `valid_until` 2026-11-25 — `DATA_QUALITY` flag, not used in planning.
- V9 passes: 10 Urban SUVs across the two stores, 110 and 120 days old at `STORE_002`/`STORE_004`.
- Objective 3 is rewritten as a measurable one: "Instagram reach ≥ 150,000, 1–31 Dec" — reach is measurable from platform data we will record.

**Turn 5 — metric proposal shown to user:**

> Sales: **3 completed sales** — the number you gave. Guardrail: cost per sale must stay under ₹33,000 to fit 1 lakh.
> Aged stock: **2 units of Urban SUV sold from Wakad stock** — your "at least 2".
> Instagram: **reach ≥ 150,000** — you said "do something", which isn't measurable; reach is the closest thing we can actually count.

**Turn 6 — user overrides one:** keeps sales at 3, tightens Instagram to "leads ≥ 40, not reach". Goal re-proposed, accepted, `approved`, version 1, `approved_by: U_17`.

**Turn 7 — handoff:** Orchestrator receives `GOAL_002` v1. Strategy can now start. Nothing was spent, nothing was published, no offer was assumed.

---

## 9. Reference-data issues found (not fixed yet)

| File | Issue | Suggested fix |
|---|---|---|
| `business_goals.json` | `target_revenue` 500,000 against `target_units` 3 (~4.35M) | Set revenue to 4,350,000, or drop the revenue target |
| `offers.json` | `OFFER_003` is `expired` but `valid_until` is 2026-11-25 | Set status `active` |
| `leads.json` | `LEAD_001` is `test_drive_booked` while `SALE_001` exists | Set lead status `sold` |
| `campaign_history.json` | `CAMP_001` revenue 9,000,000 vs 1,450,000 in `channel_data` | Align the two |
| `budgets.json` | December budget already shows `spent_amount` before December starts | Reinterpret as committed, or move the period |
| `competitor_data.json` | `"will be number"` placeholders and stray whitespace | Fill or remove those fields |
| `channel_data.json` | Website rows use `visitors`, paid rows use `impressions` | Keep, but document the per-channel schema |

---

## 10. Decisions I made (flag if you disagree)

- Blocking clarification, per your call: nothing moves until every answer is in.
- The requester approves their own goal, per your call. No second approver, so `_sop` carries no authority table at this stage.
- One goal, many objectives, each traced to a request; amendment log instead of revision chains.
- `spent_amount` read as committed spend, since the December period has not started.
- 5 days before an unanswered request expires.
- Flags warn; only missing stores and unreadable requests reject.
