# Changelog

## v2 — 2026-09-20

Dogfood on McDonald’s, Chipotle, Chick-fil-A, Taco Bell, Wendy’s (Austin grounding) + Jev-appropriateness review.

### Ordered patches applied

1. Prefer published nutrition; `nutrition_source` on every item; estimate only as fallback.
2. BYO/chain mode requires `named_builds` (vessel + proteins); no bare-protein scoring.
3. Solo-diner gates default `min_kcal=350` OR `min_protein_g=20` (snack mode = both 0).
4. Always emit full Pareto frontier **and** weighted knee/top-3.
5. Price provenance: `price_basis`, `price_confidence`; location required when cost matters for chains.
6. Review evidence tiers (`dish` | `category` | `brand_default`); brand_default capped at 6.0.
7. **Jev deliciousness only** (+ optional meal-adequacy Noul). Macro composite is **code** from protein/$/density — no redundant Jev macro Score.
8. Deliciousness rubric explicitly allows grilled/core-healthy house strengths as standouts.
9. `serving_mode`: `entree_only` | `meal_normalized`.
10. Dominated notables required (3–5) with `dominated_by` + lost axes.
11. Schema v2: units, sodium optional, `jev.deliciousness_only`, confidence gating for headline wins.

### Not changed

- Four-axis Pareto still the core decision object.
- Cost remains a code/minimization axis (never sent to Jev).
