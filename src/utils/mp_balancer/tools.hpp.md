# src/utils/mp_balancer/tools.hpp

> Converts between a configuration value's comma-separated list form and an ordered list of items, in both directions.

**Needs** — [`xrCore/xrCore.h`](../../xrCore/xrCore.h.md) · [Data: Configuration](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — [`statistics_collector.cpp`](statistics_collector.cpp.md)

**Tier floor** — T3: string splitting and joining against a frozen text format.

## Purpose

The configuration format stores a list as one value with the items separated by commas.
Every tool that reads such a value has to split it, and any tool that writes one has to
join it back. These two conversions are the pair, kept together so the separator is
written down once.

The load-bearing fact is that **the separator is a bare comma with no space**, and that
the split honours quoting — an item may itself contain a comma if it is quoted, because
shipped configuration files do exactly that. The splitting rule is the core layer's, not
this file's; what this file adds is the round trip.

## `get_string_collection`

**Contract** — splits one configuration value into its items in file order, appending them
to the caller's list rather than replacing it, and returns how many were appended. Does not
fail: a value with no separator yields one item, and an empty value yields none. Allocates
one shared text object per item.

```text
FUNCTION split_value(value : text, OUT items : list<text>) -> int
  count <- item_count(value)            # the format's own rule, quoting-aware
  FOR index IN 0 .. count - 1
    items.append(item_at(value, index))
  RETURN count
```

## `get_string_from_collection`

**Contract** — joins a list of items back into one configuration value, appending to the
caller's text rather than replacing it. Emits a separator between items and none after the
last. Does not quote, escape or trim: an item that contains a comma will not survive the
round trip through this direction.

```text
FUNCTION join_items(items : list<text>, OUT value : text)
  FOR EACH item, position IN items
    value.append(item)
    IF position IS NOT last THEN value.append(",")
```

**Notes**

- The asymmetry is real and is a defect worth fixing in a rebuild: the split understands
  quoting and the join does not, so a quoted item containing a separator splits correctly
  and then rejoins into two items. Nothing in this tool round-trips such a value today,
  which is why it was never noticed.
- Both take the destination by reference and append rather than assign. That is an
  allocation-avoidance habit of the original, not a decision; returning a fresh list is
  equally correct.
