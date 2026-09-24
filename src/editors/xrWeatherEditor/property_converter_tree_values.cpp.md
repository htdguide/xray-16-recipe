# src/editors/xrWeatherEditor/property_converter_tree_values.cpp

> One answer: no, this row cannot be typed into.

**Needs** — [`property_converter_tree_values.hpp`](property_converter_tree_values.hpp.md)
**Used by** — reached through its declarations in [`property_converter_tree_values.hpp`](property_converter_tree_values.hpp.md); callers name that, not this file.
**Tier floor** — T3: pure presentation; it never touches engine memory

## Purpose

Attached by [`property_holder_string.cpp`](property_holder_string.cpp.md) to any text row whose value must come from a chooser — a tree of asset names, or a file dialog — and where the engine asked that typing be forbidden. The whole file is that one refusal.

## State

Stateless.

## `CanConvertFrom`

**Contract** — False for every source shape. With no conversion into the row's value available, the grid cannot commit a typed string, and the row's modal chooser becomes the only way to change it.

**Notes** — Whether a rebuild needs a file here at all depends entirely on how its property grid says "not editable as text". The *decision* is what survives: an asset reference that must resolve should not be typeable, because a typo produces a weather set that loads with a missing asset and no diagnostic. The mechanism is disposable; the rule is not.
