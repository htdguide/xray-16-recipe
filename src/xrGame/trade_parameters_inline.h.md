# src/xrGame/trade_parameters_inline.h

> The fallback chain, and the loader that turns a configuration section into prices.

**Needs** — [`trade_parameters.h`](trade_parameters.h.md)
**Used by** — [`trade_parameters.h`](trade_parameters.h.md)
**Tier floor** — T2: a two-level lookup and a line parser.

## Purpose

Despite the name, this is where the substance of the trading configuration lives. Two things
here are load-bearing: the **order in which a price lookup falls back**, and the **line
format** a designer writes prices in.

## Construction

**Contract** — reads the two fallback pairs from one configuration section — a hostile and a
friendly factor for buying, and the same for selling — and leaves both action tables empty.
Blocks on the configuration layer; fails if the section or any of the four keys is missing.

**Invariants** — the four key names are in the shipped game data and are frozen.

**Notes** — the two numbers are passed to the pair type in the order (hostile, friendly),
while the pair's own parameters are (friend, enemy). The names therefore cross: what the
configuration calls the hostile factor is stored as the pair's friend factor. It does not
change any price, because the interpolation in [`trade2.cpp`](trade2.cpp.md) is written to
work whichever way round the pair is ordered and clamps to the pair's own range — which is
very likely *why* it is written that way. A rebuild should store them under the names the
configuration uses and keep the order-independent interpolation regardless.

## `enabled(action, section)`

**Contract** — may this action be performed on this item section? A refusal at **either**
level is final.

```text
FUNCTION enabled(action, section) -> bool
  IF my.action.refuses(section)        RETURN false
  IF defaults.action.refuses(section)  RETURN false
  RETURN true
```

**Invariants** — refusals accumulate downward and a party cannot override a global refusal.
An item the game refuses to trade at all is refused by everyone, and a party states only its
own additional refusals. That asymmetry with the price lookup below is deliberate.

## `factors(action, section)`

**Contract** — the multiplier pair for this action and section. The caller must have
established that the action is enabled.

```text
FUNCTION factors(action, section) -> TradeFactors
  REQUIRE enabled(action, section)
  IF my.action HAS a pair for section         RETURN it      # 1. my own price
  IF defaults.action HAS a pair for section   RETURN it      # 2. the game's price
  RETURN my.action.fallback                                  # 3. my base rate
```

**Invariants** — three levels, most specific first, and the last one is the *party's* base
rate rather than the game's. So a party with no opinion about an item still prices it by its
own character — a generous trader is generous about everything it has not been given an
explicit price for. That is what makes the configuration writable: an author sets one base
pair per trader and lists only the exceptions.

## `process(action, configuration file, section)`

**Contract** — load one action's table from a configuration section, replacing it. Fails if
the section does not exist, naming it.

```text
FUNCTION process(action, file, section)
  REQUIRE file HAS section
  action_table.clear()                       # the two tables; the fallback pair survives
  FOR EACH (key, value) IN file.section(section)
    IF key IS NOT an existing item section
      CONTINUE                               # silently skip names that are not items
    IF value IS empty
      action_table.disable(key)              # a bare key refuses
      CONTINUE
    first  := value.field(0)
    second := value HAS a second field ? value.field(1) : first
    action_table.enable(key, TradeFactors(first, second))
```

**Invariants** — the line format is the designer-facing half of the whole trading system.

*A bare key refuses.* Same convention as the show action's loader.

*One number means both.* A line with a single value sets the friendly and hostile factors
equal — "this item costs the same whoever you are" — which is what most lines are, and
writing it once is the point. An earlier, stricter version of this loader required exactly
two and failed otherwise; the relaxation is deliberate and a rebuild must keep it or a large
part of the shipped configuration stops loading.

*A key naming no existing item section is skipped in silence.* That tolerates configuration
shared between the three games, which do not all ship the same items. It also swallows typos,
with no report anywhere — a price that silently does not apply is one of the harder things to
diagnose in this data, and a rebuild would do well to report at a diagnostic level while
still not failing.

## `instance` / `clean` / `default_trade_parameters`

**Contract** — the process-wide defaults, created on first use and explicitly destroyed.
Lazy creation keeps it out of the startup ordering problem: it must not exist before the
configuration layer does, and creating it on first use guarantees that without anyone
declaring an order. Unlike most of this engine's lazy globals, this one has an explicit
teardown.

## `default_factors(action, factors)`

**Contract** — sets one action's fallback pair, which is how a party's base rates are
installed from outside after construction.

## `action(tag)`

**Contract** — selects the storage for an action tag. Six one-line selectors, three readable
and three writable; they are the mechanism that makes the generic operations above resolve to
the right table at build time, and they disappear in a rebuild that names the three actions
directly.
