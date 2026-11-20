# Cross-cutting — Handoffs, Shared State, Errors, Gates

Stages 1–7 are contracts. This file is what they all have in common: how work moves, who owns which fact, what fails, and what a human gets asked.

---

## 1. One envelope for every handoff

```
Orchestrator → Agent : { task_id, agent, input_snapshot, constraints, deadline }
Agent → Orchestrator : { task_id, status, output_ref, summary, sources,
                        assumptions, decisions, open_questions, errors, runtime }
```

- `status ∈ {ok, partial, blocked, failed}`. `partial` is a real answer and must be used.
- `output_ref` is a pointer plus a hash. Agents pass **references**, never paraphrases of each other's work.
- Any agent-to-agent communication goes through the Orchestrator. No side channels, because a side channel is an unaudited decision.

---

## 2. Who owns which fact

| Fact | Owner (single writer) | Everyone else |
|---|---|---|
| Request text, clarification answers | Stage 1 intake | read-only |
| Goal and its amendments | Stage 1 | read-only snapshot |
| Run status, task states, gates | Orchestrator | read-only |
| Market observations, competitor sightings | Research | read-only |
| Plan, budget lines, KPIs | Strategy | read-only |
| Copy, hooks, asset specs | Content | read-only |
| Published state, platform IDs, spend committed | Distribution | read-only |
| Performance numbers, attribution, learnings | Analytics | read-only |

Two writers on one fact is the defect that makes every later number arguable. When two components need the same fact, one computes it and the other reads it.

---

## 3. What accumulates, and what it is for

| Store | Written by | Grows per | Purpose |
|---|---|---|---|
`campaign_requests` | stage 1 | request | Raw truth, never edited |
`business_goals` | stage 1 | goal / amendment | What was agreed |
`campaign_runs` + `run_events` | Orchestrator | run | Audit trail; state is derived from events |
`artifact_registry` | Orchestrator | version | Exact approved bytes + hashes |
`content_registry` | Content → Distribution | asset | What was made, what happened to it |
`execution_registry` | Distribution | item | What actually went live, with platform IDs |
`performance_reports` | Analytics | period | Outcome, with maturity |
`metric_definitions` | Analytics (versioned) | definition change | Comparability |
`learnings` | Analytics | cycle | What the next cycle starts from |
`capability_matrix` | human-maintained | platform change | What can actually be automated |

Everything above is derived from what already exists, except the capability matrix and metric definitions, which are configuration.

---

## 4. Error taxonomy

| Code | Class | Retry | Owner | Effect |
|---|---|---|---|---|
| `NO_REQUEST` | blocking | no | human | Request rejected |
| `UNKNOWN_STORE` | blocking | no | human | Goal rejected, store list returned |
| `AMBIGUOUS_ENTITY` | question | no | human | Parked at `awaiting_clarification` |
| `METRIC_NOT_MEASURABLE` | question | no | human | Metric reproposed |
| `BUDGET_EXCEEDS_REMAINING` | flag | no | human | Plan may proceed with a warning |
| `NO_COVERING_OFFER` | flag | no | human | Strategy reconsiders the message |
| `OFFER_INVALID` | blocking | no | human | Content branch blocks; campaign continues |
| `MODEL_NOT_IN_STOCK` | flag | no | human | Target flagged as risky |
| `TARGET_AT_RISK` | escalate | no | human | Goal amended, never silently lowered |
| `POSITIONING_MISSING` | flag | no | human | Positioning marked `assumed` |
| `DATA_QUALITY` | flag | no | human | Affected numbers carry a caveat |
| `UNTRUSTED_CONTENT` | flag | no | system | Retrieved text treated as data |
| `TRANSPORT` | transient | 2× | system | Backoff 5s / 30s |
| `RATE_LIMIT` | transient | honour `Retry-After` | system | Slot rescheduled |
| `SCHEMA_INVALID` | contract | 1 repair | system | Then `blocked` |
| `AUTH_EXPIRED` | blocking | 1 after refresh | human | Item blocked, rest continues |
| `DUPLICATE_SUPPRESSED` | inform | no | system | No second publish |
| `PARTIAL_SOURCE` | inform | no | system | `partial` status with gaps |
| `NO_HISTORY` | inform | no | system | Gap; baseline "none available" |

Rules: retries apply only to reads; irreversible actions read back before any retry; every failure writes one event and, if a human must act, exactly one issue with an owner and a deadline.

---

## 5. Gates

| Gate | Opens | Binds to |
|---|---|---|
| `G1_strategy` | Content work | Strategy artifact hash |
| `G2_content` | Scheduling | Each asset hash, individually |
| `G3_publish` | Publishing and spend | Schedule + planned spend + asset hashes |

- The requester approves their own work (your call from stage 1).
- Approval binds to a hash. Edit anything afterwards and that item alone returns to the gate.
- No gate can be satisfied by an agent, a default, or silence.
- Timeouts never auto-approve. Silence means "not approved", forever, unless an SOP explicitly says otherwise.

---

## 6. When two components disagree

| Situation | Resolution |
|---|---|
| Research says one thing, Analytics another | Analytics wins on performance, Research wins on market. Both are shown |
| Strategy and Content disagree on a claim | The fact registry decides. Neither opinion overrides a dataset |
| Distribution reports published, Analytics sees no data | Execution registry is authority for "did it go live"; Analytics reports the gap |
| Plan says 100k, execution says 95k | Execution wins. Plans are intentions |
| Goal amended mid-run | Orchestrator pauses affected tasks, compares versions, asks whether to restart or finish |

---

## 7. Three levels of human involvement

| Level | Behaviour | Used for |
|---|---|---|
| **Block** | Nothing proceeds | publishing, spend, goal approval, expired offers |
| **Flag** | Proceeds, warning recorded | budget tight, model low on stock, stale research |
| **Inform** | Logged, no interruption | duplicate suppressed, partial source |

A system that blocks on everything is unusable; one that flags everything is unauditable. The class table above is the whole policy.

---

## 8. Privacy and consent

`leads.json` contains names and `consent_to_contact`. That is personal data, and it changes what agents may do:

- Agents may read lead records in aggregate for analytics.
- Agents may not place lead names, phone numbers or emails into creative copy, ad audiences, or research output.
- Consent governs outreach, not ad targeting. A lead without consent cannot be messaged, but an audience can still be targeted by aggregate criteria.
- Delivery moments and salesperson videos need a consent record before production, not after.

---

## 9. What will actually break first

Honest list, because a design that pretends otherwise is not credible:

1. **Platform access.** Tokens expire, ad accounts get restricted, insights lag. Hence the capability matrix and `assisted` mode.
2. **Attribution.** Most dealership sales will not carry a campaign id. The three-tier model is designed for that reality, not around it.
3. **Data freshness.** Stock and offers change during a two-week campaign. Hence re-validation at execution.
4. **Small samples.** Four leads is not a market. Confidence capping is the defence.
5. **Humans approving late.** Gates stall campaigns. The design fails loudly rather than publishing unattended.
