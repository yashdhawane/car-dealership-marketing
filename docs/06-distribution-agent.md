# Stage 6 — Distribution / Execution Agent

This is the only component that touches the outside world. Everything about it should assume the outside world is hostile, slow, and inconsistent.

---

## 1. Input

| Input | Rule |
|---|---|
| Content pack + per-asset hash | Only assets whose hash matches an approved `G2` may be queued |
| Channel and spend plan | Only lines approved at `G3` |
| Store × platform account registry | Which handle, page or ad account belongs to which store |
| Schedule | Slot, timezone (Asia/Kolkata), per store |
| Offer and stock snapshot | Re-validated at execution time, not trusted from upstream |

---

## 2. Capability matrix — the honest API table

Nothing is assumed to have an API. Each platform × store is one row, seeded by hand, kept current.

| Platform | Organic publish | Paid campaign | Insights read | Fallback mode |
|---|---|---|---|---|
| Instagram / Meta | Graph API, if the page token is valid | Marketing API, ad account required | API, delayed up to 48h | `assisted`: generate post text + image, human posts |
| Google Ads | not applicable | API, developer token + customer id | API | `assisted`: export campaign sheet |
| Google Business Profile | API where enabled, else manual | not applicable | limited | `manual` |
| YouTube | API with channel OAuth | via Google Ads | API | `assisted` |
| WhatsApp Business | template messages only, opt-in required | not applicable | limited | `manual` |
| Website / landing pages | CMS write, if connected | not applicable | analytics API | `manual` |
| Dealer CRM | two-way sync where licensed | not applicable | export | `manual_export` |

Three modes, and the system always says which one it is in:

- **`api`** — the agent performs the action and records the platform's response.
- **`assisted`** — the agent produces the exact payload, a human clicks publish, then confirms. The record is marked `assisted`, never `published_by_agent`.
- **`manual`** — the agent produces a brief and a checklist; nothing claims it went live.

An execution record that says "published" when a human did it is a lie that will eventually be discovered in an audit.

---

## 3. What it does

1. **Adapt.** Final check that the asset fits the platform (dimensions, caption length, required fields). Adaptation that changes meaning goes back to Content; trimming whitespace does not.
2. **Re-validate.** Offer still active, stock still present, store still active, asset hash still approved.
3. **Queue.** Build the exact payload, attach an idempotency key, schedule the slot.
4. **Create paid, paused.** Campaigns are created in a paused state. Nothing spends before activation.
5. **Activate only on `G3`.** Activation is a separate action with its own record.
6. **Record.** Platform IDs, timestamps, status, and evidence (screenshot or API response reference).
7. **Report.** Per-item status to the Orchestrator. Partial success is a first-class outcome.

---

## 4. Idempotency and read-back

Every write carries:

```
idempotency_key = hash(run_id, task_id, asset_hash, platform, account_id, scheduled_slot)
```

On timeout — the worst case, because you don't know if it landed — the agent **reads back first**. It queries the platform for recent posts or campaigns and looks for the key or the content. Only if nothing is found does it retry. Blind retry is how a dealership posts the same reel three times.

Deletion is never automatic. Unpublishing is an action with the same weight as publishing: it needs approval.

---

## 5. Output schema

```json
{
  "execution_id": "EXE_009",
  "run_id": "RUN_007",
  "mode_summary": { "api": 7, "assisted": 2, "manual": 0 },
  "items": [
    {
      "item_id": "IT_21",
      "asset_id": "AST_11",
      "asset_hash": "sha256:…",
      "store_id": "STORE_001",
      "platform": "instagram_organic",
      "account_id": "IG_SHIVAJINAGAR",
      "action": "publish_organic",
      "mode": "api",
      "scheduled_for": "2026-12-02T18:30:00+05:30",
      "executed_at": "2026-12-02T18:31:04+05:30",
      "status": "published",
      "platform_refs": { "media_id": "1789…", "permalink": "https://…" },
      "evidence_ref": "evidence/RUN_007/IT_21.png",
      "idempotency_key": "a1f3…"
    },
    {
      "item_id": "IT_30",
      "asset_id": "AST_31",
      "platform": "google_ads",
      "action": "create_paid_paused",
      "mode": "api",
      "status": "created_paused",
      "platform_refs": { "campaign_id": "2381…", "ad_group_id": "991…" },
      "planned_spend": 55000
    },
    {
      "item_id": "IT_41",
      "asset_id": "AST_41",
      "platform": "whatsapp_business",
      "action": "publish_organic",
      "mode": "manual",
      "status": "brief_ready",
      "blocked_reason": "no template approval, opt-in list required"
    }
  ],
  "spend_committed": { "amount": 0, "currency": "INR", "note": "all campaigns paused; activation requires G3" },
  "failures": [
    { "item_id": "IT_27", "code": "AUTH_EXPIRED", "detail": "page token expired 2026-12-01", "retryable": false }
  ]
}
```

---

## 6. Validation and guards

| Code | Rule | On failure |
|---|---|---|
| D1 | `asset_hash` matches an approved `G2` gate | Item refused |
| D2 | Offer active and covers store and date at execution time | Item `blocked` |
| D3 | Account belongs to the store, token valid | Item `blocked: AUTH_EXPIRED` |
| D4 | Cumulated planned spend ≤ `G3` ceiling | Item refused |
| D5 | Platform limits satisfied | Back to Content |
| D6 | Slot is in the future, timezone correct | Item rescheduled or refused |
| D7 | Paid action is `create_paused` unless activation is explicitly requested | Item refused |

D7 is the one that matters most: the default state of money is "not spent".

---

## 7. Failure handling

| Failure | Handling |
|---|---|
| Network timeout on publish | Read back, then decide. Never blind retry |
| Token expired | `AUTH_EXPIRED`, human refresh, item retried once |
| Platform policy rejection | Return the platform's reason verbatim to Content |
| Rate limit | Honor `Retry-After`, back off, reschedule the slot |
| Partial batch (3 of 6 posts) | Record per item; report `partial`; do not claim batch success |
| Duplicate detected | Do not publish; record `duplicate_suppressed` |
| Spend about to exceed ceiling | Refuse and raise an issue; never truncate a campaign silently |

---

## 8. Handoff

- **Analytics** receives the registry of what actually went live: platform IDs, timestamps, spend. Analytics reads this, not the plan — plans are intentions.
- **Orchestrator** receives per-item status and failures.
- **Content** receives rejected items with the platform's reason.

---

## 9. Tabletop — GOAL_002

- 6 organic posts across `STORE_001` and `STORE_002`, `api` mode: published 2 Dec, platform IDs recorded.
- 2 paid campaigns: search at 55k and social at 30k, both created **paused**.
- Activation waits for `G3` on 21 Nov 18:40. Two campaigns activate; the 15k reserve stays unallocated until the checkpoint condition is met.
- One post fails with `AUTH_EXPIRED` on the Wakad page token. The item blocks, the other five publish. A human refreshes the token, the item runs, and the record shows two attempts with one `idempotency_key` — one post, not two.
- WhatsApp is requested by nobody in this run, which is the correct outcome: no template approval, no opt-in list, so it would have come back `manual` with a brief and no claim of publication.

---

## 10. Decisions I made (flag if you disagree)

- Three execution modes, always declared. The system never claims it published something a human published.
- Paid campaigns are created paused and activated separately. Money defaults to unspent.
- Read-back before retry on any timeout; deletion requires approval.
- Partial success is reported as partial, per item.
- The capability matrix is hand-maintained configuration, because assuming an API exists is how integrations fail in week two.
