---
name: burrito-frontier
description: >-
  Use when picking best menu items via Pareto on cost, macros, deliciousness,
  and reviews — Jev for deliciousness only, published nutrition preferred,
  fixed prose+JSON output (v2).
---
# burrito-frontier

Find Pareto-optimal menu items across **cost**, **macros**, **deliciousness**, and **reviews**. Use TypeSafe **Jev** only where semantic judgment is needed; keep arithmetic and labels in code. Always return the standard prose + JSON schema.

For Jev API shapes (Choice / Noul / Score), follow the `typesafe-ai` skill and live docs at `https://docs.typesafe.ai/`.

## Patch intent (v2 — post dogfood)

Ordered fixes from US fast-food dogfood + Jev-appropriateness review:

1. Prefer **published nutrition**; estimate only as fallback (`nutrition_source` on every item).
2. **BYO / chain mode:** require named builds (never score bare proteins alone).
3. **Solo-diner gates:** default `min_kcal` / `min_protein_g` so snack SKUs cannot win on cost alone.
4. Emit **full frontier + knee/top-k** under weights (4D frontiers are large).
5. **Price provenance** for chains (`price_basis`, `price_confidence`, location required when cost matters).
6. **Review evidence tiers**; down-weight brand-default scores.
7. **Jev deliciousness only** (+ optional meal-adequacy Noul); **macros computed in code** (do not ask Jev to re-score published protein/kcal).
8. Deliciousness rubric must allow **grilled/core-healthy house strengths** to be standouts.
9. **Meal vs entree** serving mode for fast food.
10. Always emit **dominated notables** with dominator pointers.
11. Use Jev **confidence** when presenting wins; extend schema (`sodium_mg` optional, units, `jev_model`).

## Inputs

Collect what you have; ask only for missing required fields.

| Field | Required | Notes |
| --- | --- | --- |
| `menu` | yes | Photo, PDF, link, or pasted text |
| `restaurant` | yes | Name + **city/area** (required when scoring cost for chains) |
| `goal` | no | Default: solo diner, equal-weight four-axis Pareto |
| `constraints` | no | Hard filters: budget, allergies, no-pork, spice max, etc. |
| `weights` | no | Soft preference; does **not** change the frontier set, only knee/top-k ranking inside it |
| `candidate_limit` | no | Default ~12–20 mains after gates |
| `serving_mode` | no | `entree_only` (default) \| `meal_normalized` (add typical side+drink cost when chains sell combos) |
| `named_builds` | conditional | **Required** for BYO menus (Chipotle-style): 3–15 concrete assemblies with vessel + proteins + key toppings |
| `min_kcal` / `min_protein_g` | no | Solo-diner defaults: **350 kcal** OR **20g protein** (either passes). Set both to 0 for snack mode |
| `sodium_mg_max` | no | Optional hard filter when labels exist |

Normalize into:

```json
{
  "restaurant": {"name": "", "location": ""},
  "goal": "solo equal-weight Pareto",
  "constraints": [],
  "weights": {"cost": 1, "macros": 1, "deliciousness": 1, "reviews": 1},
  "serving_mode": "entree_only",
  "min_kcal": 350,
  "min_protein_g": 20,
  "menu_source": "image|url|text"
}
```

## Steps

### 1. Parse the menu
- Extract name, price, description, category, prep (grilled / fried / steamed / sauced / sushi / etc.).
- **Chains / BYO:** do not expand the full combinatorial menu. Use `named_builds` or a short list of common assemblies; record `vessel` (bowl / burrito / taco / salad / plate) as a first-class field.
- Apply `constraints`, then solo-diner gates (`min_kcal` / `min_protein_g`) unless snack mode.
- Prefer single-diner mains; shareables use **per-person** price and macros.
- Cap at `candidate_limit` after gates.

### 2. Price provenance
- Record `price_basis`: `store_app` | `city_avg` | `national` | `menu_print` | `user`.
- Record `price_confidence`: `high` | `medium` | `low`.
- For national chains, prefer the user’s city; if only national averages exist, mark confidence `low`/`medium` and say so in caveats.

### 3. Ground reviews (evidence tiers)
- Search public reviews; prefer dish-specific mentions.
- Assign `review_specificity`: `dish` | `category` | `brand_default`.
- Score 0–10 from evidence. Cap **brand_default** at **6.0** unless the user weights reviews heavily.
- Require at least light dish/category evidence before scores > 6.0.
- Optional: if you have short review snippets, you may ask Jev a **Score** or **Noul** over those snippets; never invent quotes.
- Record sources briefly.

### 4. Macros (code-first)
- Prefer **published** nutrition (official PDF, aggregator cross-check). Set `nutrition_source: published | estimated`.
- Estimate from description + prep **only** when published data is missing.
- Always compute in code: `protein_g`, `kcal`, `protein_per_dollar`, `protein_per_100kcal`, optional `sodium_mg`.
- Shareables: normalize per person.
- **Do not** ask Jev to score macro fit when protein/kcal/price are already known — that re-reads a spreadsheet.

### 5. Score with Jev (semantic only)
- Ensure `TYPESAFE_API_KEY` is available. `POST https://api.typesafe.ai/v1/systemone` with model `jev-latest` (or current docs alias).
- One batch over shared `state`: restaurant, goal, full candidate list (ids, names, prep, price, published macros, short descriptors).
- **Required — Deliciousness Score** (0–3) per candidate.

Instructions must say explicitly: *Grilled, steamed, or other core-healthy items can still be standouts when they are a real strength of this kitchen (e.g. a chain known for grilled chicken). Do not reserve the top level for fried or novelty items only.*

Criteria (default):
1. Bland or a miss for this kitchen
2. Fine but forgettable
3. Solid — would order again
4. Standout on this menu

- **Optional — Meal adequacy Noul** per candidate when gates are borderline: “Is this a satisfying solo meal (not a snack) given protein and kcal?”
- Store `score`, `confidence`, `probabilities`. If `confidence` < 0.45, do not headline that item as an axis “win” without a caveat.
- If Jev fails: say so, use a transparent heuristic deliciousness score, mark source `heuristic`.

### 6. Build objectives
After constraints + gates:

| Axis | Direction | How |
| --- | --- | --- |
| `cost` | minimize | Price (or per-person). Prefer **meal-normalized** ticket when `serving_mode=meal_normalized`. Optional alternate: `$` per 25g protein for value framing in prose — raw price still drives Pareto unless user asks otherwise. |
| `macros` | maximize | **Code only:** `0.5 * min(protein_per_dollar/3, 1) + 0.3 * min(protein_per_100kcal/15, 1) + 0.2 * min(protein_g/50, 1)` (tune only if asked). Optional soft sodium penalty when labels exist. |
| `deliciousness` | maximize | Jev Score / 3 |
| `reviews` | maximize | `review_score / 10` (after specificity caps) |

### 7. Pareto + presentation
- **Frontier:** non-dominated set on the four objectives (cost via negated price).
- **Knee / top-k:** always also rank frontier items by weighted sum of normalized objectives (default weights 1,1,1,1); present **top 3** as the primary recommendation unless the user asks for the full frontier only.
- **Dominated notables:** always include 3–5 important dominated items with `dominated_by` and axes lost (icons, canonical “healthy orders,” fan favorites).
- Prose must label **which axis** each frontier item wins — never imply frontier = healthy.

### 8. Deliver
Use **Standard output**. Lead with top-k / how to choose; then full frontier; then dominated notables; then caveats.

## Standard output

### Prose
1. One-line: restaurant, top pick (knee #1), frontier size.
2. **Top picks (knee / top-3)** under current weights — name, price, why.
3. Full frontier — name, price, axis wins, one-line why.
4. How to choose (3–4 bullets: macros vs taste vs budget).
5. Dominated notables (required) — what beat them and on which axes.
6. Caveats: nutrition_source mix; price_basis/confidence; review thinness; serving_mode; Jev confidence notes; items not on the provided menu excluded.

### JSON block

```json
{
  "skill": "burrito-frontier",
  "skill_version": 2,
  "restaurant": {"name": "", "location": ""},
  "inputs": {
    "constraints": [],
    "weights": {"cost": 1, "macros": 1, "deliciousness": 1, "reviews": 1},
    "serving_mode": "entree_only",
    "min_kcal": 350,
    "min_protein_g": 20,
    "candidate_count": 0
  },
  "units": {"price": "USD", "protein": "g", "energy": "kcal", "sodium": "mg"},
  "top_k": [],
  "frontier": [
    {
      "name": "",
      "price": 0,
      "price_basis": "city_avg",
      "price_confidence": "medium",
      "wins": ["deliciousness"],
      "why": "",
      "vessel": null,
      "prep": "",
      "nutrition_source": "published",
      "scores": {
        "cost": 0,
        "macros_composite": 0,
        "deliciousness_jev": 0,
        "deliciousness_confidence": 0,
        "review_score": 0,
        "review_specificity": "dish",
        "protein_g": 0,
        "kcal": 0,
        "protein_per_dollar": 0,
        "sodium_mg": null
      }
    }
  ],
  "dominated_notable": [
    {
      "name": "",
      "dominated_by": [""],
      "lost_axes": ["macros"],
      "why_notable": ""
    }
  ],
  "jev": {"model": "", "deliciousness_only": true},
  "sources": {"menu": "image|url|text", "reviews": [], "nutrition": []},
  "caveats": []
}
```

## Rules
- Do not recommend items absent from the provided menu (or outside `named_builds` for BYO).
- Do not fabricate review quotes, ratings, or nutrition numbers.
- Prefer published nutrition; label estimates.
- Jev = deliciousness (+ optional adequacy Noul). Macros and cost = code.
- Saving user prefs (allergies, usual weights) to memory is encouraged; do not block the run.
- Keep the skill generic: no single restaurant, city, or user baked into the recipe.
