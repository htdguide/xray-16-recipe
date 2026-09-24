# src/xrGame/trade_bool_parameters.h

> A set of item sections a party refuses to deal in.

**Needs** — [`trade_bool_parameters_inline.h`](trade_bool_parameters_inline.h.md)
**Used by** — [`trade_action_parameters.h`](trade_action_parameters.h.md) · [`trade_bool_parameters_inline.h`](trade_bool_parameters_inline.h.md)
**Tier floor** — T2: a small membership set.

## Purpose

The simplest of the three parameter shapes: a plain list of item sections for which some
trading action is switched off. Where
[`trade_factor_parameters.h`](trade_factor_parameters.h.md) says *how much*, this one says
only *whether*.

It backs two things: the disabled half of a buy or sell action (see
[`trade_action_parameters.h`](trade_action_parameters.h.md)), and the *show* action on its
own, which has no factors at all — whether an item is even displayed in the trading screen is
a yes-or-no question, so it needs no pair of multipliers.

## State

```text
RECORD TradeBoolParameters
  sections : list<text>     # item section names this action is disabled for
```

**Invariants** — a section appears at most once; adding one twice is a programming error
rather than a tolerated duplicate. The list is small — a few dozen entries at most — and is
searched linearly, which beats a hash for this size on interned names whose comparison is a
pointer comparison.

## Exported units

- `clear()` — empty the set, before a configuration reload.
- `disable(section)` — add a section.
- `disabled(section)` — is this section in the set.

See [`trade_bool_parameters_inline.h`](trade_bool_parameters_inline.h.md).
