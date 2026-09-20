---
name: burrito-frontier
description: >-
  Use when picking best menu items via Pareto on cost, macros, deliciousness,
  and reviews — Jev deliciousness-only, published nutrition preferred, taco-shop
  named builds, refined solo gates, top-k-first output (v3).
---
# burrito-frontier

Find Pareto-optimal menu items across **cost**, **macros**, **deliciousness**, and **reviews**. Use TypeSafe **Jev** only for semantic judgments; keep arithmetic and labels in code. Always return the standard prose + JSON schema.

For Jev API shapes (Choice / Noul / Score), follow the `typesafe-ai` skill and live docs at `https://docs.typesafe.ai/`.

## Version history (intent)

**v2** (national QSR dogfood + Jev review): published nutrition preferred; BYO named builds; solo gates; frontier + top-k; price provenance; review tiers; Jev deliciousness-only; grilled rubric; meal vs entree; dominated notables; confidence/schema.

**v3** (Laredo independent / regional dogfood) — ordered patches:

1. **P0 — Counter-service meal template:** taco / gordita / plate shops use `named_builds` (N-tacos or a plate); never score orphan singles when local norm is multi-item.
2. **P0 — Gate refinement:** if kcal-only passes and protein < `min_protein_g`, require meal-adequacy Noul ≥ 0.5 (or fail the gate). Default `min_kcal` raised to **400**.
3. **P1 — Sparse-nutrition mode:** set `nutrition_coverage`; soft-downweight macros axis or show estimate bands when nearly all estimated.
4. **P1 — Price conflict protocol:** if sources differ >20%, `price_confidence: low`; prefer in-store; optional `price_range`.
5. **P1 — Knee respects Jev confidence:** auto-caveat or skip as "order this" when top-k deliciousness confidence < 0.45.
6. **P2 — Present top-k by default;** full frontier as appendix; optional ε-dominance when frontier > 50% of candidates.
7. **P2 — Spanish (or local-language) review path** when menus/chatter aren't English-first.
8. **P3 — Combo serving_mode** for regional meals (taco + side + drink).

## Inputs

Collect what you have; ask only for missing required fields.

| Field | Required | Notes |
| --- | --- | --- |
| `menu` | yes | Photo, PDF, link, or pasted text |
| `restaurant` | yes | Name + **city/area** (required when scoring cost) |
| `goal` | no | Default: solo diner, equal-weight four-axis Pareto |
| `constraints` | no | Hard filters: budget, allergies, no-pork, spice max, etc. |
| `weights` | no | Soft preference; does **not** change the frontier set, only knee/top-k ranking |
| `candidate_limit` | no | Default ~12–20 mains after gates |
| `serving_mode` | no | `entree_only` (default) \| `meal_normalized` (side+drink) \| `regional_combo` (local plate/taco+side+drink norm) |
| `named_builds` | conditional | **Required** for BYO **and** counter-service taco/gordita/plate shops: 3–15 concrete meals (e.g. `3× barbacoa tacos`, `fajita plate`). Include `vessel` / unit count |
| `min_kcal` / `min_protein_g` | no | Solo-diner defaults: **400 kcal** OR **20g protein**. Both 0 = snack mode |
| `sodium_mg_max` | no | Optional hard filter when labels exist |
| `epsilon_pareto` | no | Default `auto`: if frontier share > 0.5, apply light ε-dominance or cluster before display |
| `presentation` | no | Default `top_k_first` (full frontier in appendix/JSON). `full_frontier` if user asks |

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
  "menu_source": "image|url|text"
}
```

## Steps

### 1. Parse the menu (+ meal templates)
- Extract name, price, description, category, prep (grilled / fried / steamed / sauced / sushi / etc.).
- **BYO bowls/burritos:** use `named_builds` with vessel + proteins + key toppings — never bare proteins alone.
- **Taco / gordita / plate / counter shops:** if the local norm is N tacos or a plate, **require named builds** (e.g. 3× barbacoa, mixed fajita plate). Do **not** leave single tacos as candidates when N≥2 is the usual meal — unless snack mode.
- Apply `constraints`, then solo-diner gates (below).
- Prefer single-diner mains; shareables need explicit `servings` and **per-person** price/macros before candidacy.
- Cap at `candidate_limit` after gates.

### 2. Solo-diner gates (refined)
- Pass if `kcal >= min_kcal` **OR** `protein_g >= min_protein_g` (defaults 400 / 20), unless snack mode.
- **Kcal-only hole fix:** if the item passes on kcal alone but `protein_g < min_protein_g`, run meal-adequacy **Noul**. Require Noul ≥ **0.5** to keep; otherwise drop (or mark snack-only and exclude from cost-axis wins).
- Optionally ask the same Noul for other borderline items.

### 3. Price provenance + conflict protocol
- Record `price_basis`: `store_app` | `city_avg` | `national` | `menu_print` | `delivery_app` | `user`.
- Record `price_confidence`: `high` | `medium` | `low`.
- Prefer in-store / official app over delivery aggregators when both exist.
- If two sources differ by **>20%**, set `price_confidence: low`, note both in caveats, and prefer the in-store figure when known. You may store `price_range: [low, high]` instead of false precision.
- Location required when cost matters for multi-site brands.

### 4. Ground reviews (tiers + language)
- Prefer dish-specific mentions; assign `review_specificity`: `dish` | `category` | `brand_default`.
- Cap **brand_default** at **6.0** unless the user weights reviews heavily.
- Require light dish/category evidence before scores > 6.0.
- When the menu or local chatter is Spanish-first (or another local language), **search and use that language** for dish signal; say so in caveats if English-only evidence is thin.
- Optional: Jev Score/Noul over short real snippets; never invent quotes.

### 5. Macros (code-first) + sparse-nutrition mode
- Prefer **published** nutrition; else estimate. Set per-item `nutrition_source: published | estimated`.
- Always compute in code: `protein_g`, `kcal`, `protein_per_dollar`, `protein_per_100kcal`, optional `sodium_mg`.
- Set run-level `nutrition_coverage`: `rich` | `mixed` | `sparse` (sparse ≈ almost all estimated — typical for independents).
- When `sparse`: show estimate bands in prose; **soft-downweight** the macros axis in knee ranking (e.g. multiply macros weight by 0.7) unless the user prioritizes macros; never pretend lab precision.
- **Do not** ask Jev for macro fit when numbers are already known.

### 6. Score with Jev (semantic only)
- `TYPESAFE_API_KEY` → `POST https://api.typesafe.ai/v1/systemone`, model `jev-latest` (or current docs alias).
- Batch shared `state`: restaurant, goal, candidates (ids, names, prep, price, macros, short descriptors).
- **Required — Deliciousness Score** (0–3).

Instructions must say: *Grilled, steamed, or other core-healthy items can still be standouts when they are a real strength of this kitchen. Do not reserve the top level for fried or novelty items only.*

Criteria (default):
1. Bland or a miss for this kitchen
2. Fine but forgettable
3. Solid — would order again
4. Standout on this menu

- **Meal-adequacy Noul** when required by the gate rule (and optionally for borderlines).
- Store `score`, `confidence`, `probabilities`.
- If Jev fails: transparent heuristic deliciousness; mark `heuristic`.

### 7. Build objectives
After constraints + gates:

| Axis | Direction | How |
| --- | --- | --- |
| `cost` | minimize | Price (per-person if shared). Use meal/combo-normalized ticket when `serving_mode` says so. |
| `macros` | maximize | Code: `0.5 * min(protein_per_dollar/3, 1) + 0.3 * min(protein_per_100kcal/15, 1) + 0.2 * min(protein_g/50, 1)`. Soft sodium penalty optional. Apply sparse soft-downweight in **knee only** unless user opts out. |
| `deliciousness` | maximize | Jev Score / 3 |
| `reviews` | maximize | `review_score / 10` after specificity caps |

### 8. Pareto + presentation
- **Frontier:** non-dominated set (cost via negated price).
- **ε / clustering (auto):** if `|frontier| / |candidates| > 0.5`, apply light ε-dominance or cluster near-ties so the displayed frontier isn't ~everyone; keep full set in JSON under `frontier_full` if trimmed for display.
- **Knee / top-k:** weighted sum of normalized objectives (default weights 1,1,1,1); primary UX = **top 3**.
- **Confidence rule:** if knee #1 deliciousness `confidence` < **0.45**, do not print it as the unqualified "order this" line — caveat it and prefer the next knee item with confidence ≥ 0.45 when available.
- **Dominated notables:** always 3–5 with `dominated_by` + axes lost.
- Default presentation: **top-k first**; full frontier as appendix (unless user asks for full frontier up front).
- Label which axis each item wins — never imply frontier = healthy.

### 9. Deliver
Use **Standard output**.

## Standard output

### Prose
1. One-line: restaurant, recommended pick (confidence-aware), frontier size.
2. **Top picks (knee / top-3)** — name, price, why; caveat low-confidence heads.
3. How to choose (3–4 bullets: macros vs taste vs budget).
4. Full frontier — appendix-style unless `presentation=full_frontier`.
5. Dominated notables (required).
6. Caveats: nutrition_coverage; price conflicts; review language/thinness; serving_mode; gate drops; Jev confidence; menu exclusions.

### JSON block

```json
{
  "skill": "burrito-frontier",
  "skill_version": 3,
  "restaurant": {"name": "", "location": ""},
  "inputs": {
    "constraints": [],
    "weights": {"cost": 1, "macros": 1, "deliciousness": 1, "reviews": 1},
    "serving_mode": "entree_only",
    "min_kcal": 400,
    "min_protein_g": 20,
    "presentation": "top_k_first",
    "candidate_count": 0,
    "nutrition_coverage": "sparse"
  },
  "units": {"price": "USD", "protein": "g", "energy": "kcal", "sodium": "mg"},
  "top_k": [],
  "frontier": [],
  "frontier_full": [],
  "dominated_notable": [
    {
      "name": "",
      "dominated_by": [""],
      "lost_axes": ["macros"],
      "why_notable": ""
    }
  ],
  "gated_out": [],
  "jev": {"model": "", "deliciousness_only": true},
  "sources": {"menu": "image|url|text", "reviews": [], "nutrition": [], "prices": []},
  "caveats": []
}
```

Each frontier/top_k item should carry: `price`, `price_basis`, `price_confidence`, optional `price_range`, `wins`, `why`, `vessel`, `prep`, `nutrition_source`, and `scores` (`cost`, `macros_composite`, `deliciousness_jev`, `deliciousness_confidence`, `review_score`, `review_specificity`, `protein_g`, `kcal`, `protein_per_dollar`, `sodium_mg`).

## Rules
- Do not recommend items absent from the provided menu (or outside `named_builds` when those are required).
- Do not fabricate review quotes, ratings, or nutrition numbers.
- Prefer published nutrition; label estimates; be honest when coverage is sparse.
- Jev = deliciousness (+ adequacy Noul for gates). Macros and cost = code.
- Saving user prefs to memory is encouraged; do not block the run.
- Keep the skill generic: no single restaurant, city, or user baked into the recipe.
