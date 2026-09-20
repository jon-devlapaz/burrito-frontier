# burrito-frontier

A agent **skill** that finds Pareto-optimal menu items across:

- **cost** (minimize)
- **macros** (maximize; estimated protein + density + protein/$)
- **deliciousness** (TypeSafe **Jev** Score)
- **reviews** (dish-level public signal)

## Install

Copy [`SKILL.md`](./SKILL.md) into your agent’s skills folder (e.g. as `burrito-frontier/SKILL.md`), or invoke it after importing into Grok Bot.

Requires a TypeSafe API key (`TYPESAFE_API_KEY`) for Jev scoring.

## Usage

Provide:

1. A menu (photo, PDF, link, or paste)
2. Restaurant name + location

Optional: constraints (budget, allergies), axis weights (ranking within the frontier only).

## Output

Fixed prose + JSON schema — see `SKILL.md` for the full contract (`frontier`, scores, sources, caveats).

## License

MIT
