# src/xrGame/trade_factors.h

> A price multiplier pair: what this costs a friend, and what it costs an enemy.

**Needs** — [`trade_factors_inline.h`](trade_factors_inline.h.md)
**Used by** — [`trade_factor_parameters.h`](trade_factor_parameters.h.md) · [`trade_factors_inline.h`](trade_factors_inline.h.md)
**Tier floor** — T2: an immutable pair of reals.

## Purpose

The atom of the trading economy. Everything above it —
[`trade_factor_parameters.h`](trade_factor_parameters.h.md),
[`trade_action_parameters.h`](trade_action_parameters.h.md),
[`trade_parameters.h`](trade_parameters.h.md) — is layers of lookup that end in one of these,
and the price formula in [`trade2.cpp`](trade2.cpp.md) does nothing with it but interpolate
between the two numbers by how the parties feel about each other.

Two numbers rather than one is the decision. A single multiplier plus a relation coefficient
would force one relationship between reputation and price on every item; a pair lets a
designer say that ammunition is priced the same for everyone and that a rare suit is only
sold to friends, per item section, per direction.

## State

```text
RECORD TradeFactors
  friend : real     # the multiplier at maximum goodwill
  enemy  : real     # the multiplier at minimum goodwill
```

**Invariants** — both default to one, so an item section with no authored factors trades at
its base cost. Neither may be an invalid number; the type checks on every construction and
on every read, because a corrupt factor propagates silently into a price and then into the
player's money.

**Nothing constrains which of the two is larger.** The interpolation in the price formula is
written to handle either ordering, so a designer may author a pair either way round. A
rebuild that assumes friend ≤ enemy, or the reverse, will misprice whichever half of the
shipped configuration disagrees with it.

Once constructed, a pair never changes.

## Exported units

- construction from the two multipliers, both defaulting to one.
- `friend_factor()` / `enemy_factor()` — the two values.

See [`trade_factors_inline.h`](trade_factors_inline.h.md) for the validity checks, which are
the only executable content.
