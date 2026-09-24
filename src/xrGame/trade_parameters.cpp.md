# src/xrGame/trade_parameters.cpp

> Loading the *show* action: which items a party will not even display in the trade screen.

**Needs** — [`trade_parameters.h`](trade_parameters.h.md)
**Used by** — [`trade_parameters.h`](trade_parameters.h.md)
**Tier floor** — T2: reading a configuration section into a refusal set.

## Purpose

Almost all of [`trade_parameters.h`](trade_parameters.h.md) is written generically over the
three trading actions and lives in
[`trade_parameters_inline.h`](trade_parameters_inline.h.md). This file holds the one case
that cannot be: loading the **show** action, which has no price factors and is therefore a
plain refusal set rather than the priced/refused/fallback triple the other two use.

The file also holds the one process-wide handle to the default parameters — the fallback
every party's lookup ends at.

## State

```text
process-wide : optional<TradeParameters>    # the defaults; created on first use
```

## `process(show, configuration file, section)`

**Contract** — reads one configuration section into the show action's refusal set. Replaces
whatever was there. Fails if the section does not exist.

```text
FUNCTION process_show(file, section)
  REQUIRE file HAS section
  show.clear()
  FOR EACH (key, value) IN file.section(section)
    IF value IS empty
      show.disable(key)      # a bare key with no value means "do not show this section"
```

**Invariants** — the rule is that **a key with an empty value is a refusal, and a key with
any value is ignored**. That is the same convention the priced actions use for their disabled
entries (see [`trade_parameters_inline.h`](trade_parameters_inline.h.md)), so a designer
writes one thing across all three actions: a bare line refuses, a line with numbers prices.
For the show action there is nothing to price, so every non-bare line is simply skipped.

Unlike the priced actions' loader, this one does **not** check that the key names an item
section that exists. A typo in the show list is silently harmless — the section it names is
never traded — where in a priced list it would be skipped for the same reason. Neither
reports.

**Notes** — the whole loader is one case of a generic operation that the other two actions
reach through the inline file, separated only because the show action's parameter type
differs. A rebuild whose three actions share one representation has no such split.
