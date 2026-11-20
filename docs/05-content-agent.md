# Stage 5 — Content Agent

Content turns an approved strategy into production-ready specs. It does not invent facts, it does not choose channels, and — deliberately — it does not pretend to shoot video.

---

## 1. Input

Per task, from the Orchestrator:

```json
{
  "task_id": "T3",
  "plan_ref": "PLAN_002",
  "goal_ref": { "goal_id": "GOAL_002", "version": 1 },
  "slice": {
    "objective_ids": ["OBJ_1", "OBJ_2"],
    "messages": ["MSG_1"],
    "channel": "instagram_organic",
    "formats": ["reel", "carousel", "story"],
    "store_ids": ["STORE_001"],
    "planned_count": 3
  },
  "facts": { "vehicles": ["VEH_001"], "offers": ["OFFER_001"] },
  "voice": "context/brand_voice.json",
  "rules": ["_sop/prohibited_claims.md"]
}
```

The facts slice is the only permitted source of vehicle claims. Content does not browse inventory itself, so a stock change cannot silently rewrite a script mid-run.

---

## 2. The scope boundary, stated plainly

Content produces a **creative spec**: hook, script or voiceover, on-screen text, caption, CTA, disclaimers, shot list, duration, aspect ratio, and the exact facts used. It does not render a video, and it does not fabricate an asset that looks published.

Whether the spec becomes a file is the next question, not this one:

| Mode | Who produces the file | Used when |
|---|---|---|
| `template_render` | a deterministic renderer fills a template | offer cards, price posts, carousels |
| `human_shoot` | salesperson or videographer follows the spec | walkarounds, delivery moments, salesperson videos |
| `stock_edit` | editor cuts supplied footage to the spec | trend reels, comparisons |

This is the honest answer to "the agent made a reel". An agent can write the reel; someone still has to point a camera. Saying so is what prevents a demo from quietly becoming a lie.

---

## 3. Content types and what each needs

| Type | Objective fit | Needs | Typical format |
|---|---|---|---|
| Walkaround | awareness, trust | verified variant facts, footage | reel, 20–40s |
| Feature highlight | education, consideration | spec sheet — **currently missing** | reel, carousel |
| Price / EMI explainer | education, leads | price, offer terms, finance disclaimer | carousel, reel |
| Comparison | consideration | verified facts for two models | carousel |
| Offer / promotion | leads, conversion | active offer, expiry, terms | static, story |
| Delivery moment | trust, social proof | customer consent, footage | reel, 15–30s |
| Salesperson video | trust | staff consent, script | reel |
| Trend / short-form | awareness | trend reference, brand-safety check | reel |
| Test-drive booking ad | leads | store address, booking link | carousel, static |
| Retargeting creative | conversion | site audience exists, uplift message | carousel, static |

Two rows are already blocked by missing data — feature highlights and comparisons have no spec sheet. That is a data gap surfaced at design time, not a surprise in production.

---

## 4. Output schema

```json
{
  "pack_id": "CP_005",
  "run_id": "RUN_007",
  "plan_ref": "PLAN_002",
  "master_ideas": [
    { "idea_id": "IDEA_1", "frame": "EMI, explained line by line",
      "message_id": "MSG_1", "objective_ids": ["OBJ_1", "OBJ_2"],
      "testable_against": "IDEA_2" },
    { "idea_id": "IDEA_2", "frame": "Space for the family",
      "message_id": "MSG_1", "objective_ids": ["OBJ_1"],
      "testable_against": "IDEA_1" }
  ],
  "assets": [
    {
      "asset_id": "AST_11",
      "idea_id": "IDEA_1",
      "channel": "instagram_organic",
      "format": "reel",
      "store_id": "STORE_001",
      "objective_ids": ["OBJ_1", "OBJ_2"],
      "production_mode": "human_shoot",
      "duration_sec": 30,
      "aspect_ratio": "9:16",
      "hook": "3 seconds: '1,45,000 off sounds great. Here is what you actually pay each month.'",
      "script": [
        { "t": "0-3s", "visual": "salesperson at the Urban SUV, tailgate", "vo": "1,45,000 off sounds great." },
        { "t": "3-12s", "visual": "cut to on-road price card on screen", "vo": "Here is the number people forget — on-road, after the benefit." },
        { "t": "12-24s", "visual": "EMI table, three tenures", "vo": "Three tenures. Same car. Pick what fits your month." },
        { "t": "24-30s", "visual": "store front, phone number", "vo": "Walk in at Shivaji Nagar and see the sheet." }
      ],
      "on_screen_text": ["Urban SUV · ZX Petrol", "Benefit ₹30,000", "Valid till 31 Dec 2026", "T&C apply"],
      "caption": "The discount is the easy part. The monthly number is the one that decides. Urban SUV ZX Petrol at Shivaji Nagar — ₹30,000 benefit under the Festive Offer, valid to 31 Dec 2026. Finance at approved rates, T&C apply. #Pune #UrbanSUV",
      "cta": { "type": "visit_store", "value": "Shivaji Nagar Branch" },
      "facts_used": [
        { "claim": "Urban SUV ZX Petrol", "ref": "inventory:VEH_001.variant", "verified": true },
        { "claim": "₹30,000 benefit", "ref": "offers:OFFER_001.discount_amount", "verified": true },
        { "claim": "Valid till 31 Dec 2026", "ref": "offers:OFFER_001.valid_until", "verified": true }
      ],
      "disclaimers": ["Finance at approved rates", "T&C apply", "Offer valid to 31 Dec 2026"],
      "required_footage": ["exterior 3 angles", "interior dash", "salesperson to camera", "price card insert"],
      "status": "draft"
    }
  ],
  "claim_checks": [
    { "asset_id": "AST_11", "check": "prohibited_claims", "result": "pass" },
    { "asset_id": "AST_11", "check": "offer_active_and_covers_store", "result": "pass" },
    { "asset_id": "AST_11", "check": "every_fact_traced", "result": "pass" }
  ],
  "gaps": [
    { "what": "Vehicle spec sheet (mileage, features, safety rating)", "blocks": ["feature_highlight", "comparison"] }
  ]
}
```

---

## 5. Rules

- **No fact without a reference.** Every claim in copy maps to a `facts_used` entry that resolves to inventory or offers. "Best-in-class", "lowest running cost", "highest resale" are unusable unless a document in `_context/` proves them.
- **Offer validity is checked twice.** Once at generation, again at publish. Campaigns run for weeks; an offer can expire in between. The second check is what stops a live ad from advertising a dead discount.
- **Terms travel with the number.** Any discount or EMI in copy carries its conditions. A number without terms fails validation.
- **One idea, several formats.** A master idea produces per-platform variants, but each variant is its own asset with its own hash, so approving one does not approve all.
- **Variants are capped at three per ad set.** Enough to learn, few enough to attribute.
- **Voice comes from `_context/`, not from the model.** Tone, banned phrases, and formality are read, not guessed.
- **People need consent.** Delivery moments and salesperson videos require a consent record before production, because they contain a person's face.

---

## 6. Validation

| Code | Rule | On failure |
|---|---|---|
| C1 | Every factual claim resolves to `facts_used` | Asset rejected |
| C2 | Offer is active, covers the store, covers the period | Asset `blocked` |
| C3 | Prohibited-claims scan passes | Rewrite, then reject |
| C4 | Platform limits respected (caption length, duration, aspect) | Auto-trim once, then reject |
| C5 | CTA present and matches the objective | Asset rejected |
| C6 | Discount or EMI present without terms | Asset rejected |
| C7 | Consent record exists for people on camera | Asset `blocked` |
| C8 | Asset count and formats match the strategy slice | Task `partial` |

---

## 7. Failure handling

| Failure | Handling |
|---|---|
| Spec sheet missing | Write around it, record the gap. Never invent mileage or features |
| Offer expired before publish | `blocked`, raise issue with the offer id |
| Claim cannot be substantiated | Rewrite to what is provable, or drop the claim |
| Platform rejects on review | Return to content with the platform's reason verbatim |
| Two master ideas are effectively the same | Merge, so the "test" is not two copies of one message |
| Tone drifts from `_context/` | Regenerated against the voice file; three failures escalate to a human |

---

## 8. Handoff

To **Distribution**, the content pack plus per-asset hashes. Distribution may only publish assets whose hash matches an approved gate. To **Strategy/Orchestrator**, `gaps` — because a missing spec sheet is a business problem, not a copywriting problem.

---

## 9. Tabletop — GOAL_002

**Six organic assets per store, two paid variants.** `AST_11` above is the EMI explainer; `AST_12` is the family-space reel from the same store.

What the tabletop shows:

- The ₹30,000 benefit comes from `OFFER_001`, which is valid 1 Nov – 31 Dec 2026 and covers `STORE_001` and `STORE_002`. Verified, not assumed.
- The copy states the expiry date, because the offer dies the same day the campaign does.
- `feature_highlight` and `comparison` are skipped and logged as gaps: `inventory.json` has make, model, variant and price, but no mileage, features or safety data. Writing "great mileage" would be invention.
- The paid variant is produced for a paused campaign. Nothing in content stage can spend money.

---

## 10. Decisions I made (flag if you disagree)

- Content outputs creative specs, not rendered media. Production mode is declared per asset.
- Facts are passed in as a slice and cannot be looked up independently, so a mid-run stock change cannot rewrite approved copy.
- Offer validity is checked at generation *and* at publish.
- Variants capped at three; master ideas must be genuinely different or they are merged.
- Consent is a hard block for assets containing people.
