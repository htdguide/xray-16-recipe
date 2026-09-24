# src/xrGame/trade_action_parameters.h

> One trading action's configuration: what it is priced at, what it refuses, and what it
> falls back to.

**Needs** — [`trade_factor_parameters.h`](trade_factor_parameters.h.md) · [`trade_bool_parameters.h`](trade_bool_parameters.h.md) · [`trade_action_parameters_inline.h`](trade_action_parameters_inline.h.md)
**Used by** — [`trade_action_parameters_inline.h`](trade_action_parameters_inline.h.md) · [`trade_parameters.h`](trade_parameters.h.md)
**Tier floor** — T2: two tables and a fallback value.

## Purpose

Everything one party has to say about one *direction* of trade — buying, or selling — across
all item sections. Three parts, and the three-way split is the decision:

- **priced** — the sections with an authored multiplier pair
  ([`trade_factor_parameters.h`](trade_factor_parameters.h.md));
- **refused** — the sections this action will not touch
  ([`trade_bool_parameters.h`](trade_bool_parameters.h.md));
- **fallback** — one pair used for every section that is neither.

So a section is in exactly one of three states, and a configuration author states only the
exceptions. Most sections are in neither table and trade at the fallback rate.

## State

```text
RECORD TradeActionParameters
  priced   : TradeFactorParameters
  refused  : TradeBoolParameters
  fallback : TradeFactors           # defaults to the neutral pair
```

**Invariants** — clearing clears the two tables and **leaves the fallback alone**. That is
deliberate: the fallback comes from the party's top-level configuration section, loaded once,
while the two tables are reloaded whenever the per-item configuration is reprocessed. A
rebuild that clears all three loses every party's base price factors on the first reload.

Nothing prevents a section from being in both tables. Refusal is checked first at the level
above, so it wins; but the state is reachable and nothing reports it.

## Exported units

- `clear()` — empty the two tables, keep the fallback.
- `enable(section, factors)` / `disable(section)` — record a priced or a refused section.
- `enabled(section)` / `disabled(section)` — the two membership questions.
- `factors(section)` — a priced section's pair.
- `default_factors()` (get and set) — the fallback.

See [`trade_action_parameters_inline.h`](trade_action_parameters_inline.h.md).
