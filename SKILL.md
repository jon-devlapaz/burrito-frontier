---
name: burrito-frontier
description: >-
  Use when picking the best menu items via a Pareto frontier on cost, macros,
  deliciousness, and reviews — with TypeSafe/Jev scoring and a fixed output
  schema.
---
# burrito-frontier

Find the Pareto-optimal items on a restaurant menu across **cost**, **macros**, **deliciousness**, and **reviews**. Score deliciousness (and macro fit) with TypeSafe **Jev**; return a fixed schema every time.

For Jev API shapes (Choice / Noul / Score), follow the `typesafe-ai` skill and live docs at `https://docs.typesafe.ai/`.

## Inputs

Collect what you have; ask only for missing required fields.

| Field | Required | Notes |
| --- | --- | --- |
| `menu` | yes | Photo, PDF, link, or pasted text of the menu to score |
| `restaurant` | yes | Name + city/area (for reviews and kitchen context) |
| `goal` | no | Default: solo diner optimizing the four axes equally |
| `constraints` | no | Hard filters: budget cap, allergies, no-pork, spice max, share size, etc. |
| `weights` | no | Soft preference among axes; does **not** change the frontier, only ranking *within* it |
| `candidate_limit` | no | Default ~12–20 individual entrees/mains; exclude huge party trays unless asked |

Normalize into:

```json
{
  "restaurant": {"name": "", "location": ""},
  "goal": "solo equal-weight Pareto",
  "constraints": [],
  "weights": {"cost": 1, "macros": 1, "deliciousness": 1, "reviews": 1},
  "menu_source": "image|url|text"
}
```

## Steps

### 1. Parse the menu
- Extract item name, price, short description, category, prep (grilled / fried / steamed / sauced / sushi / etc.).
- Keep only orderable items in scope (respect `constraints` and `candidate_limit`).
- Prefer mains a single diner would order; note shareable platters separately with per-person price if included.

### 2. Ground reviews
- Search public reviews for this restaurant (and location). Prefer dish-specific mentions.
- Build a 0–10 `review_score` per candidate from evidence (named praise, “favorites,” critic callouts). Default ~5.0 when no dish-level signal.
- Record sources briefly; never invent quotes.

### 3. Estimate macros
- For each candidate, estimate `protein_g` and `kcal` from description + prep (label as estimates).
- Derive helpers: `protein_per_dollar`, `protein_per_100kcal`.
- Shareables: normalize protein/kcal (and cost) **per person** for fair comparison.

### 4. Score with Jev
- Ensure `TYPESAFE_API_KEY` is available. Call `POST https://api.typesafe.ai/v1/systemone` with model `jev-latest`.
- One batch (or few batches) over the same `state`: restaurant context + candidate list.
- Per candidate, ask two **Score** questions (0–3 levels):

**Deliciousness** — likely taste vs this kitchen’s strengths and known praise.

Criteria (example):
1. Bland or a miss for this kitchen  
2. Fine but forgettable  
3. Solid — order again  
4. Standout on this menu  

**Macro fit** — high protein, controlled calories, protein-per-dollar; prefer grilled/steamed/seafood over heavy fry/cream when that matches the goal.

Criteria (example):
1. Poor macros  
2. Mediocre tradeoff  
3. Good protein for the calories/price  
4. Excellent macro pick here  

- Store raw `score`, `confidence`, and `probabilities` for each.

### 5. Build objectives
For each candidate (after constraints):

| Axis | Direction | How |
| --- | --- | --- |
| `cost` | minimize | menu price (or per-person for shares) |
| `macros` | maximize | blend Jev macro score + protein/$ + protein density |
| `deliciousness` | maximize | Jev deliciousness score normalized |
| `reviews` | maximize | `review_score / 10` |

Suggested macro composite (tune only if the user asks):

`0.5 * (jev_macros/3) + 0.3 * min(protein_per_dollar/3, 1) + 0.2 * min(protein_per_100kcal/15, 1)`

### 6. Pareto frontier
- An item is on the frontier if no other feasible item is ≥ on all four objectives and > on at least one (cost compared via negated price).
- Sort frontier for display by deliciousness, then macros, then lower cost — then re-rank with `weights` only as a presentation aid.
- List a short “dominated but notable” set when useful.

### 7. Deliver
Use the **Standard output** below. Lead with the frontier; put method caveats after. Offer one optional re-weight (budget, spice, macros-only) — do not ask a checklist.

## Standard output

Always return this shape in the user-facing reply (prose + the JSON block).

### Prose
1. One-line result: how many items on the frontier, restaurant name.
2. Frontier list — for each item: **name**, price, which axis(es) it wins, one-line why.
3. How to choose (3–4 bullets mapping goals → frontier picks).
4. Dominated notables (optional, short).
5. Caveats: estimated macros; sparse dish-level reviews; items not on the provided menu were excluded.

### JSON block

```json
{
  "skill": "burrito-frontier",
  "restaurant": {"name": "", "location": ""},
  "inputs": {
    "constraints": [],
    "weights": {"cost": 1, "macros": 1, "deliciousness": 1, "reviews": 1},
    "candidate_count": 0
  },
  "frontier": [
    {
      "name": "",
      "price": 0,
      "wins": ["deliciousness"],
      "why": "",
      "scores": {
        "cost": 0,
        "macros_composite": 0,
        "deliciousness_jev": 0,
        "deliciousness_confidence": 0,
        "review_score": 0,
        "est_protein_g": 0,
        "est_kcal": 0,
        "protein_per_dollar": 0
      }
    }
  ],
  "dominated_notable": [],
  "sources": {
    "menu": "image|url|text",
    "reviews": []
  },
  "caveats": [
    "Macros are estimates, not lab labels.",
    "Dish-level review signal may be thin."
  ]
}
```

## Rules
- Do not recommend items absent from the provided menu.
- Do not fabricate review quotes or ratings.
- If Jev/API fails, say so, fall back to transparent heuristic scores, and mark `deliciousness` source as `heuristic` in caveats.
- Saving preferences the user states (allergies, usual weights) to memory is encouraged; do not block the run on preferences.
- Keep the skill generic: no single restaurant, city, or user baked into the recipe.
