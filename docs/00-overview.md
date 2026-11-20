# Agentic Marketing Workflow — Design Overview

Seven components, one goal: let a dealership marketing person say what they want, and get a campaign that is researched, planned, produced, published and measured — with every step auditable and nothing invented.

---

## The flow

```
 human (free text)
        │
        ▼
 ┌──────────────────┐   approved goal + hash
 │ 1 Request→Goal   │──────────────────────────┐
 └──────────────────┘                          │
        ▲                                      ▼
        │ ask async, block          ┌───────────────────────┐
        │ (status: awaiting)        │ 2 Orchestrator        │
        └───────────────────────────│ router · ledger · gate │
                                    └───────────┬───────────┘
                                                │
              ┌─────────────────────────────────┼──────────────────┐
              ▼                                 ▼                  ▼
     ┌─────────────────┐              ┌──────────────────┐  ┌──────────────┐
     │ 3 Research      │──findings───▶│ 4 Strategy       │  │ task states  │
     └─────────────────┘              └────────┬─────────┘  └──────────────┘
              ▲                                │ G1 human
              │                                ▼
              │                       ┌──────────────────┐
              │                       │ 5 Content        │
              │                       └────────┬─────────┘
              │                                │ G2 human (per asset hash)
              │                                ▼
              │                       ┌──────────────────┐        ┌───────────┐
              │                       │ 6 Distribution   │──G3──▶ │ platforms │
              │                       └────────┬─────────┘        └───────────┘
              │                                │ execution records
              │                                ▼
              │                       ┌──────────────────┐
              └────learnings──────────│ 7 Analytics      │
                                      └──────────────────┘
```

---

## What each component is responsible for

| # | Component | One-line responsibility | Gate |
|---|---|---|---|
| 1 | Request → Goal | Turn a sentence into a checkable goal; refuse to invent | Goal approval |
| 2 | Orchestrator | Decide who runs next, with what, and whether they may | Owns all gates |
| 3 | Research | Dated observations with sources; gaps where data is absent | — |
| 4 | Strategy | Choose audience, message, channels, budget, KPIs, with lineage | `G1` |
| 5 | Content | Creative specs bound to verified facts | `G2` |
| 6 | Distribution | Publish and spend, in the correct mode, idempotently | `G3` |
| 7 | Analytics | Measure with defined metrics and honest attribution | — |

---

## Five invariants

1. **No fact without a source.** A claim that cannot be traced is either a gap or a labelled assumption.
2. **No number without a definition.** Metrics are versioned artefacts, not adjectives.
3. **Nothing irreversible without a matching hash.** Approval binds to bytes, not to a conversation.
4. **One writer per fact.** Every value has exactly one owning component; everyone else reads.
5. **Missing data produces a gap, never a sentence.** The system says "we did not check" out loud.

---

## Where the LLM sits, and where it does not

| Work | Deterministic | LLM |
|---|---|---|
| Request capture, resolution to IDs | ✓ | |
| Completeness and validation rules | ✓ | |
| Routing and task DAG | ✓ | (ambiguous cases only, logged) |
| Budget arithmetic, reserve rules | ✓ | |
| Offer and stock checks | ✓ | |
| Hash binding, idempotency, retries | ✓ | |
| Extraction from free text | | ✓ |
| Interpretation of findings | | ✓ |
| Message, audience and channel design | | ✓ |
| Copy and creative specs | | ✓ |
| Diagnosis wording | | ✓ |

The pattern: **the model makes judgement calls at the edges; arithmetic, money and facts run through deterministic code.** Anything the model decides is written down as a decision with a reason, so a human can disagree with it later.

---

## Three approvals, one requester

Strategy → content → spend. Each approves a hash. Editing one caption returns one asset, not the campaign. Spend is approved twice because a plan is not an amount against a live account.

---

## What this design refuses to do

| Not done | Because |
|---|---|
| Invent business facts | A goal is the user's intent, not the model's guess |
| Claim an API exists | The capability matrix says `api`, `assisted` or `manual`, per store per platform |
| Render video | Content writes specs; a camera still needs a human |
| Auto-publish or auto-spend | Money defaults to unspent; publication needs a matching hash |
| Say "caused" without a join | Attribution is verified, correlated, or unattributed |
| Put lead names in creative | PII stays in the CRM, not in ads |
| Average two conflicting numbers | Contradictions are shown, not smoothed |

---

## What accumulates

Requests, goals with amendment history, run events, approved artefacts with hashes, a content registry, an execution registry, performance reports with maturity, metric definitions, and learnings. The learnings are what make cycle two smarter than cycle one — and they are derived, never typed in.

---

## Honest limits

- Platform access will be partial; half this system runs in `assisted` mode in a real dealership.
- Attribution will be incomplete; most sales will not carry a campaign id.
- Samples will be small; one campaign and eight sales is a hypothesis, not a trend.
- Humans will approve late; the system stalls visibly rather than proceeding silently.

---

## Reading order

| File | Covers |
|---|---|
| `01-request-to-business-goal.md` | Request intake, goal schema, validation, tabletop |
| `02-orchestrator.md` | Routing, state machine, gates, failure classes |
| `03-research-agent.md` | Evidence model, observed vs interpreted, freshness |
| `04-strategy-agent.md` | Positioning, budget arithmetic, feasibility |
| `05-content-agent.md` | Creative specs, fact binding, offer validity |
| `06-distribution-agent.md` | Capability matrix, idempotency, paused spend |
| `07-analytics-agent.md` | Metric definitions, attribution tiers, learnings |
| `08-handoffs-state-errors.md` | Cross-cutting contracts and error taxonomy |
| `09-data-map.md` | Dataset → stage mapping, missing data, seed fixes |
| `10-implementation-options.md` | What to build, if anything |
