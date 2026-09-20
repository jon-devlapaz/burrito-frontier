# burrito-frontier

Grok / agent skill: Pareto-optimal menu picks on **cost**, **macros**, **deliciousness**, and **reviews**.

**v2** (post fast-food dogfood): Jev scores **deliciousness only**; macros from published nutrition in code; solo-diner gates; BYO named builds; knee/top-k + full frontier; price/review provenance.

## Install

Copy [`SKILL.md`](./SKILL.md) into your agent skills folder as `burrito-frontier/SKILL.md`, or import into Grok Bot.

Needs a TypeSafe API key for Jev deliciousness Scores.

## Usage

1. Menu (photo / PDF / link / paste)
2. Restaurant name + location
3. Optional: constraints, weights, `named_builds` (BYO), `serving_mode`

## Changelog

See [CHANGELOG.md](./CHANGELOG.md).

## License

MIT
