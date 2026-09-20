---
name: burrito-frontier
description: >-
  Use when picking best menu items via Pareto on cost, macros, deliciousness,
  and reviews — Jev deliciousness-only, shareable/by-the-pound gates, named
  builds, sparse nutrition, top-k-first. Location-agnostic (v4).
---
# burrito-frontier

Find Pareto-optimal menu items across **cost**, **macros**, **deliciousness**, and **reviews**. Use TypeSafe **Jev** only for semantic judgments; keep arithmetic and labels in code. Always return the standard prose + JSON schema.

This skill is **location-agnostic**: it works for any restaurant given a menu + place name/area for pricing and reviews. Do not bake in cities, chains, or house dishes.

For Jev API shapes (Choice / Noul / Score), follow the `typesafe-ai` skill and live docs at `https://docs.typesafe.ai/`.

## Capabilities (v4)

- Four-axis Pareto: cost ↓, macros ↑, deliciousness ↑, reviews ↑
- Jev **deliciousness only** (+ meal-adequacy Noul for gates); macros and cost in code
- Prefer published nutrition; estimate bands when sparse; optional sodium soft-penalty when labels are rich
- Named builds for BYO, counter-service multi-item meals, and by-the-pound service
- Solo-diner gates (default 400 kcal OR 20g protein) + kcal-only / shareable adequacy rules
- Price provenance, >20% conflict protocol, named-build price amplification
- Review specificity tiers + real local-language review retrieval
- Top-k-first presentation, confidence-aware recommendation, optional `recommended_alt`
- Auto ε / clustering when the frontier is huge; dominated notables including house icons

## Inputs

Collect what you have; ask only for missing required fields.

| Field | Required | Notes |
| --- | --- | --- |
| `menu` | yes | Photo, PDF, link, or pasted text |
| `restaurant` | yes | Name + **city/area** (needed for cost/reviews grounding — supplied per run, not hardcoded) |
| `goal` | no | Default: solo diner, equal-weight four-axis Pareto |
| `constraints` | no | Hard filters: budget, allergies, diet rules, spice max, etc. |
| `weights` | no | Soft preference; does **not** change the frontier set, only knee/top-k ranking |
| `candidate_limit` | no | Default ~12–20 mains after gates |
| `serving_mode` | no | `entree_only` (default) \| `meal_normalized` \| `regional_combo` \| `by_the_pound` |
| `named_builds` | conditional | **Required** for BYO, multi-item counter meals (e.g. N tacos / plates), and by-the-pound service: 3–15 concrete meals |
| `min_kcal` / `min_protein_g` | no | Solo defaults: **400 kcal** OR **20g protein**. Both 0 = snack mode |
| `sodium_mg_max` | no | Optional hard filter when labels exist |
| `epsilon_pareto` | no | Default `auto`: if frontier share > 0.5, light ε-dominance / cluster |
| `presentation` | no | Default `top_k_first`. `full_frontier` if user asks |
| `access_friction` | no | Optional per-run metadata: `long_wait` \| `limited_hours` \| `none` |

Normalize into:

```json
{
  "restaurant": {"name": "", "location": ""},
  "goal": "solo equal-weight Pareto",
  "constraints": [],
  "weights": {"cost": 1, "macros": 1, "deliciousness": 1, "reviews": 1},
  "serving_mode": "entree_only",
  "min_kcal": 400,
  "min_protein_g": 20,
  "presentation": "top_k_first",
  "access_friction": "none",
  "menu_source": "image|url|text"
}
```

## Steps

### 1. Parse the menu (+ meal templates)
- Extract name, price, description, category, prep, and `vessel` when relevant (`plate`, `taco`, `bowl`, `dip`, `chips`, `sandwich`, etc.).
- **BYO / multi-item counter meals:** require `named_builds` (e.g. N tacos or a plate). No orphan singles when multi-item is the local norm (unless snack mode).
- **By-the-pound service:** use `serving_mode=by_the_pound` and named builds like `½ lb meat + side`. Never treat raw $/lb alone as a solo meal without mass + side.
- Tag dips/cheese dips/chips as `shareable=true` / vessel dip|chips.
- Shareables need explicit `servings` and **per-person** price/macros before candidacy (or hit the shareable gate below).
- Apply constraints, then gates. Cap at `candidate_limit`.

### 2. Solo-diner + shareable gates
- Pass if `kcal >= min_kcal` **OR** `protein_g >= min_protein_g` (defaults 400 / 20), unless snack mode.
- **Kcal-only hole:** if kcal-only pass and protein < min → meal-adequacy Noul ≥ **0.5** required.
- **Shareable / dip gate:** if `shareable` or vessel ∈ {dip, chips, queso, side}: either (a) split by `servings` into a per-person candidate, or (b) require adequacy Noul ≥ 0.5 **even when** hard protein/kcal gates pass. Otherwise drop from meal candidacy (may remain as dominated notable / "start here" note).
- Record gated-out items in `gated_out`.

### 3. Price provenance + conflict (+ build amplification)
- Record `price_basis` and `price_confidence`.
- Prefer in-store / official app over delivery aggregators.
- If sources differ **>20%**, set confidence `low`, note both, prefer in-store; optional `price_range`.
- **Named-build amplification:** if the *unit* price conflicts >20%, the **N× build** inherits `price_confidence: low` (and widened `price_range`).
- For `by_the_pound`, default confidence ≤ `medium` and prefer `price_range` reflecting weight variance (~±10–15%).

### 4. Ground reviews (tiers + real local-language evidence)
- Prefer dish-specific mentions; `review_specificity`: `dish` | `category` | `brand_default` (cap brand_default at 6.0).
- When menu/chatter is not English-first, **retrieve and score real snippets** in the local language; fold into `review_score`, not caveats-only. Caveat if evidence remains thin.
- Optional Jev Score/Noul over short real snippets; never invent quotes.
- House icons that lose Pareto should still appear in dominated notables (`why_notable: local_icon`).

### 5. Macros (code-first) + sparse mode + bands
- Prefer published nutrition; else estimate. Per-item `nutrition_source`.
- Compute in code: protein, kcal, protein/$, protein/100kcal, optional sodium.
- Run-level `nutrition_coverage`: `rich` | `mixed` | `sparse`.
- When estimated: emit `kcal_range` / `protein_range`.
- When sparse: soft-downweight macros in **knee** (e.g. ×0.7) unless user prioritizes macros.
- When nutrition is **rich** and sodium is published, optional soft sodium penalty on macros composite so extreme-sodium builds don’t look equal to leaner peers.
- **Do not** ask Jev for macro fit when numbers are known.

### 6. Score with Jev (semantic only)
- TypeSafe System One + `jev-latest` (or current docs alias).
- Batch shared state with full candidate list.
- **Required — Deliciousness Score** (0–3), with grilled/core-healthy standouts allowed when they are a kitchen strength.
- **Meal-adequacy Noul** when gates require it (kcal-only hole and shareable gate).
- Store score, confidence, probabilities. Heuristic fallback if API fails.

### 7. Build objectives
| Axis | Direction | How |
| --- | --- | --- |
| `cost` | minimize | Per-person / meal-normalized / pound-build ticket as applicable |
| `macros` | maximize | Code composite (protein/$, density, protein floor); sparse knee downweight; optional sodium soft-penalty when rich |
| `deliciousness` | maximize | Jev Score / 3 |
| `reviews` | maximize | review_score / 10 after specificity caps |

### 8. Pareto + presentation
- Frontier: non-dominated set.
- Auto ε / cluster if frontier share > 50%; keep `frontier_full` if trimmed.
- Knee top-3 primary UX.
- If knee #1 deliciousness confidence < **0.45**, don’t unqualified-recommend it; prefer next ≥0.45 when available.
- If knee #2 confidence is much higher than #1 (e.g. ≥0.85 vs ≤0.60), set `recommended_alt` to #2 and mention it in prose.
- Dominated notables required; include cultural co-icons even when cost-dominated.
- If `access_friction` is set, mention in caveats (not as a Pareto axis).
- Default: top-k first; frontier appendix.

### 9. Deliver
Use **Standard output**.

## Standard output

### Prose
1. One-line: restaurant, recommended pick (confidence-aware), frontier size; mention `recommended_alt` if set.
2. Top picks (knee / top-3) — name, price, why.
3. How to choose (macros vs taste vs budget).
4. Full frontier — appendix unless asked up front.
5. Dominated notables (required; include house icons).
6. Caveats: nutrition_coverage; price conflicts; review language; serving_mode; gates; Jev confidence; access_friction; menu exclusions.

### JSON block

```json
{
  "skill": "burrito-frontier",
  "skill_version": 4,
  "restaurant": {"name": "", "location": ""},
  "inputs": {
    "constraints": [],
    "weights": {"cost": 1, "macros": 1, "deliciousness": 1, "reviews": 1},
    "serving_mode": "entree_only",
    "min_kcal": 400,
    "min_protein_g": 20,
    "presentation": "top_k_first",
    "access_friction": "none",
    "candidate_count": 0,
    "nutrition_coverage": "sparse"
  },
  "units": {"price": "USD", "protein": "g", "energy": "kcal", "sodium": "mg"},
  "recommended_pick": null,
  "recommended_alt": null,
  "top_k": [],
  "frontier": [],
  "frontier_full": [],
  "dominated_notable": [],
  "gated_out": [],
  "jev": {"model": "", "deliciousness_only": true},
  "sources": {"menu": "image|url|text", "reviews": [], "nutrition": [], "prices": []},
  "caveats": []
}
```

Each candidate/frontier item should support: price fields (`price_basis`, `price_confidence`, optional `price_range`), `shareable`, `vessel`, `prep`, `nutrition_source`, optional `kcal_range` / `protein_range`, wins/why, and scores including `deliciousness_confidence`.

## Rules
- Do not recommend items absent from the provided menu (or outside required named builds).
- Do not fabricate review quotes, ratings, or nutrition numbers.
- Prefer published nutrition; label estimates; be honest when sparse.
- Jev = deliciousness (+ adequacy Noul for gates). Macros and cost = code.
- Dips/shareables are not solo meals without per-person split or adequacy pass.
- Saving user prefs to memory is encouraged; do not block the run.
- **Keep the skill generic:** no single restaurant, city, cuisine brand, or user baked into the recipe. Location and venue are **inputs per run** only. Dogfood provenance belongs in CHANGELOG, not in the procedure.
