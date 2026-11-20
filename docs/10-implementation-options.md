# What To Do About Code

You asked for the design first. This is the follow-on decision, not a plan to start building.

---

## Four options

| | Option | Builds | Proves | Cost |
|---|---|---|---|---|
| **A** | Docs only | The nine files above | You can think | already done |
| **B** | Contract harness | Schemas + all seven stages with stubbed reasoning + fake platform adapters + a dry-run of `GOAL_002` that prints the audit trail | The contracts hold, gates enforce, nothing fabricates | ~1 day |
| **C** | Live agents | LLM calls per stage + real platform adapters | It works on a real account | weeks, and needs credentials you don't have |
| **D** | Hybrid | B, plus real LLM calls at exactly three points: request extraction, research interpretation, content draft | The judgement is real, the plumbing is deterministic | ~2–3 days |

**Recommendation: B now, D if you have the time, C never for an interview.**

Reason: an interviewer can already call an LLM. What they cannot easily see is whether a gate actually blocks a hash-mismatched asset, whether a retry duplicates a post, or whether an unattributable sale gets claimed. Those are the interesting parts, and they are all deterministic — so they are cheap to build and impossible to fake in a demo.

---

## What B contains

```
src/
  contracts/          zod schemas for every artefact (goal, report, plan, pack, execution, perf)
  stages/
    goal.ts           extract → complete → clarify → approve
    orchestrator.ts   routing table, DAG, state machine, gates, retry classes
    research.ts       internal queries + declared gaps
    strategy.ts       budget arithmetic, funnel math, positioning check
    content.ts        spec generation, fact binding, offer re-check
    distribution.ts   capability matrix, idempotency, paused campaigns
    analytics.ts      metric definitions, attribution tiers, learnings
  data/               loaders over /data
  adapters/           fake platform: publish, reject, timeout, duplicate
  fixtures/           goal-002.json, the clarification answers, the expired offer
  run.ts              one command → the whole trace, printed as tables
```

### The five assertions worth demonstrating

1. An asset whose hash no longer matches `G2` is **refused** by distribution.
2. A publish timeout triggers read-back, not a duplicate post.
3. A creative referencing `OFFER_003` **blocks** that branch while organic keeps running.
4. A sale without `campaign_id` is reported as correlated, and the words "produced" and "caused" never appear next to it.
5. A goal with a missing period **never leaves** `awaiting_clarification`.

If the harness demonstrates those five, the design is credible. If it demonstrates a pretty dashboard, it isn't.

---

## What I would not build

- **A UI.** It hides the design behind pixels and eats the available time.
- **Real platform integrations.** No credentials, no ad account, and the failure modes are the interesting part anyway — so they are simulated.
- **Video rendering.** Content produces specs, by design.
- **An agent framework.** Routing, gates and retries are thirty lines of explicit code. A framework would obscure the thing being judged.

---

## If you go ahead

Say which option. B starts with the contracts, because every stage is defined by its output shape, and the schemas are the design in executable form.
