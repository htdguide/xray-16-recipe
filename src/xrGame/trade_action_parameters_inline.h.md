# src/xrGame/trade_action_parameters_inline.h

> Delegation to the two tables, and the fallback pair.

**Needs** — [`trade_action_parameters.h`](trade_action_parameters.h.md)
**Used by** — [`trade_action_parameters.h`](trade_action_parameters.h.md)
**Tier floor** — T3: delegation.

## Purpose

The body of [`trade_action_parameters.h`](trade_action_parameters.h.md). Every operation but
one is a single hop into whichever of the two tables owns the question; a rebuild writes them
as part of the type and deletes this file.

## `clear`

**Contract** — empties the priced table and the refused table. **Not** the fallback pair.
This is the only operation here with a decision in it, and the reason is in the header: the
fallback comes from a different configuration section, loaded at a different time, and a
clear that took it out would leave the party pricing everything at whatever the neutral
default is.

## The rest

**Contract** — `enable`, `disable`, `enabled`, `disabled` and `factors` each forward to the
priced or the refused table unchanged. `default_factors` reads and writes the fallback pair
directly, with no validation beyond what the pair type does for itself.
