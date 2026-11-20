# Stage 2 — Orchestrator

The Orchestrator is a router and a ledger, not a manager with opinions. It decides *who runs next, with what, and whether it is allowed to run*. It never writes marketing. Every decision it makes should be inspectable in a log without asking an LLM what it was thinking.

---

## 1. Input

An approved Business Goal snapshot from stage 1:

```json
{ "goal_id": "GOAL_002", "version": 1, "objectives": [ /* … */ ], "snapshot_hash": "sha256:…" }
```

Plus: the run's spend envelope (derived from the goal budget and `budgets.json`), the store calendar, and the current state of any run already open for this goal.

---

## 2. What it does

| # | Responsibility | Concrete action |
|---|---|---|
| 1 | Read the goal | Loads the snapshot; refuses to start if `status != approved` or the hash does not verify |
| 2 | Plan | Turns objectives into a task DAG: which agents, in what order, with which dependencies |
| 3 | Scope context | Builds a per-task input bundle — each agent gets its slice, not the whole goal |
| 4 | Delegate | Sends one task at a time over the shared task envelope; agents never call each other |
| 5 | Track | Appends every state change to an event log; the run's `state` is a projection of that log, never hand-edited |
| 6 | Guard | Enforces gates, budget ceilings and idempotency before anything is published or spent |
| 7 | Recover | Classifies failures, retries the safe ones, escalates the rest, and can send the goal back to the user for amendment |

### Four rules that keep it honest

**Pass references, not summaries.** The Orchestrator hands Strategy a pointer to the research artifact, not its own paraphrase. A controller that summarises becomes an unaudited editor.

**One writer per fact.** Research owns observations. Strategy owns the plan. Content owns copy. Distribution owns published state. Analytics owns performance facts. The Orchestrator writes only the run log. No two components ever hold the same mutable fact.

**Deterministic where possible.** Routing is a lookup table, not a judgement call. An LLM is used only for genuinely ambiguous routing (e.g. "is this goal a repeat of a campaign we already have a playbook for?"), and its answer is recorded as a decision with a reason.

**Nothing irreversible without a gate.** Research and strategy run freely. Publishing and spending never do.

---

## 3. Routing

| Objective type | Research | Strategy | Content | Distribution | Analytics |
|---|---|---|---|---|---|
| sales | required | required | organic + paid | organic + paid | checkpoints + close-out |
| leads | required | required | organic + paid | organic + paid | checkpoints + close-out |
| awareness | required | required | organic, paid optional | per strategy | reach checkpoints |
| education | required | required | organic | organic | engagement close-out |
| trust | required | required | organic | organic | engagement close-out |

Engine-specific rules:

- **Research reuse.** Prior findings may be reused only if their `observed_at` is within 30 days; competitor observations within 14 days. Stale findings are refreshed, not silently reused.
- **First run vs repeat.** If an approved playbook exists for a structurally similar goal, Strategy still runs — but Research may be scoped to "what changed since last time".
- **Paid is conditional.** Paid tasks are created only if Strategy proposes promoted content *and* the spend envelope can cover it. Otherwise a `paid_budget_insufficient` flag is raised on the plan, not in the middle of execution.
- **Store split.** Distribution tasks are created per store × platform, because each store publishes under its own account and each spend needs its own approval.
- **Analytics timing.** A close-out task is always scheduled at `period.end + 7 days`, because sales attribution arrives after the campaign ends. Interim checkpoints are optional and defined by Strategy.

---

## 4. Data read and written

**Reads:** approved goal snapshot, `budgets.json` (remaining spend), `offers.json` + `inventory.json` (feasibility at planning time), `_sop/` (gate rules, allowed approvers, prohibited claims), `_context/` (brand rules), agent artifacts from earlier tasks.

**Writes:** the run record, the append-only event log, the artifact registry (pointers only), gate decisions. It writes **no** market, creative or performance data.

---

## 5. Output schema — CampaignRun

```json
{
  "run_id": "RUN_007",
  "goal": { "goal_id": "GOAL_002", "version": 1, "snapshot_hash": "sha256:…" },
  "state": "awaiting_strategy_approval",
  "spend_envelope": { "approved": 100000, "committed": 0, "spent": 0, "currency": "INR" },
  "plan": {
    "created_at": "2026-11-20T11:02:00+05:30",
    "tasks": [
      {
        "task_id": "T1",
        "agent": "research",
        "purpose": "Market, competitor and historical findings for Urban SUV in Pune",
        "depends_on": [],
        "input_ref": { "goal_objectives": ["OBJ_1", "OBJ_2"], "reuse_after_days": 30 },
        "status": "done",
        "attempts": 1,
        "gate": null,
        "budget_slice": 0
      },
      {
        "task_id": "T2",
        "agent": "strategy",
        "depends_on": ["T1"],
        "status": "done",
        "gate": "G1_strategy"
      },
      {
        "task_id": "T5",
        "agent": "distribution",
        "purpose": "Schedule 6 organic Instagram posts — STORE_001",
        "depends_on": ["T4"],
        "status": "blocked",
        "gate": "G3_publish",
        "budget_slice": 0,
        "idempotency_key": "RUN_007:T5:ART_11:v1"
      }
    ]
  },
  "artifacts": [
    { "artifact_id": "ART_11", "type": "content_pack", "produced_by": "T4",
      "version": 2, "uri": "artifacts/RUN_007/ART_11.json",
      "approved_by": null, "approved_at": null, "hash": "sha256:…" }
  ],
  "gates": [
    { "gate": "G1_strategy", "status": "approved", "by": "U_17", "at": "2026-11-20T12:10:00+05:30", "artifact_hash": "sha256:…" }
  ],
  "open_issues": [
    { "code": "OFFER_EXPIRED", "task_id": "T4", "detail": "Creative references OFFER_003 which expired 2026-11-25", "owner": "human", "raised_at": "2026-11-20T12:40:00+05:30" }
  ]
}
```

The **event log** is separate and append-only: `{at, type, task_id, actor, from_state, to_state, detail}`. The run's `state` is derived from it. Any disagreement between log and record is resolved in favour of the log.

---

## 6. State machine

```
created → planning → researching → strategizing → awaiting_strategy_approval
        → producing_content → awaiting_content_approval → scheduling
        → awaiting_publish_approval → publishing → live → measuring → closed
```

Side states: `awaiting_clarification` (goal amended), `blocked` (needs a human decision), `failed` (unrecoverable for this run), `cancelled`, `paused` (budget or external hold).

Every transition has a guard. Examples: no entry to `publishing` without an approved gate whose `artifact_hash` matches the current artifact; no entry to `scheduling` if the plan contains an unresolved `OFFER_EXPIRED` issue.

---

## 7. Handoff — the task envelope

Every agent call, both directions, uses one shape. This is the contract stages 3–8 all share:

```json
{
  "task_id": "T1",
  "agent": "research",
  "status": "ok",
  "output_ref": "artifacts/RUN_007/ART_01.json",
  "summary": "one paragraph a human can read",
  "sources": [{ "type": "internal_dataset", "ref": "campaign_history.json#CAMP_001", "observed_at": "2026-10-31" }],
  "assumptions": [{ "text": "Pune buyers respond to EMI framing", "basis": "one prior campaign" }],
  "open_questions": [],
  "errors": []
}
```

`status` is one of `ok`, `partial`, `blocked`, `failed`. `partial` is legitimate and must be explicit: two sources retrieved out of three is not a failure, but it is not `ok` either.

Agents return references plus a short human summary. They do not return prose dumps into the Orchestrator's context.

---

## 8. Approval gates

| Gate | Blocks | What the human sees | Invalidated by |
|---|---|---|---|
| `G1_strategy` | All content work | Objective, audience, message, channel mix, organic vs paid split, budget allocation, KPIs, stated assumptions | Any change to the strategy artifact |
| `G2_content` | Scheduling | Every asset with its channel, store, caption, verified vehicle facts and offer reference | Any edit to an approved asset |
| `G3_publish` | Publishing and any spend | Final schedule, platform accounts, planned spend per channel, offer expiry check | Any change to schedule, spend or asset |

Gates bind to an artifact **hash**. Editing an approved caption after approval breaks the hash, so that asset alone returns to `G2`. The rest of the campaign does not stop.

Spend needs `G3` even when the budget was approved in `G1`, because `G1` approves a *plan* and `G3` approves an *amount against a live account*.

### Gates stay separate, and every task is auditable

Per your call: three separate stops, and the requester can open any agent's work on its own — input, output, sources, assumptions, version, hash. Nothing is presented as one fused "campaign plan" that a human has to take on trust.

So each task must return an inspectable artifact, not a status line:

| Field | Why it exists |
|---|---|
| `input_snapshot` | what the agent was actually given, so a bad output can be traced to a bad input |
| `output_ref` + `hash` | the exact bytes the human approved |
| `sources` | where each fact came from, with its observation date |
| `assumptions` | what the agent filled in on its own |
| `decisions` | the choices it made and the reason for each |
| `open_questions` | what it could not resolve |
| `runtime` | model, prompt version, cost, duration — enough to reproduce the run |

This is a requirement on stages 3–8, not only on the Orchestrator.

---

## 9. Failure handling

| Class | Example | Action |
|---|---|---|
| Transport | timeout, 429, 5xx | Retry twice, backoff 5s / 30s |
| Contract | agent returns output that fails schema validation | One repair attempt with the validation error attached; then `blocked` |
| Business precondition | offer expired, stock gone, budget exhausted | No retry. `blocked` + `open_issue` + notify human |
| Irreversible | publish failed mid-way, ad account rejected the ad | Never auto-retry. Read back platform state first, reconcile, then decide |
| Infeasible target | Strategy judges 3 units unreachable at 1 lakh | Escalate to the user to amend the goal. The Orchestrator never lowers a target to make the plan pass |

Retries are capped at 2 and only apply to read operations. A publish retry is a duplicate-post risk, so publication is guarded by `idempotency_key = hash(run_id, task_id, artifact_version, channel, scheduled_time)` and by a read-back check.

Every failure writes one line to the event log and, if a human must act, one `open_issue` with an owner. Failures are never swallowed into a friendly summary.

---

## 10. Tabletop run — GOAL_002 v1

**Plan built (11 tasks):** T1 research → T2 strategy → **G1** → T3 organic content, T4 paid content → **G2** → T5 organic scheduling (STORE_001), T6 organic scheduling (STORE_002), T7 paid setup (Instagram + Google) → **G3** → T8 live → T9 checkpoint (weekly) → T10 close-out (2027-01-07).

**What actually happens:**

1. **T1 research times out** on one external source. Retry once, then returns `partial`: competitor observation from 2026-11-19 present, industry trend source missing. Orchestrator records `partial` and lets Strategy run — the missing source is written into `open_questions`, so the gap reaches the human instead of vanishing.
2. **T2 strategy returns** with a stated assumption: "December is a weak month; the aged-stock objective should carry the paid budget." G1 is raised.
3. **G1 approved at 12:10.** Spend envelope locked at 100,000, 70,000 to aged stock, 30,000 to test-drive ads.
4. **T4 paid content produced**, referencing `OFFER_003` for a Nashik-style discount to Pune. Content's own validation catches that `OFFER_003` is expired and does not cover `STORE_002` either. It returns `blocked` with `OFFER_INVALID`. **No spend, no ad account touched.**
5. **Orchestrator raises `open_issue` with owner `human`.** Run state → `blocked` on that branch only. T3 organic content and T5/T6 keep moving; one bad creative does not freeze the campaign.
6. **Human fixes** the creative to use `OFFER_001` (valid to 2026-12-31, covers both stores). T4 reruns as version 2.
7. **G2 approved**, then one caption is edited the next morning — its hash no longer matches, so that asset alone goes back to G2. The other nine assets stay approved and scheduled.
8. **G3 approved at 18:40.** T5–T7 publish. Each write returns a platform ID and timestamp into the artifact registry, not into the run's `state`.
9. **T9 checkpoint** on 2026-12-08 reports reach and leads per channel. T10 closes out one week after the campaign, because sales land late.

What the tabletop is meant to show: one failure does not stop the workflow, nothing irreversible happens without a matching hash, and the missing research source is visible to a human rather than smoothed over.

---

## 11. Decisions I made (flag if you disagree)

- Routing is a table plus a small set of guards, not an LLM planner.
- Three gates: strategy, content, publish/spend. All approved by the requester, per your stage-1 rule, and kept as three separate stops so each agent's work can be audited on its own.
- Retries only on reads; publishing is never retried blindly.
- Strategy may declare a target unreachable, and the only resolution is the user amending the goal.
- The run log and artifact registry are new — they are the minimum needed to satisfy "track workflow status", not a new business dataset.

---

## 12. Resolved

Gates stay separate (G1, G2, G3), with a per-task audit view behind each one. No combined approval.
