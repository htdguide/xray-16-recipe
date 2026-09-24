# src/xrGame/trade_factor_parameters.h

> Per-item-section price multiplier pairs: the sections a party *will* deal in, and on what
> terms.

**Needs** — [`trade_factors.h`](trade_factors.h.md) · [`trade_factor_parameters_inline.h`](trade_factor_parameters_inline.h.md)
**Used by** — [`trade_action_parameters.h`](trade_action_parameters.h.md) · [`trade_factor_parameters_inline.h`](trade_factor_parameters_inline.h.md)
**Tier floor** — T2: a keyed table of value pairs.

## Purpose

The positive half of one trading action's configuration: a map from item section to the
friend/enemy multiplier pair authored for it. Its negative counterpart is
[`trade_bool_parameters.h`](trade_bool_parameters.h.md), and
[`trade_action_parameters.h`](trade_action_parameters.h.md) holds one of each plus a
fallback.

Note what *absence* means here, because it is not what a reader expects: a section that is
not in this table is **not** refused. It is simply unpriced at this level, and the lookup
falls through to the global defaults and then to the action's own fallback pair. Refusal is
the other table's job. Keeping the two separate is what allows "I trade this at these rates",
"I trade this at the default rate" and "I do not trade this" to be three distinct
configurations.

## State

```text
RECORD TradeFactorParameters
  factors : map<text, TradeFactors>   # item section name -> its multiplier pair
```

**Invariants** — one entry per section; enabling a section twice is a programming error. The
table is a sorted flat array rather than a hash map, which is the right shape for a few dozen
entries looked up by an interned name.

## Exported units

- `clear()` — empty, before a configuration reload.
- `enable(section, factors)` — record a section's pair.
- `enabled(section)` — has this section a pair of its own.
- `factors(section)` — its pair; the caller must have asked `enabled` first.

See [`trade_factor_parameters_inline.h`](trade_factor_parameters_inline.h.md).
