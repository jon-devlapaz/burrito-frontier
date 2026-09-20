# Changelog

## v4 — 2026-09-20

Dogfood on famous Austin spots: Torchy’s Tacos, Terry Black’s BBQ, Matt’s El Rancho, Veracruz All Natural, Juan in a Million.

### Ordered patches applied

1. **P0** Shareable/dip vessel gate — dips/queso/chips need per-person split or adequacy Noul ≥ 0.5 even if kcal/protein pass (Bob Armstrong / Torchy’s queso).
2. **P0** BBQ `by_the_pound` template — named builds (mass + side); medium default price confidence; `price_range` for weigh variance.
3. **P1** Named-build price amplification — unit conflict >20% → N× build inherits low confidence.
4. **P1** Estimate bands — `kcal_range` / `protein_range` when estimated.
5. **P1** `access_friction` caveat (tourist line / hours) — prose/metadata only, not a Pareto axis.
6. **P2** Local-language review retrieval into scores (not caveats-only).
7. **P2** `recommended_alt` when knee #2 confidence ≫ #1.
8. **P3** Optional sodium soft-penalty when published coverage is rich.
9. **P3** Cultural co-icon dominated notables (`why_notable: local_icon`).

Schema bump to `skill_version: 4` (`recommended_pick`, `recommended_alt`, `access_friction`, shareable/vessel fields, estimate bands).

## v3 — 2026-09-20

Dogfood on famous Laredo independents/regionals (La Laguna, Taco Palenque, El Mesón de San Agustín, Suarez, Obregon's).

### Ordered patches applied

1. **P0** Counter-service meal template — taco/gordita/plate shops require `named_builds`; no orphan singles when multi-item is the local norm.
2. **P0** Gate refinement — default `min_kcal` **400**; kcal-only pass with protein below min requires meal-adequacy Noul ≥ 0.5.
3. **P1** Sparse-nutrition mode — `nutrition_coverage`; soft-downweight macros in knee when nearly all estimated.
4. **P1** Price conflict protocol — >20% source spread → `price_confidence: low`, prefer in-store, optional `price_range`.
5. **P1** Knee respects Jev confidence — don't unqualified-recommend if deliciousness confidence < 0.45.
6. **P2** Present top-k by default; full frontier as appendix; auto ε/cluster when frontier share > 50%.
7. **P2** Local-language (e.g. Spanish) review grounding when English-first signal is thin.
8. **P3** `regional_combo` serving_mode for taco+side+drink norms.

## v2 — 2026-09-20

Dogfood on McDonald’s, Chipotle, Chick-fil-A, Taco Bell, Wendy’s (Austin grounding) + Jev-appropriateness review.

Published nutrition preferred; BYO named builds; solo gates; frontier + top-k; price provenance; review tiers; **Jev deliciousness only** (macros in code); grilled rubric; meal vs entree; dominated notables; schema v2.

### Still true through v4

- Four-axis Pareto remains the core decision object.
- Cost remains a code/minimization axis (never sent to Jev).
