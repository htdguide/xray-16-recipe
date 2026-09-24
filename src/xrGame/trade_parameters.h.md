# src/xrGame/trade_parameters.h

> One party's complete trading configuration: buy, sell and show, and the fallback chain
> every lookup walks.

**Needs** — [`trade_parameters.cpp`](trade_parameters.cpp.md) · [`trade_action_parameters.h`](trade_action_parameters.h.md) · [`trade_parameters_inline.h`](trade_parameters_inline.h.md)
**Used by** — [`InventoryOwner.cpp`](InventoryOwner.cpp.md) · [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) · [`inventory_owner_inline.h`](inventory_owner_inline.h.md) · [`trade2.cpp`](trade2.cpp.md) · [`trade_parameters.cpp`](trade_parameters.cpp.md) · [`trade_parameters_inline.h`](trade_parameters_inline.h.md)
**Tier floor** — T2: three action configurations and a two-level lookup.

## Purpose

Declares the top of the trading configuration stack. Every inventory owner has one of these;
there is additionally one process-wide instance holding the game's defaults, and every lookup
that misses in a party's own parameters falls through to it.

Three actions, and only two of them have prices:

- **buy** — what this party pays for an item.
- **sell** — what this party charges for one.
- **show** — whether the item appears in the trade screen at all. No factors; a refusal set
  only, which is why its loader is the one case written out longhand in
  [`trade_parameters.cpp`](trade_parameters.cpp.md).

**The action is a type, not a value.** Each of the three is named by its own empty type, and
the operations are written generically over it, so `enabled(buy, section)` and
`enabled(show, section)` select different storage of different shapes at build time with no
runtime dispatch and no possibility of asking the show action for a price. That is the
mechanism; the decision that survives a rebuild is that **the three actions are not
interchangeable and the difference should be caught at the call site**, not by a runtime
check. A rebuild with three distinct operations, or with a sum type, expresses the same
thing.

## State

```text
RECORD TradeParameters
  buy   : TradeActionParameters
  sell  : TradeActionParameters
  show  : TradeBoolParameters
  buy_item_condition_factor : real    # see below
```

**Invariants** — the fallback pairs for buy and sell are read from a named configuration
section at construction, and are *not* cleared by a reload; see
[`trade_parameters_inline.h`](trade_parameters_inline.h.md).

`buy_item_condition_factor` is initialized to zero, is public, and is written and read by
nothing in this file — it is a tunable the trade screen consults when offering to buy a
damaged item. It sits here because this is the record a party's trading configuration is
reached through.

## Exported units

- construction from a configuration section name, defaulting to the section called `trade`.
- `clear()` — empty the buy and sell tables, keeping their fallbacks.
- `instance()` / `clean()` — the process-wide defaults, created on first use and explicitly
  destroyed.
- `enabled(action, section)` — may this party perform this action on this item section.
- `factors(action, section)` — the multiplier pair to use, after walking the fallback chain.
- `process(action, file, section)` — load one action's table from configuration.
- `default_factors(action, factors)` — set one action's fallback pair.
- `default_trade_parameters()` — the process-wide instance, by a shorter name.
