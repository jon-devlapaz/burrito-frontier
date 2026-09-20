# Changelog

## v3 — 2026-09-20

Dogfood on famous Laredo independents/regionals (La Laguna, Taco Palenque, El Mesón de San Agustín, Suarez, Obregon's).

### Ordered patches applied

1. **P0** Counter-service meal template — taco/gordita/plate shops require `named_builds` (N-tacos or plates); no orphan singles when multi-item is the local norm.
2. **P0** Gate refinement — default `min_kcal` **400**; kcal-only pass with protein below min requires meal-adequacy Noul ≥ 0.5.
3. **P1** Sparse-nutrition mode — `nutrition_coverage`; soft-downweight macros in knee when nearly all estimated; show estimate honesty.
4. **P1** Price conflict protocol — >20% source spread → `price_confidence: low`, prefer in-store, optional `price_range`.
5. **P1** Knee respects Jev confidence — don't unqualified-recommend if deliciousness confidence < 0.45; prefer next confident knee item.
6. **P2** Present top-k by default; full frontier as appendix; auto ε/cluster when frontier share > 50% (`frontier_full` retained).
7. **P2** Local-language (e.g. Spanish) review grounding when English-first signal is thin.
8. **P3** `regional_combo` serving_mode for taco+side+drink norms.

Schema bump to `skill_version: 3` (`gated_out`, `frontier_full`, `nutrition_coverage`, `prices` sources).

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

### Not changed (still true in v3)

- Four-axis Pareto remains the core decision object.
- Cost remains a code/minimization axis (never sent to Jev).
