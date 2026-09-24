# src/editors/xrWeatherEditor/property_float_reference.cpp

> One grid row aliased straight onto a real field the engine owns.

**Needs** — [`property_float_reference.hpp`](property_float_reference.hpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — reached through its declarations in [`property_float_reference.hpp`](property_float_reference.hpp.md); callers name that, not this file.
**Tier floor** — T2: holds an alias into another runtime's storage; the alias has no validity check

## Purpose

The reference-bound twin of [`property_float.cpp`](property_float.cpp.md). Identical behaviour, different binding: instead of asking the engine to read and write, the row pokes the field.

## State

```text
RECORD RealPropertyByReference
  alias : ValueAlias<real>      # aliases a field the engine owns; owned wrapper,
                                # aliased storage NOT owned
  step  : real                  # invariant: > 0, checked at construction
```

## `construct(field, step)`

**Contract** — Builds the alias and stores the step. The step must be strictly positive — a zero or negative step makes the nudge affordance silently do nothing or invert, and there is no correct behaviour to fall back to. Allocates.

## `release`

**Contract** — Frees the alias wrapper. The aliased field is not touched. Runs exactly once; see [`property_float.cpp`](property_float.cpp.md) for why the rule is stated that way.

## `GetValue` · `SetValue` · `Increment`

**Contract** — Read the aliased field, write the aliased field, and nudge by `amount × step` exactly as the accessor-bound adapter does.

**Notes** — The difference between the two binding flavours is not performance; it is whether the engine gets to *observe* the write. A reference-bound row writes without the engine knowing, so it may only be used for fields the engine re-reads each frame anyway. Every quantity in a time-of-day keyframe qualifies, which is why the flavour exists; anything with a derived cache does not.

The hazard that comes with it: the alias is valid only while the engine record holding that field is alive, and nothing here can tell. If the engine frees or reallocates the record while a row still points into it, the next keystroke writes into memory that is no longer the document. A rebuild that cannot express a bare alias should bind these through the accessor flavour instead, and lose nothing but a function call.
